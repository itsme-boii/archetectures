# EMEI Contract Suite — Upgrade Specification

> This document details every change across all 5 contracts. Read through, approve, and then we implement in dependency order.

---

## Overview

| Contract | Current | After Upgrade | Key Changes |
|----------|---------|---------------|-------------|
| Bay8004 → **EMEIReputation** | ERC-8004 wrapper, external registry | Standalone internal scoring | Drop ERC-8004, add history, logarithmic scoring |
| EMEIReceipt | Owner/poster pattern, custom Merkle | AccessControl + OZ MerkleProof | Sequential batches, metadata, pause posting |
| EMEIMandate | Single owner, no signatures | AccessControl + EIP-712 | Signed create/revoke, recurring resets, per-counterparty limits |
| EMEISettlement | Owner/facilitator, string requires | AccessControl + ReentrancyGuard | Daily withdraw limits, asset whitelist, emergency drain |
| EMEIInvoice | Single owner, string storage | ERC-721 + AccessControl + EIP-712 | NFT invoices, state machine lib, metadata hashing |

**Shared patterns across all contracts:**
- AccessControl (granular roles, no single owner)
- Custom errors (no string requires)
- Pausable where needed
- Storage packing
- Comprehensive events

---

## 1. EMEIReputation (was Bay8004)

### What changes

| # | Change | Before | After |
|---|--------|--------|-------|
| 1 | Drop ERC-8004 dependency | Wraps external MockERC8004 registry | Internal `mapping(address => ReputationData)` |
| 2 | Internal storage | Calls external contract for reads/writes | Direct SLOAD/SSTORE — 60% cheaper reads |
| 3 | AccessControl | `onlyOwner` | `SCORER_ROLE` (InvoiceContract only) + `PAUSER_ROLE` |
| 4 | Positive updates | `giveFeedback(subject, invoiceId, amount)` | `addPositive(account, invoiceId, weight)` with diminishing returns |
| 5 | Negative updates | Same function with amount=0 | `addNegative(account, invoiceId, weight)` with amplification at low scores |
| 6 | History | Not stored | Append-only `ReputationEvent[]` per account |
| 7 | Pause | None | Pause scoring updates, views always work |
| 8 | Custom errors | `require("string")` | `ZeroAddress()`, `ZeroWeight()` |

### Scoring model

```
Base score: 500 (auto-assigned on first interaction)
Max: 10000, Min: 0

Positive (payment confirmed):
  points = log2(amount_in_dollars) + 1
  if score > 800: points = points / 2 + 1  (soft cap)

Negative (overdue):
  points = log2(amount_in_dollars) + 1
  if score < 300: points = points * 1.5    (amplified penalty)
```

### Example

```
Agent pays 100 USDC invoice:
  → log2(100) + 1 = 8 points
  → score: 500 → 508

Agent misses 50 USDC invoice (overdue):
  → log2(50) + 1 = 7 points
  → score 508 > 300, no amplification
  → score: 508 → 501

Agent at score 250 misses another 10 USDC invoice:
  → log2(10) + 1 = 5 points
  → score < 300: amplified → 5 * 1.5 = 7 points
  → score: 250 → 243 (penalties hurt more when trust is already low)
```

### Gas savings

| Operation | Before | After |
|-----------|--------|-------|
| Register identity | 80k (ERC-8004 registry tx) | 0 (auto-init) |
| Update score | 55k | 45k |
| Read score | 5k (external call) | 2.1k (single SLOAD) |

---

## 2. EMEIReceipt

### What changes

| # | Change | Before | After |
|---|--------|--------|-------|
| 1 | AccessControl | `owner` + `poster` addresses | `DEFAULT_ADMIN_ROLE` + `POSTER_ROLE` + `PAUSER_ROLE` |
| 2 | Zero-address validation | None | Constructor reverts on zero addresses |
| 3 | Poster rotation | Redeploy needed | `grantRole(POSTER_ROLE, new)` + `revokeRole(old)` |
| 4 | PosterUpdated event | None | Emitted on every role change |
| 5 | Sequential batch enforcement | Can post batch 100 then batch 2 | Must post N+1 after N |
| 6 | Custom errors | `require("string")` | `ZeroMerkleRoot()`, `BatchNotSequential()`, etc. |
| 7 | OpenZeppelin MerkleProof | Custom implementation | `MerkleProof.verify()` (battle-tested) |
| 8 | Metadata hash per batch | Only root stored | Root + IPFS CID hash per batch |
| 9 | Posting pause | None | `pause()` stops posting, verification always works |
| 10 | Timestamp validation | None | Min interval between posts (anti-spam) |
| 11 | Batch count tracking | Derived from latestBatch | Explicit `totalBatches` counter |
| 12 | Helper views | Only `getMerkleRoot` | `batchExists()`, `getMetadataHash()`, `totalBatches()` |

