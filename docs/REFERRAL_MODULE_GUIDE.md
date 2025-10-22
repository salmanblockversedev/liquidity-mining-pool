# ReferralModule Implementation Guide

## Overview

The **ReferralModule** is an independent smart contract that manages a two-tier referral commission system for the ECM PoolManager ecosystem:

1. **Direct Commission**: Instant commission paid when a buyer stakes ECM using a referral code (based on staked amount)
2. **Multi-Level Reward Commission**: Off-chain calculated commissions paid when buyers claim rewards (based on reward amount, distributed via Merkle proofs)

## Architecture

### Contract Structure
```
ReferralModule (Independent Contract)
├── Referral Code Management
│   ├── Register codes with commission rates
│   ├── Track usage and limits
│   └── Validate on purchase
├── Direct Commission System
│   ├── Calculate commission on stake
│   ├── Transfer immediately OR accrue
│   └── Track buyer→referrer relationships
└── Multi-Level Reward System
    ├── Emit events for off-chain engine
    ├── Accept funded Merkle roots
    └── Allow claims with proof verification
```

### Integration with PoolManager
```
User → PoolManager.buyAndStake(poolId, maxUsdt, duration, referralCode)
       ↓
       PoolManager calculates ECM, receives USDT, auto-stakes
       ↓
       PoolManager.recordReferral(codeHash, buyer, poolId, stakedAmount)
       ↓
       ReferralModule.recordPurchaseAndPayDirect()
       ├── Validates code
       ├── Links buyer→referrer
       ├── Calculates directAmount = stakedAmount * directBps / 10000
       ├── Transfers OR accrues commission
       └── Returns (referrer, directAmount)
```

## Key Features

### 1. Referral Code System
- **Hash-based storage**: Codes stored as `keccak256(codeString)`
- **Commission configuration**: 
  - `directBps`: Direct commission rate (max 20% = 2000 bps)
  - `mlBps`: Array of multi-level rates [L1, L2, L3, ...] (max total 50%)
- **Usage limits**: Optional `maxUses` and `expiry` timestamp
- **Transfer modes**: 
  - `transferOnUse = true`: Instant transfer from contract balance
  - `transferOnUse = false`: Accrue for batch withdrawal

### 2. Anti-Fraud Security
- **Self-referral prevention**: Buyer cannot be their own referrer
- **Cyclic prevention**: Prevents 2-person referral loops
- **Immutable relationships**: `referrerOf[buyer]` set once, permanent
- **Usage tracking**: Enforces `maxUses` limits
- **Expiry validation**: Checks code expiration timestamps

### 3. Direct Commission Flow
```solidity
// Option A: Immediate Transfer (transferOnUse = true)
1. User stakes 1000 ECM
2. ReferralModule calculates: 1000 * 1000 bps / 10000 = 100 ECM
3. ReferralModule transfers 100 ECM to referrer instantly
4. Emits DirectCommissionPaid event

// Option B: Accrual (transferOnUse = false)
1. User stakes 1000 ECM
2. ReferralModule calculates: 100 ECM commission
3. directAccrued[referrer] += 100 ECM
4. Emits DirectCommissionAccrued event
5. Admin/referrer withdraws later via withdrawDirectAccrual()
```

### 4. Multi-Level Reward Commission Flow
```
Off-Chain Engine Process:
1. Listen to RewardClaimRecorded events
2. Query referrerOf[buyer] chain: L1, L2, L3, ...
3. Calculate commissions:
   - L1: rewardAmount * mlBps[0] / 10000
   - L2: rewardAmount * mlBps[1] / 10000
   - L3: rewardAmount * mlBps[2] / 10000
4. Aggregate all commissions into epoch (e.g., weekly batch)
5. Build Merkle tree: leaves = keccak256(beneficiary, token, amount, epochId)
6. Fund contract with totalAmount
7. Submit root: submitReferralPayoutRoot(epochId, token, totalAmount, root, expiry)

On-Chain Claim Process:
1. User fetches proof from off-chain API
2. User calls claimPayout(epochId, token, amount, proof)
3. ReferralModule verifies Merkle proof
4. Transfers tokens to user
5. Marks epoch claim as used
```

## Smart Contract API

### Admin Functions

#### `setPoolManager(address _poolManager)`
- Sets the authorized PoolManager address
- Only PoolManager can call integration functions

