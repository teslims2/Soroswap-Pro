# Soroswap Pro

An open-source decentralized exchange built on Soroban — constant-product liquidity pools, LP token minting, on-chain TWAP price oracles, and an automated arbitrage bot.

---

## Table of Contents

1. [Problem It Solves](#problem-it-solves)
2. [Architecture](#architecture)
3. [Project Structure](#project-structure)
4. [Getting Started](#getting-started)
5. [Contributing](#contributing)

---

## Problem It Solves

| Gap in Today's Stellar Ecosystem | What Soroswap Pro Delivers |
|---|---|
| Stellar's native DEX is order-book based — no on-chain AMM liquidity or passive LP yield on Soroban | Constant-product (x·y=k) pools on Soroban with LP token minting, so anyone can provide liquidity and earn fees |
| No on-chain price oracle for Soroban contracts to consume time-weighted prices | TWAP oracle stored on-chain per pool; any contract can call `twap(token_a, token_b, period)` for manipulation-resistant prices |
| Price divergences between Soroban AMMs and the Stellar native DEX go unexploited, leaving pools mispriced | TypeScript arbitrage bot continuously monitors the spread and executes cross-venue trades when spread exceeds 0.5% |

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Browser / UI                   │
│  Next.js 14 App Router                          │
│  ┌──────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ /swap    │  │ /pool (LP)   │  │ PriceChart│ │
│  │ price    │  │ pool share   │  │ TWAP data │ │
│  │ impact   │  │ IL estimate  │  │ recharts  │ │
│  └────┬─────┘  └──────┬───────┘  └─────┬─────┘ │
└───────┼───────────────┼────────────────┼────────┘
        │  Stellar SDK (RPC / simulate)  │
        ▼                                ▼
┌───────────────────────────────────────────────┐
│             Soroban Smart Contracts            │
│                                               │
│  ┌─────────────────────────────────────────┐  │
│  │          soroswap-amm                   │  │
│  │  create_pool  add_liquidity             │  │
│  │  remove_liquidity  swap                 │  │
│  │  get_price  twap  quote                 │  │
│  │  TWAP observations stored on-chain      │  │
│  └───────────────┬─────────────────────────┘  │
│                  │ deploys / calls             │
│  ┌───────────────▼─────────────────────────┐  │
│  │        soroswap-lp-token (per pool)     │  │
│  │  initialize  mint  burn  transfer       │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
        ▲                        ▲
        │ simulate get_price     │ submit swap tx
┌───────┴────────────────────────┴──────────────┐
│           Arbitrage Bot (TypeScript)           │
│  • Polls every 10 s                           │
│  • Compares AMM price vs Stellar Horizon       │
│    order book (/order_book endpoint)          │
│  • Executes if spread > 0.5 %                 │
└───────────────────────────────────────────────┘

Key design decisions
────────────────────
• Fee taken on amount_in before applying x·y=k, stored as fee_bps (e.g. 30 = 0.30 %)
• TWAP: each swap appends a TWAPObservation{timestamp, price_cumulative_a, price_cumulative_b};
  twap() finds the oldest observation within the requested period and returns Δcumulative/Δtime
• LP tokens are separate contract instances (one per pool) so they are fully composable
• All prices are fixed-point integers scaled by 1e7 to avoid floating point in Soroban
```

---

## Project Structure

```
Soroswap-Pro/
├── Cargo.toml                        # Rust workspace (amm + lp-token)
│
├── contracts/
│   ├── amm/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs                # AMM contract: pool logic, swap, TWAP
│   │       └── test.rs               # Soroban testutils suite (k invariant, TWAP, etc.)
│   └── lp-token/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs                # Minimal SEP-41 token: mint/burn/transfer
│
├── frontend/
│   ├── package.json
│   └── src/
│       ├── app/
│       │   ├── layout.tsx            # Root layout
│       │   ├── page.tsx              # Landing — links to swap & pool
│       │   ├── swap/
│       │   │   └── page.tsx          # Swap UI with real-time price impact
│       │   └── pool/
│       │       └── page.tsx          # LP dashboard: pool share + IL estimate
│       ├── components/
│       │   └── PriceChart.tsx        # TWAP line chart (recharts)
│       └── lib/
│           └── amm.ts                # calcAmountOut, calcPriceImpact, calcIL, executeSwap
│
└── bot/
    ├── package.json
    ├── tsconfig.json
    └── src/
        └── bot.ts                    # Arbitrage bot: poll → compare → execute
```

---

## Getting Started

### Prerequisites

- Rust + `wasm32-unknown-unknown` target: `rustup target add wasm32-unknown-unknown`
- [Stellar CLI](https://github.com/stellar/stellar-cli): `cargo install stellar-cli`
- Node.js ≥ 20

### 1 — Build Contracts

```bash
# from repo root
cargo build --release --target wasm32-unknown-unknown

# compiled artefacts
# target/wasm32-unknown-unknown/release/soroswap_amm.wasm
# target/wasm32-unknown-unknown/release/soroswap_lp_token.wasm
```

### 2 — Run Tests

```bash
cargo test
```

All tests in `contracts/amm/src/test.rs` run against the Soroban test environment, including k-invariant assertions after every swap.

### 3 — Deploy to Testnet

```bash
# deploy LP token contract (returns CONTRACT_ID)
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/soroswap_lp_token.wasm \
  --source <YOUR_SECRET_KEY> \
  --network testnet

# deploy AMM contract
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/soroswap_amm.wasm \
  --source <YOUR_SECRET_KEY> \
  --network testnet

# create XLM/USDC pool
stellar contract invoke \
  --id <AMM_CONTRACT_ID> \
  --source <YOUR_SECRET_KEY> \
  --network testnet \
  -- create_pool \
  --token_a <XLM_ADDRESS> \
  --token_b <USDC_ADDRESS> \
  --fee_bps 30
```

### 4 — Run Frontend

```bash
cd frontend
npm install
npm run dev          # http://localhost:3000
```

Set your AMM contract ID in `frontend/src/lib/amm.ts` (`AMM_CONTRACT_ID` constant).

### 5 — Run Arbitrage Bot

```bash
cd bot
npm install
# configure in src/bot.ts: AMM_CONTRACT_ID, HORIZON_URL, SECRET_KEY, POLL_INTERVAL_MS
npm start
```

---

## Contributing

### Workflow

1. Fork the repo and create a feature branch: `git checkout -b feat/your-feature`
2. Make changes; keep commits atomic and conventional (`feat:`, `fix:`, `test:`, `docs:`)
3. Ensure `cargo test` passes and `npm run build` in `frontend/` succeeds
4. Open a PR against `main` with a description covering: what changed, why, and how it was tested

### PR Guide

- Keep PRs focused — one feature or fix per PR
- Title under 70 characters; use the description for details
- Link any related issue with `Closes #<issue>`
- CI must be green before review

### Areas to Contribute

| Area | Examples |
|---|---|
| Contracts | Multi-hop routing, concentrated liquidity, governance |
| Frontend | Wallet connect (Freighter), transaction history, mobile layout |
| Bot | MEV protection, multi-pair scanning, gas optimisation |
| Tooling | Deployment scripts, Makefile, Docker compose for local testnet |
| Docs | Tutorials, architecture deep-dives, video walkthroughs |

### Code Standards

- **Rust**: `cargo clippy -- -D warnings`; no `unwrap()` in contract code — use `expect()` with descriptive messages or propagate errors
- **TypeScript**: strict mode enabled; no `any`; format with `prettier`
- **Commits**: [Conventional Commits](https://www.conventionalcommits.org/)
- **Tests**: new contract functions must have a corresponding test; assert invariants explicitly

### Reporting Issues

Open a GitHub issue with:
- Environment (OS, Rust version, Node version, Stellar CLI version)
- Steps to reproduce
- Expected vs actual behaviour
- Relevant logs or error output
