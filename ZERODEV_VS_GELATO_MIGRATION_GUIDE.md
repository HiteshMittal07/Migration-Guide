# ZeroDev SDK + Ultra Relay vs Gelato Smart Wallet SDK + Kernel Wallet Migration Guide

This guide provides a detailed comparison between two approaches for implementing Kernel-based smart wallets with different relay services, focusing on the minimal code changes required to switch between implementations.

## Overview

### Comparison 1: ZeroDev SDK + Ultra Relay vs Gelato Smart Wallet SDK + Kernel Wallet
- **SDK**: ZeroDev SDK (`@zerodev/sdk`) vs Gelato Smart Wallet SDK (`@gelatonetwork/smartwallet`)
- **Relay**: Ultra Relay vs Gelato Relay
- **Account Type**: Kernel Account with EIP-7702 support

### Comparison 2: ZeroDev SDK + Ultra Relay vs Gelato Smart Wallet SDK + Kernel Wallet
- **SDK**: ZeroDev SDK (`@zerodev/sdk`) vs Gelato Smart Wallet SDK (`@gelatonetwork/smartwallet`)
- **Relay**: Ultra Relay vs Gelato Relay
- **Account Type**: Kernel Account (ERC4337)

# Comparison 1 : Kernel Account with EIP-7702 support

## Core Differences

### 1. Account Creation

#### ZeroDev SDK + Ultra Relay
```typescript
// Step 1: Create entry point and kernel version
const entryPoint = getEntryPoint("0.7");
const kernelVersion = KERNEL_V3_3_BETA;

// Step 2: Create wallet client for authorization
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

// Step 3: Sign authorization for EIP-7702
const authorization = await walletClient.signAuthorization({
  account,
  contractAddress: KernelVersionToAddressesMap[kernelVersion].accountImplementationAddress,
});

// Step 4: Create validator
const validator = await signerToEcdsaValidator(publicClient, {
  entryPoint,
  kernelVersion,
  signer: account,
});

// Step 5: Create kernel account
const kernelAccount = await createKernelAccount(publicClient as any, {
  address: account.address,
  eip7702Auth: authorization,
  entryPoint,
  kernelVersion,
  plugins: { sudo: validator },
});
```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript
// Single step: Create kernel account using Gelato's kernel function
const account = await kernel({
  owner: signer,
  client: publicClient,
  eip7702: true,
});
```

**Migration Change**: Replace the 5-step ZeroDev account creation with Gelato's single `kernel()` function call.

### 2. Client Creation

#### ZeroDev SDK + Ultra Relay
```typescript
const kernelClient = createKernelAccountClient({
  account: kernelAccount,
  chain: baseSepolia,
  bundlerTransport: http(process.env.NEXT_PUBLIC_ULTRA_RELAY_URL || ""),
  paymaster: undefined,
  userOperation: {
    estimateFeesPerGas: async ({ bundlerClient }) => {
      return getUserOperationGasPrice(bundlerClient);
    },
  },
});
```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript
// Step 1: Create wallet client
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

// Step 2: Create Gelato smart wallet client
const smartWalletClient = await createGelatoSmartWalletClient(
  walletClient,
  {
    apiKey: process.env.NEXT_PUBLIC_SPONSOR_API_KEY || "",
  }
);
```

**Migration Change**: Replace ZeroDev's `createKernelAccountClient` with Gelato's two-step process using `createWalletClient` and `createGelatoSmartWalletClient`.

### 3. Call Data Preparation

#### ZeroDev SDK + Ultra Relay
```typescript

const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ] as const;

const callData = await kernelClient.account.encodeCalls(calls);

```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript

const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ];

const preparedCalls = await smartWalletClient.prepare({
  payment: sponsored(process.env.NEXT_PUBLIC_SPONSOR_API_KEY || ""),
  calls,
});
```

**Migration Change**: 
- Replace ZeroDev's `encodeCalls()` with Gelato's `prepare()` method

### 4. Transaction Execution

#### ZeroDev SDK + Ultra Relay
```typescript
const userOpHash = await kernelClient.sendUserOperation({
  callData,
  maxFeePerGas: BigInt(0),
  maxPriorityFeePerGas: BigInt(0),
});

const userOpReceipt = await kernelClient.waitForUserOperationReceipt({
  hash: userOpHash,
});
const hash = userOpReceipt.receipt.transactionHash;
```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript
const results = await smartWalletClient.send({ preparedCalls });
const hash = await results?.wait();
```

**Migration Change**: 
- Replace ZeroDev's `sendUserOperation()` and `waitForUserOperationReceipt()` with Gelato's `send()` and `wait()`
- Change from `userOpHash` to `results.id` for logging


## Complete Migration Example

### From ZeroDev to Gelato

