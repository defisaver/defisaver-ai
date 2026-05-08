# LI.FI MCP Reference

Use this file when the `bridging` skill needs precise LI.FI tool behavior.

## MCP Setup

```json
{
  "mcpServers": {
    "lifi": {
      "type": "http",
      "url": "https://mcp.li.quest/mcp"
    }
  }
}
```

The hosted LI.FI MCP server is read-only. It returns quotes and unsigned
`transactionRequest` objects. It does not sign or broadcast transactions.

## Tool Selection

| Need | LI.FI MCP tool |
|------|----------------|
| Resolve supported chains | `get-chains`, `get-chain-by-name`, `get-chain-by-id` |
| Resolve token address and decimals | `get-token` |
| Check whether a route exists | `get-connections` |
| Get a single best executable quote | `get-quote` |
| Compare several routes | `get-routes` |
| Resolve bridge and DEX provider keys | `get-tools` |
| Check token approval state | `get-allowance` |
| Track a submitted transfer | `get-status` |

Default rule:

- Use `get-quote` for a normal bridge request
- Use `get-routes` only when the user wants choices
- Use `get-status` only after the source-chain transaction exists

## Amount Conversion

Convert the user amount into smallest-unit integer form before calling LI.FI:

```text
smallest_unit = human_amount * (10 ^ decimals)
```

Examples:

- `10 USDC` with 6 decimals → `10000000`
- `1.5 ETH` with 18 decimals → `1500000000000000000`

## Quote Parameters

For the final executable quote, prefer token addresses returned by `get-token`:

- `fromChain`
- `toChain`
- `fromToken`
- `toToken`
- `fromAddress`
- `fromAmount`
- `toAddress`
- optional `order`
- optional `slippage`
- optional provider filters

Map user intent to `order`:

- cheapest → `CHEAPEST`
- fastest → `FASTEST`
- safest → `SAFEST`
- default → `RECOMMENDED`

## Important Quote Fields

These quote response fields drive the transaction output:

- `transactionRequest.to`
- `transactionRequest.data`
- `transactionRequest.value`
- `transactionRequest.chainId`
- `tool`
- `action.fromAmount`
- `action.fromToken`
- `action.toToken`
- `estimate.toAmount`
- `estimate.toAmountMin`
- `estimate.approvalAddress`
- `estimate.executionDuration`
- `estimate.feeCosts`
- `estimate.gasCosts`

## Approval Transaction Shape

If allowance is insufficient for an ERC20 source token, construct an exact
approval transaction before the main LI.FI transaction.

Use:

- `to` = source token contract address
- `value` = `0`
- spender = `quote.estimate.approvalAddress`
- amount = `quote.action.fromAmount`

If your runtime supports ABI encoding, the standard ERC20 call is:

```typescript
import { encodeFunctionData, erc20Abi } from 'viem'

const data = encodeFunctionData({
  abi: erc20Abi,
  functionName: 'approve',
  args: [quote.estimate.approvalAddress, BigInt(quote.action.fromAmount)],
})
```

USDT special case:

- If current allowance is non-zero but insufficient, send `approve(spender, 0)`
  first, then send the exact approval

## Status Handling

Interpret `get-status` results as:

- `NOT_FOUND` → not indexed yet, continue waiting
- `PENDING` → transfer in progress
- `DONE` + `COMPLETED` → success
- `DONE` + `PARTIAL` → user received a different destination token
- `DONE` + `REFUNDED` → bridge failed and funds were returned
- `FAILED` → permanent failure
