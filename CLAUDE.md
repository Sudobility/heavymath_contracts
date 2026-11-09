# CLAUDE.md — heavymath_contracts

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`/`git push`/`gh pr create` **only when the user explicitly asks in
> that turn**. Earlier approval does not carry forward; finishing a task is not
> permission to commit it. Do not create feature branches — stay on the current branch.

## What this is

Multi-chain prediction market smart contracts + TypeScript SDK. Package
`@sudobility/heavymath_contracts`. Percentage-based (0-100) predictions settled by a
**dual-boundary market split** — not binary bets.

- **Stack**: Solidity 0.8.24 (optimizer, 200 runs), Hardhat, viem 2, TypeScript, UUPS proxies
- **Package manager**: Bun (required). Node >= 20.18. License BUSL-1.1.
- **Only EVM is implemented.** Solana and React Native are published stubs (see below).

## Commands (all `bun run <script>`; defined in package.json)

```
compile:evm      hardhat compile
build            full build: compile:evm + build:unified + build:react-native + fix-esm-imports
build:evm        hardhat compile + tsc -p tsconfig.evm.json
test:evm         Hardhat test suite via tsx (node --import tsx/esm hardhat test); alias: test:unit / test:run
test:ci          build:unified then run compiled unified tests
lint / lint:fix  ESLint over .ts/.js
typecheck        tsc --noEmit (pretypecheck runs hardhat compile first)
format           Prettier write; format:check to verify
clean            hardhat clean
deploy:evm:{local,localhost,sepolia,mainnet}   run scripts/evm/deploy.ts on that network
upgrade:evm:{sepolia,mainnet}                  run scripts/evm/upgrade.ts (UUPS)
```

Note: `node_modules` is not checked in — run `bun install` in a fresh checkout before
any compile/test. `npx` in scripts resolves to `bun x`.

## Architecture

Three UUPS-upgradeable contracts in `contracts/` (all `_disableInitializers()` in ctor,
all end with a `__gap`):

- **PredictionMarket.sol** (~1400 lines) — market lifecycle. `Initializable` →
  `Ownable` → `Pausable` → `UUPS` → `ReentrancyGuard` (upgradeable). Fees:
  `winnerFeeBps` (default 100 = 1%, max 1000), `dealerSharePercent` (default 50).
  Status: Active / Cancelled / Resolved / Abandoned / Locked.
- **DealerNFT.sol** (~306 lines) — ERC721 license ("DealerLicense"/"DLICENSE") with
  category/subcategory permissions; public `mint()` pays `mintPrice` in the stake token.
  `WILDCARD = 0xFF` = all categories/subcategories.
- **OracleResolver.sol** (~466 lines) — oracle registration/data feeds + Chainlink
  Functions resolution. Types: Manual / PriceFeed / CustomData.

`MockUSDC.sol` (6-decimal test ERC20, not upgradeable) and `ERC1967Proxy.sol`
(re-export of OZ proxy) support tests/deploy. `contracts/interfaces/` is empty.

**Market flow**: dealer `createMarket` → predictors `placePrediction(pct 0-100, amount)`
→ after deadline anyone `lockMarket` (computes split, refunds middle zone, trims
boundaries → `Locked`) → resolve (`resolveMarket` dealer-only non-oracle /
`resolveMarketByResolver` / `resolveMarketWithOracle` / Chainlink
`requestOracleResolution`+`completeOracleResolution`) → winners `claimWinnings`,
others `claimLockRefund`/`claimRefund`. Unresolved after deadline + 24h grace →
`abandonMarket` (full refunds).

### TypeScript SDK (`src/`)

- `src/evm/index.ts` — `EVMPredictionClient` (full viem implementation). Write methods
  return `TransactionResult { hash, receiptHash? }`. `placePrediction`/`updatePrediction`
  auto-approve ERC20 via `ensureAllowance`. `stakeToken` resolved on-chain if not passed.
- `src/unified/index.ts` — `PredictionClient`, a thin wrapper exposing `.evm`
  (placeholder for future multi-chain routing).
- `src/solana/index.ts` — **stub** (`export {}`). No Solana programs, no `Anchor.toml`,
  no `programs/`/`target/` dirs exist. `build:solana`/`test:solana`/`deploy:solana:*` and
  `compile:solana` reference non-existent Cargo/Anchor infra — do not run them.
- `src/react-native/` — **does not exist yet**; only the `tsconfig.react-native.json`
  build target and the package export are wired up.

NPM exports (package.json): `.` (unified; react-native condition → react-native build),
`./evm`, `./solana`, `./react-native`. Build outputs land in `dist/unified/...` and
`dist/react-native/...`.

`calculate.ts` (repo root) is a **reference-only** TS implementation of the split
algorithm for off-chain simulation — NOT used by the contracts or SDK.

## Scripts (`scripts/evm/`)

- `deploy.ts` — **implemented.** Deploys all 3 contracts as ERC1967/UUPS proxies
  (encodes `initialize` calldata manually), optional `OWNER_MULTISIG` ownership
  transfer, writes/updates `DEPLOYED.json`. Requires `USDC_ADDRESS` (+ `PRIVATE_KEY`
  off-local).
- `upgrade.ts` — **implemented.** `UPGRADE=PredictionMarket[,DealerNFT,...] bun run
upgrade:evm:<net>`; reads proxies from `DEPLOYED.json`, deploys new impls, calls
  `upgradeToAndCall`.
- `verify.ts`, `prepare-upgrade.ts` — **placeholders** ("Phase 8"); print a warning
  and exit. Do not rely on them.
- Operational helpers (run directly via hardhat, no npm alias): `mint-dealer.ts`,
  `set-permissions.ts`, `setup-oracle.ts`, `configure-chainlink.ts`, `check-nft.ts`,
  `verify-setup.ts`, `upgrade-dealer-nft.ts`.

## Conventions & gotchas

- **viem, never ethers, in SDK/scripts.** `ethers` exists only for TypeChain generation.
  `@openzeppelin/hardhat-upgrades` and `hardhat-chai-matchers` are intentionally
  commented out in `hardhat.config.cts` (they require ethers) — UUPS proxies are
  deployed/upgraded manually.
- Hardhat config file is **`hardhat.config.cts`** (CommonJS TS), not `.ts`.
- `OracleResolver.markResolved()` is access-controlled to the PredictionMarket address —
  call `OracleResolver.setPredictionMarket(predictionMarketProxy)` after every deploy.
- Env: `hardhat.config.cts` loads `.env` first, then `.env.local` as a fallback
  (`override: false`). See `.env.example` for the full variable list. RPCs built from
  `ALCHEMY_API_KEY`; Etherscan keys per-chain or `ETHERSCAN_MULTICHAIN_API_KEY`.
- `DEPLOYED.json` holds live addresses (impl + proxy) for sepolia, arbitrumSepolia,
  hardhat. `deploy.ts`/`upgrade.ts` update it.
- Published to **GitHub Packages** (`.npmrc`: `@sudobility:registry=...github.com`).
  CI (`.github/workflows/ci-cd.yml`) uses reusable workflow
  `johnqh/workflows/.../unified-cicd.yml@main` with `publish-target: github`, on
  push/PR to `main`/`develop`.
- TypeChain target is ethers-v6, output `typechain-types/`.

## Test fixture (`test/evm/utils/fixture.ts`)

`deployPredictionFixture()` deploys DealerNFT (free mint), MockUSDC (6 dec),
OracleResolver, PredictionMarket as proxies; mints dealer licenses + permissions;
wires `setPredictionMarket`; funds + approves 100k USDC to 5 wallets. Returns all
contracts + `[owner, dealer1, dealer2, predictor1, predictor2, predictor3]` +
`publicClient`. Helpers: `USDC_DECIMALS=6`, `toUSDC()`, `advanceTime()`.

## Ecosystem (downstream consumers)

`heavymath_contracts` (here) → `heavymath_indexer` (indexes events) →
`heavymath_indexer_client` → `heavymath_lib` → `heavymath_app`. Changing an event/ABI
means updating the indexer (handlers, abis/, migrations) and republishing types; a new
contract function means updating `EVMPredictionClient` here.