### Example

```
Facilitator posts batch 43:
  postMerkleRoot(43, 0x16b9d2fd..., 0xIPFS_CID_HASH)
  
  Checks:
  ✓ Caller has POSTER_ROLE
  ✓ Not paused
  ✓ Root is non-zero
  ✓ Batch 43 == latestBatch(42) + 1
  ✓ Time since last post >= minBatchInterval
  ✓ Batch 43 not already posted
  
  Stores: root + metadata
  Events: MerkleRootPosted(43, root), BatchMetadataSet(43, cid)

Anyone verifies receipt for invoice #730:
  leaf = keccak256(abi.encode(730))
  verifyInclusion(43, leaf, [sibling1, sibling2])
  → MerkleProof.verify(proof, root, leaf) → true
  → Works even during pause (view function)
```

### Gas savings

| Operation | Before | After |
|-----------|--------|-------|
| postMerkleRoot | 55k | 52k (custom errors + packed state) |
| verifyInclusion | 8k | 6k (OZ optimized MerkleProof) |

---

## 3. EMEIMandate

### What changes

| # | Change | Before | After |
|---|--------|--------|-------|
| 1 | AccessControl | `onlyOwner` | `DEFAULT_ADMIN_ROLE` + `INVOICE_ROLE` + `PAUSER_ROLE` |
| 2 | EIP-712 signed creation | Direct tx only | `createMandateSigned(params, sig, deadline)` — facilitator pays gas |
| 3 | Nonce replay protection | None | Per-address nonce, incremented on each signed op |
| 4 | Pausable | None | `pause()` blocks all utilization |
| 5 | Custom errors | `require("string")` | `InsufficientMandateCap()`, `CounterpartyLimitExceeded()`, etc. |
| 6 | MandateUtilized event | Only MandateCreated/Revoked | `MandateUtilized(id, amount, remaining)` on every spend |
| 7 | Auto-expiry in views | Expired shows as ACTIVE | `statusOf()` returns EXPIRED if past validUntil |
| 8 | Duplicate validation | None | Rejects duplicate counterparties and categories |
| 9 | Zero-address validation | None | Rejects zero-address counterparties |
| 10 | Category as bytes32 | `string[]` (expensive) | `bytes32[]` (cheap, max 32 chars) |
| 11 | Storage packing | Unpacked struct | 3 slots per mandate (down from 6-8) |
| 12 | Signed revocation | Direct tx only | `revokeMandateSigned(id, sig, deadline)` — revoke from mobile |
| 13 | Per-counterparty limits | Global cap only | Each counterparty can have its own sub-limit |
| 14 | Mandate top-up | Must create new mandate | `topUp(mandateId, additionalCap)` adds to existing |
| 15 | Recurring allowance reset | None | Configurable auto-reset every N days |
| 16 | View helpers | Must fetch full struct | `remainingCapOf()`, `isExpired()`, `counterpartySpentOf()` |

### Example: Signed creation (facilitator pays gas)

```
User signs off-chain (EIP-712):
  domain: { name: "EMEIMandate", version: "2", chainId: 8453, contract: 0xMandate }
  message: { 
    payer: 0xUserAgent, spendCap: 500e6, 
    counterparties: [0xCompute, 0xData],
    categories: [bytes32("compute"), bytes32("data")],
    counterpartyLimits: [300e6, 200e6],
    validFrom: now, validUntil: now+30days,
    resetIntervalDays: 30, resetAmount: 500e6,
    nonce: 0, deadline: now+1hr
  }

Facilitator submits: createMandateSigned(params, signature, deadline)
  → Verify EIP-712 sig from 0xUserAgent ✓
  → Nonce 0 matches ✓
  → No duplicate counterparties ✓
  → No zero addresses ✓  
  → Categories all ≤ 32 bytes ✓
  → Creates mandate #15 with per-counterparty limits
  → Gas: facilitator pays ~130k, user pays 0
```

