# @sudobility/heavymath_contracts

Multi-chain prediction market smart contracts and a TypeScript SDK. Today the EVM
side is fully implemented (Solidity + viem client); Solana and React Native entry
points are published but are still stubs (see [Status](#status)). The prediction
mechanism is percentage-based (0-100), not binary, and settles via a dual-boundary
market split.

## Status

| Area                    | State                                                        |
| ----------------------- | ------------------------------------------------------------ |
| EVM contracts + SDK     | Implemented and deployed (Sepolia, Arbitrum Sepolia)         |
| `./solana` export       | Stub (`export {}`) — no Solana programs in the repo yet      |
| `./react-native` export | Build target only — `src/react-native/` is not yet populated |

## Prerequisites

- [Bun](https://bun.sh) (the required package manager)
- Node.js >= 20.18.0

The package is published to GitHub Packages. To install it, your `.npmrc` must map
the scope to that registry:

```
@sudobility:registry=https://npm.pkg.github.com
```

## Installation

```bash
bun add @sudobility/heavymath_contracts
```

## Usage

```typescript
import { PredictionClient } from '@sudobility/heavymath_contracts';
import { EVMPredictionClient } from '@sudobility/heavymath_contracts/evm';

const client = new EVMPredictionClient({
  predictionMarket: '0x...',
  stakeToken: '0x...', // USDC address (optional, resolved on-chain if omitted)
});

// Place a prediction (auto-approves ERC20)
await client.placePrediction(walletContext, marketId, 60, 1_000_000n);

// Create a market (requires a Dealer NFT with matching category permissions)
await client.createMarket(walletContext, {
  tokenId: 1n,
  category: 1n,
  subCategory: 2n,
  deadline: BigInt(Math.floor(Date.now() / 1000) + 86400),
  description: 'Team A wins',
});

// Read market state
const market = await client.getMarket(publicClient, marketId);
```

The SDK uses **viem** for all contract interactions. `ethers` is a dev dependency
used only for TypeChain type generation.

## Smart Contracts

Three UUPS-upgradeable Solidity 0.8.24 contracts (optimizer on, 200 runs):

- **PredictionMarket** — core market lifecycle (create, predict, lock, resolve, claim)
- **DealerNFT** — ERC721 licensing with category/subcategory permissions (public mint)
- **OracleResolver** — oracle registration, data feeds, and Chainlink Functions resolution

`MockUSDC` (6-decimal test ERC20) and a re-export of OpenZeppelin's `ERC1967Proxy`
support testing and deployment.

### How It Works

1. Dealers create markets with a deadline (must own a DealerNFT with valid permissions).
2. Users place predictions at a percentage (0-100) with USDC before the deadline.
3. After the deadline, anyone calls `lockMarket` — the dual-boundary market split is
   calculated, middle-zone predictions are refunded, and boundary stakes are trimmed.
4. The market is resolved (manually by the dealer, by an authorized resolver, or via
   an oracle/Chainlink). Winners on the resolved side share the pool proportionally.
5. Fees: winner fee (default 1%, max 10%) split between dealer and platform
   (default 50/50), both owner-configurable.

See [docs/CALCULATIONS.md](docs/CALCULATIONS.md) and [docs/CHAINLINK.md](docs/CHAINLINK.md)
for details. `calculate.ts` at the repo root is a TypeScript reference implementation of
the split algorithm (for off-chain simulation only; the on-chain Solidity is authoritative).

## Development

```bash
bun install                 # Install dependencies (Bun required)

bun run compile:evm         # Compile Solidity (hardhat compile)
bun run build               # Build everything: compile + unified + react-native TS
bun run test:evm            # Run Hardhat test suite (via tsx)
bun run lint                # ESLint (.ts, .js)
bun run typecheck           # tsc --noEmit (compiles contracts first)
bun run format              # Prettier write
```

## Deployment

Copy `.env.example` to `.env` (or `.env.local`) and set at least `USDC_ADDRESS` and,
for non-local networks, `PRIVATE_KEY`. Then:

```bash
bun run deploy:evm:local       # In-process Hardhat network
bun run deploy:evm:sepolia     # Sepolia testnet
bun run deploy:evm:mainnet     # Ethereum mainnet
```

`scripts/evm/deploy.ts` deploys all three contracts as ERC1967 (UUPS) proxies and
records addresses in `DEPLOYED.json`. After deploying, call
`OracleResolver.setPredictionMarket(...)`. UUPS upgrades are handled by
`scripts/evm/upgrade.ts` (`UPGRADE=PredictionMarket bun run upgrade:evm:sepolia`).
`verify.ts` and `prepare-upgrade.ts` are not yet implemented.

Supported networks (see `hardhat.config.cts`): hardhat, localhost, mainnet, sepolia,
polygon, optimism, arbitrum, arbitrumSepolia, base.

## SDK Entry Points

| Import                                         | Description                        |
| ---------------------------------------------- | ---------------------------------- |
| `@sudobility/heavymath_contracts`              | Unified `PredictionClient`         |
| `@sudobility/heavymath_contracts/evm`          | `EVMPredictionClient` (viem)       |
| `@sudobility/heavymath_contracts/solana`       | Stub (not yet implemented)         |
| `@sudobility/heavymath_contracts/react-native` | React Native build (not populated) |

## Related Packages

- `@sudobility/heavymath_types` — shared type definitions
- `@sudobility/heavymath_indexer_client` — indexer API client
- `@sudobility/heavymath_lib` — business logic hooks
- `heavymath_app` — frontend web application

## License

BUSL-1.1 (see [LICENSE.md](LICENSE.md))
