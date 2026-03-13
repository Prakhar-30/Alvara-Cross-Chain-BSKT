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
- BaseFee : 100,000,000,000 (1e11 wei of REACT = 0.0000001 REACT/gas)
- GasUsed : Actual gas consumed by RSC logic
- Max gas : 900,000 units per RVM transaction
- Unit     : REACT tokens
```

### 2.2 Callback Contract (Sepolia)

```
Callback Cost = Pbase × Gcallback

Where:
- Pbase     : 1,762,615 wei (~0.00176 gwei) — live Sepolia base fee
- C         : 1  ✅
- K         : 0  ✅
- Gcallback : Gas consumed during callback execution
- Unit      : ETH

Simplifies to: standard gas cost — no markup, no surcharge.
```

### 2.3 Known Constants

| Parameter | Value |
|-----------|-------|
| Event read cost | 0.0043 REACT |
| REACT price | $0.02646 |
| Event read in USD | $0.0001138 |
| Lasna BaseFee (B) | **1e11 wei REACT = 1e-7 REACT/gas** ✅ |
| Sepolia base fee (Pbase) | **1,762,615 wei (~0.00176 gwei)** ✅ |
| Sepolia coefficient C | **1** ✅ |
| Sepolia surcharge K | **0** ✅ |
| Assumed ETH price | $2,100 |

> **All unknowns resolved.** Callback cost = Pbase × Gcallback (no markup, no surcharge). Full USD costs are calculable end-to-end.

---

### 2.4 RVM BaseFee Conversion

```
BaseFee = 1e11 wei of REACT per gas unit

REACT has 18 decimals, so:
  1e11 wei REACT = 1e11 / 1e18 REACT = 0.0000001 REACT per gas

Example — Creation (360,000 RVM gas, Approach 1):
  Fee = 360,000 × 1e-7 REACT = 0.036 REACT = 0.036 × $0.02646 = $0.000953
```

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

---

## 5. Gas Cost Estimation — Approach 1

> **RVM costs now use B = 1e-7 REACT/gas.** ETH costs use live Sepolia Pbase = 0.00176 gwei (vs prior assumed 10 gwei), so ETH callback costs are significantly lower than earlier estimates. C and K still unknown — ETH callback totals are shown as `Pbase × C × (G + K)` until resolved.

### 5.1 BSKT Creation

| Step | Location | Gas | RVM REACT Cost |
|------|----------|-----|----------------|
| User calls `createMultiChainBSKT` | Ethereum | 250,000 gas | — |
| RSC reads `MultiChainBSKTCreated` | Reactive RVM | 80,000 RVM gas | 0.0043 + 0.008 = **0.0123 REACT** |
| `OriginCB.createBSKT` callback | Ethereum | 800,000 gas | — |
| RSC reads `BSKTCreated` | Reactive RVM | 60,000 RVM gas | 0.0043 + 0.006 = **0.0103 REACT** |
| Bridge RSC coordinates ETH bridging | Reactive RVM | 120,000 RVM gas | 0.003 emit + 0.012 = **0.015 REACT** |
| `BaseCB.acquireTokens` callback | Base | 200,000 gas | — |
| `ArbCB.acquireTokens` callback | Arbitrum | 150,000 gas | — |
| RSC reads `TokensLocked` × 2 | Reactive RVM | 100,000 RVM gas | 0.0086 + 0.010 = **0.0186 REACT** |
| `OriginCB.reportTokensAcquired` callback | Ethereum | 180,000 gas | — |

**RVM cost breakdown:**
```
Total RVM gas = 80,000 + 60,000 + 120,000 + 100,000 = 360,000 gas
RVM fee       = 360,000 × 1e-7 REACT = 0.036 REACT = $0.000953
Event reads   = 0.0215 REACT = $0.000569
Emissions     = 0.0085 REACT = $0.000225
─────────────────────────────────────────────
Total REACT   = 0.036 + 0.030 = 0.066 REACT = $0.001746
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 1,230,000 gas |
| Ethereum callback cost | `Pbase(0.00176 gwei) × C × (1,230,000 + K)` |
| Total REACT cost | **0.066 REACT = $0.001746** |
| RVM execution component | **0.036 REACT = $0.000953** |

