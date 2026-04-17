# Boost Position — API Reference

## Main Endpoint

```
POST https://ai.defisaver.com/api/v1/aave-v3/boost/prepare/:address/:network/:version
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
  "targetExposure": 3.0,
  "slippagePercent": 0.5
}
```

| Field | Type | Description |
|-------|------|-------------|
| targetHealthRatio | number | Target health ratio after boost (e.g. 1.8) |
| targetCollRatio | number | Target collateralization ratio after boost |
| targetSafetyRatio | number | Target safety ratio after boost |
| targetExposure | number | Target leverage exposure after boost (e.g. 3.0) |
| slippagePercent | number | Acceptable price slippage %. Default: 0.5 |

Only one target field is required per request.

## Response

### Success Response

```json
{
  "success": true,
  "price": "2426.7414242925",
  "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
  "network": 1,
  "afterPositionData": {
    "suppliedUsd": "13.29627691466445807260125225998975",
    "suppliedCollateralUsd": "13.29627691466445807260125225998975",
    "borrowLimitUsd": "9.9722076859983435544509391949923125",
    "liquidationLimitUsd": "10.371095993438277296628976762792005",
    "borrowedUsd": "3.16507229365610872598516",
    "leftToBorrowUsd": "6.8071353923422348284657791949923125",
    "collRatio": "420.0939403916542727877815314479893",
    "liqPercent": "96.15384615384615384615384615384615",
    "leveragedType": "",
    "leveragedAsset": "",
    "liquidationPrice": "",
    "healthRatio": "3.2767",
    "minHealthRatio": "1.04",
    "netApy": "2.570014796032755802698088341233873",
    "incentiveUsd": "0",
    "totalInterestUsd": "0.26037345777626886",
    "exposure": "1.31"
  },
  "usedAssetsChanged": {
    "ETH": {
      "symbol": "ETH",
      "borrowed": "0.001298483121498251",
      "borrowedUsd": "3.16507229365610872598516",
      "isBorrowed": true,
      "borrowRate": "2.28575358327964661867556817888195",
      "supplyRate": "1.71419445831593462083443767799851"
    },
    "USDC": {
      "symbol": "USDC",
      "supplied": "13.2976066753319912717284251025",
      "suppliedUsd": "13.29627691466445807260125225998975",
      "isSupplied": true,
      "collateral": true,
      "borrowed": "0",
      "borrowedUsd": "0",
      "isBorrowed": false,
      "borrowRate": "3.51580430567228746831828252634339",
      "supplyRate": "2.50234868961661946768637667709566"
    }
  },
  "steps": [
    {
      "name": "Delegate signature",
      "description": "Get delegate signature for ETH for spending by the safe wallet",
      "type": "TypedSignature",
      "txDataApiEndpoint": "/aave-v3/delegate-signature",
      "bodyParams": {
        "signAsset": "ETH",
        "signAmount": "0.000701518220122827",
        "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "proxyForExecution": "0x9768F31bd299fA1cA98EDd7Aa15Fc84d94C33f7C",
        "vTokenAddress": "0xeA51d7853EEFb32b6ee06b1C12E6dcCA88Be0fFE",
        "network": 1
      }
    },
    {
      "name": "Aave V3 boost",
      "description": "Boost USDC/ETH leveraged position on Aave V3 to reach target safety ratio of 190%",
      "type": "SafeTx",
      "txDataApiEndpoint": "/aave-v3/boost/execute-recipe",
      "bodyParams": {
        "network": 1,
        "address": "0x21dC459fbA0B1Ea037Cd221D35b928Be1C26141a",
        "proxyForExecution": "0x9768F31bd299fA1cA98EDd7Aa15Fc84d94C33f7C",
        "version": "v3default",
        "collAsset": "USDC",
        "debtAmount": "0.000701518220122827",
        "collAssetId": "3",
        "debtAsset": "ETH",
        "debtAssetId": "0",
        "vTokenAddress": "0xeA51d7853EEFb32b6ee06b1C12E6dcCA88Be0fFE",
        "minPrice": "2426.7414242925",
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
| suppliedUsd | string | Total collateral value in USD after boost | Display to user |
| borrowedUsd | string | Total debt value in USD after boost | Display to user |
| healthRatio | string | Health factor after boost (e.g. "3.2767") | **CRITICAL** — must be > 1.3 |
| minHealthRatio | string | Minimum safe health ratio | Compare against healthRatio |
| collRatio | string | Collateralization ratio % | Display as risk indicator |
| liqPercent | string | Liquidation risk % | Higher = more risk |
| liquidationPrice | string | Asset price at liquidation (empty if no price risk) | Show if not empty |
| netApy | string | Net APY after boost | Show to user |
| totalInterestUsd | string | Estimated interest cost per year | Show to user |
| exposure | string | Leverage exposure after boost | Display to user |

### Health Ratio Validation Rules

| Health Ratio | Status | Action |
|-------------|---------|--------|
| Above 2.0 | ✅ Safe | Proceed normally |
| 1.5 to 2.0 | ⚠️ Moderate | Show warning |
| 1.3 to 1.5 | 🔴 High risk | Strong warning + require confirmation |
| Below 1.3 | ☠️ Critical | **ABORT** — do not proceed |

### Transaction Steps Array

Execute in order. Each step contains:

| Field | Type | Description |
|-------|------|-------------|
| name | string | Human-readable transaction name |
| description | string | What the transaction does |
| type | string | `"TypedSignature"` or `"SafeTx"` |
| txDataApiEndpoint | string | API endpoint to call for transaction data |
| bodyParams | object | Parameters to pass to the endpoint |

**Typical step sequence for boost:**
1. `TypedSignature` — Delegate signature for debt token (off-chain, no gas)
2. `SafeTx` — Execute boost recipe (on-chain)

### Error Codes

| Code | Cause | Agent Action |
|------|-------|--------------|
| HEALTH_FACTOR_TOO_LOW | Post-boost healthRatio < 1.3 | Suggest lower target exposure |
| NO_POSITION | No open position found | Route to create-leverage-position |
| LEVERAGE_EXCEEDS_MAX | Requested > 3x | Suggest maximum 3x |
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
6. **Current healthRatio**: Must be above 1.3 before attempting boost

### After API Response

1. `response.success` must be true
2. `response.data.steps` array must not be empty
3. `response.data.afterPositionData.healthRatio` must be > 1.3 (parse as float)
4. `response.data.afterPositionData.healthRatio` must be > `minHealthRatio`
5. Each step must have non-empty `txDataApiEndpoint` and `type` fields

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

Check `parseFloat(data.borrowedUsd) > 0` to confirm position exists.
Also check current `healthRatio` — must be above 1.3 before boosting.

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
