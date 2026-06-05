# EMEI Facilitator — How It Works

> A simple explanation of what the Facilitator does, how money moves, and how humans stay in control.

---

## What Is the Facilitator?

The Facilitator is the **brain** behind Fortress's payment system. It connects three things:

1. **Privy** — holds the keys (like a safe deposit box)
2. **The Blockchain** — where money actually moves (Base + USDC)
3. **You** — the human who sets the rules

Think of it as a smart assistant that pays bills for your AI agents, but only within the budget you set.

```mermaid
flowchart TD
    Human["👤 You (Human)"]
    Agent["🤖 Your AI Agent"]
    Facilitator["🧠 Facilitator"]
    Privy["🔐 Privy (Key Safe)"]
    Chain["⛓️ Blockchain (USDC)"]

    Human -->|"Set rules & budget"| Facilitator
    Agent -->|"Request payment"| Facilitator
    Facilitator -->|"Get signature"| Privy
    Facilitator -->|"Move money"| Chain
    Human -->|"Kill switch"| Facilitator
```

---

## The Simple Analogy

Imagine you hire an assistant and give them a company credit card:

| Real World | Fortress |
|-----------|----------|
| You (cardholder) | Your human wallet |
| The assistant | Your AI agent |
| The credit card | The agent wallet |
| Monthly spending limit | The mandate (budget) |
| Card company (Visa) | The blockchain |
| The bank | Privy (holds the keys) |
| "Block this card" button | Revoke mandate (kill switch) |

The assistant can buy things within the limit. They can't change the limit. They can't buy from stores you haven't approved. And you can freeze the card instantly from your phone.

---

## How a Payment Works

```mermaid
sequenceDiagram
    participant Agent as 🤖 Your Agent
    participant F as 🧠 Facilitator
    participant P as 🔐 Privy
    participant BC as ⛓️ Blockchain

    Agent->>F: "I need to pay $5 for data"
    F->>F: Check: Is this within budget? ✓
    F->>F: Check: Is this seller approved? ✓
    F->>P: "Please sign this payment"
    P->>P: Check own rules (policy) ✓
    P-->>F: Here's the signature
    F->>BC: Submit payment (Facilitator pays gas)
    BC->>BC: Check mandate on-chain ✓
    BC-->>F: Payment confirmed ✅
    F-->>Agent: Done! Here's your receipt.
```

**Key point:** The agent never holds keys. It asks the Facilitator, which asks Privy. Two separate checks happen before money moves.

---

## The Two Safety Locks

Every payment goes through TWO independent locks. Both must pass.

```mermaid
flowchart LR
    Pay["Agent wants to pay $5"]
    Lock1{"🔒 Lock 1: Privy Policy<br/>(off-chain)"}
    Lock2{"🔒 Lock 2: On-chain Mandate<br/>(blockchain)"}
    Done["✅ Money moves"]
    Block1["❌ Blocked (never signed)"]
    Block2["❌ Blocked (reverts on-chain)"]

    Pay --> Lock1
    Lock1 -->|"Rules pass"| Lock2
    Lock1 -->|"Rules fail"| Block1
    Lock2 -->|"Rules pass"| Done
    Lock2 -->|"Rules fail"| Block2
```

Even if a hacker gets into the Facilitator's database, they still can't steal money because:
- Privy won't sign unauthorized operations (Lock 1)
- The blockchain won't execute unauthorized payments (Lock 2)
- The human can revoke everything instantly

---

## What You (The Human) Control

| Action | What Happens | Time to Take Effect |
|--------|-------------|-------------------|
| **Create a mandate** | Sets budget + rules for your agent | Immediate |
| **Revoke a mandate** | Freezes all agent spending | 1 block (~2 seconds) |
| **Withdraw funds** | Moves money back to your wallet | Instant (from buffer) |
| **View activity** | See every payment your agent made | Real-time |
| **Verify a payment** | Cryptographic proof it happened | Always available |

---

## What Your Agent Can Do (And Can't Do)

### ✅ CAN do (within mandate rules):
- Pay approved sellers for approved things
- Create invoices (charge other agents)
- Check its own balance
- Prove a payment happened

### ❌ CANNOT do (even if hacked):
- Pay addresses you didn't approve
- Spend more than the budget limit
- Spend in unapproved categories
- Withdraw to an external address
- Change its own rules
- Create more budget for itself

---

## How You Get Started

```mermaid
flowchart TD
    Step1["1️⃣ Sign up on fortress.xyz"]
    Step2["2️⃣ Create an agent wallet"]
    Step3["3️⃣ Set a budget (mandate)"]
    Step4["4️⃣ Copy the session token"]
    Step5["5️⃣ Give token to your agent code"]
    Step6["6️⃣ Agent runs autonomously"]

    Step1 --> Step2 --> Step3 --> Step4 --> Step5 --> Step6
```

1. **Sign up** — Log in with email or Google. You get a personal wallet automatically.
2. **Create an agent** — Click "New Agent" on the dashboard. This creates a separate wallet for your AI.
3. **Set budget** — Choose how much it can spend, who it can pay, and for how long.
4. **Copy token** — A session token appears. This is what your agent uses to authenticate.
5. **Add to code** — Put the token in your agent's environment variables.
6. **It works** — Your agent pays for things automatically, within your rules.

---

## Where Your Money Lives

When your agent gets paid, the money doesn't sit in a single pool. It's split into two buckets:

