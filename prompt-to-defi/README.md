# Prompt-to-DeFi Agent

Natural-language → simulated → executable DeFi strategies. Describe a goal in plain
English; an LLM turns it into a structured plan; deterministic code builds the
transactions; Tenderly simulates them; the user signs from their own wallet.

**Non-custodial. Simulate-first. Plugin-based.** The agent decides *what* to do; it
never holds keys and never hand-rolls calldata at execution time.

---

## 1. Core principles

These are non-negotiable and shape every design decision below.

| Principle | What it means in practice |
| --- | --- |
| **Non-custodial** | Backend builds *unsigned* transactions. Only the user's wallet signs. Keys never touch the server. |
| **Simulate-first** | Every plan is simulated (Tenderly for EVM, venue API for non-EVM) before anything is signed. Failures are caught before money moves. |
| **Deterministic execution** | The LLM only *selects tools and fills typed params*. All calldata is built by deterministic code — no AI in the execution path (eliminates hallucination at the money layer). |
| **Confirm gate** | A hard boundary between "simulated" and "executed". Two separate endpoints: `/plan` (+`/simulate`) and `/execute`. |
| **Plugin-per-protocol** | Every protocol action implements one `Tool` contract. Adding Aave/Uniswap/Hyperliquid is a drop-in plugin — core code never changes (Open/Closed). |
| **Execution-kind agnostic** | Tools produce *artifacts*, not "transactions". EVM calldata and non-EVM signed actions (e.g. Hyperliquid) coexist behind the same interface. |

---

## 2. Architecture

```
USER PROMPT
   │
   ▼
┌──────────────┐
│ Planner (LLM)│  OpenAI function-calling → structured Plan (ordered tool calls + args)
└──────┬───────┘
       ▼
┌──────────────┐
│  Resolver    │  discovery: best-APY vault, swap/bridge routes, names → addresses
└──────┬───────┘
       ▼
┌──────────────┐
│  Validator   │  LTV / health-factor math, balances, slippage bounds — rejects bad plans early
└──────┬───────┘
       ▼
┌──────────────┐
│  Tx Builder  │  per-tool build() → ExecutionArtifact[]  (EVM bundle, EVM tx, or signed action)
└──────┬───────┘
       ▼
┌──────────────┐
│ Sim Router   │  dispatches each artifact to the correct simulator, merges into ONE preview
└──────┬───────┘
       ▼
   PREVIEW CARD  ── user reviews ──► [ Confirm ]
       ▼
┌──────────────┐
│  Executor    │  hands unsigned tx → WALLET SIGNS → broadcast; polls status between cross-chain legs
└──────────────┘
```

### The two-phase lifecycle

**Phase 1 — Plan & Simulate (no signature):**
1. `POST /plan` with `{ prompt, walletAddress, chainId }`.
2. Planner extracts a `Plan` (ordered `ToolCall[]`).
3. Resolver fills concrete addresses / routes / discovery.
4. Validator checks balances + LTV math, rejects impossible plans.
5. Tx Builder produces `ExecutionArtifact[]`.
6. Sim Router simulates (impersonating the user — no signature needed on a fork) and returns a `Preview`.

**Phase 2 — Confirm & Execute (user signs):**
7. User clicks Confirm → `POST /execute`.
8. Backend returns the unsigned artifact(s); the wallet signs and broadcasts.
9. Same-chain bundled actions = **1 signature**. Cross-chain = a guided sequence; the executor polls bridge status before handing over the next leg.

---

## 3. Core contracts

The entire extensibility model rests on two interfaces. **Core depends on these
interfaces — never on a concrete protocol.**

```ts
// core/tool.ts

export type Capability =
  | "LEND" | "BORROW" | "SUPPLY_COLLATERAL" | "MULTIPLY"
  | "SWAP" | "BRIDGE" | "LP" | "PERP" | "STAKE";

export type UnsignedTx = {
  to: `0x${string}`;
  data: `0x${string}`;
  value: bigint;
  chainId: number;
};

// build() returns these. Discriminated union keeps the system open to new execution kinds.
export type ExecutionArtifact =
  | { kind: "approval";     chainId: number; tx: UnsignedTx }
  | { kind: "evmTx";        chainId: number; tx: UnsignedTx }            // single contract call (Aave, Compound, Uniswap)
  | { kind: "evmBundle";    chainId: number; tx: UnsignedTx }           // atomic multi-action (Morpho bundler)
  | { kind: "signedAction"; venue: string;   typedData: unknown };      // non-EVM (Hyperliquid): sign + POST

// Every protocol action implements THIS. Nothing in core knows about Morpho/Aave/etc.
export interface Tool<TArgs, TResolved = TArgs> {
  id: string;                  // "morpho.lend", "aave.supply", "hyperliquid.openPerp"
  capability: Capability;
  chains: number[];
  description: string;         // helps the planner choose
  paramSchema: ZodSchema<TArgs>;

  resolve(args: TArgs, ctx: Ctx): Promise<TResolved>;          // discovery, names → addresses, quotes
  validate(args: TResolved, ctx: Ctx): Promise<void>;         // LTV/health/balance/slippage — throw on failure
  build(args: TResolved, ctx: Ctx): Promise<ExecutionArtifact[]>;
  preview(args: TResolved, sim: SimResult): PreviewCard;      // human-readable summary
}
```