```typescript
// BEFORE: ZeroDev SDK + Ultra Relay
const entryPoint = getEntryPoint("0.7");
const kernelVersion = KERNEL_V3_3_BETA;
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});
const authorization = await walletClient.signAuthorization({
  account,
  contractAddress: KernelVersionToAddressesMap[kernelVersion].accountImplementationAddress,
});
const validator = await signerToEcdsaValidator(publicClient, {
  entryPoint,
  kernelVersion,
  signer: account,
});
const kernelAccount = await createKernelAccount(publicClient as any, {
  address: account.address,
  eip7702Auth: authorization,
  entryPoint,
  kernelVersion,
  plugins: { sudo: validator },
});

const kernelClient = createKernelAccountClient({
  account: kernelAccount,
  chain: baseSepolia,
  bundlerTransport: http(process.env.NEXT_PUBLIC_ULTRA_RELAY_URL || ""),
  paymaster: undefined,
  userOperation: {
    estimateFeesPerGas: async ({ bundlerClient }) => {
      return getUserOperationGasPrice(bundlerClient);
    },
  },
});


const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ] as const;

const callData = await kernelClient.account.encodeCalls(calls);

const userOpHash = await kernelClient.sendUserOperation({
  callData,
  maxFeePerGas: BigInt(0),
  maxPriorityFeePerGas: BigInt(0),
});
const userOpReceipt = await kernelClient.waitForUserOperationReceipt({
  hash: userOpHash,
});
const hash = userOpReceipt.receipt.transactionHash;
```

```typescript
// AFTER: Gelato Smart Wallet SDK + Kernel Wallet
const account = await kernel({
  owner: signer,
  client: publicClient,
  eip7702: true,
});

const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

const smartWalletClient = await createGelatoSmartWalletClient(
  walletClient,
  {
    apiKey: process.env.SPONSOR_API_KEY || "",
  }
);

const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ];

const preparedCalls = await smartWalletClient.prepare({
  payment: sponsored(process.env.SPONSOR_API_KEY || ""),
  calls,
});

const results = await smartWalletClient.send({ preparedCalls });
const hash = await results?.wait();
```


## Migration Checklist

### 1. Dependencies
- [ ] Remove `@zerodev/sdk` imports
- [ ] Add `@gelatonetwork/smartwallet` imports

### 2. Account Creation
- [ ] Replace ZeroDev account creation with Gelato's `kernel()` function
- [ ] Remove entry point and kernel version configuration
- [ ] Remove authorization signing
- [ ] Remove validator creation

### 3. Client Setup
- [ ] Replace `createKernelAccountClient` with `createWalletClient` + `createGelatoSmartWalletClient`
- [ ] Update bundler transport configuration
- [ ] Remove paymaster configuration

### 4. Transaction Flow
- [ ] Replace `encodeCalls()` with `prepare()`
- [ ] Replace `sendUserOperation()` with `send()`
- [ ] Replace `waitForUserOperationReceipt()` with `wait()`
- [ ] Update transaction hash extraction

### 5. Error Handling
- [ ] Update error messages and logging
- [ ] Ensure retry logic remains compatible

## Summary

The main differences between the two approaches are:

1. **Account Creation**: ZeroDev requires 5 steps vs Gelato's single function call
2. **Client Setup**: ZeroDev uses one client vs Gelato's two-client approach
3. **Transaction Flow**: ZeroDev uses `encodeCalls()` + `sendUserOperation()` vs Gelato's `prepare()` + `send()`


# Comparison 2 : Kernel Account (ERC-4337) 

## Core Differences

### 1. Account Creation

#### ZeroDev SDK + Ultra Relay
```typescript
// Step 1: Get entry point and kernel version
const entryPoint = getEntryPoint("0.7");
const kernelVersion = KERNEL_V3_1;

// Step 2: Create wallet client for authorization
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

// Step 3: Create validator
const validator = await signerToEcdsaValidator(publicClient, {
  entryPoint,
  kernelVersion,
  signer: account,
});

// Step 4: Create kernel account
const kernelAccount = await createKernelAccount(publicClient as any, {
  entryPoint,
  kernelVersion,
  plugins: { sudo: validator },
});
```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript
// Single step: Create kernel account using Gelato's kernel function
const account = await kernel({
  owner: signer,
  client: publicClient,
  eip7702: false,
});
```

**Migration Change**: Replace the 4-step ZeroDev account creation with Gelato's single `kernel()` function call.

### 2. Client Creation

#### ZeroDev SDK + Ultra Relay
```typescript
const kernelClient = createKernelAccountClient({
  account: kernelAccount,
  chain: baseSepolia,
  bundlerTransport: http(process.env.NEXT_PUBLIC_ULTRA_RELAY_URL || ""),
  paymaster: undefined,
  userOperation: {
    estimateFeesPerGas: async ({ bundlerClient }) => {
      return getUserOperationGasPrice(bundlerClient);
    },
  },
});
```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript
// Step 1: Create wallet client
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

// Step 2: Create Gelato smart wallet client
const smartWalletClient = await createGelatoSmartWalletClient(
  walletClient,
  {
    apiKey: process.env.NEXT_PUBLIC_SPONSOR_API_KEY || "",
  }
);
```

