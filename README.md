# Technical Implementation Plan: Lunar Finance × Stellar Integration

**Prepared by:** Lunar Finance
**Related submission:** Stellar Community Fund Application (Integration Track)
**Scope:** Stellar DEX Aggregation, Cross-Chain Meta Aggregation, Stellar Wallet Connect Support, Advanced Routing Engine with Transaction Simulation

---

## 1. Overview

This document sets out our technical implementation plan for bringing Stellar into Lunar Finance's unified routing system. We are building four integrated capabilities:

1. **Stellar DEX Aggregation** — unified routing across Stellar-native liquidity venues
2. **Cross-Chain Meta Aggregation** — routing between Stellar and external chains via bridge and intent-based protocols
3. **Stellar Wallet Connect Support** — native wallet integration for signing and account management
4. **Advanced Routing Engine with Transaction Simulation** — pre-execution simulation across all route types to minimize failure rates

We already operate production-grade aggregation infrastructure across 40+ DeFi protocol integrations, and we're extending that same adapter-based architecture to Stellar rather than building a parallel, Stellar-specific stack. The sections below detail our architecture, core components, data flow, and technical dependencies for each capability.

---

## 2. System Architecture (High Level)

```
                          +-------------------------------+
                          |        Lunar Finance App        |
                          |    (Swap UI / Route Selector)   |
                          +----------------+-----------------+
                                           |
                          +----------------v-----------------+
                          |      Our Unified Routing Engine    |
                          |    (quote aggregation + sim)       |
                          +---+-------------------------+-----+
                              |                         |
              +----------------v--------+   +------------v----------------+
              |   Stellar DEX Adapters    |   |  Cross-Chain Adapters       |
              |   (Aquarius, Soroswap,    |   |  (Near Intents, Allbridge,  |
              |   Stellar SDEX / path     |   |   Axelar, CCTP)              |
              |   payments)                |   |                              |
              +----------------+--------+   +------------+----------------+
                              |                         |
                          +---v-------------------------v---+
                          |    Stellar Network Layer          |
                          |   (Horizon API + Soroban RPC)     |
                          +----------------+-------------------+
                                           |
                          +----------------v-----------------+
                          |   Our Wallet Connect Layer          |
                          |   (Freighter, Lobstr, others)       |
                          +---------------------------------------+
```

**Design principle:** We're integrating Stellar using the same adapter-framework pattern we already run across our other chain integrations — each liquidity source or bridge is wrapped in a standardized adapter interface so our routing engine can query, compare, and execute across sources without source-specific logic bleeding into core routing code. This lets us move quickly on Stellar because we are extending proven infrastructure, not building from zero.

---

## 3. Component 1: Stellar DEX Aggregation

### 3.1 Objective
We will aggregate liquidity from multiple Stellar-based DEXs into a single routing layer so users get the best available price with minimal slippage, sourced from one unified interface in the Lunar Finance app.

### 3.2 Liquidity sources we are integrating

| Source | Type | Our integration method |
|---|---|---|
| **Aquarius (AQUA)** | Soroban AMM | We connect to the Aquarius AMM router contract via Soroban RPC, using Stellar SDK v15+ |
| **Soroswap** | Soroban-native DEX aggregator | We integrate Soroswap as a liquidity source in our own aggregation layer, avoiding duplication of its internal routing logic |
| **Stellar SDEX (native order book)** | Native protocol-level order book | We access this via Horizon's path payment and orderbook/offers endpoints — no smart contract required |
| **Stellar Broker** | Liquidity venue | We will complete technical discovery on this integration's exposure (contract, API, or SDEX wrapper) during Milestone 1 and finalize the adapter design accordingly |

### 3.3 Our technical approach

- **Native SDEX liquidity**: we access this through **Path Payments**, Stellar's built-in mechanism for cross-asset transfers that route through the SDEX and/or liquidity pools. We support both operation modes:
  - **Path Payment Strict Send** — our default for standard swap UX, where the user specifies what they're spending
  - **Path Payment Strict Receive** — used where a user needs an exact destination amount, such as settling an exact invoice amount through Lunar Spend
