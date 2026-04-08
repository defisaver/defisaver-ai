# Create Leverage Position — API Reference

## Endpoint
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
  "address": "0x...",
  "network": "ethereum",
  "price": "1850.25",
  "afterPositionData": {
    "healthFactor": "1.82",
    "ltv": "0.65",
    "totalCollateralUSD": "3200.00",
    "totalDebtUSD": "1600.00",
    "availableBorrowsUSD": "480.00",
    "liquidationThreshold": "0.80",
    "liquidationPriceUSD": "1067.00"
  },
  "usedAssetsChanged": {
    "collateral": {
      "asset": "ETH",
      "amount": "1.0",
      "amountUSD": "3200.00"
    },
    "debt": {
      "asset": "USDC",
      "amount": "1600.0",
      "amountUSD": "1600.00"
    }
  },
  "txs": [
    {
      "to": "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2",
      "data": "0x...",
      "value": "0x0",
      "gasLimit": "350000",
      "description": "Supply ETH and borrow USDC on Aave V3"
    }
  ]
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| success | boolean | true if calldata prepared successfully |
| address | string | User wallet address |
| network | string | Network used |
| price | string | Minimum price with fee (USD) |
| afterPositionData | object | Estimated position state after tx |
| afterPositionData.healthFactor | string | Estimated health factor (> 1.3 required) |
| afterPositionData.ltv | string | Loan to value ratio (0.0 - 1.0) |
| afterPositionData.totalCollateralUSD | string | Total collateral in USD |
| afterPositionData.totalDebtUSD | string | Total debt in USD |
| afterPositionData.availableBorrowsUSD | string | Remaining borrow capacity in USD |
| afterPositionData.liquidationThreshold | string | LTV at which position gets liquidated |
| afterPositionData.liquidationPriceUSD | string | Asset price that triggers liquidation |
| usedAssetsChanged | object | Collateral and debt assets after tx |
| txs | array | Transaction(s) ready for user to sign |
| txs[].to | string | Contract address |
| txs[].data | string | Encoded calldata |
| txs[].value | string | ETH value in hex |
| txs[].gasLimit | string | Estimated gas limit |
| txs[].description | string | Human readable description |

### txs Array

May contain multiple transactions. Common cases:

| Count | Reason |
|-------|--------|
| 1 tx | ETH collateral, no approval needed |
| 2 txs | ERC20 collateral, approval + main tx |

Always present ALL transactions to user in order.
User must sign and submit each one in sequence.

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
| HEALTH_FACTOR_TOO_LOW | Post-action HF < 1.3 | Suggest lower leverage |
| INSUFFICIENT_COLLATERAL | Not enough balance | Show required amount |
| LEVERAGE_EXCEEDS_MAX | Requested > 3x | Suggest max 3x |
| UNSUPPORTED_ASSET | Asset not on Aave V3 | List supported assets |
| UNSUPPORTED_NETWORK | Wrong network | List supported networks |
| INVALID_ADDRESS | Bad wallet format | Ask to check address |
| SIMULATION_FAILED | Contract would revert | Explain reason from details |
| API_TIMEOUT | Service unavailable | Retry once, then inform user |

## Validation Rules

Before calling this endpoint:

1. address must match ^0x[a-fA-F0-9]{40}$
2. leverage must be between 1.1 and 3.0
3. collateralAmount must be positive number
4. collateralAsset must be in supported list
5. borrowAsset must be in supported list
6. network must be in supported list

Before returning txs to user:

1. success must be true
2. txs array must not be empty
3. Each tx.data must be non-empty hex, not "" or "0x"
4. afterPositionData.healthFactor must be > 1.3
