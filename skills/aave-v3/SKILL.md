---
name: aave-v3
description: >
  Aave V3 protocol interactions — read positions, check health
  factor, understand supply/borrow/repay mechanics.
  Use when any skill needs to read or interact with Aave V3
  positions, check account health, or understand protocol limits.
allowed-tools: Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Aave V3

Non-custodial lending/borrowing protocol. Supply assets to
earn yield, borrow against collateral.

Docs: https://aave.com/docs/aave-v3/overview

## Key Concepts

| Concept | Description |
|---------|-------------|
| Supply | Deposit tokens → receive aTokens (interest-bearing) |
| Borrow | Post collateral → borrow tokens → receive variableDebtTokens |
| Health Factor | collateral value / debt value. Below 1 = liquidatable |
| LTV | Loan To Value — how much you can borrow vs collateral |
| aToken | ERC-20 receipt, balance grows automatically with interest |
| variableDebtToken | ERC-20 debt token, balance grows with interest |

## Health Factor Rules

| Health Factor | Status | Action |
|--------------|--------|--------|
| Above 2.0 | ✅ Safe | Proceed normally |
| 1.5 to 2.0 | ⚠️ Moderate | Warn user |
| 1.3 to 1.5 | 🔴 High risk | Strong warning, require confirmation |
| Below 1.3 | ☠️ Critical | Abort, suggest repay first |
| Below 1.0 | 💀 Liquidatable | Abort immediately |

**Health factor < 1 = position gets liquidated.**
This is the most important check before any write action.

## Read Position
```bash
# Get full account data
# Returns: totalCollateralBase, totalDebtBase,
# availableBorrowsBase, currentLiquidationThreshold,
# ltv, healthFactor
cast call $POOL \
  "getUserAccountData(address)(uint256,uint256,uint256,uint256,uint256,uint256)" \
  $USER --rpc-url $RPC
```

Values are in USD with 8 decimals:
- `1e8` = $1.00
- `182000000` = health factor 1.82 (divide by 1e8)

## Interest Rate Mode

Always use `2` (variable rate).
Stable rate (mode `1`) is deprecated in Aave V3.

## Supported Collateral (Ethereum)

| Asset | Max LTV | Liquidation Threshold |
|-------|---------|----------------------|
| ETH/WETH | 80% | 82.5% |
| wstETH | 78.7% | 81% |
| WBTC | 73% | 78% |

## Supported Borrow Assets

| Asset | Notes |
|-------|-------|
| USDC | Most liquid, recommended default |
| USDT | High liquidity |
| DAI | Decentralized stablecoin |

## Liquidation Price Calculation
liquidationPrice = (totalDebtUSD * liquidationThreshold)
/ collateralAmount
Example:
- 1 ETH collateral, $1600 USDC debt, 82.5% threshold
- liquidationPrice = (1600 * 1) / (1 * 0.825) = $1,939

## Common Gotchas

- Health factor < 1 = liquidatable immediately
- `getUserAccountData` returns values in base currency (USD, 8 decimals)
- Interest rate mode: always use `2` (variable). Never use `1` (deprecated)
- Supply auto-enables collateral for most assets
- aToken balance grows over time (includes accrued interest)
- variableDebtToken balance grows over time (includes accrued interest)
- Use `type(uint256).max` for full repay amount

## DeFi Saver API Endpoints

For reading positions use DeFi Saver API instead of direct RPC:
GET /api/v1/aave-v3/account/{address}/{network}/v3
→ Returns health factor, LTV, collateral, debt
GET /api/v1/aave-v3/position/{address}/{network}/v3
→ Returns combined position + market data
See addresses skill for contract addresses by network.
