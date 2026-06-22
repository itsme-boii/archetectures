# Development Plan — Workflow Compiler & Execution Engine

## System Summary

A prompt-to-execution platform where users describe DeFi strategies in natural language, an LLM compiles them into a structured workflow DAG, the orchestrator smart contract executes all actions atomically with a single signature, and a position tracking layer keeps the user's portfolio current.

Supported protocols: Morpho, Aave, CCTP (bridge), Pendle (swap/PT).
Supported chains: Ethereum (1), Base (8453). Expandable.

---

## Architecture

```
User Prompt
  │
  ▼
┌────────────────────────────────────────────────────────────┐
│ Workflow Compiler (LLM)                                    │
│ Natural language → Structured JSON DAG                     │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│ Validator                                                  │
│ Markets exist · Tokens resolve · Amounts parse · Loops ≤10 │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│ Simulator (Tenderly fork)                                  │
│ Dry-run the workflow → projected position, APY, gas        │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│ Preview → User confirms → Single signature                 │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│ Orchestrator Contract (atomic execution)                   │
│ Emits: StrategyStarted, ActionExecuted, StrategyCompleted  │
└────────────────────┬───────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────┐
│ Position Tracker                                           │
│ Event ingestion + periodic on-chain reconciliation         │
│ APY enrichment from existing APY service                   │
└────────────────────────────────────────────────────────────┘
```

---

## Workflow JSON Schema

```typescript
type Workflow = {
  start: string;
  nodes: WorkflowNode[];
};

type WorkflowNode =
  | ActionNode
  | LoopNode
  | ResultNode;

type ActionNode = {
  id: string;
  type: "action";
  action: "swap" | "supply" | "withdraw" | "borrow" | "repay" | "bridge" | "stake" | "unstake" | "lend" | "claim";
  protocol: string;
  chain: string;
  inputs: { asset: string; amount: string };
  outputs: { asset: string };
  parameters: Record<string, string>;
  next: string;
};

type LoopNode = {
  id: string;
  type: "loop";
  iterations: number;
  entry: string;
  next: string;
};

type ResultNode = {
  id: string;
  type: "result";
};
```

---

## Components

### 1. Workflow Compiler (LLM)

Converts natural language to the workflow JSON above. Uses GPT-4o with a strict system prompt (no free-form text output, JSON only). The LLM does not resolve addresses, compute calldata, or fetch on-chain data. It maps user intent to the DAG structure.

### 2. Workflow Validator

Post-LLM validation layer:
- Every referenced market exists in the market registry
- Every token symbol resolves to a known address
- Amounts are parseable (absolute values or percentages)
- Loop iterations are bounded (max 10)
- Asset flow is consistent (output of node N feeds input of node N+1)
- Chain references match supported chains

### 3. Workflow Executor (Orchestrator Contract)

A single smart contract that:
- Receives the compiled workflow
- Executes each action in sequence (swap, supply, borrow, bridge)
- Handles loops natively (no unrolling needed on-chain)
- Emits one event per action for tracking
- Reverts atomically on any failure

### 4. Position Tracker (MVP)

Hybrid approach for MVP — no webhooks required:
- Write strategy + intent to DB at submission time (status: PENDING)
- After tx confirms: derive initial positions from the deterministic workflow
- Periodic reconciliation (every 60s): read on-chain state, correct drift, catch external actions
- APY enrichment: query existing APY service for market rates, compute net strategy APY

### 5. Exit Builder

Generates the reverse workflow to unwind any position:
- Read current on-chain position state
- Build reverse actions (repay → withdraw → swap back)
- Run through the same pipeline: validate → simulate → preview → execute

---

## Position Tracking Strategy

For MVP, positions are tracked without webhooks:

| Source | When | What it catches |
|--------|------|-----------------|
| Tx confirmation | Immediately after execution | Everything we initiated |
| Periodic reconciliation (60s) | Background poller | External exits, liquidations, interest accrual |
| On-read refresh | When user opens dashboard | Ensures freshness at view time |

On-chain state always wins. The DB is a cache of the chain.

---

## User Flows

### Flow 1: Simple Lend

**Prompt:** "Lend 500 USDC to the best Morpho vault on Base"

