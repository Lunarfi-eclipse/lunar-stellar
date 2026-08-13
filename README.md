# Lunar Finance × Stellar Integration — Technical Implementation Plan


---

## 1. Overview

This document sets out the technical implementation plan for bringing Stellar into Lunar Finance's unified routing system. The integration delivers four capabilities:

1. **Stellar DEX aggregation** — unified routing across Stellar-native liquidity venues
2. **Cross-chain meta-aggregation** — routing between Stellar and external chains via bridge and intent-based protocols
3. **Stellar wallet support** — native wallet integration for signing and account management
4. **Routing engine with transaction simulation** — pre-execution simulation across all route types to reduce failure rates

Lunar Finance operates aggregation infrastructure spanning 40+ DeFi protocol integrations across multiple ecosystems. Stellar is added as a new set of adapters inside that existing system — not a parallel, Stellar-specific stack.

The plan is deliberately phased: a small set of high-liquidity sources is integrated and proven on testnet first, expanded on mainnet only after the core path works end to end. External protocol support cited in this document has been verified against live deployments as of August 2026.

---

## 2. System Architecture (High Level)

```
            +------------------------------------+
            |         Lunar Finance App          |
            |     (Swap UI / Route Selector)     |
            +-----------------+------------------+
                              |
            +-----------------v------------------+
            |       Unified Routing Engine       |
            |  (quote aggregation + simulation)  |
            +--------+------------------+--------+
                     |                  |
  +------------------v-------+   +------v---------------------+
  | Stellar DEX Adapters     |   | Cross-Chain Adapters       |
  | (Soroswap, Aquarius,     |   | (Allbridge, Near Intents,  |
  |  SushiSwap,              |   |  Axelar, CCTP)             |
  |  Stellar Broker)         |   |                            |
  +------------------+-------+   +------+---------------------+
                     |                  |
            +--------v------------------v--------+
            |       Stellar Network Layer        |
            |     (Horizon API + Soroban RPC)    |
            +-----------------+------------------+
                              |
            +-----------------v------------------+
            |      Wallet Connection Layer       |
            |  (Stellar Wallets Kit: Freighter,  |
            |   Lobstr, WalletConnect, others)   |
            +------------------------------------+
```

**Design principle:** each liquidity source or bridge is wrapped in a standardized adapter interface, so the routing engine can query, compare, and execute across sources without source-specific logic leaking into core routing code. This is the same adapter-framework pattern Lunar Finance already runs across its other chain integrations.

---

## 3. Component 1: Stellar DEX Aggregation

### 3.1 Objective

Aggregate liquidity from multiple Stellar venues into a single routing layer, so users get the best available price from one interface in the Lunar Finance app.

### 3.2 Liquidity sources

| Source | Type | Integration method |
|---|---|---|
| Soroswap | Soroban-native DEX/aggregator | Router contract via Soroban RPC, as a liquidity source in our aggregation layer |
| Aquarius (AQUA) | Soroban AMM — largest liquidity hub on Stellar (~$44M TVL, DefiLlama, Aug 2026) | AMM router contract via Soroban RPC |
| SushiSwap | Soroban DEX with v3-style concentrated liquidity, live on Stellar since Feb 2026 | Router contract via Soroban RPC — extends the SushiSwap adapters Lunar already runs on EVM chains |
| Stellar Broker | Hosted multi-source swap router (Stellar Expert team) spanning SDEX, classic AMMs, Soroswap, and Aquarius | Its REST/WebSocket router API via the official client library, wrapped in our standard adapter interface |

We are prioritizing Soroswap and Aquarius — the deepest Soroban-native venues — with SushiSwap and Stellar Broker following. Stellar's native order book (SDEX) is not a direct integration target at this stage; its liquidity remains reachable indirectly through Stellar Broker, which routes across SDEX and classic AMMs. Because Stellar Broker and Soroswap's aggregator overlap with venues we also query directly, duplicate candidate routes are deduplicated at the ranking layer (Section 6).

### 3.3 Technical approach

- **Soroban-based venues (Aquarius, Soroswap, SushiSwap)** are called via Soroban RPC. Router contract IDs are treated as versioned configuration, verified against official documentation at integration time and monitored for changes (e.g., after protocol upgrades or network resets) — never hardcoded permanently.
- **Stellar Broker** is queried through its hosted REST/WebSocket router API; returned transaction envelopes are validated against the quoted parameters (assets, amounts, slippage bounds) before they are passed to the wallet for signing.
- **Adapter interface**: every DEX source implements a common `getQuote(assetIn, assetOut, amount)` and `buildTransaction(route)` interface, so the routing engine treats Aquarius, Soroswap, SushiSwap, and Stellar Broker interchangeably.

### 3.4 Key dependencies

