# Collateral Check Params

Optimize gas and set minimum health factor for collateral checks.

> For SDK implementation, see [Collateral Check Params](../../sdk-guide/multicalls/collateral-check-params.md).

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

```solidity
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";

address creditFacade;
address creditAccount;
address creditManager;
address usdc;
address weth;

// Get token masks (each token has a unique bitmask)
uint256 usdcMask = ICreditManagerV3(creditManager).getTokenMaskOrRevert(usdc);
uint256 wethMask = ICreditManagerV3(creditManager).getTokenMaskOrRevert(weth);

// Build hints array
uint256[] memory hints = new uint256[](2);
hints[0] = usdcMask;
hints[1] = wethMask;

MultiCall[] memory calls = new MultiCall[](2);

// Set hints at the start of multicall
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setFullCheckParams,
        (
            hints,   // Check these tokens first
            10000    // minHealthFactor: 1.0 (10000 bps)
        )
    )
});

// Rest of your multicall operations
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.addCollateral,
        (usdc, 100_000 * 10**6)
    )
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

### Setting Higher Min Health Factor

Require account to maintain at least 1.2 HF:

```solidity
uint256 MIN_HF_120 = 12000; // 1.2 in basis points

uint256[] memory hints = new uint256[](0); // No hints

MultiCall[] memory calls = new MultiCall[](2);

calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setFullCheckParams,
        (hints, MIN_HF_120)
    )
});

// Operations that must maintain 1.2 HF...
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.increaseDebt,
        (borrowAmount)
    )
});
```

### Complete Gas-Optimized Strategy

```solidity
address creditManager;
address usdc;
address uniswapAdapter;

// Get mask for primary collateral token
uint256 usdcMask = ICreditManagerV3(creditManager).getTokenMaskOrRevert(usdc);

uint256[] memory hints = new uint256[](1);
hints[0] = usdcMask;

MultiCall[] memory calls = new MultiCall[](4);

// 1. Hints first - USDC will cover most of debt
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setFullCheckParams,
        (hints, 10500)  // Require 1.05 HF minimum
    )
});

// 2. Add collateral
calls[1] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.addCollateral,
        (usdc, 100_000 * 10**6)
    )
});

// 3. Borrow
calls[2] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.increaseDebt,
        (200_000 * 10**6)
    )
});

// 4. Swap some USDC to WETH
calls[3] = MultiCall({
    target: uniswapAdapter,
    callData: abi.encodeCall(
        IUniswapV3Adapter.exactInputSingle,
        (swapParams)
    )
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

## Gotchas

### Masks, Not Addresses

The hints array takes token **masks**, not addresses:

```solidity
// WRONG - passing addresses
uint256[] memory hints = new uint256[](2);
hints[0] = uint256(uint160(usdcAddress));  // Wrong!
hints[1] = uint256(uint160(wethAddress));  // Wrong!

// CORRECT - passing masks
hints[0] = ICreditManagerV3(creditManager).getTokenMaskOrRevert(usdc);
hints[1] = ICreditManagerV3(creditManager).getTokenMaskOrRevert(weth);
```

### Hints Are Optimization, Not Guarantee

The check still validates ALL enabled tokens - hints just change the order. If hints don't cover the debt, it continues with remaining tokens:

```solidity
// If USDC hint doesn't fully cover debt,
// WETH and other tokens are still checked
// Hints just potentially skip some oracle calls
```

### Min Health Factor Must Be >= 10000

You cannot set a health factor below 1.0:

```solidity
// WRONG - less than 10000 reverts
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setFullCheckParams,
        (hints, 9500)  // Reverts!
    )
});

// CORRECT - must be >= 10000
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setFullCheckParams,
        (hints, 10000)  // Exactly 1.0 - OK
    )
});
```

### Hints Don't Help Small Accounts

For accounts with few enabled tokens (< 5), hints add gas overhead without saving much. Only use for accounts with many tokens.

### Order Matters in Hints Array

Tokens are checked in the order you provide:

```solidity
// Check WETH first, then USDC
hints[0] = wethMask;
hints[1] = usdcMask;

// Check USDC first, then WETH
hints[0] = usdcMask;
hints[1] = wethMask;
```

Put your highest-value collateral first for best gas savings.

### Token Mask Calculation

Token masks are powers of 2, assigned sequentially when tokens are added to the Credit Manager:

```solidity
// First token: mask = 1 (2^0)
// Second token: mask = 2 (2^1)
// Third token: mask = 4 (2^2)
// etc.

// Always use getTokenMaskOrRevert to get the correct mask
uint256 mask = ICreditManagerV3(creditManager).getTokenMaskOrRevert(tokenAddress);
```

### Params Reset After Multicall

`setFullCheckParams` only affects the current multicall's final check. Next multicall uses defaults again:

```solidity
// Multicall 1 - with hints
ICreditFacadeV3(creditFacade).multicall(account, callsWithHints);

// Multicall 2 - back to default (no hints)
ICreditFacadeV3(creditFacade).multicall(account, callsWithoutHints);
```

### Combine Hints and Min HF

Use hints for gas optimization AND min HF for risk management:

```solidity
uint256[] memory hints = new uint256[](2);
hints[0] = primaryCollateralMask;   // Gas optimization
hints[1] = secondaryCollateralMask;

calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setFullCheckParams,
        (hints, 11000)  // Risk management: require 1.1 HF
    )
});
```

## See Also

- [Updating Quotas](./updating-quotas.md) - Determines which quoted tokens contribute to collateral checks
- [Updating Price Feeds](./updating-price-feeds.md) - Oracle calls that hints can skip
- [Debt Management](./debt-management.md) - Debt determines what TWV must cover
