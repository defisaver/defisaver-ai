3# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Claude Code plugin (skills package) for DeFi leverage management on Aave V3. It ships SKILL.md files — structured natural language instructions that AI agents load as context to perform DeFi actions. There is no application code to build or test; the "code" is the skill definitions themselves.

## Repository Structure

```
.claude-plugin/plugin.json       # Plugin metadata and skill list
skills/
  viem-integration/SKILL.md      # Core EVM primitives (address validation, amount formatting, simulation)
  create-leverage-position/
    SKILL.md                     # Main skill instructions
    references/api.md            # DeFi Saver API reference for this skill
  boost-position/                # TBD
  repay-position/                # TBD
  close-position/                # TBD
AGENTS.md                        # Agent-facing install/usage docs
README.md                        # User-facing docs
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

## Plugin Registration

`plugin.json` lists skill directories. Each entry is a path to a directory containing a `SKILL.md`. New skills must be added here to be included in the plugin.

## How Skills Interrelate

- `viem-integration` is a dependency skill — other skills reference it for EVM primitives
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
