# Create Leverage Position — API Reference

## Main Endpoint

POST /api/v1/aave-v3/leveraged-position/:address/:network/:version

### URL Parameters

| Param | Description | Example |
|-------|-------------|---------|
| address | Checksummed wallet address | `0x742d35...` |
| network | Chain ID as integer | `8453` |
| version | AaveVersions enum value | `AaveV3` |

### Network to Version Mapping

| Network | Chain ID | version param |
|---------|----------|---------------|
| Ethereum | 1 | AaveV3Ethereum |
| Optimism | 10 | AaveV3 |
| Base | 8453 | AaveV3 |
| Arbitrum | 42161 | AaveV3 |
| Linea | 59144 | AaveV3 |

### Request Body

```json
{
  "collAsset": "ETH",
  "collAmount": "1.0",
  "debtAsset": "USDC",
  "exposure": "2"
}
```

| Field | Type | Description |
|-------|------|-------------|
| collAsset | string | Collateral asset symbol (e.g. "ETH") |
| collAmount | string | Collateral amount as decimal string |
| debtAsset | string | Borrow asset symbol (e.g. "USDC") |
| exposure | string | Leverage multiplier as string (e.g. "2" for 2x) |

Note: `exposure` is the leverage multiplier — "2" means 2x leverage.
`collAmount` and `exposure` must be strings that parse to valid numbers.

### Validation Rules (from API)

- address: string, exactly 42 characters
- network: integer, min 1
- version: must be valid AaveVersions enum value
- collAsset: string, not empty
- collAmount: string, parses to positive number
- debtAsset: string, not empty
- exposure: string, parses to number > 1

## Response

```json
{
  "success": true,
  "data": {
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
      "exposure": "2.49"
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
        "borrowRate": "2.33",
        "supplyRate": "1.78",
        "interestMode": "2"
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
    "success": true,
    "price": "0.000455...",
    "address": "0x...",
    "network": 8453,
    "flashloanInfo": {
      "protocol": "morpho",
      "feeMultiplier": "1",
      "flFee": "0"
    },
    "txs": [
      { "type": "SafeTx", "to": "0x...", "data": "0x...", "value": "0x0" },
      { "type": "TypedSignature", "domain": {}, "types": {}, "message": {} }
    ]
  },
  "meta": {
    "timestamp": "2024-01-01T00:00:00.000Z",
    "requestId": "uuid"
  }
}
```

### afterPositionData Fields

| Field | Description | How to use |
|-------|-------------|------------|
| suppliedUsd | Total collateral value in USD | Display to user |
| borrowedUsd | Total debt value in USD | Display to user |
| healthRatio | Health factor (e.g. "1.3863") | CRITICAL — parse as float, check > 1.3 |
| minHealthRatio | Minimum safe health ratio | Compare against healthRatio |
| collRatio | Collateralization ratio % | Display as risk indicator |
| liqPercent | Liquidation risk % | Higher = more risk |
| netApy | Net APY (can be negative) | Show to user — negative means costs > yield |
| totalInterestUsd | Estimated interest cost per year | Show to user |
| exposure | Total asset exposure | Display to user |
| liquidationPrice | Price at liquidation (may be empty string) | Show if not empty |

### healthRatio Rules

| Value | Status | Action |
|-------|--------|--------|
| Above 2.0 | ✅ Safe | Proceed |
| 1.5 to 2.0 | ⚠️ Moderate | Warn user |
| 1.3 to 1.5 | 🔴 High risk | Strong warning, require confirmation |
| Below 1.3 | ☠️ Critical | ABORT — do not proceed |

### txs Array

Sign and submit in order.

| Type | Description | Gas required |
|------|-------------|--------------|
| SafeTx | Standard blockchain transaction | Yes |
| TypedSignature | EIP-712 off-chain signature | No |

Order: Always sign and submit in sequence.
TypedSignature requires no gas — sign only, do not submit as tx.

### flashloanInfo

| Field | Description |
|-------|-------------|
| protocol | Flashloan source (morpho, aave, etc.) |
| feeMultiplier | Fee multiplier |
| flFee | Fee amount — "0" means free (Morpho) |

## Supporting Endpoints

### Validate Address (before any call)
GET /api/v1/utils/validate-address/:address

```json
{ "address": "0x...", "isValid": true, "checksumAddress": "0x742d35..." }
```

Always call this first. Use `checksumAddress` in all subsequent calls.

### Check Existing Position
GET /api/v1/aave-v3/account/:address/:network/:version

Use to check if a position already exists before creating a new one.
Check: `parseFloat(data.borrowedUsd) > 0` → position exists.

### Get Gas Price (for display)
GET /api/v1/utils/gas-price/:chainId

```json
{ "chainId": 8453, "chainName": "Base", "gasPrice": "...", "gasPriceFormatted": "0.01 gwei" }
```

## Error Responses

```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human readable description"
}
```

### Error Codes

| Code | Cause | Agent Action |
|------|-------|--------------|
| HEALTH_FACTOR_TOO_LOW | Post-action healthRatio < 1.3 | Suggest lower leverage |
| INSUFFICIENT_COLLATERAL | Not enough balance | Show required amount |
| LEVERAGE_EXCEEDS_MAX | Requested > 3x | Suggest max 3x |
| UNSUPPORTED_ASSET | Asset not on protocol | Relay error naturally |
| UNSUPPORTED_NETWORK | Wrong network | List supported chain IDs |
| INVALID_ADDRESS | Bad wallet format | Ask to check address |
| SIMULATION_FAILED | Contract would revert | Explain reason |
| API_TIMEOUT | Service unavailable | Retry once |

## Validation Before Calling

1. address: use `/api/v1/utils/validate-address/` first
2. network: must be one of 1, 10, 8453, 42161, 59144
3. version: use "AaveV3Ethereum" for chainId 1, "AaveV3" for all others
4. collAmount: positive number as string
5. exposure: number > 1 as string (maps from UI leverage multiplier)

## Validation After Response

1. response.success must be true
2. response.data.txs array must not be empty
3. response.data.afterPositionData.healthRatio must be > 1.3 (parse as float)
4. response.data.afterPositionData.healthRatio must be > minHealthRatio
5. Each SafeTx must have non-empty `data` field
