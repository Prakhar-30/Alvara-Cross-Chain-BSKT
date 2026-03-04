# Alvara × Reactive Integration
## Approach Comparison & Gas Cost Analysis

---

## 1. Overview

This document analyzes two architectural approaches for integrating Alvara Protocol's Basket Token Standard (BSKT) with the Reactive Network to enable multi-chain basket creation across Ethereum, Base, and Arbitrum.

### What We're Solving

Users should be able to create a single BSKT token on Ethereum that holds assets across multiple chains, while:
- Maintaining a single LP token pool on Ethereum
- Keeping the existing user experience (interact only with Ethereum)
- Correctly tracking cross-chain value for LP minting/burning

---

## 2. Reactive Network — Cost Formulas

### 2.1 Reactive Contract (Lasna Testnet)

```
RVM Transaction Fee = BaseFee × GasUsed

Where:
- BaseFee : Base fee per gas unit (block n+1), in REACT/gas  ← UNKNOWN
- GasUsed : Actual gas consumed by RSC logic
- Max gas : 900,000 units per RVM transaction
- Unit     : REACT tokens
```

### 2.2 Callback Contract (Sepolia)

```
Callback Cost = Pbase × C × (Gcallback + K)

Where:
- Pbase     : Base gas price (tx.gasprice or block.basefee)  ← assumed 10 gwei
- C         : Pricing coefficient for Sepolia                ← UNKNOWN
- Gcallback : Gas consumed during callback execution
- K         : Fixed gas surcharge for Sepolia                ← UNKNOWN
- Unit      : ETH
```

### 2.3 Known Constants

| Parameter | Value |
|-----------|-------|
| Event read cost | 0.0043 REACT |
| REACT price | $0.02646 |
| Event read in USD | $0.0001138 |
| Assumed Pbase | 10 gwei (testnet) |
| Assumed ETH price | $2,500 |

> **⚠️ Unknowns:** Lasna BaseFee (B), Sepolia coefficient C, and Sepolia surcharge K must be filled in to get exact USD totals. RVM costs are expressed as `B × gas` until BaseFee is provided.

---

## 3. Architecture Summary

### Approach 1 — Vault-Only / Registry

**Core idea:** Deploy a `MultiChainRegistry` on Ethereum as the source of truth for all cross-chain holdings. BSKTPair is modified to query the registry for value calculation when a basket is flagged as multi-chain. Reactive Network keeps the registry synchronized with actual vault holdings on destination chains.

**Key contracts:**
- `MultiChainRegistry` (Ethereum) — stores token amounts, vault addresses, chain values per BSKT
- `OriginCallback` (Ethereum) — orchestrates operations, updates registry, holds ETH before bridging
- `TokenVault` (Base / Arbitrum) — per-basket secure token storage
- `DestinationCallback` (Base / Arbitrum) — acquires tokens via local DEX, manages vault
- Modified `BSKTPair` — checks `Registry.isMultiChain()` before value calculation

**BSKTPair change required:**
```solidity
function mint(address to) external returns (uint256 liquidity) {
    bool isMultiChain = Registry.isMultiChain(address(bskt));
    uint256 totalValue = isMultiChain
        ? Registry.getTotalValue(address(bskt))   // registry path
        : calculateValueFromBalances();            // standard path
    ...
}
```

---

### Approach 2 — Automated Receipt Token Deployment

**Core idea:** Deploy minimal ERC20 "receipt tokens" on Ethereum (e.g. `rUSDe-Base`) representing assets locked in vaults on destination chains. BSKTPair sees these as normal ERC20s. A `RouterAdapter` intercepts `getAmountsOut()` calls and uses a `PriceOracle` for receipt token pricing instead of Uniswap.

**Key contracts:**
- `ReceiptToken` (Ethereum) — EIP-1167 clone, 1:1 backed by locked asset
- `ReceiptFactory` (Ethereum) — deploys receipt tokens via CREATE2 (deterministic, reusable)
- `ReceiptRegistry` (Ethereum) — maps chainId + underlyingToken → receiptToken
- `PriceOracle` (Ethereum) — stores receipt token prices in WETH, updated by callbacks
- `RouterAdapter` (Ethereum) — wraps Uniswap router, intercepts receipt token price queries
- `OriginCallback` (Ethereum) — mints/burns receipts, updates oracle
- `TokenVault` (Base / Arbitrum) — same as Approach 1