```mermaid
flowchart TD
    Payment["💵 Incoming Payment (50 USDC)"]
    Split{"Split based on your sweep limit"}
    Checking["🏦 Checking (Spendable)<br/>Available instantly for next payment"]
    Savings["📈 Savings (Yield)<br/>Earning ~8% APY in DeFi"]
    Buffer["🛟 Buffer Pool<br/>Emergency liquidity (5%)"]

    Payment --> Split
    Split -->|"Fill checking to target"| Checking
    Split -->|"Excess"| Savings
    Split -->|"5% of excess"| Buffer
```

- **Checking**: Ready to spend immediately (like a debit card balance)
- **Savings**: Growing via Morpho/Aerodrome yield strategies (like a high-yield savings account)
- **Buffer**: Shared emergency pool for instant withdrawals

The system automatically moves money from Savings → Checking when your balance gets low.

---

## Receipts & Proof

Every payment gets a cryptographic receipt anchored on the blockchain every 30 seconds. This means:

- ✅ Anyone can verify any payment happened (no trust required)
- ✅ Auditors can reconstruct full payment history
- ✅ Your agent can prove it paid (for dispute resolution)
- ✅ Immutable — nobody can delete or alter the record

---

## What Happens If Something Goes Wrong

| Scenario | What Happens | Your Action |
|----------|-------------|------------|
| Agent goes rogue | Bounded by mandate — max damage = remaining budget | Revoke mandate (1 tap) |
| Session token leaked | Attacker limited to same rules as your agent | Revoke token (1 tap) + rotate |
| Facilitator hacked | Two locks still hold — can't get unauthorized signatures | Revoke mandate from dashboard |
| You lose your phone | Privy recovery (email, social) restores access | Recover via Privy |

The design philosophy: **assume everything between you and the blockchain will fail**, and make sure the on-chain mandate still protects your money.

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Payment confirmation | ~2 seconds (Base block time) |
| Receipt anchoring | Every 30 seconds |
| Kill switch latency | 1 block (~2 seconds) |
| Gas cost to you | $0 (Fortress sponsors all gas) |
| Min mandate budget | 1 USDC |
| Max agents per user | Unlimited |
| Max mandates per agent | Unlimited |

................................................................................................................................................................


# EMEI Facilitator — Developer Reference

> Complete technical specification for building and integrating with the EMEI Facilitator.
> Covers architecture, auth, wallet management, SDK design, API contracts, background services, and deployment.

---

## 1. System Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Layer"]
        FE["Dashboard (React)"]
        SDK_TS["SDK — TypeScript"]
        SDK_PY["SDK — Python"]
        WEBHOOK["Alchemy Webhooks"]
    end

    subgraph Facilitator["EMEI Facilitator (Rust / Axum)"]
        API["HTTP API Layer"]
        AUTH["Auth Module"]
        PRIVY_CLIENT["Privy Client"]
        TX_QUEUE["Tx Queue (PostgreSQL)"]
        WORKERS["Background Workers"]
    end

    subgraph Privy["Privy Infrastructure"]
        EMB["Embedded Wallets (humans)"]
        SRV["Server Wallets (agents + hot)"]
        POL["Wallet Policies"]
        SIG["Signing Service"]
    end

    subgraph Chain["Base (On-Chain)"]
        INV["EMEIInvoice"]
        MAN["EMEIMandate"]
        SET["EMEISettlement"]
        REC["EMEIReceipt"]
        REP["EMEIReputation"]
        SAT["Satellite Vaults"]
    end

    subgraph Storage["Persistence"]
        PG["PostgreSQL"]
        REDIS["Redis (cache)"]
    end

    FE & SDK_TS & SDK_PY --> API
    WEBHOOK --> API
    API --> AUTH
    AUTH --> PRIVY_CLIENT
    PRIVY_CLIENT --> SRV & POL & SIG
    API --> TX_QUEUE
    TX_QUEUE --> WORKERS
    WORKERS --> PRIVY_CLIENT
    WORKERS --> Chain
    API --> PG & REDIS
    WORKERS --> PG
```

---

## 2. Authentication Design

### Two-tier auth model

| Tier | Credential | Lifetime | Used By | For |
|------|-----------|----------|---------|-----|
| **User Auth** | Privy JWT | Hours (auto-refresh in browser) | Dashboard, mandate ops, withdrawals | Actions requiring human intent |
| **Agent Auth** | Session Token (`frt_sk_live_...`) | Until revoked | SDK (headless) | Autonomous agent operations |

### Auth flow diagram

```mermaid
sequenceDiagram
    participant U as Human (Browser)
    participant A as Agent (Server)
    participant F as Facilitator
    participant P as Privy

    Note over U,P: User Auth (Tier 1)
    U->>P: Login (email/Google/passkey)
    P-->>U: JWT (did:privy:abc123)
    U->>F: POST /emei/mandate (Bearer JWT)
    F->>P: Verify JWT
    P-->>F: ✓ user_id confirmed
    F->>F: Proceed with human-intent operation

    Note over A,P: Agent Auth (Tier 2)
    A->>F: POST /emei/x402/sign (Bearer frt_sk_live_xxx)
    F->>F: hash(token) → lookup session_tokens table
    F->>F: Validate: not revoked, not expired
    F->>F: Resolve: agent_address, server_wallet_id
    F->>F: Proceed with agent operation
```

### Session Token Structure

```
frt_sk_live_7f8a9b2c4d5e6f...
 │   │  │    │
 │   │  │    └─ 32 bytes random (crypto-secure)
 │   │  └─ environment: "live" | "test"
 │   └─ type: "sk" (session key)
 └─ prefix: "frt" (fortress)
