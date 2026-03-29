# Updating Price Feeds

Push fresh price data for on-demand oracles (Pyth, Redstone).

> For Solidity implementation, see [Updating Price Feeds](../../solidity-guide/multicalls/updating-price-feeds.md).

## Why

You update price feeds when:

- **Using on-demand oracles** - Pyth and Redstone require fresh payloads with each transaction
- **Multicalls fail** - Stale-price style errors often mean missing updates
- **Withdrawals** - Safe pricing may require both main and reserve feeds to be current

Some tokens use pull-based oracles that do not update automatically. You must push payloads before operations that depend on them.

## What

`onDemandPriceUpdates(PriceUpdate[] updates)` forwards an array of updates to the price layer. Each element is a `PriceUpdate` struct (from `IPriceFeedStore`):

| Field       | Type    | Description                                                    |
| ----------- | ------- | -------------------------------------------------------------- |
| `priceFeed` | address | On-chain price feed contract to update (not the token address) |
| `data`      | bytes   | Oracle-specific payload (e.g. Pyth VAA, Redstone package)      |

**Critical rule:** The `onDemandPriceUpdates` call must be the **first** entry in the multicall. If any other call runs before it, the facade reverts.

You can pass **multiple** `PriceUpdate` structs in a **single** `onDemandPriceUpdates` call (preferred) instead of issuing several facade calls.

## How

```typescript
import { encodeFunctionData } from "viem";
import { iCreditFacadeV300MulticallAbi } from "@gearbox-protocol/sdk";

// Obtain payloads off-chain (example: Pyth Hermes)
const mainFeedData = await pythClient.getPriceUpdateData([mainFeedId]);

const priceUpdates = [
  {
    priceFeed: wethMainPriceFeedAddress,
    data: mainFeedData[0] as `0x${string}`,
  },
];

const calls = [
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "onDemandPriceUpdates",
      args: [priceUpdates],
    }),
  },
  service.prepareAddCollateral(usdcAddress, amount),
  service.prepareIncreaseDebt(debtAmount),
];
```

### Multiple feeds in one call

```typescript
const priceUpdates = [
  { priceFeed: feedAddress1, data: payload1 as `0x${string}` },
  { priceFeed: feedAddress2, data: payload2 as `0x${string}` },
];

const calls = [
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "onDemandPriceUpdates",
      args: [priceUpdates],
    }),
  },
  // ...other operations
];
```

### Main and reserve feeds (e.g. withdrawals / safe pricing)

Safe pricing uses main and reserve feeds where configured. Supply **one `PriceUpdate` per price feed contract** that needs a fresh payload:

```typescript
const priceUpdates = [
  { priceFeed: tokenMainPriceFeed, data: mainPayload as `0x${string}` },
  { priceFeed: tokenReservePriceFeed, data: reservePayload as `0x${string}` },
];
```

Resolve `tokenMainPriceFeed` / `tokenReservePriceFeed` from your market’s price oracle configuration (compressor or config), not from the token address alone.

### Discovering feeds that need updates

```typescript
import { priceFeedCompressorAbi } from "@gearbox-protocol/sdk";

const feedInfo = await client.readContract({
  address: priceFeedCompressorAddress,
  abi: priceFeedCompressorAbi,
  functionName: "getUpdatablePriceFeeds",
  args: [creditManagerAddress],
});

const tokensNeedingUpdate = feedInfo.filter((f) => f.needsUpdate);
```

## Gotchas

### Must be first

```typescript
// WRONG — price updates after another call revert
const calls = [
  service.prepareAddCollateral(token, amount),
  priceUpdatesCall, // reverts
];

// CORRECT
const calls = [priceUpdatesCall, service.prepareAddCollateral(token, amount)];
```

### `priceFeed` is not the token

Encoding must target the **price feed contract** address your market uses for that asset. The oracle SDK returns `bytes` payloads; pairing them with the wrong `priceFeed` address fails validation on-chain.

### Fresh payloads

Generate update data immediately before the transaction. Cached payloads often expire within minutes.

### Not every asset needs pull updates

Chainlink-style feeds that update on-chain independently usually do not need `onDemandPriceUpdates`. Use market metadata / compressor output to see which feeds are on-demand.

## See Also

- [Controlling Slippage](./controlling-slippage.md) - Stale prices and sandwich risk
- [Withdrawing Collateral](./withdrawing-collateral.md) - Safe pricing and feeds
- [Collateral Check Params](./collateral-check-params.md) - Health checks and pricing modes
