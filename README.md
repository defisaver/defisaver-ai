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
"long ETH 2x with 1 ETH collateral"
"open a leveraged ETH position"
"I want to bet ETH price goes up"

## What It Does

- Understands natural language DeFi intent
- Validates all inputs before any blockchain interaction
- Shows you exactly what will happen before you sign
- Returns transaction data ready for your wallet to sign
- Never executes anything without your confirmation

## Skills

| Skill | Description                                  |
|-------|----------------------------------------------|
| create-leverage-position | Open a new leveraged long position           |
| boost-position | Increase leverage on existing position (TBD) |
| repay-position | Repay debt to reduce risk (TBD)              |
| close-position | Exit position completely (TBD)               |

## Supported Protocols

- TBD

## Supported Assets

| Role | Assets |
|------|--------|
| Collateral | ETH, wstETH, WBTC |
| Borrow | USDC, DAI, USDT |
(TBD)

## Requirements

- Wallet address
- Collateral asset in wallet
- ETH for gas

## How It Works

1. You describe what you want in natural language
2. Plugin validates your inputs
3. Plugin calls DeFi Saver API to prepare the transaction
4. You see a full preview (collateral, debt, health factor, gas)
5. You confirm
6. Plugin returns calldata for your wallet to sign
7. You sign and submit

The plugin never touches your private keys.

## Safety

- Health factor checked before every action
- Transactions aborted if health factor would drop below 1.3 (TBD)
- Maximum leverage capped at 3x (TBD)
- Full preview shown before every transaction

## License

MIT