```ts
// simulators/router.ts

export interface Simulator {
  supports(kind: ExecutionArtifact["kind"]): boolean;
  simulate(artifacts: ExecutionArtifact[], ctx: Ctx): Promise<SimResult>;
}

// Registered simulators:
//   TenderlySimulator    → evmTx | evmBundle | approval  (fork + impersonate + state diff)
//   HyperliquidSimulator → signedAction                  (analytical: fill/margin/liq from venue API)
//   CrossChainSimulator  → splits a multi-chain plan, simulates each leg, stitches results
//
// SimulationRouter.route():
//   1. group artifacts by (simulator, chain)
//   2. EVM same-chain  → ONE Tenderly bundle (step 2 sees step 1's state)
//   3. cross-chain     → per-leg sims
//   4. non-EVM venues  → venue simulator
//   5. merge into a single Preview
```

```ts
// core/types.ts

export type Plan = {
  planId: string;
  chainId: number;
  walletAddress: `0x${string}`;
  steps: ToolCall[];                 // ordered
};

export type ToolCall = { tool: string; args: unknown };  // args validated against the tool's paramSchema

export type Preview = {
  planId: string;
  humanSummary: string;
  resolved: Record<string, unknown>; // concrete choices, shown for transparency
  simulation: {
    success: boolean;
    revertReason?: string;
    gasEstimate?: string;
    balanceChanges: { token: string; before: string; after: string }[];
    positionAfter?: { collateral: string; debt: string; ltv: number; lltv: number };
  };
  artifacts: ExecutionArtifact[];    // signed in Phase 2
};
```

---

## 4. Project structure

```
src/
  core/
    tool.ts            # Tool interface, Capability, ExecutionArtifact
    registry.ts        # ToolRegistry: register + chain-filtered OpenAI function schemas
    pipeline.ts        # resolve → validate → build → simulate orchestration
    context.ts         # Ctx: wallet, chainId, viem clients, data backbone
    types.ts           # Plan, ToolCall, Preview, SimResult, PreviewCard
  agents/
    planner/           # OpenAI client, zodToOpenAIFunction, system prompt
    resolver/          # discovery: best APY, routes, market lookup
    validator/         # shared risk/LTV math
    executor/          # signature orchestration + cross-chain status polling
  simulators/
    tenderly.ts
    hyperliquid.ts
    crosschain.ts
    router.ts
  tools/               # ◄── every protocol is a self-contained plugin
    morpho/   lend.ts  supplyBorrow.ts  multiply.ts
    lifi/     swap.ts  bridge.ts
    aave/                # future
    uniswap/             # future
    compound/            # future
    hyperliquid/         # future
  protocols/           # thin SDK adapters (pin versions, isolate churn)
  api/                 # /plan, /simulate, /execute (Fastify)
  config/              # chains, RPCs, env loading
  index.ts
test/
.env.example
package.json
tsconfig.json
```

---

## 5. Tech stack

| Concern | Choice |
| --- | --- |
| Runtime | Node 20+, TypeScript (strict) |
| Chain I/O | `viem` |
| Morpho | `@morpho-org/blue-api-sdk`, `@morpho-org/blue-sdk(-viem)`, `@morpho-org/bundler-sdk-viem` |
| Swaps / bridges | `@lifi/sdk` |
| LLM | OpenAI (function-calling) + `zod` tool schemas |
| Simulation | Tenderly Simulation API (EVM); venue APIs for non-EVM |
| API | Fastify |
| Validation | `zod` |

> SDK method names drift between versions. Pin versions and wrap each SDK in a thin
> adapter under `protocols/` so version churn stays isolated from the pipeline.

---

## 6. Getting started

### Prerequisites
- Node 20+
- Tenderly account (API key + account/project slug)
- OpenAI API key
- RPC URLs for target chains (Base Sepolia for the MVP)

### Setup
```bash
npm install
cp .env.example .env   # fill in keys
npm run dev
```

### Environment variables (`.env.example`)
```bash
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4o

TENDERLY_ACCESS_KEY=
TENDERLY_ACCOUNT_SLUG=
TENDERLY_PROJECT_SLUG=

# RPCs (MVP targets)
RPC_BASE_SEPOLIA=
RPC_ARBITRUM_SEPOLIA=

PORT=3000
```