### 5.2 Contribution

| Step | Location | Gas | RVM REACT Cost |
|------|----------|-----|----------------|
| `BSKT.contribute` emit | Ethereum | 80,000 | — |
| RSC + `OriginCB.processContribution` | Eth + RVM | 100,000 | 0.0043 + 0.005 = 0.0093 REACT |
| Bridge RSC executes | Reactive RVM | 80,000 RVM gas | 0.008 REACT |
| `BaseCB.acquireTokens` | Base | 180,000 | — |
| `ArbCB.acquireTokens` | Arbitrum | 130,000 | — |
| RSC reads `TokensLocked` × 2 | Reactive RVM | 80,000 RVM gas | 0.0086 + 0.008 = 0.0166 REACT |
| `OriginCB.reportContribution` callback | Ethereum | 150,000 | 0.002 emit |

```
Total RVM gas = 50,000 + 80,000 + 80,000 = 210,000 gas
RVM fee       = 210,000 × 1e-7 REACT = 0.021 REACT = $0.000556
Event reads + emissions = 0.025 REACT = $0.000662
─────────────────────────────────────────────
Total REACT   = 0.021 + 0.025 = 0.046 REACT = $0.001217
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 330,000 gas |
| Ethereum callback cost | `Pbase(0.00176 gwei) × C × (330,000 + K)` |
| Total REACT cost | **0.046 REACT = $0.001217** |

### 5.3 Withdrawal

| Step | Location | Gas | RVM REACT Cost |
|------|----------|-----|----------------|
| `BSKTPair.withdraw` emit | Ethereum | 60,000 | — |
| RSC + `OriginCB.processWithdrawal` | Eth + RVM | 120,000 | 0.0043 + 0.005 = 0.0093 REACT |
| Bridge RSC | Reactive RVM | 100,000 RVM gas | 0.010 REACT |
| `BaseCB` vault withdraw + ETH swap | Base | 200,000 | — |
| `ArbCB` vault withdraw + ETH swap | Arbitrum | 180,000 | — |
| RSC reads `ETHReadyForBridge` × 2 | Reactive RVM | 80,000 RVM gas | 0.0086 + 0.008 = 0.0166 REACT |
| Bridge ETH back × 2 | Bridge txs | 160,000 | — |
| `OriginCB.reportWithdrawalComplete` | Ethereum | 180,000 | 0.002 emit |

```
Total RVM gas = 50,000 + 100,000 + 80,000 = 230,000 gas
RVM fee       = 230,000 × 1e-7 REACT = 0.023 REACT = $0.000609
Event reads + emissions = 0.025 REACT = $0.000662
─────────────────────────────────────────────
Total REACT   = 0.023 + 0.025 = 0.048 REACT = $0.001270
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 380,000 gas |
| Ethereum callback cost | `Pbase(0.00176 gwei) × C × (380,000 + K)` |
| Total REACT cost | **0.048 REACT = $0.001270** |

### 5.4 Rebalance

| Step | Location | Gas | RVM REACT Cost |
|------|----------|-----|----------------|
| `BSKT.rebalance` emit | Ethereum | 80,000 | — |
| RSC + `OriginCB.processRebalance` | Eth + RVM | 150,000 | 0.0043 + 0.006 = 0.0103 REACT |
| `BaseCB.executeRebalance` | Base | 250,000 | — |
| RSC reads `TokensRebalanced` × 1-2 | Reactive RVM | 60,000 RVM gas | 0.0086 + 0.006 = 0.0146 REACT |
| `OriginCB.reportRebalanceComplete` | Ethereum | 280,000 | 0.002 emit |