```

Storage (never store plaintext):
```sql
CREATE TABLE session_tokens (
    id BIGSERIAL PRIMARY KEY,
    token_hash TEXT UNIQUE NOT NULL,          -- sha256(plaintext)
    agent_id BIGINT REFERENCES agents(id) NOT NULL,
    scopes TEXT[] DEFAULT '{sign,read}',
    ip_allowlist CIDR[],                      -- optional IP restriction
    rate_limit_rpm INT DEFAULT 60,            -- requests per minute
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,                   -- NULL = never
    revoked_at TIMESTAMPTZ,                   -- NULL = active
    last_used_at TIMESTAMPTZ,
    last_ip INET
);
CREATE INDEX idx_session_tokens_hash ON session_tokens(token_hash);
```

---

## 3. Wallet Management

### Wallet types in the system

```mermaid
flowchart TD
    subgraph Privy_Wallets["All Wallets (Privy-Managed)"]
        EMB["Embedded Wallet<br/>Human owns key<br/>0xAlice"]
        AGENT["Server Wallet (Agent)<br/>Privy holds key<br/>0xAgent"]
        HOT["Server Wallet (Hot Pool)<br/>Privy holds key<br/>0xHot1, 0xHot2..."]
    end

    EMB -->|"signs mandates, withdrawals"| MAN_CONTRACT["EMEIMandate"]
    AGENT -->|"signs invoices (EIP-712)"| INV_CONTRACT["EMEIInvoice"]
    HOT -->|"submits txs (pays gas)"| ALL_CONTRACTS["All Contracts"]
```

### Agent wallet creation flow

```mermaid
sequenceDiagram
    participant U as User (JWT)
    participant F as Facilitator
    participant P as Privy
    participant DB as PostgreSQL

    U->>F: POST /emei/agents { name: "trading-bot" }
    F->>F: Verify JWT → user_id
    F->>P: createServerWallet({ metadata: { owner: user_id, name, role: "agent" } })
    P-->>F: { wallet_id: "sw_001", address: "0xAgent1" }
    F->>P: setWalletPolicy("sw_001", { allowed_contracts: [], transfer_limit: "0" })
    F->>F: Generate session token (random 32 bytes)
    F->>DB: INSERT agents(...), INSERT session_tokens(hash(token), ...)
    F-->>U: { agent_address: "0xAgent1", session_token: "frt_sk_live_xxx" }
```

### Hot wallet pool setup

Created once at deployment time (not per-user):

```rust
// During facilitator bootstrap
for i in 0..NUM_HOT_WALLETS {
    let wallet = privy.create_server_wallet({
        metadata: { role: "hot_wallet", index: i, created_by: "facilitator" }
    });
    // Grant FACILITATOR_ROLE on-chain to each hot wallet address
    // Store wallet_id in config
}
```

Hot wallets have `FACILITATOR_ROLE` on-chain. They submit transactions for `collect()`, `markOverdue()`, `markExpired()`, `postMerkleRoot()`, `topUpFromYield()`.

### Privy API interactions

| Operation | Privy API Call | When |
|-----------|---------------|------|
| Create agent wallet | `POST /api/v1/server_wallets` | User creates agent |
| Set/update policy | `PATCH /api/v1/server_wallets/{id}/policy` | Mandate create/revoke |
| Sign typed data (EIP-712) | `POST /api/v1/server_wallets/{id}/sign_typed_data` | Agent signs invoice/mandate |
| Send transaction | `POST /api/v1/server_wallets/{id}/send_transaction` | Hot wallet submits tx |
| List wallets by metadata | `GET /api/v1/server_wallets?metadata.owner={user_id}` | List user's agents |

---

## 4. Mandate Lifecycle (Two-Guardrail Sync)

### Create mandate

```mermaid
sequenceDiagram
    participant U as Human
    participant F as Facilitator
    participant P as Privy
    participant C as Chain

    U->>F: POST /emei/mandate { spend_cap, counterparties, categories, valid_from/until }
    F->>F: Verify JWT → user_id
    F->>F: Look up agent for this user
    F->>C: Read EMEIMandate.nonces(user_address) → N
    F->>F: Build EIP-712 CreateMandate message (nonce=N, deadline=now+5min)
    F->>P: Sign typed data with user's embedded wallet
    P-->>F: signature
    F->>P: Send tx from hot wallet: createMandateSigned(params, user, deadline, sig)
    P-->>F: tx_hash
    Note over F: Wait for confirmation or enqueue
    F->>P: Update agent policy to mirror mandate rules
    F->>F: Store mandate in DB (mandate_id, agent, policy_version)
    F-->>U: { mandate_id, tx_hash }
```

### Revoke mandate

```mermaid
sequenceDiagram
    participant U as Human
    participant F as Facilitator
    participant P as Privy
    participant C as Chain

    U->>F: DELETE /emei/mandate/{id}
    F->>F: Verify JWT → confirm user owns this mandate
    F->>C: Read EMEIMandate.nonces(user_address) → N
    F->>F: Build EIP-712 RevokeMandate message (nonce=N)
    F->>P: Sign typed data with user's embedded wallet
    P-->>F: signature
    F->>P: Send tx from hot wallet: revokeMandateSigned(id, deadline, sig)
    F->>P: Disable agent policy (allowed_contracts: [], transfer_limit: "0")
    F-->>U: { tx_hash } — agent frozen immediately
