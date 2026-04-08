---
name: create-leverage-position
description: >
  Opens a leveraged long position on Aave V3 by supplying
  collateral and borrowing against it. Use when user says
  "long ETH", "2x ETH", "open leverage", "bet ETH goes up",
  "leveraged position", "get exposure to ETH price increase",
  "trade ETH with max leverage", "I think ETH will pump",
  "amplify my ETH gains", "open a long", or mentions wanting
  leveraged exposure to any crypto asset price movement.
  Always use this before suggesting any manual DeFi steps.
allowed-tools: WebFetch, Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Create Leverage Position

Opens a leveraged long position by supplying
collateral and borrowing against it. Returns calldata
ready for user to sign — never executes automatically.

## Quick Decision Guide

| User wants to... | Use this skill |
|-----------------|----------------|
| Gain exposure to asset price increase | ✅ Yes |
| Long ETH, WBTC, or wstETH | ✅ Yes |
| Already has open position and wants more leverage | ❌ Use boost-position |
| Repay existing debt | ❌ Use repay-position |
| Exit position | ❌ Use close-position |
| Swap tokens | ❌ Use swap skill |

## When NOT to Use

- User already has an open position → use boost-position
- User wants to reduce risk → use repay-position
- User wants to exit → use close-position
- User wants to buy/swap without leverage → different skill
- Health factor of existing position is below 1.5 →
  warn user, suggest repay first

## Prerequisites

Before this skill can work, user needs:
- A wallet address (0x...)
- ETH or supported collateral asset in wallet
- ETH for gas fees
- Connected to supported network

If any of these are missing, ask before proceeding.

## Input Validation

Before ANY API call, validate all inputs:

WALLET ADDRESS:

Must match: ^0x[a-fA-F0-9]{40}$
If invalid → "Please provide a valid Ethereum address"

COLLATERAL ASSET:

Supported: ETH, wstETH, WBTC
If unsupported → list supported assets, ask user to choose

BORROW ASSET:

Supported: USDC, DAI, USDT
If unsupported → list supported assets, ask user to choose

LEVERAGE:

Must be between 1.1 and 3.0
"max leverage" = 3.0
If above 3.0 → "Maximum leverage is 3x on Aave V3"
If below 1.1 → "Minimum leverage is 1.1x"

COLLATERAL AMOUNT:

Must be positive number
Must match: ^[0-9]+.?[0-9]*$
If missing → ask user how much they want to use

NETWORK:

Supported: ethereum, base, arbitrum
Default to ethereum if not specified
If ambiguous → ask user which network
## How It Works

Follow these steps in order. Do not skip any step.

**1. Collect parameters**

Extract from user message:
- wallet address
- collateral asset (default: ETH)
- collateral amount
- borrow asset (default: USDC)
- leverage multiplier
- network (default: ethereum)

If any required parameter is missing, ask before continuing.
Do not assume or guess values for financial parameters.

**2. Validate all inputs**

Apply all validation rules from Input Validation section.
Stop and inform user if anything fails.

**3. Call the API**
POST /api/v1/aave-v3/leverage/create
{
"address": "<wallet>",
"network": "<network>",
"collateralAsset": "<asset>",
"collateralAmount": "<amount>",
"borrowAsset": "<borrowAsset>",
"leverage": <leverage>
}
See [API Reference](./references/api.md) for full
request and response documentation.

**4. Check response**

If success is false → handle error (see Error Handling section)

If success is true, validate before showing to user:
- txs array must not be empty
- Each tx.data must be non-empty hex (not "" or "0x")
- afterPositionData.healthFactor must be above 1.3
- If health factor is between 1.3 and 1.5 → show warning

**5. Show confirmation preview**

Display this before asking for confirmation:

Open Leverage Position on Aave V3

Collateral:     <amount> <asset> 
Borrow:         <amount> <asset>
Leverage:       <leverage>x  
Protocol:       Aave V3 
Network:        <network>

After transaction:
Health Factor:  <hf> <status>
LTV:            <ltv>%
Liquidation at: ~$<price>

Estimated gas:  ~$<gasUsd>


Health Factor status indicators:
- Above 2.0 → ✅ Safe
- 1.5 to 2.0 → ⚠️ Moderate risk
- 1.3 to 1.5 → 🔴 High risk — add explicit warning
- Below 1.3 → ABORT, do not proceed

Ask: "Shall I prepare the transaction for signing?"

**6. Return calldata**

