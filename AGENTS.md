# DeFi Saver AI Plugin

Works with any agent that supports the SKILL.md format:
Claude Code, Cursor, Gemini CLI, Codex CLI, and others.

## Install
```bash
npx skills add defisaver/defisaver-ai
```

## What This Plugin Does

Manage leveraged positions using natural language.
No need to understand DeFi protocols — just describe what you want.

## Skills Included

| Skill | Trigger examples |
|-------|-----------------|
| create-leverage-position | "long ETH 2x", "bet ETH goes up" |
| boost-position | "increase my leverage", "boost my position" |
| repay-position | "repay my loan", "reduce my risk" |
| close-position | "close my position", "exit my long" |

## Requirements

- Wallet address (0x...)
- ETH or supported collateral in wallet
- ETH for gas fees
- Supported network: Ethereum, Base, Arbitrum (TBD)

## Supported Assets

Collateral: ETH, wstETH, WBTC (TBD)
Borrow: USDC, DAI, USDT (TBD)

## Important

This plugin prepares transactions for your signature.
It never signs or submits transactions automatically.
You always review and confirm before anything happens on-chain.
