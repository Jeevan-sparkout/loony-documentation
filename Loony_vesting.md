# LoonyVesting Smart Contract Integration Guide
`LoonyVesting.sol`

Welcome to the integration guide for the **LoonyVesting** smart contract. This document provides clear, actionable instructions, data structures, methods, events, and code snippets designed specifically for both **Frontend** and **Backend** developers to interact with the vesting schedule contract.

---

##  Architecture & Design Overview

The `LoonyVesting` contract manages the lockup and distribution schedules of `$LOONY` tokens purchased during the crowdsale, as well as the special multi-phase vesting schedules for protocol founders.

### Key Features:
- **Dual-Track Vesting**: Separate schedules and rules for public crowdsale buyers (non-founders) and internal team members (founders).
- **Post-Sale Activation**: Vesting logic is strictly locked until the crowdsale contract finalizes the crowdsale by calling `startVestingAfterSale()`, which transitions `saleCompleted` to `true` and records the exact finalization timestamp.
- **Round-Based Cliff Support**: Non-founder grants have customizable cliff periods linked to the crowdsale round in which they purchased (e.g., 3 months cliff for private round, 2 months cliff for public rounds).
- **Discrete Monthly Releases**: Tokens unlock in discrete intervals of a "month" (default `2629743` seconds) rather than a continuous per-second unlock.
- **Grant Revocability**: Contract owner can revoke a user's grant, automatically sending already vested tokens to the recipient and returning non-vested tokens to the crowdsale reserve.
- **Founders Custom Distribution**: Fully customizable cliff, interval, and dual-phase distribution logic.
- **Upgradeability**: Fully upgradeable using the **UUPS (Universal Upgradeable Proxy Standard)** pattern.

---

##  Essential Decimal & Math Specs

### 1. Token Precision
- **LOONY (`loonyToken`)**: **18 Decimals** (e.g., `1 LOONY = 10^18 wei`).

### 2. Time-Based Variables
- **Standard Month (`monthTimeInSeconds`)**: Default set to `2629743` seconds ($\approx 30.44$ days). This can be updated by the owner to speed up testing or change schedule frequency.

---

### 3. Vesting Schedule Logic

#### Non-Founder Schedule
- **TGE Portion**: 15% is transferred directly to the buyer's wallet immediately during the crowdsale purchase (handled by `LoonyCrowdsale`).
- **Locked Portion (85%)**: Transferred to the `LoonyVesting` contract and added to the user's round-specific grant.
- **Start Time**: Vesting start is computed as:
  $$\text{Effective Start Time} = \text{saleCompletionTime} + \text{cliffPeriod}$$
- **Unlock Mechanics**: 
  - Claims remain completely locked ($0$ vested) until `block.timestamp >= Effective Start Time + monthTimeInSeconds` (i.e. one full month after the cliff ends).
  - Unlocks occur in discrete monthly chunks:
    $$\text{Amount per Month} = \frac{\text{Vesting Pool Size} - \text{Already Claimed}}{\text{Total Duration (Months)} - \text{Months Already Claimed}}$$
    $$\text{Vested Amount} = \text{Elapsed Months since Start} \times \text{Amount per Month}$$

#### Founder Schedule (18 Months Duration)
- **TGE Unlock**: A custom percentage (e.g., `15%` or `20%`) is unlocked and sent *immediately* upon being added to the contract via `addFounders()`.
- **Remaining Pool (e.g., 85%)**: Vests in two distinct phases over **18 months** starting from `saleCompletionTime + cliffPeriod` (default cliff is 6 months):
  
  > [!NOTE]
  > **Founder Vesting Phases:**
  > - **Phase 1 (Months 1 to 6 of vesting)**: Vests **25%** of the remaining pool, distributed evenly at `4.167%` of the pool per month.
  > - **Phase 2 (Months 7 to 18 of vesting)**: Vests the remaining **75%** of the pool, distributed evenly at `6.25%` of the pool per month.

