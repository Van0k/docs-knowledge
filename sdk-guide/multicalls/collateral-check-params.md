# Collateral Check Params

Optimize gas and set minimum health factor for collateral checks.

> For Solidity implementation, see [Setting Collateral Check Params](../../solidity-guide/multicalls.md#setting-collateral-check-params).

## Why

You set collateral check params when:

- **Optimizing gas** - Hint which tokens cover the debt to skip unnecessary oracle calls
- **Risk management** - Enforce a minimum health factor above 1.0
- **Large accounts** - Many enabled tokens make default checks expensive
- **Automated systems** - Bots can benefit from consistent gas costs

The collateral check iterates through enabled tokens, summing value until it exceeds debt. Hints tell it which tokens to check first, potentially skipping expensive oracle calls.

## What

`setFullCheckParams` configures two things:

1. **Collateral hints** - Token masks to prioritize during the check
2. **Min health factor** - Minimum acceptable HF (in basis points, 10000 = 1.0)

If you know your USDC and WETH cover the debt, pass their masks as hints. The check evaluates them first and may stop early without checking other tokens.

## How

### Basic Usage with Hints

```typescript
import { encodeFunctionData } from "viem";
import {
  iCreditFacadeV300MulticallAbi,
  creditManagerAbi,
} from "@gearbox-protocol/sdk";
import { getContract } from "viem";

const creditManager = getContract({
  address: cmAddress,
  abi: creditManagerAbi,
  client: publicClient,
});

// Get token masks (each token has a unique bitmask)
const usdcMask = await creditManager.read.getTokenMaskOrRevert([usdcAddress]);
const wethMask = await creditManager.read.getTokenMaskOrRevert([wethAddress]);

const calls = [
  // Set hints at the start of multicall
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "setFullCheckParams",
      args: [
        [usdcMask, wethMask], // Check these tokens first
        10000, // minHealthFactor: 1.0 (10000 bps)
      ],
    }),
  },

  // Rest of your multicall
  service.prepareAddCollateral(usdcAddress, amount),
  // ...
];
```

### Setting Higher Min Health Factor

Require account to maintain at least 1.2 HF:

```typescript
const MIN_HF_120 = 12000; // 1.2 in basis points

const calls = [
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "setFullCheckParams",
      args: [
        [], // No hints
        MIN_HF_120,
      ],
    }),
  },
  // Operations...
];
```

### Complete Example: Gas-Optimized Multicall

```typescript
import { encodeFunctionData, getContract } from "viem";
import {
  GearboxSDK,
  createCreditAccountService,
  iCreditFacadeV300MulticallAbi,
  creditManagerAbi,
} from "@gearbox-protocol/sdk";

const sdk = await GearboxSDK.attach({ client, marketConfigurators: [] });
const service = createCreditAccountService(sdk, 310);
const market = sdk.marketRegister.findByCreditManager(cmAddress);

const creditManager = getContract({
  address: market.creditManager.address,
  abi: creditManagerAbi,
  client: publicClient,
});

// Get masks for tokens that will cover the debt
const usdcMask = await creditManager.read.getTokenMaskOrRevert([usdcAddress]);

const calls = [
  // Hints first - USDC will cover most of debt
  {
    target: market.creditFacade.address,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "setFullCheckParams",
      args: [
        [usdcMask], // USDC covers debt, check it first
        10500, // Require 1.05 HF minimum
      ],
    }),
  },

  service.prepareAddCollateral(usdcAddress, 100_000n * 10n ** 6n),
  service.prepareIncreaseDebt(200_000n * 10n ** 6n),

  // Swap some USDC to WETH
  {
    target: uniswapAdapter,
    callData: encodeSwap(/* ... */),
  },
];
```

## Gotchas

### Masks, Not Addresses

The hints array takes token **masks**, not addresses:

```typescript
// WRONG - passing addresses
args: [[usdcAddress, wethAddress], 10000];

// CORRECT - passing masks
const usdcMask = await creditManager.read.getTokenMaskOrRevert([usdcAddress]);
args: [[usdcMask], 10000];
```

### Hints Are Optimization, Not Guarantee

The check still validates ALL enabled tokens - hints just change the order. If hints don't cover the debt, it continues with remaining tokens.

```typescript
// If USDC hint doesn't cover debt, WETH and other tokens are still checked
// Hints just potentially skip some oracle calls
```

### Min Health Factor Must Be >= 10000

You cannot set a health factor below 1.0:

```typescript
// WRONG - less than 10000 reverts
args: [[], 9500]; // Reverts!

// CORRECT - must be >= 10000
args: [[], 10000]; // Exactly 1.0
args: [[], 11000]; // 1.1
```

### Hints Don't Help Small Accounts

For accounts with few enabled tokens (< 5), hints add gas overhead without saving much. Only use for accounts with many tokens.

### Order Matters in Hints Array

Tokens are checked in the order you provide:

```typescript
// Check WETH first, then USDC
args: [[wethMask, usdcMask], 10000];

// Check USDC first, then WETH
args: [[usdcMask, wethMask], 10000];
```

Put your highest-value collateral first for best gas savings.

### Each Check Calls Oracle Once

Without hints, the check iterates through all enabled tokens by mask order until TWV >= debt. With hints:

1. Check hinted tokens first
2. If TWV >= debt, stop early
3. If not, continue with remaining tokens

Best case: hints cover debt, skip other oracle calls.
Worst case: hints don't help, all tokens checked anyway.

### Params Reset After Multicall

`setFullCheckParams` only affects the current multicall's final check. Next multicall uses defaults again.

### Computing Token Masks

Token masks are powers of 2, assigned sequentially when tokens are added to the Credit Manager:

```typescript
// First token: mask = 1 (2^0)
// Second token: mask = 2 (2^1)
// Third token: mask = 4 (2^2)
// etc.

// Always use getTokenMaskOrRevert to get the correct mask
const mask = await creditManager.read.getTokenMaskOrRevert([tokenAddress]);
```

### Can Combine with Other Params

Use hints for gas optimization AND min HF for risk management:

```typescript
args: [
  [primaryCollateralMask, secondaryCollateralMask], // Gas optimization
  11000, // Risk management: require 1.1 HF
];
```

## See Also

- [Updating Quotas](./updating-quotas.md) - Which quoted tokens contribute to collateral checks
- [Price Updates](./updating-price-feeds.md) - Oracle calls that hints can skip
- [Debt Management](./debt-management.md) - Debt determines what TWV must cover
