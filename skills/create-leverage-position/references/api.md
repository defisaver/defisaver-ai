# Create Leverage Position — API Reference

## Main Endpoint

POST https://ai.defisaver.com/api/v1/aave-v3/create/prepare/:address/:network/:version

### URL Parameters

| Param | Description | Example |
|-------|-------------|---------|
| address | Checksummed wallet address | `0x742d35...` |
| network | Chain ID as integer | `8453` |
| version | AaveVersions enum value | `v3default` |

### Network to Version Mapping

| Network | Chain ID | version param |
|---------|----------|---------------|
| Ethereum | 1 | v3default |
| Optimism | 10 | v3default |
| Base | 8453 | v3default |
| Arbitrum | 42161 | v3default |
| Linea | 59144 | v3default |

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

### Success Response

```json
{
  "success": true,
  "data": {
    "afterPositionData": {
      "suppliedUsd": "25.02249286932539208388853552925",
      "suppliedCollateralUsd": "25.02249286932539208388853552925",
      "borrowLimitUsd": "18.7668696519940440629164016469375",
      "liquidationLimitUsd": "19.517544438073805825433057712815",
      "borrowedUsd": "14.998499999999999016179499",
      "leftToBorrowUsd": "3.7683696519940450467369026469375",
      "healthRatio": "1.3012",
      "minHealthRatio": "1.04",
      "collRatio": "166.8333024590818663548272668402476",
      "liqPercent": "96.15384615384615384615384615384615",
      "liquidationPrice": "3049.702052475560379937662934863397",
      "netApy": "2.794610848196383428916865304021716",
      "totalInterestUsd": "0.28013159214859936",
      "exposure": "2.49"
    },
    "usedAssetsChanged": {
      "ETH": {
        "borrowed": "0.00639982008151605",
        "borrowedUsd": "14.998499999999999016179499",
        "symbol": "ETH",
        "isBorrowed": true,
        "borrowRate": "2.29013420611008720582031673683454",
        "supplyRate": "1.72075288622533782741822462786493"
      },
      "USDC": {
        "supplied": "25.0249953688622783117197075",
        "suppliedUsd": "25.02249286932539208388853552925",
        "symbol": "USDC",
        "isSupplied": true,
        "collateral": true,
        "supplyRate": "2.49222719058699435499909605328950"
      }
    },
    "txs": [
      {
        "name": "Approval",
        "description": "Approve USDC for spending by the safe wallet",
        "type": "SafeTx",
        "to": "0xA0b86a33E6441e8C533B126bE2E2d13b78DE13b6",
        "value": "0",
        "data": "0x095ea7b3000000000000000000000000..."
      },
      {
        "name": "Aave V3 leveraged position creation",
        "description": "Create a 2x leveraged position on Aave V3",
        "type": "SafeTx",
        "to": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "value": "0",
        "data": "0x84bb1e42000000000000000000000000..."
      }
    ],
    "flashloanInfo": {
      "protocol": "morpho",
      "feeMultiplier": "1",
      "flFee": "0"
    },
    "price": "2347.72152615",
    "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
    "network": 1,
    "feePercent": "0.25"
  },
  "meta": {
    "timestamp": "2024-01-01T00:00:00.000Z",
    "requestId": "uuid-v4"
  }
}
```

### Error Response