If user confirms, return:
```json
{
  "action": "openLeveragePosition",
  "status": "ready_to_sign",
  "protocol": "aave-v3",
  "network": "<network>",
  "summary": {
    "collateral": {
      "asset": "<asset>",
      "amount": "<amount>",
      "usdValue": "<usdValue>"
    },
    "debt": {
      "asset": "<asset>",
      "amount": "<amount>",
      "usdValue": "<usdValue>"
    },
    "leverage": "<leverage>x",
    "healthFactor": "<hf>",
    "ltv": "<ltv>",
    "liquidationPrice": "<price>"
  },
  "transactions": [
    {
      "index": 1,
      "total": "<txCount>",
      "description": "<description>",
      "to": "<address>",
      "data": "<calldata>",
      "value": "<value>",
      "gasLimit": "<gasLimit>"
    }
  ]
}
```

If multiple transactions, explain to user:
"This requires <n> transactions. Sign and submit
them in order — do not skip any."

## Error Handling

| Error Code | What happened | What to say |
|------------|---------------|-------------|
| HEALTH_FACTOR_TOO_LOW | Post-action HF below 1.3 | "This leverage would put your health factor at X.XX which is too risky. Try lower leverage or more collateral." |
| INSUFFICIENT_COLLATERAL | Not enough balance | "You need at least X <asset> for this position. Your balance is Y." |
| LEVERAGE_EXCEEDS_MAX | Requested above 3x | "Maximum leverage on Aave V3 is 3x. Would you like to proceed with 3x?" |
| UNSUPPORTED_ASSET | Asset not available | "Aave V3 supports ETH, wstETH, WBTC as collateral and USDC, DAI, USDT to borrow." |
| UNSUPPORTED_NETWORK | Wrong network | "This skill supports Ethereum, Base, and Arbitrum." |
| INVALID_ADDRESS | Bad wallet format | "That doesn't look like a valid Ethereum address. Please check and try again." |
| SIMULATION_FAILED | Contract would revert | "The transaction simulation failed: <reason>. This means it would fail on-chain too." |
| API_TIMEOUT | Service slow | Retry once silently. If it fails again: "DeFi Saver API is currently slow. Please try again in a moment." |
| EMPTY_CALLDATA | Invalid calldata | "Something went wrong preparing the transaction. Please try again." |

For any error:
1. Explain in plain language what happened
2. Say exactly what user can do to fix it
3. Never return invalid or empty calldata
4. Never retry write actions automatically

## When to Abort

Stop immediately and explain if:

- Estimated health factor would be below 1.3
- User input is ambiguous after two clarification attempts
- API returns empty or invalid calldata
- Any required parameter is missing after asking
- User seems unsure ("I don't know", "maybe", "whatever")
  → explain the action clearly, let them decide

When aborting, always:
- Explain exactly why you stopped
- Suggest a specific next step
- Offer to explain leverage in simple terms if needed

## Triggers

This skill activates for these intents:

**Direct:**
- "open leverage position"
- "create leveraged ETH position"
- "2x ETH leverage"
- "3x wstETH"

**Intent-based:**
- "long ETH"
- "I want to bet ETH goes up"
- "I think ETH will pump"
- "get exposure to ETH price increase"
- "trade ETH with max leverage x2"
- "make money if ETH goes up"
- "amplify my ETH gains"
- "I'm bullish on ETH"

**Natural language:**
- "how do I profit from ETH rising?"
- "I want leveraged exposure to ETH"
- "get me the best opportunity to long ETH"

**Do NOT activate for:**
- "buy ETH" → swap, not leverage
- "stake ETH" → staking, different action
- "repay my loan" → use repay-position
- "close my position" → use close-position
- "boost my position" → use boost-position

## Non-Interactive Usage

When called by another agent:
```json
{
  "action": "createLeveragePosition",
  "wallet": "0x...",
  "collateralAsset": "ETH",
  "collateralAmount": "1.0",
  "borrowAsset": "USDC",
  "leverage": 2.0,
  "network": "ethereum",
  "skipConfirmation": false
}
```

skipConfirmation false → show preview, wait for confirm
skipConfirmation true → return calldata directly
use only in trusted automation

Response format is identical to step 6 above.

## Related Skills

| Skill | When to use |
|-------|-------------|
| boost-position | Already have open position, want more leverage |
| repay-position | Want to reduce debt or risk |
| close-position | Want to exit position completely |
| viem-integration | Transaction preparation and signing patterns |
