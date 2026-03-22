# DARKBOARD

**Decentralized Privacy-Preserving OTC Dark Pool on Stellar Soroban**

## 1. Project Overview
DARKBOARD is a decentralized, privacy-preserving Over-The-Counter (OTC) dark pool trading protocol natively engineered for the Stellar Soroban smart contract ecosystem. It serves institutional traders, decentralized treasury managers, Decentralized Autonomous Organizations (DAOs), and high-net-worth individuals (HNWIs) who require the ability to execute massive block trades without suffering from slippage or signaling their intentions to the broader public market. 

On traditional Automated Market Makers (AMMs) like Soroswap or public limit order books, executing a million-dollar swap instantly alerts MEV (Maximal Extractable Value) searchers, arbitrageurs, and front-runners, resulting in predatory price manipulation before the trade even settles. DARKBOARD solves this critical market inefficiency by utilizing a sealed-bid, off-chain matching engine coupled with a trustless on-chain cryptographic settlement vault. Traders deposit assets into a non-custodial Soroban vault and sign cryptographically sealed intent orders. The off-chain engine matches these intents in total privacy. When a perfect match is found, the engine submits the cryptographically linked signatures to the `darkboard_engine` smart contract, which verifies the mathematical proofs and atomically swaps the assets between the two parties in a single ledger close. No market impact occurs, no order book is broadcasted, and exact trade details are only revealed on-chain after the execution is finalized.

## 2. System Design Principles
1. **Zero-Knowledge Intent Secrecy:** The primary directive of the protocol is absolute pre-trade privacy. Order sizes, target prices, and asset pairs must never be published to the public ledger prior to execution. Intents are signed locally by the user's wallet and submitted directly to the encrypted off-chain matching engine via secure WebSockets.
2. **Non-Custodial Escrow:** The off-chain matching engine has zero access to user funds. It only possesses the authority to submit matching, mathematically paired signatures to the Soroban smart contract. If the matching engine is compromised or goes offline, user funds remain safely locked in the `darkboard_vault` contract and can be unilaterally withdrawn by the depositor after the lock-up expiration ledger.
3. **Atomic Batch Settlement:** To eliminate counterparty risk, the settlement of paired trades must be mathematically atomic. The Soroban contract must guarantee that Party A receives Asset Y if and only if Party B receives Asset X. If any condition fails (e.g., insufficient balance, expired signature, or mismatched price parameters), the entire transaction strictly reverts.
4. **Deterministic Fee Routing:** Protocol sustainability is guaranteed through deterministic fee extraction executed precisely at the moment of atomic settlement. The contract calculates the basis point fee directly from the traded output amounts and routes it to the protocol treasury before releasing the remaining funds to the traders.

## 3. Technology Stack
1. **Smart Contract Language:** Rust version 1.76.0. Selected for its uncompromising memory safety, strict zero-cost abstractions, and highly optimized compilation to WebAssembly (WASM), which is mandatory for Soroban execution.
2. **Smart Contract Framework:** Soroban SDK version 20.0.0. Selected because it provides the official, actively maintained Stellar host environment bindings required to access ledger state, verify cryptographic signatures, and execute cross-contract token transfers.
3. **Frontend Framework:** Next.js version 14.1.0 using the App Router architecture. Selected for its ability to deliver optimized, server-rendered React applications with strict route-level code splitting, ensuring the trading terminal loads in milliseconds.
4. **Frontend Language:** TypeScript version 5.4.2. Selected to enforce rigid compile-time type checking across the entire application stack, specifically when bridging the boundary between the React frontend and the generated XDR transaction payloads.
5. **Wallet Integration:** Freighter API version 2.0.0. Selected as the premier, heavily audited browser extension wallet for interacting with Stellar Protocol 20+ features and signing Soroban authorization payloads.
6. **Matching Engine Environment:** Rust version 1.76.0 with Tokio version 1.36.0. Selected for building a hyper-performant, multi-threaded off-chain matching engine capable of parsing thousands of WebSocket intent messages per second without garbage collection pauses.
7. **Database Architecture:** PostgreSQL version 16 configured with TimescaleDB extensions. Selected for its robust relational data integrity and highly optimized time-series querying capabilities, which are essential for storing historical settlement data.
8. **ORM Layer:** Prisma version 5.10.0. Selected to provide a strictly typed database client for the administrative API endpoints.
9. **Infrastructure and Hosting:** AWS Elastic Container Service (ECS) for deploying the Rust matching engine, Vercel for the Next.js frontend delivery, and Supabase for managed, highly available PostgreSQL database hosting.

