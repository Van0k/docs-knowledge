# Liquidation Bots

Build bots that monitor credit accounts and execute profitable liquidations.

## Overview

Liquidation bots need to:

1. Find accounts with low health factors
2. Filter for liquidatable accounts
3. Compute optimal liquidation paths
4. Execute liquidations profitably

This guide covers each step with verified SDK patterns.

---

## Understanding Liquidation

**WHY:** Know what you're building before writing code.

### When Accounts Become Liquidatable

An account becomes liquidatable when its health factor drops below 1.0:

```
Health Factor = Total Weighted Collateral Value / Total Debt

Where:
- Weighted Value = Sum of (Token Balance * Price * Liquidation Threshold)
- Total Debt = Principal + Accrued Interest + Quota Fees
```

Health factor is scaled by 10000, so `healthFactor < 10000` means liquidatable.

### The Liquidation Process

1. **Liquidator calls** `creditFacade.liquidateCreditAccount()`
2. **Protocol converts** collateral to underlying token
3. **Debt is repaid** from converted collateral
4. **Liquidator receives** premium (configured per Credit Manager)
5. **Remaining funds** go to account owner (if any)

The liquidator provides the multicall that handles collateral conversion. This is where profit comes from - efficient routing means better conversion rates.

---

## Finding Liquidatable Accounts

**WHY:** Efficiently scan all accounts to find opportunities.

### Using CreditAccountCompressor

The `CreditAccountCompressor` has built-in health factor filtering:

```typescript
import {
  creditAccountCompressorAbi,
  AP_CREDIT_ACCOUNT_COMPRESSOR,
  VERSION_RANGE_310,
} from "@gearbox-protocol/sdk";

const [accountCompressor] = sdk.addressProvider.mustGetLatest(
  AP_CREDIT_ACCOUNT_COMPRESSOR,
  VERSION_RANGE_310,
);

// Find accounts with HF < 1.0 (10000 in basis points)
const [accounts, total] = await client.readContract({
  address: accountCompressor,
  abi: creditAccountCompressorAbi,
  functionName: "getCreditAccounts",
  args: [
    creditManagerAddress,
    {
      owner: "0x0000000000000000000000000000000000000000", // Any owner
      minHealthFactor: 0n,
      maxHealthFactor: 10000n, // HF < 1.0
      includeZeroDebt: false,
      reverting: false,
    },
    0n, // offset
  ],
});

console.log(`Found ${accounts.length} accounts with HF < 1.0`);
```

### Filter by isLiquidatable

The `isLiquidatable` field accounts for additional protocol checks:

```typescript
const liquidatable = accounts.filter((a) => a.isLiquidatable);
console.log(`${liquidatable.length} are actually liquidatable`);

for (const account of liquidatable) {
  console.log(`Account: ${account.addr}`);
  console.log(`  Health Factor: ${Number(account.healthFactor) / 10000}`);
  console.log(`  Debt: ${account.debt}`);
  console.log(`  Collaterals:`);

  for (const token of account.tokens) {
    if (token.balance > 0n) {
      console.log(`    ${token.symbol}: ${token.balance}`);
    }
  }
}
```

### Pagination for Large Result Sets

The compressor returns paginated results. Iterate through all pages:

```typescript
async function getAllLiquidatableAccounts(
  creditManager: `0x${string}`,
): Promise<CreditAccountData[]> {
  const filter = {
    owner: "0x0000000000000000000000000000000000000000" as const,
    minHealthFactor: 0n,
    maxHealthFactor: 10000n,
    includeZeroDebt: false,
    reverting: false,
  };

  let offset = 0n;
  let allAccounts: CreditAccountData[] = [];

  while (true) {
    const [accounts, total] = await client.readContract({
      address: accountCompressor,
      abi: creditAccountCompressorAbi,
      functionName: "getCreditAccounts",
      args: [creditManager, filter, offset],
    });

    const liquidatable = accounts.filter((a) => a.isLiquidatable);
    allAccounts.push(...liquidatable);

    offset += BigInt(accounts.length);
    if (offset >= total) break;
  }

  return allAccounts;
}
```

---

## Account Analysis

**WHY:** Understand an account's composition before liquidating.

### Collateral Breakdown