```

---

## 5. EIP-712 Nonce Management

### Two nonce systems (both required)

| Nonce | Type | Where | Who Manages |
|-------|------|-------|-------------|
| Ethereum tx nonce | Sequential counter per EOA | Blockchain | **Privy** (for all wallets) |
| EIP-712 app nonce | `nonces[signer]` mapping | EMEI contracts | **Facilitator** (query + include in message) |

### App nonce flow

```rust
// Facilitator pseudocode for any EIP-712 signed operation
async fn sign_typed_operation(signer: Address, contract: Address, build_message: Fn) -> Signature {
    // 1. Acquire per-signer lock (prevent concurrent nonce race)
    let lock = app_nonce_locks.get_or_create(signer);
    let _guard = lock.lock().await;

    // 2. Get current app nonce from contract (or cache)
    let nonce = match nonce_cache.get(signer) {
        Some(n) => n,
        None => {
            let n = contract.nonces(signer).call().await?;
            nonce_cache.set(signer, n);
            n
        }
    };

    // 3. Build the typed data with this nonce
    let typed_data = build_message(nonce, deadline);

    // 4. Request signature from Privy
    let signature = privy.sign_typed_data(wallet_id, typed_data).await?;

    // 5. Optimistically increment local cache
    nonce_cache.set(signer, nonce + 1);

    // 6. If tx later fails with "invalid nonce", resync from chain
    signature
}
```

### Why Privy's tx nonce doesn't conflict

Privy handles the Ethereum transaction nonce for the **hot wallet** (the address that calls `createMandateSigned`). The EIP-712 app nonce is for the **signer** (the human/agent whose intent is being proven). These are different addresses:

```
Hot wallet 0xHot (tx nonce managed by Privy) 
  calls createMandateSigned(..., signature_by_0xAlice_with_app_nonce_3)

Contract checks:
  - Ethereum tx nonce of 0xHot → Privy handled this ✓
  - App nonce of 0xAlice in the signed message → Facilitator handled this ✓
```

---

## 6. Transaction Pipeline

### Architecture (with Privy-managed hot wallets)

```mermaid
flowchart LR
    SRC["enqueue_tx()<br/>(collect · overdue · receipt)"]
    Q[("PostgreSQL tx_queue<br/>priority DESC, SKIP LOCKED")]
    W1["tx_sender_1<br/>(hot_wallet sw_h01)"]
    W2["tx_sender_2<br/>(hot_wallet sw_h02)"]
    WN["tx_sender_N<br/>(hot_wallet sw_h0N)"]
    PRIVY["Privy send_transaction"]
    CHAIN["Base RPC"]
    CONF["Confirm + update DB"]
    REAPER["tx_reaper<br/>(reclaim stuck, 120s)"]

    SRC --> Q
    Q --> W1 & W2 & WN
    W1 & W2 & WN --> PRIVY
    PRIVY --> CHAIN
    CHAIN --> CONF
    REAPER -.-> Q
