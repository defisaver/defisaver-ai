---
name: viem-integration
description: >
  EVM blockchain interactions using viem. Use for reading
  blockchain data, preparing transactions, validating addresses,
  formatting token amounts, and simulating contract calls.
  Always use this before preparing any DeFi transaction.
allowed-tools: Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# viem Integration

Core EVM interactions for DeFi Saver plugin. Provides
address validation, amount formatting, contract simulation,
and transaction preparation patterns.

## Quick Reference

| Task | Method |
|------|--------|
| Validate address | `isAddress(addr)` |
| Format to wei | `parseUnits(amount, decimals)` |
| Format from wei | `formatUnits(bigint, decimals)` |
| ETH to wei | `parseEther(amount)` |
| Wei to ETH | `formatEther(bigint)` |
| Simulate contract | `publicClient.simulateContract(...)` |

## Client Setup
```typescript
import { createPublicClient, http } from 'viem'
import { mainnet, base, arbitrum } from 'viem/chains'

const publicClient = createPublicClient({
  chain: mainnet,
  transport: http(),
})
```

## Address Validation

Always validate before any API call or tx preparation:
```typescript
import { isAddress, getAddress } from 'viem'

// Validate
if (!isAddress(userInput)) {
  throw new Error('Invalid Ethereum address')
}

// Normalize to checksum format
const address = getAddress(userInput)
```

## Amount Formatting
```typescript
import { parseUnits, formatUnits, parseEther, formatEther } from 'viem'

// User input → contract (wei)
parseUnits('1.5', 18)   // ETH: 1500000000000000000n
parseUnits('100', 6)    // USDC: 100000000n

// Contract (wei) → display
formatUnits(1500000000000000000n, 18)  // "1.5"
formatUnits(100000000n, 6)             // "100.0"

// ETH shorthand
parseEther('1.5')           // 1500000000000000000n
formatEther(1500000000000000000n)  // "1.5"
```

## Token Decimals

| Token | Decimals |
|-------|----------|
| ETH / WETH / wstETH | 18 |
| WBTC | 8 |
| USDC | 6 |
| USDT | 6 |
| DAI | 18 |

## Contract Simulation

ALWAYS simulate before preparing calldata.
Simulation catches errors before user signs and pays gas.
```typescript
import { createPublicClient, http } from 'viem'

try {
  const { request } = await publicClient.simulateContract({
    address: contractAddress,
    abi: contractAbi,
    functionName: 'supply',
    args: [asset, amount, onBehalfOf, 0],
    account: userWallet,
  })
  // Simulation passed → safe to prepare calldata
  return request
} catch (error) {
  // Simulation failed → do not proceed
  handleSimulationError(error)
}
```

## Error Handling
```typescript
import {
  ContractFunctionExecutionError,
  InsufficientFundsError,
  UserRejectedRequestError,
} from 'viem'

try {
  await publicClient.simulateContract(...)
} catch (error) {
  if (error instanceof ContractFunctionExecutionError) {
    // Smart contract reverted
    // error.shortMessage contains readable reason
    return `Transaction would fail: ${error.shortMessage}`
  }
  if (error instanceof InsufficientFundsError) {
    return 'Not enough ETH to cover gas fees'
  }
  if (error instanceof UserRejectedRequestError) {
    return 'Transaction rejected by user'
  }
  return `Unexpected error: ${error.message}`
}
```

## Read Contract
```typescript
const balance = await publicClient.readContract({
  address: tokenAddress,
  abi: erc20Abi,
  functionName: 'balanceOf',
  args: [userAddress],
})
```

## Supported Networks

| Network | Chain ID | viem import |
|---------|----------|-------------|
| Ethereum | 1 | `mainnet` |
| Base | 8453 | `base` |
| Arbitrum | 42161 | `arbitrum` |

## Common Gotchas

- Never hardcode private keys — always use environment variables
- Always simulate before preparing calldata
- USDC/USDT have 6 decimals, not 18 — wrong decimals = wrong amounts
- `isAddress()` is case-insensitive but always normalize with `getAddress()`
- bigint arithmetic: use `BigInt(value)` not `parseInt()`