```typescript
interface CollateralPosition {
  token: string;
  symbol: string;
  balance: bigint;
  valueInUnderlying: bigint;
  liquidationThreshold: number;
}

function analyzeCollateral(account: CreditAccountData): CollateralPosition[] {
  return account.tokens
    .filter((t) => t.balance > 0n)
    .map((t) => ({
      token: t.token,
      symbol: t.symbol,
      balance: t.balance,
      valueInUnderlying: t.balanceInUnderlying,
      liquidationThreshold: Number(t.lt) / 100,
    }))
    .sort((a, b) => Number(b.valueInUnderlying - a.valueInUnderlying));
}

const positions = analyzeCollateral(account);
console.log("Collateral by value:");
for (const pos of positions) {
  console.log(
    `  ${pos.symbol}: ${pos.valueInUnderlying} (LT: ${pos.liquidationThreshold}%)`,
  );
}
```

### Estimating Profit

```typescript
interface LiquidationEstimate {
  totalCollateralValue: bigint;
  debt: bigint;
  liquidationPremium: bigint;
  estimatedProfit: bigint;
}

function estimateLiquidation(
  account: CreditAccountData,
  premiumBps: number, // e.g., 400 = 4%
): LiquidationEstimate {
  const totalValue = account.tokens.reduce(
    (sum, t) => sum + t.balanceInUnderlying,
    0n,
  );

  const premium = (totalValue * BigInt(premiumBps)) / 10000n;

  // Simplified: assumes perfect conversion
  const estimatedProfit = totalValue - account.debt;

  return {
    totalCollateralValue: totalValue,
    debt: account.debt,
    liquidationPremium: premium,
    estimatedProfit: estimatedProfit > 0n ? estimatedProfit : 0n,
  };
}
```

---

## Building the Liquidation Multicall

**WHY:** The multicall handles collateral conversion and determines profit.

### Basic Structure

A liquidation multicall typically:

1. Updates stale price feeds (if needed)
2. Swaps collateral tokens to underlying
3. Repays debt (handled by protocol)

```typescript
import { encodeFunctionData } from "viem";
import { iCreditFacadeV300MulticallAbi } from "@gearbox-protocol/sdk";

// Build liquidation multicall
const calls: Array<{ target: `0x${string}`; callData: `0x${string}` }> = [];

// 1. Update any stale price feeds first (must be the first multicall step)
if (stalePriceFeeds.length > 0) {
  calls.push({
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "onDemandPriceUpdates",
      args: [
        stalePriceFeeds.map((feed) => ({
          priceFeed: feed.priceFeed,
          data: feed.data,
        })),
      ],
    }),
  });
}

// 2. Swap collateral to underlying via adapters
for (const collateral of collateralToSwap) {
  const adapter = await creditManager.read.contractToAdapter([
    collateral.protocol,
  ]);

  calls.push({
    target: adapter,
    callData: encodeFunctionData({
      abi: adapterAbi,
      functionName: "swap",
      args: [collateral.swapParams],
    }),
  });
}
```

### Using Slippage Protection

Always protect against sandwich attacks:

```typescript
// Store expected minimum output
calls.push({
  target: creditFacadeAddress,
  callData: encodeFunctionData({
    abi: iCreditFacadeV300MulticallAbi,
    functionName: "storeExpectedBalances",
    args: [[{ token: underlyingToken, amount: minExpectedOutput }]],
  }),
});

// Perform swap
calls.push({
  target: adapter,
  callData: encodeFunctionData({
    abi: adapterAbi,
    functionName: "swap",
    args: [swapParams],
  }),
});

// Verify slippage
calls.push({
  target: creditFacadeAddress,
  callData: encodeFunctionData({
    abi: iCreditFacadeV300MulticallAbi,
    functionName: "compareBalances",
    args: [],
  }),
});
```

See [Controlling Slippage](../multicalls/controlling-slippage.md) for details.

---

## Executing Liquidation

**WHY:** Actually perform the liquidation and capture profit.

### The liquidateCreditAccount Call

```typescript
// Get credit facade for the account's credit manager
const market = sdk.marketRegister.findByCreditManager(account.creditManager);

// Execute liquidation
const hash = await walletClient.writeContract({
  address: market.creditFacade.address,
  abi: creditFacadeAbi,
  functionName: "liquidateCreditAccount",
  args: [
    account.addr, // Credit account to liquidate
    receiverAddress, // Where to send remaining funds
    calls, // Liquidation multicall
  ],
});

console.log(`Liquidation submitted: ${hash}`);

// Wait for confirmation
const receipt = await client.waitForTransactionReceipt({ hash });
console.log(
  `Liquidation ${receipt.status === "success" ? "succeeded" : "failed"}`,
);
```

