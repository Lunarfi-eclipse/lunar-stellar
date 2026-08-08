# Lunar Finance × Stellar Integration — Technical Implementation Plan

**Prepared by:** Lunar Finance
**Related submission:** Stellar Community Fund Application (Integration Track)
**Status:** Planning — implementation begins at project start; this repository holds the implementation plan
**Last updated:** August 2026

---

## 1. Overview

This document sets out the technical implementation plan for bringing Stellar into Lunar Finance's unified routing system. The integration delivers four capabilities:

1. **Stellar DEX aggregation** — unified routing across Stellar-native liquidity venues
2. **Cross-chain meta-aggregation** — routing between Stellar and external chains via bridge and intent-based protocols
3. **Stellar wallet support** — native wallet integration for signing and account management
4. **Routing engine with transaction simulation** — pre-execution simulation across all route types to reduce failure rates

Lunar Finance operates aggregation infrastructure spanning 40+ DeFi protocol integrations across multiple ecosystems. Stellar is added as a new set of adapters inside that existing system — not a parallel, Stellar-specific stack.

The plan is deliberately phased: a small set of high-liquidity sources is integrated and proven on testnet first, expanded on mainnet only after the core path works end to end. Where an external protocol's Stellar support is not yet confirmed, this document says so explicitly rather than assuming it (see Sections 4, 9, and 10).

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
  | (SDEX path payments,     |   | (Allbridge, Near Intents,  |
  |  Aquarius, Soroswap,     |   |  Axelar†, CCTP†)           |
  |  Stellar Broker*)        |   |                            |
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

\* Subject to Milestone 1 technical discovery (Section 3.2).
† Subject to availability verification on Stellar (Section 4.2).

**Design principle:** each liquidity source or bridge is wrapped in a standardized adapter interface, so the routing engine can query, compare, and execute across sources without source-specific logic leaking into core routing code. This is the same adapter-framework pattern Lunar Finance already runs across its other chain integrations.

---

## 3. Component 1: Stellar DEX Aggregation

### 3.1 Objective

Aggregate liquidity from multiple Stellar venues into a single routing layer, so users get the best available price from one interface in the Lunar Finance app.

### 3.2 Liquidity sources

| Source | Type | Integration method | Phase |
|---|---|---|---|
| Stellar SDEX (native order book) | Protocol-level order book | Horizon path payment and orderbook endpoints; no smart contract required | Milestone 1 |
| Soroswap | Soroban-native DEX/aggregator | Router contract via Soroban RPC, as a liquidity source in our aggregation layer | Milestone 2 |
| Aquarius (AQUA) | Soroban AMM | AMM router contract via Soroban RPC | Milestone 2 |
| Stellar Broker | Liquidity venue | Technical discovery in Milestone 1 (contract, API, or SDEX wrapper); adapter built only if discovery confirms a viable integration surface | Milestone 3 |

We are starting with SDEX because it requires no contract integration and covers the deepest XLM/USDC/EURC liquidity, then layering Soroban AMMs on top.

### 3.3 Technical approach

- **Native SDEX liquidity** is accessed through **path payments**, Stellar's built-in mechanism for cross-asset transfers routed through the SDEX and/or liquidity pools. Both operation modes are supported:
  - **Path Payment Strict Send** — the default for standard swap UX, where the user specifies the input amount
  - **Path Payment Strict Receive** — used where an exact destination amount is required, such as settling an invoice through Lunar Spend
- **Path discovery** queries Horizon's `/paths/strict-send` and `/paths/strict-receive` endpoints, which return viable multi-hop conversion paths between a source and destination asset. These form the candidate route set before ranking.
- **Soroban-based AMMs (Aquarius, Soroswap)** are called via Soroban RPC. Router contract IDs are treated as versioned configuration, verified against official documentation at integration time and monitored for changes (e.g., after protocol upgrades or network resets) — never hardcoded permanently.
- **Adapter interface**: every DEX source implements a common `getQuote(assetIn, assetOut, amount)` and `buildTransaction(route)` interface, so the routing engine treats SDEX, Aquarius, and Soroswap interchangeably.

