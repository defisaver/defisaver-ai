---
name: repay-position
description: >
  Repays debt on an existing DeFi Saver position to reduce risk and improve
  the health ratio. Uses a flashloan to sell collateral and repay borrowed assets.

  ACTIVATE when user wants to: repay debt, reduce leverage, lower risk, improve
  health ratio, deleverage, partially close a position, avoid liquidation.
  Phrases: "repay my loan", "reduce my risk", "lower my health ratio",
  "I'm worried about liquidation", "reduce my debt", "deleverage my position",
  "my health ratio is too low", "I want to be safer", "partially close my position",
  "reduce my leverage from 3x to 2x", "pay back some debt", "I'm scared of liquidation",
  "health ratio is at 1.4, help me", "take some risk off", "reduce my exposure a bit".

  DO NOT ACTIVATE when: user has no existing position,
  user wants more leverage → boost-position,
  user wants to exit completely → close-position.
allowed-tools: WebFetch, Read
compatibility: Requires internet access to https://ai.defisaver.com
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Repay Position

Reduces debt on an existing position by selling part of the collateral and
repaying borrowed assets via flashloan. Returns unsigned transaction data —
never executes automatically.

See [API reference](./references/api.md) for endpoints, request/response fields,
and error codes.

## Quick Decision Guide

| User wants to... | Action |
|-----------------|--------|
| Reduce debt / improve health ratio | ✅ This skill |
| Reach a target health ratio | ✅ This skill |
| Partially deleverage | ✅ This skill |
| Preview what repaying would look like | ✅ This skill (show preview, ask to confirm) |
| Exit position completely | ❌ close-position |
| Add more leverage | ❌ boost-position |
| No open position | ❌ Inform user, nothing to repay |

## Prerequisites

- Wallet address (0x...)
- An existing open position (borrowedUsd > 0)
- ETH for gas fees (repay sells collateral via flashloan, user needs gas)
- Supported network: Ethereum, Optimism, Base, Arbitrum, Linea

## How It Works

Follow these steps in order.

**Step 1 — Collect parameters**

Extract from user message:
- `wallet` — required, ask if missing
- `network` — default: ethereum
- Repay intent — one of:
  - `repayAmount` — specific amount of debt to repay (e.g. "$500 of debt")
  - `targetHealthRatio` — desired health ratio after repay (e.g. "get me to 2.0")
  - `repayPercent` — percentage of debt to repay (e.g. "repay 50% of my debt")
  - If none specified, ask: "How much would you like to repay?
    You can give me an amount, a target health ratio, or a percentage of your debt."

Network → chainId mapping:

| Network | chainId |
|---------|---------|
| ethereum | 1 |
| optimism | 10 |
| base | 8453 |
| arbitrum | 42161 |
| linea | 59144 |

**Step 2 — Validate wallet address**

```
GET https://ai.defisaver.com/api/v1/utils/validate-address/{address}
```

Response: `{ success, data: { address, isValid, checksumAddress } }`

If `data.isValid` is false → stop. "Please provide a valid Ethereum address."
Use `data.checksumAddress` for all subsequent calls.

**Step 3 — Fetch existing position**

Use `version: "v3default"` for all networks.

```
GET https://ai.defisaver.com/api/v1/aave-v3/account/{checksumAddress}/{chainId}/v3default
```

If `parseFloat(data.borrowedUsd) === 0` → no position exists. Say:
> "You don't have an open position to repay. Would you like to open one?"

Show current position to user:
> "Current position: $X collateral, $Y debt, health ratio Z.
> Liquidation risk: <liqPercent>%."

If health ratio is already comfortable (above 2.0), confirm the user really wants to repay:
> "Your health ratio is already at Z — you're in a safe range.
> Would you still like to repay some debt?"

**Step 4 — Call the API**

```
POST https://ai.defisaver.com/api/v1/aave-v3/repay/prepare/{checksumAddress}/{chainId}/v3default
```

Map user intent to request body — include only the relevant target field:

```json
{
  "targetHealthRatio": "<float, e.g. 2.0>",
  "slippagePercent": 0.5
}
```

