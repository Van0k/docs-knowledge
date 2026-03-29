# Multicalls

Build and execute multicalls in Solidity.

> For SDK multicall helpers, see [Multicalls](../sdk-guide/multicalls.md).

## Detailed Operation Guides

For comprehensive documentation of each operation:

- [Adding Collateral](multicalls/adding-collateral.md) - Transfer tokens with approval patterns
- [Debt Management](multicalls/debt-management.md) - Borrowing, repayment, and constraints
- [Updating Quotas](multicalls/updating-quotas.md) - Quota mechanics and limits
- [Withdrawing Collateral](multicalls/withdrawing-collateral.md) - Safe pricing and health impact
- [Controlling Slippage](multicalls/controlling-slippage.md) - Balance delta protection
- [Making External Calls](multicalls/making-external-calls.md) - Adapter interaction patterns
- [Updating Price Feeds](multicalls/updating-price-feeds.md) - On-demand oracle data
- [Collateral Check Params](multicalls/collateral-check-params.md) - Health check optimization
- [Setting Bot Permissions](multicalls/set-bot-permissions.md) - Configure bot access control

## The MultiCall Structure

```solidity
struct MultiCall {
    address target;   // CreditFacade or allowed Adapter
    bytes callData;   // Encoded function call
}
```

## ICreditFacadeV3Multicall Operations

All multicall operations are defined in `ICreditFacadeV3Multicall`:

```solidity
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
```

### Protocol Operations

| Function                  | Signature                                                                          |
| ------------------------- | ---------------------------------------------------------------------------------- |
| `addCollateral`           | `(address token, uint256 amount)`                                                  |
| `addCollateralWithPermit` | `(address token, uint256 amount, uint256 deadline, uint8 v, bytes32 r, bytes32 s)` |
| `withdrawCollateral`      | `(address token, uint256 amount, address to)`                                      |
| `increaseDebt`            | `(uint256 amount)`                                                                 |
| `decreaseDebt`            | `(uint256 amount)`                                                                 |
| `updateQuota`             | `(address token, int96 quotaChange, uint96 minQuota)`                              |

### Safety Operations

| Function                | Signature                                   |
| ----------------------- | ------------------------------------------- |
| `onDemandPriceUpdate`   | `(address token, bool reserve, bytes data)` |
| `storeExpectedBalances` | `(BalanceDelta[] deltas)`                   |
| `compareBalances`       | `()`                                        |
| `setFullCheckParams`    | `(uint256[] hints, uint16 minHF)`           |
| `setBotPermissions`     | `(address bot, uint192 permissions)`        |

## Encoding Multicalls

Use `abi.encodeCall` for type-safe encoding:

```solidity
MultiCall[] memory calls = new MultiCall[](3);

// Add collateral
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.addCollateral,
        (usdc, 10_000 * 10**6)
    )
});

// Borrow
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.increaseDebt,
        (40_000 * 10**6)
    )
});

// Set quota
calls[2] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (weth, 50_000 * 10**6, 50_000 * 10**6)
    )
});
```

## Adapter Calls

External protocol calls go through adapters. Get the adapter address from Credit Manager:

```solidity
// Get adapter for Uniswap V3
address uniswapV3Adapter = ICreditManagerV3(creditManager).contractToAdapter(UNISWAP_V3_ROUTER);
require(uniswapV3Adapter != address(0), "Adapter not found");

// Encode adapter call
calls[4] = MultiCall({
    target: uniswapV3Adapter,
    callData: abi.encodeCall(
        IUniswapV3Adapter.exactInputSingle,
        (ISwapRouter.ExactInputSingleParams({
            tokenIn: usdc,
            tokenOut: weth,
            fee: 500,
            recipient: address(0), // Adapter overrides to credit account
            deadline: block.timestamp + 3600,
            amountIn: 50_000 * 10**6,
            amountOutMinimum: 0, // Using Gearbox slippage check instead
            sqrtPriceLimitX96: 0
        }))
    )
});
```

## Complete Multicall Example

