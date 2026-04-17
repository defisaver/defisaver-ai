---
name: create-leverage-position
description: >
  Opens a new leveraged long position via DeFi Saver using a flashloan —
  supplies collateral and borrows against it in a single transaction.

  ACTIVATE when user wants to: go long on any asset, get leveraged exposure,
  amplify price gains, open a new position with leverage.
  Phrases that trigger this skill: "long ETH", "2x ETH", "3x wstETH on base",
  "I'm bullish on ETH", "ETH is going to pump", "I want more exposure to ETH",
  "bet ETH goes up", "trade with leverage", "get me 2x exposure",
  "I only have 1 ETH but want exposure to 2 ETH worth of price movement",
  "amplify my gains if ETH pumps", "I think WBTC is undervalued",
  "open a leveraged position on optimism", "cheap leverage on base",
  "I'm bullish on wstETH and want yield while being leveraged",
  "maximum leverage", "highest leverage possible", "what would a 2x ETH
  position look like", "simulate a leveraged position for me".

  DO NOT ACTIVATE when: user already has an open position → boost-position,
  user wants to reduce risk or repay debt → repay-position,
  user wants to exit completely → close-position,
  user wants to swap or buy without leverage,
  user wants to stake.
allowed-tools: WebFetch, Read
compatibility: Requires internet access to https://ai.defisaver.com
license: MIT
metadata:
  author: defisaver
  version: "1.1.0"
---

# Create Leverage Position

Opens a leveraged long position via DeFi Saver using a flashloan to supply
collateral and borrow in a single transaction. Returns unsigned transaction
data ready for the user to sign — never executes automatically.

See [API reference](./references/api.md) for endpoints, error codes, and
response field descriptions.

## Quick Decision Guide

| User wants to... | Action |
|-----------------|--------|
| Gain exposure to asset price increase | ✅ This skill |
| Long any supported collateral asset | ✅ This skill |
| Simulate / preview a leveraged position | ✅ This skill (show preview, ask to confirm) |
| Already has a position, wants more leverage | ❌ boost-position |
| Reduce debt or risk | ❌ repay-position |
| Exit position completely | ❌ close-position |
| Buy/swap without leverage | ❌ Different skill |
| Existing healthRatio below 1.5 | ❌ Warn, suggest repay first |

## Prerequisites

- Wallet address (0x...)
- ETH or supported collateral asset in wallet
- ETH for gas fees
- Supported network: Ethereum, Optimism, Base, Arbitrum, Linea

If any are missing, ask before proceeding. Do not guess financial values.

## How It Works

Follow these steps in order. Do not skip any step.

**Step 1 — Collect parameters**

Extract from user message:
- `wallet` — required, must ask if missing
- `collAsset` — collateral asset symbol (default: ETH)
- `collAmount` — collateral amount (required, must ask if missing)
- `debtAsset` — borrow asset (default: most liquid stablecoin on network)
- `leverage` — multiplier (maps to `exposure` in API: 2x → "2", 3x → "3")
- `network` — default: ethereum

Network → chainId mapping:

| Network | chainId |
|---------|---------|
| ethereum | 1 |
| optimism | 10 |
| base | 8453 |
| arbitrum | 42161 |
| linea | 59144 |

Leverage rules:
- Range: 1.1–3.0
- "max" or "maximum" → 3.0
- Above 3.0 → "Maximum leverage is 3x. Shall I use 3x?"
- Below 1.1 → "Minimum leverage is 1.1x."

If user doesn't specify borrow asset, tell them:
"I'll borrow a stablecoin to fund your position."
Never enumerate a fixed list of stablecoins.

If user asks to simulate or preview without committing → run all steps
through Step 6, show preview, then ask: "Would you like to proceed?"

**Step 2 — Validate wallet address**

```
GET https://ai.defisaver.com/api/v1/utils/validate-address/{address}
```

Response: `{ success, data: { address, isValid, checksumAddress } }`

If `data.isValid` is false → stop. "Please provide a valid Ethereum address."
Use `data.checksumAddress` for all subsequent calls.

**Step 3 — Check for existing position**

Use `version: "v3default"` for all networks.

```
GET https://ai.defisaver.com/api/v1/aave-v3/account/{checksumAddress}/{chainId}/v3default
```