**RouterAdapter intercept:**
```solidity
function getAmountsOut(uint256 amountIn, address[] memory path)
    external view returns (uint256[] memory amounts)
{
    address tokenIn = path[0];
    if (ReceiptRegistry.isReceiptToken(tokenIn) && path[path.length-1] == WETH) {
        uint256 price = Oracle.getPrice(tokenIn);
        require(!Oracle.isPriceStale(tokenIn), "Price stale");
        amounts[path.length - 1] = (amountIn * price) / 1e18;
    } else {
        return IUniswapRouter(realRouter).getAmountsOut(amountIn, path);
    }
}
```

> **⚠️ Architectural mismatch note:** The actual `BSKTPair.calculateShareLP` in the Alvara codebase uses `_totalReservedETH`, which cross-references token amounts against ETH allocations at mint time — it does **not** call `getAmountsOut`. Approach 2's RouterAdapter premise requires a more significant rewrite of BSKTPair's valuation logic than initially described.

---

## 4. Approach Recommendation

### Winner: Approach 1

| Criteria | Approach 1 | Approach 2 |
|----------|-----------|-----------|
| BSKTPair changes required | Minimal (isMultiChain check) | Significant (valuation logic rewrite) |
| Alignment with existing codebase | High — matches ETH-weighted reserve model | Low — requires oracle-based pricing not in current BSKTPair |
| Security surface | Registry + vaults | Oracle staleness + receipt minting + CREATE2 |
| Permanent on-chain artifacts | None beyond registry entries | Permanent receipt ERC20 per cross-chain token pair |
| Gas cost (creation) | Lower | Higher by ~27% (first deploy) |
| Gas cost (contribution) | Lower | Higher |
| Gas cost (rebalance) | Lower | Higher |
| REACT event reads | Fewer | More (price reporting adds events) |
| Audit complexity | Medium | High |
| Oracle dependency | None | Yes — staleness risk for LP minting |

**Approach 2's only advantage** is that receipt tokens are reusable across baskets (saving ~280k gas on repeat creation) and withdrawal is marginally cheaper (~20k gas). Neither justifies the added complexity.

---

## 5. Gas Cost Estimation — Approach 1

### 5.1 BSKT Creation

| Step | Location | Gas / REACT | ETH Cost (est.) |
|------|----------|-------------|-----------------|
| User calls `createMultiChainBSKT` | Ethereum | 250,000 gas | ~$6.25 |
| RSC reads `MultiChainBSKTCreated` | Reactive RVM | 0.0043 REACT + B×80,000 | $0.000114 + B×80k REACT |
| `OriginCB.createBSKT` callback | Ethereum | 800,000 gas | ~$20.00 |
| RSC reads `BSKTCreated` | Reactive RVM | 0.0043 REACT + B×60,000 | $0.000114 + B×60k REACT |
| Bridge RSC coordinates ETH bridging | Reactive RVM | B×120,000 + ~0.003 REACT emit | B×120k REACT |
| `BaseCB.acquireTokens` callback | Base | 200,000 gas | ~$0.50 |
| `ArbCB.acquireTokens` callback | Arbitrum | 150,000 gas | ~$0.30 |
| RSC reads `TokensLocked` × 2 | Reactive RVM | 0.0086 REACT + B×100,000 | $0.000228 + B×100k REACT |
| `OriginCB.reportTokensAcquired` callback | Ethereum | 180,000 gas | ~$4.50 |
| `BSKTPair.mint` (within above) | Ethereum | included above | — |

**Totals:**

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 1,230,000 gas |
| Ethereum callback cost (est.) | ~$30.75 |
| REACT (reads + emissions) | 0.0215 + 0.0085 = **0.0300 REACT** |
| REACT cost in USD | **$0.000794** |
| RVM execution | **B × 360,000 REACT** |

### 5.2 Contribution

