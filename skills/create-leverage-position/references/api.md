# Create Leverage Position — API Reference

## Endpoint
POST /api/v1/aave-v3/leverage/create

## Request
```json
{
  "address": "0x...",
  "network": "ethereum",
  "collateralAsset": "ETH",
  "collateralAmount": "1.0",
  "borrowAsset": "USDC",
  "leverage": 2.0
}
```

### Request Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| address | string | yes | User wallet address (0x...) |
| network | string | yes | ethereum, base, arbitrum |
| collateralAsset | string | yes | Asset to use as collateral |
| collateralAmount | string | yes | Amount as decimal string |
| borrowAsset | string | yes | Asset to borrow |
| leverage | number | yes | Multiplier between 1.1 and 3.0 |

### Supported Assets

| Asset | Role | Decimals |
|-------|------|----------|
| ETH | collateral | 18 |
| wstETH | collateral | 18 |
| WBTC | collateral | 8 |
| USDC | borrow | 6 |
| DAI | borrow | 18 |
| USDT | borrow | 6 |

### Supported Networks

| Network | Value |
|---------|-------|
| Ethereum Mainnet | ethereum |
| Base | base |
| Arbitrum | arbitrum |

## Response
```json
{
  "success": true,
  "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
  "network": 1,
  "price": "0.0004558988114161121299484228924268",
  "afterPositionData": {
    "suppliedUsd": "5517.55",
    "suppliedCollateralUsd": "5517.55",
    "borrowLimitUsd": "4440.79",
    "liquidationLimitUsd": "4578.81",
    "borrowedUsd": "3302.73",
    "leftToBorrowUsd": "1138.06",
    "ratio": "134.46",
    "collRatio": "167.06",
    "liqRatio": "0.9698",
    "liqPercent": "96.98",
    "healthRatio": "1.3863",
    "minHealthRatio": "1.031",
    "liquidationPrice": "",
    "netApy": "-0.9048",
    "incentiveUsd": "0",
    "totalInterestUsd": "-20.04",
    "exposure": "2.49",
    "minCollRatio": "124.24",
    "collLiquidationRatio": "120.50"
  },
  "usedAssetsChanged": {
    "ETH": {
      "symbol": "ETH",
      "supplied": "2.503",
      "suppliedUsd": "5502.43",
      "isSupplied": true,
      "collateral": true,
      "borrowed": "0.00253",
      "borrowedUsd": "5.56",
      "isBorrowed": true,
      "interestMode": "2",
      "borrowRate": "2.33",
      "supplyRate": "1.78",
      "limit": "49.10"
    },
    "USDC": {
      "symbol": "USDC",
      "borrowed": "3297.34",
      "borrowedUsd": "3297.16",
      "isBorrowed": true,
      "borrowRate": "3.59",
      "supplyRate": "2.61"
    }
  },
  "flashloanInfo": {
    "protocol": "morpho",
    "feeMultiplier": "1",
    "flFee": "0"
  },
  "txs": [
    { "type": "SafeTx", "to": "0x...", "data": "0x...", "value": "0x0" },
    { "type": "TypedSignature", "domain": {}, "types": {}, "message": {} },
    { "type": "SafeTx", "to": "0x...", "data": "0x...", "value": "0x0" }
  ]
}
```

## Response Fields

### Top Level

| Field | Type | Description |
|-------|------|-------------|
| success | boolean | true if calldata prepared successfully |
| address | string | User wallet address |
| network | number | Chain ID (1=Ethereum, 8453=Base, 42161=Arbitrum) |
| price | string | Execution price in ETH (very small number) |
| afterPositionData | object | Estimated position state after tx |
| usedAssetsChanged | object | Per-asset state after tx (keyed by symbol) |
| flashloanInfo | object | Flashloan details used for leverage |
| txs | array | Transactions to sign in order |

### afterPositionData Fields

