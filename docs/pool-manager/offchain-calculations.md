1. **Maps all relevant state variables** tracking inflows and outflows of ECM tokens across **all pools**.
2. **Aggregates** total expected vs. actual ECM balance in the `PoolManager` contract.
3. **Calculates the total deficit** (if any) and reports **how much ECM is needed to cover it**.
4. **Per-pool breakdown** for auditability.

---

## 🔍 Step 1: Map All ECM Inflows and Outflows

### ✅ **ECM Inflows (Tokens received by contract)**
| Source | State Variable(s) | Description |
|--------|-------------------|-------------|
| Sale allocation | `pool.allocatedForSale` | ECM sent by admin for user purchases |
| Reward allocation | `pool.allocatedForRewards` | ECM sent by admin for staking rewards |
| Liquidity refill | Implicit via `refillPoolManager()` | Refunded ECM from LiquidityManager → increases contract balance |

> **Total Inflow per pool** = `allocatedForSale + allocatedForRewards`

> **Note**: Refills are already reflected in contract balance and **reduce net outflow**—handled via `liquidityPoolOwedECM`.

---

### 🚫 **ECM Outflows (Tokens no longer in contract)**
| Destination | State Variable(s) | Description |
|------------|-------------------|-------------|
| Users (principal) | `pool.lifetimeUnstakeVolume` | ECM returned to users on unstake |
| Rewards paid | `pool.rewardsPaid` | ECM paid as rewards (immediate or vested) |
| Penalties | `pool.totalPenaltiesCollected` | ECM slashed and sent to `penaltyReceiver` |
| Liquidity sent | `pool.liquidityPoolOwedECM` | **Net** ECM transferred to LiquidityManager (after refills) |

> **Total Outflow per pool** =  
> `rewardsPaid + liquidityPoolOwedECM + lifetimeUnstakeVolume + totalPenaltiesCollected`

> ✅ **Why `liquidityPoolOwedECM` and not `ecmMovedToLiquidity`?**  
> Because `refillPoolManager()` reduces `liquidityPoolOwedECM`, making it the **net outstanding amount** owed to liquidity system.

---

### 🧮 **Expected Balance per Pool**
```text
expectedBalance[poolId] = (allocatedForSale + allocatedForRewards)
                        - (rewardsPaid + liquidityPoolOwedECM + lifetimeUnstakeVolume + totalPenaltiesCollected)
```

### 🏦 **Actual Balance**
- Read once: `actualBalance = ecm.balanceOf(PoolManager.address)`

---

## 📜 Step 2: Pseudo-Code for Reconciliation Script

```pseudo
// ==============================
// OFF-CHAIN POOL RECONCILIATION SCRIPT
// ==============================

INPUT:
- PoolManager contract address (pmAddr)
- ECM token contract address (ecmAddr)
- Total number of pools: N = pmAddr.poolCount()
- Web3/RPC provider (to read on-chain state)

OUTPUT:
- totalExpectedBalance: uint256
- actualBalance: uint256
- totalDeficit: uint256
- perPoolReport: list of { poolId, expected, outflows, deficit }
- totalECMNeededToCoverDeficit: uint256

// ------------------------------
// STEP 1: Fetch actual ECM balance in PoolManager
// ------------------------------
actualBalance = ecmAddr.balanceOf(pmAddr)

// ------------------------------
// STEP 2: Initialize accumulators
// ------------------------------
totalExpectedBalance = 0
totalDeficit = 0
perPoolReport = []

// ------------------------------
// STEP 3: Iterate over all pools
// ------------------------------
for poolId from 0 to N - 1:
    pool = pmAddr.getPoolInfo(poolId)

    // Inflows
    allocatedSale = pool.allocatedForSale
    allocatedRewards = pool.allocatedForRewards
    totalInflow = allocatedSale + allocatedRewards

    // Outflows (net)
    rewardsPaid = pool.rewardsPaid
    liquidityOwed = pool.liquidityPoolOwedECM          // NET owed (after refills)
    unstakedPrincipal = pool.lifetimeUnstakeVolume
    penalties = pool.totalPenaltiesCollected

    totalOutflow = rewardsPaid + liquidityOwed + unstakedPrincipal + penalties

    // Expected balance for this pool
    expected = totalInflow - totalOutflow
    if expected < 0:
        // This should never happen in a healthy system
        log("WARNING: Negative expected balance for pool", poolId)

    totalExpectedBalance += expected

    // Compute per-pool deficit
    // Note: We cannot know per-pool actual balance (all pools share one contract balance)
    // So per-pool "deficit" is conceptual; only total deficit is real
    // But we track expected per pool for reporting

    perPoolReport.append({
        poolId: poolId,
        allocatedForSale: allocatedSale,
        allocatedForRewards: allocatedRewards,
        rewardsPaid: rewardsPaid,
        liquidityOwed: liquidityOwed,
        unstakedPrincipal: unstakedPrincipal,
        penalties: penalties,
        expectedBalance: expected
    })

// ------------------------------
// STEP 4: Compute global deficit
// ------------------------------
// The contract holds one ECM balance shared by all pools
// So total expected across all pools must equal actual balance (plus any surplus)

if actualBalance >= totalExpectedBalance:
    totalDeficit = 0
    surplus = actualBalance - totalExpectedBalance
    log("✅ Reconciliation OK. Surplus ECM:", surplus)
else:
    totalDeficit = totalExpectedBalance - actualBalance
    log("❌ DEFICIT DETECTED:", totalDeficit, "ECM")

totalECMNeededToCoverDeficit = totalDeficit

// ------------------------------
// STEP 5: Generate report
// ------------------------------
print("=== POOL MANAGER RECONCILIATION REPORT ===")
print("Total Pools:", N)
print("Actual ECM in Contract:", actualBalance)
print("Total Expected ECM (sum over pools):", totalExpectedBalance)
print("Total Deficit:", totalDeficit)
print("ECM Required to Restore Solvency:", totalECMNeededToCoverDeficit)

print("\n--- Per-Pool Breakdown ---")
for report in perPoolReport:
    print("Pool", report.poolId, ":")
    print("  Allocated (Sale + Rewards):", report.allocatedForSale + report.allocatedForRewards)
    print("  Outflows (Rewards + Liquidity + Unstake + Penalties):",
          report.rewardsPaid + report.liquidityOwed + report.unstakedPrincipal + report.penalties)
    print("  Expected Balance:", report.expectedBalance)

if totalDeficit > 0:
    print("\n🚨 ACTION REQUIRED: Inject", totalDeficit, "ECM into PoolManager to cover deficit.")
else:
    print("\n✅ No action needed. Contract is solvent.")
```

