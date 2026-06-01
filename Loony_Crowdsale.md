# LoonyCrowdsale Smart Contract Integration Guide
`LoonyCrowdsale.sol`

Welcome to the integration guide for the **LoonyCrowdsale** smart contract. This document provides clear, actionable instructions, data structures, methods, events, and code snippets designed specifically for both **Frontend** and **Backend** developers.

---

##  Architecture & Design Overview
The `LoonyCrowdsale` contract manages the public token sale of `$LOONY` in exchange for `USDT`. 

### Key Features:
- **Round-Based Pricing**: Pricing rounds with custom rates, cliff durations, and token limits.
- **Vesting Integration**: Automatically creates vesting schedules via the `LoonyVesting` contract for the locked portion of purchases.
- **TGE Unlock**: Automatically transfers a percentage (default `15%`) of the purchased tokens directly to the buyer's wallet immediately, vesting the remaining `85%`.
- **Private Sale Whitelisting**: The first round (Round 0) is a private sale requiring manual whitelist request and owner approval.
- **Upgradeability**: Fully upgradeable using the **UUPS (Universal Upgradeable Proxy Standard)** pattern.
- **Deflationary Quarterly Burns**: Allows burning up to 5,000,000 `$LOONY` from a reserve wallet at regular intervals (simulated quarters).

---

##  Essential Decimal & Math Specs

> [!IMPORTANT]
> **Token Decimals & Conversions:**
> - **USDT (`usdtToken`)**: **6 Decimals** (e.g., `1 USDT = 1,000,000`).
> - **LOONY (`loonyToken`)**: **18 Decimals** (e.g., `1 LOONY = 10^18 wei`).
> - **Purchase Rate Math**: 
>   $$\text{LOONY Received (18 decimals)} = \frac{\text{USDT Spent (6 decimals)} \times 10^{18}}{\text{rate}}$$
>   - *Example (Round 0: rate = 10,000)*: 
>     $$\text{LOONY Amount} = \frac{1,000,000 \times 10^{18}}{10,000} = 100 \times 10^{18} \text{ (100 LOONY)}$$

| Stage Index | Stage Name | Rate Parameter | Conversion Ratio | Cliff Duration |
| :--- | :--- | :--- | :--- | :--- |
| **0** | Private Sale | `10000` | $1 \text{ USDT} = 100 \text{ LOONY}$ | 3 Months (`7889229s`) |
| **1** | Round 1 | `15000` | $1 \text{ USDT} \approx 66.67 \text{ LOONY}$ | 2 Months (`5259486s`) |
| **2** | Round 2 | `20000` | $1 \text{ USDT} = 50 \text{ LOONY}$ | 2 Months (`5259486s`) |
| **3** | Round 3 | `30000` | $1 \text{ USDT} \approx 33.33 \text{ LOONY}$ | 2 Months (`5259486s`) |

---

##  Read-Only APIs (Querying State)

Use these functions to fetch data for dashboards, user profiles, or indexer sync.

### 1. `getTokenAmount(uint256 usdtAmount)`
Returns the amount of `$LOONY` (with 18 decimals) a buyer would receive for a specific `usdtAmount` (6 decimals) in the current active round.
- **Visibility**: `public view`
- **Output**: `uint256`

### 2. `userInfo(address user)`
Public mapping providing purchase stats for a specific user.
- **Visibility**: `public view`
- **Returns**:
  - `usdtContributed`: Total `USDT` spent (6 decimals)
  - `loonyReceived`: Total `$LOONY` purchased (18 decimals, TGE + vested combined)
  - `TGEUnlockedAmount`: Total `$LOONY` unlocked immediately at TGE (18 decimals)

### 3. `rounds(uint256 index)`
Public array containing round structural configurations.
- **Visibility**: `public view`
- **Returns**:
  - `rate`: Rate coefficient (e.g., `10000`)
  - `tokensForSale`: Total allocation for this round (18 decimals)
  - `tokensSold`: Amount sold in this round (18 decimals)
  - `cliffPeriodInSeconds`: Cliff duration in seconds

### 4. `getRounds()`
Helper view returning all round parameters in parallel arrays.
- **Visibility**: `external view`
- **Returns**:
  - `uint256[] rates`: All round rates
  - `uint256[] tokensForSale`: All round allocation capacities
  - `uint256[] tokensSold`: All round sold tokens

### 5. `currentRound()`
- **Visibility**: `public view`
- **Returns**: `uint256` (Index of the current active round, e.g. `0`, `1`, `2`, `3`)

### 6. `whitelisted(address user)`
- **Visibility**: `public view`
- **Returns**: `bool` (True if the user is whitelisted for the Private Sale)

### 7. `isRequested(address user)`
- **Visibility**: `public view`
- **Returns**: `bool` (True if the user has requested a whitelist slot and is pending review)

### 8. `getPendingWhitelistRequests()`
- **Visibility**: `external view`
- **Returns**: `address[]` (List of pending whitelist applicants)

---

##  Write APIs (State Mutations)

### 1. `requestWhitelist()`
Called by users during `currentRound == 0` to request access to the private sale.
- **Access Control**: Anyone
- **Requirements**: Current round must be private sale (`0`), user must not be already whitelisted or pending.

### 2. `buyTokens(uint256 usdtAmount)`
The main purchase function.
- **Access Control**: Anyone (nonReentrant)
- **Requirements**:
  1. User must first `approve` the crowdsale address to spend `usdtAmount` on the USDT token contract.
  2. If `currentRound == 0` (Private Sale), the caller's address **must** be whitelisted.
  3. The caller **must not** be a registered founder in `LoonyVesting`.
  4. Contract must hold sufficient `$LOONY` to satisfy the immediate TGE unlock (`15%`) and transfer the remaining `85%` to the Vesting contract.