```
Total RVM gas = 60,000 + 60,000 = 120,000 gas
RVM fee       = 120,000 × 1e-7 REACT = 0.012 REACT = $0.000318
Event reads + emissions = 0.019 REACT = $0.000503
─────────────────────────────────────────────
Total REACT   = 0.012 + 0.019 = 0.031 REACT = $0.000820
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 510,000 gas |
| Ethereum callback cost | `Pbase(0.00176 gwei) × C × (510,000 + K)` |
| Total REACT cost | **0.031 REACT = $0.000820** |

---

## 6. Gas Cost Estimation — Approach 2

### 6.1 BSKT Creation (First Time — No Receipt Reuse)

```
Total RVM gas = 80,000 + 120,000 + 150,000 = 550,000 gas  (includes extra price-reporting reads)
RVM fee       = 550,000 × 1e-7 REACT = 0.055 REACT = $0.001455
Event reads + emissions = 0.0421 REACT = $0.001114
─────────────────────────────────────────────
Total REACT   = 0.055 + 0.0421 = 0.097 REACT = $0.002568
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 1,700,000 gas |
| Ethereum callback cost | `Pbase(0.00176 gwei) × C × (1,700,000 + K)` |
| Total REACT cost | **0.097 REACT = $0.002568** |

**Receipt Reuse (repeat creation):**

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 1,420,000 gas |
| Ethereum callback cost | `Pbase(0.00176 gwei) × C × (1,420,000 + K)` |
| Total REACT cost | **0.097 REACT = $0.002568** (unchanged — price reporting still fires) |

### 6.2 Contribution

```
Total RVM gas = 50,000 + 120,000 = 280,000 (+ 110,000 extra vs A1 for price reporting)
RVM fee       = 280,000 × 1e-7 REACT = 0.028 REACT = $0.000741
Event reads + emissions = 0.030 REACT = $0.000794
─────────────────────────────────────────────
Total REACT   = 0.058 REACT = $0.001535
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 400,000 gas |
| Total REACT cost | **0.058 REACT = $0.001535** |

### 6.3 Withdrawal

```
Total RVM gas = 50,000 + 80,000 + 70,000 = 200,000 gas
RVM fee       = 200,000 × 1e-7 REACT = 0.020 REACT = $0.000529
Event reads + emissions = 0.022 REACT = $0.000582
─────────────────────────────────────────────
Total REACT   = 0.042 REACT = $0.001111
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 360,000 gas |
| Total REACT cost | **0.042 REACT = $0.001111** |

### 6.4 Rebalance

```
Total RVM gas = 60,000 + 80,000 + 20,000 = 160,000 gas
RVM fee       = 160,000 × 1e-7 REACT = 0.016 REACT = $0.000423
Event reads + emissions = 0.025 REACT = $0.000662
─────────────────────────────────────────────
Total REACT   = 0.041 REACT = $0.001085
```

| Metric | Value |
|--------|-------|
| Ethereum callback gas | 570,000 gas (new token) / 410,000 (existing) |
| Total REACT cost | **0.041 REACT = $0.001085** |

---

## 7. Full Comparison Tables

### 7.1 Ethereum Gas (Callbacks + User Txs)

| Operation | Approach 1 | Approach 2 (First) | Approach 2 (Reuse) | A1 vs A2 (First) |
|-----------|-----------|-------------------|-------------------|-----------------|
| Creation | 1,230,000 | 1,700,000 | 1,420,000 | **A1 saves 470k (−27%)** |
| Contribution | 330,000 | 400,000 | 400,000 | **A1 saves 70k (−17%)** |
| Withdrawal | 380,000 | 360,000 | 360,000 | A2 saves 20k (−5%) |
| Rebalance | 510,000 | 570,000 (new) | 410,000 (existing) | **A1 saves 60k (−10%)** |

### 7.2 Total REACT Costs (RVM Execution + Reads + Emissions)

| Operation | Approach 1 REACT | Approach 1 USD | Approach 2 REACT | Approach 2 USD | Difference |
|-----------|-----------------|----------------|-----------------|----------------|------------|
| Creation | 0.066 | $0.001746 | 0.097 | $0.002568 | A2 costs +47% more |
| Contribution | 0.046 | $0.001217 | 0.058 | $0.001535 | A2 costs +26% more |
| Withdrawal | 0.048 | $0.001270 | 0.042 | $0.001111 | A2 saves 12% |
| Rebalance | 0.031 | $0.000820 | 0.041 | $0.001085 | A2 costs +32% more |

### 7.3 RVM Execution Gas (at B = 1e-7 REACT/gas)