---

## 7. API

### `POST /plan`
Request:
```json
{ "prompt": "Lend 1000 USDC to the highest-APY USDC vault on Morpho",
  "walletAddress": "0x...", "chainId": 84532 }
```
Response: a `Preview` (human summary + simulation result + artifacts to sign).

### `POST /simulate`
Re-run simulation for an existing `planId` (e.g. after adjusting slippage). Returns `Preview`.

### `POST /execute`
Request: `{ "planId": "..." }`. Returns the unsigned artifact(s) for the wallet to sign;
streams/polls status for cross-chain legs.

---

## 8. Adding a new tool (the extensibility playbook)

This is the whole point of the architecture. Two cases:

### Case A — EVM protocol that fits the existing simulator (Aave, Compound, Uniswap)
1. Create `tools/aave/supply.ts` implementing `Tool`:
   - `resolve`  → market/reserve lookup via Aave SDK
   - `validate` → health-factor / balance checks
   - `build`    → return `{ kind: "evmTx", ... }`
   - `preview`  → summary card
2. Register it: `registry.register(aaveSupply)`.
3. Done. The planner can now pick it; Tenderly already simulates `evmTx`. **No core changes.**

### Case B — Non-EVM venue needing a new simulator (Hyperliquid)
1. Create `tools/hyperliquid/openPerp.ts`; `build` returns `{ kind: "signedAction", venue: "hyperliquid", ... }`.
2. Create `simulators/hyperliquid.ts` implementing `Simulator` for `signedAction`
   (compute fill / margin / liquidation from the venue API). The router auto-selects it.
3. Register the tool: `registry.register(hyperliquidOpenPerp)`.
4. Done. Every future perp venue reuses the same simulator.

**The rule:** protocols depend on `core`; `core` never depends on a protocol. Keep that
arrow pointing one way and you can add protocols indefinitely.

---

## 9. Build roadmap

| Phase | Scope | Tools |
| --- | --- | --- |
| **0 — MVP** | `/plan → /simulate → /execute` loop on Base Sepolia, non-custodial, Tenderly | `lifi.swap`, `morpho.lend`, `morpho.supplyBorrow`, `morpho.multiply` |
| **1 — Breadth** | More protocols + a real data backbone (APYs, TVL, prices) | `aave.*`, `compound.*`, `uniswap.*` |
| **2 — Cross-chain + agents** | LiFi bridging in plans; split monolith into specialized agents; risk scoring | `lifi.bridge`, perps |
| **3 — Platform** | Strategy templates / creator framework, wallet-based recommendations | — |
| **4 — Automation & hardening** | Position monitoring, auto-deleverage (account abstraction / session keys), audits, mainnet | — |

### Recommended first build order (Phase 0)
1. Skeleton: `core/` interfaces + `registry` + `pipeline` + `SimulationRouter`.
2. `lifi.swap` end-to-end (plan → simulate → preview → execute). Proves the loop.
3. `morpho.supplyBorrow` (single bundled tx) + LTV math.
4. `morpho.lend` + "highest-APY vault" discovery.
5. `morpho.multiply` via bundler + flashloan + embedded LiFi swap.
6. Frontend chat UI + wallet connect.

---

## 10. Security notes

- The backend **never** stores private keys or signs transactions.
- Nothing is broadcast that hasn't passed simulation.
- Validate every LLM-produced `ToolCall` against the tool's `paramSchema` before resolving — the LLM is treated as **untrusted input**.
- Identifiers (vault/market addresses) come from the Resolver/data backbone, never from free-text the LLM invented.
- Enforce slippage bounds and max-LTV guards in `validate()`, server-side, regardless of what the prompt asked for.
- Cross-chain: if a later leg fails after a bridge succeeded, funds rest safely as tokens on the destination chain; the executor re-offers the remaining step.

---

## Phase 0 build target

Build the Phase 0 skeleton in the order listed in §9. It should implement the contracts
above: the `core/` pipeline (resolve → validate → build → simulate), an OpenAI planner,
a Tenderly simulator behind the `SimulationRouter`, and the first plugins (`lifi.swap`,
`morpho.supplyBorrow`, `morpho.lend`) plus an `aave.supply` stub that demonstrates the
zero-core-change extensibility model (README §8). Keep all SDK-specific calls isolated in
`src/protocols/` adapters so version churn never reaches the pipeline.

Suggested dependencies: `viem`, `zod`, `zod-to-json-schema`, `openai`, `fastify`, `dotenv`
(dev: `typescript`, `tsx`, `@types/node`). REST-based integrations (LiFi quote API, Tenderly
simulate-bundle API, Morpho GraphQL API) need no extra SDKs; the Morpho bundler is the one
place a dedicated SDK pays off when you batch actions into a single signature.
