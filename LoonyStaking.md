# LoonyStaking Smart Contract Integration Guide
`LoonyStaking.sol`

Welcome to the integration guide for the **LoonyStaking** smart contract. This document provides clear, actionable instructions, data structures, methods, events, and code snippets designed specifically for both **Frontend** and **Backend** developers to interact with the staking pools ecosystem.

---

##  Architecture & Design Overview

The `LoonyStaking` contract manages dynamic staking pools of `$LOONY` tokens. It incentivizes long-term holding while providing high-yield rewards powered by early-exit penalties.

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                        LoonyStaking                             │
  │                                                                 │
  │  Admin creates Pools with dynamic config:                       │
  │    - maturityPeriod (seconds)                                   │
  │    - tierPeriods[] + tierPenalties[] (early exit penalty tiers)  │
  │    - APY (annual percentage yield)                              │
  │                                                                 │
  │  Reward Model (Maturity All-or-Nothing + Penalty Yield):        │
  │    timeReward = (amount × APY / 100) ONLY IF MATURED            │
  │    penaltyReward = amount × accPenaltyPerShare / 1e18 - debt    │
  │                                                                 │
  │  Penalty Model (tier-based):                                    │
  │    If unstake before maturity:                                  │
  │      penalty = amount × tierPenalty / 100                       │
  │      50% burned  →  contract burn address                       │
  │      50% redistributed → accPenaltyPerShare ↑                   │
  └──────────────────────────────────────────────────────────────────┘
