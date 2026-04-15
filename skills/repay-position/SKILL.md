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

> **TODO:** Endpoint and request body to be confirmed with backend documentation.

```
POST https://ai.defisaver.com/api/v1/aave-v3/[TBD]/{checksumAddress}/{chainId}/{version}
```

Body:
```json
{
  "TODO": "fields TBD"
}
```

**Step 5 — Validate response**

If `response.success` is false → relay error in plain language.

If `response.success` is true:
- `response.data.txs` must not be empty
- Parse `response.data.afterPositionData.healthRatio` as float
- New `healthRatio` must be above 1.3

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

Map `response.data.txs`:

| API type | Output type | raw_tx fields |
|----------|-------------|---------------|
| SafeTx (first, if ERC20) | `"approval"` | `{ chain_id, to, value, data }` |
| SafeTx (main action) | `"action"` | `{ chain_id, to, value, data }` |
| TypedSignature | `"typed_signature"` | `{ domain, types, message }` |

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
- `txs` array is empty after successful API response

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