#### `registerReferralCode(...)`
```solidity
function registerReferralCode(
    bytes32 codeHash,        // keccak256(abi.encodePacked(codeString))
    address owner,            // Referrer address
    uint16 directBps,         // Direct commission (e.g., 1000 = 10%)
    uint16[] calldata mlBps,  // Multi-level rates [500, 300, 200] = [5%, 3%, 2%]
    uint32 maxUses,           // 0 = unlimited
    uint64 expiry,            // 0 = never expires
    bool transferOnUse        // true = instant, false = accrue
) external onlyOwner
```

**Constraints**:
- `directBps` ≤ 2000 (20%)
- `mlBps.length` ≤ 10 levels
- `sum(mlBps)` ≤ 5000 (50%)

#### `updateReferralCode(...)`
Updates commission rates for existing code (same signature as register)

#### `revokeReferralCode(bytes32 codeHash)`
Deactivates a code (sets `active = false`)

#### `activateReferralCode(bytes32 codeHash)`
Reactivates a revoked code

#### `submitReferralPayoutRoot(...)`
```solidity
function submitReferralPayoutRoot(
    uint256 epochId,
    address token,
    uint256 totalAmount,
    bytes32 merkleRoot,
    uint64 expiry
) external onlyOwner
```
Submits a Merkle root for batch reward commission payouts.
**Requirements**:
- Contract must have `totalAmount` of `token` balance
- EpochId must be unique

#### `withdrawUnclaimed(uint256 epochId, address to)`
Withdraws unclaimed funds after epoch expiry (admin only)

#### `fundContract(address token, uint256 amount)`
Funds contract for direct commission transfers

### Integration Functions (Called by PoolManager)

#### `recordPurchaseAndPayDirect(...)`
```solidity
function recordPurchaseAndPayDirect(
    bytes32 codeHash,
    address buyer,
    uint256 poolId,
    uint256 stakedAmount,
    address token
) external onlyPoolManager returns (address referrer, uint256 directAmount)
```
**Process**:
1. Validates referral code (active, not expired, under maxUses)
2. Prevents self-referral and cycles
3. Sets `referrerOf[buyer] = code.owner` (once)
4. Calculates `directAmount = stakedAmount * directBps / 10000`
5. Transfers OR accrues commission
6. Returns referrer and amount for PoolManager logging

#### `recordRewardClaimEvent(...)`
```solidity
function recordRewardClaimEvent(
    address claimant,
    uint256 poolId,
    uint256 rewardAmount,
    bytes32 claimTxHash
) external onlyPoolManager
```
Emits `RewardClaimRecorded` event for off-chain engine to process.

### User Functions

#### `claimPayout(...)`
```solidity
function claimPayout(
    uint256 epochId,
    address token,
    uint256 amount,
    bytes32[] calldata proof
) external nonReentrant
```
Claims multi-level reward commission using Merkle proof.

**Verification**:
- Epoch exists and is funded
- User hasn't claimed in this epoch
- Merkle proof valid for `keccak256(abi.encodePacked(user, token, amount, epochId))`

#### `withdrawDirectAccrual(uint256 amount)`
Withdraws accrued direct commissions (for `transferOnUse = false` mode).
Pass `amount = 0` to withdraw all.

### View Functions

#### `getReferrer(address buyer) → address`
Returns the referrer for a buyer (or address(0) if none)

#### `getReferralCodeInfo(bytes32 codeHash) → (...)`
Returns complete referral code configuration

#### `getPayoutRootInfo(uint256 epochId) → (...)`
Returns Merkle root information and claim statistics

#### `hasClaimed(uint256 epochId, address user) → bool`
Checks if user has claimed in a specific epoch

#### `getDirectAccrual(address referrer) → uint256`
Returns accrued direct commission balance

#### `calculateDirectCommission(bytes32 codeHash, uint256 stakedAmount) → uint256`
Calculates expected direct commission (off-chain preview)

#### `getReferralChain(address buyer, uint8 maxLevels) → address[]`
Returns referral chain [L1, L2, L3, ...] for a buyer

## Commission Calculation Rules

### Direct Commission (Staked Amount Based)
```solidity
directAmount = floor(stakedAmount * directBps / 10000)
```
**Example**: 
- Staked: 1000 ECM
- directBps: 1000 (10%)
- Commission: 1000 * 1000 / 10000 = 100 ECM