- **Founder Phase Math**:
  Let $P$ be the founder's vesting pool (85% of total allocation) and $M$ be the complete months elapsed since the founder vesting start time:
  - If $M \le 6$:
    $$\text{Vested From Pool} = \frac{P \times 25 \times M}{600}$$
  - If $6 < M \le 18$:
    $$\text{Vested From Pool} = \frac{P \times 25}{100} + \frac{P \times 75 \times (M - 6)}{1200}$$
  - If $M > 18$:
    $$\text{Vested From Pool} = P$$

---

##  Read-Only APIs (Querying State)

Use these functions to populate user account dashboards, display locking schedules, and construct claiming interfaces.

### 1. `getGrantDetails(address _recipient, uint256 _roundId)`
Fetches comprehensive parameters of a specific round-based grant.
- **Visibility**: `external view`
- **Returns**:
  - `uint256 startTime`: Effective start timestamp (cliff included) or `0` if sale is not completed.
  - `uint256 amount`: Total tokens in the vesting pool (18 decimals).
  - `uint256 grantVestingDuration`: Total duration in months.
  - `uint256 monthsClaimed`: Total month intervals already claimed.
  - `uint256 totalClaimed`: Total tokens already transferred to the user (18 decimals).
  - `uint256 remainingAmount`: Remaining locked tokens.
  - `uint256 nextClaim`: Next block timestamp when a claim becomes active. Returns `0` if all periods claimed or sale is active.
  - `uint256 cliffPeriod`: Cliff period length in seconds.
  - `bool isFounderGrant`: Whether this is a founder grant (always false for this function).

### 2. `getFounderDetails(address _founder)`
Fetches detailed info for a team founder grant.
- **Visibility**: `external view`
- **Returns**:
  - `uint256 startTime`: Effective vesting start timestamp (cliff included).
  - `uint256 vestingAmount`: The 85% vesting pool (18 decimals).
  - `uint256 initialAmount`: Complete allocation (TGE + vesting pool combined).
  - `uint256 tgeAmount`: TGE portion instantly unlocked at addition (18 decimals).
  - `uint256 grantVestingDuration`: Stated in months (always `18`).
  - `uint256 monthsClaimed`: Months already claimed.
  - `uint256 totalClaimed`: Aggregate tokens claimed (includes the TGE portion).
  - `uint256 remainingAmount`: Remaining locked tokens.
  - `uint256 nextClaim`: Timestamp of next claim availability.
  - `uint256 cliffPeriod`: Lockup cliff duration in seconds.
  - `uint256 vestedAmount`: Current total amount claimable right now.
  - `uint256 nextVestingAmount`: Number of tokens that will unlock at the next monthly interval.

### 3. `getTotalGrantDetails(address _user)`
Aggregated data helper across **all rounds** the user participated in. Perfect for general portfolio summaries.
- **Visibility**: `external view`
- **Returns**:
  - `uint256 totalAllocated`: Total vested tokens allocated across all rounds.
  - `uint256 totalAvailableToClaim`: Combined amount of tokens claimable right now.
  - `uint256 totalClaimed`: Aggregated tokens already claimed.

### 4. `calculateGrantClaim(address _recipient, uint256 _roundId)`
Performs the math to evaluate claimable tokens at the current block.
- **Visibility**: `public view`
- **Returns**:
  - `uint256 monthsVested`: Number of newly matured month intervals.
  - `uint256 amountVested`: New claimable tokens (18 decimals).

### 5. `getUserRounds(address _user)`
Queries all rounds in which a user has an active grant.
- **Visibility**: `external view`
- **Returns**: `uint256[]` (Indices of rounds, e.g. `[0, 1, 2]`)

### 6. `isFounder(address founder)`
- **Returns**: `bool` (True if address is registered as founder)

### 7. `getFounders()`
- **Returns**: `address[]` (List of registered founder addresses)

---

##  Write APIs (State Mutations)

###  User-Facing Write APIs

#### 1. `claimVestedTokens()`
The primary user interaction method for regular participants. Consolidates vested tokens across all participating rounds in a single call.
- **Access Control**: Public (Anyone, `nonReentrant`, `whenNotPaused`)
- **Requirements**: Crowdsale must be finalized (`saleCompleted == true`).

#### 2. `founderClaimTokens()`
The claim method dedicated strictly to registered founders.
- **Access Control**: Public (Anyone, `nonReentrant`, `whenNotPaused`)
- **Requirements**: Caller must be a registered founder.