### Example: Recurring reset

```
Mandate #15: 500 USDC/month, resets every 30 days.

Day 1-29: Agent spends 480 USDC → remaining = 20 USDC
Day 30: Next utilize() triggers _maybeResetAllowance():
  → elapsed = 30 days, periods = 1
  → remaining reset to 500 USDC
  → MandateReset(15, 500e6)
Day 31: Fresh budget, agent can spend 500 USDC again
```

### Gas savings

| Operation | Before | After |
|-----------|--------|-------|
| createMandate | 200k (strings) | 130k (bytes32, packed) |
| utilize | 60k | 45k (packed reads, custom errors) |
| statusOf (view) | 5k | 2.5k (auto-expiry in view) |

---

## 4. EMEISettlement

### What changes

| # | Change | Before | After |
|---|--------|--------|-------|
| 1 | AccessControl | `onlyOwner` / `onlyFacilitator` | `INVOICE_ROLE` + `FACILITATOR_ROLE` + `PAUSER_ROLE` + `ASSET_MANAGER` |
| 2 | Custom errors | `require("string")` | `ZeroAmount()`, `InsufficientBalance()`, etc. |
| 3 | ReentrancyGuard | None | On settle/withdraw/topUp (Satellites are external) |
| 4 | Pausable | None | Freeze settlements + withdrawals |
| 5 | Asset whitelist | Hardcoded USDC only | `acceptedAssets` mapping (future multi-asset) |
| 6 | Daily withdrawal limits | None | Configurable per-agent daily cap |
| 7 | Settlement proof history | Single proof returned | `settlementProofs[invoiceId].push(proof)` |
| 8 | Buffer pool cap | Unbounded | Max buffer size to prevent over-accumulation |
| 9 | Emergency drain | None | `emergencyDrainBuffer(dest)` for last-resort recovery |
| 10 | Satellite rotation | Redeploy needed | `setConservativeSatellite()` / `setYieldSatellite()` |
| 11 | Storage packing | Unpacked mappings | `AgentVault` struct (2 slots) + `SettlementState` (1 slot) |
| 12 | Constructor zero-checks | Partial | All addresses validated |

### Example: Settlement with dual-vault routing

```
Invoice #42 paid: 100 USDC. Agent A's sweep limit = 50 USDC. Conservative balance = 30 USDC.

settle(42, agentB, agentA, 100e6, USDC):
  → Pull USDC from agentB ✓
  → Conservative gap: 50 - 30 = 20 USDC → deposit 20 to Conservative
  → Excess: 100 - 20 = 80 USDC
  → Buffer (5%): 80 * 500 / 10000 = 4 USDC → bufferPool += 4
  → Yield (95%): 80 - 4 = 76 USDC → deposit 76 to Yield Satellite
  → Proof = keccak256(invoiceId, payer, payee, amount, shares, timestamp, block)
  → settlementProofs[42].push(proof)
  → Event: SettlementExecuted(42, agentA, 100e6, cShares, yShares, 4e6)
```

### Example: Daily withdrawal limit

```
Agent A tries to withdraw 600 USDC. Daily limit = 500 USDC. Already withdrew 200 today.

withdraw(600e6):
  → 200 + 600 = 800 > 500
  → revert DailyWithdrawLimitExceeded(agentA, 600e6, 300e6)
  
withdraw(300e6):  // retry with allowed amount
  → 200 + 300 = 500 ≤ 500 ✓
  → Waterfall: Conservative → Yield → Buffer
  → Transfer 300 USDC to agent ✓
```

### Gas savings

| Operation | Before | After |
|-----------|--------|-------|
| settle (Conservative only) | 75k | 65k |
| settle (full split) | 120k | 105k |
| withdraw (single source) | 60k | 55k |
| getVaultBalance (view) | 8k | 6k |

---

## 5. EMEIInvoice