```

### Key Features:
- **Dual-Reward Mechanism**:
  1. **Time-Based APY (Maturity Yield)**: A fixed APY reward calculated as an "all-or-nothing" yield once the stake's maturity duration has elapsed. Early exit yields strictly $0$ time-based rewards.
  2. **Deflationary Penalty Redistribution Yield**: Continuous yield distributed directly to remaining stakers whenever another participant exits early and pays a penalty. This yield is **claimable continuously at any time (uncapped)**.
- **Multiple Stakes per User**: Users can create multiple independent stakes within the same pool. Each stake has its own index (`stakeIndex`), principal, and lockup timer.
- **Dynamic Tier-Based Penalties**: Early withdrawals incur penalties scaled according to how long the user has staked, split into ascending duration tiers.
- **Deflationary Burns**: 50% of all early exit penalties are burned permanently from the circulating supply; the remaining 50% is distributed to the active stakers of the pool.
- **Staking Deadlines & Limits**: Pools can be configured with maximum staking capacity limits (`maxStakingAmount`) and staking end times (`stakingEndTime`).
- **Upgradeability**: Fully upgradeable using the **UUPS (Universal Upgradeable Proxy Standard)** pattern.

---

##  Essential Decimal & Math Specs

### 1. Token Decimals & Scaling
- **LOONY (`stakingToken`)**: **18 Decimals** (e.g., `1 LOONY = 10^18 wei`).
- **Precision Scale Factor (`PRECISION`)**: `1e18` (used internally for reward share calculations).

---

### 2. Dual-Reward Math Formulas

#### A) Time-Based Yield (Fixed APY at Maturity)
Staker earns this reward **strictly at or after maturity** (`duration >= maturityPeriod`). Time yield is strictly $0$ if unstaked early.
$$\text{Time APY Reward} = \frac{\text{Staked Amount} \times \text{apy}}{100}$$

#### B) Penalty Redistribution Yield (Continuous Yield)
When early unstakers exit, their penalty is split into equal halves:
$$\text{Penalty Amount} = \frac{\text{Staked Amount} \times \text{Tier Penalty Percentage}}{100}$$
$$\text{Burn Portion} = \frac{\text{Penalty Amount}}{2}$$
$$\text{Redistribution Portion} = \text{Penalty Amount} - \text{Burn Portion}$$

The redistribution portion is added to the global share accumulator bump:
$$\Delta\text{accPenaltyPerShare} = \frac{\text{Redistribution Portion} \times 10^{18}}{\text{Pool Total Staked}}$$

A staker's pending penalty yield is computed dynamically as:
$$\text{Pending Penalty} = \left(\frac{\text{Staked Amount} \times \text{accPenaltyPerShare}}{10^{18}}\right) - \text{penaltyDebt}$$

*`penaltyDebt` is snapshotted upon staking, claiming, or unstaking to prevent retroactive double-claiming.*

---

### 3. Tier Penalty Calculation Example
Consider a pool configured as follows:
- `maturityPeriod = 30 days` (`2592000` seconds)
- `tierPeriods = [10 days, 20 days, 30 days]`
- `tierPenalties = [30, 25, 20]` (represents 30%, 25%, 20%)

If a staker exits early, their penalty percentage matches their elapsed duration:
- $\text{Staking Duration} < 10 \text{ Days} \implies \mathbf{30\%}$ penalty
- $10 \text{ Days} \le \text{Staking Duration} < 20 \text{ Days} \implies \mathbf{25\%}$ penalty
- $20 \text{ Days} \le \text{Staking Duration} < 30 \text{ Days} \implies \mathbf{20\%}$ penalty
- $\text{Staking Duration} \ge 30 \text{ Days} \implies \mathbf{0\%}$ penalty (fully matured)

---

##  Read-Only APIs (Querying State)

### 1. `getPoolInfo(uint256 _poolId)`
Fetches configuration parameters and current state variables of a staking pool.
- **Visibility**: `external view`
- **Returns**:
  - `uint256 maturityPeriod`: Pool maturity duration in seconds.
  - `uint256 apy`: Annual percentage yield (e.g. `12` = 12% yield).
  - `uint256 totalStaked`: Total tokens currently staked in this pool.
  - `uint256 totalRewardsDistributed`: Total rewards claimed from this pool.
  - `uint256 accRewardPerShare`: Kept for interface compatibility.
  - `uint256 penaltyPool`: Total lifetime penalties processed (burn + redistribution).
  - `uint256 lastUpdateTime`: Timestamp of the last pool update.
  - `bool isActive`: Whether pool is accepting stakes/claims.
  - `uint256 participantCount`: Active staking wallets currently in this pool.
  - `uint256 accPenaltyPerShare`: Global penalty accumulator (scaled by 1e18).
  - `uint256 stakingEndTime`: Timestamp after which staking is disabled (`0` = no deadline).
  - `uint256 maxStakingAmount`: Max total tokens that can be staked in this pool (`0` = no limit).

### 2. `getUserStake(address _user, uint256 _poolId, uint256 _stakeIndex)`
Queries detailed information of a user's specific staking index position.
- **Visibility**: `external view`
- **Returns**: `UserStakeInfo` struct:
  - `uint256 amount`: Staked principal (18 decimals).
  - `uint256 startTimestamp`: Start time of the stake.
  - `uint256 rewardDebt`: Internal accounting variable.
  - `uint256 penaltyDebt`: Penalty yield debt snapshot.
  - `bool hasClaimedMaturityAPY`: True if the maturity APY reward was already claimed.

### 3. `getPoolTiers(uint256 _poolId)`
Returns the exit penalty tiers configuration.
- **Returns**: `(uint256[] periods, uint256[] penalties)` (Periods in seconds, Penalties in percentage 0-100).

### 4. `pendingRewards(address _user, uint256 _poolId, uint256 _stakeIndex)`
Calculates total pending rewards (Time APY + continuous penalty yield) claimable at the current block.
- **Returns**: `uint256` (18 decimals).

### 5. `getPenaltyInfo(address _user, uint256 _poolId, uint256 _stakeIndex)`
Helper providing the penalty status if a staker withdrew at the current second. **Crucial for frontend warnings.**
- **Returns**:
  - `uint256 penaltyPercent`: Current early withdrawal penalty percent (0-100).
  - `uint256 penaltyAmount`: Deduction amount in tokens (18 decimals).
  - `uint256 currentTier`: Tier index matched (or `type(uint256).max` if fully matured).

### 6. `getMaturityStatus(address _user, uint256 _poolId, uint256 _stakeIndex)`
- **Returns**: `(uint256 remainingTime, bool isMatured)` (Seconds remaining, maturity status).

### 7. `availableRewardBalance()`
- **Returns**: `uint256` (Reward token reserves funded in the staking contract, available to pay out APY).

---

##  Write APIs (State Mutations)

###  User-Facing Write APIs

#### 1. `stake(uint256 _poolId, uint256 _amount)`
Enters a user stake in the selected pool.
- **Access Control**: Public (Anyone, `nonReentrant`, `whenNotPaused`)
- **Requirements**:
  - Staker must first `approve` the staking contract address to spend `_amount` on the `$LOONY` contract.
  - Pool must be active (`isActive == true`) and current time must be prior to `stakingEndTime` (if set).
  - Staking amount must not exceed remaining capacity under `maxStakingAmount`.

#### 2. `unstake(uint256 _poolId, uint256 _stakeIndex)`
Withdraws the user stake. Automatically distributes earned time rewards and penalty yields, deducts early-exit penalty (if applicable), and closes the position.
- **Access Control**: Public (Anyone, `nonReentrant`, `whenNotPaused`)

#### 3. `claimRewards(uint256 _poolId, uint256 _stakeIndex)`
Harvests pending penalty yields and maturity APY rewards (if matured) without unstaking the principal.
- **Access Control**: Public (Anyone, `nonReentrant`, `whenNotPaused`)

---

###  Admin-Only Write APIs

These functions are restricted to the contract owner (`onlyOwner`) and are essential for configuring, managing, and securing the staking ecosystem.

#### 1. `createStakingPool(uint256 _maturityPeriod, uint256[] calldata _tierPeriods, uint256[] calldata _tierPenalties, uint256 _apy, uint256 _stakingEndTime, uint256 _maxStakingAmount)`
Deploys and configures a new staking pool in the ecosystem.
- **Access Control**: `onlyOwner`
- **Arguments**:
  - `_maturityPeriod`: Staking maturation duration in seconds (must be $> 0$).
  - `_tierPeriods`: Ascending array of period boundaries in seconds.
  - `_tierPenalties`: Corresponding early unstaking penalty percentages (must match length).
  - `_apy`: Maturity Annual Percentage Yield (e.g. `15` for 15% yield).
  - `_stakingEndTime`: Timestamp after which staking deposits are disabled (`0` = no end time).
  - `_maxStakingAmount`: Total pool cap limit (`0` = unlimited).
- **Returns**: `poolId` (index of the created pool).

#### 2. `updatePoolEndTimes(uint256[] calldata _poolIds, uint256[] calldata _endTimes)`
Batch updates the expiration timestamps (`stakingEndTime`) for multiple pools simultaneously.
- **Access Control**: `onlyOwner`

#### 3. `updatePool(uint256 _poolId, uint256 _maturityPeriod, uint256[] calldata _tierPeriods, uint256[] calldata _tierPenalties, uint256 _apy)`
Updates the main reward/penalty configuration for a specified pool.
- **Access Control**: `onlyOwner`
- **Note**: Triggers an internal pool update to lock in currently accumulated staker rewards under the previous configuration before applying new parameters.

#### 4. `updatePoolMaxLimit(uint256 _poolId, uint256 _maxStakingAmount)`
Adjusts the total staking cap capacity for an active pool.
- **Access Control**: `onlyOwner`

#### 5. `pausePool(uint256 _poolId)`
Suspends staking, unstaking, and claiming operations for a single pool.
- **Access Control**: `onlyOwner`

#### 6. `resumePool(uint256 _poolId)`
Re-activates a paused pool.
- **Access Control**: `onlyOwner`

#### 7. `withdrawToken(address _tokenContract, uint256 _amount)`
Emergency rescue mechanism allowing the owner to recover any ERC-20 token accidentally sent to the contract, or unused staking rewards.
- **Access Control**: `onlyOwner`, `nonReentrant`

#### 8. `pause()` / `unpause()`
Globally pauses or unpauses all user-facing interactions across the entire staking contract.
- **Access Control**: `onlyOwner`

#### 9. `UpdateLoonyToken(address _loonyToken)`
Updates the underlying token address used for staking.
- **Access Control**: `onlyOwner`

---

##  Smart Contract Events

| Event Name | Signature | When is it Emitted? |
| :--- | :--- | :--- |
| **`PoolCreated`** | `event PoolCreated(uint256 indexed poolId, uint256 maturityPeriod, uint256 apy, uint256 tierCount)` | When a new pool is configured and deployed by the admin. |
| **`Staked`** | `event Staked(address indexed user, uint256 indexed poolId, uint256 indexed stakeIndex, uint256 amount)` | Emitted when a new stake is created. |
| **`Unstaked`** | `event Unstaked(address indexed user, uint256 indexed poolId, uint256 indexed stakeIndex, uint256 amount, uint256 reward, uint256 penalty)` | Emitted when unstaking. Displays net reward paid and penalty deducted. |
| **`RewardsClaimed`** | `event RewardsClaimed(address indexed user, uint256 indexed poolId, uint256 indexed stakeIndex, uint256 amount)` | Emitted upon reward claim execution. |
| **`PenaltyBurned`** | `event PenaltyBurned(uint256 indexed poolId, uint256 amount)` | Emitted when early exit penalty (50%) is burned. |
| **`PenaltyRedistributed`** | `event PenaltyRedistributed(uint256 indexed poolId, uint256 amount)` | Emitted when early exit penalty (50%) is redistributed. |

---

##  Code Snippets for Developers

### Frontend Integration (Ethers.js v6)

#### 1. Fetching Staking Pool Details & User Stakes
```typescript
import { ethers } from "ethers";

