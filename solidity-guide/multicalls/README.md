# Multicall Operations

Detailed guides for encoding each multicall operation in Solidity.

> For TypeScript/SDK implementation, see [Multicalls](../../sdk-guide/multicalls/README.md).

## Why This Section?

The main [multicalls.md](../multicalls.md) covers the fundamentals: MultiCall struct encoding, call order, and the diff pattern. This section goes deeper on each operation - when you need it, complete Solidity examples, and what can go wrong.

## Quick Reference

| Operation           | Function                                    | When to Use                                | Guide                                                   |
| ------------------- | ------------------------------------------- | ------------------------------------------ | ------------------------------------------------------- |
| Add Collateral      | `addCollateral`                             | Deposit tokens to increase health factor   | [Adding Collateral](./adding-collateral.md)             |
| Increase Debt       | `increaseDebt`                              | Borrow from pool                           | [Debt Management](./debt-management.md)                 |
| Decrease Debt       | `decreaseDebt`                              | Repay borrowed funds                       | [Debt Management](./debt-management.md)                 |
| Update Quota        | `updateQuota`                               | Enable/adjust quota for a token exposure         | [Updating Quotas](./updating-quotas.md)                 |
| Withdraw Collateral | `withdrawCollateral`                        | Remove tokens from account                 | [Withdrawing Collateral](./withdrawing-collateral.md)   |
| Slippage Control    | `storeExpectedBalances` / `compareBalances` | Protect swaps from sandwich attacks        | [Controlling Slippage](./controlling-slippage.md)       |
| External Calls      | Adapter-specific                            | Interact with Uniswap, Curve, etc.         | [Making External Calls](./making-external-calls.md)     |
| Price Updates       | `onDemandPriceUpdate`                       | Update Pyth/Redstone feeds                 | [Updating Price Feeds](./updating-price-feeds.md)       |
| Check Params        | `setFullCheckParams`                        | Optimize gas, set min health factor        | [Collateral Check Params](./collateral-check-params.md) |
| Bot Permissions     | `setBotPermissions`                         | Grant/revoke bot access to account actions | [Setting Bot Permissions](./set-bot-permissions.md)     |

## Page Structure

Each operation guide follows the same structure:

1. **Why** - When you need this operation
2. **What** - What it does and how it fits the system
3. **How** - Working Solidity code
4. **Gotchas** - Common mistakes and edge cases
5. **See Also** - Related operations and SDK reference

## Core Encoding Pattern

All multicall operations use the same encoding pattern:

```solidity
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";

MultiCall[] memory calls = new MultiCall[](1);

calls[0] = MultiCall({
    target: creditFacade,  // or adapter address for external calls
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.functionName,
        (param1, param2, ...)
    )
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

## Call Order Requirements

Some operations have strict ordering rules:

1. **Price updates (`onDemandPriceUpdates`)** - Must be first in the calls array
2. **Collateral check params (`setFullCheckParams`)** - Should be early, affects final check
3. **External calls** - Can be anywhere after price updates
4. **Slippage checks** - `storeExpectedBalances` before swaps, `compareBalances` after

## Import Patterns

Standard imports for multicall operations:

```solidity
// Core interfaces
import {ICreditFacadeV3} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";
import {ICreditManagerV3} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditManagerV3.sol";

// For slippage protection
import {BalanceDelta} from "@gearbox-protocol/core-v3/contracts/libraries/BalancesLogic.sol";

// Adapter interfaces (examples)
import {IUniswapV3Adapter} from "@gearbox-protocol/integrations-v3/contracts/interfaces/uniswap/IUniswapV3Adapter.sol";
import {IYearnV2Adapter} from "@gearbox-protocol/integrations-v3/contracts/interfaces/yearn/IYearnV2Adapter.sol";
```

## Related

- [Multicalls Overview](../multicalls.md) - Fundamentals and the diff pattern
- [Credit Manager](../../reference/credit-manager.md) - Underlying execution layer
- [SDK Multicalls](../../sdk-guide/multicalls/README.md) - TypeScript implementation