### 3. `approveWhitelist(address user)`
Approves a pending whitelist request.
- **Access Control**: `onlyOwner`

### 4. `rejectWhitelist(address user)`
Rejects a pending whitelist request.
- **Access Control**: `onlyOwner`

### 5. `setSaleRound(uint256 newRoundIndex)`
Manually advances the crowdsale stage. Unsold tokens from the current round are automatically carried forward and added to the new round's allocation capacity.
- **Access Control**: `onlyOwner`
- **Requirements**: New round index must be greater than `currentRound`, and the current round's minimum period (`roundMinPeriods[currentRound]`) must have elapsed.

---

## ⚡ Smart Contract Events (Webhook / Graph Indexing)

| Event Name | Signature | When is it Emitted? |
| :--- | :--- | :--- |
| **`TokensPurchased`** | `event TokensPurchased(address indexed buyer, uint256 amount)` | Emitted upon a successful token purchase. `amount` is the total LOONY purchased. |
| **`TGEUnlocked`** | `event TGEUnlocked(address indexed user, uint256 amount)` | Emitted when TGE portion (15%) is instantly transferred to buyer. |
| **`WhitelistRequested`** | `event WhitelistRequested(address indexed user)` | When a user requests whitelist slot for Private Sale. |
| **`WhitelistApproved`** | `event WhitelistApproved(address indexed user)` | When admin approves whitelist. |
| **`WhitelistRejected`** | `event WhitelistRejected(address indexed user)` | When admin rejects whitelist. |
| **`RoundAdvanced`** | `event RoundAdvanced(uint256 indexed newStage)` | Emitted when sale advances to next round (manually or automatically when sold out). |
| **`AdminRoundChanged`** | `event AdminRoundChanged(uint256 indexed oldRound, uint256 indexed newRound, uint256 tokensTransferred)` | Manually advanced round by admin carrying forward `tokensTransferred` unsold tokens. |
| **`CrowdsaleFinalized`** | `event CrowdsaleFinalized()` | Emitted when crowdsale is finalized and vesting is globally initiated. |

---

##  Code Snippets for Developers

### Frontend Integration (Ethers.js v6)

#### 1. Checking Purchase Qualification & Quote
```typescript
import { ethers } from "ethers";

async function checkAndQuote(userAddress: string, usdtInputAmount: string, crowdsaleContract: ethers.Contract) {
  const currentRound = await crowdsaleContract.currentRound();
  
  if (currentRound === 0n) {
    const isWhitelisted = await crowdsaleContract.whitelisted(userAddress);
    if (!isWhitelisted) {
      const isPending = await crowdsaleContract.isRequested(userAddress);
      return { allowed: false, reason: isPending ? "Whitelist request pending" : "Not whitelisted" };
    }
  }
  
  // Convert 100 USDT input (6 decimals)
  const usdtWei = ethers.parseUnits(usdtInputAmount, 6);
  const expectedLoony = await crowdsaleContract.getTokenAmount(usdtWei);
  
  return {
    allowed: true,
    expectedLoony: ethers.formatUnits(expectedLoony, 18),
    usdtWei
  };
}
```

#### 2. Executing Purchase Workflow
```typescript
import { ethers } from "ethers";

async function purchaseLoonyTokens(usdtAmountFormatted: string, signer: ethers.Signer, crowdsaleAddress: string, usdtAddress: string) {
  const usdtAmount = ethers.parseUnits(usdtAmountFormatted, 6);
  
  const usdtABI = ["function approve(address spender, uint256 amount) public returns (bool)"];
  const crowdsaleABI = ["function buyTokens(uint256 usdtAmount) external"];
  
  const usdtContract = new ethers.Contract(usdtAddress, usdtABI, signer);
  const crowdsaleContract = new ethers.Contract(crowdsaleAddress, crowdsaleABI, signer);
  
  console.log("Approving USDT transfer...");
  const approveTx = await usdtContract.approve(crowdsaleAddress, usdtAmount);
  await approveTx.wait();
  
  console.log("Executing purchase...");
  const purchaseTx = await crowdsaleContract.buyTokens(usdtAmount);
  const receipt = await purchaseTx.wait();
  
  console.log("Tokens Purchased Successfully!", receipt);
}
```

### Backend Event Listener & Database Indexer

```javascript
const { ethers } = require("ethers");

async function indexCrowdsaleEvents(crowdsaleAddress, providerUrl) {
  const provider = new ethers.JsonRpcProvider(providerUrl);
  const abi = [
    "event TokensPurchased(address indexed buyer, uint256 amount)",
    "event TGEUnlocked(address indexed user, uint256 amount)",
    "event WhitelistRequested(address indexed user)"
  ];
  
  const contract = new ethers.Contract(crowdsaleAddress, abi, provider);
  
  contract.on("TokensPurchased", async (buyer, amount, event) => {
    console.log(`[TokensPurchased] User: ${buyer}, Total purchased: ${ethers.formatUnits(amount, 18)} LOONY`);
    // Insert purchase history into database, update user's total investment status.
  });

  contract.on("TGEUnlocked", async (user, amount, event) => {
    console.log(`[TGEUnlocked] User: ${user}, TGE Transfer: ${ethers.formatUnits(amount, 18)} LOONY`);
  });

  contract.on("WhitelistRequested", async (user, event) => {
    console.log(`[WhitelistRequested] User: ${user} applied for private sale.`);
    // Send admin notification or queue for review dashboard
  });
}
```