### Handling Partial Liquidation

In some configurations, partial liquidation is possible. Check the Credit Manager configuration:

```typescript
// Full liquidation only if account is deeply underwater
// Partial liquidation may be allowed above certain HF threshold
```

---

## Bot Architecture

**WHY:** Production bots need proper design for reliability and competitiveness.

### Monitoring Loop

```typescript
async function monitoringLoop() {
  const POLL_INTERVAL = 3000; // 3 seconds

  while (true) {
    try {
      // Scan all credit managers
      for (const cm of creditManagers) {
        const accounts = await getAllLiquidatableAccounts(cm);

        for (const account of accounts) {
          // Analyze opportunity
          const estimate = estimateLiquidation(account, liquidationPremiumBps);

          if (estimate.estimatedProfit > minProfitThreshold) {
            await attemptLiquidation(account);
          }
        }
      }
    } catch (error) {
      console.error("Monitoring error:", error);
    }

    await sleep(POLL_INTERVAL);
  }
}
```

### Simulation Before Execution

Always simulate before sending transactions:

```typescript
async function attemptLiquidation(account: CreditAccountData) {
  const calls = buildLiquidationMulticall(account);

  // Simulate first
  try {
    await client.simulateContract({
      address: creditFacadeAddress,
      abi: creditFacadeAbi,
      functionName: "liquidateCreditAccount",
      args: [account.addr, receiverAddress, calls],
      account: liquidatorAddress,
    });
  } catch (error) {
    console.log(`Simulation failed for ${account.addr}:`, error);
    return;
  }

  // Simulation passed, execute
  try {
    const hash = await walletClient.writeContract({
      address: creditFacadeAddress,
      abi: creditFacadeAbi,
      functionName: "liquidateCreditAccount",
      args: [account.addr, receiverAddress, calls],
    });

    console.log(`Liquidation tx: ${hash}`);
  } catch (error) {
    console.error(`Execution failed:`, error);
  }
}
```

### Competition Considerations

Liquidation is competitive. Other bots are scanning the same accounts.

**Strategies:**

- **Speed:** Use faster RPC endpoints, optimize code paths
- **Gas:** Pay higher gas for priority (use `maxPriorityFeePerGas`)
- **Efficiency:** Better swap routing means higher profit, can afford more gas
- **Flashbots:** Use MEV-protected submission to avoid frontrunning

```typescript
// Higher priority fee for competitive liquidations
const hash = await walletClient.writeContract({
  address: creditFacadeAddress,
  abi: creditFacadeAbi,
  functionName: "liquidateCreditAccount",
  args: [account.addr, receiverAddress, calls],
  maxPriorityFeePerGas: parseGwei("3"), // Higher tip
});
```

---

## Complete Example: Simple Liquidation Bot

