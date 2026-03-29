# Withdrawing Collateral

Remove tokens from a Credit Account to an external address.

> For SDK implementation, see [Withdrawing Collateral](../../sdk-guide/multicalls/withdrawing-collateral.md).

## Why

You need to withdraw collateral when:

- **Taking profits** - Remove excess collateral while maintaining healthy position
- **Rebalancing** - Move funds between accounts or protocols
- **Closing account** - Extract all remaining tokens before closing

Withdrawing collateral decreases your account's total weighted value (TWV), which lowers the health factor.

## What

`withdrawCollateral` transfers tokens from the Credit Account to a specified recipient. On execution:

1. Tokens are transferred from the Credit Account to the recipient
2. If withdrawing the full balance, the token may be disabled as collateral
3. Safe pricing is activated for the collateral check (uses min of main and reserve price feeds)

**Important:** Withdrawals trigger stricter collateral checks using safe prices, so ensure sufficient buffer above the liquidation threshold.

## How

```solidity
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";

address creditFacade;
address usdc;
address creditAccount;
address recipient;

MultiCall[] memory calls = new MultiCall[](1);

// Withdraw 5,000 USDC
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.withdrawCollateral,
        (usdc, 5_000 * 10**6, recipient)
    )
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

### Withdraw Full Balance

Use `type(uint256).max` to withdraw the entire token balance:

```solidity
// Withdraw all USDC from the account
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.withdrawCollateral,
        (usdc, type(uint256).max, recipient)
    )
});
```

### Withdraw During Closure

When closing an account, you can withdraw all tokens to yourself:

```solidity
MultiCall[] memory calls = new MultiCall[](2);

// Repay all debt first
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.decreaseDebt,
        (type(uint256).max)
    )
});

// Withdraw all collateral
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.withdrawCollateral,
        (token, type(uint256).max, recipient)
    )
});

ICreditFacadeV3(creditFacade).closeCreditAccount(creditAccount, calls);
```

## Gotchas

### Safe Prices Are Used

After withdrawal, collateral checks use safe pricing (minimum of main and reserve feeds). This means:

- Your effective collateral value may be lower than expected
- Maintain buffer above liquidation threshold
- Check safe prices before withdrawing large amounts

### Forbidden Tokens Block Withdrawal

If any forbidden token is enabled as collateral on the account, withdrawals are blocked in multicalls:

```solidity
// This will revert if the account has forbidden tokens enabled
ICreditFacadeV3(creditFacade).multicall(creditAccount, withdrawCalls);
```

Solution: First swap forbidden tokens to allowed ones, then withdraw.

### Phantom Token Unwrapping

If the token being withdrawn is a phantom token (e.g., staked position token), it's automatically unwrapped:

1. Phantom token is withdrawn from the vault/pool
2. The underlying deposited token is sent to the recipient
3. No slippage protection - assumed to happen at non-manipulatable rate

### Dust Handling

For clean account closure, use `type(uint256).max` to handle dust amounts:

```solidity
// This handles any remaining wei
callData: abi.encodeCall(
    ICreditFacadeV3Multicall.withdrawCollateral,
    (token, type(uint256).max, recipient)
)
```

### Recipient Validation

The recipient address must be a valid address. Common patterns:

```solidity
// Withdraw to account owner
withdrawCollateral(token, amount, owner);

// Withdraw to another protocol/contract
withdrawCollateral(token, amount, anotherContract);

// NEVER use address(0) - tokens will be lost!
```

## See Also

- [Adding Collateral](./adding-collateral.md) - The reverse operation
- [Debt Management](./debt-management.md) - Repay debt before large withdrawals
- [Updating Quotas](./updating-quotas.md) - Token collateral management through quota updates
