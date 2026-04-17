---
name: execute-steps
description: >
  Execute DeFi Saver transaction steps using viem wallet client.
  Utility skill for create-leverage-position, boost-position, repay-position,
  and close-position skills. Handles TypedSignature and SafeTx execution
  in sequence with proper error handling and receipt tracking.
allowed-tools: Read
license: MIT
metadata:
  author: defisaver
  version: "1.0.0"
---

# Execute Steps — Transaction Execution Utility

Executes a sequence of DeFi Saver transaction steps using viem wallet client.
Each step calls its API endpoint, then executes the returned transaction data.
Supports TypedSignature (off-chain) and SafeTx (on-chain) step types.

## When to Use This Skill

This skill is referenced by position management skills when they need to
execute transaction steps after user confirmation:
- create-leverage-position → after Step 8 confirmation
- boost-position → after user approves boost
- repay-position → after user approves repay
- close-position → after user approves close

Do not use directly — always called from parent position management skills.

## Prerequisites

- viem wallet client configured
- Private key access for signing
- RPC endpoint for target network
- Gas fee configuration
- Steps array from DeFi Saver API response

## Step Execution Flow

### Input Format

Expects `data.steps` array from DeFi Saver API response:
```javascript
const data = {
  steps: [
    {
      name: "Approval",
      description: "Approve USDC for spending",
      type: "SafeTx",
      txDataApiEndpoint: "/approval/asset",
      bodyParams: { /* step-specific params */ }
    },
    {
      name: "Position Creation",
      description: "Create leveraged position",
      type: "SafeTx", 
      txDataApiEndpoint: "/aave-v3/create/execute-recipe",
      bodyParams: { /* step-specific params */ }
    }
  ]
}
```

### Execution Code Pattern

```javascript
const { privateKeyToAccount } = require('viem/accounts');
const { createWalletClient, createPublicClient, http, parseGwei } = require('viem');
const { parseUnits } = require('viem');
const { mainnet, optimism, base, arbitrum, linea } = require('viem/chains');

// Setup based on network
const account = privateKeyToAccount(PROVIDED_PRIVATE_KEY);
const chain = getChainById(data.network); // Map chainId to viem chain
const client = createWalletClient({
  account,
  chain,
  transport: http(PROVIDED_RPC),
});
const publicClient = createPublicClient({
  chain,
  transport: http(PROVIDED_RPC),
});

const userAddress = account.address;

// Fetch gas price from DeFi Saver API
const gasResponse = await fetch(`https://ai.defisaver.com/api/v1/utils/gas-price/${chain.id}`);
const gasData = await gasResponse.json();
const maxFeePerGas = parseUnits(gasData.data.gasPrice, 'gwei');
const maxPriorityFeePerGas = parseUnits(gasData.data.gasPrice, 'gwei');

console.log('User address:', userAddress);
console.log('Gas price:', gasData.data.gasPriceFormatted);
let previousTxReceipt = undefined;
let previousReturnValue = undefined;

// Execute steps in sequence
for (const step of data.steps) {
  console.log('Executing step:', step.description);
  
  // Call step API endpoint with previous results
  const response = await request(app)
    .post(`/api/v1${step.txDataApiEndpoint}`)
    .send({ ...step.bodyParams, previousTxReceipt, previousReturnValue });

  const stepData = response.body.data;
  previousReturnValue = stepData;
  
  if (step.type === 'TypedSignature') {
    // Sign off-chain message
    previousTxReceipt = await account.signTypedData(stepData.message);
    console.log('Signed TypedSignature:', previousTxReceipt);
  } else if (step.type === 'SafeTx') {
    // Send blockchain transaction
    previousTxReceipt = await client.sendTransaction({
      ...stepData.txParams,
      maxFeePerGas,
      maxPriorityFeePerGas,
      chainId: chain.id,
      gas: 3000000, // Conservative gas limit
    });
    
    console.log('Sent transaction:', previousTxReceipt);
    
    // Wait for confirmation
    const transaction = await publicClient.waitForTransactionReceipt({
      hash: previousTxReceipt,
    });
    
    console.log('Transaction confirmed:', transaction.transactionHash);
  } else {
    throw new Error(`Unsupported step type: ${step.type}`);
  }
}
```

## Network Chain Mapping

Map DeFi Saver network IDs to viem chains:

```javascript
function getChainById(chainId) {
  switch(chainId) {
    case 1: return mainnet;
    case 10: return optimism; 
    case 8453: return base;
    case 42161: return arbitrum;
    case 59144: return linea;
    default: throw new Error(`Unsupported network: ${chainId}`);
  }
}
```

## Error Handling

### Step API Call Failures
```javascript
if (!response.ok || !response.body.success) {
  throw new Error(`Step ${step.name} failed: ${response.body.message}`);
}
```

### Transaction Failures
```javascript
try {
  previousTxReceipt = await client.sendTransaction(txParams);
} catch (error) {
  throw new Error(`Transaction ${step.name} failed: ${error.message}`);
}
```

### Gas Estimation
```javascript
// Estimate gas before sending
const gasEstimate = await publicClient.estimateGas({
  ...stepData.txParams,
  account: userAddress,
});

const txParams = {
  ...stepData.txParams,
  gas: gasEstimate + BigInt(50000), // Add buffer
  maxFeePerGas,
  maxPriorityFeePerGas,
  chainId: chain.id,
};
```

## Environment Variables Required

```bash
# Private key for signing (test/development only)
TEST_ACCOUNT_PRIVATE_KEY=0x...

# RPC endpoints by network
RPC_URL_1=https://eth-mainnet.alchemyapi.io/v2/...     # Ethereum
RPC_URL_10=https://opt-mainnet.g.alchemy.com/v2/...   # Optimism  
RPC_URL_8453=https://base-mainnet.g.alchemy.com/v2/... # Base
RPC_URL_42161=https://arb-mainnet.g.alchemy.com/v2/... # Arbitrum
RPC_URL_59144=https://linea-mainnet.infura.io/v3/...   # Linea
```

## Safety Considerations

⚠️ **NEVER use this in production with hardcoded private keys**
⚠️ **Always validate transaction parameters before signing**
⚠️ **Use proper gas estimation to avoid failed transactions**
⚠️ **Implement proper error recovery for failed steps**

### Production Recommendations
- Use wallet connect or hardware wallet integration
- Implement user confirmation for each transaction
- Add slippage protection for price-sensitive operations
- Use multicall for atomic operations where possible
- Implement proper logging and monitoring

## Integration with Position Skills

Parent skills should:
1. Validate all inputs and get user confirmation
2. Show transaction preview with gas costs
3. Call this execution utility only after explicit user approval
4. Handle execution errors gracefully
5. Provide clear feedback on transaction status

Example integration:
```javascript
// In create-leverage-position skill after Step 7 confirmation
if (userConfirmed) {
  try {
    await executeSteps(response.data);
    return "✅ Position created successfully!";
  } catch (error) {
    return `❌ Transaction failed: ${error.message}`;
  }
}
```

## Dependencies

```json
{
  "viem": "^2.0.0",
}
```

## Related Skills

- [create-leverage-position](../create-leverage-position/SKILL.md) — Uses for position creation
- [boost-position](../boost-position/SKILL.md) — Uses for leverage increase
- [repay-position](../repay-position/SKILL.md) — Uses for debt repayment  
- [close-position](../close-position/SKILL.md) — Uses for position closure
- [viem-integration](../viem-integration/SKILL.md) — Provides viem utilities