interface UserPosition {
  stakeIndex: number;
  stakedAmount: string;
  startTimestamp: number;
  maturityDate: Date;
  isMatured: boolean;
  remainingSeconds: number;
  pendingReward: string;
  exitPenaltyPercent: number;
  exitPenaltyAmount: string;
}

async function fetchStakingDetails(
  userAddress: string,
  poolId: number,
  stakingContract: ethers.Contract
): Promise<{ poolStaked: string; poolAPY: number; activeStakes: UserPosition[] }> {
  // 1. Fetch pool state
  const poolInfo = await stakingContract.getPoolInfo(poolId);
  const totalStaked = ethers.formatUnits(poolInfo[2], 18);
  const apy = Number(poolInfo[1]);
  
  // 2. Fetch User stakes count
  const stakeCount = await stakingContract.getUserStakesCount(userAddress, poolId);
  const activeStakes: UserPosition[] = [];
  
  for (let i = 0; i < Number(stakeCount); i++) {
    const stakeInfo = await stakingContract.getUserStake(userAddress, poolId, i);
    if (stakeInfo.amount === 0n) continue; // Skip unstaked/empty slots
    
    const [remTime, isMatured] = await stakingContract.getMaturityStatus(userAddress, poolId, i);
    const pendingReward = await stakingContract.pendingRewards(userAddress, poolId, i);
    const [penaltyPct, penaltyAmt] = await stakingContract.getPenaltyInfo(userAddress, poolId, i);
    
    activeStakes.push({
      stakeIndex: i,
      stakedAmount: ethers.formatUnits(stakeInfo.amount, 18),
      startTimestamp: Number(stakeInfo.startTimestamp),
      maturityDate: new Date((Number(stakeInfo.startTimestamp) + Number(poolInfo[0])) * 1000),
      isMatured,
      remainingSeconds: Number(remTime),
      pendingReward: ethers.formatUnits(pendingReward, 18),
      exitPenaltyPercent: Number(penaltyPct),
      exitPenaltyAmount: ethers.formatUnits(penaltyAmt, 18)
    });
  }
  
  return { poolStaked: totalStaked, poolAPY: apy, activeStakes };
}
```

#### 2. Execute Staking Workflow
```typescript
import { ethers } from "ethers";