### Multi-Level Reward Commission (Reward Amount Based)
```solidity
// Off-chain calculation per level
level1Commission = floor(rewardAmount * mlBps[0] / 10000)
level2Commission = floor(rewardAmount * mlBps[1] / 10000)
level3Commission = floor(rewardAmount * mlBps[2] / 10000)
```

**Example**:
- Reward claimed: 500 ECM
- mlBps: [500, 300, 200] = [5%, 3%, 2%]
- L1 commission: 500 * 500 / 10000 = 25 ECM
- L2 commission: 500 * 300 / 10000 = 15 ECM
- L3 commission: 500 * 200 / 10000 = 10 ECM

**Rounding**: Uses floor division (integer math). Total funded amount must equal sum of all beneficiary amounts.

## Events

### Referral Code Management
```solidity
event ReferralCodeRegistered(bytes32 indexed codeHash, address indexed owner, uint16 directBps, uint16[] mlBps, bool transferOnUse);
event ReferralCodeRevoked(bytes32 indexed codeHash);
```

### Direct Commissions
```solidity
event ReferrerLinked(address indexed buyer, address indexed referrer, bytes32 indexed codeHash);
event DirectCommissionPaid(address indexed referrer, address indexed buyer, uint256 indexed poolId, uint256 stakedAmount, uint256 amount, bytes32 codeHash);
event DirectCommissionAccrued(address indexed referrer, address indexed buyer, uint256 indexed poolId, uint256 stakedAmount, uint256 amount, bytes32 codeHash);
event DirectAccrualWithdrawn(address indexed referrer, uint256 amount, address indexed to);
```

### Reward Commissions
```solidity
event RewardClaimRecorded(address indexed claimant, uint256 indexed poolId, uint256 rewardAmount, bytes32 indexed claimTxHash);
event ReferralPayoutRootSubmitted(uint256 indexed epochId, bytes32 indexed root, address token, uint256 totalAmount, uint64 expiry);
event ReferralPayoutClaimed(uint256 indexed epochId, address indexed claimer, address token, uint256 amount);
event UnclaimedFundsWithdrawn(uint256 indexed epochId, address indexed to, uint256 amount);
```

## Off-Chain Engine Requirements

### Responsibilities
1. **Event Monitoring**: Subscribe to `RewardClaimRecorded` events from ReferralModule
2. **Graph Maintenance**: Build and maintain buyer→referrer relationship graph
3. **Commission Calculation**: 
   - For each claim, traverse referral chain
   - Calculate per-level commissions using `mlBps` configuration
   - Skip levels with no referrer (address(0))
4. **Epoch Aggregation**: 
   - Batch commissions into epochs (daily/weekly)
   - Sum per-beneficiary amounts
5. **Merkle Tree Generation**:
   - Leaf format: `keccak256(abi.encodePacked(beneficiary, token, amount, epochId))`
   - Sort leaves before building tree
6. **Funding & Submission**:
   - Transfer `totalAmount` to ReferralModule
   - Call `submitReferralPayoutRoot()`
7. **Proof API**: Serve Merkle proofs via REST API for users to claim

### Database Schema (Recommended)
```sql
-- Referrer relationships
CREATE TABLE referrer_links (
    buyer ADDRESS PRIMARY KEY,
    referrer ADDRESS,
    code_hash BYTES32,
    linked_at TIMESTAMP
);

-- Purchase tracking
CREATE TABLE purchases (
    tx_hash BYTES32 PRIMARY KEY,
    buyer ADDRESS,
    pool_id UINT256,
    staked_amount UINT256,
    code_hash BYTES32,
    timestamp TIMESTAMP
);

-- Direct commissions
CREATE TABLE direct_commissions (
    id SERIAL PRIMARY KEY,
    referrer ADDRESS,
    buyer ADDRESS,
    pool_id UINT256,
    amount UINT256,
    paid BOOLEAN,
    paid_tx BYTES32,
    timestamp TIMESTAMP
);

-- Reward claims
CREATE TABLE reward_claims (
    claim_tx BYTES32 PRIMARY KEY,
    claimant ADDRESS,
    pool_id UINT256,
    reward_amount UINT256,
    timestamp TIMESTAMP
);

-- Multi-level commissions (calculated)
CREATE TABLE ml_commissions (
    id SERIAL PRIMARY KEY,
    epoch_id UINT256,
    source_claim_tx BYTES32,
    level UINT8,
    beneficiary ADDRESS,
    amount UINT256,
    FOREIGN KEY (source_claim_tx) REFERENCES reward_claims(claim_tx)
);

-- Payout epochs
CREATE TABLE payout_epochs (
    epoch_id UINT256 PRIMARY KEY,
    token ADDRESS,
    total_amount UINT256,
    merkle_root BYTES32,
    submitted_tx BYTES32,
    funded BOOLEAN,
    expiry TIMESTAMP,
    created_at TIMESTAMP
);

-- Claims tracking
CREATE TABLE payout_claims (
    id SERIAL PRIMARY KEY,
    epoch_id UINT256,
    beneficiary ADDRESS,
    amount UINT256,
    proof JSONB,
    claimed_tx BYTES32,
    claimed_at TIMESTAMP,
    UNIQUE(epoch_id, beneficiary)
);
```