**Workflow:**
```json
{ "start": "lend_1", "nodes": [
  { "id": "lend_1", "type": "action", "action": "lend", "protocol": "Morpho", "chain": "Base", "inputs": { "asset": "USDC", "amount": "500" }, "outputs": { "asset": "mUSDC" }, "parameters": { "vault": "auto-best-apy" }, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Position created:** Morpho vault supply, 500 USDC deposited, earning supply APY.

---

### Flow 2: Split Allocation (Percentage-Based)

**Prompt:** "I have 1000 USDC. Put 60% in Aave, 20% in Morpho, 20% bridge to Base via CCTP"

**Workflow:**
```json
{ "start": "supply_aave", "nodes": [
  { "id": "supply_aave", "type": "action", "action": "supply", "protocol": "Aave", "chain": "Ethereum", "inputs": { "asset": "USDC", "amount": "60%" }, "outputs": { "asset": "aUSDC" }, "parameters": {}, "next": "supply_morpho" },
  { "id": "supply_morpho", "type": "action", "action": "lend", "protocol": "Morpho", "chain": "Ethereum", "inputs": { "asset": "USDC", "amount": "20%" }, "outputs": { "asset": "mUSDC" }, "parameters": {}, "next": "bridge_cctp" },
  { "id": "bridge_cctp", "type": "action", "action": "bridge", "protocol": "CCTP", "chain": "Ethereum", "inputs": { "asset": "USDC", "amount": "20%" }, "outputs": { "asset": "USDC" }, "parameters": { "destChain": "Base" }, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Positions created:** Aave supply (600 USDC), Morpho supply (200 USDC), CCTP in-transit (200 USDC).

---

### Flow 3: Leveraged PT Loop (Infinit-style)

**Prompt:** "Swap USDC to PT-apyUSD on Pendle, supply as collateral to Morpho, borrow USDC at 75% LTV, repeat 5 more times"

**Workflow:**
```json
{ "start": "swap_1", "nodes": [
  { "id": "swap_1", "type": "action", "action": "swap", "protocol": "Pendle", "chain": "Ethereum", "inputs": { "asset": "USDC", "amount": "100%" }, "outputs": { "asset": "PT-apyUSD" }, "parameters": { "market": "apyUSD (18 Jun 2026)" }, "next": "supply_1" },
  { "id": "supply_1", "type": "action", "action": "supply", "protocol": "Morpho", "chain": "Ethereum", "inputs": { "asset": "PT-apyUSD", "amount": "100%" }, "outputs": {}, "parameters": { "market": "PT-apyUSD-USDC", "lltv": "86%" }, "next": "borrow_1" },
  { "id": "borrow_1", "type": "action", "action": "borrow", "protocol": "Morpho", "chain": "Ethereum", "inputs": {}, "outputs": { "asset": "USDC" }, "parameters": { "ltv": "75%", "market": "PT-apyUSD-USDC" }, "next": "loop_1" },
  { "id": "loop_1", "type": "loop", "iterations": 5, "entry": "swap_1", "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Positions created:** Morpho collateral (accumulated PT), Morpho debt (accumulated USDC borrow). Net APY = L × PT_yield − (L−1) × borrow_APY.

---

### Flow 4: Supply and Borrow (No Loop)

**Prompt:** "Supply 1 WETH as collateral to Morpho WETH/USDC market on Base, borrow 2000 USDC"

**Workflow:**
```json
{ "start": "supply_1", "nodes": [
  { "id": "supply_1", "type": "action", "action": "supply", "protocol": "Morpho", "chain": "Base", "inputs": { "asset": "WETH", "amount": "1" }, "outputs": {}, "parameters": { "market": "WETH/USDC", "lltv": "86%" }, "next": "borrow_1" },
  { "id": "borrow_1", "type": "action", "action": "borrow", "protocol": "Morpho", "chain": "Base", "inputs": {}, "outputs": { "asset": "USDC" }, "parameters": { "amount": "2000", "market": "WETH/USDC" }, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Positions created:** Morpho collateral (1 WETH), Morpho debt (2000 USDC).

---

### Flow 5: Cross-Chain Strategy

**Prompt:** "Bridge 500 USDC from Ethereum to Base via CCTP, then lend it all to Aave on Base"

**Workflow:**
```json
{ "start": "bridge_1", "nodes": [
  { "id": "bridge_1", "type": "action", "action": "bridge", "protocol": "CCTP", "chain": "Ethereum", "inputs": { "asset": "USDC", "amount": "500" }, "outputs": { "asset": "USDC" }, "parameters": { "destChain": "Base" }, "next": "supply_1" },
  { "id": "supply_1", "type": "action", "action": "supply", "protocol": "Aave", "chain": "Base", "inputs": { "asset": "USDC", "amount": "100%" }, "outputs": { "asset": "aUSDC" }, "parameters": {}, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Positions created:** CCTP bridge (500 USDC in-transit → complete), then Aave supply (500 USDC) on Base.

---

### Flow 6: Exit / Unwind Position

**Prompt:** User clicks "Exit" on their leveraged Morpho position.

**Generated reverse workflow:**
```json
{ "start": "repay_1", "nodes": [
  { "id": "repay_1", "type": "action", "action": "repay", "protocol": "Morpho", "chain": "Ethereum", "inputs": { "asset": "USDC", "amount": "100%" }, "outputs": {}, "parameters": { "market": "PT-apyUSD-USDC" }, "next": "withdraw_1" },
  { "id": "withdraw_1", "type": "action", "action": "withdraw", "protocol": "Morpho", "chain": "Ethereum", "inputs": {}, "outputs": { "asset": "PT-apyUSD" }, "parameters": { "market": "PT-apyUSD-USDC" }, "next": "swap_1" },
  { "id": "swap_1", "type": "action", "action": "swap", "protocol": "Pendle", "chain": "Ethereum", "inputs": { "asset": "PT-apyUSD", "amount": "100%" }, "outputs": { "asset": "USDC" }, "parameters": {}, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Result:** Position closed. User receives USDC back.

---

### Flow 7: Partial Withdrawal

**Prompt:** "Withdraw 50% of my USDC from Aave on Base"

**Workflow:**
```json
{ "start": "withdraw_1", "nodes": [
  { "id": "withdraw_1", "type": "action", "action": "withdraw", "protocol": "Aave", "chain": "Base", "inputs": { "asset": "aUSDC", "amount": "50%" }, "outputs": { "asset": "USDC" }, "parameters": {}, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Position update:** Aave supply reduced by 50%. Reconciliation confirms new balance.

---

### Flow 8: Multi-Protocol Yield Farming

**Prompt:** "Swap 1000 USDC to WETH on LiFi, supply WETH to Aave on Base, borrow 500 USDC against it, lend the USDC to Morpho"

**Workflow:**
```json
{ "start": "swap_1", "nodes": [
  { "id": "swap_1", "type": "action", "action": "swap", "protocol": "LiFi", "chain": "Base", "inputs": { "asset": "USDC", "amount": "1000" }, "outputs": { "asset": "WETH" }, "parameters": {}, "next": "supply_1" },
  { "id": "supply_1", "type": "action", "action": "supply", "protocol": "Aave", "chain": "Base", "inputs": { "asset": "WETH", "amount": "100%" }, "outputs": { "asset": "aWETH" }, "parameters": {}, "next": "borrow_1" },
  { "id": "borrow_1", "type": "action", "action": "borrow", "protocol": "Aave", "chain": "Base", "inputs": {}, "outputs": { "asset": "USDC" }, "parameters": { "amount": "500" }, "next": "lend_1" },
  { "id": "lend_1", "type": "action", "action": "lend", "protocol": "Morpho", "chain": "Base", "inputs": { "asset": "USDC", "amount": "100%" }, "outputs": { "asset": "mUSDC" }, "parameters": {}, "next": "end" },
  { "id": "end", "type": "result" }
]}
```

**Positions created:** Aave collateral (WETH), Aave debt (500 USDC), Morpho supply (500 USDC).

---

### Flow 9: User Exits Externally (Reconciliation Catches It)

**Scenario:** User opened a position through our platform, then repays directly on app.morpho.org.

**What happens:**
1. No event from our contract — we didn't initiate the action.
2. Next reconciliation cycle (within 60s): reads on-chain position → debt = 0, collateral = 0.
3. Position marked CLOSED in DB.
4. Dashboard reflects the exit.

No webhook required. On-chain read is the truth.

---

### Flow 10: Failed Execution (Atomic Revert)

**Scenario:** User submits a 5x loop. On loop 3, slippage exceeds tolerance. Transaction reverts.

**What happens:**
1. Orchestrator contract reverts atomically — entire tx fails, nothing changes on-chain.
2. Backend receives tx hash → checks receipt → status = reverted.
3. Strategy marked FAILED in DB. No positions written.
4. User notified: "Strategy failed: swap slippage exceeded on loop 3."
5. User's original funds are untouched in their wallet.

---

## Delivery Phases

| Phase | Scope | Outcome |
|-------|-------|---------|
| 1 | Workflow Compiler (LLM) + Validator | Prompt → validated JSON, testable without contracts |
| 2 | Simulation integration | Workflow JSON → Tenderly fork → projected position + APY preview |
| 3 | Orchestrator contract | Executes validated workflows atomically, emits events |
| 4 | Position Tracker (MVP) | Post-tx position writes + periodic on-chain reconciliation |
| 5 | Net APY Calculator | Leveraged loop APY using APY service + position state |
| 6 | Exit Builder | Reverse workflow generation + execution through same pipeline |
| 7 | Dashboard | Portfolio view (our DeBank) with live positions, APY, health |

---

## Non-Goals (MVP)

- Historical APY charts
- Social/copy-trading features
- Protocol governance integration
- Limit orders / conditional execution
- Multi-sig support
- Auto-rebalancing
- Webhook-based event ingestion (deferred to Phase 2 of position tracking)