| Operation | Approach 1 RVM Gas | Approach 1 RVM REACT | Approach 2 RVM Gas | Approach 2 RVM REACT | A1 Saving |
|-----------|-------------------|---------------------|--------------------|---------------------|-----------|
| Creation | 360,000 | 0.036 REACT | 550,000 | 0.055 REACT | **0.019 REACT** |
| Contribution | 210,000 | 0.021 REACT | 280,000 | 0.028 REACT | **0.007 REACT** |
| Withdrawal | 230,000 | 0.023 REACT | 200,000 | 0.020 REACT | A2 saves 0.003 REACT |
| Rebalance | 120,000 | 0.012 REACT | 160,000 | 0.016 REACT | **0.004 REACT** |

### 7.4 Estimated USD Summary (Pbase = 0.00176 gwei, $2,500 ETH, C and K unknown)

| Operation | Approach 1 REACT $ | Approach 2 REACT $ | ETH callback formula |
|-----------|-------------------|--------------------|----------------------|
| Creation | $0.001746 | $0.002568 | `0.00176 gwei × C × (G + K)` |
| Contribution | $0.001217 | $0.001535 | `0.00176 gwei × C × (G + K)` |
| Withdrawal | $0.001270 | $0.001111 | `0.00176 gwei × C × (G + K)` |
| Rebalance | $0.000820 | $0.001085 | `0.00176 gwei × C × (G + K)` |

> REACT costs remain small in USD. ETH callback costs depend on C and K — at 0.00176 gwei Pbase vs the old assumed 10 gwei, they will be ~5,700× lower than the old estimates once C and K are confirmed. The relative advantage of Approach 1 in Ethereum gas is unchanged.

---

## 8. All Unknowns Resolved ✅

All parameters are now known. Full per-operation ETH callback costs (C=1, K=0, Pbase=0.00176 gwei, ETH=$2,500):

| Operation | Approach 1 Gas | Approach 1 ETH $ | Approach 2 Gas | Approach 2 ETH $ |
|-----------|---------------|-----------------|----------------|-----------------|
| Creation | 1,230,000 | **$0.00454** | 1,700,000 | **$0.00628** |
| Contribution | 330,000 | **$0.00122** | 400,000 | **$0.00148** |
| Withdrawal | 380,000 | **$0.00140** | 360,000 | **$0.00133** |
| Rebalance | 510,000 | **$0.00188** | 570,000 | **$0.00211** |

**Full lifecycle cost per user (Approach 1 — 1 creation + 3 contributions + 1 withdrawal + 2 rebalances):**
```
ETH callbacks : $0.00454 + (3 × $0.00122) + $0.00140 + (2 × $0.00188) = $0.01336
REACT total   : 0.066 + (3 × 0.046) + 0.048 + (2 × 0.031) = 0.314 REACT = $0.0083
─────────────────────────────────────────────────────────────────────────────
Total Reactive infrastructure cost per user lifecycle = ~$0.0217
```

> Alvara's total Reactive overhead per user is **~$0.022**. A $5–10 crosschain BSKT fee is not covering Reactive costs — those are negligible. The fee would be covering user-side Ethereum mainnet gas and bridging costs, which at mainnet prices (10–30 gwei) dominate entirely.

---

## 9. Final Recommendation

**Use Approach 1.**

- Architecturally aligned with the existing `BSKTPair` ETH-weighted reserve valuation model
- Requires minimal changes to core contracts (one `isMultiChain` branch in BSKTPair)
- No oracle dependency — eliminates staleness risk for LP minting
- No permanent ERC20 token proliferation on Ethereum
- 17–27% cheaper in Ethereum gas on the most frequent operations (creation, contribution)
- 47% lower total REACT cost on creation at live BaseFee (B = 1e-7 REACT/gas)
- Simpler to audit, simpler to reason about, simpler to maintain

Approach 2's receipt token reuse benefit is real but only materializes at scale across many baskets using the same cross-chain token pairs, and the gas saving (~280k on repeat creation) does not offset the architectural complexity, oracle risk, and the fundamental mismatch with BSKTPair's actual valuation logic as currently written.
