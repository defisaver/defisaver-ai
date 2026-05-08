---
name: bridging
description: >
  Bridge tokens across chains or bridge+swap into a different destination
  asset using LI.FI quotes and unsigned transactions.

  ACTIVATE when user wants to: bridge assets between chains, move funds to
  another network, transfer a token cross-chain, receive a different token on
  the destination chain, compare bridge routes, or check the status of a
  cross-chain transfer.
  Phrases: "bridge 100 USDC from Ethereum to Base", "move my ETH to arbitrum",
  "send funds from mainnet to base", "swap USDC on Ethereum for ETH on Base",
  "cheapest bridge to move WETH to Optimism", "track my bridge status".

  DO NOT ACTIVATE when: user wants only leverage management →
  create-leverage-position, boost-position, repay-position, or close-position;
  user wants market data only; user wants a same-chain leveraged trade.
allowed-tools: WebFetch, Read
compatibility: Requires LI.FI MCP server configured at https://mcp.li.quest/mcp
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Bridging

Bridge or bridge+swap through the LI.FI MCP server. Use LI.FI tools for all
live chain, token, route, fee, and approval discovery. Do not hardcode bridge
providers, chain IDs, token decimals, spender addresses, or token addresses.

See [LI.FI MCP reference](./references/mcp.md) for tool mapping, amount
conversion, and approval transaction details.

## Quick Decision Guide

| User wants to... | Action |
|-----------------|--------|
| Bridge same token to another chain | `get-quote` |
| Bridge and receive a different token | `get-quote` |
| Compare multiple routes or choose a provider | `get-routes`, then `get-quote` with matching filters |
| Check whether a route exists | `get-connections` |
| Check transfer status after submission | `get-status` |
| Same-chain swap only | Use this skill only if there is no more specific swap skill |

## Prerequisites

- `fromChain` — required
- `toChain` — required
- `fromToken` — required
- `fromAmount` — required
- `fromAddress` — required
- `toToken` — optional, defaults to same symbol as `fromToken`
- `toAddress` — optional, defaults to `fromAddress`

If any required field is missing, ask before quoting. Do not guess the source
chain from a wallet address.

## Status Requests

If the user is asking about an existing transfer instead of creating one:

1. Require `txHash`
2. Prefer `fromChain`, `toChain`, and `bridge` if the user has them
3. Call `get-status`
4. Translate the result:
   - `NOT_FOUND` or `PENDING` → still processing
   - `DONE` + `COMPLETED` → transfer succeeded
   - `DONE` + `PARTIAL` → transfer completed with a different destination token
   - `DONE` + `REFUNDED` → transfer failed and funds were returned on the source chain
   - `FAILED` → permanent failure, user may need manual recovery

If this is a status request, stop after reporting status. Do not start a new quote flow.

## How It Works

Follow these steps in order.

**Step 1 — Collect parameters**

Extract from the user message:

- `fromChain` — source chain, required
- `toChain` — destination chain, required
- `fromToken` — source token symbol or address, required
- `fromAmount` — human-readable amount, required
- `fromAddress` — sender wallet, required
- `toToken` — optional; if omitted, use the same symbol as `fromToken`
- `toAddress` — optional; if omitted, use `fromAddress`
- `order` — optional route preference

Route preference mapping:

- "cheapest" → `CHEAPEST`
- "fastest" → `FASTEST`
- "safest" → `SAFEST`
- anything else or omitted → `RECOMMENDED`

If the user names a specific bridge or exchange:

1. Call `get-tools`
2. Match the provider name to a returned `key`
3. Use that `key` in `preferBridges`, `allowBridges`, `preferExchanges`, or `allowExchanges`

Never invent LI.FI provider keys.

**Step 2 — Resolve chains**

Use `get-chain-by-name`, `get-chain-by-id`, or `get-chains` to resolve both chains.

If either chain is not supported by LI.FI, stop and explain that the route
cannot be quoted through LI.FI.

**Step 3 — Resolve tokens**

Use `get-token` for:

- source token on `fromChain`
- destination token on `toChain`

If `toToken` is omitted, use the same token symbol as `fromToken`.

Always use LI.FI-resolved decimals and addresses. Do not reuse addresses from
other skills, because the LI.FI route may expect a different canonical token.

**Step 4 — Convert the amount**

Convert `fromAmount` from human-readable units into the smallest unit using the
resolved source token decimals.

Examples:

- `10 USDC` with 6 decimals → `10000000`
- `1.5 ETH` with 18 decimals → `1500000000000000000`

Use a plain integer string. Never send scientific notation.

**Step 5 — Choose the quoting path**

Use `get-connections` first only when the user asks whether a route exists, or
when you need to confirm a route before fetching a quote.

For most requests, call `get-quote` with:

- `fromChain`
- `toChain`
- `fromToken`
- `toToken`
- `fromAddress`
- `fromAmount`
- `toAddress`
- `order`
- optional bridge or exchange filters
- optional `slippage` only if the user explicitly asked for it