### API Endpoints (Recommended)
```
GET  /api/v1/proofs/{epochId}/{beneficiary}
     → { epochId, beneficiary, amount, proof: [...], token }

GET  /api/v1/epochs
     → { epochs: [{ epochId, totalAmount, claimed, expiry, ... }] }

GET  /api/v1/referrer/{address}/stats
     → { totalDirectEarned, totalMLEarned, pendingClaims: [...] }

GET  /api/v1/buyer/{address}/chain
     → { chain: [L1, L2, L3, ...] }

POST /api/v1/calculate-commission
     Body: { rewardAmount, buyerAddress }
     → { levels: [{ level, beneficiary, amount }, ...] }
```

## Integration with PoolManager

### Required Changes to PoolManager

#### 1. Add ReferralModule State Variable
```solidity
IReferralModule public referralModule;

function setReferralModule(address _referralModule) external onlyOwner {
    referralModule = IReferralModule(_referralModule);
}
```

#### 2. Modify buyAndStake to Accept Referral Code
```solidity
function buyAndStake(
    uint256 poolId,
    uint256 maxUsdtAmount,
    uint256 selectedStakeDuration,
    string calldata referralCode  // NEW PARAMETER
) external nonReentrant whenNotPaused {
    // ... existing validation ...

    // Hash referral code
    bytes32 codeHash = keccak256(abi.encodePacked(referralCode));

    // ... existing buy logic: calculate ECM, transfer USDT, update accounting ...

    // BEFORE auto-staking, record referral
    if (address(referralModule) != address(0)) {
        (address referrer, uint256 directAmount) = referralModule.recordPurchaseAndPayDirect(
            codeHash,
            msg.sender,
            poolId,
            ecmToAllocate,
            address(pool.ecm)
        );
        // Optional: emit event with referrer info
    }

    // ... continue with auto-stake logic ...
}
```

#### 3. Emit Reward Claim Events
```solidity
function unstake(uint256 poolId) external nonReentrant {
    // ... existing unstake logic ...

    // After claiming rewards, record event
    if (address(referralModule) != address(0) && pending > 0) {
        bytes32 claimTxHash = keccak256(abi.encodePacked(
            msg.sender, 
            poolId, 
            pending, 
            block.timestamp
        ));
        referralModule.recordRewardClaimEvent(
            msg.sender,
            poolId,
            pending,
            claimTxHash
        );
    }
}

function claimRewards(uint256 poolId) external nonReentrant {
    // ... existing claim logic ...

    // After claiming, record event
    if (address(referralModule) != address(0) && pending > 0) {
        bytes32 claimTxHash = keccak256(abi.encodePacked(
            msg.sender, 
            poolId, 
            pending, 
            block.timestamp
        ));
        referralModule.recordRewardClaimEvent(
            msg.sender,
            poolId,
            pending,
            claimTxHash
        );
    }
}
```

### Funding Strategies

#### Option A: Pre-fund ReferralModule (Recommended)
```solidity
// Admin funds ReferralModule with ECM tokens for direct commissions
// ReferralModule transfers directly to referrers
ECM.approve(referralModule, 100000e18);
ReferralModule.fundContract(ECMAddress, 100000e18);
```

#### Option B: PoolManager Transfers to Referrer
```solidity
// PoolManager calculates commission and transfers from its balance
(address referrer, uint256 directAmount) = referralModule.recordPurchaseAndPayDirect(...);
if (directAmount > 0) {
    pool.ecm.safeTransfer(referrer, directAmount);
}
```