```

### Job schema

```sql
CREATE TABLE tx_queue (
    id BIGSERIAL PRIMARY KEY,
    to_address TEXT NOT NULL,
    calldata BYTEA NOT NULL,
    priority SMALLINT NOT NULL DEFAULT 5,
    source TEXT NOT NULL,                      -- "collector", "scanner", "batcher"
    status TEXT NOT NULL DEFAULT 'pending',    -- pending, assigned, submitted, confirmed, failed
    assigned_wallet TEXT,                       -- hot wallet ID that picked it up
    tx_hash TEXT,
    attempts SMALLINT DEFAULT 0,
    max_attempts SMALLINT DEFAULT 3,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    assigned_at TIMESTAMPTZ,
    submitted_at TIMESTAMPTZ,
    confirmed_at TIMESTAMPTZ,
    error TEXT
);
CREATE INDEX idx_tx_queue_pending ON tx_queue(priority DESC) WHERE status = 'pending';
```

### Priority levels

| Priority | Source | Transaction |
|----------|--------|-------------|
| 10 | receipt_batcher | `postMerkleRoot()` |
| 8 | explicit collect (user request) | `collect()` |
| 5 | overdue_scanner | `markOverdue()`, `markExpired()` |
| 4 | sweeper | `topUpFromYield()` |
| 2 | auto_collector | background `collect()` |

### tx_sender worker (simplified)

```rust
async fn tx_sender_loop(state: Arc<AppState>, hot_wallet_id: String, cancel: CancellationToken) {
    loop {
        select! {
            _ = cancel.cancelled() => break,
            _ = tokio::time::sleep(Duration::from_millis(500)) => {
                // Claim a job (SKIP LOCKED prevents conflicts between workers)
                let job = state.db.claim_next_tx_job(&hot_wallet_id).await;
                if let Some(job) = job {
                    match privy.send_transaction(&hot_wallet_id, &job.to_address, &job.calldata).await {
                        Ok(tx_hash) => {
                            state.db.mark_tx_submitted(job.id, &tx_hash).await;
                            // Wait for confirmation (poll or webhook)
                        }
                        Err(e) => {
                            state.db.mark_tx_failed(job.id, &e.to_string()).await;
                        }
                    }
                }
            }
        }
    }
}
```

---

## 7. Background Services

### Service registry

```rust
pub fn start_services(state: Arc<AppState>) -> Vec<JoinHandle<()>> {
    let cancel = state.cancel.clone();
    vec![
        spawn(auto_collector(state.clone(), cancel.clone())),
        spawn(overdue_scanner(state.clone(), cancel.clone())),
        spawn(expiry_scanner(state.clone(), cancel.clone())),
        spawn(receipt_batcher(state.clone(), cancel.clone())),
        spawn(sweeper(state.clone(), cancel.clone())),
        spawn(event_indexer(state.clone(), cancel.clone())),
        spawn(tx_reaper(state.clone(), cancel.clone())),
        spawn(webhook_worker(state.clone(), cancel.clone())),
        // N tx_sender workers (one per hot wallet)
        ...spawn_tx_senders(state.clone(), hot_wallet_ids, cancel),
    ]
}
```

### Service details

| Service | Interval | Logic |
|---------|----------|-------|
| **auto_collector** | 10s | Scan PRESENTED invoices → match mandates → enqueue `collect()` |
| **overdue_scanner** | 60s | Find PRESENTED invoices past due → enqueue `markOverdue()` |
| **expiry_scanner** | 60s | Find invoices past `expiresAt` → enqueue `markExpired()` |
| **receipt_batcher** | 30s | Drain pending receipts → Merkle tree → enqueue `postMerkleRoot(batch, root, metadataHash)` |
| **sweeper** | 30s | Check agents with `spendableBalance < sweepLimit` → enqueue `topUpFromYield()` |
| **event_indexer** | 5s | Poll chain for new events → index into PostgreSQL |
| **tx_reaper** | 120s | Reclaim jobs stuck in `assigned`/`submitted` > 2min |
| **webhook_worker** | event-driven | Process Alchemy webhook payloads → update event status |
| **tx_sender** (×N) | 500ms poll | Claim jobs from queue → submit via Privy → confirm |

### Auto-collector mandate matching (new ABI)

```rust
async fn find_matching_mandate(state: &AppState, invoice: &Invoice) -> Option<u64> {
    let mandate_ids = contract.getMandatesByPayer(invoice.payer).call().await?;

    for mandate_id in mandate_ids {
        // Use lightweight view functions (no full struct fetch)
        let status = contract.statusOf(mandate_id).call().await?;
        if status != MandateStatus::ACTIVE { continue; }

        let remaining = contract.remainingCapOf(mandate_id).call().await?;
        if remaining < invoice.amount { continue; }

        // Check counterparty + category from cached mandate data in DB
        let mandate = db.get_mandate(mandate_id).await?;
        if !mandate.counterparties.contains(&invoice.issuer) { continue; }
        if !mandate.categories.contains(&invoice.category) { continue; }

        return Some(mandate_id);
    }
    None
}
```

---

## 8. API Reference

### Route definitions

```rust
pub fn emei_routes() -> Router<Arc<AppState>> {
    Router::new()
        // Agent management (User Auth)
        .route("/emei/agents", post(agents::create_agent))
        .route("/emei/agents", get(agents::list_agents))
        .route("/emei/agents/{id}/token", post(agents::rotate_token))
        .route("/emei/agents/{id}/revoke", post(agents::revoke_token))

        // Mandate lifecycle (User Auth)
        .route("/emei/mandate", post(mandate::create_mandate))
        .route("/emei/mandate/{id}", delete(mandate::revoke_mandate))
        .route("/emei/mandate/{id}/topup", post(mandate::top_up))

        // Invoice lifecycle (Agent Auth)
        .route("/emei/invoice", post(invoice::create_invoice))
        .route("/emei/present", post(invoice::present_invoice))
        .route("/emei/pay", post(invoice::pay_invoice))
        .route("/emei/collect", post(invoice::collect_invoice))
        .route("/emei/milestone", post(invoice::pay_milestone))

        // x402 pay-per-use (Agent Auth)
        .route("/emei/x402/sign", post(x402::sign_payment))
        .route("/emei/x402/verify", post(x402::verify_and_settle))

        // Query (Public - no auth)
        .route("/emei/invoice/{id}", get(query::get_invoice))
        .route("/emei/statement", get(query::get_statement))
        .route("/emei/reputation/{address}", get(query::get_reputation))
        .route("/emei/balance/{address}", get(query::get_balance))
        .route("/emei/verify/{id}", get(receipt::verify_receipt))
        .route("/emei/paylink/{id}", get(paylink::get_paylink))

        // Withdrawal (User Auth)
        .route("/emei/withdraw", post(withdraw::withdraw_funds))

        // Public dashboard (no auth)
        .nest("/emei/public", public::router())

        // Ops
        .route("/health", get(health::health_check))
        .route("/emei/ops/status", get(dashboard::ops_status))
        .route("/emei/webhook", post(webhook::handle_webhook))
}
```

### Key request/response types

```rust
// --- Agent Management ---
#[derive(Deserialize)]
pub struct CreateAgentRequest {
    pub name: String,
    pub tags: Option<Vec<String>>,
    pub ip_allowlist: Option<Vec<String>>,  // CIDR ranges
}

#[derive(Serialize)]
pub struct CreateAgentResponse {
    pub agent_address: String,
    pub session_token: String,  // shown ONCE, never stored plaintext
    pub name: String,
}

// --- Mandate ---
#[derive(Deserialize)]
pub struct CreateMandateRequest {
    pub agent_address: String,
    pub spend_cap: String,                      // USDC base units (6 decimals)
    pub approved_counterparties: Vec<String>,
    pub approved_categories: Vec<String>,       // will be keccak256'd to bytes32
    pub valid_from: u64,                        // unix timestamp
    pub valid_until: u64,
    pub counterparty_limits: Option<Vec<String>>,  // per-counterparty sub-limits
    pub reset_interval_days: Option<u16>,        // recurring budget (0 = one-time)
    pub reset_amount: Option<String>,           // amount to reset to
}