| Step | Location | Gas | REACT |
|------|----------|-----|-------|
| `BSKT.contribute` emit | Ethereum | 80,000 | — |
| RSC + `OriginCB.processContribution` | Eth + RVM | 100,000 | 0.0043 + B×50,000 |
| Bridge RSC executes | Reactive RVM | — | B×80,000 |
| `BaseCB.acquireTokens` | Base | 180,000 | — |
| `ArbCB.acquireTokens` | Arbitrum | 130,000 | — |
| RSC reads `TokensLocked` × 2 | Reactive RVM | — | 0.0086 + B×80,000 |
| `OriginCB.reportContribution` callback | Ethereum | 150,000 | 0.002 emit |

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 330,000 gas |
| Ethereum callback cost (est.) | ~$8.25 |
| REACT (reads + emissions) | **0.0250 REACT** |
| REACT cost in USD | **$0.000662** |
| RVM execution | **B × 210,000 REACT** |

### 5.3 Withdrawal

| Step | Location | Gas | REACT |
|------|----------|-----|-------|
| `BSKTPair.withdraw` emit | Ethereum | 60,000 | — |
| RSC + `OriginCB.processWithdrawal` | Eth + RVM | 120,000 | 0.0043 + B×50,000 |
| Bridge RSC | Reactive RVM | — | B×100,000 |
| `BaseCB` vault withdraw + ETH swap | Base | 200,000 | — |
| `ArbCB` vault withdraw + ETH swap | Arbitrum | 180,000 | — |
| RSC reads `ETHReadyForBridge` × 2 | Reactive RVM | — | 0.0086 + B×80,000 |
| Bridge ETH back × 2 | Bridge txs | 160,000 | — |
| `OriginCB.reportWithdrawalComplete` | Ethereum | 180,000 | 0.002 emit |

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 380,000 gas |
| Ethereum callback cost (est.) | ~$9.50 |
| REACT (reads + emissions) | **0.0250 REACT** |
| REACT cost in USD | **$0.000662** |
| RVM execution | **B × 230,000 REACT** |

### 5.4 Rebalance

| Step | Location | Gas | REACT |
|------|----------|-----|-------|
| `BSKT.rebalance` emit | Ethereum | 80,000 | — |
| RSC + `OriginCB.processRebalance` | Eth + RVM | 150,000 | 0.0043 + B×60,000 |
| `BaseCB.executeRebalance` | Base | 250,000 | — |
| RSC reads `TokensRebalanced` × 1-2 | Reactive RVM | — | 0.0086 + B×60,000 |
| `OriginCB.reportRebalanceComplete` | Ethereum | 280,000 | 0.002 emit |

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 510,000 gas |
| Ethereum callback cost (est.) | ~$12.75 |
| REACT (reads + emissions) | **0.0190 REACT** |
| REACT cost in USD | **$0.000503** |
| RVM execution | **B × 120,000 REACT** |

---

## 6. Gas Cost Estimation — Approach 2

### 6.1 BSKT Creation (First Time — No Receipt Reuse)

| Step | Location | Gas / REACT | ETH Cost (est.) |
|------|----------|-------------|-----------------|
| User calls `createMultiChainBSKT` | Ethereum | 250,000 gas | ~$6.25 |
| RSC reads event | Reactive RVM | 0.0043 + B×80,000 | $0.000114 |
| `OriginCB.triggerReceiptDeployment` callback | Ethereum | 320,000 gas | ~$8.00 |
| `OriginCB.createBSKT` callback | Ethereum | 850,000 gas | ~$21.25 |
| RSC reads `BSKTCreated` + Bridge RSC | Reactive RVM | 0.0043 + B×120,000 + 0.004 emit | $0.000114 |
| `BaseCB.acquireTokens` + price report | Base | 230,000 gas | ~$0.55 |
| `ArbCB.acquireTokens` + price report | Arbitrum | 180,000 gas | ~$0.40 |
| RSC reads `TokensLocked` + `PriceReported` × 2 chains | Reactive RVM | 0.0172 REACT + B×150,000 | $0.000455 |
| `OriginCB.reportAcquisitionComplete` callback | Ethereum | 280,000 gas | ~$7.00 |