async function executeStake(
  poolId: number,
  stakeAmountFormatted: string,
  signer: ethers.Signer,
  stakingAddress: string,
  loonyAddress: string
) {
  const amountWei = ethers.parseUnits(stakeAmountFormatted, 18);
  
  const tokenABI = ["function approve(address spender, uint256 amount) public returns (bool)"];
  const stakingABI = ["function stake(uint256 poolId, uint256 amount) external"];
  
  const tokenContract = new ethers.Contract(loonyAddress, tokenABI, signer);
  const stakingContract = new ethers.Contract(stakingAddress, stakingABI, signer);
  
  console.log("Approving token staking...");
  const approveTx = await tokenContract.approve(stakingAddress, amountWei);
  await approveTx.wait();
  
  console.log("Executing stake transaction...");
  const stakeTx = await stakingContract.stake(poolId, amountWei);
  const receipt = await stakeTx.wait();
  
  console.log("Staking successful!", receipt);
}
```

#### 3. Unstaking (Early or Matured)
```typescript
import { ethers } from "ethers";

async function executeUnstake(
  poolId: number,
  stakeIndex: number,
  signer: ethers.Signer,
  stakingContract: ethers.Contract
) {
  const userAddress = await signer.getAddress();
  const [penaltyPct, penaltyAmt] = await stakingContract.getPenaltyInfo(userAddress, poolId, stakeIndex);
  
  if (penaltyPct > 0n) {
    const confirm = window.confirm(
      `Warning: You are unstaking early. You will incur a ${penaltyPct}% penalty (${ethers.formatUnits(penaltyAmt, 18)} LOONY). Proceed?`
    );
    if (!confirm) return;
  }
  
  console.log(`Unstaking index ${stakeIndex} from Pool ${poolId}...`);
  const tx = await stakingContract.unstake(poolId, stakeIndex);
  const receipt = await tx.wait();
  console.log("Unstaked successfully!", receipt);
}
```

---

### Backend Event Listener & Database Indexer

```javascript
const { ethers } = require("ethers");