---

###  Admin-Only Write APIs

These administrative functions are reserved strictly for the contract owner (`onlyOwner`) or designated crowdsale contract linkage.

#### 1. `addCrowdsaleAddress(address _crowdsaleAddress)`
Configures the linkage to the authorized crowdsale contract.
- **Access Control**: `onlyOwner`

#### 2. `addTokenGrant(address _recipient, uint256 _amount, uint256 _cliffPeriod, uint256 _vestingDuration, uint256 _roundId)`
Manually registers or increases a regular user's round-specific grant schedule.
- **Access Control**: Owner or linked crowdsale address (`onlyOwner` / `onlyCrowdsale`)

#### 3. `addFounders(address[] memory _founders, uint256 _amount, uint256 _cliff, uint256 _founderVestingInterval, uint256 _tgeAmount)`
Initializes the list of founders and immediately unlocks/dispatches their respective TGE portions. Can only be initialized once.
- **Access Control**: `onlyOwner`, `nonReentrant`

#### 4. `revokeTokenGrant(address _recipient, uint256 _roundId)`
Revokes an active regular user grant. Instantly dispatches already accrued vested tokens to the user, and reclaims all unvested remaining tokens, transferring them back to the crowdsale reserve.
- **Access Control**: `onlyOwner`, `nonReentrant`

#### 5. `updateMonthsTime(uint256 _monthsInSeconds)`
Modifies the duration definition of a standard vesting "month" (default `2629743` seconds).
- **Access Control**: `onlyOwner`

#### 6. `setToken(address _tokenAddress)`
Re-configures the target ERC-20 token managed by the vesting vault.
- **Access Control**: `onlyOwner`

#### 7. `updateFoundersLockInPeriod(uint256 _cliffPeriod)`
Adjusts the cliff period duration for the team founder grants.
- **Access Control**: `onlyOwner`

#### 8. `withdrawToken(address _tokenAddress, uint256 _amount)`
Enables the recovery of any ERC-20 tokens accidentally sent to the contract.
- **Access Control**: `onlyOwner`

#### 9. `withdrawEther(uint256 _amount)`
Enables the recovery of any native blockchain gas tokens (e.g. BNB/ETH) from the contract balance.
- **Access Control**: `onlyOwner`

---

## ⚡ Smart Contract Events

| Event Name | Signature | When is it Emitted? |
| :--- | :--- | :--- |
| **`GrantAdded`** | `event GrantAdded(address indexed recipient, uint256 roundId)` | Emitted when a round grant is added or increased. |
| **`GrantTokensClaimed`** | `event GrantTokensClaimed(address indexed recipient, uint256 amount)` | Emitted when tokens are successfully claimed by a founder. |
| **`MultiRoundTokensClaimed`** | `event MultiRoundTokensClaimed(address indexed recipient, uint256 totalAmount, uint256 roundCount)` | Emitted when a standard user claims from all rounds. |
| **`GrantRevoked`** | `event GrantRevoked(address recipient, uint256 amountVested, uint256 amountNotVested)` | Emitted upon a grant revocation by the owner. |
| **`VestingStarted`** | `event VestingStarted()` | Emitted when the crowdsale is closed and vesting globally begins. |
| **`MonthTimeUpdated`** | `event MonthTimeUpdated(uint256 _intervalTime)` | Emitted if the admin updates the standard month interval in seconds. |

---

##  Code Snippets for Developers

### Frontend Integration (Ethers.js v6)