**Totals (First Time):**

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 1,700,000 gas |
| Ethereum callback cost (est.) | ~$42.50 |
| REACT (reads + emissions) | 0.0301 + 0.012 = **0.0421 REACT** |
| REACT cost in USD | **$0.001114** |
| RVM execution | **B × 550,000 REACT** |

**Totals (Receipt Reuse — `triggerReceiptDeployment` drops to ~40k gas):**

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 1,420,000 gas |
| Ethereum callback cost (est.) | ~$35.50 |
| REACT (reads + emissions) | **0.0421 REACT** (unchanged — price reporting still happens) |
| RVM execution | **B × 550,000 REACT** |

### 6.2 Contribution

| Step | Location | Gas | REACT |
|------|----------|-----|-------|
| `BSKT.contribute` emit | Ethereum | 80,000 | — |
| RSC + `OriginCB.processContribution` | Eth + RVM | 100,000 | 0.0043 + B×50,000 |
| `BaseCB` + `ArbCB` acquire + price report | Base + Arb | 460,000 | — |
| RSC reads `TokensLocked` + `PriceReported` × 2 | Reactive RVM | — | 0.0172 + B×120,000 |
| `OriginCB.reportContribution` (mint receipts + oracle + LP mint) | Ethereum | 220,000 | 0.003 emit |

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 400,000 gas |
| Ethereum callback cost (est.) | ~$10.00 |
| REACT (reads + emissions) | **0.0300 REACT** |
| REACT cost in USD | **$0.000794** |
| RVM execution | **B × 280,000 REACT** |

### 6.3 Withdrawal

| Step | Location | Gas | REACT |
|------|----------|-----|-------|
| `BSKTPair.withdraw` emit | Ethereum | 60,000 | — |
| RSC + `OriginCB.processWithdrawal` (receipt→underlying mapping) | Eth + RVM | 140,000 | 0.0043 + B×50,000 |
| `BaseCB` + `ArbCB` vault withdraw + ETH convert | Base + Arb | 380,000 | — |
| RSC reads `ETHReadyForBridge` × 2 | Reactive RVM | — | 0.0086 + B×80,000 |
| `OriginCB.reportWithdrawalComplete` (burn receipts × 2 + burn LP + ETH transfer) | Ethereum | 220,000 | 0.002 emit |

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 360,000 gas |
| Ethereum callback cost (est.) | ~$9.00 |
| REACT (reads + emissions) | **0.0220 REACT** |
| REACT cost in USD | **$0.000582** |
| RVM execution | **B × 200,000 REACT** |

### 6.4 Rebalance

| Step | Location | Gas | REACT |
|------|----------|-----|-------|
| `BSKT.rebalance` emit | Ethereum | 80,000 | — |
| Deploy new receipt if new token | Ethereum | 160,000 / 20,000 | — |
| RSC + `OriginCB.processRebalance` | Eth + RVM | 150,000 | 0.0043 + B×60,000 |
| `BaseCB.executeRebalance` + `PriceReported` | Base | 270,000 | — |
| RSC reads × 3 events | Reactive RVM | — | 0.0129 + B×80,000 |
| `OriginCB.reportRebalanceComplete` (burn + mint receipts + oracle + BSKT.updateTokens) | Ethereum | 340,000 | 0.003 emit |

| Metric | Value |
|--------|-------|
| Ethereum callback gas (new token) | 570,000 gas |
| Ethereum callback cost (est.) | ~$14.25 |
| REACT (reads + emissions) | **0.0250 REACT** |
| REACT cost in USD | **$0.000662** |
| RVM execution | **B × 160,000 REACT** |

---

## 7. Full Comparison Tables

### 7.1 Ethereum Gas (Callbacks + User Txs)

| Operation | Approach 1 | Approach 2 (First) | Approach 2 (Reuse) | A1 vs A2 (First) |
|-----------|-----------|-------------------|-------------------|-----------------|
| Creation | 1,230,000 | 1,700,000 | 1,420,000 | **A1 saves 470k (−27%)** |
| Contribution | 330,000 | 400,000 | 400,000 | **A1 saves 70k (−17%)** |
| Withdrawal | 380,000 | 360,000 | 360,000 | A2 saves 20k (−5%) |
| Rebalance | 510,000 | 570,000 (new) | 410,000 (existing) | **A1 saves 60k (−10%)** |

