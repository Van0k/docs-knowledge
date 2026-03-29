# Multicall Operations

Detailed guides for each multicall operation in TypeScript.

> For Solidity multicall encoding, see [Multicalls](../../solidity-guide/multicalls.md).

## Why This Section?

The main [multicalls.md](../multicalls.md) covers the basics: SDK service helpers, combining with raw encoding, and a complete example. This section goes deeper on each operation - when you need it, complete examples, and what can go wrong.

## Quick Reference

| Operation           | SDK Helper                    | When to Use                                | Guide                                                   |
| ------------------- | ----------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| Add Collateral      | `prepareAddCollateral()`      | Deposit tokens to increase health factor   | [Adding Collateral](./adding-collateral.md)             |
| Increase Debt       | `prepareIncreaseDebt()`       | Borrow from pool                           | [Debt Management](./debt-management.md)                 |
| Decrease Debt       | `prepareDecreaseDebt()`       | Repay borrowed funds                       | [Debt Management](./debt-management.md)                 |
| Update Quota        | `prepareUpdateQuota()`        | Enable/adjust quota token exposure         | [Updating Quotas](./updating-quotas.md)                 |
| Withdraw Collateral | `prepareWithdrawCollateral()` | Remove tokens from account                 | [Withdrawing Collateral](./withdrawing-collateral.md)   |
| Slippage Control    | Manual encoding               | Protect swaps from sandwich attacks        | [Controlling Slippage](./controlling-slippage.md)       |
| External Calls      | Manual encoding               | Interact with Uniswap, Curve, etc.         | [Making External Calls](./making-external-calls.md)     |
| Bot Permissions     | Manual encoding               | Grant/revoke bot access to account actions | [Setting Bot Permissions](./set-bot-permissions.md)     |
| Price Updates       | Manual encoding               | Update Pyth/Redstone feeds                 | [Updating Price Feeds](./updating-price-feeds.md)       |
| Check Params        | Manual encoding               | Optimize gas, set min health factor        | [Collateral Check Params](./collateral-check-params.md) |

## Page Structure

Each operation guide follows the same structure:

1. **Why** - When you need this operation
2. **What** - What it does and how it fits the system
3. **How** - Working TypeScript code
4. **Gotchas** - Common mistakes and edge cases
5. **See Also** - Related operations and Solidity reference

## SDK Helpers vs Manual Encoding

**Five operations have SDK helpers** via `createCreditAccountService`:

- `prepareAddCollateral(token, amount)`
- `prepareIncreaseDebt(amount)`
- `prepareDecreaseDebt(amount)`
- `prepareUpdateQuota(token, change, minQuota)`
- `prepareWithdrawCollateral(token, amount, to)`

**These operations require manual encoding** with viem's `encodeFunctionData`:

- `storeExpectedBalances` / `compareBalances`
- `onDemandPriceUpdates`
- `setFullCheckParams`
- `setBotPermissions`

All manual encoding uses `iCreditFacadeV300MulticallAbi` from `@gearbox-protocol/sdk`.

## Related

- [Multicalls Overview](../multicalls.md) - Basic patterns and complete example
- [Credit Accounts](../credit-accounts.md) - Account data and services
- [Solidity Multicalls](../../solidity-guide/multicalls.md) - On-chain implementation details
