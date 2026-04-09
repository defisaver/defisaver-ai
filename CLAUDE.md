# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Claude Code plugin (skills package) for DeFi leverage management on Aave V3. It ships SKILL.md files — structured natural language instructions that AI agents load as context to perform DeFi actions. There is no application code to build or test; the "code" is the skill definitions themselves.

## Development Setup
```bash
# Test plugin locally
claude --plugin-dir .

# Test with specific skill
claude --plugin-dir . --skill create-leverage-position
```

## Repository Structure

```
.claude-plugin/plugin.json       # Plugin metadata and skill list
skills/
  viem-integration/SKILL.md      # Core EVM primitives (address validation, amount formatting, simulation)
  addresses/SKILL.md              # Contract addresses per network
  aave-v3/SKILL.md                # Aave V3 protocol reference
  defi-saver/SKILL.md             # Master routing skill
  create-leverage-position/
    SKILL.md                     # Main skill instructions
    references/api.md            # DeFi Saver API reference for this skill
  boost-position/                # Increase leverage on existing position
  repay-position/                # Repay debt to reduce risk
  close-position/                # Exit position completely
AGENTS.md                        # Agent-facing install/usage docs
README.md                        # User-facing docs
evals/                           # Skill evaluation test cases
```

## Skill File Format

Skills are Markdown files with YAML frontmatter:

```yaml
---
name: skill-name
description: >
  When to activate this skill (used for trigger matching)
allowed-tools: WebFetch, Read   # tools the agent may use
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---
```

The body is structured natural language instructions for the AI agent. Key sections: Quick Decision Guide, Prerequisites, Input Validation, step-by-step How It Works, Error Handling, Triggers, Non-Interactive Usage, Related Skills.

Keep SKILL.md files under 500 lines. Use `references/` for detailed docs loaded on demand.

## Plugin Registration

`plugin.json` lists skill directories. Each entry is a path to a directory containing a `SKILL.md`. New skills must be added here to be included in the plugin.

## How Skills Interrelate

- `viem-integration` is a dependency skill — other skills reference it for EVM primitives
- `defi-saver` is the master routing skill — handles intent detection and routes to the right sub-skill
- `create-leverage-position` calls the DeFi Saver API (documented in `references/api.md`) and returns unsigned calldata
- The agent never executes transactions; it returns calldata for the user to sign

## DeFi Domain Rules Encoded in Skills

- Health factor must stay above 1.3 (abort) / warn below 1.5
- Leverage: 1.1x min, 3.0x max
- Collateral assets: ETH, wstETH, WBTC
- Borrow assets: USDC, DAI, USDT
- Networks: ethereum (default), base, arbitrum
- ERC20 collateral requires 2 transactions (approve + main tx); ETH requires 1
- Always validate inputs before API calls; always simulate before preparing calldata
- Never retry write actions automatically on failure

## Output Format

All skills return:
```json
{
  "success": true,
  "transactions": [
    {
      "type": "approval | typed_signature | action",
      "description": "...",
      "raw_tx": { "chain_id": 1, "to": "0x...", "value": "0", "data": "0x..." }
    }
  ],
  "summary": {}
}
```

On failure:
```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human readable explanation",
  "details": {}
}
```

Transaction type rules:
- `approval` = token approval (ERC20 only), always before `action`
- `typed_signature` = EIP-712 off-chain signature, no gas required
- `action` = main blockchain transaction

## Testing Skills

Always test with these scenarios:
```
# Input validation
"long ETH 2x wallet 0xinvalid"
→ Expected: address validation error

# Unsupported asset
"long DOGE 2x"
→ Expected: list of supported assets

# Leverage limit
"long ETH 10x"
→ Expected: max 3x message

# Missing parameters
"long ETH"
→ Expected: ask for amount and wallet

# Valid (API will fail but validation passes)
"long ETH 2x with 1 ETH, wallet 0x742d35Cc6634C0532925a3b844Bc454e4438f44e"
→ Expected: API call attempt, timeout/error message
```

See `evals/` directory for comprehensive test cases per skill.