### 7.2 REACT Costs (Reads + Emissions, excluding RVM BaseFee)

| Operation | Approach 1 REACT | Approach 1 USD | Approach 2 REACT | Approach 2 USD | Difference |
|-----------|-----------------|----------------|-----------------|----------------|------------|
| Creation | 0.0300 | $0.000794 | 0.0421 | $0.001114 | A2 costs +40% more |
| Contribution | 0.0250 | $0.000662 | 0.0300 | $0.000794 | A2 costs +20% more |
| Withdrawal | 0.0250 | $0.000662 | 0.0220 | $0.000582 | A2 saves 12% |
| Rebalance | 0.0190 | $0.000503 | 0.0250 | $0.000662 | A2 costs +32% more |

### 7.3 RVM Execution Gas (multiply by BaseFee B for REACT cost)

| Operation | Approach 1 RVM Gas | Approach 2 RVM Gas | A1 Saving |
|-----------|-------------------|--------------------|-----------|
| Creation | B × 360,000 | B × 550,000 | **B × 190,000** |
| Contribution | B × 210,000 | B × 280,000 | **B × 70,000** |
| Withdrawal | B × 230,000 | B × 200,000 | A2 saves B × 30,000 |
| Rebalance | B × 120,000 | B × 160,000 | **B × 40,000** |

### 7.4 Estimated USD Summary (at 10 gwei, $2500 ETH, excluding C/K/B unknowns)

| Operation | Approach 1 ETH Cost | Approach 2 ETH Cost | Approach 1 REACT $ | Approach 2 REACT $ |
|-----------|--------------------|--------------------|-------------------|-------------------|
| Creation | ~$30.75 | ~$42.50 | $0.000794 | $0.001114 |
| Contribution | ~$8.25 | ~$10.00 | $0.000662 | $0.000794 |
| Withdrawal | ~$9.50 | ~$9.00 | $0.000662 | $0.000582 |
| Rebalance | ~$12.75 | ~$14.25 | $0.000503 | $0.000662 |

> REACT costs are negligible in USD at current pricing. The dominant cost driver is Ethereum callback gas, where Approach 1 consistently wins.

---

## 8. Unknowns — Complete the Calculation

To get fully precise USD costs, the following values are needed:

| Unknown | Used In | How to Find |
|---------|---------|-------------|
| `BaseFee` (B) on Lasna Testnet | RVM Fee = B × GasUsed | Check Lasna Testnet block explorer or Reactive docs |
| Sepolia coefficient `C` | Callback Cost = Pbase × C × (G + K) | Reactive Network documentation or team |
| Sepolia fixed surcharge `K` | Same formula | Same source |
| Callback emission rate (per byte/word) | Precise emission REACT cost | Reactive documentation |

**Once BaseFee is known:**
```
Approach 1 Creation RVM cost = B × 360,000 REACT = B × 360,000 × $0.02646 USD
Approach 2 Creation RVM cost = B × 550,000 REACT = B × 550,000 × $0.02646 USD
Difference = B × 190,000 × $0.02646 USD saved by Approach 1
```

---

## 9. Final Recommendation

**Use Approach 1.**

- Architecturally aligned with the existing `BSKTPair` ETH-weighted reserve valuation model
- Requires minimal changes to core contracts (one `isMultiChain` branch in BSKTPair)
- No oracle dependency — eliminates staleness risk for LP minting
- No permanent ERC20 token proliferation on Ethereum
- 17–27% cheaper in Ethereum gas on the most frequent operations (creation, contribution)
- 40% fewer REACT events consumed on creation
- Simpler to audit, simpler to reason about, simpler to maintain

Approach 2's receipt token reuse benefit is real but only materializes at scale across many baskets using the same cross-chain token pairs, and the gas saving (~280k on repeat creation) does not offset the architectural complexity, oracle risk, and the fundamental mismatch with BSKTPair's actual valuation logic as currently written.
