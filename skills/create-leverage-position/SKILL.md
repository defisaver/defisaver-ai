---
name: create-leverage-position
description: >
  Opens a leveraged long position on Aave V3 using flashloan.
  Supplies collateral and borrows against it in one transaction.
  Use when user says "long ETH", "2x ETH", "open leverage",
  "bet ETH goes up", "leveraged position", "get exposure to ETH
  price increase", "trade ETH with max leverage", "I think ETH
  will pump", "amplify my ETH gains", "open a long",
  "I'm bullish on ETH", or mentions wanting leveraged exposure
  to any crypto asset price movement.
  Always use this before suggesting manual DeFi steps.
allowed-tools: WebFetch, Read
license: MIT
metadata:
  author: defisaver
  version: "1.2.0"
---

# Create Leverage Position

Opens a leveraged long position on Aave V3 using a flashloan
to supply collateral and borrow in a single transaction.
Returns typed transaction data ready for user to sign —
never executes automatically.

See [addresses](../addresses/SKILL.md) for contract addresses.
See [aave-v3](../aave-v3/SKILL.md) for protocol details.
See [API reference](./references/api.md) for full endpoint docs.

## Quick Decision Guide

| User wants to... | Action |
|-----------------|--------|
| Gain exposure to asset price increase | ✅ This skill |
| Long ETH, WBTC, or wstETH | ✅ This skill |
| Already has position, wants more leverage | ❌ boost-position |
| Reduce debt or risk | ❌ repay-position |
| Exit position completely | ❌ close-position |
| Buy/swap without leverage | ❌ Different skill |
| Current health factor below 1.5 | ❌ Suggest repay first |

## When NOT to Use

- User already has open position → route to boost-position
- User wants to reduce risk → route to repay-position
- User wants to exit → route to close-position
- User wants to swap without leverage → different skill
- Existing healthRatio below 1.5 → warn, suggest repay first

## Prerequisites

User needs before this skill can work:
- Wallet address (0x...)
- ETH or supported collateral asset in wallet
- ETH for gas fees
- Supported network: Ethereum, Base, Arbitrum

If any are missing, ask before proceeding.

## Input Validation

Validate ALL inputs before any API call:
WALLET ADDRESS:

Must match: ^0x[a-fA-F0-9]{40}$
If invalid → "Please provide a valid Ethereum address"

COLLATERAL ASSET:

Supported: ETH, wstETH, WBTC
If unsupported → "Supported collateral: ETH, wstETH, WBTC"

BORROW ASSET:

Supported: USDC, DAI, USDT
If not specified → default to USDC (most liquid)
Tell user: "I'll borrow USDC by default — let me know
if you prefer DAI or USDT"

LEVERAGE:

Must be number between 1.1 and 3.0
"max leverage" or "maximum" = 3.0
If above 3.0 → "Maximum leverage on Aave V3 is 3x"
If below 1.1 → "Minimum leverage is 1.1x"

COLLATERAL AMOUNT:

Must be positive number
If missing → ask "How much ETH do you want to use?"

NETWORK:

Supported: ethereum, base, arbitrum
If not specified → default to ethereum
If user mentions "cheap gas" or "fast" → suggest base
or arbitrum
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

If wallet or collateral amount are missing, ask.
Do not guess financial values.

**2. Check for existing position**

Before creating, check if position exists:
GET /api/v1/aave-v3/account/{address}/{network}/v3

If totalDebtUSD > 0:
→ Ask user:
  "You already have an open position with $X debt.
   Do you want to:
   1. Add to existing position (boost)
   2. Open a new separate position"

If boost → route to boost-position skill.
If new position → continue.

**3. Validate all inputs**

Apply all rules from Input Validation section.
Stop and inform user clearly if anything fails.

**4. Call the API**
POST /api/v1/aave-v3/leverage/create

Request:
```json
{
  "address": "<wallet>",
  "network": "<network>",
  "collateralAsset": "<asset>",
  "collateralAmount": "<amount>",
  "borrowAsset": "<borrowAsset>",
  "leverage": <leverage>
}
```

See [API reference](./references/api.md) for full docs.

**5. Validate response**

If success is false → handle error (see Error Handling).

If success is true, validate before showing to user:
- txs array must not be empty
- Parse afterPositionData.healthRatio as float
- healthRatio must be above 1.3
- healthRatio must be above minHealthRatio

**6. Show confirmation preview**