## 4. Smart Contract Architecture

### Contract 1: `darkboard_vault.rs`
**Responsibility:** The secure, non-custodial escrow layer. It accepts token deposits from users, locks them under a specific intent hash, and strictly governs the withdrawal logic if an order is cancelled or expires.
**Public Functions:**
1. `deposit_assets(env: Env, user: Address, token: Address, amount: i128, intent_hash: BytesN<32>, unlock_ledger: u32) -> Result<(), VaultError>`
2. `withdraw_assets(env: Env, user: Address, token: Address, amount: i128, intent_hash: BytesN<32>) -> Result<(), VaultError>`
3. `get_locked_balance(env: Env, user: Address, token: Address) -> i128`

**Storage Keys:**
1. Key: `VaultDeposit(Address, BytesN<32>)` (User Address, Intent Hash) | Type: Persistent | Value: `DepositRecord` | TTL Strategy: Maintained indefinitely until explicitly withdrawn or settled by the engine.

**Error Types:**
1. `VaultError::InsufficientUserBalance`
2. `VaultError::IntentHashAlreadyExists`
3. `VaultError::UnlockLedgerNotReached`
4. `VaultError::DepositRecordNotFound`

**Events Emitted:**
1. `AssetsDepositedEvent { user: Address, token: Address, amount: i128, intent_hash: BytesN<32>, unlock_ledger: u32 }`
2. `AssetsWithdrawnEvent { user: Address, token: Address, amount: i128, intent_hash: BytesN<32> }`

### Contract 2: `darkboard_engine.rs`
**Responsibility:** The execution and settlement layer. It verifies the cryptographic signatures of both matched parties, verifies that the asset amounts satisfy the original signed intent parameters, and executes the cross-transfer by invoking the vault contract.
**Public Functions:**
1. `settle_trade(env: Env, maker_intent: OrderIntent, taker_intent: OrderIntent, maker_sig: BytesN<64>, taker_sig: BytesN<64>) -> Result<BytesN<32>, EngineError>`
2. `verify_intent_signature(env: Env, intent: OrderIntent, signature: BytesN<64>) -> Result<(), EngineError>`
3. `cancel_intent(env: Env, user: Address, intent_hash: BytesN<32>) -> Result<(), EngineError>`

**Storage Keys:**
1. Key: `SettledIntent(BytesN<32>)` | Type: Persistent | Value: `u32` (Settlement Ledger) | TTL Strategy: Maintained indefinitely to act as an immutable double-spend prevention mechanism.
2. Key: `CancelledIntent(BytesN<32>)` | Type: Persistent | Value: `bool` | TTL Strategy: Maintained indefinitely.

**Error Types:**
1. `EngineError::InvalidSignature`
2. `EngineError::IntentAlreadySettled`
3. `EngineError::IntentCancelled`
4. `EngineError::PriceMismatch`
5. `EngineError::UnauthorizedEngineSubmission`

**Events Emitted:**
1. `TradeSettledEvent { maker: Address, taker: Address, token_a: Address, token_b: Address, amount_a: i128, amount_b: i128, settlement_hash: BytesN<32> }`
2. `IntentCancelledEvent { user: Address, intent_hash: BytesN<32> }`

### Contract 3: `darkboard_fee.rs`
**Responsibility:** Calculates and accumulates protocol trading fees generated by the `darkboard_engine` during settlement.
**Public Functions:**
1. `calculate_fee(env: Env, amount: i128) -> Result<i128, FeeError>`
2. `collect_fee(env: Env, from: Address, token: Address, amount: i128) -> Result<(), FeeError>`
3. `update_fee_tier(env: Env, admin: Address, basis_points: u32) -> Result<(), FeeError>`
4. `withdraw_fees(env: Env, admin: Address, token: Address, destination: Address, amount: i128) -> Result<(), FeeError>`

**Storage Keys:**
1. Key: `FeeConfiguration` | Type: Instance | Value: `u32` (Basis Points) | TTL Strategy: Bumped on every trade execution.
2. Key: `ProtocolAdmin` | Type: Instance | Value: `Address` | TTL Strategy: Bumped on every admin interaction.

**Error Types:**
1. `FeeError::UnauthorizedAdmin`
2. `FeeError::InvalidBasisPoints`
3. `FeeError::InsufficientFeeReserves`