### 3.4 Key dependencies

- `@stellar/stellar-sdk` (official JavaScript SDK) for transaction building and Soroban contract interaction
- Horizon API access for SDEX path queries and account state
- Soroban RPC access for AMM contract calls and simulation
- Trustline handling — Stellar requires an account trustline for any non-native asset; the swap flow detects missing trustlines and handles creation transparently (Section 7.1)

---

## 4. Component 2: Cross-Chain Meta-Aggregation

### 4.1 Objective

Enable routing of assets into and out of Stellar from external chains, making Stellar both a source and destination of liquidity within Lunar's multichain routing system.

### 4.2 Bridge and intent protocols

Milestone 2 commits to **two** protocols in production — one bridge-based, one intent-based. The remaining candidates are integrated in Milestone 3 as availability is confirmed. Support for Stellar varies in maturity across these protocols, so Milestone 1 includes a verification pass on each before adapter work is scheduled.

| Protocol | Model | Role in routing | Status |
|---|---|---|---|
| Allbridge | Bridge (liquidity-pool) | Direct bridge route for supported pairs; Lunar already integrates Allbridge on other chains | Committed (Milestone 2) |
| Near Intents | Intent-based | Solver network fulfills cross-chain intents where a direct bridge route is unavailable or inefficient | Committed (Milestone 2), pending availability verification |
| Axelar | Bridge / general message passing | Broader chain coverage for less common destinations | Milestone 3, pending availability verification |
| CCTP (Circle) | Native USDC burn-and-mint | Preferred USDC route where available — avoids wrapped-asset risk | Milestone 3, contingent on CCTP support for Stellar |

**A note on USDC:** USDC on Stellar is natively issued by Circle, which simplifies the asset side of this integration considerably. Whether transfers can use CCTP specifically depends on Circle shipping CCTP support for Stellar; until then, USDC routes use the committed bridge/intent protocols. We will not represent CCTP as available before it is.

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

- **Candidate generation**: for a given request, the engine gathers viable routes from the SDEX path payment endpoints, the Soroban AMM adapters, and the cross-chain adapters.
- **Simulation before submission**:
  - SDEX path payments are pre-validated via Horizon's path-finding endpoints.
  - Soroban contract calls (Aquarius, Soroswap) use Soroban RPC's `simulateTransaction` to preview the outcome and catch failures — insufficient liquidity, slippage-limit breach, sequence number or time-bounds errors — before the user signs.
  - Cross-chain routes have their source-chain leg fully validated and their destination-chain outcome estimated; the difference is reflected honestly in the per-route reliability indicator in the UI.
- **Ranking**: viable routes are ranked by a composite of price, fees, estimated settlement time, and simulation-confirmed reliability. The top route is preselected; alternates remain user-selectable.
- **Failure filtering**: routes that fail simulation are discarded before display.
- **Execution**: on confirmation, the engine builds the final transaction (or transaction set for multi-hop/cross-chain routes), hands it to the wallet layer for signing, and submits via Horizon/Soroban RPC.

### 6.3 Success metric

Target: **under 2% of signed, submitted Stellar-route transactions failing on-chain**, measured over a trailing 30-day window beginning four weeks after production launch (allowing one tuning cycle). Failure rates are tracked per route type (SDEX, Soroban AMM, cross-chain) from day one so the metric is observable, not aspirational.

### 6.4 Key dependencies

- Soroban RPC simulation endpoint; Horizon path-finding endpoints
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
- Milestone 3 includes an external-facing security review of the Stellar adapters and signing path before general availability.

### 7.5 Testing

- All adapters are built and tested against Stellar **testnet** (Horizon testnet, Soroban testnet RPC) before any mainnet exposure.
- Failure-path testing — insufficient liquidity, missing trustlines, expired paths, bridge timeout, partial cross-chain failure — is a first-class part of the suite, not an afterthought to happy-path swaps.

---

## 8. Implementation Milestones

Sixteen weeks of effort from project start, in three milestones. Each has explicit exit criteria; a milestone is not "done" until they pass.

