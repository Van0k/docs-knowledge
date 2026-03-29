# Revoke Allowances (Deprecated)

`revokeAdapterAllowances` is no longer available on Gearbox V3.1 `ICreditFacadeV3Multicall`.

## Notes

- Do not rely on a multicall step to batch-revoke allowances from the Credit Account; use currently supported facade operations and normal ERC-20 allowance patterns where applicable.
- For automation access control, see [Setting Bot Permissions](./set-bot-permissions.md).

## See also

- [Making External Calls](./making-external-calls.md) — adapter interaction patterns
- [Solidity reference](../../solidity-guide/multicalls/revoke-allowances.md) — matching deprecation note
