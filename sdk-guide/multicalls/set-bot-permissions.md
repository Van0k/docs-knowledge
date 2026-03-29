# Setting Bot Permissions

Grant or revoke bot access to manage a Credit Account via `botMulticall`.

> For Solidity implementation, see [Setting Bot Permissions](../../solidity-guide/multicalls/set-bot-permissions.md).

## Why

Use `setBotPermissions` when:

- **Automation** - Let a keeper rebalance or maintain a position
- **Scoped delegation** - Allow only specific actions instead of full account control
- **Incident response** - Revoke a compromised bot in one multicall step

## What

`setBotPermissions(bot, permissions)` stores a `uint192` bitmask of actions the bot may perform on your behalf (subject to Credit Facade checks).

Common permission bits (from core contracts):

- `ADD_COLLATERAL_PERMISSION = 1 << 0`
- `INCREASE_DEBT_PERMISSION = 1 << 1`
- `DECREASE_DEBT_PERMISSION = 1 << 2`
- `WITHDRAW_COLLATERAL_PERMISSION = 1 << 5`
- `UPDATE_QUOTA_PERMISSION = 1 << 6`
- `SET_BOT_PERMISSIONS_PERMISSION = 1 << 8`
- `EXTERNAL_CALLS_PERMISSION = 1 << 16`

## How

```typescript
import { encodeFunctionData } from "viem";
import { iCreditFacadeV300MulticallAbi } from "@gearbox-protocol/sdk";

const ADD_COLLATERAL_PERMISSION = 1n << 0n;
const UPDATE_QUOTA_PERMISSION = 1n << 6n;
const EXTERNAL_CALLS_PERMISSION = 1n << 16n;

const botPermissions =
  ADD_COLLATERAL_PERMISSION |
  UPDATE_QUOTA_PERMISSION |
  EXTERNAL_CALLS_PERMISSION;

const calls = [
  {
    target: creditFacadeAddress,
    callData: encodeFunctionData({
      abi: iCreditFacadeV300MulticallAbi,
      functionName: "setBotPermissions",
      args: [botAddress, botPermissions],
    }),
  },
];

await market.creditFacade.write.multicall([creditAccountAddress, calls]);
```

### Revoke all bot permissions

```typescript
callData: encodeFunctionData({
  abi: iCreditFacadeV300MulticallAbi,
  functionName: 'setBotPermissions',
  args: [botAddress, 0n],
}),
```

## Gotchas

### Unknown permission bits revert

The facade rejects masks with bits outside the defined permission set.

### Bot role validation

The chosen mask must be compatible with the bot type configured on the facade.

### Least privilege

Grant only what the bot needs. Debt increase and withdrawals are high-impact permissions.

## See Also

- [Making External Calls](./making-external-calls.md) - What `EXTERNAL_CALLS_PERMISSION` enables
- [Debt Management](./debt-management.md) - Borrow/repay permissions
- [Multicalls Overview](../multicalls.md) - Combining helpers with manual encoding
