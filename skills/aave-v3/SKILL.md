---
name: aave-v3
description: >
  Aave V3 lending protocol reference for DeFi Saver plugin.
  Use when a skill needs Aave V3 protocol details, health factor rules,
  position data, or market information. Internal reference only.
allowed-tools: Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Aave V3 — Protocol Reference

Internal reference skill for Aave V3 protocol details.
Used by other skills that need position data, health factor rules,
or market information.

## Key Concepts

### Health Factor (healthRatio)
- Ratio of collateral value to borrow value, adjusted for LTV
- Above 1.0 = position is solvent
- Below 1.0 = position can be liquidated
- DeFi Saver minimum: 1.3 (skill aborts below this)
- Warning threshold: 1.5

### LTV (Loan-to-Value)
- Maximum borrow capacity relative to collateral
- Example: 80% LTV on $1000 collateral = max $800 borrowable

### Liquidation Threshold
- Point at which position can be liquidated (higher than LTV)
- Example: 85% liquidation threshold

### Interest Rate Modes
- Mode 2 = variable rate (always used for leverage positions)

## Versions

| Version | Networks |
|---------|----------|
| v3default | All networks |

Use `v3default` for all networks.

## Health Factor Rules (enforced by skills)

| Value | Status | Skill Action |
|-------|--------|-------------|
| Above 2.0 | ✅ Safe | Proceed normally |
| 1.5 to 2.0 | ⚠️ Moderate risk | Warn user |
| 1.3 to 1.5 | 🔴 High risk | Strong warning, confirm before proceeding |
| Below 1.3 | ☠️ Critical | ABORT — never proceed |

## Leverage Limits

- Minimum: 1.1x
- Maximum: 3.0x (protocol cap to prevent immediate liquidation risk)

## DeFi Saver API Integration

Use these endpoints to read position data via DeFi Saver API
instead of direct RPC calls.

### Get Account Position
GET /api/v1/aave-v3/account/{address}/{network}/{version}

Returns current position: healthFactor, LTV, collateral, debt.
Use before any write action to check current state.

### Get Combined Position + Market Data
GET /api/v1/aave-v3/position/{address}/{network}/{version}

Returns { account, market } combined.
Use for full position overview.

### Network to Version Mapping

| Network | Chain ID | version |
|---------|----------|---------|
| Ethereum | 1 | v3default |
| Optimism | 10 | v3default |
| Base | 8453 | v3default |
| Arbitrum | 42161 | v3default |
| Linea | 59144 | v3default |

### Key Response Fields

| Field | Type | Description |
|-------|------|-------------|
| healthRatio | string | Parse as float, check > 1.3 |
| borrowedUsd | string | Parse as float, > 0 means active position |
| suppliedUsd | string | Total collateral value in USD |
| leftToBorrowUsd | string | Remaining borrow capacity |
| collRatio | string | Collateralization ratio % |
| liqPercent | string | Liquidation risk % |
| netApy | string | Net APY (can be negative) |
