# Adding Collateral

Deposit tokens from your wallet to a Credit Account.

> For Solidity implementation, see [Adding Collateral](../../solidity-guide/multicalls.md#adding-collateral).

## Why

You need to add collateral when:

- **Opening an account** - Initial deposit to enable borrowing
- **Improving health factor** - Account approaching liquidation threshold
- **Enabling more borrowing** - Current collateral limits how much you can borrow

Adding collateral increases your account's total weighted value (TWV), which improves the health factor and allows larger debt positions.

## What

`addCollateral` transfers tokens from your wallet to the Credit Account. On execution:

1. The Credit Manager calls `transferFrom` to move tokens from caller to Credit Account
2. The token is enabled as collateral (if not already enabled)
3. Quoted tokens are NOT auto-enabled - you must set a quota separately

**Important:** Approve tokens to the **Credit Manager**, not the Credit Facade. The Credit Manager is the contract that actually executes the transfer.

## How

```typescript
import { GearboxSDK, createCreditAccountService } from "@gearbox-protocol/sdk";

const sdk = await GearboxSDK.attach({ client, marketConfigurators: [] });
const service = createCreditAccountService(sdk, 310);

// Build the multicall
const calls = [service.prepareAddCollateral(usdcAddress, 10_000n * 10n ** 6n)];

// First, approve to Credit Manager (not Facade!)
const market = sdk.marketRegister.findByCreditManager(cmAddress);
await usdcContract.write.approve([
  market.creditManager.address,
  10_000n * 10n ** 6n,
]);

// Execute on existing account
await market.creditFacade.write.multicall([creditAccountAddress, calls]);

// Or open new account with collateral
await market.creditFacade.write.openCreditAccount([
  ownerAddress,
  calls,
  0n, // referralCode
]);
```

### Using Permit (No Separate Approval)

For EIP-2612 compatible tokens, you can avoid the separate approval transaction:

```typescript
import { encodeFunctionData } from "viem";
import { iCreditFacadeV300MulticallAbi } from "@gearbox-protocol/sdk";

// Sign permit message (details depend on your wallet setup)
const { v, r, s, deadline } = await signPermit(/* ... */);

const calls = [
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "addCollateralWithPermit",
      args: [tokenAddress, amount, deadline, v, r, s],
    }),
  },
];
```

## Gotchas

### Approve to Credit Manager, Not Facade

The most common mistake. The Credit Manager executes the `transferFrom`, so it needs the approval:

```typescript
// CORRECT
await token.write.approve([creditManager.address, amount]);

// WRONG - will fail
await token.write.approve([creditFacade.address, amount]);
```

### Quoted Tokens Need Quota

Adding a quoted token as collateral does NOT automatically enable it. You must also call `updateQuota`:

```typescript
const calls = [
  service.prepareAddCollateral(quotedTokenAddress, amount),
  service.prepareUpdateQuota(quotedTokenAddress, quotaAmount, minQuota),
];
```

### Direct transfers are risky and do not replace `addCollateral`

Sending tokens directly to a Credit Account (`transfer`) does not run `addCollateral` and does not set up quota. For quoted collateral, value still does not count toward health until you `updateQuota` with active debt. Prefer `addCollateral` (and quota updates when needed) inside a multicall instead of relying on manual transfers.

### Invalid Collateral Tokens

Only tokens recognized by the Credit Manager can be used as collateral. Transferring unrecognized tokens to a Credit Account may result in them being stuck (only governance can recover).

Check if a token is valid:

```typescript
// This reverts if token is not valid collateral
const mask = await creditManager.read.getTokenMaskOrRevert([tokenAddress]);
```

## See Also

- [Withdrawing Collateral](./withdrawing-collateral.md) - The reverse operation
- [Debt Management](./debt-management.md) - Often combined with adding collateral
- [Updating Quotas](./updating-quotas.md) - Required for quoted tokens