```json
{
  "success": false,
  "error": "ERROR_CODE", 
  "message": "Human readable description",
  "details": {
    "field": "additional context if applicable"
  }
}
```
}
```

### Response Field Descriptions

#### afterPositionData Fields

| Field | Type | Description | Usage |
|-------|------|-------------|-------|
| suppliedUsd | string | Total collateral value in USD | Display to user |
| borrowedUsd | string | Total debt value in USD | Display to user |
| healthRatio | string | Health factor (e.g. "1.3863") | **CRITICAL** — parse as float, must be > 1.3 |
| minHealthRatio | string | Minimum safe health ratio | Compare against healthRatio |
| collRatio | string | Collateralization ratio % | Display as risk indicator |
| liqPercent | string | Liquidation risk % | Higher = more risk |
| liquidationPrice | string | Asset price at liquidation | Show if not empty |
| netApy | string | Net APY (can be negative) | Negative means costs > yield |
| totalInterestUsd | string | Estimated interest cost per year | Show to user |
| exposure | string | Total asset exposure multiplier | Display to user |

#### Health Ratio Validation Rules

| Health Ratio | Status | Action |
|-------------|---------|---------|
| Above 2.0 | ✅ Safe | Proceed normally |
| 1.5 to 2.0 | ⚠️ Moderate | Show warning |
| 1.3 to 1.5 | 🔴 High risk | Strong warning + confirmation |
| Below 1.3 | ☠️ Critical | **ABORT** — do not proceed |

#### Transaction Array (txs)

Execute in order. Each transaction object contains:

| Field | Type | Description |
|-------|------|-------------|
| name | string | Human-readable transaction name |
| description | string | What the transaction does |
| type | string | Always "SafeTx" for blockchain transactions |
| to | string | Contract address to call |
| value | string | ETH value to send (usually "0") |
| data | string | Encoded transaction data |

**Transaction Types by Collateral:**
- **ETH collateral**: 1 transaction (main action)
- **ERC20 collateral**: 2 transactions (approval + main action)

#### flashloanInfo

| Field | Description |
|-------|-------------|
| protocol | Flashloan provider (morpho, aave, etc.) |
| feeMultiplier | Fee calculation multiplier |
| flFee | Fee amount — "0" means free |

### Error Codes

| Code | Cause | Agent Action |
|------|-------|--------------|
| HEALTH_FACTOR_TOO_LOW | healthRatio < 1.3 after transaction | Suggest lower leverage |
| INSUFFICIENT_COLLATERAL | Not enough balance | Show required amount |
| LEVERAGE_EXCEEDS_MAX | Requested > 3x | Suggest maximum 3x |
| UNSUPPORTED_ASSET | Asset not available on protocol | List supported assets |
| UNSUPPORTED_NETWORK | Network not supported | List supported networks |
| INVALID_ADDRESS | Malformed wallet address | Ask user to verify address |
| SIMULATION_FAILED | Transaction would revert | Explain failure reason |
| API_TIMEOUT | Service temporarily unavailable | Retry once, then report |

## Validation Requirements

### Before API Call

1. **address**: Use `/api/v1/utils/validate-address/` first
2. **network**: Must be one of 1, 10, 8453, 42161, 59144
3. **version**: Use "v3default" for all networks
4. **collAmount**: Positive number as string
5. **exposure**: Number > 1 as string (maps from UI leverage multiplier)

### After API Response

1. `response.success` must be true
2. `response.data.steps` array must not be empty
3. `response.data.afterPositionData.healthRatio` must be > 1.3 (parse as float)
4. `response.data.afterPositionData.healthRatio` must be > `minHealthRatio`
5. Each SafeTx must have non-empty `data` field

## Supporting Endpoints

### Validate Address (before any call)

```
GET https://ai.defisaver.com/api/v1/utils/validate-address/:address
```

Response:
```json
{
  "success": true,
  "data": { 
    "address": "0x...", 
    "isValid": true, 
    "checksumAddress": "0x742d35..." 
  }
}
```

Always call this first. Use `checksumAddress` in all subsequent calls.

### Check Existing Position

```
GET https://ai.defisaver.com/api/v1/aave-v3/account/:address/:network/:version
```

Use to check if a position already exists before creating a new one.
Check: `parseFloat(data.borrowedUsd) > 0` → position exists.

### Get Gas Price (for display)

```
GET https://ai.defisaver.com/api/v1/utils/gas-price/:chainId
```

Response:
```json
{ 
  "chainId": 8453, 
  "chainName": "Base", 
  "gasPrice": "...", 
  "gasPriceFormatted": "0.01 gwei" 
}
```