Use `get-routes` only when the user explicitly wants route comparison, provider
choice, or several alternatives.

If you use `get-routes`:

1. Show the top route options with tool, expected output, fees, and duration
2. Ask the user to choose if needed
3. Fetch the final executable quote with `get-quote` using matching provider filters

Prefer `get-quote` for the final executable transaction because it returns a
complete `transactionRequest`.

**Step 6 — Check approval requirements**

If the source token is native, skip approval.

If the source token is an ERC20:

1. Read `quote.estimate.approvalAddress`
2. Call `get-allowance` with:
   - `chain` = source chain
   - `tokenAddress` = source token address
   - `ownerAddress` = `fromAddress`
   - `spenderAddress` = `quote.estimate.approvalAddress`
3. Compare the current allowance with `quote.action.fromAmount`

If allowance is insufficient:

- Approval is required before the main bridge transaction
- Build an exact `approve(spender, amount)` transaction using the source token
  contract and `quote.action.fromAmount`
- Use `quote.estimate.approvalAddress` as the spender
- Never hardcode the spender address

USDT special case:

- If the token is USDT and current allowance is non-zero but still insufficient,
  first add a reset approval to `0`, then add the exact approval transaction

This skill never signs or broadcasts any transaction.

**Step 7 — Show a preview**

Before returning transactions, show a clear preview:

```text
Bridge — LI.FI
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
From:             <fromAmount> <fromToken.symbol> on <fromChain.name>
To:               <toToken.symbol> on <toChain.name>
Recipient:        <toAddress>
Expected Receive: <estimate.toAmount formatted>
Minimum Receive:  <estimate.toAmountMin formatted, if present>
Route:            <quote.tool>
ETA:              <estimate.executionDuration> seconds
Approval:         required | not required
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Fees:
<summarize estimate.feeCosts and estimate.gasCosts>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If this is cross-chain, remind the user:

> "This transfer completes asynchronously after the source-chain transaction is
> confirmed. After submission, track it with `get-status` using the source tx hash."

Ask: "Shall I prepare the unsigned transactions?"

**Step 8 — Return unsigned transactions**

Return transactions in execution order:

1. Optional approval reset for USDT
2. Optional approval transaction
3. Main LI.FI action transaction from `quote.transactionRequest`

Output format:

```json
{
  "success": true,
  "transactions": [
    {
      "type": "approval",
      "description": "Approve token spending for LI.FI",
      "raw_tx": {
        "chain_id": 1,
        "to": "0x...",
        "value": "0",
        "data": "0x..."
      }
    },
    {
      "type": "action",
      "description": "Bridge via LI.FI",
      "raw_tx": {
        "chain_id": 1,
        "to": "0x...",
        "value": "0x0",
        "data": "0x..."
      }
    }
  ],
  "summary": {
    "fromChain": "ethereum",
    "toChain": "base",
    "fromToken": "USDC",
    "toToken": "USDC",
    "fromAmount": "10",
    "estimatedToAmount": "9.96",
    "minimumToAmount": "9.90",
    "route": "across",
    "approvalRequired": true,
    "executionDurationSeconds": 240
  },
  "tracking": {
    "bridge": "across",
    "fromChain": 1,
    "toChain": 8453
  }
}
```

For the main transaction:

- `chain_id` = `quote.transactionRequest.chainId`
- `to` = `quote.transactionRequest.to`
- `value` = `quote.transactionRequest.value`
- `data` = `quote.transactionRequest.data`

For approval transaction construction, use the pattern in
[references/mcp.md](./references/mcp.md#approval-transaction-shape).

## When to Abort

Stop immediately if:

- Required parameters are missing
- Source or destination chain is not supported by LI.FI
- Source or destination token cannot be resolved
- No route exists for the requested transfer
- The user is unclear about the source chain or token
- LI.FI returns a client error that indicates invalid request parameters

## Triggers — User Story Scenarios

### Cross-chain bridge

- "Bridge 500 USDC from Ethereum to Base"
- "Move my ETH from Arbitrum to Optimism"
- "Send 0.2 WETH from Base to mainnet"

### Bridge and swap

- "Bridge USDC from Ethereum and receive ETH on Base"
- "Move 1 ETH from Arbitrum to Base and convert it to USDC"

### Route comparison

- "What is the cheapest bridge from Ethereum to Base for USDC?"
- "Show me the fastest route to move ETH to Arbitrum"
- "Can I use Across for this bridge?"

### Status tracking

- "Check the status of my bridge, tx 0x..."
- "Did my cross-chain transfer complete?"

## Related Skills

| Skill | When to use |
|-------|-------------|
| defi-saver | Master router for leverage, market data, and bridging |
| create-leverage-position | Open a leveraged position after funds are in place |
| boost-position | Increase leverage on an existing position |
| repay-position | Reduce debt or risk |
| close-position | Exit a leveraged position |
