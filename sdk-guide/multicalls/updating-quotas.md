# Updating Quotas

Enable or adjust exposure to quota-based collateral tokens.

> For Solidity implementation, see [Updating Quotas](../../solidity-guide/multicalls.md#updating-quotas).

## Why

You update quotas when:

- **Enabling a quota token** - Required before that token counts as collateral
- **Increasing exposure** - Need more of a token to count toward health factor
- **Reducing exposure** - Lower quota to reduce quota interest costs
- **Closing positions** - Zero quotas before full debt repayment

Quotas control how much of a token's value counts as collateral. Without a quota, even holding a quota token contributes zero to your health factor.

## What

`updateQuota` changes your quota for a specific token:

1. If increasing from zero, the token is enabled as collateral
2. If decreasing to zero, the token is disabled as collateral
3. Quota increase may be limited by global capacity (per-pool limits)
4. Quota interest accrues based on your quota amount

**Key parameters:**

- `token` - The quota token address
- `quotaChange` - Delta to apply (positive = increase, negative = decrease)
- `minQuota` - Minimum acceptable resulting quota (prevents partial fills)

The `minQuota` parameter protects you: if the pool can only give you 80% of your requested quota, and you set `minQuota` to 100% of your request, the transaction reverts instead of accepting partial quota.

## How

```typescript
import { GearboxSDK, createCreditAccountService } from "@gearbox-protocol/sdk";

const sdk = await GearboxSDK.attach({ client, marketConfigurators: [] });
const service = createCreditAccountService(sdk, 310);
const market = sdk.marketRegister.findByCreditManager(cmAddress);

// Request 50,000 USDC worth of quota
const quotaAmount = 50_000n * 10n ** 6n;

const calls = [
  service.prepareUpdateQuota(wethAddress, quotaAmount, quotaAmount),
];

await market.creditFacade.write.multicall([creditAccountAddress, calls]);
```

### Decrease Quota

```typescript
// Decrease quota by 20,000 (negative change)
const decrease = -20_000n * 10n ** 6n;

const calls = [service.prepareUpdateQuota(wethAddress, decrease, 0n)];
```

### Zero Quota Entirely

Pass `type(int96).min` to disable quota completely:

```typescript
// int96 minimum value
const INT96_MIN = BigInt.asIntN(96, -1n * 2n ** 95n);

const calls = [service.prepareUpdateQuota(wethAddress, INT96_MIN, 0n)];
```

### Common Pattern: Enable Quota After Swap

After swapping into a quota token, you need to enable quota for it to count:

```typescript
const calls = [
  // Swap USDC to WETH via adapter
  {
    target: uniswapV3Adapter,
    callData: encodeFunctionData({
      abi: uniswapV3AdapterAbi,
      functionName: "exactInputSingle",
      args: [swapParams],
    }),
  },
  // Enable quota for the received WETH
  service.prepareUpdateQuota(wethAddress, quotaAmount, quotaAmount),
];
```

## Gotchas

### Check Quota Limits Before Requesting

Each quota token has a pool-wide limit. If the limit is reached, your request fails (or gets partial fill):

```typescript
// Check available quota capacity
const quotaKeeper = market.quotaKeeper;
const quotaInfo = await quotaKeeper.read.getQuotaInfo([wethAddress]);

const available = quotaInfo.limit - quotaInfo.totalQuoted;
if (requested > available) {
  console.log(`Only ${available} quota available, requested ${requested}`);
}
```

### minQuota Prevents Partial Fills

If you need exactly 100 units of quota:

```typescript
// SAFE - reverts if less than 100 available
service.prepareUpdateQuota(token, 100n, 100n);

// RISKY - accepts partial fill
service.prepareUpdateQuota(token, 100n, 0n);
```

### Per-Account Quota Maximum

Each account has an implicit max quota of `8 * maxDebt` per asset. You cannot exceed this even if pool capacity exists.

### Zero Quotas Before Zero Debt

You cannot have active quotas with zero debt. When closing an account:

```typescript
const INT96_MIN = BigInt.asIntN(96, -1n * 2n ** 95n);
const MAX_UINT256 = 2n ** 256n - 1n;

const calls = [
  // Zero ALL quotas first
  service.prepareUpdateQuota(token1, INT96_MIN, 0n),
  service.prepareUpdateQuota(token2, INT96_MIN, 0n),
  // Then repay debt
  service.prepareDecreaseDebt(MAX_UINT256),
];
```

### Cannot Update Quotas on Zero-Debt Account

If your account has zero debt, quota updates fail. You must have active debt to hold quotas.

### Quota Tokens vs Non-Quota Tokens

Not all tokens are quota tokens. Non-underlying tokens that use **quota** must have a non-zero quota to count as collateral; quota changes enable or disable that token for collateral purposes. The underlying asset is treated separately by default.

Check whether a token is quota-based and how it is priced using the Credit Manager and market configuration.

### Forbidden Tokens Block Quota Increases

If your account has forbidden tokens enabled, you cannot increase any quotas. Disable forbidden tokens first.

## See Also

- [Debt Management](./debt-management.md) - Quotas require active debt
- [Adding Collateral](./adding-collateral.md) - Often combined with quota updates
- [Setting Bot Permissions](./set-bot-permissions.md) - Delegate quota updates to bots
