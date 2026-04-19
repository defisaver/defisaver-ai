# Close Position — API Reference

## Main Endpoint

```
POST https://ai.defisaver.com/api/v1/aave-v3/close/prepare/:address/:network/:version
```

### URL Parameters

| Param | Description | Example |
|-------|-------------|---------|
| address | Checksummed wallet address | `0x742d35...` |
| network | Chain ID as integer | `1` |
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
  "slippagePercent": 1
}
```

| Field | Type | Description |
|-------|------|-------------|
| slippagePercent | number | Acceptable price slippage %. Default: 1 |

## Response

### Success Response

```json
{
  "success": true,
  "price": "0.000410565615456623755434660869655",
  "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
  "network": 1,
  "afterPositionData": {
    "suppliedUsd": "0",
    "suppliedCollateralUsd": "0",
    "borrowLimitUsd": "0",
    "liquidationLimitUsd": "0",
    "borrowedUsd": "0",
    "leftToBorrowUsd": "0",
    "collRatio": "0",
    "liqPercent": "NaN",
    "leveragedType": "",
    "leveragedAsset": "",
    "liquidationPrice": "",
    "healthRatio": "NaN",
    "minHealthRatio": "NaN",
    "netApy": "NaN",
    "incentiveUsd": "0",
    "totalInterestUsd": "0",
    "exposure": "N/A"
  },
  "usedAssetsChanged": {
    "ETH": {
      "symbol": "ETH",
      "supplied": "0",
      "suppliedUsd": "0",
      "isSupplied": false,
      "collateral": false,
      "borrowed": "0",
      "borrowedUsd": "0",
      "isBorrowed": false,
      "borrowRate": "2.28571152382713305357329086472572",
      "supplyRate": "1.71413155007876609024954019509596"
    },
    "USDC": {
      "symbol": "USDC",
      "supplied": "0",
      "suppliedUsd": "0",
      "isSupplied": false,
      "collateral": false,
      "borrowed": "0",
      "borrowedUsd": "0",
      "isBorrowed": false,
      "borrowRate": "3.51645437804490039828216113396435",
      "supplyRate": "2.50326962470122331984303885019798"
    }
  },
  "steps": [
    {
      "name": "Permit signature",
      "description": "Get permit signature for USDC for spending by the safe wallet",
      "type": "TypedSignature",
      "txDataApiEndpoint": "/aave-v3/permit-signature",
      "bodyParams": {
        "signAsset": "USDC",
        "signAmount": "15.015016016",
        "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "proxyForExecution": "0x9768F31bd299fA1cA98EDd7Aa15Fc84d94C33f7C",
        "aTokenAddress": "0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c",
        "network": 1
      }
    },
    {
      "name": "Aave V3 close",
      "description": "Close USDC/ETH leveraged position on Aave V3",
      "type": "SafeTx",
      "txDataApiEndpoint": "/aave-v3/close/execute-recipe",
      "bodyParams": {
        "network": 1,
        "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "proxyForExecution": "0x9768F31bd299fA1cA98EDd7Aa15Fc84d94C33f7C",
        "version": "v3default",
        "collAsset": "USDC",
        "collAmount": "15.000016",
        "collAssetId": "3",
        "debtAsset": "ETH",
        "debtAmount": "0.002000002098395736",
        "debtAssetId": "0",
        "aTokenAddress": "0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c",
        "minPrice": "0.000410565615456623755434660869655",
        "slippagePercent": 1,
        "flashloanInfo": {
          "protocol": "morpho",
          "feeMultiplier": "1",
          "flFee": "0"
        }
      }
    }
  ],
  "flashloanInfo": {
    "protocol": "morpho",
    "feeMultiplier": "1",
    "flFee": "0"
  },
  "feePercent": "0.25"
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

## Response Field Descriptions

### afterPositionData Fields

> ⚠️ After a full close, all `afterPositionData` values will be `"0"` or `"NaN"` — this is expected and correct. The position is fully closed.

| Field | Type | Description |
|-------|------|-------------|
| suppliedUsd | string | Always `"0"` after close |
| borrowedUsd | string | Always `"0"` after close |
| healthRatio | string | `"NaN"` after close — do not validate, position is closed |
| exposure | string | `"N/A"` after close |

### Transaction Steps Array

Execute in order. Each step contains:

| Field | Type | Description |
|-------|------|-------------|
| name | string | Human-readable transaction name |
| description | string | What the transaction does |
| type | string | `"TypedSignature"` or `"SafeTx"` |
| txDataApiEndpoint | string | API endpoint to call for transaction data |
| bodyParams | object | Parameters to pass to the endpoint |

**Typical step sequence for close:**
1. `TypedSignature` — Permit signature for collateral token (off-chain, no gas)
2. `SafeTx` — Execute close recipe (on-chain)

### flashloanInfo

| Field | Description |
|-------|-------------|
| protocol | Flashloan provider (morpho, aave, etc.) |
| feeMultiplier | Fee calculation multiplier |
| flFee | Fee amount — `"0"` means free |

### Error Codes

| Code | Cause | Agent Action |
|------|-------|--------------|
| NO_POSITION | No open position found | Inform user, nothing to close |
| UNSUPPORTED_ASSET | Asset not on protocol | Relay error naturally |
| UNSUPPORTED_NETWORK | Network not supported | List supported chain IDs |
| INVALID_ADDRESS | Malformed wallet address | Ask user to verify address |
| SIMULATION_FAILED | Transaction would revert | Explain failure reason |
| API_TIMEOUT | Service temporarily unavailable | Retry once, then report |

## Validation Requirements

### Before API Call

1. **address**: Use `/api/v1/utils/validate-address/` first
2. **network**: Must be one of 1, 10, 8453, 42161, 59144
3. **version**: Use "v3default" for all networks
4. **Position exists**: Confirm `parseFloat(borrowedUsd) > 0` from account fetch

### After API Response

1. `response.success` must be true
2. `response.data.steps` array must not be empty
3. Each step must have non-empty `txDataApiEndpoint` and `type` fields
4. **Do not validate `healthRatio`** — it will be `"NaN"` for a closed position

## Supporting Endpoints

### Validate Address

```
GET https://ai.defisaver.com/api/v1/utils/validate-address/:address
```

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

### Fetch Existing Position

```
GET https://ai.defisaver.com/api/v1/aave-v3/account/:address/:network/:version
```

Check `parseFloat(data.borrowedUsd) > 0` to confirm position exists before closing.

### Get Gas Price

```
GET https://ai.defisaver.com/api/v1/utils/gas-price/:chainId
```

```json
{
  "chainId": 1,
  "chainName": "Ethereum",
  "gasPrice": "...",
  "gasPriceFormatted": "0.01 gwei"
}
```