8-call strategy: price update, collateral, borrow, slippage setup, swap, deposit, slippage check, quota:

```solidity
address accountOwner;
address creditManager;
address creditFacade;
address usdc;
address weth;
address yvWETH;
address uniswapV3Router;
bytes memory yvWETH_priceData;

// Assume exchange rate: 2000 USDC/yvWETH

MultiCall[] memory calls = new MultiCall[](8);

// 1. On-demand price update (must be first)
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.onDemandPriceUpdate,
        (yvWETH, false, yvWETH_priceData)
    )
});

// 2. Add collateral
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.addCollateral,
        (usdc, 10_000 * 10**6)
    )
});

// 3. Borrow
calls[2] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.increaseDebt,
        (40_000 * 10**6)
    )
});

// 4. Store expected balances for slippage check
// Min output: (50000 / 2000) * 0.995 = 24.875 yvWETH
BalanceDelta[] memory deltas = new BalanceDelta[](1);
deltas[0] = BalanceDelta({
    token: yvWETH,
    amount: (25 * 10**18) * 995 / 1000
});

calls[3] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.storeExpectedBalances,
        (deltas)
    )
});

// 5. Swap via Uniswap
address uniswapV3Adapter = ICreditManagerV3(creditManager).contractToAdapter(uniswapV3Router);

ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
    tokenIn: usdc,
    tokenOut: weth,
    fee: 500,
    recipient: address(0), // Adapter overrides
    deadline: block.timestamp + 3600,
    amountIn: 50_000 * 10**6,
    amountOutMinimum: 0, // Using Gearbox slippage check
    sqrtPriceLimitX96: 0
});

calls[4] = MultiCall({
    target: uniswapV3Adapter,
    callData: abi.encodeCall(IUniswapV3Adapter.exactInputSingle, (params))
});

// 6. Deposit to Yearn using diff pattern
address yvWETHAdapter = ICreditManagerV3(creditManager).contractToAdapter(yvWETH);

calls[5] = MultiCall({
    target: yvWETHAdapter,
    callData: abi.encodeCall(IYearnV2Adapter.depositDiff, (1)) // Leave 1 wei
});

// 7. Compare balances (slippage check)
calls[6] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(ICreditFacadeV3Multicall.compareBalances, ())
});

// 8. Set quota for yvWETH
calls[7] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.updateQuota,
        (yvWETH, 50_000 * 10**6, 50_000 * 10**6)
    )
});

// Approve collateral to Credit Manager (not Facade!)
IERC20(usdc).approve(creditManager, 10_000 * 10**6);

// Execute
ICreditFacadeV3(creditFacade).openCreditAccount(accountOwner, calls, 0);
```

## The "Diff" Pattern

Adapters implement `*_diff` functions for handling unknown amounts:

- **Standard function**: Requires exact `amountIn`
- **Diff function**: Calculates `amountIn = currentBalance - leftoverAmount`

This is essential when the exact output of a previous operation is unknown.

```solidity
// After a swap, deposit all WETH except 1 wei to Yearn
calls[5] = MultiCall({
    target: yvWETHAdapter,
    callData: abi.encodeCall(IYearnV2Adapter.depositDiff, (1))
});
```

## Best Practices

1. **Price updates first**: Always put `onDemandPriceUpdates` at the start if using pull-based oracles
2. **Slippage protection**: Always use `storeExpectedBalances` before swaps and `compareBalances` after
3. **Approve to Credit Manager**: Token approvals go to Credit Manager, not Credit Facade
4. **Gas optimization**: Use `setFullCheckParams` with hints for accounts with many tokens
5. **Dust management**: Use `type(uint256).max` in `withdrawCollateral` to empty balances

## Next Steps

- [Multicall Operations](multicalls/README.md) - Individual operation guides
- [Use Cases](use-cases/README.md) - Adapter development and protocol integration
- [Pool Operations](pool-operations.md) - Direct pool interaction

For architectural background, see [Multicall System](../concepts/multicall-system.md).
