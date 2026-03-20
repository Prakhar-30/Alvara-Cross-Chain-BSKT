# Alvara × Reactive Integration
## Approach Comparison & Gas Cost Analysis

---

## 0. Summary

**Recommendation: Use Approach 1 (Vault-Only / Registry)**

**Alvara's total Reactive infrastructure cost per user lifecycle is ~$0.008–$0.083**, depending on mainnet gas conditions — essentially negligible.

| Scenario | Mainnet gas price | Alvara cost per user lifecycle |
|----------|-------------------|-------------------------------|
| Current (live) | 0.036 gwei | **~$0.008** |
| 30-day avg | 0.2 gwei | **~$0.009** |
| Network congestion | 10 gwei | **~$0.083** |

> A $5–10 crosschain BSKT fee is **60–1,250× Alvara's actual Reactive overhead** even under congestion. The fee is justified by user-side Ethereum gas and bridging costs — not by Reactive infrastructure, which remains negligible across all realistic gas scenarios.

*Details and derivations follow below.*

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

### 2.1 Reactive Contract (Lasna / RVM)

```
RVM Transaction Fee = BaseFee × GasUsed

Where:
- BaseFee : 1e11 wei REACT = 1e-7 REACT per gas  ✅ (live, from cast base-fee)
- GasUsed : Actual gas consumed by RSC logic
- Max gas : 900,000 units per RVM transaction
- Unit    : REACT tokens
```

### 2.2 Callback Contract (Ethereum Mainnet)

```
Callback Cost = Pbase × Gcallback

Where:
- Pbase     : Ethereum mainnet base fee (variable — see §2.3)
- C         : 1  ✅
- K         : 0  ✅
- Gcallback : Gas consumed during callback execution on mainnet
- Unit      : ETH

Simplifies to: standard Ethereum gas cost — no markup, no surcharge.
```

### 2.3 Known Parameters

| Parameter | Value |
|-----------|-------|
| Lasna RVM BaseFee | 1e-7 REACT/gas |
| Callback coefficient C | 1 |
| Callback surcharge K | 0 |
| Mainnet gas — current (live) | **0.036 gwei** |
| Mainnet gas — 30-day avg | **0.2 gwei** |
| Mainnet gas — congestion scenario | **10 gwei** |
| REACT price | $0.02646 |
| ETH price | $2,100 |
| Event read cost | 0.0043 REACT |

> **All unknowns resolved.** Costs are fully calculable end-to-end.

### 2.4 RVM BaseFee Conversion

```
BaseFee = 1e-7 REACT per gas unit

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

> **⚠️ Architectural mismatch note:** The actual `BSKTPair.calculateShareLP` uses `_totalReservedETH`, which cross-references token amounts against ETH allocations at mint time — it does **not** call `getAmountsOut`. Approach 2's RouterAdapter premise requires a more significant rewrite of BSKTPair's valuation logic than initially described.

---

## 4. Gas Cost Estimation — Approach 1

> All callback costs execute on **Ethereum mainnet**. Three scenarios shown: 0.036 gwei (live), 0.2 gwei (30-day avg), 10 gwei (congestion).

### 4.1 BSKT Creation

| Step | Location | Gas |
|------|----------|-----|
| User calls `createMultiChainBSKT` | Ethereum | 250,000 |
| RSC reads `MultiChainBSKTCreated` | RVM | 80,000 RVM gas |
| `OriginCB.createBSKT` callback | Ethereum mainnet | 800,000 |
| RSC reads `BSKTCreated` | RVM | 60,000 RVM gas |
| Bridge RSC coordinates ETH bridging | RVM | 120,000 RVM gas |
| `BaseCB.acquireTokens` callback | Base | 200,000 |
| `ArbCB.acquireTokens` callback | Arbitrum | 150,000 |
| RSC reads `TokensLocked` × 2 | RVM | 100,000 RVM gas |
| `OriginCB.reportTokensAcquired` callback | Ethereum mainnet | 180,000 |

```
Total RVM gas    = 360,000 → 0.036 REACT = $0.000953
Event reads      = 0.0215 REACT = $0.000569
Emissions        = 0.0085 REACT = $0.000225
Total REACT      = 0.066 REACT = $0.001746

