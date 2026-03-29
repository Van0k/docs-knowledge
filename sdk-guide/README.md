# SDK Guide (TypeScript)

The Gearbox SDK provides typed access to protocol state and operations. It wraps contract calls, handles encoding, and provides cached market data through a clean TypeScript interface.

## Prerequisites

- Node.js 18+
- Basic familiarity with [viem](https://viem.sh/)
- Understanding of async/await patterns

## What the SDK Provides

| Component         | Purpose                                                 |
| ----------------- | ------------------------------------------------------- |
| `GearboxSDK`      | Main entry point with `attach()` initialization         |
| `marketRegister`  | Cached market data access                               |
| `addressProvider` | Contract discovery wrapper                              |
| Plugins           | Extended functionality (AccountsPlugin, AdaptersPlugin) |
| Services          | Credit account operations and multicall helpers         |

## Guide Structure

1. **[Setup](setup.md)** - Installation, SDK initialization, basic configuration
2. **[Reading Data](reading-data.md)** - Market queries, account state, pool data
3. **[Credit Accounts](credit-accounts.md)** - Account operations via services
4. **[Multicalls](multicalls.md)** - Building and executing multicalls

## When to Use the SDK

| Use Case              | Recommended                      |
| --------------------- | -------------------------------- |
| Frontend applications | Yes                              |
| Analytics dashboards  | Yes                              |
| Liquidation bots      | Yes (or direct compressor calls) |
| Backend services      | Yes                              |
| On-chain contracts    | No (use Solidity Guide)          |

The SDK handles ABI management, type conversions, and caching internally. For on-chain integrations or gas-optimized bot implementations, consider the [Solidity Guide](../solidity-guide/README.md).

## Architecture Understanding

For conceptual background on how Gearbox works:

- [Credit Suite Architecture](../concepts/credit-suite.md) - How Credit Managers, Facades, and Configurators work together
- [Pool Architecture](../concepts/pools.md) - Lending pools and ERC-4626 compliance
- [Multicall System](../concepts/multicall-system.md) - How multicalls execute and validate

---

## Detailed Guides

### Multicall Operations

Complete reference for each multicall operation with TypeScript examples:

| Operation                                                        | Description                             |
| ---------------------------------------------------------------- | --------------------------------------- |
| [Adding Collateral](multicalls/adding-collateral.md)             | Transfer tokens to credit account       |
| [Debt Management](multicalls/debt-management.md)                 | Borrow and repay                        |
| [Updating Quotas](multicalls/updating-quotas.md)                 | Manage collateral quotas                |
| [Withdrawing Collateral](multicalls/withdrawing-collateral.md)   | Remove tokens from account              |
| [Controlling Slippage](multicalls/controlling-slippage.md)       | Protect against price movement          |
| [Making External Calls](multicalls/making-external-calls.md)     | Interact with DeFi protocols            |
| [Updating Price Feeds](multicalls/updating-price-feeds.md)       | On-demand oracle updates                |
| [Collateral Check Params](multicalls/collateral-check-params.md) | Optimize health checks                  |
| [Setting Bot Permissions](multicalls/set-bot-permissions.md)     | Configure bot access to account actions |

See [Multicalls Overview](multicalls.md) for building and executing multicalls.

### Use Case Guides

Application-specific guides for common development scenarios:

| Building            | Guide                                                       | Focus                           |
| ------------------- | ----------------------------------------------------------- | ------------------------------- |
| Web UI / Dashboard  | [Frontend Applications](use-cases/frontend-applications.md) | Data display, real-time updates |
| Analytics / Indexer | [Backend Services](use-cases/backend-services.md)           | Historical data, event indexing |
| Liquidation Bot     | [Liquidation Bots](use-cases/liquidation-bots.md)           | Monitoring, Router execution    |
