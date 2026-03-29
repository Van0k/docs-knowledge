# Multicalls

Build and execute multicalls using SDK service helpers.

> For Solidity multicall encoding, see [Multicalls](../solidity-guide/multicalls.md).

## Service Multicall Helpers

The SDK provides structured multicall builders via `createCreditAccountService`:

```typescript
import { GearboxSDK, createCreditAccountService } from "@gearbox-protocol/sdk";

const sdk = await GearboxSDK.attach({ client, marketConfigurators: [] });
const service = createCreditAccountService(sdk, 310);
```

## Available Service Methods

| Method                                         | Operation                  |
| ---------------------------------------------- | -------------------------- |
| `prepareAddCollateral(token, amount)`          | Add collateral from wallet |
| `prepareIncreaseDebt(amount)`                  | Borrow from pool           |
| `prepareDecreaseDebt(amount)`                  | Repay debt                 |
| `prepareUpdateQuota(token, change, minQuota)`  | Adjust token quota         |
| `prepareWithdrawCollateral(token, amount, to)` | Remove collateral          |

## Detailed Operation Guides

For comprehensive documentation of each operation:

**SDK Helper Operations:**

- [Adding Collateral](multicalls/adding-collateral.md) - Transfer tokens with approval patterns
- [Debt Management](multicalls/debt-management.md) - Borrowing, repayment, and constraints
- [Updating Quotas](multicalls/updating-quotas.md) - Quota mechanics and limits
- [Withdrawing Collateral](multicalls/withdrawing-collateral.md) - Safe pricing and health impact

**Manual Encoding Operations:**

- [Controlling Slippage](multicalls/controlling-slippage.md) - Balance delta protection
- [Making External Calls](multicalls/making-external-calls.md) - Adapter interaction patterns
- [Updating Price Feeds](multicalls/updating-price-feeds.md) - On-demand oracle data
- [Collateral Check Params](multicalls/collateral-check-params.md) - Health check optimization
- [Setting Bot Permissions](multicalls/set-bot-permissions.md) - Bot access control

## Building a Multicall

```typescript
// Build multicall with SDK helpers
const calls = [
  // Add 10,000 USDC as collateral
  service.prepareAddCollateral(usdcAddress, 10_000n * 10n ** 6n),

  // Borrow 40,000 USDC (5x leverage)
  service.prepareIncreaseDebt(40_000n * 10n ** 6n),

  // Set quota for destination token
  service.prepareUpdateQuota(
    wethAddress,
    50_000n * 10n ** 6n,
    50_000n * 10n ** 6n,
  ),
];
```

## Executing Multicalls

### On Existing Account

```typescript
const market = sdk.marketRegister.findByCreditManager(cmAddress);

await market.creditFacade.write.multicall([creditAccountAddress, calls]);
```

### Opening with Multicall

```typescript
const hash = await market.creditFacade.write.openCreditAccount([
  ownerAddress,
  calls,
  0n, // referralCode
]);
```

## Combining SDK Helpers with Raw Encoding

For adapter calls or custom operations, combine SDK helpers with manual encoding:

```typescript
import { encodeFunctionData } from "viem";
import { iCreditFacadeV300MulticallAbi } from "@gearbox-protocol/sdk";

const calls = [
  // SDK helpers for standard operations
  service.prepareAddCollateral(usdcAddress, 10_000n * 10n ** 6n),
  service.prepareIncreaseDebt(40_000n * 10n ** 6n),

  // Manual encoding for adapter calls
  {
    target: uniswapV3Adapter,
    callData: encodeFunctionData({
      abi: uniswapV3AdapterAbi,
      functionName: "exactInputSingle",
      args: [
        {
          tokenIn: usdcAddress,
          tokenOut: wethAddress,
          fee: 500,
          recipient: "0x0000000000000000000000000000000000000000", // Adapter overrides
          deadline: BigInt(Math.floor(Date.now() / 1000) + 3600),
          amountIn: 50_000n * 10n ** 6n,
          amountOutMinimum: 0n,
          sqrtPriceLimitX96: 0n,
        },
      ],
    }),
  },

  // SDK helper for quota
  service.prepareUpdateQuota(
    wethAddress,
    50_000n * 10n ** 6n,
    50_000n * 10n ** 6n,
  ),
];
```

## Slippage Protection

Add slippage checks around external calls:

```typescript
const calls = [
  // Store expected balances before swap
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "storeExpectedBalances",
      args: [[{ token: wethAddress, amount: minExpectedWeth }]],
    }),
  },

  // Swap operation
  {
    target: uniswapV3Adapter,
    callData: encodeFunctionData({
      abi: uniswapV3AdapterAbi,
      functionName: "exactInputSingle",
      args: [swapParams],
    }),
  },

  // Compare balances after swap
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "compareBalances",
      args: [],
    }),
  },
];
```

## On-Demand Price Updates

If using pull-based oracles, add price updates first:

```typescript
const calls = [
  // Price updates must be first
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "onDemandPriceUpdates",
      args: [
        [
          {
            priceFeed: wethPriceFeedAddress,
            data: priceData,
          },
        ],
      ],
    }),
  },

  // Then standard operations
  service.prepareAddCollateral(usdcAddress, amount),
  service.prepareIncreaseDebt(debtAmount),
];
```

## Getting Adapter Addresses

Retrieve adapter addresses from the Credit Manager:

```typescript
import { getContract } from "viem";
import { creditManagerAbi } from "@gearbox-protocol/sdk";

const creditManager = getContract({
  address: cmAddress,
  abi: creditManagerAbi,
  client: publicClient,
});

// Get adapter for a protocol (e.g., Uniswap V3 Router)
const uniswapV3Adapter = await creditManager.read.contractToAdapter([
  UNISWAP_V3_ROUTER,
]);
```

## Complete Example

```typescript
import {
  GearboxSDK,
  createCreditAccountService,
  iCreditFacadeV300MulticallAbi,
} from "@gearbox-protocol/sdk";
import {
  encodeFunctionData,
  createPublicClient,
  createWalletClient,
  http,
} from "viem";
import { mainnet } from "viem/chains";

async function leveragePosition() {
  const publicClient = createPublicClient({
    chain: mainnet,
    transport: http(),
  });

  const sdk = await GearboxSDK.attach({
    client: publicClient,
    marketConfigurators: [],
  });

  const service = createCreditAccountService(sdk, 310);

  // Find market
  const market = sdk.marketRegister.findByCreditManager(cmAddress);

  // Build multicall
  const calls = [
    // Add collateral
    service.prepareAddCollateral(usdcAddress, 10_000n * 10n ** 6n),

    // Borrow
    service.prepareIncreaseDebt(40_000n * 10n ** 6n),

    // Set quota for final token
    service.prepareUpdateQuota(
      targetToken,
      50_000n * 10n ** 6n,
      50_000n * 10n ** 6n,
    ),
  ];

  // Execute
  const walletClient = createWalletClient({
    chain: mainnet,
    transport: http(),
    account: myAccount,
  });

  const hash = await walletClient.writeContract({
    address: market.creditFacade.address,
    abi: creditFacadeAbi,
    functionName: "openCreditAccount",
    args: [myAccount.address, calls, 0n],
  });

  console.log(`Transaction: ${hash}`);
}
```

## Best Practices

1. **Always use slippage protection** when performing swaps
2. **Price updates first** if using pull-based oracles
3. **Set quotas** for tokens you'll hold as collateral
4. **Approve collateral** to Credit Manager (not Facade) before adding

For architectural background, see [Multicall System](../concepts/multicall-system.md).