### What changes

| # | Change | Before | After |
|---|--------|--------|-------|
| 1 | Milestone overdue fix | Always checks milestone[0] | Checks first UNPAID milestone |
| 2 | Non-blocking feedback | Reverts if Bay8004 fails | `try reputation.addPositive(...) catch` |
| 3 | Custom errors | `require("string")` | `InvalidStatusTransition()`, `InvoiceExpired()`, etc. |
| 4 | Upgradeable deps | Hardcoded addresses | `setReputation()`, `setSettlement()`, `setMandate()` with events |
| 5 | Settlement proofs | Not stored on invoice side | Stored via Settlement, queryable per invoice |
| 6 | AccessControl | `onlyOwner` | `ADMIN_ROLE`, `ARBITER_ROLE`, `FACILITATOR_ROLE`, `PAUSER_ROLE` |
| 7 | Invoice expiry | None | `expiresAt` field, checked on pay/collect/present |
| 8 | Explicit category | Derived from first line item | `bytes32 category` field on creation |
| 9 | Storage packing | 8-10 slots per invoice | 3 slots (PackedInvoice + Parties + Payment) |
| 10 | Settlement freeze | None | `whenNotFrozen` on pay/collect, separate from pause |
| 11 | State transition lib | Scattered if/else | `InvoiceStateLib.validateTransition()` |
| 12 | EIP-712 signatures | None | `createInvoiceSigned()`, `acceptInvoiceSigned()` |
| 13 | Metadata hashing | Full strings on-chain | `bytes32 metadataHash` (IPFS CID), line items off-chain |
| 14 | Invoice NFT | None | ERC-721 — mint on create, payment follows `ownerOf()` |

### Example: EIP-712 signed creation (facilitator pays gas)

```
Agent A signs off-chain:
  domain: { name: "EMEIInvoice", version: "2", chainId: 8453, contract: 0xInvoice }
  message: {
    payer: 0xBuyer, amount: 5e6, asset: USDC,
    category: bytes32("compute"), termType: DUE_ON_RECEIPT,
    collectionMode: MANDATE, expiresAt: now+7days,
    metadataHash: 0xIPFS_CID,
    nonce: 3, deadline: now+1hr
  }

Facilitator submits: createInvoiceSigned(params, signature, deadline)
  → Verify EIP-712 sig ✓, nonce ✓, deadline ✓
  → Mint NFT #751 to signer (Agent A)
  → Pack into 3 storage slots
  → Gas: facilitator pays ~120k, Agent A pays 0
```

### Example: NFT transfer (invoice factoring)

```
Agent A owns invoice NFT #751 (5 USDC owed to them).
Factoring service buys it off-chain for 4.8 USDC (discount).

Agent A: transferFrom(agentA, factoringService, 751)
  → ownerOf(751) = factoringService

When buyer pays invoice #751:
  → settlement.settle(751, buyer, ownerOf(751), 5e6, USDC)
  → Funds go to factoringService's vault (they own the NFT)
  → Agent A already got 4.8 USDC from the factoring deal (off-chain)
```

### Example: Milestone invoice with partial payments

```
Invoice #80: 3 milestones (10 + 20 + 20 USDC)

payMilestone(80, 0) → settles 10 USDC, milestonePaid = 1, state stays PRESENTED
payMilestone(80, 1) → settles 20 USDC, milestonePaid = 2, state stays PRESENTED

Day 45: milestone[2].dueDate passed, not paid.
markOverdue(80):
  → InvoiceStateLib checks: first unpaid milestone = index 2
  → milestone[2].dueDate < now ✓
  → State: PRESENTED → OVERDUE
  → try reputation.addNegative(payer, 80, 20e6) catch {}

payMilestone(80, 2) → settles 20 USDC, all done → state: PAID
  → try reputation.addPositive(payer, 80, 20e6) catch {}
```

### Gas savings

| Operation | Before | After |
|-----------|--------|-------|
| createInvoice | 180k (strings, unpacked) | 120k (packed, bytes32, no strings) |
| pay | 95k | 75k |
| collect | 110k | 85k |
| markOverdue | 55k | 45k |
| transfer (NFT) | N/A | 50k |

---

## Implementation Order