- **Path discovery**: we query Horizon's `/paths/strict-send` and `/paths/strict-receive` endpoints, which return all viable multi-hop conversion paths between a source and destination asset, to build our candidate route set before ranking by price and reliability.
- **Soroban-based AMMs (Aquarius, Soroswap)**: we interact with these via Soroban RPC rather than Horizon directly. Each has its own router contract ID that we call through the Soroban RPC endpoint; we treat these addresses as versioned configuration that we verify and update as needed, since router contract IDs are updated periodically (e.g., following network resets).
- **Adapter interface**: each DEX source implements our common `getQuote(assetIn, assetOut, amount)` and `buildTransaction(route)` interface, so our routing engine treats SDEX, Aquarius, and Soroswap interchangeably at the routing layer.

### 3.4 Our key dependencies
- Stellar SDK (v15+) for Soroban contract interaction
- Horizon API access for SDEX path queries
- Soroban RPC access for AMM contract calls
- Trustline handling — Stellar requires an account trustline for any non-native asset, so our wallet/onboarding flow handles trustline creation transparently the first time a user interacts with a given asset (e.g., their first EURC or AQUA swap)

---

## 4. Component 2: Cross-Chain Meta Aggregation

### 4.1 Objective
We will enable routing of assets into and out of Stellar from external chains, using Stellar as both a source and destination of liquidity within our broader multichain routing system.

### 4.2 Bridge/intent protocols we are integrating

