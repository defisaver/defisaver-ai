# Repay Position — API Reference

## Main Endpoint

```
POST https://ai.defisaver.com/api/v1/aave-v3/repay/prepare/:address/:network/:version
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

Include only one target field based on user intent:

```json
{
  "targetHealthRatio": 2.0,
  "slippagePercent": 0.5
}
```

| Field | Type | Description |
|-------|------|-------------|
| targetHealthRatio | number | Target health ratio after repay (e.g. 2.0) |
| targetCollRatio | number | Target collateralization ratio after repay |
| targetSafetyRatio | number | Target safety ratio after repay |
| targetExposure | number | Target leverage exposure after repay (e.g. 1.5) |
| slippagePercent | number | Acceptable price slippage %. Default: 0.5 |

Only one target field is required per request.

## Response

### Success Response

```json
{
  "success": true,
  "price": "0.00042247779158978832402269045733",
  "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
  "network": 1,
  "afterPositionData": {
    "suppliedUsd": "14.5114727076",
    "suppliedCollateralUsd": "14.5114727076",
    "borrowLimitUsd": "10.8836045307",
    "liquidationLimitUsd": "11.318948711928",
    "borrowedUsd": "4.3455254251300465803154036752787080881516617",
    "leftToBorrowUsd": "6.5380791055699534196845963247212919118483383",
    "collRatio": "333.9405776728534892521299560360944",
    "liqPercent": "96.15384615384615384615384615384615",
    "leveragedType": "",
    "leveragedAsset": "",
    "liquidationPrice": "",
    "healthRatio": "2.6047",
    "minHealthRatio": "1.04",
    "netApy": "2.591501463310469685868631611042617",
    "incentiveUsd": "0",
    "totalInterestUsd": "0.26345067258457977",
    "exposure": "1.42"
  },
  "usedAssetsChanged": {
    "ETH": {
      "symbol": "ETH",
      "borrowed": "0.00179422099112073967250000000011507559",
      "borrowedUsd": "4.3455254251300465803154036752787080881516617",
      "isBorrowed": true,
      "borrowRate": "2.28597271920541948050983888885760",
      "supplyRate": "1.71452223820019236024913430656649"
    },
    "USDC": {
      "symbol": "USDC",
      "supplied": "14.512924",
      "suppliedUsd": "14.5114727076",
      "isSupplied": true,
      "collateral": true,
      "borrowed": "0",
      "borrowedUsd": "0",
      "isBorrowed": false,
      "borrowRate": "3.51415258216934113331955877049452",
      "supplyRate": "2.50000951398396863717639741161782"
    }
  },
  "steps": [
    {
      "name": "Permit signature",
      "description": "Get permit signature for USDC for spending by the safe wallet",
      "type": "TypedSignature",
      "txDataApiEndpoint": "/aave-v3/repay/permit-signature",
      "bodyParams": {
        "signAsset": "USDC",
        "signAmount": "0.487077",
        "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "proxyForExecution": "0x9768F31bd299fA1cA98EDd7Aa15Fc84d94C33f7C",
        "aTokenAddress": "0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c",
        "network": 1
      }
    },
    {
      "name": "Aave V3 repay",
      "description": "Repay USDC/ETH leveraged position on Aave V3 to reach target safety ratio of 250%",
      "type": "SafeTx",
      "txDataApiEndpoint": "/aave-v3/repay/execute-recipe",
      "bodyParams": {
        "network": 1,
        "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "proxyForExecution": "0x9768F31bd299fA1cA98EDd7Aa15Fc84d94C33f7C",
        "version": "v3default",
        "collAsset": "USDC",
        "collAmount": "0.487077",
        "collAssetId": "3",
        "debtAsset": "ETH",
        "supplied": "15.000001",
        "borrowed": "0.002000000206414919",
        "debtAssetId": "0",
        "aTokenAddress": "0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c",
        "minPrice": "0.00042247779158978832402269045733",
        "slippagePercent": 0.2,
        "flashloanInfo": null
      }
    }
  ],
  "flashloanInfo": null,
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

| Field | Type | Description | Usage |
|-------|------|-------------|-------|
| suppliedUsd | string | Total collateral value in USD after repay | Display to user |
| borrowedUsd | string | Total debt value in USD after repay | Display to user |
| healthRatio | string | Health factor after repay (e.g. "2.6047") | **CRITICAL** — must be > 1.3 |
| minHealthRatio | string | Minimum safe health ratio | Compare against healthRatio |
| collRatio | string | Collateralization ratio % | Display as risk indicator |
| liqPercent | string | Liquidation risk % | Higher = more risk |
| liquidationPrice | string | Asset price at liquidation (empty if no price risk) | Show if not empty |
| netApy | string | Net APY after repay | Show to user |
| totalInterestUsd | string | Estimated interest cost per year | Show to user |
| exposure | string | Leverage exposure after repay | Display to user |

### Health Ratio Validation Rules

| Health Ratio | Status | Action |
|-------------|---------|--------|
| Above 2.0 | ✅ Safe | Proceed normally |
| 1.5 to 2.0 | ⚠️ Moderate | Suggest repaying more |
| 1.3 to 1.5 | 🔴 High risk | Warn strongly, suggest repaying more |
| Below 1.3 | ☠️ Critical | Flag as API error — should not happen after repay |

### Transaction Steps Array

Execute in order. Each step contains:

| Field | Type | Description |
|-------|------|-------------|
| name | string | Human-readable transaction name |
| description | string | What the transaction does |
| type | string | `"TypedSignature"` or `"SafeTx"` |
| txDataApiEndpoint | string | API endpoint to call for transaction data |
| bodyParams | object | Parameters to pass to the endpoint |

**Typical step sequence for repay:**
1. `TypedSignature` — Permit signature for collateral token (off-chain, no gas)
2. `SafeTx` — Execute repay recipe (on-chain)

### Error Codes

| Code | Cause | Agent Action |
|------|-------|--------------|
| HEALTH_FACTOR_TOO_LOW | healthRatio < 1.3 | Should not occur on repay — flag as API error |
| NO_POSITION | No open position found | Inform user, suggest create-leverage-position |
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
4. **target field**: Exactly one of `targetHealthRatio`, `targetCollRatio`, `targetSafetyRatio`, or `targetExposure`
5. **slippagePercent**: Positive number, default 0.5

### After API Response

1. `response.success` must be true
2. `response.data.steps` array must not be empty
3. `response.data.afterPositionData.healthRatio` must be > 1.3 (parse as float)
4. Each step must have non-empty `txDataApiEndpoint` and `type` fields

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

Check `parseFloat(data.borrowedUsd) > 0` to confirm position exists before calling repay.

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