// --- Invoice (new ABI) ---
#[derive(Deserialize)]
pub struct CreateInvoiceRequest {
    pub payer: String,
    pub amount: String,                         // uint96
    pub asset: String,
    pub category: String,                       // will be keccak256'd to bytes32
    pub metadata_hash: Option<String>,          // IPFS CID or keccak of line items
    pub terms: TermsRequest,
    pub collection_mode: String,                // "mandate" | "pay_link"
    pub expires_at: u64,                        // unix timestamp
}

// --- x402 ---
#[derive(Deserialize)]
pub struct X402SignRequest {
    pub invoice_id: u64,
    pub amount: String,
    pub pay_to: String,
}

#[derive(Serialize)]
pub struct X402SignResponse {
    pub signature: String,
    pub signer: String,
    pub deadline: u64,
}
```

---

## 9. SDK Design

### Package structure

```
@fortress/sdk/
├── src/
│   ├── index.ts          ← exports FortressSDK + FortressAgent
│   ├── sdk.ts            ← FortressSDK class (human ops)
│   ├── agent.ts          ← FortressAgent class (agent ops)
│   ├── x402.ts           ← fortressFetch implementation
│   ├── types.ts          ← shared types
│   └── errors.ts         ← typed errors
├── package.json
└── README.md
```

### FortressSDK (human operations)

```typescript
export class FortressSDK {
  private authToken: string;
  private baseUrl: string;

  constructor(opts: { authToken: string; baseUrl?: string }) { ... }

  // Agent management
  async createAgent(opts: { name: string; tags?: string[] }): Promise<Agent>
  async listAgents(): Promise<Agent[]>
  async revokeAgentToken(agentId: string): Promise<void>
  async rotateAgentToken(agentId: string): Promise<{ session_token: string }>

  // Mandate management
  async createMandate(opts: CreateMandateOpts): Promise<Mandate>
  async revokeMandate(mandateId: number): Promise<TxHash>
  async topUpMandate(mandateId: number, amount: string): Promise<TxHash>
  async listMandates(): Promise<Mandate[]>

  // Funds
  async withdraw(amount: string): Promise<TxHash>
  async getBalance(address?: string): Promise<Balance>
}
```

### FortressAgent (headless agent operations)

```typescript
export class FortressAgent {
  private sessionToken: string;
  private baseUrl: string;

  constructor(opts: { sessionToken: string; baseUrl?: string }) { ... }

  // Pay-per-use (the core DX)
  async fortressFetch(url: string, opts?: RequestInit & {
    maxPayment?: string;   // max willingness to pay
  }): Promise<Response>

  // Invoice operations
  async createInvoice(params: CreateInvoiceParams): Promise<Invoice>
  async present(invoiceId: number): Promise<TxHash>

  // Reads
  async balance(): Promise<Balance>
  async reputation(address?: string): Promise<number>
  async statement(opts?: StatementQuery): Promise<Event[]>
  async verify(invoiceId: number): Promise<VerifyResult>
  async getInvoice(invoiceId: number): Promise<Invoice>
}
```

### fortressFetch internals

```typescript
async fortressFetch(url: string, opts?: FetchOpts): Promise<Response> {
  // 1. Try the request normally
  const res = await fetch(url, opts);

  // 2. If not 402, return as-is
  if (res.status !== 402) return res;

  // 3. Parse payment requirements from 402 response
  const paymentReq = await res.json();
  // { price: "1000000", pay_to: "0x...", invoice_id: 42, asset: "0xUSDC..." }

  // 4. Check willingness to pay
  if (opts?.maxPayment && BigInt(paymentReq.price) > BigInt(opts.maxPayment)) {
    throw new PaymentTooExpensiveError(paymentReq.price, opts.maxPayment);
  }

  // 5. Request signature from Facilitator
  const signRes = await this.post("/emei/x402/sign", {
    invoice_id: paymentReq.invoice_id,
    amount: paymentReq.price,
    pay_to: paymentReq.pay_to,
  });

  // 6. Retry with payment header
  return fetch(url, {
    ...opts,
    headers: {
      ...opts?.headers,
      "X-PAYMENT": JSON.stringify({
        signature: signRes.signature,
        signer: signRes.signer,
        invoice_id: paymentReq.invoice_id,
        deadline: signRes.deadline,
      }),
    },
  });
}
```

---

## 10. Database Schema (Complete)

```sql
-- Users (mapped to Privy identities)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    privy_user_id TEXT UNIQUE NOT NULL,
    embedded_address TEXT NOT NULL,
    email TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Agent wallets (Privy server wallets)
CREATE TABLE agents (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) NOT NULL,
    name TEXT NOT NULL,
    agent_address TEXT UNIQUE NOT NULL,
    server_wallet_id TEXT UNIQUE NOT NULL,     -- Privy wallet ID
    tags TEXT[],
    status TEXT DEFAULT 'active',              -- active | suspended | archived
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(user_id, name)
);

-- Session tokens
CREATE TABLE session_tokens (
    id BIGSERIAL PRIMARY KEY,
    token_hash TEXT UNIQUE NOT NULL,
    agent_id BIGINT REFERENCES agents(id) NOT NULL,
    scopes TEXT[] DEFAULT '{sign,read}',
    ip_allowlist CIDR[],
    rate_limit_rpm INT DEFAULT 60,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    last_ip INET
);