```typescript
import { createPublicClient, createWalletClient, http, parseGwei } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";
import {
  GearboxSDK,
  creditAccountCompressorAbi,
  AP_CREDIT_ACCOUNT_COMPRESSOR,
  VERSION_RANGE_310,
} from "@gearbox-protocol/sdk";

const MIN_PROFIT_USD = 100n * 10n ** 6n; // $100 minimum profit

async function runLiquidationBot(creditManagerAddress: `0x${string}`) {
  const client = createPublicClient({
    chain: mainnet,
    transport: http(),
  });

  const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
  const walletClient = createWalletClient({
    account,
    chain: mainnet,
    transport: http(),
  });

  const sdk = await GearboxSDK.attach({
    client,
    marketConfigurators: [],
  });

  const [accountCompressor] = sdk.addressProvider.mustGetLatest(
    AP_CREDIT_ACCOUNT_COMPRESSOR,
    VERSION_RANGE_310,
  );

  const market = sdk.marketRegister.findByCreditManager(creditManagerAddress);

  console.log(`Monitoring ${market.creditManagers[0].address}`);
  console.log(`Liquidator: ${account.address}`);

  while (true) {
    try {
      // Find liquidatable accounts
      const [accounts] = await client.readContract({
        address: accountCompressor,
        abi: creditAccountCompressorAbi,
        functionName: "getCreditAccounts",
        args: [
          creditManagerAddress,
          {
            owner: "0x0000000000000000000000000000000000000000",
            minHealthFactor: 0n,
            maxHealthFactor: 10000n,
            includeZeroDebt: false,
            reverting: false,
          },
          0n,
        ],
      });

      const liquidatable = accounts.filter((a) => a.isLiquidatable);

      if (liquidatable.length > 0) {
        console.log(`Found ${liquidatable.length} liquidatable accounts`);

        for (const target of liquidatable) {
          const totalValue = target.tokens.reduce(
            (sum, t) => sum + t.balanceInUnderlying,
            0n,
          );

          const estimatedProfit = totalValue - target.debt;

          if (estimatedProfit > MIN_PROFIT_USD) {
            console.log(`Profitable opportunity: ${target.addr}`);
            console.log(`  Debt: ${target.debt}`);
            console.log(`  Value: ${totalValue}`);
            console.log(`  Est. Profit: ${estimatedProfit}`);

            // Build and execute liquidation
            // (simplified - real bot would compute optimal swaps)
            const calls = buildLiquidationCalls(target, market);

            try {
              // Simulate
              await client.simulateContract({
                address: market.creditFacade.address,
                abi: creditFacadeAbi,
                functionName: "liquidateCreditAccount",
                args: [target.addr, account.address, calls],
                account: account.address,
              });

              // Execute
              const hash = await walletClient.writeContract({
                address: market.creditFacade.address,
                abi: creditFacadeAbi,
                functionName: "liquidateCreditAccount",
                args: [target.addr, account.address, calls],
                maxPriorityFeePerGas: parseGwei("2"),
              });

              console.log(`Liquidation submitted: ${hash}`);
            } catch (error) {
              console.log(`Failed to liquidate ${target.addr}:`, error);
            }
          }
        }
      }
    } catch (error) {
      console.error("Loop error:", error);
    }

    // Poll every 3 seconds
    await new Promise((resolve) => setTimeout(resolve, 3000));
  }
}

function buildLiquidationCalls(
  account: CreditAccountData,
  market: MarketData,
): Array<{ target: `0x${string}`; callData: `0x${string}` }> {
  const calls: Array<{ target: `0x${string}`; callData: `0x${string}` }> = [];
  const creditFacade = market.creditFacade.address;
  const underlying = market.pool.underlying.address;

  // 1. Swap each non-underlying collateral token to underlying via adapter
  for (const token of account.tokens) {
    if (token.balance <= 1n) continue; // skip dust
    if (token.token === underlying) continue; // skip underlying itself

    // Use Uniswap V3 adapter for swaps (simplified: hardcoded router)
    const uniswapAdapter = market.adapters?.["UNISWAP_V3_ROUTER"];
    if (!uniswapAdapter) continue;

    // exactAllInputSingle swaps entire balance minus 1 wei
    calls.push({
      target: uniswapAdapter,
      callData: encodeFunctionData({
        abi: uniswapV3AdapterAbi,
        functionName: "exactAllInputSingle",
        args: [
          {
            tokenIn: token.token,
            tokenOut: underlying,
            fee: 3000, // 0.3% pool (use 500 for stablecoin pairs)
            deadline: BigInt(Math.floor(Date.now() / 1000) + 3600),
            rateMinRAY: 0n, // No slippage protection (simplified)
            sqrtPriceLimitX96: 0n,
          },
        ],
      }),
    });
  }

  return calls;
}
```

---

## Gotchas

### Price Updates Must Come First

If any price feeds are stale, update them at the start of your multicall:

```typescript
// WRONG: Swap first, then update prices (will fail)
// CORRECT: Update prices first, then swap
const calls = [...priceUpdateCalls, ...swapCalls];
```

See [Updating Price Feeds](../multicalls/updating-price-feeds.md).

### Account State Can Change

Between scanning and executing, another bot may liquidate the account:

```typescript
try {
  await walletClient.writeContract({ ... });
} catch (error) {
  if (error.message.includes('account not liquidatable')) {
    console.log('Account already liquidated by another bot');
  }
}
```

### Gas Estimation

Liquidation gas costs vary based on:

- Number of collateral tokens
- Complexity of swaps
- Price feed updates needed

Always estimate gas before calculating profitability.

---

## Next Steps

- [Making External Calls](../multicalls/making-external-calls.md) - Swap patterns via adapters
- [Controlling Slippage](../multicalls/controlling-slippage.md) - Protect against MEV
- [Updating Price Feeds](../multicalls/updating-price-feeds.md) - Required for stale oracles
- [Compressors Reference](../../reference/compressors.md) - Complete filter options
