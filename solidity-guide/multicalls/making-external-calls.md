# Making External Calls

Interact with DeFi protocols through adapters.

> For SDK implementation, see [Making External Calls](../../sdk-guide/multicalls/making-external-calls.md).

## Why

External calls enable Credit Accounts to:

- **Swap tokens** - Trade via Uniswap, Curve, Balancer
- **Provide liquidity** - Add to LP pools
- **Stake** - Deposit into yield protocols (Yearn, Convex, Aura)
- **Leverage farm** - Build complex DeFi strategies

All external interactions go through adapters - specialized contracts that translate Credit Account calls to protocol-specific formats.

## What

Adapters wrap external protocol contracts. On execution:

1. Credit Account calls adapter (not the protocol directly)
2. Adapter translates the call and executes on the target protocol
3. Adapter routes output tokens back to Credit Account
4. Token states are updated automatically

**Important:** Never call external protocols directly from a Credit Account. Only use registered adapters.

## How

### Finding Adapters

Get the adapter address for a target protocol:

```solidity
address uniswapRouter = 0x...; // Target protocol contract
address uniswapAdapter = ICreditManagerV3(creditManager).contractToAdapter(uniswapRouter);
require(uniswapAdapter != address(0), "Adapter not found");
```

### Basic Swap via Uniswap

```solidity
import {ICreditFacadeV3} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";
import {ISwapRouter} from "@uniswap/v3-periphery/contracts/interfaces/ISwapRouter.sol";
import {IUniswapV3Adapter} from "@gearbox-protocol/integrations-v3/contracts/interfaces/uniswap/IUniswapV3Adapter.sol";

address creditFacade;
address creditAccount;
address creditManager;
address usdc;
address weth;

// Get the Uniswap V3 adapter
address uniswapRouter = 0xE592427A0AEce92De3Edee1F18E0157C05861564;
address uniswapAdapter = ICreditManagerV3(creditManager).contractToAdapter(uniswapRouter);

MultiCall[] memory calls = new MultiCall[](1);

// Encode the swap params
ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
    tokenIn: usdc,
    tokenOut: weth,
    fee: 500,                    // 0.05% pool
    recipient: address(0),       // Adapter overrides to credit account
    deadline: block.timestamp + 3600,
    amountIn: 50_000 * 10**6,
    amountOutMinimum: 0,         // Use Gearbox slippage check instead
    sqrtPriceLimitX96: 0
});

calls[0] = MultiCall({
    target: uniswapAdapter,
    callData: abi.encodeCall(IUniswapV3Adapter.exactInputSingle, (params))
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

### Using the Diff Pattern

When you don't know the exact input amount (e.g., after a previous swap), use diff functions:

```solidity
// After swapping to WETH, deposit all but 1 wei to Yearn
address yearnVault = 0x...; // yvWETH address
address yearnAdapter = ICreditManagerV3(creditManager).contractToAdapter(yearnVault);

calls[1] = MultiCall({
    target: yearnAdapter,
    callData: abi.encodeCall(
        IYearnV2Adapter.depositDiff,
        (1)  // Leave 1 wei of WETH, deposit the rest
    )
});
```

### Multi-Protocol Strategy

Swap on Uniswap, then stake in Convex:

```solidity
MultiCall[] memory calls = new MultiCall[](3);

// 1. Swap USDC -> WETH on Uniswap
calls[0] = MultiCall({
    target: uniswapAdapter,
    callData: abi.encodeCall(IUniswapV3Adapter.exactInputSingle, (swapParams))
});

// 2. Swap WETH -> stETH on Curve
calls[1] = MultiCall({
    target: curveAdapter,
    callData: abi.encodeCall(
        ICurveV1Adapter.exchange,
        (0, 1, wethAmount, minStethOut)
    )
});

// 3. Deposit stETH to Convex (using diff)
calls[2] = MultiCall({
    target: convexBoosterAdapter,
    callData: abi.encodeCall(
        IConvexV1BoosterAdapter.depositDiff,
        (convexPoolId, 1, true)  // Leave 1 wei, stake = true
    )
});
```

### Balancer Multi-Hop Swap

```solidity
import {IBalancerV2VaultAdapter, SingleSwap, SwapKind} from "...";

SingleSwap memory singleSwap = SingleSwap({
    poolId: balancerPoolId,
    kind: SwapKind.GIVEN_IN,
    assetIn: IAsset(usdc),
    assetOut: IAsset(weth),
    amount: 50_000 * 10**6,
    userData: ""
});

calls[0] = MultiCall({
    target: balancerAdapter,
    callData: abi.encodeCall(
        IBalancerV2VaultAdapter.swap,
        (singleSwap, fundManagement, 0, block.timestamp + 3600)
    )
});
```

## Gotchas

### Recipient is Always Credit Account

Adapters override the recipient parameter:

```solidity
// Even if you specify a different recipient...
params.recipient = someOtherAddress;

// ...the adapter routes output to the Credit Account
// This is a security feature - you can't extract funds via adapters
```

### Approvals are Handled Automatically

Adapters manage token approvals internally. You don't need to approve tokens to the adapter or target protocol.

### Only Allowed Adapters Work

The Credit Manager only accepts calls to registered adapters:

```solidity
// This reverts if the adapter isn't registered
ICreditFacadeV3(creditFacade).multicall(account, [
    MultiCall({target: unregisteredAdapter, callData: ...})
]);
```

### Check Adapter Type

Different adapter versions have different interfaces:

```solidity
bytes32 adapterType = IAdapter(adapter).contractType();
// e.g., "ADAPTER::UNISWAP_V3_ROUTER"
//       "ADAPTER::CURVE_V1_STABLE_NG"
//       "ADAPTER::CVX_V1_BOOSTER"
```

### Diff Functions Require Balance

Diff functions calculate: `amountIn = currentBalance - leftoverAmount`

If the balance is less than `leftoverAmount`, the operation will fail:

```solidity
// If WETH balance is 0.5 ETH and you call:
depositDiff(1 ether);  // REVERTS - not enough balance

// Correct: leave less than current balance
depositDiff(1);  // Deposits 0.5 ETH - 1 wei
```

### External Calls Set Permission Flag

The first adapter call sets `EXTERNAL_CONTRACT_WAS_CALLED_FLAG`:

```solidity
// This flag affects:
// - Additional validation in collateral checks
// - Gas estimation for health factor calculation
```

### Token Enabling

Adapters automatically enable output tokens. After a swap:

- Output token mask is set on the Credit Account
- Token counts toward collateral (if it has quota or is underlying)

## See Also

- [Controlling Slippage](./controlling-slippage.md) - Protect swap outputs
- [Updating Quotas](./updating-quotas.md) - Ensure received quoted tokens are collateralized