-- Mandate cache (mirrors on-chain for fast lookup)
CREATE TABLE mandates (
    id BIGSERIAL PRIMARY KEY,
    mandate_id BIGINT UNIQUE NOT NULL,
    agent_id BIGINT REFERENCES agents(id) NOT NULL,
    payer_address TEXT NOT NULL,
    spend_cap TEXT NOT NULL,
    remaining_cap TEXT NOT NULL,
    approved_counterparties TEXT[] NOT NULL,
    approved_categories TEXT[] NOT NULL,        -- bytes32 hex strings
    counterparty_limits TEXT[],
    valid_from BIGINT NOT NULL,
    valid_until BIGINT NOT NULL,
    reset_interval_days SMALLINT DEFAULT 0,
    reset_amount TEXT,
    status TEXT DEFAULT 'active',
    policy_version INT DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    revoked_at TIMESTAMPTZ
);

-- Indexed on-chain events
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    block_number BIGINT NOT NULL,
    tx_hash TEXT NOT NULL,
    log_index INT NOT NULL,
    timestamp BIGINT NOT NULL,
    invoice_id BIGINT,
    payer TEXT,
    issuer TEXT,
    amount TEXT,
    params JSONB,
    status TEXT DEFAULT 'confirmed',
    UNIQUE(tx_hash, log_index)
);
CREATE INDEX idx_events_invoice ON events(invoice_id);
CREATE INDEX idx_events_payer ON events(payer, timestamp DESC);

-- Transaction queue
CREATE TABLE tx_queue (
    id BIGSERIAL PRIMARY KEY,
    to_address TEXT NOT NULL,
    calldata BYTEA NOT NULL,
    priority SMALLINT DEFAULT 5,
    source TEXT NOT NULL,
    status TEXT DEFAULT 'pending',
    assigned_wallet TEXT,
    tx_hash TEXT,
    attempts SMALLINT DEFAULT 0,
    max_attempts SMALLINT DEFAULT 3,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    assigned_at TIMESTAMPTZ,
    submitted_at TIMESTAMPTZ,
    confirmed_at TIMESTAMPTZ,
    error TEXT
);
CREATE INDEX idx_tx_pending ON tx_queue(priority DESC) WHERE status = 'pending';