| Field | Type | When to use |
|-------|------|-------------|
| targetHealthRatio | number | User gives a target health ratio (e.g. "get me to 2.0") |
| targetCollRatio | number | User gives a target collateral ratio |
| targetSafetyRatio | number | User gives a target safety ratio |
| targetExposure | number | User gives a target leverage (e.g. "reduce to 1.5x") |
| slippagePercent | number | Default: 0.5. Acceptable price slippage % |

Only one target field is needed — use whichever maps best to user intent.
See [api.md](./references/api.md) for full request/response details.

**Step 5 — Validate response**

If `response.success` is false → relay error in plain language.

If `response.success` is true:
- `response.data.steps` must not be empty
- Parse `response.data.afterPositionData.healthRatio` as float
- New `healthRatio` must be above 1.3
- Each step must have a non-empty `txDataApiEndpoint` and `type` field

**Step 6 — Show confirmation preview**

Fetch gas price:
```
GET https://ai.defisaver.com/api/v1/utils/gas-price/{chainId}
```

Display:
```
Repay Position — DeFi Saver
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
New Collateral:   <afterPositionData.suppliedUsd> USD
New Debt:         <afterPositionData.borrowedUsd> USD
Debt Repaid:      <difference from current borrowedUsd> USD
Network:          <network>
Gas Price:        <gasPriceFormatted>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
After repay:
Health Ratio:     <afterPositionData.healthRatio> <indicator>
Coll. Ratio:      <afterPositionData.collRatio>%
Liq. Risk:        <afterPositionData.liqPercent>%
Net APY:          <afterPositionData.netApy>%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Health ratio indicators:
- Above 2.0 → ✅ Safe
- 1.5–2.0 → ⚠️ Moderate risk — note: "Consider repaying more for a safer margin."
- 1.3–1.5 → 🔴 Still high risk — warn: "Health ratio will still be close to
  liquidation after this repay. Consider repaying more."
- Below 1.3 → this should not happen after a repay — flag as API error

Ask: "Shall I prepare the transactions for signing?"

**Step 7 — Return transaction data**

Map `response.data.steps` to output format:

| Step Field | Maps To |
|------------|--------|
| step.type | Transaction type: "SafeTx" → "action", "TypedSignature" → "typed_signature" |
| step.name | Transaction name |
| step.description | Transaction description |
| step.txDataApiEndpoint | API endpoint to call for transaction data |

Return:
```json
{
  "success": true,
  "transactions": [...],
  "summary": {
    "suppliedUsd": "...",
    "borrowedUsd": "...",
    "healthRatio": "...",
    "collRatio": "...",
    "liqPercent": "...",
    "netApy": "..."
  }
}
```

After returning, always add:
> "Your debt has been reduced. New health ratio after signing: <healthRatio>.
> Continue monitoring your position — if <asset> price drops further,
> your risk may increase again."

## When to Abort

Stop immediately if:
- No existing position found
- Post-repay healthRatio unexpectedly below 1.3 (API error)
- `steps` array is empty after successful API response

## Triggers — User Story Scenarios

### Worried about liquidation
- "My health ratio is at 1.35, I'm scared — what do I do?"
  → Show current state, explain risk, suggest repay amount to reach 2.0
- "I don't want to get liquidated, help me"
  → Fetch position, show risk, propose repay to safe level
- "ETH dropped and my position looks risky, help"

### Target health ratio
- "Get my health ratio to 2.0"
- "I want to be at 2.5 health ratio, repay what's needed"
- "Bring me to a safer level"

### Specific amount
- "Repay $500 of my debt, wallet 0x..."
- "Pay back 1000 USDC of my loan"
- "Reduce my debt by half"

### Deleverage intent
- "Reduce my leverage from 3x to 2x"
- "Take some risk off my ETH position"
- "I want less exposure, reduce my position a bit"
- "I'm less bullish now, reduce my leverage"

### Cautious / exploratory
- "What would happen if I repaid $500?"
- "Show me what my position looks like if I reduce it"
  → Run full flow through preview, ask: "Would you like to proceed?"

### Do NOT activate for
- "Close my position entirely" → close-position
- "I want more leverage" → boost-position
- "I don't have a position" → create-leverage-position

## Related Skills

| Skill | When to use |
|-------|-------------|
| close-position | Exit position completely |
| boost-position | Add more leverage to existing position |
| create-leverage-position | Open a new leveraged position |
| execute-steps | Transaction execution utility (after user confirmation) |
