---
name: close-position
description: >
  Fully closes an existing DeFi Saver leveraged position. Repays all debt
  and withdraws all collateral in a single flashloan transaction.

  ACTIVATE when user wants to: close their position completely, exit a long,
  unwind everything, take profits, stop being leveraged, exit DeFi.
  Phrases: "close my position", "exit my long", "unwind my position",
  "I'm done with this position", "take profits", "exit everything",
  "close my ETH long", "I no longer want to be leveraged",
  "sell everything and get my money back", "I'm bearish now, close it",
  "close out my position", "exit my trade", "get me out", "full close",
  "I changed my mind, close the position".

  DO NOT ACTIVATE when: user wants to partially reduce debt → repay-position,
  user wants to reduce leverage but keep a position → repay-position,
  user has no open position.
allowed-tools: WebFetch, Read
compatibility: Requires internet access to https://ai.defisaver.com
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Close Position

Fully exits a leveraged position by repaying all debt and withdrawing all
collateral via a flashloan in a single transaction. Returns unsigned
transaction data — never executes automatically.

## Quick Decision Guide

| User wants to... | Action |
|-----------------|--------|
| Exit the position completely | ✅ This skill |
| Preview what closing would return | ✅ This skill (show preview, ask to confirm) |
| Partially reduce debt but stay in position | ❌ repay-position |
| Reduce leverage but keep some exposure | ❌ repay-position |
| No open position | ❌ Inform user, nothing to close |
| Open a new position | ❌ create-leverage-position |

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
- `network` — default: ethereum, or infer from context

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

Determine `version`:
- chainId 1 → `"AaveV3Ethereum"`
- chainId 10, 8453, 42161, 59144 → `"AaveV3"`

```
GET https://ai.defisaver.com/api/v1/aave-v3/account/{checksumAddress}/{chainId}/{version}
```

If `parseFloat(data.borrowedUsd) === 0` → no open position. Say:
> "You don't have an open position to close."

Show current position clearly before asking:
> "Current position: $X collateral, $Y debt, health ratio Z.
> Closing will repay all $Y debt and return your remaining collateral to your wallet."

If health ratio is very low (below 1.3), add context:
> "Note: your health ratio is low. Closing now may return less collateral
> than expected due to the current position state."

**Step 4 — Confirm intent**

This is a full exit. Always confirm explicitly:
> "This will completely close your position and return your collateral
> (minus debt repayment and fees) to your wallet. Are you sure?"

If user says "simulate" or "show me" without committing → continue to
preview without final confirmation until they explicitly confirm.

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
- Each SafeTx in `txs` must have a non-empty `data` field

**Step 7 — Show confirmation preview**

Fetch gas price:
```
GET https://ai.defisaver.com/api/v1/utils/gas-price/{chainId}
```

Display:
```
Close Position — DeFi Saver
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Debt to repay:    <current borrowedUsd> USD
Collateral:       <current suppliedUsd> USD
Est. returned:    <suppliedUsd - borrowedUsd - fees> USD (approx)
Network:          <network>
Gas Price:        <gasPriceFormatted>
Flashloan:        <flashloanInfo.protocol> (fee: <flashloanInfo.flFee>)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
After closing:
Debt:             $0
Health Ratio:     N/A (position closed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Transactions:     <txs.length>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If `flashloanInfo.flFee` is not "0", note:
> "Flashloan fee: <flFee>. This is deducted from your returned collateral."

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
    "debtRepaid": "<borrowedUsd>",
    "collateralReleased": "<suppliedUsd>",
    "flashloanProtocol": "<flashloanInfo.protocol>",
    "flashloanFee": "<flashloanInfo.flFee>",
    "network": "<network>"
  }
}
```

After returning, always add:
> "Once signed and submitted, your position will be fully closed.
> Your remaining collateral (minus fees) will be returned to your wallet."

## When to Abort

Stop immediately if:
- No existing position found
- User is unsure — offer repay-position as a partial alternative
- `txs` array is empty after successful API response

## Triggers — User Story Scenarios

### Decided to exit
- "Close my ETH position, wallet 0x..."
- "I'm done with this trade, close it"
- "Exit my leveraged position on base"

### Taking profits
- "ETH pumped, I want to take profits and close"
- "I've made enough, close the position"
- "Cash out my leveraged position"

### Changing market view
- "I'm bearish now, close my ETH long"
- "I don't think ETH is going up anymore, get me out"
- "Changed my mind, close the position"
- "I'm less confident, exit everything"

### Worried about risk, wants full exit
- "My health ratio is too low, just close it all"
- "I can't monitor this, close the position"
- "I want to stop worrying about liquidation, just close it"
  → If health ratio is still safe, clarify: "You could also just repay
    some debt and stay in the position. Do you want a full exit or just
    to reduce risk?" Then route accordingly.

### Exploratory
- "What would I get back if I closed my position now?"
- "Show me the numbers if I close"
  → Run full flow through preview, ask: "Would you like to proceed?"

### Do NOT activate for
- "Reduce my debt by $500" → repay-position
- "Lower my leverage to 2x" → repay-position
- "I want to open a new position" → create-leverage-position

## Related Skills

| Skill | When to use |
|-------|-------------|
| repay-position | Reduce debt but stay in position |
| boost-position | Add more leverage |
| create-leverage-position | Open a new leveraged position |