If `parseFloat(data.borrowedUsd) > 0` → position exists. Ask:
> "You already have an open position with $X in debt.
> Would you like to (1) add to it — boost-position, or (2) open a new one?"

If boost → route to boost-position. If new → continue.

**Step 4 — Validate remaining inputs**

- Network: must be one of 1, 10, 8453, 42161, 59144. Otherwise stop and list supported networks.
- `collAmount`: must parse to a positive number. Strip commas (e.g. "1,000" → "1000").
- `leverage`: must be 1.1–3.0.
- `collAsset`: if user specified one, proceed — let API reject unsupported assets. Do not enumerate a fixed list.

**Step 5 — Call the API**

```
POST https://ai.defisaver.com/api/v1/aave-v3/create/prepare/{checksumAddress}/{chainId}/v3default
```

Body:
```json
{
  "collAsset": "<symbol, e.g. ETH>",
  "collAmount": "<amount as string, e.g. '1.0'>",
  "debtAsset": "<symbol, e.g. USDC>",
  "exposure": "<leverage as string, e.g. '2'>"
}
```

`exposure` = leverage multiplier. 2x leverage → `"2"`. See [api.md](./references/api.md).

**Step 6 — Validate response**

If `response.success` is false → handle error per [Error Handling](./references/api.md#error-handling).

If `response.success` is true, check before showing preview:
- `response.data.steps` must not be empty
- Parse `response.data.afterPositionData.healthRatio` as float
- `healthRatio` must be above 1.3 — if not, ABORT
- `healthRatio` must be above `response.data.afterPositionData.minHealthRatio`
- Each step in `steps` must have a non-empty `txDataApiEndpoint` and `type` field

**Step 7 — Show confirmation preview**

First fetch gas price:
```
GET https://ai.defisaver.com/api/v1/utils/gas-price/{chainId}
```
Response: `{ success, data: { gasPrice, gasPriceFormatted } }`

Then display:

```
Open Leverage Position — DeFi Saver
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Collateral:       <afterPositionData.suppliedUsd> USD
Debt:             <afterPositionData.borrowedUsd> USD
Leverage:         <afterPositionData.exposure>x
Network:          <network>
Gas Price:        <gasPriceFormatted>
Flashloan:        <flashloanInfo.protocol> (fee: <flashloanInfo.flFee>)
Service Fee:      <feePercent>%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
After transaction:
Health Ratio:     <afterPositionData.healthRatio> <indicator>
Coll. Ratio:      <afterPositionData.collRatio>%
Safety Ratio:     <afterPositionData.ratio>%
Liq. Risk:        <afterPositionData.liqPercent>%
Net APY:          <afterPositionData.netApy>%
Est. Interest:    <afterPositionData.totalInterestUsd> USD/year
Liq. Price:       <afterPositionData.liquidationPrice, if not empty>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Steps:     <steps.length>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Health ratio indicators:
- Above 2.0 → ✅ Safe
- 1.5–2.0 → ⚠️ Moderate risk
- 1.3–1.5 → 🔴 High risk — add: "Warning: health ratio will be close to
  liquidation. Consider using lower leverage or more collateral."
- Below 1.3 → ABORT — do not show preview, do not proceed

If `netApy` is negative:
> "Note: Net APY is <netApy>% — borrowing costs exceed yield.
> This position profits only if <asset> price increases."

If `steps.length > 1`, explain before asking:
> "This requires <n> steps — sign and submit in order."
>
> Step types:
> - ERC20 collateral: first step is a token approval, second is the main action
> - ETH collateral: single step
> - TypedSignature: off-chain signature only, no gas required

If position is large (`suppliedUsd > $10,000`):
> "Note: large positions may experience price impact.
> Final execution price may differ slightly from this preview."

Ask: "Shall I prepare the transactions for signing?"

**Step 8 — Return transaction data**

Map `response.data.steps` to output format:

| Step Field | Maps To | Description |
|------------|---------|-------------|
| step.type | Output transaction type | "SafeTx" → "action" or "approval", "TypedSignature" → "typed_signature" |
| step.name | Transaction name | Human-readable transaction identifier |
| step.description | Transaction description | What the transaction does |
| step.txDataApiEndpoint | API endpoint | Endpoint to call for transaction data |

Transaction type mapping:
- SafeTx (first, if ERC20 collateral) → `"approval"`
- SafeTx (main action or ETH collateral) → `"action"`
- TypedSignature → `"typed_signature"`

ETH collateral → 1 SafeTx → type `"action"`.
ERC20 collateral → 2 SafeTx → first is `"approval"`, second is `"action"`.

Return:
```json
{
  "success": true,
  "transactions": [
    {
      "type": "action",
      "description": "Open leveraged position via DeFi Saver",
      "raw_tx": {
        "chain_id": "<chainId>",
        "to": "<SafeTx.to>",
        "value": "<SafeTx.value>",
        "data": "<SafeTx.data>"
      }
    }
  ],
  "summary": {
    "suppliedUsd": "<afterPositionData.suppliedUsd>",
    "borrowedUsd": "<afterPositionData.borrowedUsd>",
    "leverage": "<afterPositionData.exposure>x",
    "healthRatio": "<afterPositionData.healthRatio>",
    "collRatio": "<afterPositionData.collRatio>",
    "safetyRatio": "<afterPositionData.ratio>",
    "liqPercent": "<afterPositionData.liqPercent>",
    "netApy": "<afterPositionData.netApy>",
    "flashloanProtocol": "<flashloanInfo.protocol>",
    "flashloanFee": "<flashloanInfo.flFee>",
    "serviceFee": "<feePercent>%"
  }
}
```

After returning, always add:
> "Monitor your health ratio regularly. If <asset> price drops significantly,
> your position may be at risk of liquidation. Current liquidation risk: <liqPercent>%.
> Liquidation means a third party can automatically seize and sell your collateral
> to repay the debt."

## When to Abort

Stop immediately and explain why if:
- `healthRatio` below 1.3 after API call
- `healthRatio` below `minHealthRatio`
- `steps` array is empty after a successful API response
- User input is still ambiguous after two clarification attempts
- User seems unsure or unfamiliar with leverage risks

When aborting: explain why, suggest a specific next step (e.g. lower leverage,
more collateral), offer to explain leveraged positions in plain terms.

## Triggers — User Story Scenarios

### Crypto-native, direct intent
- "Long ETH 2x with 1 ETH on base, wallet 0x..."
- "3x wstETH, 0.5 collateral, arbitrum, wallet 0x..."
- "Open a max leverage ETH position on ethereum"

### Bullish sentiment, vague
- "I'm bullish on ETH" → ask for amount, leverage, wallet
- "I think ETH will pump hard" → ask for params
- "Market looks good, I want to leverage up" → ask which asset and how much

### Price prediction framing
- "I think ETH is going to 10k — help me get more exposure"
- "WBTC looks undervalued, I want to bet on it"
- "ETH is at the bottom, I want to 3x my gains"

### Non-DeFi framing (non-crypto-native users)
- "I want to bet on ETH going up"
- "Can I get more than 1x exposure to ETH price?"
- "I have 1 ETH but I want to profit like I have 2 ETH"
- "How do I amplify my ETH gains without buying more?"

### Yield-seeking
- "I want leveraged exposure with a yield-bearing asset"
- "wstETH leverage — I want staking yield while being leveraged"
- "Best way to get leveraged exposure and still earn yield"

### Network / gas conscious
- "Long ETH cheaply" → suggest base or optimism, ask for params
- "Fastest network for leverage" → suggest base or arbitrum
- "Long ETH on optimism, cheap gas"

### Exploratory / simulate
- "What would a 2x ETH long position look like?"
- "Show me the numbers on a 3x ETH position before I decide"
- "Simulate a leveraged position for me, don't execute yet"
  → Run full flow through preview, ask: "Would you like to proceed?"

### Maximum leverage
- "Maximum leverage on ETH, wallet 0x..."
- "Highest leverage possible"
- "Go as leveraged as I can"
  → Use 3.0x, show preview with strong health ratio warning if applicable

### Do NOT activate for
- "Buy ETH" → swap, not leverage
- "Stake ETH" → staking
- "Repay my loan" → repay-position
- "Close my position" → close-position
- "Boost my position" → boost-position
- "Swap ETH for USDC" → different skill

## Related Skills

| Skill | When to use |
|-------|-------------|
| boost-position | Existing position, user wants more leverage |
| repay-position | Reduce debt or risk |
| close-position | Exit position completely |
| addresses | Contract and token addresses per network |
| viem-integration | EVM transaction preparation patterns |
| execute-steps | Transaction execution utility (after user confirmation) |
