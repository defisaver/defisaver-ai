---
name: defi-saver
description: >
  Master routing skill for DeFi Saver leveraged positions.
  Use when user wants to manage leverage: open, boost, repay, or close
  a position. Also handles market data questions and best opportunity
  requests. Routes to the correct sub-skill based on user intent.
allowed-tools: WebFetch, Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# DeFi Saver — Master Routing Skill

Routes user intent to the correct sub-skill for managing leveraged
positions via DeFi Saver. Handles market data queries via DefiLlama MCP.

## Quick Decision Guide

| User wants to... | Route to |
|-----------------|----------|
| Open new leveraged long | create-leverage-position |
| Increase leverage on existing position | boost-position |
| Repay debt / reduce risk | repay-position |
| Exit position completely | close-position |
| Best opportunity / market data | DefiLlama MCP → then route |
| Current prices / yields | DefiLlama MCP |

## Market Data (DefiLlama)

For questions about best opportunities, market data,
or comparing protocols, use the DefiLlama MCP tools:

| User asks... | DefiLlama tool | Then... |
|-------------|----------------|---------|
| "best opportunity to long ETH" | get_yield_pools | Show options, ask user to choose |
| "compare Aave vs Compound" | get_protocol_metrics | Show comparison |
| "current ETH price" | get_token_prices | Show price |
| "best USDC borrow rate" | get_yield_pools | Filter by USDC borrow |

After getting market data, route to appropriate skill for execution.

Example flow:
1. User: "get me the best opportunity to long ETH"
2. Call get_yield_pools with ETH filter
3. Show top 3 opportunities with APY and risk
4. Ask: "Which would you like to open?"
5. Route to create-leverage-position with chosen params

## Routing Rules

Always check for existing position context before routing:
- If user mentions "my position" or "existing" → boost or repay/close
- If user mentions "open", "new", "create" → create-leverage-position
- If user mentions "more leverage", "increase" → boost-position
- If user mentions "repay", "reduce risk", "lower debt" → repay-position
- If user mentions "close", "exit", "unwind" → close-position

## Supported Networks

| Network | Chain ID | Value |
|---------|----------|-------|
| Ethereum | 1 | ethereum |
| Optimism | 10 | optimism |
| Base | 8453 | base |
| Arbitrum | 42161 | arbitrum |
| Linea | 59144 | linea |

Default: ethereum

## Supported Assets

Collateral and borrow assets are resolved dynamically based on
protocol availability. Do not enumerate a fixed list to users.

## Safety Constraints (apply to all sub-skills)

- Health ratio below 1.3 → ABORT
- Leverage above 3x → REJECT
- Never sign or submit transactions
- Always show preview before returning calldata
- Never auto-retry write actions