#### Option C: Accrual Mode with Batch Payouts
```solidity
// Use transferOnUse = false
// Periodically admin batch-pays accrued commissions
for (referrer in referrers) {
    uint256 accrued = referralModule.getDirectAccrual(referrer);
    if (accrued > 0) {
        ECM.transfer(referrer, accrued);
        referralModule.withdrawDirectAccrualFor(referrer, accrued, referrer);
    }
}
```

## Security Considerations

### On-Chain Security
1. **Self-Referral Prevention**: Enforced at contract level
2. **Cyclic Prevention**: Checks for 2-person loops
3. **Immutable Relationships**: `referrerOf` set once, prevents gaming
4. **Reentrancy Protection**: All state-changing functions use `nonReentrant`
5. **Access Control**: `onlyOwner` for admin, `onlyPoolManager` for integration
6. **Merkle Proof Verification**: Prevents unauthorized claims
7. **Double Claim Prevention**: `claimedInEpoch` mapping tracks claims
8. **Funding Verification**: `submitReferralPayoutRoot` checks contract balance

### Off-Chain Security
1. **Event Monitoring**: Must be fault-tolerant, handle chain reorgs
2. **Commission Calculation**: Must use exact same rounding as Solidity
3. **Merkle Tree**: Must be deterministic, reproducible
4. **API Security**: Rate limiting, authentication for proof endpoints
5. **Database Integrity**: Foreign keys, transaction consistency
6. **Audit Trail**: Log all calculations for dispute resolution

### Anti-Sybil Measures
1. **KYC Integration**: Off-chain verification before code issuance
2. **Usage Limits**: `maxUses` per code
3. **Time Limits**: `expiry` timestamps
4. **Rate Limiting**: Admin monitors unusual patterns
5. **Referrer Caps**: Optional per-referrer earning limits (off-chain)

## Testing Checklist

### Unit Tests
- ✅ Register referral code with valid params
- ✅ Reject invalid commission rates (> max)
- ✅ Reject self-referral
- ✅ Prevent cyclic referrals
- ✅ Direct commission transfer mode
- ✅ Direct commission accrual mode
- ✅ Merkle proof verification (valid)
- ✅ Merkle proof verification (invalid)
- ✅ Double claim prevention
- ✅ Epoch expiry and withdrawal

### Integration Tests
- ✅ Full flow: buyAndStake with referral code
- ✅ Direct commission paid to referrer
- ✅ Reward claim event emission
- ✅ Multi-level commission calculation (off-chain sim)
- ✅ Merkle root submission and claims
- ✅ Contract funding and balance tracking

### Security Tests
- ✅ Self-referral attempt (should revert)
- ✅ Cyclic referral attempt (should revert)
- ✅ Forge Merkle proof (should revert)
- ✅ Underfunded root submission (should revert)
- ✅ Reentrancy attacks
- ✅ Access control violations

## Example Scenarios

### Scenario 1: Simple Direct Commission
```
Setup:
- UserA creates code "ALICE10" with directBps = 1000 (10%)
- Admin registers: registerReferralCode(keccak256("ALICE10"), userA, 1000, [], 0, 0, true)

Flow:
- UserB stakes 1000 ECM using "ALICE10"
- PoolManager calls recordPurchaseAndPayDirect(hash, userB, 0, 1000e18, ECM)
- ReferralModule:
  ✓ Links: referrerOf[userB] = userA
  ✓ Calculates: 1000 * 1000 / 10000 = 100 ECM
  ✓ Transfers 100 ECM to userA
  ✓ Emits DirectCommissionPaid

Result: UserA instantly receives 100 ECM
```

### Scenario 2: Multi-Level Reward Commission
```
Setup:
- UserA owns code with mlBps = [500, 300, 200] (5%, 3%, 2%)
- UserB referred by UserA (L1)
- UserA referred by UserX (L2)
- UserX referred by UserY (L3)
- Chain: UserB → UserA → UserX → UserY

Flow:
1. UserB claims 1000 ECM rewards
2. PoolManager emits RewardClaimRecorded(userB, 0, 1000e18, txHash)
3. Off-chain engine:
   - Queries chain: [userA, userX, userY]
   - Calculates:
     * userA: 1000 * 500 / 10000 = 50 ECM (L1)
     * userX: 1000 * 300 / 10000 = 30 ECM (L2)
     * userY: 1000 * 200 / 10000 = 20 ECM (L3)
   - Aggregates into Epoch 2025-W43
4. Admin funds 10,000 ECM (many claims aggregated)
5. Admin calls submitReferralPayoutRoot(202543, ECM, 10000e18, root, expiry)
6. UserA claims: claimPayout(202543, ECM, 50e18, proof)
7. UserX claims: claimPayout(202543, ECM, 30e18, proof)
8. UserY claims: claimPayout(202543, ECM, 20e18, proof)

Result: Each receives their multi-level commission from aggregated reward claims
```