**Migration Change**: Replace ZeroDev's `createKernelAccountClient` with Gelato's two-step process using `createWalletClient` and `createGelatoSmartWalletClient`.

### 3. Call Data Preparation

#### ZeroDev SDK + Ultra Relay
```typescript

const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ] as const;

const callData = await kernelClient.account.encodeCalls(calls);

```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript

const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ];

const preparedCalls = await smartWalletClient.prepare({
  payment: sponsored(process.env.NEXT_PUBLIC_SPONSOR_API_KEY || ""),
  calls,
});
```

**Migration Change**: 
- Replace ZeroDev's `encodeCalls()` with Gelato's `prepare()` method

### 4. Transaction Execution

#### ZeroDev SDK + Ultra Relay
```typescript
const userOpHash = await kernelClient.sendUserOperation({
  callData,
  maxFeePerGas: BigInt(0),
  maxPriorityFeePerGas: BigInt(0),
});

const userOpReceipt = await kernelClient.waitForUserOperationReceipt({
  hash: userOpHash,
});
const hash = userOpReceipt.receipt.transactionHash;
```

#### Gelato Smart Wallet SDK + Kernel Wallet
```typescript
const results = await smartWalletClient.send({ preparedCalls });
const hash = await results?.wait();
```

**Migration Change**: 
- Replace ZeroDev's `sendUserOperation()` and `waitForUserOperationReceipt()` with Gelato's `send()` and `wait()`
- Change from `userOpHash` to `results.id` for logging


## Complete Migration Example

### From ZeroDev to Gelato

```typescript
// BEFORE: ZeroDev SDK + Ultra Relay
const entryPoint = getEntryPoint("0.7");
const kernelVersion = KERNEL_V3_3_BETA;
const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

const validator = await signerToEcdsaValidator(publicClient, {
  entryPoint,
  kernelVersion,
  signer: account,
});
const kernelAccount = await createKernelAccount(publicClient as any, {
  entryPoint,
  kernelVersion,
  plugins: { sudo: validator },
});

const kernelClient = createKernelAccountClient({
  account: kernelAccount,
  chain: baseSepolia,
  bundlerTransport: http(process.env.NEXT_PUBLIC_ULTRA_RELAY_URL || ""),
  paymaster: undefined,
  userOperation: {
    estimateFeesPerGas: async ({ bundlerClient }) => {
      return getUserOperationGasPrice(bundlerClient);
    },
  },
});


const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ] as const;

const callData = await kernelClient.account.encodeCalls(calls);

const userOpHash = await kernelClient.sendUserOperation({
  callData,
  maxFeePerGas: BigInt(0),
  maxPriorityFeePerGas: BigInt(0),
});
const userOpReceipt = await kernelClient.waitForUserOperationReceipt({
  hash: userOpHash,
});
const hash = userOpReceipt.receipt.transactionHash;
```

```typescript
// AFTER: Gelato Smart Wallet SDK + Kernel Wallet
const account = await kernel({
  owner: signer,
  client: publicClient,
  eip7702: false,
});

const walletClient = createWalletClient({
  account,
  chain: baseSepolia,
  transport: http(""),
});

const smartWalletClient = await createGelatoSmartWalletClient(
  walletClient,
  {
    apiKey: process.env.SPONSOR_API_KEY || "",
  }
);

const calls = [
    {
      to: "0x0000000000000000000000000000000000000000" as `0x${string}`,
      value: BigInt(0),
      data: "0x",
    },
  ];

const preparedCalls = await smartWalletClient.prepare({
  payment: sponsored(process.env.SPONSOR_API_KEY || ""),
  calls,
});

const results = await smartWalletClient.send({ preparedCalls });
const hash = await results?.wait();
```


## Migration Checklist

### 1. Dependencies
- [ ] Remove `@zerodev/sdk` imports
- [ ] Add `@gelatonetwork/smartwallet` imports

### 2. Account Creation
- [ ] Replace ZeroDev account creation with Gelato's `kernel()` function
- [ ] Remove entry point and kernel version configuration
- [ ] Remove validator creation

### 3. Client Setup
- [ ] Replace `createKernelAccountClient` with `createWalletClient` + `createGelatoSmartWalletClient`
- [ ] Update bundler transport configuration
- [ ] Remove paymaster configuration

### 4. Transaction Flow
- [ ] Replace `encodeCalls()` with `prepare()`
- [ ] Replace `sendUserOperation()` with `send()`
- [ ] Replace `waitForUserOperationReceipt()` with `wait()`
- [ ] Update transaction hash extraction

### 5. Error Handling
- [ ] Update error messages and logging
- [ ] Ensure retry logic remains compatible

## Summary

The main differences between the two approaches are:

1. **Account Creation**: ZeroDev requires 4 steps vs Gelato's single function call
2. **Client Setup**: ZeroDev uses one client vs Gelato's two-client approach
3. **Transaction Flow**: ZeroDev uses `encodeCalls()` + `sendUserOperation()` vs Gelato's `prepare()` + `send()`