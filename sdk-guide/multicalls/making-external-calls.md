# Making External Calls

Interact with external protocols (Uniswap, Curve, Yearn, etc.) from your Credit Account.

> For Solidity implementation, see [Making External Calls](../../solidity-guide/multicalls.md#making-external-calls).

## Why

You make external calls when:

- **Swapping tokens** - Trade via Uniswap, Curve, or other DEXs
- **Depositing to vaults** - Stake in Yearn, Lido, or yield strategies
- **Managing LP positions** - Add/remove liquidity on various protocols
- **Executing complex strategies** - Chain multiple protocol interactions

Credit Accounts interact with external protocols through **adapters** - whitelisted contracts that translate your calls into safe operations.

## What

External calls flow through adapters:

1. You encode a call targeting an adapter address
2. Credit Facade routes the call to the adapter
3. Adapter builds the actual calldata for the external protocol
4. Adapter requests token approvals if needed
5. Credit Manager executes the call from the Credit Account
6. Credit Account acts as the "user" from the external protocol's perspective
7. Adapter returns which tokens to enable/disable based on the operation

**Key insight:** The Credit Account makes the actual call, so it receives the output tokens directly. You never touch the funds - they stay in the Credit Account.

## How

### Step 1: Get Adapter Address

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

// Returns 0x0 if no adapter exists for this protocol
if (uniswapV3Adapter === "0x0000000000000000000000000000000000000000") {
  throw new Error("No adapter for this protocol");
}
```

### Step 2: Encode the Adapter Call

```typescript
import { encodeFunctionData } from "viem";
import { uniswapV3AdapterAbi } from "@gearbox-protocol/integrations-v3";

const swapParams = {
  tokenIn: usdcAddress,
  tokenOut: wethAddress,
  fee: 500,
  recipient: "0x0000000000000000000000000000000000000000", // Adapter overrides this
  deadline: BigInt(Math.floor(Date.now() / 1000) + 3600),
  amountIn: 50_000n * 10n ** 6n,
  amountOutMinimum: 24n * 10n ** 18n, // Slippage protection
  sqrtPriceLimitX96: 0n,
};

const calls = [
  {
    target: uniswapV3Adapter,
    callData: encodeFunctionData({
      abi: uniswapV3AdapterAbi,
      functionName: "exactInputSingle",
      args: [swapParams],
    }),
  },
];
```

### Complete Example: Swap with Slippage Protection

```typescript
import { encodeFunctionData } from "viem";
import {
  GearboxSDK,
  createCreditAccountService,
  iCreditFacadeV300MulticallAbi,
} from "@gearbox-protocol/sdk";

const sdk = await GearboxSDK.attach({ client, marketConfigurators: [] });
const service = createCreditAccountService(sdk, 310);
const market = sdk.marketRegister.findByCreditManager(cmAddress);

// Get adapter
const uniswapV3Adapter = await market.creditManager.read.contractToAdapter([
  UNISWAP_V3_ROUTER,
]);

const calls = [
  // Add collateral (SDK helper)
  service.prepareAddCollateral(usdcAddress, 50_000n * 10n ** 6n),

  // Borrow (SDK helper)
  service.prepareIncreaseDebt(200_000n * 10n ** 6n),

  // Slippage protection start
  {
    target: market.creditFacade.address,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "storeExpectedBalances",
      args: [[{ token: wethAddress, amount: 99n * 10n ** 18n }]], // Expect ~100 WETH
    }),
  },

  // Swap via adapter (manual encoding)
  {
    target: uniswapV3Adapter,
    callData: encodeFunctionData({
      abi: uniswapV3AdapterAbi,
      functionName: "exactInputSingle",
      args: [swapParams],
    }),
  },

  // Slippage protection end
  {
    target: market.creditFacade.address,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "compareBalances",
      args: [],
    }),
  },

  // Set quota for received token (SDK helper)
  service.prepareUpdateQuota(
    wethAddress,
    250_000n * 10n ** 6n,
    250_000n * 10n ** 6n,
  ),
];

// Approve collateral to Credit Manager
await usdcContract.write.approve([
  market.creditManager.address,
  50_000n * 10n ** 6n,
]);

// Execute
await market.creditFacade.write.openCreditAccount([ownerAddress, calls, 0n]);
```

### Diff Functions

Many adapters have `_diff` variants that operate on "entire balance minus 1":

```typescript
// Instead of specifying exact amount...
{ functionName: 'deposit', args: [exactAmount] }

// Use diff to deposit all USDC (minus 1 wei)
{ functionName: 'depositDiff', args: [1n] }
```

This is useful when you don't know the exact balance after previous operations.

## Gotchas

### Adapter ABIs Need Separate Import

SDK exports core ABIs, but adapter ABIs often need separate import:

```typescript
// Core ABIs from SDK
import { iCreditFacadeV300MulticallAbi } from "@gearbox-protocol/sdk";

// Adapter ABIs from integrations package
import { uniswapV3AdapterAbi } from "@gearbox-protocol/integrations-v3";
```

Check what's available in `@gearbox-protocol/integrations-v3`.

### Not All Protocols Have Adapters

An adapter must exist for each protocol you want to interact with. Check with `contractToAdapter`:

```typescript
const adapter = await creditManager.read.contractToAdapter([protocolAddress]);

if (adapter === "0x0000000000000000000000000000000000000000") {
  // No adapter - this protocol isn't integrated
  throw new Error("Protocol not supported");
}
```

### Recipient Parameter is Overridden

Many DEX functions have a `recipient` parameter. Adapters override this to ensure tokens go to the Credit Account, not an arbitrary address:

```typescript
// You can pass any address here - adapter ignores it
const swapParams = {
  recipient: "0x0000000000000000000000000000000000000000", // Will be overridden
  // ...
};
```

### Always Use Slippage Protection

External calls are vulnerable to sandwich attacks. Always wrap swaps with slippage checks:

```typescript
const calls = [storeExpectedBalances, adapterSwapCall, compareBalances];
```

### Adapter Function Signatures May Differ

Adapter functions may have slightly different signatures than the underlying protocol:

```typescript
// Uniswap Router: exactInputSingle(params) returns (uint256)
// Adapter: exactInputSingle(params) -> also enables output token

// Yearn Vault: deposit(amount) returns (shares)
// Adapter: depositDiff(leftoverAmount) -> deposits all balance minus leftoverAmount
```

Read the adapter interface documentation for exact signatures.

### Quoted tokens still need quota

Adapter swaps can change balances, but **quota tokens** only count toward collateral after a successful `updateQuota` (with active debt). Plan the multicall so quota updates happen when you intend an asset to contribute to health.

## See Also

- [Controlling Slippage](./controlling-slippage.md) - Protect your swaps
- [Multicalls Overview](../multicalls.md) - Combining SDK helpers with manual encoding
- [Updating Quotas](./updating-quotas.md) - Enable collateral for quota tokens
