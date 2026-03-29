# Setting Bot Permissions

Grant or revoke bot access to manage a Credit Account.

## Why

Use `setBotPermissions` when:

- **Automation** - Let a keeper/bot rebalance or maintain positions
- **Scoped delegation** - Allow only specific actions instead of full control
- **Security hardening** - Revoke permissions quickly if a bot is compromised

## What

`setBotPermissions` configures a bitmask of actions a `bot` can perform on your account through `botMulticall`.

| Parameter     | Type    | Description                |
| ------------- | ------- | -------------------------- |
| `bot`         | address | Bot address to authorize   |
| `permissions` | uint192 | Bitmask of allowed actions |

Common permission bits:

- `ADD_COLLATERAL_PERMISSION = 1 << 0`
- `INCREASE_DEBT_PERMISSION = 1 << 1`
- `DECREASE_DEBT_PERMISSION = 1 << 2`
- `WITHDRAW_COLLATERAL_PERMISSION = 1 << 5`
- `UPDATE_QUOTA_PERMISSION = 1 << 6`
- `SET_BOT_PERMISSIONS_PERMISSION = 1 << 8`
- `EXTERNAL_CALLS_PERMISSION = 1 << 16`

## How

### Grant Limited Bot Permissions

```solidity
import {ICreditFacadeV3Multicall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3Multicall.sol";
import {MultiCall} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";

address creditFacade;
address creditAccount;
address bot;

uint192 constant ADD_COLLATERAL_PERMISSION = 1 << 0;
uint192 constant UPDATE_QUOTA_PERMISSION = 1 << 6;
uint192 constant EXTERNAL_CALLS_PERMISSION = 1 << 16;

uint192 botPermissions =
    ADD_COLLATERAL_PERMISSION
    | UPDATE_QUOTA_PERMISSION
    | EXTERNAL_CALLS_PERMISSION;

MultiCall[] memory calls = new MultiCall[](1);
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setBotPermissions,
        (bot, botPermissions)
    )
});

ICreditFacadeV3(creditFacade).multicall(creditAccount, calls);
```

### Revoke All Bot Permissions

```solidity
// Set permissions mask to zero to revoke all access for the bot
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setBotPermissions,
        (bot, uint192(0))
    )
});
```

### Allow Bot to Manage Other Bots

```solidity
uint192 constant SET_BOT_PERMISSIONS_PERMISSION = 1 << 8;

// Use with care: this lets the bot update permissions for other bots
calls[0] = MultiCall({
    target: creditFacade,
    callData: abi.encodeCall(
        ICreditFacadeV3Multicall.setBotPermissions,
        (bot, SET_BOT_PERMISSIONS_PERMISSION)
    )
});
```

## Gotchas

### Invalid Bits Revert

Only known permission bits are allowed. Unknown bits in `permissions` revert.

### Bot Type Must Match Required Permissions

The facade validates that the selected permission mask is compatible with the bot's expected role.

### Principle of Least Privilege

Start with minimal permissions and add only what automation needs. Avoid granting debt or withdrawal permissions unless required.

### Revocation Is Immediate

Setting permissions to `0` removes bot access right away for subsequent bot calls.

## See Also

- [Making External Calls](./making-external-calls.md) - External actions bots may execute
- [Debt Management](./debt-management.md) - Debt permissions and constraints
- [Withdrawing Collateral](./withdrawing-collateral.md) - Withdrawal permissions and risk
