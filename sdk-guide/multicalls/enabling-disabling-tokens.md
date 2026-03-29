# Enabling and Disabling Tokens (Deprecated)

`enableToken` and `disableToken` are no longer available on Gearbox V3.1 `ICreditFacadeV3Multicall`.

## Use these instead

- **Quoted collateral tokens:** enable or disable via quota using [Updating Quotas](./updating-quotas.md) and `updateQuota` (non-zero quota enables collateral; zero / `type(int96).min` disables).
- **Bots:** delegate allowed actions with [Setting Bot Permissions](./set-bot-permissions.md).

## See also

- [Multicall Operations](./README.md) — current operation list
- [Solidity reference](../../solidity-guide/multicalls/enabling-disabling-tokens.md) — same deprecation note for on-chain docs