**Events Emitted:**
1. `FeeCollectedEvent { token: Address, amount: i128, settlement_hash: BytesN<32> }`

## 5. Data Flow Diagrams

**Action 1: Submitting a Dark Pool Intent**
1. The user connects their Freighter wallet to the Next.js trading terminal.
2. The user selects the asset they wish to sell (e.g., 500,000 USDC) and the asset they wish to buy (e.g., 5,000,000 XLM).
3. The Next.js frontend constructs a JSON `OrderIntent` object defining the specific token addresses, exact amounts, expiry ledger, and a randomly generated cryptographic nonce.
4. The `@darkboard/sdk` serializes the `OrderIntent` into a Soroban XDR byte payload.
5. The frontend prompts the Freighter wallet to cryptographically sign the XDR byte payload using the user's private key.
6. The user submits a transaction to `darkboard_vault.rs` calling `deposit_assets` to lock their 500,000 USDC securely on-chain.
7. Upon successful deposit confirmation, the frontend transmits the signed `OrderIntent` JSON and the transaction receipt to the off-chain Rust Matching Engine via an encrypted WebSocket connection.
8. The Matching Engine validates the signature, verifies the on-chain vault deposit, and silently adds the intent to the encrypted memory pool.

**Action 2: Off-Chain Matching and On-Chain Settlement**
1. Another institutional trader connects to the terminal and submits a reverse intent (selling XLM to buy USDC) following the exact steps in Action 1.
2. The off-chain Rust Matching Engine constantly iterates over the memory pool comparing token pairs and price ratios.
3. The Engine detects a perfect mathematical intersection between the Maker's intent and the Taker's intent.
4. The Engine constructs a multi-parameter Soroban transaction invoking the `settle_trade` function on `darkboard_engine.rs`.
5. The Engine injects the Maker's original signature, the Taker's original signature, and both intent payloads into the transaction.
6. The Engine signs the transaction with the designated protocol execution key and submits it to the Stellar Horizon RPC.
7. The `darkboard_engine.rs` contract mathematically verifies both user signatures natively on-chain.
8. The contract queries `darkboard_vault.rs` to ensure both deposits are securely locked and the intents have not expired or been cancelled.
9. The contract calls `darkboard_fee.rs` to deduct the protocol basis points from the final output amounts.
10. The contract instructs `darkboard_vault.rs` to unlock the assets and cross-transfer them to the respective recipient addresses.
11. The `SettledIntent` keys are written to persistent storage to guarantee the signatures can never be replayed.
12. The `TradeSettledEvent` is emitted to the network, and the off-chain engine updates the clients via WebSocket.

**Action 3: Intent Cancellation and Asset Retrieval**
1. A user decides they no longer want to execute their trade.
2. The user submits a transaction to `darkboard_engine.rs` calling `cancel_intent` using the exact intent hash.
3. The contract writes the intent hash to the `CancelledIntent` persistent storage map.
4. The user submits a subsequent transaction to `darkboard_vault.rs` calling `withdraw_assets`.
5. The vault contract queries the engine contract; observing the intent is permanently cancelled, it releases the locked funds back to the user's wallet.
6. The off-chain matching engine detects the on-chain cancellation event and aggressively purges the intent from its active memory pool.

## 6. SDK Architecture

### TypeScript SDK: `@darkboard/sdk`
**Class:** `DarkboardClient`
**Exported Functions:**
1. `constructor(network: string, rpcUrl: string, vaultAddress: string, engineAddress: string) -> DarkboardClient`
2. `buildOrderIntent(makerAddress: string, sellToken: string, buyToken: string, sellAmount: string, buyAmount: string, expiryLedger: number, nonce: string) -> OrderIntent`
3. `serializeIntentToXDR(intent: OrderIntent) -> Buffer`
4. `signIntentPayload(xdrPayload: Buffer, walletConnector: any) -> Promise<string>`
5. `buildDepositTransaction(intent: OrderIntent) -> Promise<Transaction>`
6. `buildCancelTransaction(intentHash: string) -> Promise<Transaction>`
7. `buildWithdrawTransaction(intent: OrderIntent) -> Promise<Transaction>`
8. `connectToMatchingEngine(webSocketUrl: string, authKey: string) -> WebSocketConnection`