---

## 📌 Key Notes

1. **Shared Balance**: All pools share **one contract balance**—you cannot isolate per-pool actual balances. Reconciliation is **aggregate-only**.
2. **Refills Handled Correctly**: By using `liquidityPoolOwedECM` (not `ecmMovedToLiquidity`), refills from `LiquidityManager` are automatically accounted for.
3. **Deficit = Systemic Risk**: A deficit means the contract **cannot fulfill all user claims** (unstakes + rewards). Immediate top-up required.
4. **Surplus is OK**: Extra tokens may exist due to rounding, unclaimed dust, or manual deposits—non-critical.

---

```solidity
function calculateExpectedRewards(uint256 poolId, address user, uint256 durationSeconds) external view returns (uint256 expectedRewards) {
    if (poolId >= poolCount) revert PoolDoesNotExist();
    Pool storage pool = pools[poolId];
    UserInfo storage userInf = userInfo[poolId][user];
    if (pool.totalStaked == 0 || userInf.staked == 0) {
        return 0;
    }

    if (pool.rewardStrategy == RewardStrategy.WEEKLY) {
        // Calculate using weekly rewards
        uint256 totalPoolRewards = 0;
        uint256 timeProcessed = 0;
        uint256 currentWeekIndex = pool.weeklyRewardIndex;
        uint256 currentTime = pool.lastRewardTime;

        while (timeProcessed < durationSeconds && currentWeekIndex < pool.weeklyRewards.length) {
            uint256 weekEndTime = pool.weeklyRewardStart + (currentWeekIndex + 1) * WEEK_SECONDS;
            uint256 timeLeftInDuration = durationSeconds - timeProcessed;

            if (currentTime >= weekEndTime) {
                currentWeekIndex++;
                currentTime = weekEndTime;
                continue;
            }

            uint256 timeLeftInWeek = weekEndTime - currentTime;
            uint256 timeInThisWeek = timeLeftInDuration < timeLeftInWeek ? timeLeftInDuration : timeLeftInWeek;

            uint256 weekReward = pool.weeklyRewards[currentWeekIndex];
            uint256 rewardRate = (weekReward * PRECISION) / WEEK_SECONDS;
            totalPoolRewards += (timeInThisWeek * rewardRate) / PRECISION;

            timeProcessed += timeInThisWeek;
            currentTime += timeInThisWeek;

            if (timeInThisWeek == timeLeftInWeek) {
                currentWeekIndex++;
                currentTime = weekEndTime;
            }
        }
        expectedRewards = (userInf.staked * totalPoolRewards) / pool.totalStaked;
    }
    // ... existing LINEAR/MONTHLY cases
}
```