# LoonyToken Smart Contract Integration Guide
`LoonyToken.sol`

Welcome to the integration guide for the **LoonyToken** smart contract. This document provides clear, actionable instructions, data structures, methods, events, and code snippets designed specifically for both **Frontend** and **Backend** developers working with the `$LOONY` token.

---

##  Architecture & Design Overview
`LoonyToken` is the utility token powering the Loony Ecosystem. It extends standard ERC-20 functionality with a customizable transaction fee model called the **Engine Tax**, standard gasless approvals (`ERC20Permit`), and full upgradeability.

### Key Features:
- **UUPS Upgradeable**: Fully upgradeable using the **UUPS (Universal Upgradeable Proxy Standard)** pattern.
- **Engine Tax (3% Transfer Fee)**: Every standard token transfer triggers a 3.0% fee split between Art & Charity DAO, Development & Operations, and Staking Pools.
- **Exclusion Mechanics**: Key protocol components (Crowdsale, Vesting, Staking contracts, and target tax wallets) are excluded from fees to prevent tax-on-tax loops.
- **ERC-20 Permit (EIP-2612)**: Supports signature-based approvals, allowing gasless user transactions and single-transaction execution patterns.
- **Supply Deflation**: Includes a public `burn` method to permanently reduce token supply.

---

##  Essential Decimal & Math Specs

> [!IMPORTANT]
> **Token Decimals & Basis Points:**
> - **LOONY (`loonyToken`)**: **18 Decimals** (e.g., `1 LOONY = 10^18 wei`).
> - **Fee Math Basis Points (BPS)**: `10000 = 100%`. E.g., `300 BPS = 3.0%`.
> - **Total Fee Cap**: Administrative fee changes are strictly capped at a combined maximum of **10.0%** (`1000 BPS`) to ensure security.

### Default Fee Split:
$$\text{Total Fee} = 300 \text{ BPS (3.0\%)}$$

| Fee Category | BPS Value | Percentage | Target Wallet / Destination |
| :--- | :--- | :--- | :--- |
| **Art & Charity** | `150` | 1.5% | `artCharityWallet` (DAO Fund) |
| **Development** | `100` | 1.0% | `developmentWallet` (Ops/Dev) |
| **Staking Yield** | `50` | 0.5% | `stakingWallet` (Real Yield rewards) |

---

##  Read-Only APIs (Querying State)

Use these functions to fetch token configurations, balances, and fee exclusions.

### 1. Standard ERC-20 Views
- **`name()`**: Returns the string `"LOONY"`.
- **`symbol()`**: Returns the string `"LOONY"`.
- **`decimals()`**: Returns `18`.
- **`totalSupply()`**: Returns the current total outstanding supply (18 decimals).
- **`balanceOf(address account)`**: Returns the token balance of the specified address.
- **`allowance(address owner, address spender)`**: Returns the approved limit for `spender` on behalf of `owner`.

### 2. Fee Config & Wallets
- **`feeEnabled()`**: Returns `bool` indicating if the transfer tax is currently active.
- **`totalFee()`**: Combined BPS fee (e.g. `300` for 3.0%).
- **`artCharityFee()`** / `artCharityWallet()`: Current fee BPS and destination for charity.
- **`developmentFee()`** / `developmentWallet()`: Current fee BPS and destination for dev ops.
- **`stakingFee()`** / `stakingWallet()`: Current fee BPS and destination for staking rewards.

### 3. Exclusions
- **`isExcludedFromFee(address account)`**: Returns `bool` indicating if `account` is exempt from the transfer tax.

### 4. ERC-20 Permit Details
- **`nonces(address owner)`**: Returns the current cryptographic nonce for EIP-2612 permits.
- **`DOMAIN_SEPARATOR()`**: Returns the EIP-712 domain separator used in signing permit actions.

---

##  Write APIs (State Mutations)

###  User-Facing Write APIs

#### 1. Standard ERC-20 Mutations
- **`transfer(address recipient, uint256 amount)`**: Moves tokens to recipient. Tax is automatically deducted from `amount` if active and neither address is excluded.
- **`approve(address spender, uint256 amount)`**: Sets approval limit.
- **`transferFrom(address sender, address recipient, uint256 amount)`**: Executed by approved spenders.

#### 2. `permit(address owner, address spender, uint256 value, uint256 deadline, uint8 v, bytes32 r, bytes32 s)`
Allows gasless approvals via off-chain signatures.
- **Access Control**: Public (Anyone)

#### 3. `burn(uint256 amount)`
Permanently burns tokens from the caller's balance, decreasing total supply.
- **Access Control**: Public (Anyone)

---

###  Admin-Only Write APIs

Administrative functions reserved strictly for the owner (`onlyOwner`).

#### 1. Wallet Updates
- **`setArtCharityWallet(address _wallet)`**
- **`setDevelopmentWallet(address _wallet)`**
- **`setStakingWallet(address _wallet)`**
- *Note: Updating a wallet automatically updates its exemption state (new wallet is excluded, old wallet is removed from exclusions).*

#### 2. Fee Rate Customizations (BPS)
- **`setArtCharityFee(uint256 _fee)`**
- **`setDevelopmentFee(uint256 _fee)`**
- **`setStakingFee(uint256 _fee)`**
- *Requirement: Combined sum of all three fees must not exceed `1000` (10.0%).*