#### 1. Loading Vesting Schedule & Claim Status
```typescript
import { ethers } from "ethers";

interface RoundVestingInfo {
  roundId: number;
  totalAllocated: string;
  totalClaimed: string;
  availableToClaim: string;
  nextClaimDate: Date | null;
  cliffSeconds: number;
}

async function fetchUserVestingPortfolio(userAddress: string, vestingContract: ethers.Contract): Promise<{
  summary: { totalAllocated: string; totalClaimed: string; totalAvailable: string };
  rounds: RoundVestingInfo[];
}> {
  // 1. Fetch Aggregated details
  const [allocated, claimable, claimed] = await vestingContract.getTotalGrantDetails(userAddress);
  
  // 2. Fetch list of participated rounds
  const rounds: bigint[] = await vestingContract.getUserRounds(userAddress);
  
  const roundDetails: RoundVestingInfo[] = [];
  
  for (const roundId of rounds) {
    const details = await vestingContract.getGrantDetails(userAddress, roundId);
    
    // details format: [startTime, amount, vestingDuration, monthsClaimed, totalClaimed, remainingAmount, nextClaim, cliffPeriod, isFounder]
    const nextClaimTs = Number(details[6]);
    
    roundDetails.push({
      roundId: Number(roundId),
      totalAllocated: ethers.formatUnits(details[1], 18),
      totalClaimed: ethers.formatUnits(details[4], 18),
      availableToClaim: ethers.formatUnits(details[1] - details[4], 18), // or call calculateGrantClaim
      nextClaimDate: nextClaimTs > 0 ? new Date(nextClaimTs * 1000) : null,
      cliffSeconds: Number(details[7])
    });
  }
  
  return {
    summary: {
      totalAllocated: ethers.formatUnits(allocated, 18),
      totalAvailable: ethers.formatUnits(claimable, 18),
      totalClaimed: ethers.formatUnits(claimed, 18),
    },
    rounds: roundDetails
  };
}
```

#### 2. Claiming All Vested Tokens
```typescript
import { ethers } from "ethers";

async function executeVestingClaim(signer: ethers.Signer, vestingAddress: string) {
  const vestingABI = [
    "function claimVestedTokens() external",
    "function getTotalGrantDetails(address user) view returns (uint256, uint256, uint256)"
  ];
  
  const vestingContract = new ethers.Contract(vestingAddress, vestingABI, signer);
  const userAddress = await signer.getAddress();
  
  const [, claimable] = await vestingContract.getTotalGrantDetails(userAddress);
  
  if (claimable === 0n) {
    throw new Error("No tokens available to claim at this moment.");
  }
  
  console.log(`Executing claim for ${ethers.formatUnits(claimable, 18)} LOONY...`);
  const tx = await vestingContract.claimVestedTokens();
  const receipt = await tx.wait();
  
  console.log("Tokens claimed successfully!", receipt);
}
```

---

### Backend Event Listener & Database Indexer

```javascript
const { ethers } = require("ethers");

async function indexVestingEvents(vestingAddress, providerUrl) {
  const provider = new ethers.JsonRpcProvider(providerUrl);
  const abi = [
    "event GrantAdded(address indexed recipient, uint256 roundId)",
    "event MultiRoundTokensClaimed(address indexed recipient, uint256 totalAmount, uint256 roundCount)",
    "event GrantRevoked(address recipient, uint256 amountVested, uint256 amountNotVested)",
    "event VestingStarted()"
  ];
  
  const contract = new ethers.Contract(vestingAddress, abi, provider);
  
  contract.on("GrantAdded", async (recipient, roundId, event) => {
    console.log(`[GrantAdded] Recipient: ${recipient}, Round ID: ${roundId}`);
    // Sync DB: Flag that this user now has vesting locked in for this round.
  });

  contract.on("MultiRoundTokensClaimed", async (recipient, totalAmount, roundCount, event) => {
    console.log(`[Claimed] Recipient: ${recipient}, Claimed: ${ethers.formatUnits(totalAmount, 18)} LOONY from ${roundCount} rounds.`);
    // Sync DB: Decrement locks, record claim event history, update portfolio values.
  });

  contract.on("GrantRevoked", async (recipient, amountVested, amountNotVested, event) => {
    console.log(`[Revoked] Recipient: ${recipient}, Vested paid: ${ethers.formatUnits(amountVested, 18)} LOONY, Unvested clawed back: ${ethers.formatUnits(amountNotVested, 18)} LOONY.`);
    // Sync DB: Mark grant status as revoked, delete pending locks.
  });

  contract.on("VestingStarted", async (event) => {
    console.log("[VestingStarted] Vesting is now officially active protocol-wide!");
    // Sync DB: Calculate and update global vesting start timestamps.
  });
}
```