### Milestone 1 (Weeks 1–6): Core Stellar integration, on testnet

- Stellar network layer (Horizon + Soroban RPC clients) and adapter scaffolding
- Wallet connection via Stellar Wallets Kit; Freighter and Lobstr signing paths tested
- SDEX adapter: path-payment quoting and execution for XLM, USDC, EURC, AQUA
- Availability verification for Near Intents, Axelar, and CCTP on Stellar; technical discovery for Stellar Broker
- **Exit criteria:** a user can connect Freighter or Lobstr and complete an XLM ↔ USDC swap via SDEX path payment on testnet, with trustline handling working end to end

### Milestone 2 (Weeks 7–12): Aggregation and simulation, mainnet beta

- Soroswap and Aquarius adapters live alongside SDEX
- Simulation layer live for all route types; route ranking across DEX sources
- Two cross-chain protocols in production (one bridge-based, one intent-based), per Section 4.2
- **Exit criteria:** mainnet beta with the four launch assets; aggregated quotes returned from at least three Stellar liquidity sources; simulated-then-executed swaps succeeding on mainnet

### Milestone 3 (Weeks 13–16): Launch and hardening

- General availability of Stellar routes in the production app
- Additional integrations as verified in Milestone 1 (Stellar Broker; Axelar and/or CCTP when available)
- Performance tuning (quote latency, success rate), failure-path stress testing, security review
- **Exit criteria:** Stellar routes in GA; failure-rate instrumentation live per Section 6.3; security review findings resolved or accepted

Items marked contingent (Stellar Broker, Axelar, CCTP) are stretch scope: their absence does not block GA, and grant deliverables are the committed items above.

---

## 9. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| A named protocol lacks (or delays) Stellar support — notably CCTP | Cross-chain scope narrows | Availability verified in Milestone 1 before adapter work is scheduled; committed scope (Section 4.2) does not depend on contingent protocols |
| Soroban RPC reliability / rate limits under production quote load | Degraded quoting or simulation | Provider redundancy and caching at the network layer; SDEX routes (Horizon-only) remain available independently |
| Thin liquidity on Soroban AMMs for launch assets | Poor pricing vs. expectations | SDEX-first sequencing; routes are ranked on simulated output, so thin sources naturally lose ranking rather than harming users |
| Router contract changes or network resets invalidate pinned addresses | Broken adapter until updated | Contract IDs held as versioned config with monitoring, not hardcoded (Section 7.4) |
| Cross-chain partial failure (source leg succeeds, destination fails) | Stuck user funds | Defined retry/refund reconciliation path (Section 4.3), reused from existing production bridge infrastructure |
| Timeline slip in Soroban adapter work | Milestone 2 delay | SDEX-only aggregation is a shippable fallback; milestone exit criteria gate scope, not the reverse |

---

## 10. Assumptions and External Dependencies

- Public Horizon and Soroban RPC infrastructure (or commercial providers) remains available for testnet and mainnet.
- Aquarius and Soroswap router contracts remain deployed and documented on mainnet.
- Milestone durations are effort-based from project start with a dedicated integration team; calendar dates shift with award timing.
- Protocol availability claims in Section 4.2 are re-verified at Milestone 1 — this document intentionally avoids committing to third-party roadmaps.

## 11. Out of Scope

- Changes to Lunar's fiat on/off-ramp
- Market making or liquidity provisioning on Stellar venues
- Custodial key management of any kind
- Wallets outside the Stellar Wallets Kit ecosystem at launch

---

## 12. Team and Delivery Track Record

- Production aggregation infrastructure spanning 40+ DeFi protocol integrations (DEX aggregators, bridges, and intent protocols) across EVM chains, Solana, and others.
- The adapter architecture described here is the one running in production today — Stellar adds adapters to a proven system rather than standing up a new one.
- The team covers backend, frontend, blockchain, and infrastructure engineering, with prior delivery of cross-chain swap and bridge products to retail users; the simulation-first design in Section 6 comes directly from that operating experience.

---

*This document is the technical companion to our Stellar Community Fund Application (Integration Track).*