#### 3. General Controls
- **`setFeeEnabled(bool _enabled)`**: Toggles the Engine Tax on/off globally.
- **`setExcludedFromFee(address account, bool excluded)`**: Whitelists or removes an address from fee exemptions.

---

## ⚡ Smart Contract Events

| Event Signature | Trigger Condition |
| :--- | :--- |
| **`Transfer(address indexed from, address indexed to, uint256 value)`** | Standard ERC-20 transfer. Triggers multiple times per transfer if fees are active (to split destinations). |
| **`Approval(address indexed owner, address indexed spender, uint256 value)`** | When spender's allowance is set or modified. |
| **`ExcludedFromFee(address indexed account, bool excluded)`** | When an address is whitelisted/removed from fee exemptions. |
| **`FeeEnabledUpdated(bool enabled)`** | When the transfer tax is toggled globally. |
| **`ArtCharityWalletUpdated`** / `DevelopmentWalletUpdated` / `StakingWalletUpdated` | When tax wallet addresses are adjusted. |

---

##  Code Snippets for Developers

### Frontend Integration (Ethers.js v6)

#### 1. Checking Balance & Fee Estimates
```typescript
import { ethers } from "ethers";

async function queryTokenAndEstimateTransfer(
  userAddress: string,
  recipientAddress: string,
  rawTransferAmount: string,
  tokenContract: ethers.Contract
) {
  const decimals = await tokenContract.decimals();
  const balance = await tokenContract.balanceOf(userAddress);
  const amountWei = ethers.parseUnits(rawTransferAmount, decimals);

  if (balance < amountWei) {
    throw new Error("Insufficient balance");
  }

  const feeEnabled = await tokenContract.feeEnabled();
  const isSenderExcluded = await tokenContract.isExcludedFromFee(userAddress);
  const isRecipientExcluded = await tokenContract.isExcludedFromFee(recipientAddress);

  const taxApplies = feeEnabled && !isSenderExcluded && !isRecipientExcluded;

  let recipientReceives = amountWei;
  let taxDeducted = 0n;

  if (taxApplies) {
    const artCharityFee = await tokenContract.artCharityFee();
    const developmentFee = await tokenContract.developmentFee();
    const stakingFee = await tokenContract.stakingFee();
    const totalFeeBps = artCharityFee + developmentFee + stakingFee;

    taxDeducted = (amountWei * totalFeeBps) / 10000n;
    recipientReceives = amountWei - taxDeducted;
  }

  return {
    rawTransferAmount: ethers.formatUnits(amountWei, decimals),
    recipientReceives: ethers.formatUnits(recipientReceives, decimals),
    taxDeducted: ethers.formatUnits(taxDeducted, decimals),
    taxApplies
  };
}
```

#### 2. Executing Gasless Approval (EIP-2612 Permit)
This allows users to approve a smart contract (e.g. Staking or Vesting) using an off-chain signature instead of spending gas on `approve()`.
```typescript
import { ethers } from "ethers";

async function signTokenPermit(
  signer: ethers.JsonRpcSigner,
  tokenAddress: string,
  spenderAddress: string,
  amountFormatted: string
) {
  const tokenABI = [
    "function name() view returns (string)",
    "function nonces(address owner) view returns (uint256)",
    "function DOMAIN_SEPARATOR() view returns (bytes32)"
  ];
  
  const tokenContract = new ethers.Contract(tokenAddress, tokenABI, signer);
  
  const ownerAddress = await signer.getAddress();
  const tokenName = await tokenContract.name();
  const nonce = await tokenContract.nonces(ownerAddress);
  const amount = ethers.parseUnits(amountFormatted, 18);
  const deadline = Math.floor(Date.now() / 1000) + 3600; // 1 hour from now

  const network = await signer.provider.getNetwork();
  const chainId = network.chainId;

  // Domain specification conforming to EIP-712
  const domain = {
    name: tokenName,
    version: "1", // Upgradeable contracts generally default version to '1' or package name
    chainId: Number(chainId),
    verifyingContract: tokenAddress
  };

  const types = {
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" }
    ]
  };

  const value = {
    owner: ownerAddress,
    spender: spenderAddress,
    value: amount,
    nonce: nonce,
    deadline: deadline
  };

  // Trigger wallet signature request
  const signatureHex = await signer.signTypedData(domain, types, value);
  const sig = ethers.Signature.from(signatureHex);

  return {
    owner: ownerAddress,
    spender: spenderAddress,
    value: amount.toString(),
    deadline,
    v: sig.v,
    r: sig.r,
    s: sig.s
  };
}
```

---

### Backend Event Listener & Indexer

```javascript
const { ethers } = require("ethers");

async function indexTokenTransfers(tokenAddress, providerUrl) {
  const provider = new ethers.JsonRpcProvider(providerUrl);
  const abi = [
    "event Transfer(address indexed from, address indexed to, uint256 value)"
  ];
  
  const contract = new ethers.Contract(tokenAddress, abi, provider);
  
  contract.on("Transfer", async (from, to, value, event) => {
    console.log(`[Transfer] ${ethers.formatUnits(value, 18)} LOONY from ${from} to ${to}`);
    
    // Example: Detect if transfer is a tax deduction or a standard user action
    // Backend DB can index exact movement history for portfolio tracking.
  });
}
```