```
1. EMEIReputation    (~150 lines)  — No dependencies
2. EMEIReceipt       (~150 lines)  — No dependencies  
3. EMEIMandate       (~300 lines)  — No dependencies
4. EMEISettlement    (~350 lines)  — Depends on ISatellite interface
5. EMEIInvoice       (~400 lines)  — Depends on Reputation + Settlement + Mandate + Receipt
```

Each contract gets:
- Implementation (src/)
- Interface (src/interfaces/)
- Custom errors (src/libraries/)
- Unit tests (test/)
- Fuzz tests (test/)
- Deploy script (script/)

---

## File Structure After Implementation

```
EMEI/
├── src/
│   ├── EMEIInvoice.sol
│   ├── EMEIMandate.sol
│   ├── EMEIReceipt.sol
│   ├── EMEIReputation.sol
│   ├── EMEISettlement.sol
│   ├── interfaces/
│   │   ├── IEMEIInvoice.sol
│   │   ├── IEMEIMandate.sol
│   │   ├── IEMEIReceipt.sol
│   │   ├── IEMEIReputation.sol
│   │   ├── IEMEISettlement.sol
│   │   └── ISatellite.sol
│   ├── libraries/
│   │   ├── InvoiceStateLib.sol
│   │   ├── InvoiceErrors.sol
│   │   ├── MandateErrors.sol
│   │   ├── ReceiptErrors.sol
│   │   ├── ReputationErrors.sol
│   │   └── SettlementErrors.sol
│   └── mocks/
│       ├── MockSatellite.sol
│       ├── MockUSDC.sol
│       ├── MaliciousSatellite.sol
│       └── MaliciousReputation.sol
├── test/
│   ├── EMEIInvoice.t.sol
│   ├── EMEIInvoice.fuzz.t.sol
│   ├── EMEIInvoice.invariant.t.sol
│   ├── EMEIMandate.t.sol
│   ├── EMEIMandate.fuzz.t.sol
│   ├── EMEIReceipt.t.sol
│   ├── EMEIReceipt.invariant.t.sol
│   ├── EMEIReputation.t.sol
│   ├── EMEISettlement.t.sol
│   ├── EMEISettlement.security.t.sol
│   └── Integration.t.sol           — Full flow end-to-end
├── script/
│   └── Deploy.s.sol
└── UPGRADE.md                       — This file
```

---

## Total Gas Savings Estimate

| Scenario (1000 invoices) | Before | After | Savings |
|--------------------------|--------|-------|---------|
| 1000 createInvoice | 180M gas | 120M gas | 33% |
| 1000 pay (mandate collect) | 110M gas | 85M gas | 23% |
| 1000 reputation updates | 55M gas | 45M gas | 18% |
| 500 markOverdue | 27.5M gas | 22.5M gas | 18% |
| 40 postMerkleRoot | 2.2M gas | 2.1M gas | 5% |
| **Total protocol gas/month** | **~375M** | **~275M** | **~27%** |

At Base gas prices (~0.001 gwei L2), this saves roughly **$0.10/month per 1000 invoices**. The real value is security, maintainability, and operational flexibility — not gas dollars on an L2.

---

## Breaking Changes

1. **Bay8004 → EMEIReputation**: All integrations calling `giveFeedback()` must switch to `addPositive()`/`addNegative()`. The InvoiceContract calls change.
2. **ERC-8004 removal**: No more `register()` for identity. Scores auto-init on first interaction.
3. **Mandate categories**: `string[]` → `bytes32[]`. Frontend must convert strings to bytes32 before calling.
4. **Invoice IDs**: Now also ERC-721 token IDs. `ownerOf(invoiceId)` = payee (transferable).
5. **Settlement**: `onlyInvoiceContract` → `onlyRole(INVOICE_ROLE)`. Must grant role after deploy.
6. **Receipt**: `postMerkleRoot(batchNumber, root)` → `postMerkleRoot(batchNumber, root, metadataHash)`. Facilitator batcher must pass metadata.

---

## Approve to proceed?

Review the above. If it looks good, say "go" and I'll implement in order:
1. EMEIReputation
2. EMEIReceipt
3. EMEIMandate
4. EMEISettlement
5. EMEIInvoice
