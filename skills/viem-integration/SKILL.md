---
name: viem-integration
description: >
  Developer reference for TypeScript/JavaScript clients building on top of
  the DeFi Saver plugin. Provides address validation patterns, token decimal
  reference, and network chain IDs for viem-based integrations.
  Use when a developer asks how to integrate this plugin into a TypeScript
  or JavaScript application, or needs chain IDs and token decimals.
  Not needed during normal agent-driven skill execution.
allowed-tools: Read
compatibility: Reference material for TypeScript/JavaScript developers using viem.
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# viem Integration — Developer Reference

> **Note:** This skill is a reference for developers building TypeScript or
> JavaScript applications on top of the DeFi Saver plugin. It is not used
> by the LLM agent during normal skill execution — skills call the
> DeFi Saver API directly via HTTP. This reference is for client-side
> code that wraps or extends the plugin output.

## DeFi Saver Utils API

These are the HTTP endpoints used by all skills. Useful if you are building
a wrapper client or need to call them directly.

### Validate Address
```
GET https://ai.defisaver.com/api/v1/utils/validate-address/{address}
→ { success, data: { address, isValid, checksumAddress } }
```
Always call this first. Use `checksumAddress` in all subsequent calls.

### Get Native Balance
```
GET https://ai.defisaver.com/api/v1/utils/balance/{address}/{chainId}
→ { success, data: { balance, balanceFormatted } }
```

### Get Gas Price
```
GET https://ai.defisaver.com/api/v1/utils/gas-price/{chainId}
→ { success, data: { gasPrice, gasPriceFormatted } }
```

### Get Chain Info
```
GET https://ai.defisaver.com/api/v1/utils/chain/{chainId}
→ { success, data: { chainId, chainName, supported } }
```

## Supported Networks

| Network | chainId | viem chain import |
|---------|---------|-------------------|
| Ethereum | 1 | `mainnet` |
| Optimism | 10 | `optimism` |
| Base | 8453 | `base` |
| Arbitrum | 42161 | `arbitrum` |
| Linea | 59144 | `linea` |

## Token Decimals

| Token | Decimals |
|-------|----------|
| ETH / WETH / wstETH | 18 |
| WBTC | 8 |
| USDC | 6 |
| USDT | 6 |
| DAI | 18 |

## Address Validation (viem)

```typescript
import { isAddress, getAddress } from 'viem'

if (!isAddress(userInput)) {
  throw new Error('Invalid Ethereum address')
}
const checksumAddress = getAddress(userInput)
```

## Amount Formatting (viem)

```typescript
import { parseUnits, formatUnits } from 'viem'

// User input → wei
parseUnits('1.5', 18)  // ETH
parseUnits('100', 6)   // USDC

// Wei → display
formatUnits(1500000000000000000n, 18)  // "1.5"
formatUnits(100000000n, 6)             // "100.0"
```

## Common Gotchas

- USDC and USDT have 6 decimals, not 18
- WBTC has 8 decimals, not 18
- Always normalize addresses with `getAddress()` before API calls
- ETH native address: `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`