-- Pending receipt hashes (drained by batcher)
CREATE TABLE pending_receipts (
    id BIGSERIAL PRIMARY KEY,
    receipt_hash BYTEA NOT NULL,
    invoice_id BIGINT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- App nonce cache (EIP-712 nonces per signer)
CREATE TABLE app_nonces (
    signer_address TEXT PRIMARY KEY,
    contract_address TEXT NOT NULL,
    current_nonce BIGINT NOT NULL DEFAULT 0,
    synced_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 11. Contract Interactions (New ABI)

### Contract bindings

```rust
// contracts/mod.rs
pub mod invoice;      // IEMEIInvoice (ERC-721, EIP-712)
pub mod mandate;      // IEMEIMandate (recurring, per-counterparty limits)
pub mod settlement;   // IEMEISettlement (dual-tranche, buffer)
pub mod receipt;      // IEMEIReceipt (3-param postMerkleRoot)
pub mod reputation;   // IEMEIReputation (replaces Bay8004 + ERC8004)
```

### Key ABI changes from old → new

| Contract | Old | New |
|----------|-----|-----|
| Invoice.CreateInvoiceParams | `amount: uint256`, `lineItems[]` on-chain | `amount: uint96`, `category: bytes32`, `metadataHash: bytes32`, `expiresAt: uint40` — line items off-chain |
| Invoice states | ISSUED, PRESENTED, PAID, OVERDUE, REJECTED | CREATED, PRESENTED, PAID, OVERDUE, EXPIRED, DISPUTED |
| Invoice functions | — | Adds `createInvoiceSigned`, `payMilestone`, `markExpired`, `dispute` |
| Mandate.CreateMandateParams | `spendCap: uint256`, categories as string | `spendCap: uint96`, categories as `bytes32[]`, adds `counterpartyLimits`, `resetIntervalDays`, `resetAmount` |
| Mandate functions | `validateAndDecrement` | Replaced by `utilize()`. Adds `createMandateSigned`, `revokeMandateSigned`, `topUp`, `statusOf`, `remainingCapOf` |
| Receipt.postMerkleRoot | 2 params: `(batch, root)` | 3 params: `(batch, root, metadataHash)` |
| Reputation | Bay8004 + ERC8004 (two contracts) | Single `EMEIReputation` with `addPositive`, `addNegative`, `scoreOf`, `getReputationData` |

### EIP-712 type hashes

```rust
// For createInvoiceSigned
const CREATE_INVOICE_TYPEHASH: &str = "CreateInvoice(address payer,uint96 amount,address asset,bytes32 category,bytes32 metadataHash,uint8 termType,uint16 netDays,uint8 collectionMode,uint40 expiresAt,uint256 nonce,uint256 deadline)";

// For createMandateSigned
const CREATE_MANDATE_TYPEHASH: &str = "CreateMandate(uint96 spendCap,address[] approvedCounterparties,bytes32[] approvedCategories,uint96[] counterpartyLimits,uint40 validFrom,uint40 validUntil,uint16 resetIntervalDays,uint96 resetAmount,uint256 nonce,uint256 deadline)";

// For revokeMandateSigned
const REVOKE_MANDATE_TYPEHASH: &str = "RevokeMandate(uint256 mandateId,uint256 nonce,uint256 deadline)";

// EIP-712 Domain (same for all EMEI contracts on Base)
const DOMAIN: EIP712Domain = {
    name: "EMEI",
    version: "1",
    chainId: 8453,  // Base mainnet
    verifyingContract: <contract_address>
};
```

---

## 12. Security Model

### Threat model & mitigations

| Threat | Mitigation |
|--------|-----------|
| Session token leaked | Bounded by mandate (max loss = remaining cap). Instant revoke via dashboard. |
| Facilitator DB compromised | Tokens stored as SHA-256 hashes. Attacker gets hashes, not usable tokens. |
| Facilitator backend compromised | Can only get signatures within Privy policy. On-chain mandate still blocks unauthorized payments. |
| Privy compromised | On-chain mandate rejects anything outside rules. Human revokes mandate immediately. |
| Hot wallet compromised | Hot wallet has FACILITATOR_ROLE only. Can trigger collect/sweep but can't bypass mandate rules. |
| Replay attack (EIP-712) | Per-signer nonce + deadline. Same signature can't be used twice. |
| Nonce front-running | Per-signer mutex in facilitator serializes signing. Nonce read → sign → submit is atomic per signer. |

### Rate limiting

```rust
// Per session token
const MAX_SIGN_REQUESTS_PER_MINUTE: u32 = 60;
const MAX_PAYMENT_AMOUNT_PER_HOUR: u128 = 10_000_000_000;  // 10,000 USDC

// Per agent (across all tokens)
const MAX_PAYMENTS_PER_HOUR: u32 = 200;

// Anomaly triggers (alert user, don't block)
const VELOCITY_ALERT_THRESHOLD: u32 = 10;  // >10 payments in 60s
const SINGLE_PAYMENT_ALERT: u128 = 1_000_000_000;  // >1000 USDC single payment
```

---

## 13. Configuration

### Environment variables

```bash
# Chain
EMEI_RPC_URL=https://mainnet.base.org
EMEI_CHAIN_ID=8453

# Contract addresses (Base mainnet)
EMEI_INVOICE_ADDRESS=0x...
EMEI_MANDATE_ADDRESS=0x...
EMEI_SETTLEMENT_ADDRESS=0x...
EMEI_RECEIPT_ADDRESS=0x...
EMEI_REPUTATION_ADDRESS=0x...

# Privy
PRIVY_APP_ID=your-app-id
PRIVY_APP_SECRET=your-app-secret
PRIVY_HOT_WALLET_IDS=sw_h01,sw_h02,sw_h03   # comma-separated

# Storage
DATABASE_URL=postgres://user:pass@host:5432/emei
REDIS_URL=redis://host:6379

# Intervals (seconds)
EMEI_BATCH_INTERVAL=30
EMEI_COLLECT_INTERVAL=10
EMEI_OVERDUE_INTERVAL=60
EMEI_SWEEP_INTERVAL=30

# Webhooks
ALCHEMY_WEBHOOK_SIGNING_KEY=whsec_...

# Alerts
ALERT_WEBHOOK_URL=https://hooks.slack.com/...
```

---

## 14. Deployment

### Docker

```dockerfile
FROM rust:1.79-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release -p emei-facilitator

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/emei-server /usr/local/bin/
EXPOSE 8080
CMD ["emei-server"]
```

### Required infrastructure

| Service | Purpose | Recommended |
|---------|---------|-------------|
| PostgreSQL 15+ | Events, tx_queue, agents, sessions | Neon / RDS |
| Redis 7+ | App nonce cache, rate limiting | Upstash / ElastiCache |
| Base RPC | Chain queries + event polling | Alchemy / QuickNode |
| Privy | Wallet creation, signing, policies | privy.io (managed) |

### Health check

```
GET /health → 200
{
  "status": "ok",
  "rpc": "connected",
  "db": "connected",
  "redis": "connected",
  "privy": "connected",
  "hot_wallets": 3,
  "tx_queue_pending": 12,
  "receipt_queue_pending": 45,
  "uptime_seconds": 86400
}
```

---

## 15. Build Order

| Phase | What | Depends On | Deliverable |
|-------|------|-----------|-------------|
| 1 | Regenerate ABIs from new Solidity interfaces | EMEI contracts deployed | `abi/*.json` |
| 2 | Privy client module | Privy account + API keys | `src/privy/` |
| 3 | Auth module (JWT verify + session tokens) | Privy client, PostgreSQL | `src/auth/` |
| 4 | Agent management routes | Auth, Privy client | `POST/GET /emei/agents` |
| 5 | Mandate routes (with policy sync) | Auth, Privy policies, EIP-712 signing | `POST/DELETE /emei/mandate` |
| 6 | Refactor auto_collector for new ABI | New contract bindings | `services/collector.rs` |
| 7 | Refactor scanner (overdue + expiry) | New contract bindings | `services/scanner.rs` |
| 8 | Update batcher (3-param postMerkleRoot) | New receipt ABI | `services/batcher.rs` |
| 9 | New sweeper service | Settlement ABI | `services/sweeper.rs` |
| 10 | Replace tx_sender with Privy send_transaction | Privy hot wallets configured | `services/tx_sender.rs` |
| 11 | x402 routes | Agent auth, signing flow | `routes/x402.rs` |
| 12 | TypeScript SDK | Facilitator API stable | `@fortress/sdk` |
| 13 | Python SDK | Facilitator API stable | `fortress-py` |
| 14 | Anomaly detection + alerts | All services running | `src/security/` |
| 15 | Load testing + deployment | Everything | Production |

