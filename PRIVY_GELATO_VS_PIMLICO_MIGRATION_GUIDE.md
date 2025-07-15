# Privy : LIGHT ACCOUNT + PIMLICO vs Gelato Smart Wallet SDK + LIGHT ACCOUNT Migration Guide

This guide provides a detailed comparison between two approaches for implementing light account smart wallets with different bundlers/paymasters services, focusing on the minimal code changes required to switch between implementations.

## Overview

- **SDK**: Permissionless vs Gelato Smart Wallet SDK (`@gelatonetwork/smartwallet`)
- **Bundlers/Paymasters**: Pimlico vs Gelato Relay
- **Account Type**: Light Account

## Integration Steps

## Core Differences

### Client Creation

#### Light Account + Pimlico

```typescript
const pimlicoSmartAccountClient = createSmartAccountClient({
  chain: arbitrum,
  account: smartAccountClient?.account,
  bundlerTransport: http(`/api/pimlico?chainId=${arbitrum.id}`),
  userOperation: {
    estimateFeesPerGas: async () => {
      return {
        maxFeePerGas: BigInt(0),
        maxPriorityFeePerGas: BigInt(0),
      };
    },
  },
  pollingInterval: 500,
});
```

#### Gelato Smart Wallet SDK + Light Account

**Imports**

```typescript
import { gelatoBundlerActions } from "@gelatonetwork/smartwallet/adapter";
import { sponsored, WalletEncoding } from "@gelatonetwork/smartwallet";
import { useWalletClient } from "wagmi";
```

**Code**

```typescript
const { data: walletClient } = useWalletClient();

const smartAccount = await toLightSmartAccount({
  client: publicClient as any,
  owner: walletClient as any,
  entryPoint: {
    address: entryPoint07Address,
    version: "0.7",
  },
  version: "2.0.0",
});

const gelatoSmartAccountClient = createSmartAccountClient({
  account: smartAccount,
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

**Migration Change**: Remove Pimlico’s client config and replace it with Gelato’s Smart Account Client Adapter.

### 3. Sending Transaction

#### Light Account + Pimlico

```typescript
const hash = await pimlicoSmartAccountClient.sendTransaction({
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
const userOpHash = await gelatoSmartAccountClient.sendUserOperation({
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
import { useCallback } from "react";
import { createSmartAccountClient } from "permissionless";
import { useSmartWallets } from "@privy-io/react-auth/smart-wallets";
import { arbitrum } from "viem/chains";
import { http } from "viem";

interface ICalls {
  to: string;
  data: string;
}

export const useSendSmartWalletOrder = () => {
  const { client: smartAccountClient } = useSmartWallets();

  const prepareAndSendUserOperation = useCallback(
    async (calls: ICalls[]) => {
      const pimlicoSmartAccountClient = createSmartAccountClient({
        chain: arbitrum,
        account: smartAccountClient?.account,
        bundlerTransport: http(`/api/pimlico?chainId=${arbitrum.id}`),
        userOperation: {
          estimateFeesPerGas: async () => {
            return {
              maxFeePerGas: BigInt(0),
              maxPriorityFeePerGas: BigInt(0),
            };
          },
        },
        pollingInterval: 500,
      });

      const signedTransaction =
        await pimlicoSmartAccountClient?.sendTransaction({
          calls: calls,
        });

      return {
        signedTransaction,
      };
    },
    [smartAccountClient?.account]
  );

  const sendSmartWalletOrder = useCallback(
    async (calls: ICalls[]) => {
      try {
        const { signedTransaction } = await prepareAndSendUserOperation(calls);
        return { hash: signedTransaction };
      } catch (err: any) {
        console.log("err", err);
      }
    },
    [prepareAndSendUserOperation]
  );

  return { sendSmartWalletOrder };
};
```

```typescript
// AFTER: Gelato Smart Wallet SDK + Light Account
import { useCallback } from "react";
import { createSmartAccountClient } from "permissionless";
import { arbitrum } from "viem/chains";
import { useWalletClient } from "wagmi";
import { http } from "viem";

interface ICalls {
  to: string;
  data: string;
}

export const useSendSmartWalletOrder = () => {
  const { data: walletClient } = useWalletClient();

  const prepareAndSendUserOperation = useCallback(
    async (calls: ICalls[]) => {
      const smartAccount = await toLightSmartAccount({
        client: publicClient as any,
        owner: walletClient as any,
        entryPoint: {
          address: entryPoint07Address,
          version: "0.7",
        },
        version: "2.0.0",
      });

      const gelatoSmartAccountClient = createSmartAccountClient({
        account: smartAccount,
        chain: arbitrumSepolia,
        // Important: Chain transport (chain rpc) must be passed here instead of bundler transport
        bundlerTransport: http(),
      }).extend(
        gelatoBundlerActions({
          payment: sponsored(process.env.NEXT_PUBLIC_SPONSOR_API_KEY as string),
          encoding: WalletEncoding.LightAccount,
        })
      );

      const userOpHash = await gelatoSmartAccountClient?.sendUserOperation({
        calls: calls,
      });

      const receipt =
        await gelatoSmartAccountClient.waitForUserOperationReceipt({
          hash: userOpHash,
        });

      const transactionHash = receipt.receipt.transactionHash;

      return {
        transactionHash,
      };
    },
    [smartAccountClient?.account]
  );

  const sendSmartWalletOrder = useCallback(
    async (calls: ICalls[]) => {
      try {
        const { transactionHash } = await prepareAndSendUserOperation(calls);
        return { hash: transactionHash };
      } catch (err: any) {
        console.log("err", err);
      }
    },
    [prepareAndSendUserOperation]
  );

  return { sendSmartWalletOrder };
};
```

## Summary

The main differences between the two approaches are:

1. **Account Creation**: The current implementation uses useSmartWallets() from Pyth, which creates an EntryPoint v0.6 account. However, this is not compatible with Gelato Smart Wallets, which require EntryPoint v0.7.

   So You must explicitly create a LightAccount using EntryPoint v0.7 to ensure compatibility with the Gelato Smart Wallet system.

2. **Client Creation**: When setting up the smart account client, extend it with `gelatoBundlerActions` to enable Gelato-specific functionalities.

3. **Transaction Flow**:
   - Pimlico: Uses `sendTransaction()` method
   - Gelato: Uses `sendUserOperation()` method
4. **Paymaster Integration**:
   - Pimlico: Explicit paymaster client configuration
   - Gelato: Built-in sponsored payment through bundler actions
