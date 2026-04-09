# DeFi Saver AI Plugin

Works with any agent that supports the SKILL.md format:
Claude Code, Cursor, Gemini CLI, Codex CLI, and others.

## Install
```bash
npx skills add defisaver/defisaver-ai
```

## What This Plugin Does

Manage leveraged positions via DeFi Saver using natural language.
No need to understand DeFi protocols — just describe what you want.

## Skills Included

| Skill | Trigger examples |
|-------|-----------------|
| create-leverage-position | "long ETH 2x", "bet ETH goes up", "open leveraged position" |
| boost-position | "increase my leverage", "boost my position" |
| repay-position | "repay my loan", "reduce my risk", "lower health risk" |
| close-position | "close my position", "exit my long", "unwind position" |

## Requirements

- Wallet address (0x...)
- ETH or supported collateral in wallet
- ETH for gas fees
- Supported network: Ethereum, Optimism, Base, Arbitrum, Linea

## Supported Assets

Collateral: ETH-based assets and major crypto tokens
Borrow: Liquid stablecoins (chosen automatically)

## Output Format

All skills return unsigned transactions:
```json
{
  "success": true,
  "transactions": [
    {
      "type": "approval | typed_signature | action",
      "description": "Human readable description",
      "raw_tx": {
        "chain_id": 1,
        "to": "0x...",
        "value": "0",
        "data": "0x..."
      }
    }
  ],
  "summary": {}
}
```

Sign and submit transactions in order.
`typed_signature` transactions require signing only (no gas).

## Safety

This plugin NEVER:
- Signs transactions on your behalf
- Accesses private keys
- Executes without explicit user confirmation
- Proceeds with health ratio below 1.3
- Accepts leverage above 3x

All output is unsigned. You always control signing.