async function indexStakingEvents(stakingAddress, providerUrl) {
  const provider = new ethers.JsonRpcProvider(providerUrl);
  const abi = [
    "event Staked(address indexed user, uint256 indexed poolId, uint256 indexed stakeIndex, uint256 amount)",
    "event Unstaked(address indexed user, uint256 indexed poolId, uint256 indexed stakeIndex, uint256 amount, uint256 reward, uint256 penalty)",
    "event RewardsClaimed(address indexed user, uint256 indexed poolId, uint256 indexed stakeIndex, uint256 amount)",
    "event PenaltyBurned(uint256 indexed poolId, uint256 amount)",
    "event PenaltyRedistributed(uint256 indexed poolId, uint256 amount)"
  ];
  
  const contract = new ethers.Contract(stakingAddress, abi, provider);
  
  contract.on("Staked", async (user, poolId, stakeIndex, amount, event) => {
    console.log(`[Staked] User: ${user}, Pool: ${poolId}, Index: ${stakeIndex}, Amount: ${ethers.formatUnits(amount, 18)} LOONY`);
    // Insert new stake row in database with active status.
  });

  contract.on("Unstaked", async (user, poolId, stakeIndex, amount, reward, penalty, event) => {
    console.log(`[Unstaked] User: ${user}, Pool: ${poolId}, Index: ${stakeIndex}, Principal: ${ethers.formatUnits(amount, 18)} LOONY, Reward: ${ethers.formatUnits(reward, 18)} LOONY, Penalty: ${ethers.formatUnits(penalty, 18)} LOONY`);
    // Mark stake row as withdrawn in database, add event logs, record penalty statistics.
  });

  contract.on("RewardsClaimed", async (user, poolId, stakeIndex, amount, event) => {
    console.log(`[RewardsClaimed] User: ${user}, Pool: ${poolId}, Index: ${stakeIndex}, Claimed: ${ethers.formatUnits(amount, 18)} LOONY`);
    // Update total rewards claimed for index position in database.
  });

  contract.on("PenaltyBurned", async (poolId, amount, event) => {
    console.log(`[Burned] Pool: ${poolId}, Burned: ${ethers.formatUnits(amount, 18)} LOONY`);
    // Accumulate metrics on burned tokens.
  });

  contract.on("PenaltyRedistributed", async (poolId, amount, event) => {
    console.log(`[Redistribution] Pool: ${poolId}, Distributed: ${ethers.formatUnits(amount, 18)} LOONY`);
    // Update global pool metrics to display continuous yields.
  });
}
```
