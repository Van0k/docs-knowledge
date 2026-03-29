# Updating Quotas

Manage collateral quotas for quoted tokens.

> For SDK implementation, see [Updating Quotas](../../sdk-guide/multicalls/updating-quotas.md).

## Why

Quotas are required for non-underlying tokens to count as collateral:

- **Enable collateral** - Purchase quota to enable a token as collateral
- **Increase exposure** - Raise quota limit to hold more of a token
- **Reduce fees** - Lower quota to reduce ongoing quota fees
- **Disable collateral** - Set quota to zero to disable token

Without a quota, holding a token on a Credit Account with debt risks losing it to liquidators since it doesn't count toward your health factor.

## What

`updateQuota` changes the quota for a specific token. On execution:

1. Quota change is applied (can be positive or negative)
2. If quota goes from zero to positive, token is enabled as collateral
3. If quota goes to zero, token is disabled as collateral
4. Quota fees are applied based on the change

| Parameter     | Type    | Description                                    |
| ------------- | ------- | ---------------------------------------------- |
| `token`       | address | The quoted token (cannot be underlying)        |
| `quotaChange` | int96   | Delta in underlying units (negative to reduce) |
| `minQuota`    | uint96  | Minimum resulting quota to not revert          |

## How

### Enable a Token as Collateral

```solidity
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";

address creditFacade;
address creditAccount;
address weth;

MultiCall[] memory calls = new MultiCall[](1);

// Enable WETH with 50,000 USDC quota
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (weth, int96(50_000 * 10**6), uint96(50_000 * 10**6))
    )
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

### Increase Existing Quota

```solidity
// Add 25,000 USDC to existing quota
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (weth, int96(25_000 * 10**6), uint96(75_000 * 10**6))  // minQuota = expected total
    )
});
```

### Decrease Quota

```solidity
// Reduce quota by 20,000 USDC
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (weth, int96(-20_000 * 10**6), uint96(0))  // minQuota = 0 (any remaining is fine)
    )
});
```

### Disable Token Completely

Use `type(int96).min` to fully disable:

```solidity
// Completely disable WETH as collateral
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (weth, type(int96).min, uint96(0))
    )
});
```

### Combined: Add Token + Set Quota

When adding a quoted token as collateral, always pair with quota update:

```solidity
MultiCall[] memory calls = new MultiCall[](2);

// 1. Add the token
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.addCollateral,
        (quotedToken, amount)
    )
});

// 2. Enable it with quota
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (quotedToken, int96(quotaAmount), uint96(quotaAmount))
    )
});
```

## Gotchas

### Account Must Have Debt

Quotas can only be updated when the account has non-zero debt:

```solidity
// This reverts if debt == 0
ICreditFacadeV3Multicall.updateQuota(token, change, minQuota);
```

For zero-debt accounts, first borrow, then update quotas.

### Forbidden Tokens Cannot Increase

If a token is forbidden, you cannot increase its quota:

```solidity
// Reverts if WETH is forbidden
updateQuota(weth, int96(1000 * 10**6), 0);

// But you CAN decrease or disable
updateQuota(weth, int96(-1000 * 10**6), 0);  // OK
updateQuota(weth, type(int96).min, 0);        // OK
```

### Quota Limits

Each market has per-account quota limits:

```solidity
// Check the limit
uint96 maxQuota = IPoolQuotaKeeperV3(quotaKeeper).getQuotaLimit(token);

// Resulting quota must be <= maxQuota
```

### Quota Fees

Quotas incur ongoing fees:

- **Increase fee**: One-time fee on quota increase
- **Quota rate**: Ongoing rate (similar to interest) on quota amount

Check rates before setting large quotas:

```solidity
(uint16 rate, uint192 cumulativeIndexLU, uint96 quotaIncreaseFee) =
    IPoolQuotaKeeperV3(quotaKeeper).getQuotaRate(token);
```

### minQuota Parameter

Use `minQuota` to prevent front-running:

```solidity
// If someone reduces your quota before your tx, this reverts instead of accepting less
updateQuota(token, int96(50_000e6), uint96(50_000e6));

// With minQuota = 0, any resulting quota is accepted (more gas efficient but risky)
updateQuota(token, int96(50_000e6), uint96(0));
```

### Zero Debt Closure Requirement

Before closing with zero debt, all quotas must be disabled:

```solidity
MultiCall[] memory calls = new MultiCall[](3);

// 1. Disable all quotas
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (quotedToken1, type(int96).min, 0)
    )
});

calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (quotedToken2, type(int96).min, 0)
    )
});

// 2. Repay all debt
calls[2] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.decreaseDebt,
        (type(uint256).max)
    )
});
```

## See Also

- [Adding Collateral](./adding-collateral.md) - Add tokens before setting quota
- [Debt Management](./debt-management.md) - Debt required for quota updates
- [Setting Bot Permissions](./set-bot-permissions.md) - Delegate quota updates to bots