### Scenario 3: Accrual Mode with Batch Withdrawal
```
Setup:
- Code registered with transferOnUse = false
- UserA owns code with directBps = 1000

Flow:
1. UserB stakes 500 ECM → directAccrued[userA] += 50 ECM
2. UserC stakes 700 ECM → directAccrued[userA] += 70 ECM
3. UserD stakes 1000 ECM → directAccrued[userA] += 100 ECM
4. Total accrued: 220 ECM
5. Admin periodically calls:
   withdrawDirectAccrualFor(userA, 220e18, userA)
6. Transfers 220 ECM to userA

Result: Batch payment reduces gas costs, admin controls payout timing
```

## Deployment Guide

### Step 1: Deploy ReferralModule
```javascript
const ReferralModule = await ethers.getContractFactory("ReferralModule");
const referralModule = await ReferralModule.deploy();
await referralModule.deployed();
console.log("ReferralModule deployed:", referralModule.address);
```

### Step 2: Configure ReferralModule
```javascript
// Set PoolManager as authorized caller
await referralModule.setPoolManager(poolManager.address);

// Fund contract for direct commissions
await ecmToken.approve(referralModule.address, ethers.utils.parseEther("100000"));
await referralModule.fundContract(ecmToken.address, ethers.utils.parseEther("100000"));
```

### Step 3: Register Referral Codes
```javascript
// Example: 10% direct, 5%-3%-2% multi-level
const codeHash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes("PROMO10"));
await referralModule.registerReferralCode(
    codeHash,
    referrerAddress,
    1000, // 10% direct
    [500, 300, 200], // 5%, 3%, 2% multi-level
    0, // unlimited uses
    0, // no expiry
    true // transfer immediately
);
```

### Step 4: Update PoolManager
```javascript
// Set ReferralModule address in PoolManager
await poolManager.setReferralModule(referralModule.address);
```

### Step 5: Deploy Off-Chain Engine
```bash
# Start event listener
npm run start:referral-engine

# Start API server for proof serving
npm run start:referral-api
```

## Monitoring & Analytics

### Key Metrics
- **Total Direct Commissions Paid**: `totalDirectPaid`
- **Total Multi-Level Commissions Paid**: `totalMultiLevelPaid`
- **Active Referral Codes**: Count of active codes
- **Total Referrer Links**: Count of `referrerOf` mappings
- **Pending Accruals**: Sum of `directAccrued` values
- **Epoch Claim Rate**: `claimed / totalAmount` per epoch

### Dashboard Queries
```javascript
// Total commissions paid
const totalDirect = await referralModule.totalDirectPaid();
const totalML = await referralModule.totalMultiLevelPaid();

// Referrer stats
const referrer = "0x...";
const accrued = await referralModule.getDirectAccrual(referrer);
const chain = await referralModule.getReferralChain(referrer, 10);

// Epoch stats
const epoch = 202543;
const [root, token, total, claimed, funded] = await referralModule.getPayoutRootInfo(epoch);
const claimRate = (claimed * 100) / total;
```

## Troubleshooting

### Common Issues

**Problem**: "InsufficientBalance" error on direct commission
**Solution**: Fund ReferralModule with ECM tokens via `fundContract()`

**Problem**: "CodeNotFound" error
**Solution**: Verify code is registered with exact `keccak256(codeString)` hash

**Problem**: "InvalidProof" error on claim
**Solution**: Ensure Merkle tree was built with sorted leaves, verify leaf format

**Problem**: Off-chain engine not processing claims
**Solution**: Check event listener connection, verify database has claim records

**Problem**: Self-referral preventing purchases
**Solution**: Implement code validation in frontend before transaction

## Conclusion

The ReferralModule provides a robust, gas-efficient two-tier referral system:
- **Direct commissions** provide instant rewards for referrers
- **Multi-level reward commissions** enable network effects with off-chain calculation
- **Merkle proofs** allow cheap, auditable batch distributions
- **Security features** prevent gaming and fraud
- **Flexible configuration** supports various commission structures

Integration with PoolManager is minimal, maintaining separation of concerns while enabling powerful referral marketing capabilities.
