# LIGHT ACCOUNT + PIMLICO vs Gelato Smart Wallet SDK + LIGHT ACCOUNT Migration Guide

This guide provides a detailed comparison between two approaches for implementing light account smart wallets with different bundlers/paymasters services, focusing on the minimal code changes required to switch between implementations.

## Overview

- **SDK**: Permissionless vs Gelato Smart Wallet SDK (`@gelatonetwork/smartwallet`)
- **Bundlers/Paymasters**: Pimlico vs Gelato Relay
- **Account Type**: Light Account

## Integration Steps

### 1. Account Creation

Steps for creating account is same for both approaches.

```typescript
// Step 1: Create signer
const signer = privateKeyToAccount(privateKey);

// Step 2: Create Public client
const publicClient = createPublicClient({
  chain: baseSepolia,
  transport: http(""),
});

// Step 3: Create Light Smart Account
const account = await toLightSmartAccount({
  client: publicClient,
  entryPoint: {
    address: entryPoint06Address,
    version: "0.6",
  },
  owner: signer,
  version: "1.1.0",
});
```

## Core Differences

### 2. Client Creation

#### Light Account + Pimlico

```typescript
// Create Pimlico client
const pimlicoClient = createPimlicoClient({
  transport: http(process.env.NEXT_PUBLIC_PIMLICO_URL || ""),
  entryPoint: {
    address: entryPoint06Address,
    version: "0.6",
  },
});

// Create SmartAccountClient
const smartAccountClient = createSmartAccountClient({
  account,
  chain: arbitrumSepolia,
  bundlerTransport: http(process.env.NEXT_PUBLIC_PIMLICO_URL || ""),
  paymaster: pimlicoClient,
  userOperation: {
    estimateFeesPerGas: async () => {
      return (await pimlicoClient.getUserOperationGasPrice()).fast;
    },
  },
});
```

#### Gelato Smart Wallet SDK + Light Account

**Imports**

```typescript
import { gelatoBundlerActions } from "@gelatonetwork/smartwallet/adapter";
import { sponsored, WalletEncoding } from "@gelatonetwork/smartwallet";
```

**Code**

```typescript
const smartAccountClient = createSmartAccountClient({
  account: account,
  chain: arbitrumSepolia,
  // Important: Chain transport (chain rpc) must be passed here instead of bundler transport
  bundlerTransport: http(),
}).extend(
  gelatoBundlerActions({
    payment: sponsored(process.env.NEXT_PUBLIC_SPONSOR_API_KEY as string),
    encoding: WalletEncoding.LightAccount,
  })
);
```

**Migration Change**: Remove Pimlico’s Paymaster client and replace it with Gelato’s one-step Smart Account Client integration.

### 3. Sending Transaction

#### Light Account + Pimlico

```typescript
const hash = await smartAccountClient.sendTransaction({
  calls: [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ],
});
```

#### Gelato Smart Wallet SDK + Light Account

```typescript
const userOpHash = await smartAccountClient.sendUserOperation({
  calls: [
    {
      to: zeroAddress,
      data: "0x",
      value: BigInt(0),
    },
  ],
});
```

**Migration Change**:

- Replace `sendTransaction` with `sendUserOperation` method

## Complete Migration Example

### From Light Account + Pimlico to Gelato Smart Wallet SDK + Light Account

```typescript
// BEFORE: Light Account + Pimlico
// Step 1: Create signer
const signer = privateKeyToAccount(privateKey);

// Step 2: Create Public client
const publicClient = createPublicClient({
  chain: baseSepolia,
  transport: http(""),
});

// Step 3: Create Light Smart Account
const account = await toLightSmartAccount({
  client: publicClient,
  entryPoint: {
    address: entryPoint06Address,
    version: "0.6",
  },
  owner: signer,
  version: "1.1.0",
});

// Create Pimlico client
const pimlicoClient = createPimlicoClient({
  transport: http(process.env.NEXT_PUBLIC_PIMLICO_URL || ""),
  entryPoint: {
    address: entryPoint06Address,
    version: "0.6",
  },
});

// Create SmartAccountClient
const smartAccountClient = createSmartAccountClient({
  account,
  chain: arbitrumSepolia,
  bundlerTransport: http(process.env.NEXT_PUBLIC_PIMLICO_URL || ""),
  paymaster: pimlicoClient,
  userOperation: {
    estimateFeesPerGas: async () => {
      return (await pimlicoClient.getUserOperationGasPrice()).fast;
    },
  },
});

const hash = await smartAccountClient.sendTransaction({
  calls: [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ],
});
```

```typescript
// AFTER: Gelato Smart Wallet SDK + Light Account
// Step 1: Create signer
const signer = privateKeyToAccount(privateKey);

// Step 2: Create Public client
const publicClient = createPublicClient({
  chain: baseSepolia,
  transport: http(""),
});

// Step 3: Create Light Smart Account
const account = await toLightSmartAccount({
  client: publicClient,
  entryPoint: {
    address: entryPoint06Address,
    version: "0.6",
  },
  owner: signer,
  version: "1.1.0",
});

// Create SmartAccountClient with Gelato integration
const smartAccountClient = createSmartAccountClient({
  account: account,
  chain: arbitrumSepolia,
  // Important: Chain transport (chain rpc) must be passed here instead of bundler transport
  bundlerTransport: http(),
}).extend(
  gelatoBundlerActions({
    payment: sponsored(process.env.NEXT_PUBLIC_SPONSOR_API_KEY as string),
    encoding: WalletEncoding.LightAccount,
  })
);

const userOpHash = await smartAccountClient.sendUserOperation({
  calls: [
    {
      to: zeroAddress,
      data: "0x",
      value: BigInt(0),
    },
  ],
});
```

## Summary

The main differences between the two approaches are:

1. **Account Creation**: Same for both approaches (3 steps)
2. **Client Setup**:
   - Pimlico: Requires separate Pimlico client creation + SmartAccountClient with paymaster
   - Gelato: Single SmartAccountClient with Gelato bundler actions extension
3. **Transaction Flow**:
   - Pimlico: Uses `sendTransaction()` method
   - Gelato: Uses `sendUserOperation()` method
4. **Paymaster Integration**:
   - Pimlico: Explicit paymaster client configuration
   - Gelato: Built-in sponsored payment through bundler actions