| Field | Description | How to use |
|-------|-------------|------------|
| suppliedUsd | Total collateral value in USD | Display to user |
| borrowedUsd | Total debt value in USD | Display to user |
| leftToBorrowUsd | Remaining borrow capacity | Display to user |
| healthRatio | Health factor (e.g. "1.3863") | CRITICAL — check before proceeding |
| minHealthRatio | Minimum safe health ratio | Compare against healthRatio |
| collRatio | Collateralization ratio % | Display as risk indicator |
| liqPercent | Liquidation risk % | Higher = more risk |
| netApy | Net APY of position (can be negative) | Show to user — negative means costs > yield |
| totalInterestUsd | Estimated interest cost | Show to user |
| exposure | Total ETH exposure | Display to user |
| liquidationPrice | Price at liquidation (may be empty string) | Show if not empty |

### healthRatio Rules

| Value | Status | Action |
|-------|--------|--------|
| Above 2.0 | ✅ Safe | Proceed |
| 1.5 to 2.0 | ⚠️ Moderate | Warn user |
| 1.3 to 1.5 | 🔴 High risk | Strong warning, require confirmation |
| Below 1.3 | ☠️ Critical | ABORT — do not proceed |

### usedAssetsChanged Fields

Object keyed by asset symbol (ETH, USDC, USDT, etc.)

| Field | Description |
|-------|-------------|
| supplied | Amount supplied as collateral |
| suppliedUsd | Supplied value in USD |
| borrowed | Amount borrowed |
| borrowedUsd | Borrowed value in USD |
| borrowRate | Current borrow APR % |
| supplyRate | Current supply APR % |
| collateral | Whether used as collateral |
| interestMode | "2" = variable (always) |

### txs Array

Array of typed transaction objects. Sign and submit in order.

| Type | Description |
|------|-------------|
| SafeTx | Standard transaction (to, data, value) |
| TypedSignature | EIP-712 typed signature (domain, types, message) |

May contain 1, 2, or 3 transactions depending on:
- Token approvals needed
- Flashloan structure
- Safe wallet vs EOA

**Always present ALL transactions to user in order.**
**User must sign each one before submitting the next.**

### flashloanInfo

| Field | Description |
|-------|-------------|
| protocol | Flashloan source (morpho, aave, etc.) |
| feeMultiplier | Fee applied to flashloan |
| flFee | Flashloan fee amount |

If flFee is "0" — flashloan is free (Morpho).
Show protocol to user: "Uses Morpho flashloan (no fee)"

## Error Responses
```json
{
  "success": false,
  "error": "HEALTH_FACTOR_TOO_LOW",
  "message": "Estimated health factor 1.21 is below minimum 1.3",
  "details": {
    "estimatedHealthFactor": "1.21",
    "minimumHealthFactor": "1.3"
  }
}
```

### Error Codes

| Code | Cause | Agent Action |
|------|-------|--------------|
| HEALTH_FACTOR_TOO_LOW | Post-action healthRatio < 1.3 | Suggest lower leverage |
| INSUFFICIENT_COLLATERAL | Not enough balance | Show required amount |
| LEVERAGE_EXCEEDS_MAX | Requested > 3x | Suggest max 3x |
| UNSUPPORTED_ASSET | Asset not on Aave V3 | List supported assets |
| UNSUPPORTED_NETWORK | Wrong network | List supported networks |
| INVALID_ADDRESS | Bad wallet format | Ask to check address |
| SIMULATION_FAILED | Contract would revert | Explain reason |
| API_TIMEOUT | Service unavailable | Retry once |

## Validation Before Calling

1. address must match ^0x[a-fA-F0-9]{40}$
2. leverage must be between 1.1 and 3.0
3. collateralAmount must be positive number
4. collateralAsset must be in supported list
5. borrowAsset must be in supported list
6. network must be in supported list

## Validation After Response

1. success must be true
2. txs array must not be empty
3. afterPositionData.healthRatio must be > 1.3
   (compare as float, not string)
4. afterPositionData.healthRatio must be > afterPositionData.minHealthRatio