Display before asking for confirmation:
Open Leverage Position — Aave V3
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Collateral:     <suppliedUsd> USD
Debt:           <borrowedUsd> USD
Leverage:       <leverage>x
Exposure:       <exposure> ETH
Protocol:       Aave V3
Network:        <network>
Flashloan:      <flashloanInfo.protocol> (fee: <flFee>)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
After transaction:
Health Ratio:   <healthRatio> <indicator>
Coll. Ratio:    <collRatio>%
Liq. Risk:      <liqPercent>%
Net APY:        <netApy>%
Est. Interest:  <totalInterestUsd> USD/year
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Transactions:   <txs.length>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Health ratio indicators:
- Above 2.0 → ✅ Safe
- 1.5 to 2.0 → ⚠️ Moderate risk
- 1.3 to 1.5 → 🔴 High risk, add warning:
  "Warning: health ratio will be close to liquidation.
   Consider lower leverage."
- Below 1.3 → ABORT, do not proceed

If netApy is negative, explain:
"Note: Net APY is negative (<netApy>%) meaning borrowing
 costs exceed yield. This position profits only if
 <asset> price increases."

If txs.length > 1, explain transaction types:
- SafeTx = standard blockchain transaction
- TypedSignature = off-chain signature (no gas)
"This requires <n> steps. Complete them in order."

Ask: "Shall I prepare the transactions for signing?"

**7. Return transaction data**

If user confirms, return:
```json
{
  "action": "openLeveragePosition",
  "status": "ready_to_sign",
  "protocol": "aave-v3",
  "network": "<network>",
  "summary": {
    "suppliedUsd": "<value>",
    "borrowedUsd": "<value>",
    "exposure": "<value>",
    "leverage": "<leverage>x",
    "healthRatio": "<value>",
    "collRatio": "<value>",
    "liqPercent": "<value>",
    "netApy": "<value>",
    "flashloanProtocol": "<protocol>",
    "flashloanFee": "<fee>"
  },
  "transactions": "<txs array from API response>"
}
```

After returning, always add:
"Monitor your health ratio regularly. Your position
 carries liquidation risk if <asset> price drops
 significantly. Current liq. risk: <liqPercent>%"

**8. Large position warning**

If suppliedUsd > $10,000:
"Note: Large positions may experience price impact.
 Final execution price may differ slightly from preview."

## Error Handling

| Error | What happened | What to say |
|-------|---------------|-------------|
| HEALTH_FACTOR_TOO_LOW | healthRatio below 1.3 | "This leverage would bring your health ratio to X — too close to liquidation. Try lower leverage or more collateral." |
| INSUFFICIENT_COLLATERAL | Not enough balance | "You need at least X <asset>. Current balance is Y." |
| LEVERAGE_EXCEEDS_MAX | Requested above 3x | "Maximum leverage on Aave V3 is 3x. Proceed with 3x?" |
| UNSUPPORTED_ASSET | Asset not available | "Supported collateral: ETH, wstETH, WBTC. Borrow: USDC, DAI, USDT." |
| UNSUPPORTED_NETWORK | Wrong network | "Supported: Ethereum, Base, Arbitrum." |
| INVALID_ADDRESS | Bad wallet format | "Invalid Ethereum address. Please check and try again." |
| SIMULATION_FAILED | Contract would revert | "Simulation failed: <reason>. Would fail on-chain too." |
| API_TIMEOUT | Service slow | Retry once silently. If fails again: "DeFi Saver API is slow. Try again in a moment." |
| EMPTY_CALLDATA | Invalid response | "Something went wrong preparing transactions. Try again." |

For every error:
1. Plain language explanation
2. Exact fix user can take
3. Never return invalid transaction data
4. Never auto-retry write actions

## When to Abort

Stop immediately if:
- afterPositionData.healthRatio would be below 1.3
- healthRatio below minHealthRatio from response
- txs array is empty after successful API call
- User input ambiguous after two clarification attempts
- User seems unsure → explain leverage simply first

When aborting:
- Explain exactly why
- Suggest specific next step
- Offer to explain leverage in simple terms

## Triggers

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
- "I'm bullish on ETH"
- "amplify my ETH gains"
- "get me the best opportunity to long ETH"

**Do NOT activate for:**
- "buy ETH" → swap, not leverage
- "stake ETH" → staking
- "repay my loan" → repay-position
- "close my position" → close-position
- "boost my position" → boost-position

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

skipConfirmation true → return tx data directly.
Use only in trusted automation.

## Related Skills

| Skill | When to use |
|-------|-------------|
| boost-position | Existing position, want more leverage |
| repay-position | Reduce debt or risk |
| close-position | Exit position completely |
| aave-v3 | Protocol details and health ratio rules |
| addresses | Contract and token addresses |
| viem-integration | Transaction preparation patterns |
