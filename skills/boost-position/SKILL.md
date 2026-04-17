---
name: boost-position
description: >
  Increases leverage on an existing DeFi Saver position using a flashloan.
  Borrows more against existing collateral to buy more of the collateral asset.

  ACTIVATE when user wants to: add leverage to an existing position, increase
  their current exposure, go more bullish on an asset they're already long on.
  Phrases: "boost my position", "increase my leverage", "add more leverage",
  "go more leveraged", "I want more exposure", "top up my ETH long",
  "increase my ETH position from 2x to 3x", "I'm more bullish now, add leverage",
  "my position is at 1.5x, I want 2x", "increase exposure on my existing position".

  DO NOT ACTIVATE when: user has no existing open position → create-leverage-position,
  user wants to add more collateral without increasing leverage,
  user wants to reduce risk → repay-position,
  user wants to exit → close-position.
allowed-tools: WebFetch, Read
compatibility: Requires internet access to https://ai.defisaver.com
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Boost Position

Increases leverage on an existing position by borrowing more against current
collateral and using a flashloan to buy more of the collateral asset.
Returns unsigned transaction data — never executes automatically.

## Quick Decision Guide

| User wants to... | Action |
|-----------------|--------|
| More leverage on existing position | ✅ This skill |
| Preview what higher leverage would look like | ✅ This skill (show preview, ask to confirm) |
| No position yet, wants to open one | ❌ create-leverage-position |
| Reduce risk or repay debt | ❌ repay-position |
| Exit position completely | ❌ close-position |
| Health ratio below 1.5 | ❌ Warn — boosting will lower health ratio further |

## Prerequisites

- Wallet address (0x...)
- An existing open position (borrowedUsd > 0)
- ETH for gas fees
- Supported network: Ethereum, Optimism, Base, Arbitrum, Linea

## How It Works

Follow these steps in order.

**Step 1 — Collect parameters**

Extract from user message:
- `wallet` — required, ask if missing
- `network` — default: ethereum
- `boostAmount` — how much to boost by, OR `targetLeverage` — desired leverage after boost
- If neither is clear, ask: "How much would you like to increase your leverage?
  You can tell me a target (e.g. '3x') or an amount to add."

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
> "You don't have an open position yet. Would you like to open one?"
> If yes → route to create-leverage-position.

Show current position summary before asking what to do:
> "Current position: $X collateral, $Y debt, health ratio Z."

**Step 4 — Safety check before proceeding**

Current `healthRatio` from Step 3:
- Below 1.5 → warn: "Your current health ratio is X. Boosting will lower it
  further. This increases your liquidation risk. Are you sure you want to proceed?"
- Below 1.3 → ABORT. "Health ratio is too close to liquidation. Boost would
  put your position at critical risk. Consider repaying first."

**Step 5 — Call the API**

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

**Step 6 — Validate response**

If `response.success` is false → relay error in plain language.

If `response.success` is true:
- `response.data.txs` must not be empty
- Parse `response.data.afterPositionData.healthRatio` as float
- `healthRatio` must be above 1.3 — if not, ABORT
- `healthRatio` must be above `minHealthRatio`

**Step 7 — Show confirmation preview**

Fetch gas price:
```
GET https://ai.defisaver.com/api/v1/utils/gas-price/{chainId}
```

Display:
```
Boost Position — DeFi Saver
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
New Collateral:   <afterPositionData.suppliedUsd> USD
New Debt:         <afterPositionData.borrowedUsd> USD
Network:          <network>
Gas Price:        <gasPriceFormatted>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
After boost:
Health Ratio:     <afterPositionData.healthRatio> <indicator>
Coll. Ratio:      <afterPositionData.collRatio>%
Liq. Risk:        <afterPositionData.liqPercent>%
Net APY:          <afterPositionData.netApy>%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Health ratio indicators:
- Above 2.0 → ✅ Safe
- 1.5–2.0 → ⚠️ Moderate risk
- 1.3–1.5 → 🔴 High risk — strong warning required
- Below 1.3 → ABORT

Ask: "Shall I prepare the transactions for signing?"

**Step 8 — Return transaction data**

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
> "Your leverage is now higher. Monitor your health ratio closely —
> if <asset> price drops, your liquidation risk increases.
> Current liquidation risk: <liqPercent>%."

## When to Abort

Stop immediately if:
- No existing position found
- Current healthRatio below 1.3
- Post-boost healthRatio below 1.3 or below minHealthRatio
- `txs` array is empty after successful API response

## Triggers — User Story Scenarios

### Clear intent, existing trader
- "Boost my ETH position, wallet 0x..."
- "Increase my leverage to 3x, wallet 0x..."
- "Add more leverage to my wstETH position on base"

### More bullish, wants more exposure
- "ETH is looking really strong, I want to increase my position"
- "I'm more confident now, can I go from 2x to 2.5x?"
- "Market dipped and recovered, I want to add more leverage"

### Specific target leverage
- "I want to be at 3x leverage, I'm currently at 2x"
- "Increase my ETH position exposure from 2 to 2.5"

### Exploratory
- "What would my position look like if I boosted it?"
- "Show me the numbers if I go to 3x leverage"
  → Run full flow through preview, ask: "Would you like to proceed?"

### Do NOT activate for
- "Open a new position" → create-leverage-position
- "Repay my debt" → repay-position
- "Close my position" → close-position
- "My health ratio is too low" → repay-position first

## Related Skills

| Skill | When to use |
|-------|-------------|
| create-leverage-position | No position yet, open a new one |
| repay-position | Reduce debt or risk |
| close-position | Exit position completely |
| execute-steps | Transaction execution utility (after user confirmation) |
