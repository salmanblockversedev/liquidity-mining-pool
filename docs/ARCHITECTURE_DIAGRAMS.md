# ECM Liquidity Mining Pool - Complete Architecture Documentation

## Table of Contents
1. [System Overview](#system-overview)
2. [Five-Contract Architecture](#five-contract-architecture)
3. [Component Details](#component-details)
4. [Data Flow Diagrams](#data-flow-diagrams)
5. [Integration Patterns](#integration-patterns)
6. [Security Architecture](#security-architecture)

---

## System Overview

### High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                   ECM Liquidity Mining Pool Ecosystem                          │
│                                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                    │
│  │   Frontend   │───▶│ PoolManager  │───▶│  Uniswap V2  │                    │
│  │   (Vue.js)   │    │ (Core Logic) │    │  (Pricing)   │                    │
│  └──────────────┘    └──────┬───────┘    └──────────────┘                    │
│                              │                                                  │
│                    ┌─────────┼─────────┐                                       │
│                    │         │         │                                       │
│           ┌────────▼───┐ ┌──▼──────┐ ┌▼──────────────┐                       │
│           │ Referral   │ │ Vesting │ │  Liquidity    │                       │
│           │   System   │ │ Manager │ │   Manager     │                       │
│           └────────────┘ └─────────┘ └───────────────┘                       │
│                │                                                                │
│         ┌──────┴───────┐                                                       │
│         │              │                                                        │
│    ┌────▼─────┐  ┌────▼─────┐                                                │
│    │ Referral │  │ Referral  │                                                │
│    │ Voucher  │  │  Module   │                                                │
│    └──────────┘  └───────────┘                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### Contract Relationships

```
                            ┌────────────────────┐
                            │   Uniswap V2 Pair  │
                            │  (Price Oracle)    │
                            └─────────┬──────────┘
                                      │
                                      │ getReserves()
                                      ▼
┌──────────────┐           ┌──────────────────────┐           ┌────────────────┐
│    User      │──────────▶│   PoolManager        │──────────▶│ ReferralVoucher│
│  (Buyer)     │buyAndStake│   (Core Logic)       │verify     │ (EIP-712)      │
└──────────────┘           └──────────┬───────────┘           └────────────────┘
                                      │
                            ┌─────────┼─────────┐
                            │         │         │
                            ▼         ▼         ▼
                   ┌─────────────┐  ┌──────────────┐  ┌───────────────┐
                   │ Referral    │  │   Vesting    │  │  Liquidity    │
                   │  Module     │  │   Manager    │  │   Manager     │
                   │(Commissions)│  │  (Rewards)   │  │ (Uniswap LP)  │
                   └─────────────┘  └──────────────┘  └───────────────┘
                            │                                  │
                            │ Off-chain                        │
                            ▼                                  ▼
                   ┌─────────────┐              ┌────────────────────────┐
                   │ Merkle Tree │              │ Uniswap V2 Router      │
                   │   Engine    │              │ (Liquidity Operations) │
                   └─────────────┘              └────────────────────────┘
```

---

## Five-Contract Architecture

### Contract Summary

| Contract | Lines | Purpose | Key Features |
|----------|-------|---------|--------------|
| **PoolManager** | 2221 | Token sale, staking, rewards | USDT→ECM conversion, 3 reward strategies, penalties, analytics |
| **ReferralVoucher** | 247 | EIP-712 verification | Signature validation, usage tracking, issuer management |
| **ReferralModule** | 505 | Multi-level commissions | 2-tier system, Merkle distribution, anti-gaming |
| **VestingManager** | 526 | Linear vesting | Pro-rata release, partial claiming, revocation |
| **LiquidityManager** | 409 | Uniswap operations | Isolated LP management, callback tracking |

---

## Component Details

### 1. PoolManager - Central Hub

```
┌──────────────────────────────────────────────────────────────┐
│                       POOLMANAGER                            │
│                    contracts/PoolManager.sol                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY RESPONSIBILITIES:                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  ✓ USDT→ECM token purchases (Uniswap V2 pricing)          │
│  ✓ Auto-staking on purchase (single-call UX)              │
│  ✓ Reward distribution (LINEAR/MONTHLY/WEEKLY)            │
│  ✓ Early unstaking penalties (configurable 0-100%)        │
│  ✓ Referral voucher verification & integration            │
│  ✓ Vesting integration for reward claims                  │
│  ✓ Comprehensive analytics & metrics                      │
│                                                              │
│  CORE STATE VARIABLES:                                      │
│  • mapping(uint256 => Pool) pools                          │
│    - 17 fields per pool (tokens, rates, analytics)        │
│  • mapping(uint256 => mapping(address => UserInfo))       │
│    - 12 fields per user (stakes, rewards, history)        │
│  • IVestingManager vestingManager                         │
│  • IReferralVoucher referralVoucher                       │
│  • IReferralModule referralModule                         │
│  • IUniswapV2Router02 uniswapRouter (immutable)           │
│                                                              │
│  REWARD STRATEGIES:                                         │
│  ┌────────────────────────────────────────────────────┐    │
│  │ LINEAR:   Constant rate/second over duration      │    │
│  │           rewardRatePerSecond × elapsed            │    │
│  │                                                     │    │
│  │ MONTHLY:  Fixed amounts per 30-day period         │    │
│  │           monthlyRewards[index] pro-rata           │    │
│  │                                                     │    │
│  │ WEEKLY:   Fixed amounts per 7-day period          │    │
│  │           weeklyRewards[index] pro-rata            │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  CRITICAL FUNCTIONS:                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ USER OPERATIONS:                                   │    │
│  │ • buyAndStake() - Purchase + auto-stake            │    │
│  │ • buyExactECMAndStake() - Exact amount purchase    │    │
│  │ • unstake() - Withdraw with penalty check          │    │
│  │ • claimRewards() - Claim or vest rewards           │    │
│  │ • pendingRewards() - Real-time reward calculation  │    │
│  │                                                     │    │
│  │ ADMIN OPERATIONS:                                  │    │
│  │ • createPool() - Initialize new pool               │    │
│  │ • allocateForSale/Rewards() - Fund pools           │    │
│  │ • setLinearRewardRate() - Configure LINEAR         │    │
│  │ • setMonthlyRewards() - Configure MONTHLY          │    │
│  │ • setWeeklyRewards() - Configure WEEKLY            │    │
│  │                                                     │    │
│  │ ANALYTICS:                                         │    │
│  │ • calculateAPR() - Strategy-specific APR           │    │
│  │ • calculateExpectedRewards() - Future projection   │    │
│  │ • calculateROI() - Return on investment            │    │
│  │ • calculateTVL() - Total value locked              │    │
│  │ • getPoolAnalytics() - Comprehensive metrics       │    │
│  │ • getUserAnalytics() - Per-user statistics         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  REWARD ACCOUNTING (accRewardPerShare Pattern):            │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 1. accRewardPerShare += (reward × 1e18)/totalStaked│    │
│  │ 2. userPending = (staked × accReward/1e18)-debt   │    │
│  │ 3. userDebt updated on stake/unstake               │    │
│  │                                                     │    │
│  │ Precision: 1e18 scaling prevents rounding errors   │    │
│  │ Canonical: MasterChef-style reward distribution    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  KEY EVENTS:                                                │
│  • BoughtAndStaked(user, ecm, usdt, duration, referrer)   │
│  • Unstaked / EarlyUnstaked(user, principal, slashed)     │
│  • RewardsClaimed / RewardsVested(user, amount)            │
│  • LinearRewardRateSet / MonthlyRewardsSet                 │
│  • PoolCreated / ECMAllocatedForSale                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2. ReferralVoucher - EIP-712 Verification

```
┌──────────────────────────────────────────────────────────────┐
│                    REFERRALVOUCHER                           │
│                contracts/ReferralVoucher.sol                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY RESPONSIBILITIES:                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  ✓ EIP-712 typed signature verification                    │
│  ✓ Usage tracking (single/multi-use vouchers)             │
│  ✓ Expiry enforcement (timestamp validation)              │
│  ✓ Authorized issuer management (whitelist)               │
│  ✓ Voucher revocation (admin emergency stop)              │
│                                                              │
│  VOUCHER STRUCTURE (VoucherInput):                          │
│  ┌────────────────────────────────────────────────────┐    │
│  │ bytes32 vid       - Unique voucher ID              │    │
│  │   = keccak256(codeHash, owner, nonce)              │    │
│  │                                                     │    │
│  │ bytes32 codeHash  - Hash of referral code string   │    │
│  │   = keccak256("PROMO2024")                         │    │
│  │                                                     │    │
│  │ address owner     - Referrer who owns this code    │    │
│  │                                                     │    │
│  │ uint16 directBps  - Direct commission rate (bps)   │    │
│  │   500 = 5%, 1000 = 10%, max 2000 = 20%            │    │
│  │                                                     │    │
│  │ bool transferOnUse - Payment mode                  │    │
│  │   true  = Immediate transfer to referrer           │    │
│  │   false = Accrue for later withdrawal              │    │
│  │                                                     │    │
│  │ uint64 expiry     - Expiration timestamp           │    │
│  │   block.timestamp must be < expiry                 │    │
│  │                                                     │    │
│  │ uint32 maxUses    - Usage limit                    │    │
│  │   0 = unlimited (perfect for referral codes)       │    │
│  │   1 = single-use                                   │    │
│  │   N = N-use limit                                  │    │
│  │                                                     │    │
│  │ uint256 nonce     - Unique nonce for vid           │    │
│  │   Used to generate unique vid per voucher          │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  EIP-712 DOMAIN SEPARATOR:                                  │
│  ┌────────────────────────────────────────────────────┐    │
│  │ name: "ReferralVoucher"                            │    │
│  │ version: "1"                                       │    │
│  │ chainId: Current chain ID                          │    │
│  │ verifyingContract: ReferralVoucher address         │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  TYPE HASH:                                                  │
│  ReferralVoucher(                                           │
│    bytes32 vid,                                             │
│    bytes32 codeHash,                                        │
│    address owner,                                           │
│    uint16 directBps,                                        │
│    bool transferOnUse,                                      │
│    uint64 expiry,                                           │
│    uint32 maxUses,                                          │
│    uint256 nonce                                            │
│  )                                                           │
│                                                              │
│  VERIFICATION FLOW:                                         │
│  1. PoolManager calls verifyAndConsume()                   │
│  2. Check expiry: block.timestamp < voucher.expiry         │
│  3. Check revocation: !voucherRevoked[vid]                 │
│  4. Check usage: voucherUses[vid] < maxUses (if > 0)       │
│  5. Recover signer from EIP-712 signature                  │
│  6. Validate: isIssuer[signer] == true                     │
│  7. Increment: voucherUses[vid]++                          │
│  8. Return: VoucherResult struct                           │
│                                                              │
│  VOUCHER RESULT OUTPUT:                                     │
│  • address owner      - Referrer address                   │
│  • bytes32 codeHash   - Code identifier                    │
│  • uint16 directBps   - Commission rate                    │
│  • bool transferOnUse - Payment mode                       │
│  • uint32 usesRemaining - Remaining uses                   │
│                                                              │
│  SECURITY FEATURES:                                         │
│  ✓ onlyPoolManager modifier prevents direct calls          │
│  ✓ Replay protection via usage tracking                    │
│  ✓ Domain separator prevents cross-chain attacks           │
│  ✓ Issuer whitelist for signature authority                │
│  ✓ Emergency revocation capability                         │
│                                                              │
│  KEY FUNCTIONS:                                             │
│  • verifyAndConsume() - Main verification (PoolManager)    │
│  • addIssuer() / removeIssuer() - Manage signers (Owner)   │
│  • revokeVoucher() - Cancel specific voucher (Owner)       │
│  • setPoolManager() - Set authorized caller (Owner)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 3. ReferralModule - Two-Tier Commissions

```
┌──────────────────────────────────────────────────────────────┐
│                     REFERRALMODULE                           │
│                 contracts/ReferralModule.sol                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  TWO-TIER COMMISSION SYSTEM:                                │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│                                                              │
│  TIER 1: DIRECT COMMISSION (Immediate/Accrued)             │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Trigger: When buyer stakes ECM                     │    │
│  │ Basis:   Staked amount (principal)                 │    │
│  │ Rate:    From voucher directBps (0-2000 bps)       │    │
│  │ Payment: Immediate OR accrued (transferOnUse flag) │    │
│  │                                                     │    │
│  │ Formula: commission = staked × directBps / 10000   │    │
│  │                                                     │    │
│  │ Example: 1000 ECM staked, 500 bps (5%)            │    │
│  │          → 50 ECM direct commission                │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  TIER 2: MULTI-LEVEL REWARD COMMISSION (Merkle)            │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Trigger: When rewards are claimed                  │    │
│  │ Basis:   Reward amount (not principal)             │    │
│  │ Rate:    Pool-level config (per level)             │    │
│  │ Levels:  Up to 10 referral levels                  │    │
│  │ Payment: Merkle proof claims (batch processing)    │    │
│  │                                                     │    │
│  │ Formula: L1 = reward × mlBps[0] / 10000           │    │
│  │          L2 = reward × mlBps[1] / 10000           │    │
│  │          ... up to L10                             │    │
│  │                                                     │    │
│  │ Example: 500 ECM reward, [500,300,200] bps        │    │
│  │          L1: 25 ECM, L2: 15 ECM, L3: 10 ECM       │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  KEY STATE VARIABLES:                                       │
│  • mapping(address => address) referrerOf                  │
│    - Immutable buyer→referrer relationship                 │
│  • mapping(address => uint256) directAccrued               │
│    - Pending direct commissions (if accrued mode)          │
│  • mapping(uint256 => uint16[]) poolLevelConfig            │
│    - Pool-specific ML commission rates [L1...L10]          │
│  • mapping(uint256 => ReferralPayoutRoot) payoutRoots      │
│    - Merkle roots for epoch-based distributions            │
│  • mapping(uint256 => mapping(address => bool)) claimed    │
│    - Tracks claims per epoch per user                      │
│                                                              │
│  ANTI-GAMING RULES:                                         │
│  ✓ No self-referral (buyer ≠ referrer)                     │
│  ✓ No cyclic referrals (2-person loops blocked)           │
│  ✓ Referrer immutable once set                             │
│  ✓ Max 10 referral levels                                  │
│  ✓ Total ML commission ≤ 50% (5000 bps)                    │
│  ✓ Direct commission ≤ 20% (2000 bps)                      │
│                                                              │
│  MERKLE TREE STRUCTURE:                                     │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Leaf = keccak256(abi.encodePacked(                 │    │
│  │   beneficiary,  // Address receiving commission    │    │
│  │   token,        // ECM token address               │    │
│  │   amount,       // Total commission for epoch      │    │
│  │   epochId       // Unique epoch identifier         │    │
│  │ ))                                                  │    │
│  │                                                     │    │
│  │ Tree built with OpenZeppelin MerkleProof           │    │
│  │ Sorted leaves for deterministic root               │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  CRITICAL FUNCTIONS:                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ DIRECT COMMISSION:                                 │    │
│  │ • recordPurchaseAndPayDirect() - Process & pay     │    │
│  │ • withdrawDirectAccrual() - User claims accrued    │    │
│  │ • linkReferrer() - Establish relationship          │    │
│  │                                                     │    │
│  │ MULTI-LEVEL COMMISSION:                            │    │
│  │ • recordRewardClaimEvent() - Emit for off-chain    │    │
│  │ • submitReferralPayoutRoot() - Upload Merkle root  │    │
│  │ • claimReferral() - Claim via Merkle proof         │    │
│  │ • withdrawUnclaimed() - Recover expired funds      │    │
│  │                                                     │    │
│  │ CONFIGURATION:                                     │    │
│  │ • setPoolLevelConfig() - Set ML rates per pool     │    │
│  │ • setPoolManager() - Authorize integration         │    │
│  │ • fundContract() - Deposit ECM for commissions     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  KEY EVENTS:                                                │
│  • ReferrerLinked(buyer, referrer, codeHash)               │
│  • DirectCommissionPaid/Accrued(referrer, amount)          │
│  • RewardClaimRecorded(claimant, poolId, reward)           │
│  • ReferralPayoutRootSubmitted(epochId, root, total)       │
│  • ReferralPayoutClaimed(epochId, claimer, amount)         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4. VestingManager - Linear Vesting

```
┌──────────────────────────────────────────────────────────────┐
│                    VESTINGMANAGER                            │
│                 contracts/VestingManager.sol                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY RESPONSIBILITIES:                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  ✓ Create linear vesting schedules for rewards            │
│  ✓ Pro-rata token release over configured duration        │
│  ✓ Track multiple vesting schedules per user              │
│  ✓ Allow partial claiming of vested tokens                │
│  ✓ Admin revocation support (emergency)                   │
│                                                              │
│  VESTING SCHEDULE STRUCTURE:                                │
│  ┌────────────────────────────────────────────────────┐    │
│  │ address beneficiary  - User receiving tokens       │    │
│  │ address token        - Token being vested (ECM)    │    │
│  │ uint256 totalAmount  - Total vesting amount        │    │
│  │ uint256 start        - Vesting start timestamp     │    │
│  │ uint256 duration     - Vesting period (seconds)    │    │
│  │ uint256 claimed      - Amount already claimed      │    │
│  │ uint256 poolId       - Associated pool ID          │    │
│  │ bool revoked         - Revocation status           │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  VESTING FORMULA (Linear):                                  │
│  ┌────────────────────────────────────────────────────┐    │
│  │ if (now < start)                                   │    │
│  │     return 0  // Not started                       │    │
│  │                                                     │    │
│  │ if (now ≥ start + duration)                        │    │
│  │     return totalAmount  // Fully vested            │    │
│  │                                                     │    │
│  │ else                                               │    │
│  │     elapsed = now - start                          │    │
│  │     return (totalAmount × elapsed) / duration      │    │
│  │     // Pro-rata vesting                            │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  EXAMPLE VESTING TIMELINE:                                  │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Total: 1000 ECM, Duration: 180 days               │    │
│  │                                                     │    │
│  │ Day 0:   0 ECM vested (0%)                         │    │
│  │ Day 45:  250 ECM vested (25%)                      │    │
│  │ Day 90:  500 ECM vested (50%)                      │    │
│  │ Day 135: 750 ECM vested (75%)                      │    │
│  │ Day 180: 1000 ECM vested (100%)                    │    │
│  │                                                     │    │
│  │ User can claim partially at any time:              │    │
│  │ - Claim 200 on Day 45 → 50 still vested           │    │
│  │ - Claim 300 on Day 90 → 250 available (500-250)   │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  KEY STATE VARIABLES:                                       │
│  • mapping(uint256 => VestingSchedule) vestingSchedules    │
│  • mapping(address => uint256[]) userVestingIds            │
│    - All vesting IDs for a user                            │
│  • mapping(address => bool) authorizedCreators             │
│    - Only PoolManager can create vestings                  │
│  • uint256 nextVestingId - Auto-incrementing ID            │
│                                                              │
│  INTEGRATION WITH POOLMANAGER:                              │
│  1. PoolManager: claimRewards() or unstake()               │
│  2. Check: pool.vestRewardsByDefault                       │
│  3. If true: Transfer ECM to VestingManager                │
│  4. VestingManager.createVesting() called                  │
│  5. Returns vestingId                                       │
│  6. User tracks vestingId for future claims                │
│                                                              │
│  CRITICAL FUNCTIONS:                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ USER OPERATIONS:                                   │    │
│  │ • claimVested() - Claim available vested tokens    │    │
│  │ • calculateVested() - View vested amount (view)    │    │
│  │ • getUserVestings() - Get all schedules (view)     │    │
│  │                                                     │    │
│  │ POOLMANAGER OPERATIONS:                            │    │
│  │ • createVesting() - Create new schedule            │    │
│  │   (Only authorized creators)                       │    │
│  │                                                     │    │
│  │ ADMIN OPERATIONS:                                  │    │
│  │ • addAuthorizedCreator() - Authorize PoolManager   │    │
│  │ • revokeVesting() - Emergency stop vesting         │    │
│  │ • emergencyWithdraw() - Recover stuck tokens       │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  AUTHORIZATION MODEL:                                       │
│  ✓ Only authorized creators (PoolManager) can create       │
│  ✓ Only beneficiary can claim their vested tokens          │
│  ✓ Only owner can revoke or emergency withdraw             │
│                                                              │
│  KEY EVENTS:                                                │
│  • VestingCreated(vestingId, beneficiary, amount, duration)│
│  • VestedClaimed(vestingId, beneficiary, amount, remaining)│
│  • VestingRevoked(vestingId, beneficiary, unvested)        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 5. LiquidityManager - Uniswap Operations

```
┌──────────────────────────────────────────────────────────────┐
│                   LIQUIDITYMANAGER                           │
│                contracts/LiquidityManager.sol                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PRIMARY RESPONSIBILITIES:                                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  ✓ Add liquidity to Uniswap V2 pools                       │
│  ✓ Remove liquidity from Uniswap V2 pools                  │
│  ✓ Manage LP tokens (send to treasury)                     │
│  ✓ Callback to PoolManager for tracking                    │
│  ✓ Isolated architecture (limits blast-radius)             │
│                                                              │
│  KEY STATE VARIABLES:                                       │
│  • IUniswapV2Router02 immutable uniswapRouter              │
│    - Set once in constructor, cannot change                │
│  • IPoolManager poolManager                                 │
│    - For recordLiquidityAdded() callbacks                  │
│  • address treasury                                         │
│    - Receives LP tokens from liquidity operations          │
│  • mapping(address => mapping(address => uint256))         │
│    totalLiquidityAdded                                     │
│    - Historical tracking per token pair                    │
│                                                              │
│  ADD LIQUIDITY PARAMS:                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │ address tokenA, tokenB - Token pair                │    │
│  │ uint256 amountADesired - Desired tokenA amount     │    │
│  │ uint256 amountBDesired - Desired tokenB amount     │    │
│  │ uint256 amountAMin     - Min tokenA (slippage)     │    │
│  │ uint256 amountBMin     - Min tokenB (slippage)     │    │
│  │ address to             - LP token recipient        │    │
│  │ uint256 deadline       - Transaction deadline      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  INTEGRATION FLOW WITH POOLMANAGER:                         │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 1. Admin: PoolManager.transferToLiquidityManager() │    │
│  │    - Transfers ECM and USDT to LiquidityManager    │    │
│  │    - Updates pool.liquidityPoolOwedECM             │    │
│  │                                                     │    │
│  │ 2. Admin: LiquidityManager.addLiquidityWithTracking()│  │
│  │    - Approves tokens for Uniswap Router            │    │
│  │    - Calls router.addLiquidity()                   │    │
│  │    - Uniswap mints LP tokens to treasury           │    │
│  │                                                     │    │
│  │ 3. LiquidityManager → PoolManager callback:        │    │
│  │    - Calls poolManager.recordLiquidityAdded()      │    │
│  │    - PoolManager updates tracking:                 │    │
│  │      * pool.ecmAddedToUniswap                      │    │
│  │      * pool.usdtAddedToUniswap                     │    │
│  │                                                     │    │
│  │ 4. LiquidityManager: Emit LiquidityAdded event     │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  CRITICAL FUNCTIONS:                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ • addLiquidity() - Add without tracking            │    │
│  │   (Basic operation, no callback)                   │    │
│  │                                                     │    │
│  │ • addLiquidityWithTracking() - Add + callback      │    │
│  │   (Integrated with PoolManager tracking)           │    │
│  │                                                     │    │
│  │ • removeLiquidity() - Remove from pair             │    │
│  │   (Burns LP tokens, returns underlying)            │    │
│  │                                                     │    │
│  │ • withdrawLPTokens() - Send LP to address          │    │
│  │   (Treasury management)                            │    │
│  │                                                     │    │
│  │ • setTreasury() - Update LP recipient              │    │
│  │ • setPoolManager() - Update callback target        │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  SECURITY FEATURES:                                         │
│  ✓ Owner-only operations (multisig recommended)            │
│  ✓ Slippage protection (amountAMin/amountBMin)             │
│  ✓ Deadline enforcement (transaction expiry)               │
│  ✓ No access to PoolManager internals                      │
│  ✓ Explicit token transfers only (no sweeping)             │
│  ✓ Isolated architecture (separate from staking)           │
│                                                              │
│  WHY ISOLATED ARCHITECTURE?                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 1. Limits blast-radius of potential bugs           │    │
│  │ 2. Clearer separation of concerns                  │    │
│  │ 3. Easier to audit liquidity operations            │    │
│  │ 4. Can upgrade/replace without touching staking    │    │
│  │ 5. Multisig can control liquidity independently    │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  KEY EVENTS:                                                │
│  • LiquidityAdded(poolId, ecm, usdt, lpTokens)             │
│  • LiquidityRemoved(lpAmount, ecmReceived, usdtReceived)   │
│  • LPTokenWithdrawn(token, amount, to)                     │
│  • TreasuryUpdated(oldTreasury, newTreasury)               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagrams

### 1. Purchase & Stake Flow (with Referral)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE PURCHASE & STAKE FLOW                        │
│                      (With Referral Integration)                          │
└──────────────────────────────────────────────────────────────────────────┘

   ┌──────────┐
   │   User   │
   └────┬─────┘
        │
        │ 1. Generate EIP-712 signature (off-chain)
        │    - Sign VoucherInput with referrer's private key
        │    - Contains: codeHash, owner, directBps, etc.
        │
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  buyAndStake(poolId, maxUsdtAmount, duration,            │
   │               voucherInput, signature)                    │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 2. VALIDATION PHASE
        ├──► Check pool.active == true
        ├──► Verify selectedDuration in allowedStakeDurations
        ├──► Check maxUsdtAmount >= minPurchase
        │
        │ 3. PRICE DISCOVERY (Uniswap V2)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  IUniswapV2Pair.getReserves()                            │
   │  → (reserveECM, reserveUSDT, blockTimestamp)             │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 4. ECM CALCULATION
        ├──► ecmRaw = getAmountOut(maxUsdtAmount, reserveUSDT, reserveECM)
        ├──► ecmFloored = (ecmRaw / 500 ether) * 500 ether
        ├──► Require: ecmFloored >= 500 ether
        │
        │ 5. INVERSE USDT CALCULATION
        ├──► usdtRequired = getAmountIn(ecmFloored, reserveUSDT, reserveECM)
        ├──► Require: usdtRequired <= maxUsdtAmount (slippage protection)
        │
        │ 6. REFERRAL VERIFICATION
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ReferralVoucher.verifyAndConsume(voucherInput, sig)     │
   │  → Returns: VoucherResult{owner, codeHash, directBps...} │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 7. USDT TRANSFER
        ├──► USDT.safeTransferFrom(user, PoolManager, usdtRequired)
        │
        │ 8. POOL ACCOUNTING
        ├──► pool.sold += ecmFloored
        ├──► pool.collectedUSDT += usdtRequired
        ├──► user.bought += ecmFloored
        │
        │ 9. AUTO-STAKE
        ├──► _updatePoolRewards(poolId)  // Update accRewardPerShare
        ├──► pool.totalStaked += ecmFloored
        ├──► user.staked += ecmFloored
        ├──► user.stakeStart = block.timestamp
        ├──► user.stakeDuration = selectedDuration
        ├──► user.rewardDebt = (user.staked × accRewardPerShare) / 1e18
        │
        │ 10. DIRECT COMMISSION PAYMENT
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ReferralModule.recordPurchaseAndPayDirect()             │
   │  • commission = ecmFloored × directBps / 10000           │
   │  • if (transferOnUse) → ECM transfer to referrer         │
   │  • else → directAccrued[referrer] += commission          │
   │  • Emits: DirectCommissionPaid/Accrued                   │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 11. LINK REFERRER
        ├──► ReferralModule.linkReferrer(buyer, referrer, codeHash)
        ├──► referrerOf[buyer] = referrer (immutable)
        │
        │ 12. REFUND EXCESS USDT
        ├──► leftover = maxUsdtAmount - usdtRequired
        ├──► if (leftover > 0) USDT.safeTransfer(user, leftover)
        │
        │ 13. EMIT EVENTS
        ├──► BoughtAndStaked(user, poolId, ecmFloored, usdtRequired, duration)
        ├──► ReferrerLinked(buyer, referrer, codeHash)
        │
        ▼
   ┌──────────┐
   │ Complete │
   └──────────┘
```

### 2. Reward Claim Flow (with Vesting & Multi-Level Commissions)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        REWARD CLAIM FLOW                                  │
│               (With Vesting & Multi-Level Commission)                     │
└──────────────────────────────────────────────────────────────────────────┘

   ┌──────────┐
   │   User   │
   └────┬─────┘
        │
        │ 1. Call claimRewards(poolId)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  PoolManager.claimRewards(poolId)                        │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 2. UPDATE POOL REWARDS
        ├──► _updatePoolRewards(poolId)
        │    └─► Calculate elapsed time since lastRewardTime
        │    └─► If LINEAR: reward = elapsed × rewardRatePerSecond
        │    └─► If MONTHLY: reward = sum of completed months
        │    └─► accRewardPerShare += (reward × 1e18) / totalStaked
        │    └─► lastRewardTime = block.timestamp
        │
        │ 3. CALCULATE PENDING REWARDS
        ├──► pending = (user.staked × accRewardPerShare / 1e18) - user.rewardDebt
        ├──► Require: pending > 0
        │
        │ 4. UPDATE USER STATE
        ├──► user.rewardDebt = (user.staked × accRewardPerShare) / 1e18
        ├──► user.totalRewardsClaimed += pending
        │
        │ 5. DECIDE: VEST OR DIRECT TRANSFER
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Is pool.vestRewardsByDefault == true?                   │
   └────┬──────────────────────────────────────────┬──────────┘
        │ YES                                       │ NO
        │                                           │
        ▼                                           ▼
   ┌─────────────────────────────────┐   ┌──────────────────┐
   │  VESTING PATH                   │   │  DIRECT PATH     │
   └────┬────────────────────────────┘   └────┬─────────────┘
        │                                       │
        │ 6a. Transfer to VestingManager        │ 6b. Transfer to user
        ├──► ECM.safeTransfer(                  ├──► ECM.safeTransfer(
        │      vestingManager,                  │      user, pending)
        │      pending)                         │
        │                                       │ 7b. Emit event
        │ 7a. Create vesting schedule           ├──► RewardsClaimed(
        ▼                                       │      user, poolId, pending)
   ┌─────────────────────────────────┐        │
   │  VestingManager.createVesting() │        │
   │  • beneficiary = user            │        │
   │  • totalAmount = pending         │        │
   │  • start = block.timestamp       │        │
   │  • duration = pool.vestingDuration│       │
   │  • Returns: vestingId            │        │
   └────┬────────────────────────────┘        │
        │                                       │
        │ 8a. Emit event                        │
        ├──► RewardsVested(                     │
        │      user, poolId, pending,           │
        │      vestingId)                       │
        │                                       │
        └───────────────────┬───────────────────┘
                            │
        ┌───────────────────┴───────────────────┐
        │ 9. RECORD FOR MULTI-LEVEL COMMISSIONS │
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ReferralModule.recordRewardClaimEvent(                  │
   │    claimant = user,                                      │
   │    poolId,                                               │
   │    rewardAmount = pending                                │
   │  )                                                        │
   │  • Emits: RewardClaimRecorded(user, poolId, pending)     │
   │  • Off-chain: Monitor event for commission calculation   │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 10. OFF-CHAIN PROCESS (Backend calculates commissions)
        │
        ├──► Walk referral chain: user → referrer → referrer2 → ...
        ├──► For each level (up to 10):
        │    └─► commission[level] = pending × mlBps[level] / 10000
        │
        ├──► Build Merkle tree with all commissions
        │    └─► Leaf = keccak256(beneficiary, token, amount, epochId)
        │
        ├──► Generate Merkle proof for each beneficiary
        │
        │ 11. ADMIN SUBMITS MERKLE ROOT
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ReferralModule.submitReferralPayoutRoot(                │
   │    epochId,                                              │
   │    token = ECM address,                                  │
   │    totalAmount = sum of all commissions,                 │
   │    merkleRoot,                                           │
   │    expiry = block.timestamp + 30 days                    │
   │  )                                                        │
   │  • ECM.safeTransferFrom(admin, ReferralModule, totalAmount)│
   │  • Emits: ReferralPayoutRootSubmitted                    │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 12. BENEFICIARIES CLAIM COMMISSIONS (users with proof)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ReferralModule.claimReferral(                           │
   │    epochId,                                              │
   │    token = ECM,                                          │
   │    amount = their commission,                            │
   │    proof = Merkle proof array                            │
   │  )                                                        │
   │  • Verify proof against root                             │
   │  • Check: !claimedInEpoch[epochId][msg.sender]          │
   │  • ECM.safeTransfer(msg.sender, amount)                  │
   │  • claimedInEpoch[epochId][msg.sender] = true            │
   │  • Emits: ReferralPayoutClaimed                          │
   └──────────────────────────────────────────────────────────┘
```

### 3. Unstake Flow (with Early Penalty & Vesting)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          UNSTAKE FLOW                                     │
│                  (With Early Penalty & Rewards)                           │
└──────────────────────────────────────────────────────────────────────────┘

   ┌──────────┐
   │   User   │
   └────┬─────┘
        │
        │ 1. Call unstake(poolId)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  PoolManager.unstake(poolId)                             │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 2. UPDATE POOL REWARDS
        ├──► _updatePoolRewards(poolId)
        │
        │ 3. CALCULATE PENDING REWARDS
        ├──► pending = (user.staked × accRewardPerShare / 1e18) - user.rewardDebt
        │
        │ 4. CHECK IF EARLY UNSTAKE
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Is block.timestamp < user.stakeStart + user.stakeDuration?│
   └────┬──────────────────────────────────────────┬──────────┘
        │ YES (Early)                               │ NO (Mature)
        │                                           │
        ▼                                           ▼
   ┌─────────────────────────────────┐   ┌──────────────────┐
   │  EARLY UNSTAKE PATH             │   │  MATURE PATH     │
   └────┬────────────────────────────┘   └────┬─────────────┘
        │                                       │
        │ 5a. Calculate penalty                 │ 5b. No penalty
        ├──► slashed = user.staked ×           ├──► slashed = 0
        │      pool.penaltyBps / 10000         ├──► remaining = user.staked
        ├──► remaining = user.staked -         │
        │      slashed                         │
        │                                       │
        │ 6a. Transfer slashed to receiver      │
        ├──► ECM.safeTransfer(                  │
        │      pool.penaltyReceiver,            │
        │      slashed)                         │
        │                                       │
        │ 7a. Emit early unstake event          │ 7b. Emit unstake event
        ├──► EarlyUnstaked(user,               ├──► Unstaked(user,
        │      poolId, remaining,               │      poolId, remaining)
        │      slashed)                         │
        │                                       │
        └───────────────────┬───────────────────┘
                            │
        ┌───────────────────┴───────────────────┐
        │ 8. UPDATE POOL & USER STATE           │
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  • pool.totalStaked -= user.staked                       │
   │  • user.staked = 0                                       │
   │  • user.rewardDebt = 0                                   │
   │  • user.stakeStart = 0                                   │
   │  • user.stakeDuration = 0                                │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 9. TRANSFER PRINCIPAL (minus penalty if early)
        ├──► ECM.safeTransfer(user, remaining)
        │
        │ 10. HANDLE REWARDS (same as claimRewards flow)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  if (pending > 0)                                        │
   │    if (vestRewardsByDefault)                             │
   │      → Transfer to VestingManager                        │
   │      → Create vesting schedule                           │
   │    else                                                  │
   │      → ECM.safeTransfer(user, pending)                   │
   │                                                          │
   │    → ReferralModule.recordRewardClaimEvent()             │
   │      (For multi-level commission processing)             │
   └──────────────────────────────────────────────────────────┘
```

### 4. Liquidity Management Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      LIQUIDITY MANAGEMENT FLOW                            │
│                 (PoolManager → LiquidityManager → Uniswap)                │
└──────────────────────────────────────────────────────────────────────────┘

   ┌──────────┐
   │  Admin   │
   └────┬─────┘
        │
        │ 1. Decide liquidity amounts from pool allocations
        │    Example: 100,000 ECM + 50,000 USDT
        │
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  PoolManager.transferToLiquidityManager(                 │
   │    ecmAmount = 100000e18,                                │
   │    usdtAmount = 50000e6,                                 │
   │    liquidityManagerAddress                               │
   │  )                                                        │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 2. VALIDATION
        ├──► Check: ecmAmount <= unallocated pool balance
        ├──► Check: usdtAmount <= pool.collectedUSDT
        │
        │ 3. UPDATE POOL TRACKING
        ├──► pool.liquidityPoolOwedECM += ecmAmount
        │
        │ 4. TRANSFER TOKENS
        ├──► ECM.safeTransfer(liquidityManager, ecmAmount)
        ├──► USDT.safeTransfer(liquidityManager, usdtAmount)
        │
        │ 5. EMIT EVENT
        ├──► LiquidityTransferred(liquidityManager, ecmAmount, usdtAmount)
        │
        ▼
   ┌──────────┐
   │  Admin   │  (Now interacts with LiquidityManager)
   └────┬─────┘
        │
        │ 6. Add liquidity with tracking
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  LiquidityManager.addLiquidityWithTracking(              │
   │    AddLiquidityParams{                                   │
   │      tokenA: ECM,                                        │
   │      tokenB: USDT,                                       │
   │      amountADesired: 100000e18,                          │
   │      amountBDesired: 50000e6,                            │
   │      amountAMin: 95000e18,  // 5% slippage               │
   │      amountBMin: 47500e6,   // 5% slippage               │
   │      to: treasury,                                       │
   │      deadline: block.timestamp + 300                     │
   │    },                                                    │
   │    poolId,                                               │
   │    ECM address                                           │
   │  )                                                        │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 7. APPROVE TOKENS
        ├──► ECM.approve(uniswapRouter, 100000e18)
        ├──► USDT.approve(uniswapRouter, 50000e6)
        │
        │ 8. ADD LIQUIDITY TO UNISWAP V2
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  IUniswapV2Router02.addLiquidity(                        │
   │    tokenA, tokenB,                                       │
   │    amountADesired, amountBDesired,                       │
   │    amountAMin, amountBMin,                               │
   │    to = treasury,                                        │
   │    deadline                                              │
   │  )                                                        │
   │  → Returns: (amountA, amountB, liquidity)                │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 9. CALLBACK TO POOLMANAGER
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  PoolManager.recordLiquidityAdded(                       │
   │    poolId,                                               │
   │    ecmAmount = amountA,                                  │
   │    usdtAmount = amountB                                  │
   │  )                                                        │
   │  • pool.ecmAddedToUniswap += ecmAmount                   │
   │  • pool.usdtAddedToUniswap += usdtAmount                 │
   │  • Emits: LiquidityRecorded(poolId, ecmAmount, usdtAmount)│
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 10. UPDATE LIQUIDITYMANAGER STATE
        ├──► totalLiquidityAdded[ECM][USDT] += ecmAmount + usdtAmount
        │
        │ 11. EMIT EVENT
        ├──► LiquidityAdded(poolId, amountA, amountB, liquidity)
        │
        │ 12. LP TOKENS SENT TO TREASURY
        │     (Uniswap automatically sent LP tokens to `to` address)
        │
        ▼
   ┌──────────┐
   │ Complete │
   └──────────┘

   NOTE: LP tokens held by treasury can later be:
   • Burned (permanent liquidity lock)
   • Held (treasury retains value)
   • Withdrawn via LiquidityManager.withdrawLPTokens()
```

### 5. Vesting Claim Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         VESTING CLAIM FLOW                                │
│                   (Periodic Token Release)                                │
└──────────────────────────────────────────────────────────────────────────┘

   ┌──────────┐
   │   User   │  (Has vestingId from previous reward claim)
   └────┬─────┘
        │
        │ 1. Check vested amount (view function)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  VestingManager.calculateVested(vestingId)               │
   │  → Returns: vestedAmount                                 │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ Formula:
        │ • if (now < start) → 0
        │ • if (now ≥ start + duration) → totalAmount
        │ • else → (totalAmount × (now - start)) / duration
        │
        │ 2. Decide to claim
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  VestingManager.claimVested(vestingId)                   │
   └────┬─────────────────────────────────────────────────────┘
        │
        │ 3. LOAD VESTING SCHEDULE
        ├──► schedule = vestingSchedules[vestingId]
        │
        │ 4. VALIDATE
        ├──► Require: msg.sender == schedule.beneficiary
        ├──► Require: !schedule.revoked
        │
        │ 5. CALCULATE CLAIMABLE
        ├──► vested = calculateVested(vestingId)
        ├──► claimable = vested - schedule.claimed
        ├──► Require: claimable > 0
        │
        │ 6. UPDATE STATE
        ├──► schedule.claimed += claimable
        │
        │ 7. TRANSFER TOKENS
        ├──► ECM.safeTransfer(msg.sender, claimable)
        │
        │ 8. EMIT EVENT
        ├──► VestedClaimed(vestingId, msg.sender, claimable,
        │                   totalAmount - schedule.claimed)
        │
        ▼
   ┌──────────┐
   │ Complete │
   └──────────┘

   EXAMPLE TIMELINE:
   ┌────────────────────────────────────────────────────────┐
   │ Vesting Schedule:                                      │
   │   Total: 1000 ECM                                      │
   │   Duration: 100 days                                   │
   │   Start: Day 0                                         │
   │                                                        │
   │ Day 0:  claimVested() → 0 ECM (not started)           │
   │ Day 25: claimVested() → 250 ECM (25% vested)          │
   │ Day 50: claimVested() → 250 ECM (500-250 already claimed)│
   │ Day 75: claimVested() → 250 ECM (750-500 already claimed)│
   │ Day 100: claimVested() → 250 ECM (1000-750 = fully vested)│
   │                                                        │
   │ User can claim multiple times, accumulates linearly    │
   └────────────────────────────────────────────────────────┘
```

---

## Detailed Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            ON-CHAIN LAYER                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                        PoolManager                              │    │
│  │                                                                 │    │
│  │  • buyAndStake(poolId, usdt, duration, referralCode)           │    │
│  │  • buyExactECMAndStake(poolId, ecm, usdt, duration, refCode)   │    │
│  │  • unstake(poolId) → emits RewardClaimRecorded                 │    │
│  │  • claimRewards(poolId) → emits RewardClaimRecorded            │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                               │                                          │
│                               │ Integration                              │
│                               ▼                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     ReferralModule                              │    │
│  │                                                                 │    │
│  │  Storage:                                                       │    │
│  │  • mapping(bytes32 => ReferralCode) referralCodes              │    │
│  │  • mapping(address => address) referrerOf                      │    │
│  │  • mapping(address => uint256) directAccrued                   │    │
│  │  • mapping(uint256 => ReferralPayoutRoot) payoutRoots          │    │
│  │  • mapping(uint256 => mapping(address => bool)) claimedInEpoch │    │
│  │                                                                 │    │
│  │  Direct Commission Functions:                                  │    │
│  │  • recordPurchaseAndPayDirect() → instant or accrue            │    │
│  │  • withdrawDirectAccrual() → user claims accrued               │    │
│  │                                                                 │    │
│  │  Multi-Level Commission Functions:                             │    │
│  │  • recordRewardClaimEvent() → emit for off-chain               │    │
│  │  • submitReferralPayoutRoot() → Merkle root upload             │    │
│  │  • claimPayout() → verify proof and transfer                   │    │
│  │                                                                 │    │
│  │  Admin Functions:                                              │    │
│  │  • registerReferralCode() → create new code                    │    │
│  │  • revokeReferralCode() → deactivate code                      │    │
│  │  • fundContract() → deposit ECM for commissions                │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          OFF-CHAIN LAYER                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                      Event Listener                             │    │
│  │                                                                 │    │
│  │  Subscribe to:                                                  │    │
│  │  • ReferrerLinked                                              │    │
│  │  • DirectCommissionPaid/Accrued                                │    │
│  │  • RewardClaimRecorded  ◄─────────┐                           │    │
│  │  • ReferralPayoutClaimed           │                           │    │
│  │                                    │                           │    │
│  └────────────────────────────────────┼───────────────────────────┘    │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                   Commission Calculator                         │    │
│  │                                                                 │    │
│  │  For each RewardClaimRecorded event:                           │    │
│  │  1. Query referrerOf[buyer] chain                              │    │
│  │  2. Get referral code's mlBps configuration                    │    │
│  │  3. Calculate per-level commissions:                           │    │
│  │     L1: rewardAmount × mlBps[0] / 10000                        │    │
│  │     L2: rewardAmount × mlBps[1] / 10000                        │    │
│  │     L3: rewardAmount × mlBps[2] / 10000                        │    │
│  │  4. Store in database                                          │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     Epoch Aggregator                            │    │
│  │                                                                 │    │
│  │  • Batch commissions into epochs (daily/weekly)                │    │
│  │  • Sum per-beneficiary amounts                                 │    │
│  │  • Calculate totalAmount for epoch                             │    │
│  │  • Trigger Merkle tree generation                              │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                  Merkle Tree Generator                          │    │
│  │                                                                 │    │
│  │  • Create leaves: keccak256(beneficiary, token, amount, epoch) │    │
│  │  • Sort leaves for consistency                                 │    │
│  │  • Build Merkle tree                                           │    │
│  │  • Generate root hash                                          │    │
│  │  • Store proofs for each beneficiary                           │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    Funding & Submission                         │    │
│  │                                                                 │    │
│  │  1. Transfer totalAmount ECM to ReferralModule                 │    │
│  │  2. Call submitReferralPayoutRoot(epochId, token, total, root) │    │
│  │  3. Store transaction hash                                     │    │
│  │  4. Mark epoch as funded                                       │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                       │                                  │
│                                       ▼                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                      REST API Server                            │    │
│  │                                                                 │    │
│  │  Endpoints:                                                     │    │
│  │  GET  /proofs/:epochId/:beneficiary → { proof, amount }        │    │
│  │  GET  /epochs → list of all epochs                             │    │
│  │  GET  /referrer/:address/stats → earnings summary              │    │
│  │  GET  /buyer/:address/chain → referral chain                   │    │
│  │  POST /calculate → preview commission                          │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         DATABASE LAYER                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  referrer_links          purchases              direct_commissions      │
│  ┌──────────────┐        ┌──────────────┐      ┌──────────────┐        │
│  │ buyer        │        │ tx_hash      │      │ id           │        │
│  │ referrer     │        │ buyer        │      │ referrer     │        │
│  │ code_hash    │        │ pool_id      │      │ buyer        │        │
│  │ linked_at    │        │ staked_amt   │      │ amount       │        │
│  └──────────────┘        │ code_hash    │      │ paid         │        │
│                          │ timestamp    │      └──────────────┘        │
│                          └──────────────┘                               │
│                                                                          │
│  reward_claims           ml_commissions        payout_epochs            │
│  ┌──────────────┐        ┌──────────────┐      ┌──────────────┐        │
│  │ claim_tx     │        │ id           │      │ epoch_id     │        │
│  │ claimant     │        │ epoch_id     │      │ token        │        │
│  │ pool_id      │        │ source_tx    │      │ total_amount │        │
│  │ reward_amt   │        │ level        │      │ merkle_root  │        │
│  │ timestamp    │        │ beneficiary  │      │ funded       │        │
│  └──────────────┘        │ amount       │      │ expiry       │        │
│                          └──────────────┘      └──────────────┘        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Data Flow: Purchase with Direct Commission

```
┌───────┐                                                    ┌───────┐
│ USER  │                                                    │ ADMIN │
│(Buyer)│                                                    │       │
└───┬───┘                                                    └───┬───┘
    │                                                            │
    │ 1. Fund ReferralModule with ECM                            │
    │    ECM.approve(ref, 100000e18)                             │
    │    ref.fundContract(ecm, 100000e18) ◄──────────────────────┤
    │                                                            │
    │ 2. Register referral code                                  │
    │    ref.registerReferralCode(                               │
    │      hash("ALICE10"),                                      │
    │      userA,                                                │
    │      1000,  // 10% direct                                  │
    │      [500,300,200],  // 5%,3%,2% ML                        │
    │      0, 0, true                                            │
    │    ) ◄─────────────────────────────────────────────────────┤
    │                                                            │
    ▼
┌───────────────────────────────────────────────────────────────┐
│                     USER BUYS & STAKES                        │
└───────────────────────────────────────────────────────────────┘
    │
    │ 3. buyAndStake(poolId, 1000 USDT, 90 days, "ALICE10")
    ▼
┌──────────────────────────────────────────────────────────────────┐
│                         PoolManager                              │
│                                                                  │
│  • Validate parameters                                          │
│  • Calculate ECM: 1000 ECM                                      │
│  • Transfer USDT from user                                      │
│  • Update pool accounting                                       │
│  • Hash code: keccak256("ALICE10")                             │
│  •─────────────────────────────────────────────┐               │
│                                                  │               │
└──────────────────────────────────────────────────┼───────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                       ReferralModule                             │
│  recordPurchaseAndPayDirect(hash, buyer, 0, 1000e18, ecm)       │
│                                                                  │
│  1. Validate code (active, not expired)                         │
│  2. Prevent self-referral: buyer ≠ userA ✓                      │
│  3. Prevent cycle: referrerOf[userA] ≠ buyer ✓                  │
│  4. Link: referrerOf[buyer] = userA                             │
│  5. Calculate: 1000 × 1000 / 10000 = 100 ECM                    │
│  6. Transfer: ECM.transfer(userA, 100 ECM) ✓                    │
│  7. Emit: DirectCommissionPaid(userA, buyer, 0, 1000, 100)      │
│  8. Return: (userA, 100)                                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                         PoolManager                              │
│                                                                  │
│  • Continue auto-stake flow                                     │
│  • Update user.staked += 1000 ECM                               │
│  • Emit: BoughtAndStaked(0, buyer, 1000, usdtPaid, 90days)      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
                                               ┌───────┐
                                               │ USER A│
                                               │Receives│
                                               │100 ECM│
                                               └───────┘
```

## Data Flow: Reward Claim with Multi-Level Commission

```
┌───────┐                                                    
│ USER  │                                                    
│(Buyer)│                                                    
└───┬───┘                                                    
    │
    │ User claims 500 ECM rewards
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│                         PoolManager                              │
│  unstake(poolId) or claimRewards(poolId)                         │
│                                                                  │
│  • Calculate pending: 500 ECM                                   │
│  • Transfer 500 ECM to user                                     │
│  • Generate claimTxHash                                         │
│  •─────────────────────────────────────────────┐               │
│                                                  │               │
└──────────────────────────────────────────────────┼───────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                       ReferralModule                             │
│  recordRewardClaimEvent(buyer, 0, 500e18, txHash)                │
│                                                                  │
│  • Emit: RewardClaimRecorded(buyer, 0, 500e18, txHash)          │
│  • No on-chain calculation (gas savings!)                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   │ Event emitted
                                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Off-Chain Engine                           │
│                                                                  │
│  1. Listen to RewardClaimRecorded event                         │
│  2. Query referral chain:                                       │
│     referrerOf[buyer] = userA (L1)                              │
│     referrerOf[userA] = userX (L2)                              │
│     referrerOf[userX] = userY (L3)                              │
│                                                                  │
│  3. Get code config: mlBps = [500, 300, 200]                    │
│                                                                  │
│  4. Calculate commissions:                                      │
│     userA (L1): 500 × 500 / 10000 = 25 ECM                      │
│     userX (L2): 500 × 300 / 10000 = 15 ECM                      │
│     userY (L3): 500 × 200 / 10000 = 10 ECM                      │
│                                                                  │
│  5. Store in database (ml_commissions table)                    │
│                                                                  │
│  6. Wait for epoch batch window (e.g., 7 days)                  │
│                                                                  │
│  7. Aggregate all commissions:                                  │
│     Epoch 202543: 1000 claims × avg 50 ECM = 50,000 ECM         │
│                                                                  │
│  8. Build Merkle tree:                                          │
│     Leaf₁ = keccak256(userA, ecm, 2500, 202543)  // many claims │
│     Leaf₂ = keccak256(userX, ecm, 1500, 202543)                 │
│     Leaf₃ = keccak256(userY, ecm, 1000, 202543)                 │
│     ... hundreds more leaves ...                                │
│                                                                  │
│  9. Generate Merkle root                                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                           Admin                                  │
│                                                                  │
│  1. Transfer 50,000 ECM to ReferralModule                       │
│     ECM.transfer(ref, 50000e18)                                 │
│                                                                  │
│  2. Submit Merkle root                                          │
│     ref.submitReferralPayoutRoot(                               │
│       202543,                    // epochId                     │
│       ecm,                       // token                       │
│       50000e18,                  // totalAmount                 │
│       merkleRoot,                // root hash                   │
│       expiry                     // 30 days from now            │
│     )                                                           │
│                                                                  │
│  3. Contract verifies balance ≥ 50000 ECM ✓                     │
│  4. Stores root and marks funded                                │
│  5. Emits: ReferralPayoutRootSubmitted(202543, root, ...)       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                          User A Claims                           │
│                                                                  │
│  1. Fetch proof from API:                                       │
│     GET /api/proofs/202543/userA                                │
│     → { amount: "2500000000000000000000", proof: [...] }        │
│                                                                  │
│  2. Call contract:                                              │
│     ref.claimPayout(                                            │
│       202543,                    // epochId                     │
│       ecm,                       // token                       │
│       2500e18,                   // amount (2500 ECM)           │
│       proof                      // Merkle proof array          │
│     )                                                           │
│                                                                  │
│  3. ReferralModule verifies:                                    │
│     • Root exists and funded ✓                                  │
│     • Not already claimed ✓                                     │
│     • Merkle proof valid ✓                                      │
│                                                                  │
│  4. Transfer: ECM.transfer(userA, 2500e18) ✓                    │
│  5. Mark: claimedInEpoch[202543][userA] = true                  │
│  6. Update: root.claimed += 2500e18                             │
│  7. Emit: ReferralPayoutClaimed(202543, userA, ecm, 2500)       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                                   │
                                                   ▼
                                               ┌───────┐
                                               │ USER A│
                                               │Receives│
                                               │2500ECM│
                                               └───────┘

┌──────────────────────────────────────────────────────────────────┐
│                    UserX and UserY repeat steps 1-2              │
│                    to claim their shares (1500, 1000 ECM)        │
└──────────────────────────────────────────────────────────────────┘
```

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     SECURITY LAYERS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: Smart Contract Security                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • ReentrancyGuard on all state-changing functions        │  │
│  │ • SafeERC20 for all token transfers                      │  │
│  │ • Custom errors (gas-efficient)                          │  │
│  │ • Explicit function visibility                           │  │
│  │ • No unchecked external calls                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Layer 2: Access Control                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • onlyOwner: Admin functions (register codes, roots)     │  │
│  │ • onlyPoolManager: Integration hooks (record purchase)   │  │
│  │ • Public: User claims (permissionless with proof)        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Layer 3: Anti-Gaming                                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • Self-referral prevention: buyer ≠ referrer             │  │
│  │ • Cyclic prevention: referrerOf[ref] ≠ buyer             │  │
│  │ • Immutable relationships: referrerOf set once           │  │
│  │ • Code usage limits: maxUses enforcement                 │  │
│  │ • Code expiry: timestamp validation                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Layer 4: Financial Security                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • Balance verification before transfers                  │  │
│  │ • Double claim prevention (claimedInEpoch mapping)       │  │
│  │ • Merkle proof verification                              │  │
│  │ • Commission rate caps (20% direct, 50% ML total)        │  │
│  │ • Funded root requirement before claims                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Layer 5: Off-Chain Security                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ • Event listener fault tolerance                         │  │
│  │ • Database transaction consistency                       │  │
│  │ • API rate limiting                                      │  │
│  │ • Merkle tree deterministic generation                   │  │
│  │ • Audit trail for all calculations                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Commission Rate Boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMMISSION CONSTRAINTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Direct Commission (Staked Amount Based)                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  MIN: 0 bps (0%)                                         │  │
│  │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │  │
│  │                                                           │  │
│  │  TYPICAL: 500-1500 bps (5%-15%)                          │  │
│  │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  │  │
│  │                                                           │  │
│  │  MAX: 2000 bps (20%) ◄── Hard limit                      │  │
│  │  ████████████████████████████████████████████████████████  │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Multi-Level Commission (Reward Amount Based)                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  Per Level: No individual limit                          │  │
│  │  Total Sum: MAX 5000 bps (50%) ◄── Hard limit            │  │
│  │                                                           │  │
│  │  Example Valid Configurations:                           │  │
│  │  • [500, 300, 200] = 10% total ✓                         │  │
│  │  • [1000, 800, 600, 400, 200] = 30% total ✓              │  │
│  │  • [2000, 1500, 1000, 500] = 50% total ✓                 │  │
│  │  • [3000, 3000] = 60% total ✗ (exceeds limit)            │  │
│  │                                                           │  │
│  │  Level Depth: MAX 10 levels ◄── Hard limit               │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Gas Comparison: On-Chain vs. Off-Chain Multi-Level

```
┌─────────────────────────────────────────────────────────────────┐
│               GAS COST COMPARISON                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Scenario: 10-level referral chain, 100 reward claims           │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ON-CHAIN CALCULATION (Hypothetical)                     │  │
│  │                                                           │  │
│  │  Per claim:                                              │  │
│  │  • Traverse 10 levels: ~20k gas                          │  │
│  │  • Calculate 10 commissions: ~10k gas                    │  │
│  │  • Transfer to 10 beneficiaries: ~21k × 10 = 210k gas    │  │
│  │  • Total per claim: ~240k gas                            │  │
│  │                                                           │  │
│  │  For 100 claims: 240k × 100 = 24,000,000 gas             │  │
│  │  At 50 gwei: ~$240 (assuming $2000 ETH)                  │  │
│  │                                                           │  │
│  │  ████████████████████████████████████████████████████████  │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  OFF-CHAIN CALCULATION + MERKLE (Actual)                 │  │
│  │                                                           │  │
│  │  Per claim:                                              │  │
│  │  • Emit event: ~30k gas                                  │  │
│  │                                                           │  │
│  │  Per epoch (once for 100 claims):                        │  │
│  │  • Submit Merkle root: ~70k gas                          │  │
│  │                                                           │  │
│  │  Per beneficiary claim:                                  │  │
│  │  • Verify proof + transfer: ~50k gas                     │  │
│  │                                                           │  │
│  │  Total: (30k × 100) + 70k + (50k × 100) = 8,070k gas     │  │
│  │  At 50 gwei: ~$80 (assuming $2000 ETH)                   │  │
│  │                                                           │  │
│  │  ████████                                                 │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  💰 SAVINGS: 66% gas reduction                                   │
│  💰 COST SAVINGS: ~$160 per 100 claims                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Notes

- All diagrams represent production-ready implementation
- Security features are enforced at multiple layers
- Off-chain calculation provides massive gas savings
- System designed for scalability (thousands of referrers)
- Integration with PoolManager is minimal and non-invasive