### Python SDK: `darkboard-py`
**Class:** `DarkboardClient`
**Exported Functions:**
1. `__init__(self, network: str, rpc_url: str, vault_address: str, engine_address: str) -> None`
2. `build_order_intent(self, maker_address: str, sell_token: str, buy_token: str, sell_amount: str, buy_amount: str, expiry_ledger: int, nonce: str) -> Dict`
3. `serialize_intent_to_xdr(self, intent: Dict) -> bytes`
4. `sign_intent_payload(self, xdr_payload: bytes, secret_key: str) -> str`
5. `build_deposit_transaction(self, intent: Dict) -> Dict`
6. `build_cancel_transaction(self, intent_hash: str) -> Dict`
7. `submit_intent_to_engine(self, intent: Dict, signature: str, engine_url: str) -> Dict`

## 7. Frontend Architecture
1. **Page: `/` (Landing Page):** Presents the institutional value proposition of DARKBOARD, highlighting cryptographic privacy and zero slippage guarantees.
2. **Page: `/trade` (Trading Terminal):** The core application interface. Features a minimalist, high-contrast dark mode UI where users input token parameters without seeing a public order book.
3. **Page: `/portfolio` (Active Orders and Balances):** Displays the user's currently active intents, locked vault balances, and historical executed trades fetched directly from the indexing API.
4. **Component: `TokenPairSelector.tsx`:** A highly optimized modal utilizing fuzzy search to allow users to select verified Stellar ecosystem assets.
5. **Component: `IntentSignerModal.tsx`:** A secure overlay that guides the user through the two-step process: signing the off-chain intent payload, and subsequently signing the on-chain vault deposit transaction.
6. **Component: `WebSocketStatusIndicator.tsx`:** A real-time connection status light (Green/Yellow/Red) ensuring the user knows their browser is actively communicating with the matching engine.
7. **Hook: `useDarkboardEngine.ts`:** Manages the WebSocket lifecycle, handles heartbeat pings, and dispatches incoming settlement notifications to the React global state.
8. **Hook: `useVaultBalances.ts`:** Continuously polls the Horizon RPC to monitor the user's locked assets in the `darkboard_vault` contract.
9. **Context: `StellarNetworkContext.tsx`:** Maintains global application state regarding the active Horizon RPC node, network passphrase, and contract identifier configurations.

## 8. Off-Chain Infrastructure
1. **Service: `darkboard-matcher` (Rust):** The core off-chain matching engine. Written entirely in Rust using the Tokio asynchronous runtime. It maintains a highly optimized Red-Black tree in memory to instantly detect overlapping price constraints across thousands of active intents.
2. **Service: `darkboard-indexer` (Node.js):** A background worker script that continuously polls the Stellar Horizon API for `TradeSettledEvent` and `IntentCancelledEvent`. It parses the Soroban XDR responses and normalizes them into JSON objects.
3. **Service: `darkboard-api` (Express.js):** A RESTful API interface that serves historical trade volume statistics and user portfolio histories from the database, preventing the Next.js frontend from having to execute slow, client-side Horizon RPC pagination.
4. **Database Configuration:** A PostgreSQL database managed by Prisma. The schema includes strict relational models for `HistoricalTrade`, `ActiveVaultDeposit`, and `ProtocolFeeRevenue`.
5. **Operational Specs:** The Rust matching engine is containerized via Docker and deployed to a highly isolated AWS EC2 instance. The Node.js API and Indexer are deployed to AWS Elastic Container Service (ECS) with auto-scaling policies enabled.

## 9. Security Model
1. **Threat Vector: Intent Replay Attacks.** An attacker intercepts a signed `OrderIntent` and attempts to submit it to the matching engine multiple times to drain the user's vault balance.
   **Mitigation:** The `darkboard_engine.rs` contract maintains a persistent `SettledIntent` storage map. During `settle_trade`, the contract computes the SHA-256 hash of the intent payload. If that hash already exists in the storage map, the transaction instantly reverts with `EngineError::IntentAlreadySettled`.
2. **Threat Vector: Engine Front-Running.** A compromised matching engine operator attempts to view incoming intents and execute their own trades ahead of the users to capture the spread.
   **Mitigation:** Because trades are executed at the exact price ratio explicitly signed by the user in the `OrderIntent`, the engine cannot inject slippage or alter the execution price. The engine operator can only refuse to match an order (censorship), but they cannot steal value through price manipulation. Users can unilaterally call `cancel_intent` on-chain at any time to bypass a censoring engine.
