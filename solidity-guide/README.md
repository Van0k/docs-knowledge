# Solidity Guide

Integrate directly with Gearbox contracts from your Solidity code. This guide covers on-chain integration patterns for smart contract developers.

## Prerequisites

- Solidity 0.8.x experience
- Familiarity with interface-based contract interaction
- Understanding of ERC-20 and common DeFi patterns

## What You'll Learn

| Topic              | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| Credit Accounts    | Contract discovery, CreditFacade, CreditManager interactions |
| Multicall Encoding | Build and execute multicalls in Solidity                     |
| Pool Operations    | Deposit, withdraw, and read pool state                       |

## Guide Structure

1. **[Credit Accounts](credit-accounts.md)** - Contract discovery, ICreditFacadeV3, ICreditManagerV3 interfaces
2. **[Multicalls](multicalls.md)** - MultiCall struct encoding, adapter calls
3. **[Pool Operations](pool-operations.md)** - IPoolV3 deposit/withdraw, ERC-4626 functions

Note: Contract discovery patterns (AddressProvider, ContractsRegister) are covered in the Credit Accounts guide.

## Key Interfaces

```solidity
// Core interfaces you'll use
import {IAddressProviderV3} from "@gearbox-protocol/core-v3/contracts/interfaces/IAddressProviderV3.sol";
import {ICreditFacadeV3} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditFacadeV3.sol";
import {ICreditManagerV3} from "@gearbox-protocol/core-v3/contracts/interfaces/ICreditManagerV3.sol";
import {IPoolV3} from "@gearbox-protocol/core-v3/contracts/interfaces/IPoolV3.sol";
```

## When to Use Solidity Integration

| Use Case                      | Recommended        |
| ----------------------------- | ------------------ |
| On-chain protocol integration | Yes                |
| Building adapters             | Yes                |
| Composable strategies         | Yes                |
| Backend services              | No (use SDK Guide) |
| Frontend applications         | No (use SDK Guide) |

For TypeScript/JavaScript applications, see the [SDK Guide](../sdk-guide/README.md).

## Architecture Understanding

For conceptual background on how Gearbox works:

- [Credit Suite Architecture](../concepts/credit-suite.md) - How Credit Managers, Facades, and Configurators work together
- [Pool Architecture](../concepts/pools.md) - Lending pools and ERC-4626 compliance
- [Multicall System](../concepts/multicall-system.md) - How multicalls execute and validate

---

## Detailed Guides

### Multicall Operations

Complete reference for each multicall operation with Solidity examples:

| Operation                                                        | Description                               |
| ---------------------------------------------------------------- | ----------------------------------------- |
| [Adding Collateral](multicalls/adding-collateral.md)             | Transfer tokens to credit account         |
| [Debt Management](multicalls/debt-management.md)                 | Borrow and repay                          |
| [Updating Quotas](multicalls/updating-quotas.md)                 | Manage collateral quotas                  |
| [Withdrawing Collateral](multicalls/withdrawing-collateral.md)   | Remove tokens from account                |
| [Controlling Slippage](multicalls/controlling-slippage.md)       | Protect against price movement            |
| [Making External Calls](multicalls/making-external-calls.md)     | Interact with DeFi protocols via adapters |
| [Updating Price Feeds](multicalls/updating-price-feeds.md)       | On-demand oracle updates (Pyth, Redstone) |
| [Collateral Check Params](multicalls/collateral-check-params.md) | Optimize health checks with hints         |
| [Setting Bot Permissions](multicalls/set-bot-permissions.md)     | Configure bot access to account actions   |

See [Multicalls Overview](multicalls.md) for the diff pattern and complete encoding examples.

### Use Case Guides

Integration-specific guides for common development scenarios:

| Building             | Guide                                                     | Focus                                                 |
| -------------------- | --------------------------------------------------------- | ----------------------------------------------------- |
| New DeFi Adapter     | [Adapter Development](use-cases/adapter-development.md)   | AbstractAdapter, security patterns, diff functions    |
| Strategy Contract    | [Protocol Integration](use-cases/protocol-integration.md) | Multicall building, access control, automation        |
| Core Extensions      | [Core Extension](use-cases/core-extension.md)             | Extending core contracts, advanced customizations     |
| Liquidation Contract | [Liquidation Bots](use-cases/liquidation-bots.md)         | On-chain liquidation, flash loans, keeper integration |