- `@stellar/stellar-sdk` (official JavaScript SDK, v16+) for transaction building and Soroban contract interaction
- Horizon API access for account state, transaction submission, and streaming
- Soroban RPC access for AMM contract calls and simulation
- Stellar Broker router API access (REST/WebSocket) via its client library
- Trustline handling — Stellar requires an account trustline for any non-native asset; the swap flow detects missing trustlines and handles creation transparently (Section 7.1)

---

## 4. Component 2: Cross-Chain Meta-Aggregation

### 4.1 Objective

Enable routing of assets into and out of Stellar from external chains, making Stellar both a source and destination of liquidity within Lunar's multichain routing system.

### 4.2 Bridge and intent protocols

All four protocols below have **live Stellar support**, verified against their deployments in August 2026. Integration is still sequenced — a first wave of two protocols (one bridge-based, one intent-based), then the remainder — so each adapter ships with full settlement tracking and failure-path coverage rather than all landing at once.

| Protocol | Model | Role in routing | Status |
|---|---|---|---|
| Allbridge | Bridge (liquidity-pool) | Direct bridge route for supported pairs; Lunar already integrates Allbridge on other chains | First wave (Stellar support live since 2024) |
| Near Intents | Intent-based | Solver network fulfills cross-chain intents where a direct bridge route is unavailable or inefficient | First wave (Stellar live since Aug 2025) |
| Axelar | Bridge / general message passing | Broader chain coverage for less common destinations | Second wave (ITS live on Stellar) |
| CCTP (Circle) | Native USDC burn-and-mint | Preferred route for USDC — burn-and-mint of native USDC avoids wrapped-asset risk | Second wave (live on Stellar since May 2026) |

**A note on USDC:** USDC on Stellar is natively issued by Circle, and Circle's CCTP is live on Stellar as of May 2026 — native USDC moves between Stellar and other CCTP chains by burn-and-mint, with no wrapped assets. Given USDC's centrality to Lunar's other product features (fiat ramp, Lunar Spend), the engine defaults to CCTP whenever both source and destination chains support it; first-wave protocols cover USDC routes until the CCTP adapter ships.

### 4.3 Technical approach

- **Adapter pattern, consistent with Section 3**: each protocol is wrapped in an adapter exposing `getCrossChainQuote(sourceChain, destChain, asset, amount)`, returning estimated output, fees, and estimated settlement time.
- **Routing decision logic**: for a given cross-chain request, the engine queries all applicable adapters in parallel and ranks routes by a weighted score of price, fee, and speed/reliability (Section 6).
- **Settlement tracking**: cross-chain transactions settle across two confirmations (source-chain lock/burn, destination-chain release/mint). The existing transaction status tracker is extended with a Stellar listener built on Horizon's streaming (Server-Sent Events) endpoint to detect destination-side completion.
- **Partial-failure handling**: defined retry/refund logic for the state where the source leg succeeds and the destination leg fails, reusing the reconciliation approach from Lunar's existing bridge infrastructure (Section 7.3).

### 4.4 Key dependencies

- Protocol SDKs or REST APIs for each integrated bridge/intent protocol
- Horizon streaming for real-time Stellar-side confirmation
- Lunar's existing cross-chain status/reconciliation ledger, reused rather than rebuilt

---

## 5. Component 3: Stellar Wallet Support

### 5.1 Objective

Integrate Stellar-native wallets for transaction signing and account management, with the same "connect wallet" experience users get on Lunar's other supported chains.

### 5.2 Approach

Rather than building one-off connectors per wallet, the connection layer is built on the **Stellar Wallets Kit**, the community-standard wallet abstraction for Stellar dApps. This provides Freighter, Lobstr (via WalletConnect), xBull, Albedo, and others through a single interface — more wallets at launch, less bespoke connection code to maintain.