3. **Threat Vector: Vault Drain via Malicious Engine.** The matching engine submits forged pairs to `settle_trade` to route vault assets to an attacker's address.
   **Mitigation:** The `darkboard_engine.rs` contract natively executes `env.crypto().ed25519_verify()` on the provided signatures. It forces a strict mathematical requirement that the output addresses receiving the assets mathematically match the public keys that signed the intents. The engine cannot redirect funds.
4. **Threat Vector: Signature Expiration Exploitation.** A matching engine holds onto a valid intent for months and executes it when market conditions drastically change against the user.
   **Mitigation:** Every `OrderIntent` contains a rigid `expiry_ledger` field. During `settle_trade`, the Soroban contract asserts `env.ledger().sequence() <= intent.expiry_ledger`. If the current network ledger surpasses the signed limit, the execution reverts entirely.

## 10. Integration Points
1. **Stellar Asset Contract (SAC):** DARKBOARD natively interacts with all SAC-wrapped tokens (e.g., native XLM, USDC) using the standard `token::Client` interface. The `darkboard_vault.rs` contract executes authorized `transfer` and `transfer_from` operations directly through these native token interfaces.
2. **Stellar Horizon RPC:** The Next.js frontend, the TypeScript SDK, and the Node.js indexer rely entirely on standard Horizon RPC endpoints for transaction submission, simulation, and event ingestion.
3. **WebSockets (wss://):** The primary communication vector between the client browser and the Rust matching engine, ensuring sub-second latency for intent submission and execution notification.

## 11. Testing Strategy
1. **Contract Unit Tests (`test_engine.rs`, `test_vault.rs`):** Executes isolated tests on individual contract functions using the native `soroban_sdk::testutils` framework. Exhaustively tests signature failures, exact ledger expiration boundaries, and precision fee arithmetic.
2. **Contract Integration Tests (`test_settlement_flow.rs`):** Deploys mock token contracts, mints balances, deploys the vault, deploys the engine, and simulates a full end-to-end atomic cross-transfer between two mocked trading accounts, asserting exact final balance changes.
3. **Engine Benchmarks (`benches/matching_engine.rs`):** Utilizes the Rust `criterion` framework to load-test the off-chain matching engine memory pool, ensuring it can successfully detect overlapping price constraints across 100,000 concurrent intents in under 50 milliseconds.
4. **SDK Unit Tests (`client.test.ts`):** Utilizes the Jest framework to assert that the TypeScript SDK serializes the `OrderIntent` JSON into the exact XDR byte structure expected by the Rust smart contract verification logic.
5. **End-to-End Tests (Playwright):** Simulates two distinct browser sessions. Browser A deposits and signs a sell intent. Browser B deposits and signs a buy intent. Asserts that the matching engine detects the pair, submits the transaction, and updates both UIs simultaneously to reflect the successful settlement.

## 12. Deployment Architecture
1. **Testnet Deployment Sequence:**
   1. Compile all Soroban WASM binaries using `soroban contract build --profile release`.
   2. Execute `soroban contract optimize` to strip symbols and reduce deployment footprint.
   3. Deploy `darkboard_fee.wasm` to Stellar Testnet; capture the Contract ID and initialize with Testnet Admin keys.
   4. Deploy `darkboard_vault.wasm` to Stellar Testnet.
   5. Deploy `darkboard_engine.wasm` to Stellar Testnet; initialize it by linking the `darkboard_vault` and `darkboard_fee` Contract IDs.
   6. Authorize the `darkboard_engine` contract address within the `darkboard_vault` to permit cross-contract asset transfers.
   7. Deploy the Rust matching engine to an AWS EC2 instance configured for the Testnet environment.
   8. Deploy the Next.js frontend to Vercel, pointing environment variables to the Testnet RPC and Engine WebSocket URL.
2. **Mainnet Deployment Sequence:**
   1. Re-compile and re-optimize all binaries for the Mainnet environment.
   2. Execute deployments using offline, hardware-wallet secured multi-signature accounts for the Protocol Admin roles.
   3. Initialize all linkages precisely as outlined in the Testnet sequence.
   4. Configure strictly firewalled, production-grade AWS ECS instances for the matching engine to ensure maximum uptime and DDoS protection.
3. **Post-Deploy Verification:** Execute the `scripts/verify_deployment.sh` script to simulate a microscopic test trade on Mainnet to guarantee that the vault authorization logic and engine settlement logic execute flawlessly.

## 13. Upgrade and Governance Strategy
1. The `darkboard_vault`, `darkboard_engine`, and `darkboard_fee` contracts fully implement the official Soroban WASM upgrade interface.
2. The `upgrade` function strictly requires authorization from the Protocol Governance Multisig via `env.auth()`.
3. The Governance Multisig is configured as a 4-of-7 multi-signature account comprising core developers, institutional liquidity providers, and independent security auditors.
4. Any proposed WASM upgrade must be compiled reproducibly, audited by a third-party firm, deployed to Testnet, and publicly disclosed to the trader community with a 14-day timelock before the multisig signs the Mainnet execution transaction.

---

## PART 2 — FULL PRODUCTION PROGRESS TRACKER

### Phase 0 — Repository & Environment Setup
- [ ] Initialize Git repository named `darkboard-monorepo`.
- [ ] Configure Turborepo for monorepo management across contracts, frontend, API, and matching engine.
- [ ] Create `contracts`, `sdk`, `api`, `app`, and `engine` subdirectories.
- [ ] Initialize a new Rust workspace in the `contracts` directory.
- [ ] Configure `Cargo.toml` with workspace members `darkboard-vault`, `darkboard-engine`, and `darkboard-fee`.
- [ ] Initialize a standalone Rust binary project in the `engine` directory for the matching server.
- [ ] Install Rust toolchain version 1.76.0.
- [ ] Install `soroban-cli` version 20.0.0 globally.
- [ ] Add `wasm32-unknown-unknown` target to the Rust toolchain.
- [ ] Set up GitHub Actions workflow file `.github/workflows/test-contracts.yml` to run `cargo test`.
- [ ] Set up GitHub Actions workflow file `.github/workflows/test-engine.yml` to run matching engine benchmarks.

### Phase 1 — Smart Contract Development
- [ ] Write the `VaultError` enum definition in `darkboard-vault/src/errors.rs`.
- [ ] Implement the `deposit_assets(env: Env, user: Address, token: Address, amount: i128, intent_hash: BytesN<32>, unlock_ledger: u32)` function in `darkboard-vault/src/lib.rs`.
- [ ] Implement the `withdraw_assets(env: Env, user: Address, token: Address, amount: i128, intent_hash: BytesN<32>)` function in `darkboard-vault/src/lib.rs`.
- [ ] Write the `OrderIntent` struct and `EngineError` enum in `darkboard-engine/src/types.rs`.
- [ ] Implement the `verify_intent_signature(env: Env, intent: OrderIntent, signature: BytesN<64>)` cryptographic function in `darkboard-engine/src/verify.rs`.
- [ ] Implement the `settle_trade(env: Env, maker_intent: OrderIntent, taker_intent: OrderIntent, maker_sig: BytesN<64>, taker_sig: BytesN<64>)` atomic transfer function in `darkboard-engine/src/lib.rs`.
- [ ] Implement the `cancel_intent(env: Env, user: Address, intent_hash: BytesN<32>)` logic in `darkboard-engine/src/lib.rs`.
- [ ] Write the `FeeError` enum in `darkboard-fee/src/errors.rs`.
- [ ] Implement the `calculate_fee(env: Env, amount: i128)` and `collect_fee` logic in `darkboard-fee/src/lib.rs`.

### Phase 2 — Contract Testing
- [ ] Write unit test `test_deposit_fails_with_insufficient_allowance` in `darkboard-vault/tests/test_vault.rs`.
- [ ] Write unit test `test_withdraw_fails_before_unlock_ledger` in `darkboard-vault/tests/test_vault.rs`.
- [ ] Write unit test `test_verify_signature_rejects_altered_intent_payload` in `darkboard-engine/tests/test_verify.rs`.
- [ ] Write unit test `test_settle_trade_reverts_on_price_mismatch` in `darkboard-engine/tests/test_engine.rs`.
- [ ] Write unit test `test_settle_trade_reverts_on_expired_ledger` in `darkboard-engine/tests/test_engine.rs`.
- [ ] Write unit test `test_settle_trade_prevents_signature_replay` in `darkboard-engine/tests/test_engine.rs`.
- [ ] Write unit test `test_fee_calculation_handles_rounding_precision` in `darkboard-fee/tests/test_fee.rs`.
- [ ] Write integration test `test_end_to_end_atomic_swap_flow` spanning all three contracts in `darkboard-engine/tests/test_integration.rs`.

### Phase 3 — SDK Development (TypeScript)
- [ ] Initialize a new Node.js package in the `sdk/typescript` directory.
- [ ] Install dependencies: `@stellar/stellar-sdk`.
- [ ] Configure `tsconfig.json` for strict type checking and ESModule compilation.
- [ ] Write the `DarkboardClient` class constructor in `sdk/typescript/src/client.ts`.
- [ ] Write the `buildOrderIntent` method to normalize user inputs into the structured JSON format.
- [ ] Write the `serializeIntentToXDR` method utilizing the Soroban XDR definitions.
- [ ] Write the `signIntentPayload` method to interface with Freighter wallet signatures.
- [ ] Write the `buildDepositTransaction` method to construct the Soroban invocation payload.
- [ ] Write the `connectToMatchingEngine` WebSocket handler utility.
- [ ] Set up Jest testing framework in the SDK directory.
- [ ] Write Jest test `test_buildOrderIntent_formats_data_correctly`.
- [ ] Write Jest test `test_serializeIntentToXDR_matches_rust_struct_layout`.

### Phase 4 — SDK Development (Python)
- [ ] Initialize a new Poetry project in the `sdk/python` directory.
- [ ] Install dependency `stellar-sdk` and `websockets` for Python.
- [ ] Write the `DarkboardClient` class constructor in `sdk/python/darkboard/client.py`.
- [ ] Write the `build_order_intent` method.
- [ ] Write the `serialize_intent_to_xdr` method.
- [ ] Write the `sign_intent_payload` method.
- [ ] Write the `submit_intent_to_engine` asynchronous WebSocket method.
- [ ] Set up `pytest` configuration.
- [ ] Write `pytest` test asserting XDR byte serialization accuracy.

### Phase 5 — Matching Engine Development (Rust)
- [ ] Configure `engine/Cargo.toml` with `tokio`, `warp` (for WebSockets), and `soroban-sdk` dependencies.
- [ ] Implement the `WebSocketServer` struct to accept and authenticate incoming client connections.
- [ ] Implement the `MemoryPool` struct utilizing an optimized Red-Black tree for $O(\log n)$ order insertion.
- [ ] Write the `OrderMatcher` core loop to detect overlapping `buy_amount` and `sell_amount` ratios.
- [ ] Write the `TransactionSubmitter` module to dynamically build the `settle_trade` XDR payload.
- [ ] Write the `submit_to_horizon` asynchronous function to broadcast the settlement transaction to the Stellar network.
- [ ] Write unit test `test_matcher_correctly_pairs_perfect_overlapping_orders`.
- [ ] Write unit test `test_matcher_ignores_orders_with_mismatched_assets`.

### Phase 6 — Frontend Development (Next.js)
- [ ] Initialize Next.js 14 App Router project in the `app` directory.
- [ ] Configure `tsconfig.json` to enforce `.tsx` and `.ts` strictness.
- [ ] Install Tailwind CSS, Framer Motion, and `@stellar/freighter-api`.
- [ ] Build the layout structure and global navigation bar in `app/src/app/layout.tsx`.
- [ ] Build the institutional landing page in `app/src/app/page.tsx`.
- [ ] Build the `app/src/app/trade/page.tsx` view for the core terminal interface.
- [ ] Build the `TokenPairSelector.tsx` modal component.
- [ ] Build the `IntentSignerModal.tsx` step-by-step cryptographic signature UI.
- [ ] Build the `WebSocketStatusIndicator.tsx` persistent layout component.
- [ ] Build the `app/src/app/portfolio/page.tsx` view.
- [ ] Implement the `useDarkboardEngine.ts` hook to manage the WebSocket state machine.
- [ ] Implement the `useVaultBalances.ts` hook to read on-chain Soroban storage values.

### Phase 7 — Off-Chain API & Indexer Infrastructure
- [ ] Initialize an Express.js project in the `api` directory.
- [ ] Initialize Supabase project and obtain PostgreSQL connection strings.
- [ ] Initialize Prisma ORM with `npx prisma init`.
- [ ] Write Prisma schema in `api/prisma/schema.prisma` defining `HistoricalTrade` and `ActiveVaultDeposit` models.
- [ ] Run `npx prisma db push` to synchronize the database schema.
- [ ] Write the `indexer.ts` background script to aggressively poll the Horizon RPC for `TradeSettledEvent` notifications.
- [ ] Write Prisma insertion logic inside `indexer.ts` to log executed trades.
- [ ] Write Express route `GET /api/portfolio/:address` to fetch user trade history.
- [ ] Write Express route `GET /api/statistics/volume` to aggregate global protocol volume metrics.
- [ ] Create robust `Dockerfile` configurations for both the API and the Indexer services.

### Phase 8 — Documentation
- [ ] Write `README.md` at the monorepo root explaining project structure and startup commands.
- [ ] Write `contracts/README.md` detailing the Soroban architecture and XDR serialization requirements.
- [ ] Write `engine/README.md` explaining the WebSocket payload schema and matching algorithms.
- [ ] Write `sdk/typescript/README.md` containing installation instructions and intent construction examples.
- [ ] Create the `docs/architecture.md` file pasting this entire architecture specification.
- [ ] Create the `docs/api-reference.md` file documenting the Express API endpoints.
- [ ] Write comprehensive inline JSDoc and Rustdoc comments across all exported functions.

### Phase 9 — Security & Audit Preparation
- [ ] Run `cargo clippy --all-targets` and rigorously resolve all warnings in the Rust contracts and matching engine.
- [ ] Run `npm run lint` across the Next.js and API codebases and resolve all warnings.
- [ ] Document the precise mathematical layout of the signature verification sequence in `docs/security.md`.
- [ ] Write an explicit threat model detailing how malicious matching engines are neutralized by Soroban cryptography.
- [ ] Compile frozen, reproducible versions of the WASM binaries and generate SHA-256 checksums for the external audit package.

### Phase 10 — Testnet Deployment & QA
- [ ] Fund a Stellar Testnet administrative account using the Friendbot faucet.
- [ ] Deploy `darkboard_fee.wasm` to Testnet and initialize.
- [ ] Deploy `darkboard_vault.wasm` to Testnet.
- [ ] Deploy `darkboard_engine.wasm` to Testnet, link it to the vault, and establish cross-contract authorization.
- [ ] Provision an AWS EC2 instance, load the `darkboard-matcher` Docker image, and start the WebSocket server.
- [ ] Update `.env.testnet` files in the frontend and API directories with the new Testnet contract IDs and WebSocket IPs.
- [ ] Deploy the Next.js frontend to a Vercel staging environment.
- [ ] Deploy the API and Indexer to a staging AWS ECS cluster.
- [ ] Perform a rigorous manual end-to-end test on the staging URL utilizing two distinct browsers to verify off-chain matching and on-chain settlement.
- [ ] Remediate any bugs discovered during manual QA testing.

### Phase 11 — Mainnet Deployment
- [ ] Generate strict offline hardware wallet keys for the Mainnet Governance Multisig.
- [ ] Configure the multisig account parameters on Stellar Mainnet.
- [ ] Deploy `darkboard_fee.wasm` and `darkboard_vault.wasm` to Mainnet.
- [ ] Deploy `darkboard_engine.wasm` to Mainnet and execute the initialization linkages.
- [ ] Update production environment variables across Vercel, AWS ECS, and the EC2 matching engine instance to point to Mainnet Horizon endpoints.
- [ ] Push the final production release to the master branch to trigger live deployments.
- [ ] Execute a live, micro-transaction test trade on Mainnet to cryptographically guarantee that the matching engine can successfully execute the `settle_trade` invocation.

### Phase 12 — Wave Program Onboarding
- [ ] Configure the open-source repository tags and descriptions on GitHub.
- [ ] Apply to the Stellar Drips Wave Program platform designating the repository as a priority maintainer ecosystem.
- [ ] Populate the `.github/ISSUE_TEMPLATE/contributor_task.md` file to enforce standardized Pull Request submissions.
- [ ] Create exactly 15 highly specific "Good First Issue" tickets targeting the upcoming short, one-week sprint.
- [ ] Tag the issues with the appropriate Wave Program labels, ensuring a balanced distribution of Rust contract tasks and Next.js frontend tasks.
- [ ] Write a `CONTRIBUTING.md` guide specifically tailored to the "Fix, Merge, and Earn" rhythm, detailing how contributors should mock the WebSocket connection during local frontend development.

### Phase 13 — Post-Launch Maintenance Setup
- [ ] Set up Datadog error tracking in the Next.js frontend to monitor WebSocket disconnection spikes.
- [ ] Set up Prometheus and Grafana dashboards for the Rust matching engine to monitor memory pool size and matching latency.
- [ ] Configure PagerDuty critical alerts for any Supabase database connection failures or Horizon RPC timeouts.
- [ ] Establish a strictly scheduled calendar review for the Protocol Admin to execute `withdraw_fees` and route treasury funds.