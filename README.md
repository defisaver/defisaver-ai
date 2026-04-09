# DeFi Saver AI Plugin

Leverage management on Aave V3 using natural language.
Open, boost, repay, and close leveraged positions —
just describe what you want.

## Install
```bash
npx skills add defisaver/defisaver-ai
```

## Quick Start

After installing, try these in your agent:
```
"long ETH 2x with 1 ETH collateral"
"open a leveraged ETH position"
"I want to bet ETH price goes up"
```

## What It Does

- Understands natural language DeFi intent
- Validates all inputs before any blockchain interaction
- Shows you exactly what will happen before you sign
- Returns unsigned transaction data for your wallet to sign
- Never executes anything without your confirmation

## Skills

| Skill | Description |
|-------|-------------|
| create-leverage-position | Open a new leveraged long position |
| boost-position | Increase leverage on existing position |
| repay-position | Repay debt to reduce risk |
| close-position | Exit position completely |

## Supported Protocols

- Aave V3 on Ethereum, Base, Arbitrum
- Flashloans via Morpho (zero fee)

## Supported Assets

| Role | Assets |
|------|--------|
| Collateral | ETH, wstETH, WBTC |
| Borrow | USDC, DAI, USDT |

## Requirements

- Wallet address (0x...)
- Collateral asset in wallet
- ETH for gas

## How It Works

1. You describe what you want in natural language
2. Plugin validates your inputs
3. Plugin calls DeFi Saver API to prepare the transaction
4. You see a full preview (collateral, debt, health ratio, gas)
5. You confirm
6. Plugin returns unsigned transaction data
7. You sign and submit with your wallet

## Safety

This plugin is designed with safety as the top priority:

**What this plugin NEVER does:**
- Signs transactions on your behalf
- Accesses or stores private keys
- Executes transactions without your explicit confirmation
- Proceeds if health ratio would drop below 1.3
- Accepts leverage above 3x
- Operates on unsupported networks

**What happens before every transaction:**
- Health ratio checked before and after (estimated)
- Transaction simulated to catch failures before you sign
- Full preview shown: collateral, debt, health ratio, liquidation price
- Explicit confirmation required

**Output is always unsigned:**
All transactions are returned as `raw_tx: { chain_id, to, value, data }`.
You sign with your own wallet. No private key ever touches this plugin.

## License

MIT