Mainnet callback gas = 1,230,000
At 0.036 gwei → 1,230,000 × 0.036e-9 × $2,100 = $0.000093
At 0.200 gwei → 1,230,000 × 0.200e-9 × $2,100 = $0.000517
At 10.0  gwei → 1,230,000 × 10.0e-9  × $2,100 = $0.025830
```

| Metric | 0.036 gwei (live) | 0.2 gwei (30d avg) | 10 gwei (congestion) |
|--------|-------------------|--------------------|----------------------|
| Mainnet callback cost | $0.000093 | $0.000517 | $0.025830 |
| REACT cost | $0.001746 | $0.001746 | $0.001746 |
| **Total Alvara cost** | **$0.001839** | **$0.002263** | **$0.027576** |

### 4.2 Contribution

| Step | Location | Gas |
|------|----------|-----|
| `BSKT.contribute` emit | Ethereum | 80,000 |
| RSC + `OriginCB.processContribution` | Eth + RVM | 100,000 |
| Bridge RSC executes | RVM | 80,000 RVM gas |
| `BaseCB.acquireTokens` | Base | 180,000 |
| `ArbCB.acquireTokens` | Arbitrum | 130,000 |
| RSC reads `TokensLocked` × 2 | RVM | 80,000 RVM gas |
| `OriginCB.reportContribution` callback | Ethereum mainnet | 150,000 |

```
Mainnet callback gas = 330,000
At 0.036 gwei → $0.000025
At 0.200 gwei → $0.000139
At 10.0  gwei → $0.006930
REACT = 0.046 REACT = $0.001217
```

| Metric | 0.036 gwei (live) | 0.2 gwei (30d avg) | 10 gwei (congestion) |
|--------|-------------------|--------------------|----------------------|
| Mainnet callback cost | $0.000025 | $0.000139 | $0.006930 |
| REACT cost | $0.001217 | $0.001217 | $0.001217 |
| **Total Alvara cost** | **$0.001242** | **$0.001356** | **$0.008147** |

### 5.3 Withdrawal

| Step | Location | Gas |
|------|----------|-----|
| `BSKTPair.withdraw` emit | Ethereum | 60,000 |
| RSC + `OriginCB.processWithdrawal` | Eth + RVM | 120,000 |
| Bridge RSC | RVM | 100,000 RVM gas |
| `BaseCB` vault withdraw + ETH swap | Base | 200,000 |
| `ArbCB` vault withdraw + ETH swap | Arbitrum | 180,000 |
| RSC reads `ETHReadyForBridge` × 2 | RVM | 80,000 RVM gas |
| Bridge ETH back × 2 | Bridge txs | 160,000 |
| `OriginCB.reportWithdrawalComplete` | Ethereum mainnet | 180,000 |

```
Mainnet callback gas = 380,000
At 0.036 gwei → $0.000029
At 0.200 gwei → $0.000160
At 10.0  gwei → $0.007980
REACT = 0.048 REACT = $0.001270
```

| Metric | 0.036 gwei (live) | 0.2 gwei (30d avg) | 10 gwei (congestion) |
|--------|-------------------|--------------------|----------------------|
| Mainnet callback cost | $0.000029 | $0.000160 | $0.007980 |
| REACT cost | $0.001270 | $0.001270 | $0.001270 |
| **Total Alvara cost** | **$0.001299** | **$0.001430** | **$0.009250** |

### 4.4 Rebalance

| Step | Location | Gas |
|------|----------|-----|
| `BSKT.rebalance` emit | Ethereum | 80,000 |
| RSC + `OriginCB.processRebalance` | Eth + RVM | 150,000 |
| `BaseCB.executeRebalance` | Base | 250,000 |
| RSC reads `TokensRebalanced` × 1-2 | RVM | 60,000 RVM gas |
| `OriginCB.reportRebalanceComplete` | Ethereum mainnet | 280,000 |

```
Mainnet callback gas = 510,000
At 0.036 gwei → $0.000039
At 0.200 gwei → $0.000214
At 10.0  gwei → $0.010710
REACT = 0.031 REACT = $0.000820
```

| Metric | 0.036 gwei (live) | 0.2 gwei (30d avg) | 10 gwei (congestion) |
|--------|-------------------|--------------------|----------------------|
| Mainnet callback cost | $0.000039 | $0.000214 | $0.010710 |
| REACT cost | $0.000820 | $0.000820 | $0.000820 |
| **Total Alvara cost** | **$0.000859** | **$0.001034** | **$0.011530** |

---

## 5. Gas Cost Estimation — Approach 2

### 5.1 BSKT Creation

```
Mainnet callback gas = 1,700,000 (first) / 1,420,000 (reuse)
At 0.036 gwei → $0.000129 / $0.000108
At 0.200 gwei → $0.000714 / $0.000596
At 10.0  gwei → $0.035700 / $0.029820
REACT = 0.097 REACT = $0.002568
```

### 5.2 Contribution

```
Mainnet callback gas = 400,000
At 0.036 gwei → $0.000030
At 0.200 gwei → $0.000168
At 10.0  gwei → $0.008400
REACT = 0.058 REACT = $0.001535
```

### 5.3 Withdrawal

```
Mainnet callback gas = 360,000
At 0.036 gwei → $0.000027
At 0.200 gwei → $0.000151
At 10.0  gwei → $0.007560
REACT = 0.042 REACT = $0.001111
```

### 5.4 Rebalance

```
Mainnet callback gas = 570,000 (new token) / 410,000 (existing)
At 0.036 gwei → $0.000043 / $0.000031
At 0.200 gwei → $0.000239 / $0.000172
At 10.0  gwei → $0.011970 / $0.008610
REACT = 0.041 REACT = $0.001085
```

---

## 6. Full Comparison Tables

### 6.1 Ethereum Mainnet Callback Gas

| Operation | Approach 1 | Approach 2 (First) | Approach 2 (Reuse) | A1 vs A2 |
|-----------|-----------|-------------------|-------------------|----------|
| Creation | 1,230,000 | 1,700,000 | 1,420,000 | **A1 saves 470k (−27%)** |
| Contribution | 330,000 | 400,000 | 400,000 | **A1 saves 70k (−17%)** |
| Withdrawal | 380,000 | 360,000 | 360,000 | A2 saves 20k (−5%) |
| Rebalance | 510,000 | 570,000 (new) | 410,000 (existing) | **A1 saves 60k (−10%)** |

### 6.2 Total REACT Costs (RVM + Reads + Emissions)

| Operation | A1 REACT | A1 USD | A2 REACT | A2 USD | Diff |
|-----------|----------|--------|----------|--------|------|
| Creation | 0.066 | $0.001746 | 0.097 | $0.002568 | A2 +47% |
| Contribution | 0.046 | $0.001217 | 0.058 | $0.001535 | A2 +26% |
| Withdrawal | 0.048 | $0.001270 | 0.042 | $0.001111 | A2 −12% |
| Rebalance | 0.031 | $0.000820 | 0.041 | $0.001085 | A2 +32% |

### 6.3 Alvara Total Cost Per User Lifecycle — Approach 1
*(1 creation + 3 contributions + 1 withdrawal + 2 rebalances)*

```
At 0.036 gwei (live):
  Mainnet callbacks : $0.000093 + (3×$0.000025) + $0.000029 + (2×$0.000039) = $0.000275
  REACT             : $0.001746 + (3×$0.001217) + $0.001270 + (2×$0.000820) = $0.007307
  Total             = ~$0.008

At 0.2 gwei (30-day avg):
  Mainnet callbacks : $0.000517 + (3×$0.000139) + $0.000160 + (2×$0.000214) = $0.001522
  REACT             : $0.007307 (unchanged)
  Total             = ~$0.009

At 10 gwei (congestion):
  Mainnet callbacks : $0.025830 + (3×$0.006930) + $0.007980 + (2×$0.010710) = $0.076020
  REACT             : $0.007307 (unchanged)
  Total             = ~$0.083
```

| Gas scenario | Alvara lifecycle cost | vs $5 fee | vs $10 fee |
|---|---|---|---|
| 0.036 gwei (live) | **~$0.008** | 625× headroom | 1,250× headroom |
| 0.2 gwei (30-day avg) | **~$0.009** | 556× headroom | 1,111× headroom |
| 10 gwei (congestion) | **~$0.083** | 60× headroom | 120× headroom |

---

A $5–10 crosschain BSKT fee is not recovering Reactive costs — those are negligible at every gas level seen in the past 30 days. The fee is covering user-side Ethereum mainnet gas complexity, bridging mechanics, and protocol margin, all of which justify it independently.