- **Launch targets**: Freighter (the reference extension wallet) and Lobstr (mobile-first, aligned with Lunar's mobile-first product) are the first-class, fully tested paths. Other kit-supported wallets are enabled on a best-effort basis.
- **Signing flow**: transactions are constructed with the Stellar SDK, passed to the connected wallet as an XDR-encoded envelope for signing, and submitted through Horizon (or Soroban RPC for contract transactions).
- **Account and trustline state**: on connection, the account's balances and trustlines are read via Horizon (`/accounts/{account_id}`) to determine whether a trustline must be established before a swap can proceed.
- **Session management**: Stellar wallet sessions plug into Lunar's existing session/permission model, keeping one consistent mental model across chains.

### 5.3 Key dependencies

- Stellar Wallets Kit; Freighter API; WalletConnect (for Lobstr)
- Horizon account endpoints for balance/trustline state
- Lunar's existing multi-chain wallet connection abstraction

---

## 6. Component 4: Routing Engine with Transaction Simulation

### 6.1 Objective

Extend the routing engine to simulate every candidate execution path — DEX and cross-chain — before submission, catching failures pre-execution rather than on-chain.

### 6.2 Technical approach

- **Candidate generation**: for a given request, the engine gathers viable routes from the Soroban DEX adapters (Aquarius, Soroswap, SushiSwap), the Stellar Broker router, and the cross-chain adapters.
- **Simulation before submission**:
  - Soroban contract calls (Aquarius, Soroswap, SushiSwap) use Soroban RPC's `simulateTransaction` to preview the outcome and catch failures — insufficient liquidity, slippage-limit breach, sequence number or time-bounds errors before the user signs.
  - Routes returned by Stellar Broker are re-validated against their quotes (assets, amounts, slippage bounds) before signing.
  - Cross-chain routes have their source-chain leg fully validated and their destination-chain outcome estimated; the difference is reflected honestly in the per-route reliability indicator in the UI.
- **Ranking**: viable routes are ranked by a composite of price, fees, estimated settlement time, and simulation-confirmed reliability. The top route is preselected; alternates remain user-selectable.
- **Failure filtering**: routes that fail simulation are discarded before display.
- **Execution**: on confirmation, the engine builds the final transaction (or transaction set for multi-hop/cross-chain routes), hands it to the wallet layer for signing, and submits via Horizon/Soroban RPC.

### 6.3 Success metric

Target: **under 2% of signed, submitted Stellar-route transactions failing on-chain**, measured over a trailing 30-day window beginning four weeks after production launch (allowing one tuning cycle). Failure rates are tracked per route type (Soroban AMM, Stellar Broker, cross-chain) from day one so the metric is observable, not aspirational.

### 6.4 Key dependencies

- Soroban RPC simulation endpoint
- Lunar's shared route-scoring logic, kept chain-agnostic so scoring is consistent across Stellar and other supported chains

---

## 7. Cross-Cutting Technical Considerations

### 7.1 Trustline handling

Non-native Stellar assets require an account trustline before they can be held. The flow detects missing trustlines pre-swap and, where appropriate, bundles the trustline operation into the same transaction — the user sees a swap, not a raw Stellar-specific error.

### 7.2 Fee and reliability transparency

Every route shown to a user surfaces all-in cost (price + network fee + any bridge/protocol fee) and a settlement-time estimate, not just headline price — consistent with the rest of the app.

### 7.3 Error handling and fallback

If a preferred route is unavailable or fails simulation, the engine falls back to the next-best viable route rather than surfacing a dead end. Cross-chain partial failures follow the defined retry/refund path in Section 4.3.

### 7.4 Security

- All Soroban contract addresses (Aquarius router, Soroswap router, etc.) are pinned, verified against official documentation at integration time, and monitored for changes.
- Signing flows never expose private key material to Lunar's backend, consistent with the non-custodial principle of the core wallet infrastructure.
- An external-facing security review of the Stellar adapters and signing path precedes general availability.

### 7.5 Testing

- All adapters are built and tested against Stellar **testnet** (Horizon testnet, Soroban testnet RPC) before any mainnet exposure.
- Failure-path testing — insufficient liquidity, missing trustlines, stale quotes, bridge timeout, partial cross-chain failure — is a first-class part of the suite, not an afterthought to happy-path swaps.

---

## 8. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Protocol chain support ≠ asset-pair support — a live protocol may not cover a needed pair | Some cross-chain routes unavailable | Pair-level coverage verified per adapter before it ships; the engine falls back across the other integrated protocols |
| Soroban RPC reliability / rate limits under production quote load | Degraded quoting or simulation | Provider redundancy and caching at the network layer; Stellar Broker's hosted API provides an independent quoting path |
| Thin liquidity on Soroban AMMs for launch assets | Poor pricing vs. expectations | Aquarius (~$44M TVL) anchors depth; routes are ranked on simulated output, so thin sources naturally lose ranking rather than harming users |
| Router contract changes or network resets invalidate pinned addresses | Broken adapter until updated | Contract IDs held as versioned config with monitoring, not hardcoded (Section 7.4) |
| Cross-chain partial failure (source leg succeeds, destination fails) | Stuck user funds | Defined retry/refund reconciliation path (Section 4.3), reused from existing production bridge infrastructure |
| Slower-than-expected Soroban adapter work | Delayed aggregation rollout | Stellar Broker's router API — one API integration spanning multiple venues — is a shippable interim path while direct adapters mature |

---

## 9. Assumptions and External Dependencies

- Public Horizon and Soroban RPC infrastructure (or commercial providers) remains available for testnet and mainnet.
- Aquarius, Soroswap, and SushiSwap router contracts remain deployed and documented on mainnet.
- Stellar Broker's hosted router API remains available — it is also the indirect path to SDEX and classic-AMM liquidity.
- Protocol support in Section 4.2 was verified against live deployments in August 2026 and is re-checked at integration time — this document intentionally avoids committing to third-party roadmaps.

---

*This document is the technical companion to our Stellar Community Fund Application (Integration Track).*
