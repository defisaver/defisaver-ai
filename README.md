# DeFi Saver AI Plugin

Leverage management and cross-chain bridging using natural language.
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
"bridge 100 USDC from Ethereum to Base"
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
| bridging | Bridge assets across chains with LI.FI quotes |

## Supported Protocols

- Leading lending protocols on Ethereum, Optimism, Base, Arbitrum, and Linea
- Flashloans via Morpho (zero fee)
- Cross-chain bridging and bridge+swap via LI.FI MCP

## Supported Assets

Collateral assets and borrow assets are resolved dynamically based on
protocol availability.
Common collateral: ETH, wstETH, and other major assets.
Borrow asset: automatically selected liquid stablecoin.

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

For bridge requests, the plugin uses LI.FI MCP to resolve chains and tokens,
get a route quote, check approval requirements, and return unsigned
transactions in the same style.

## MCP Setup For Bridging

Configure the LI.FI MCP server in your client:

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
