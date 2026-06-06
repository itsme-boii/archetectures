# emei-facilitator

Payment orchestration engine for autonomous AI agents on Base.

This service is the operational backbone of the EMEI protocol. It coordinates wallet creation, mandate enforcement, gas-sponsored transaction submission, receipt anchoring, and balance rebalancing — without holding any private keys.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         EMEI FACILITATOR                                 │
│                                                                         │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │  HTTP    │  │  Background  │  │  Privy Client  │  │  Chain RPC   │ │
│  │  API     │  │  Services    │  │  (signing,     │  │  (read-only  │ │
│  │  (Axum)  │  │  (9 workers) │  │   wallets,     │  │   eth_call,  │ │
│  │          │  │              │  │   policies)    │  │   getLogs)   │ │
│  └────┬─────┘  └──────┬───────┘  └───────┬────────┘  └──────┬───────┘ │
│       │                │                  │                   │         │
│       └────────────────┴──────────────────┴───────────────────┘         │
│                                    │                                    │
│                           ┌────────┴────────┐                           │
│                           │   PostgreSQL    │                           │
│                           │   (source of    │                           │
│                           │    truth for    │                           │
│                           │    off-chain    │                           │
│                           │    state)       │                           │
│                           └─────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
             ┌───────────┐  ┌────────────┐  ┌────────────┐
             │   Privy   │  │    Base    │  │   Redis    │
             │  (HSM/TEE │  │  (on-chain │  │  (cache,   │
             │   keys)   │  │   source   │  │   rate     │
             │           │  │   of truth │  │   limits)  │
             └───────────┘  └────────────┘  └────────────┘
```

---

## Design Principles

**Zero local key material.** Every wallet — agent wallets, hot wallet pool — lives in Privy's HSM infrastructure. The facilitator requests signatures; it never produces them. A compromised facilitator cannot forge transactions.

**On-chain is always the source of truth.** If PostgreSQL is lost, the entire state can be rebuilt from chain events. The local database is a performance cache, not a ledger.

**Fail safe, never fail dangerous.** Every distributed operation (mandate creation = on-chain tx + Privy policy update) is designed so that partial failure results in a more restrictive state (agent can't spend), never a more permissive one (agent spends without authorization).

**Isolation by priority.** The transaction queue is split into three dedicated lanes (receipt, maintenance, collection) with reserved wallet capacity. A flood of low-priority collection jobs can never starve receipt anchoring or overdue marking.

**Race-free concurrency.** The auto-collector uses PostgreSQL's `FOR UPDATE SKIP LOCKED` to atomically claim invoices. Multiple facilitator replicas can run simultaneously without double-processing.

---

## Module Map

```
src/
├── bin/server.rs           Entry point. Connects infra, spawns workers, serves HTTP.
├── lib.rs                  Public API: emei_router() + start_services()
│
├── config.rs               Environment variable loading with strict validation.
├── error.rs                Unified error type → HTTP status codes + JSON bodies.
├── state.rs                Arc<AppState>. Shared across all routes and services.
├── types.rs                Address validation helpers.
├── merkle.rs               Sorted-pair Merkle tree for receipt batching.
├── redis_client.rs         Rate limiting and caching (no nonce management).
│
├── auth/                   Two-tier authentication.
│   ├── extractors.rs       UserAuth (Privy JWT), AgentAuth (session token) — Axum extractors.
│   └── session.rs          Token lifecycle: create (SHA-256 stored), validate, rotate, revoke.
│
├── privy/                  Privy Server Wallet API integration.
│   ├── client.rs           HTTP client with retry, auth headers, structured error mapping.
│   ├── types.rs            Request/response types matching Privy's API contract.
│   └── policies.rs         Translate mandate params → Privy wallet policy format.
│
├── contracts/              Alloy-generated typed bindings for EMEI smart contracts.
│   ├── invoice.rs          EMEIInvoice (ERC-721 NFT invoices, EIP-712 signed creation).
│   ├── mandate.rs          EMEIMandate (recurring resets, per-counterparty limits).
│   ├── settlement.rs       EMEISettlement (dual-tranche vaults, buffer pool, sweep).
│   ├── receipt.rs          EMEIReceipt (3-param postMerkleRoot with metadataHash).
│   └── reputation.rs       EMEIReputation (addPositive/addNegative, scoreOf).
│
├── db/                     Typed PostgreSQL queries, organized by domain.
│   ├── schema.rs           Full DDL. Idempotent (IF NOT EXISTS). Single source of table defs.
│   ├── agents.rs           Agent wallet CRUD + user upsert.
│   ├── invoices.rs         Invoice index ops. The atomic collector claim query lives here.
│   ├── mandates.rs         Outbox pattern: sync intent → chain confirm → privy sync/rollback.
│   ├── tx_queue.rs         Priority queue with SKIP LOCKED claiming, per-queue indexes.
│   ├── events.rs           On-chain event index for statement API and deduplication.
│   └── receipts.rs         Pending receipt hash insert + batch drain.
│
├── routes/                 HTTP route handlers, grouped by auth requirement.
│   ├── agents.rs           POST/GET /emei/agents, rotate token, revoke (UserAuth).
│   ├── mandate.rs          POST/DELETE /emei/mandate — outbox + immediate policy sync (UserAuth).
│   ├── invoice.rs          create, present, pay, collect (AgentAuth).
│   ├── query.rs            balance, reputation, invoice detail, statement (Public).
│   ├── withdraw.rs         POST /emei/withdraw — vault → user wallet (UserAuth).
│   ├── receipt.rs          GET /emei/verify/{id} — Merkle inclusion check (Public).
│   ├── paylink.rs          GET /emei/paylink/{id} — pre-encoded calldata for x402 (Public).
│   ├── health.rs           GET /health — DB, Redis, chain, queue depths.
│   ├── public.rs           /emei/public/stats, /events — dashboard reads (Public).
│   └── webhook.rs          POST /emei/webhook — Alchemy HMAC-verified event ingestion.
│
└── services/               Background workers. Each owns one concern.
    ├── collector.rs         Auto-collect: atomic SQL claim → enqueue collect() to collection queue.
    ├── scanner.rs           Overdue + expiry detection → enqueue markOverdue/markExpired.
    ├── batcher.rs           Receipt hash drain → Merkle tree → enqueue postMerkleRoot.
    ├── sweeper.rs           Detect low spendable balance → enqueue topUpFromYield.
    ├── buffer_monitor.rs    Buffer pool health → adjust max instant withdrawal threshold.
    ├── reconciler.rs        Outbox pattern: retry Privy sync, rollback orphaned policies.
    ├── tx_sender.rs         Per-wallet loop: claim from queue → Privy send_transaction.
    ├── tx_reaper.rs         Reclaim stuck jobs after 120s timeout.
    └── indexer.rs           Poll eth_getLogs → populate invoice_index + mandate_index + events.
```

---

## Security Model

### Trust Boundaries

| Layer | Trusted For | NOT Trusted For |
|-------|------------|-----------------|
| **On-chain contracts** | Settlement math, mandate enforcement, state transitions | — |
| **Privy** | Key custody, policy enforcement (pre-sign), nonce management | — |
| **Facilitator** | Liveness (payments flow, receipts anchor) | Safety (can't steal, can't exceed mandate) |
| **PostgreSQL** | Performance (fast queries) | Correctness (rebuildable from chain) |
| **Redis** | Rate limiting, caching | Any persistent state |

### Two-Guardrail Architecture

Every agent payment passes through two independent authorization checks:

1. **Privy Policy (off-chain, pre-sign).** Before any signature is produced, Privy checks: is the target contract allowed? Is the method allowed? Is the recipient in the allowlist? Is the amount within the transfer limit? If ANY check fails, the signature is never produced.

2. **On-chain Mandate (at execution).** Even with a valid signature, the EMEIMandate contract verifies: cap, counterparties, categories, time window. If ANY check fails, the transaction reverts.

Both layers must pass. Either alone is sufficient to block unauthorized spending.

### Session Token Security

- Stored as SHA-256 hash (plaintext never persisted)
- Mandatory 90-day TTL with rotation warning at day 75
- Optional IP CIDR allowlist per token
- Per-token rate limiting (60 req/min default)
- Instant revocation (DB flag, next request fails)
- Blast radius = agent's remaining mandate cap to pre-approved addresses

### Compromise Analysis

| Scenario | Attacker Capability | Max Damage | Recovery |
|----------|-------------------|-----------|---------|
| Session token stolen | Spend within mandate rules | Remaining mandate cap | Revoke token (1 tap, <5s) |
| Facilitator DB compromised | Token hashes (unusable), cached state | Zero financial | Rebuild from chain |
| Facilitator code compromised | Request signatures within existing policies | Mandate caps to approved addresses | Revoke mandates |
| Privy fully compromised | Sign any operation | On-chain mandates still reject unauthorized | Revoke all mandates on-chain |

---

## Transaction Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEDICATED QUEUE ARCHITECTURE                       │
│                                                                     │
│  ┌─────────────────┐  ┌───────────────────┐  ┌──────────────────┐ │
│  │  RECEIPT QUEUE   │  │ MAINTENANCE QUEUE  │  │ COLLECTION QUEUE │ │
│  │  postMerkleRoot  │  │ markOverdue        │  │ collect()        │ │
│  │                  │  │ markExpired        │  │                  │ │
│  │  Priority: 10    │  │ topUpFromYield     │  │  Priority: 2-8   │ │
│  └────────┬─────────┘  └────────┬──────────┘  └─────┬──────┬─────┘ │
│           │                     │                    │      │       │
│    ┌──────▼──────┐       ┌──────▼──────┐      ┌─────▼──┐ ┌─▼────┐ │
│    │  HOT WALLET │       │  HOT WALLET │      │ HOT    │ │ HOT  │ │
│    │  #1         │       │  #2         │      │ WAL #3 │ │ W #4 │ │
│    │  DEDICATED  │       │  DEDICATED  │      │ SHARED │ │SHARED│ │
│    └─────────────┘       └─────────────┘      └────────┘ └──────┘ │
│         │                       │                                   │
│         └───── steal from collection when idle ─────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

Dedicated wallets guarantee that critical operations (receipt anchoring, overdue marking) always have available capacity regardless of collection volume. When idle, they steal work from the collection queue to maximize throughput.

---

## Auto-Collector (Scalable Design)

The collector never makes RPC calls during matching. It reads from a PostgreSQL index populated in real-time by the event indexer.

```sql
-- Single atomic query: find matches → lock rows → mark claimed → return pairs.
-- FOR UPDATE OF i SKIP LOCKED prevents race conditions between replicas.
WITH claimable AS (
    SELECT i.invoice_id, m.mandate_id
    FROM invoice_index i
    JOIN mandate_index m ON m.payer = i.payer
        AND m.status = 0
        AND m.remaining_cap >= i.amount
        AND m.valid_from <= NOW_EPOCH
        AND m.valid_until >= NOW_EPOCH
        AND i.issuer = ANY(m.counterparties)
        AND i.category = ANY(m.categories)
    WHERE i.status = 1 AND i.collection_mode = 0
        AND i.collect_submitted = FALSE
        AND i.expires_at > NOW_EPOCH
    ORDER BY i.created_at ASC LIMIT 50
    FOR UPDATE OF i SKIP LOCKED
)
UPDATE invoice_index SET collect_submitted = TRUE FROM claimable
WHERE invoice_index.invoice_id = claimable.invoice_id
RETURNING claimable.invoice_id, claimable.mandate_id;
```

Performance: sub-5ms at 100k invoices/day. O(1) per cycle regardless of invoice volume.

---

## Distributed Consistency (Outbox Pattern)

Mandate creation requires two systems to agree (chain + Privy). They cannot be atomic.

```
User creates mandate
  │
  ├── 1. Write intent to mandate_sync table (outbox)
  ├── 2. Submit createMandateSigned() on-chain
  ├── 3. Wait for MandateCreated event (chain confirmation)
  ├── 4. Attempt Privy policy update (best-effort)
  │       ├── Success → mark privy_status = 'synced'
  │       └── Failure → reconciler retries every 30s
  └── Return to user
```

Failure modes are safe by construction:
- Chain confirmed + Privy fails → agent can't spend (too restrictive, reconciler fixes in ≤30s)
- Chain fails + Privy updated → on-chain rejects everything (reconciler rolls back policy)
- Both fail → nothing happened, user retries

The reconciler service continuously heals divergence. Alert at 10 consecutive failures.

---

## Nonce Management

| Nonce Type | Purpose | Managed By |
|-----------|---------|-----------|
| Ethereum tx nonce | Prevents raw transaction replay | **Privy** (internal to send_transaction) |
| EIP-712 app nonce | Prevents signed message replay | **Facilitator** (query contract, per-signer mutex) |

The facilitator holds no Ethereum tx nonce state. Privy handles sequencing for all wallets (hot + agent). The only nonce the facilitator tracks is the per-signer application nonce in the EMEI contracts, used when building EIP-712 signed messages.

---

## Authentication

| Route Class | Auth | Credential | Lifetime |
|------------|------|-----------|----------|
| Agent management, mandates, withdraw | UserAuth | Privy JWT | Hours (browser session) |
| Invoice, present, pay, collect, x402 | AgentAuth | Session token (`frt_sk_live_...`) | 90 days (mandatory rotation) |
| Balance, reputation, verify, statement | Public | None | — |
| Webhook | HMAC | Alchemy signing key | Permanent |

---

## Configuration

All configuration via environment variables. No config files.

```bash
# Required
EMEI_RPC_URL=https://mainnet.base.org
EMEI_CHAIN_ID=8453
EMEI_INVOICE_ADDRESS=0x...
EMEI_MANDATE_ADDRESS=0x...
EMEI_SETTLEMENT_ADDRESS=0x...
EMEI_RECEIPT_ADDRESS=0x...
EMEI_REPUTATION_ADDRESS=0x...
PRIVY_APP_ID=...
PRIVY_APP_SECRET=...
PRIVY_HOT_WALLET_IDS=sw_h01,sw_h02,sw_h03,sw_h04,sw_h05
DATABASE_URL=postgres://user:pass@host:5432/emei
REDIS_URL=redis://host:6379

# Optional (defaults shown)
EMEI_BATCH_INTERVAL=30
EMEI_COLLECT_INTERVAL=10
EMEI_OVERDUE_INTERVAL=60
EMEI_SWEEP_INTERVAL=30
EMEI_RECONCILE_INTERVAL=30
EMEI_SESSION_TOKEN_TTL_DAYS=90
ALCHEMY_WEBHOOK_SIGNING_KEY=whsec_...
ALERT_WEBHOOK_URL=https://hooks.slack.com/...
```

---

## Running

### Development

```bash
cp .env.example .env
# Fill in values
cargo run --bin emei-server
```

### Production (Docker)

```bash
docker build -t emei-facilitator .
docker run -p 8080:8080 --env-file .env emei-facilitator
```

### Infrastructure Requirements

| Service | Purpose | Minimum |
|---------|---------|---------|
| PostgreSQL 15+ | All persistent state | 1 instance, 20 connections |
| Redis 7+ | Rate limiting, app nonce cache | 1 instance |
| Base RPC | Chain reads + event polling | Alchemy Growth (or equivalent) |
| Privy | Wallet creation, signing, policies | Enterprise plan |

---

## API Summary

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | /emei/agents | UserAuth | Create agent wallet |
| GET | /emei/agents | UserAuth | List user's agents |
| POST | /emei/agents/{id}/rotate | UserAuth | Rotate session token |
| POST | /emei/agents/{id}/revoke | UserAuth | Revoke all tokens, freeze agent |
| POST | /emei/mandate | UserAuth | Create spending mandate |
| DELETE | /emei/mandate/{id} | UserAuth | Revoke mandate (kill switch) |
| POST | /emei/invoice | AgentAuth | Create invoice |
| POST | /emei/present | AgentAuth | Present invoice |
| POST | /emei/pay | AgentAuth | Pay invoice directly |
| POST | /emei/collect | AgentAuth | Collect via mandate |
| POST | /emei/withdraw | UserAuth | Withdraw from vault |
| GET | /emei/invoice/{id} | Public | Invoice details |
| GET | /emei/balance/{addr} | Public | Vault balance + yield |
| GET | /emei/reputation/{addr} | Public | On-chain reputation score |
| GET | /emei/statement | Public | Payment history |
| GET | /emei/verify/{id} | Public | Merkle inclusion proof |
| GET | /emei/paylink/{id} | Public | Pre-encoded pay calldata |
| GET | /emei/public/stats | Public | Protocol statistics |
| GET | /emei/public/events | Public | Recent events |
| GET | /health | Public | System health |
| POST | /emei/webhook | HMAC | Alchemy event ingestion |

---

## Background Services (9 workers)

| Service | Interval | Queue | Purpose |
|---------|----------|-------|---------|
| auto_collector | 10s | collection | Match invoices to mandates, enqueue collect() |
| overdue_scanner | 60s | maintenance | Detect overdue invoices, enqueue markOverdue() |
| expiry_scanner | 60s | maintenance | Detect expired invoices, enqueue markExpired() |
| receipt_batcher | 30s | receipt | Build Merkle tree, enqueue postMerkleRoot() |
| sweeper | 30s | maintenance | Rebalance: Yield → Conservative when balance low |
| buffer_monitor | 30s | — | Adjust withdrawal caps based on buffer health |
| reconciler | 30s | — | Retry Privy policy syncs, rollback orphans |
| tx_reaper | 120s | — | Reclaim stuck tx_queue jobs |
| event_indexer | 2s | — | Poll chain events → populate Postgres indexes |

Plus N tx_sender workers (one per hot wallet, queue-assigned with work stealing).

---

## Key Invariants

These properties hold under all conditions, including partial failures:

1. **No key material on disk or in memory.** All signing delegated to Privy.
2. **Mandate cap is a hard ceiling.** Even a compromised facilitator cannot exceed it.
3. **Receipt batches are sequential.** Batch N+1 can only exist after batch N.
4. **Session tokens expire.** No infinite-lived credentials. Max 90 days.
5. **Dedicated queues never starve.** Receipt and maintenance wallets are reserved.
6. **Collector claims are atomic.** FOR UPDATE SKIP LOCKED prevents double-processing.
7. **Outbox guarantees eventual sync.** Privy policy catches up within 30s of chain confirmation.
8. **Failure mode is always restrictive.** Partial sync = agent can't spend (not: agent overspends).

---

## License

Proprietary. Fortress Inc. All rights reserved.