| Protocol | Model | Role in our routing |
|---|---|---|
| **Near Intents** | Intent-based routing | Solver network fulfills cross-chain intents; we use this where a direct bridge route is unavailable or inefficient |
| **Allbridge** | Bridge-based (lock/mint or liquidity-pool) | Direct bridge route for supported asset pairs into/out of Stellar |
| **Axelar** | Bridge-based, general message passing | Broader chain coverage for less common destination chains |
| **CCTP (Circle's Cross-Chain Transfer Protocol)** | Native USDC burn-and-mint | Our preferred route for USDC transfers specifically, since CCTP avoids wrapped-asset risk by burning and minting native USDC on each side |

### 4.3 Our technical approach

- **Adapter pattern, consistent with Section 3**: we wrap each bridge/intent protocol in an adapter exposing our standardized `getCrossChainQuote(sourceChain, destChain, asset, amount)` interface, returning estimated output amount, fee, and estimated settlement time.
- **Routing decision logic**: for a given cross-chain request, our routing engine queries all applicable adapters in parallel and selects the optimal path based on a weighted score of price, fee, and speed/reliability (see Section 6 for how this feeds into our route simulation).
- **CCTP priority for USDC**: given USDC's centrality to our other product features (fiat ramp, Lunar Spend), we default to CCTP whenever both source and destination chains support it, since it avoids the smart-contract and liquidity risk inherent in wrapped-asset bridges.
- **Settlement tracking**: cross-chain transactions settle across two distinct confirmations (source-chain burn/lock, destination-chain mint/release). We extend our existing transaction status tracker with a Stellar-specific listener built on Horizon's transaction streaming endpoint to detect destination-side completion.

### 4.4 Our key dependencies
- Bridge/intent protocol SDKs or REST APIs for Near Intents, Allbridge, Axelar, and CCTP
- Horizon streaming (Server-Sent Events) for real-time Stellar-side transaction confirmation
- Our shared cross-chain status/reconciliation ledger, reused from our existing bridge and payments infrastructure rather than built fresh for Stellar

---

## 5. Component 3: Stellar Wallet Connect Support

### 5.1 Objective
We will integrate Stellar-native wallets for transaction signing and account management, giving our users a native Stellar experience without leaving the Lunar Finance app.

### 5.2 Wallets we are integrating at launch
- **Freighter** — the reference Stellar wallet, widely used across the ecosystem
- **Lobstr** — mobile-first Stellar wallet, a strong fit given our mobile-first product
- We are building against a standardized connection interface, not hardcoding to these two, so we can add further Stellar wallets without core changes

### 5.3 Our technical approach

- **Connection layer**: we implement Stellar wallet connection using the same abstraction pattern we already use for our EVM wallet connections, giving users one consistent "connect wallet" experience across chains.
- **Signing flow**: we construct Stellar transactions via Horizon/Soroban SDK, pass the XDR-encoded transaction envelope to the connected wallet for signing, and submit the signed transaction back through Horizon.
- **Account/trustline state**: on wallet connection, we query the account's existing trustlines and balances via Horizon (`/accounts/{account_id}`) to determine whether a trustline needs to be established before a given swap can proceed.
- **Session management**: we align Stellar wallet sessions with our existing session/permission model, so the experience is consistent with the rest of our app rather than introducing a separate mental model for users.

### 5.4 Our key dependencies
- Freighter and Lobstr connection SDKs/APIs
- Horizon account endpoint for balance/trustline state
- Consistency with our existing multi-chain wallet connection flow

---

## 6. Component 4: Advanced Routing Engine with Transaction Simulation

### 6.1 Objective
We are extending our routing engine to simulate every candidate execution path — DEX and cross-chain — before submission, catching failures pre-execution rather than on-chain, in order to minimize failed transactions and strengthen user trust in the platform.

### 6.2 Our technical approach

- **Multi-path candidate generation**: for a given swap/bridge request, our engine generates all viable candidate routes by querying:
  - Stellar SDEX path payment endpoints (Strict Send/Receive)
  - Soroban AMM adapters (Aquarius, Soroswap)
  - Cross-chain bridge/intent adapters (Section 4)
- **Simulation before submission**:
  - For SDEX path payments, we use Horizon's path-finding endpoints for pre-flight validation of path viability.
  - For Soroban contract calls (Aquarius, Soroswap), we use Soroban RPC's transaction simulation capability to preview the contract call's outcome and catch failures — insufficient liquidity, slippage limit breach, expired ledger sequence — before the user signs and submits.
  - For cross-chain routes, we validate the source-chain leg fully and estimate the destination-chain outcome, reflecting this transparently in our UI's reliability indicator per route.
- **Path ranking**: after simulation, we rank viable paths by a composite score — price, fee, estimated settlement time, and simulation-confirmed reliability — and present the top path to the user, with alternate paths available if they prefer to override.
- **Pre-validation failure handling**: any route that fails simulation is discarded before it's ever shown to the user. This is the core mechanism behind our target of a sub-2% failed-transaction rate across all routes.
- **Execution**: once a user confirms, our engine builds the final transaction (or transaction set, for multi-hop/cross-chain routes), routes it to our wallet connect layer (Section 5) for signing, and submits via Horizon/Soroban RPC.

### 6.3 Our key dependencies
- Soroban RPC simulation endpoint
- Horizon path-finding endpoints
- Our shared route-scoring logic, kept in the routing engine's chain-agnostic core so scoring is consistent across Stellar and our other supported chains

---

## 7. Cross-Cutting Technical Considerations

### 7.1 Asset/trustline handling
Non-native Stellar assets require an account trustline before they can be held or received. We handle this transparently: our flow detects missing trustlines pre-swap and includes the trustline operation in the transaction automatically where appropriate, rather than surfacing a raw Stellar-specific error to the user.

### 7.2 Fee/reliability transparency
Consistent with our approach elsewhere in the app, every route we show a user surfaces an all-in cost (price + network fee + any bridge/protocol fee) and a settlement time estimate, not just headline price.

### 7.3 Error handling and fallback
- If a preferred route (e.g., CCTP for USDC) is unavailable or fails simulation, our engine automatically falls back to the next-best viable route rather than surfacing a dead end to the user.
- Cross-chain routes have defined retry/refund logic for partial-failure states (source-chain leg succeeds, destination-chain leg fails), consistent with the reconciliation approach we run across our other cross-border payment features.

### 7.4 Security
- We pin and verify all Soroban contract addresses (Aquarius router, Soroswap router, etc.) against official documentation at integration time, and monitor for changes through an ongoing verification process rather than hardcoding them permanently.
- Our trustline and signing flows never expose private key material to our backend, consistent with the non-custodial principle underpinning our core wallet infrastructure.

### 7.5 Testing
- We test all adapters against Stellar **testnet** endpoints (Horizon testnet, Soroban testnet RPC) before mainnet integration.
- We treat failure-path testing (insufficient liquidity, missing trustline, expired paths, bridge timeout) as a first-class part of our test suite, not an afterthought to happy-path swap testing.

---

*This document is a technical companion to our Stellar Community Fund Application (Integration Track).*

## Repository Structure

```
lunar-stellar/
├── docs/                          # Technical documentation
├── src/
│   ├── adapters/
│   │   ├── dex/                   # Aquarius, Soroswap, SDEX, Stellar Broker
│   │   └── cross-chain/           # Near Intents, Allbridge, Axelar, CCTP
│   ├── routing/                   # Unified routing engine
│   ├── simulation/                # Pre-execution transaction simulation
│   ├── wallet/                    # Freighter, Lobstr wallet connect
│   └── stellar/                   # Horizon API + Soroban RPC layer
└── tests/
```
