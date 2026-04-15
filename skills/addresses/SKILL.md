---
name: addresses
description: >
  Verified contract and token addresses for onchain interactions.
  Use when any skill needs a contract address, token address,
  or needs to verify an address exists on-chain.
  Always use this before hardcoding any address.
allowed-tools: Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Addresses

> **Internal reference only.** Do not expose this list to users as a
> definitive list of supported assets — availability depends on the
> protocol and network at execution time.

Verified contract and token addresses for DeFi Saver plugin.

> **CRITICAL:** Never guess or hallucinate an address.
> Wrong address = lost funds. Always use this file.

## Lookup Order

1. Check this file first
2. If not here, check protocol official docs
3. Verify with cast before interacting:
   `cast code <address> --rpc-url $RPC`

## Aave V3 Contracts

### Ethereum Mainnet (chainId: 1)

| Contract | Address |
|----------|---------|
| Pool | `0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2` |
| PoolAddressesProvider | `0x2f39d218133AFaB8F2B819B1066c7E434Ad94E9e` |
| PoolDataProvider | `0x7B4EB56E7CD4b454BA8ff71E4518426369a138a3` |

### Base (chainId: 8453)

| Contract | Address |
|----------|---------|
| Pool | `0xA238Dd80C259a72e81d7e4664a9801593F98d1c5` |
| PoolAddressesProvider | `0xe20fCBdBfFC4Dd138cE8b2E6FBb6CB49777ad64D` |

### Optimism (chainId: 10)

| Contract | Address |
|----------|---------|
| Pool | `0x794a61358D6845594F94dc1DB02A252b5b4814aD` |
| PoolAddressesProvider | `0xa97684ead0e402dC232d5A977953DF7ECBaB3CDb` |

### Arbitrum (chainId: 42161)

| Contract | Address |
|----------|---------|
| Pool | `0x794a61358D6845594F94dc1DB02A252b5b4814aD` |
| PoolAddressesProvider | `0xa97684ead0e402dC232d5A977953DF7ECBaB3CDb` |

### Linea (chainId: 59144)

| Contract | Address |
|----------|---------|
| Pool | `0xc47b8C00b0f69a36fa203Ffeac0334874574a8Ac` |
| PoolAddressesProvider | `0x89502c3731F69DDC95B65753708A07F8Cd0373F4` |
| PoolDataProvider | `0x47cd4b507B81cB831669c71c7077f4daF6762FF4` |

## Token Addresses

### Ethereum Mainnet

| Token | Address | Decimals |
|-------|---------|----------|
| ETH (native) | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` | 18 |
| WETH | `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2` | 18 |
| wstETH | `0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0` | 18 |
| WBTC | `0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599` | 8 |
| USDC | `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48` | 6 |
| USDT | `0xdAC17F958D2ee523a2206206994597C13D831ec7` | 6 |
| DAI | `0x6B175474E89094C44Da98b954EedeAC495271d0F` | 18 |

### Base (chainId: 8453)

| Token | Address | Decimals |
|-------|---------|----------|
| ETH (native) | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` | 18 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| USDC | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | 6 |
| DAI | `0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb` | 18 |

### Optimism (chainId: 10)

| Token | Address | Decimals |
|-------|---------|----------|
| ETH (native) | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` | 18 |
| WETH | `0x4200000000000000000000000000000000000006` | 18 |
| wstETH | `0x1F32b1c2345538c0c6f582fCB022739c4A194Ebb` | 18 |
| USDC | `0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85` | 6 |
| USDT | `0x94b008aA00579c1307B0EF2c499aD98a8ce58e58` | 6 |
| DAI | `0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1` | 18 |

### Arbitrum (chainId: 42161)

| Token | Address | Decimals |
|-------|---------|----------|
| ETH (native) | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` | 18 |
| WETH | `0x82aF49447D8a07e3bd95BD0d56f35241523fBab1` | 18 |
| wstETH | `0x5979D7b546E38E414F7E9822514be443A4800529` | 18 |
| WBTC | `0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f` | 8 |
| USDC | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` | 6 |
| USDT | `0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9` | 6 |
| DAI | `0xDA10009cBd5D07dd0CeCc66161FC93D7c9000da1` | 18 |

### Linea (chainId: 59144)

| Token | Address | Decimals |
|-------|---------|----------|
| ETH (native) | `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` | 18 |
| WETH | `0xe5D7C2a44FfDDf6b295A15c148167daaAf5Cf34` | 18 |
| wstETH | `0xB5beDd42000b71FddE22D3eE8a79Bd49A568fC8F` | 18 |
| WBTC | `0x3aAB2285ddcDdaD8edf438C1bAB47e1a9D05a9b4` | 8 |
| USDC | `0x176211869cA2b568f2A7D4EE941E073a821EE1ff` | 6 |
| USDT | `0xA219439258ca9da29E9Cc4cE5596924745e12B93` | 6 |

## Verification

Always verify before interacting with any address:
```bash
# Check contract exists
cast code <address> --rpc-url $RPC

# Check token name
cast call <address> "name()(string)" --rpc-url $RPC

# Check token decimals
cast call <address> "decimals()(uint8)" --rpc-url $RPC
```

## Gotchas

- ETH native address is `0xEeee...EEEE` — not zero address
- USDC has 6 decimals — not 18
- USDT has 6 decimals — not 18
- WBTC has 8 decimals — not 18
- wstETH address differs per network
- Always double-check network before using address
