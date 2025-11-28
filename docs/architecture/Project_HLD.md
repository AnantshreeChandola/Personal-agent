# Personal Agent — High-Level Architecture (HLD) v3.4
_Context-aware • Preview-first • Signed plans • Human-approved • **Durable multi-agent** execution_

**Purpose:** Single source of truth for the system’s components, contracts, and responsibilities.  
**Scope:** Conceptual architecture only — **no API routes** here. Each component owns its own SPEC/LLD.

---

## 0) Architecture at a Glance

~~~mermaid
flowchart LR
 U[User] --> IN[Intake & Reason]
  IN --> RAG((ContextRAG: prefs/history/exemplars))
  IN --> PLIB[Plan Library]
  PLIB --> RET[Retrieve & Score Plans]
  IN --> REG[Plugin Registry- n8n bindings]
  REG --> SEL[Select Tools]
  SEL --> PLAN[Planner dry_run plan]
  PLAN --> SIG[Signer Ed25519]
  SIG --> PREV[Preview Orchestrator n8n, read-only]
  PREV -->|Preview card + evidence| U
  U -->|Approve Gate A/B/…| GATE[Approval Gates]
  GATE --> EXE[n8n Execute short jobs via connectors]
  GATE --> DUR[Temporal Durable Orchestrator long jobs]
  EXE --> BIND[Binding Resolver plan → n8n node params]
  EXE --> CONN[n8n Connector Nodes GCal/Gmail/HTTP/Slack…]
  DUR --> ACT[Temporal Activities HTTP/SDK calls as needed]
  CONN --> AUD[Audit & Metrics]
  ACT --> AUD
  CONN --> PW[PlanWriter → Plan Library + History]
  ACT --> PW
  AUD --> U
  PW --> RAG
~~~

**Two runtimes**
- **n8n** — interactive glue (Preview card, approvals, short synchronous steps).  
- **Temporal** — **durable engine** for long-running/stateful work (poll/sleep/retry/fan-out/signals/compensation).

---

## 1) Core Principles

- **Preview-first safety:** Every flow produces a **side-effect-free Preview**; writes happen **only after** explicit approval.  
- **Deterministic planning:** Planner is **stateless**; the plan is canonicalized and **Ed25519-signed** before execution.  
- **Context engineering:** Fetch **just enough** context as tiny, typed **`evidence[]`** (tiers), never raw blobs.  
- **Separation of concerns:** Orchestrators orchestrate; n8n connectors perform I/O; Temporal adds durability.  
- **Auditability:** Correlate everything on `plan_id`; store outcomes and derived facts, not raw private content.

---

## 2) Canonical Contracts (Source of Truth)

### 2.0 Deterministic Inputs (Planner)
The Planner is a pure function of this frozen tuple:
- **Intent vN** (finalized, versioned)
- **Evidence vK** (from ContextRAG; small, typed)
- **Registry vR** (connector catalog snapshot)
- **Policy vC** (GLOBAL_SPEC/config version)
Same tuple ⇒ same canonical plan bytes ⇒ same hash/signature.

### 2.1 Intent (Intake → Planner)
~~~json
{
  "intent": "<action>",
  "entities": {},
  "constraints": {},
  "tz": "America/Chicago",
  "user_id": "<uuid>",
  "context_budget": 1
}
~~~

### 2.2 Evidence Item (ContextRAG → Planner; small & typed)
~~~json
{
  "type": "preference|history|contact|plan|exemplar",
  "key": "meeting_duration_min",
  "value": 30,
  "confidence": 0.82,
  "source_ref": "kv:prefs/duration",
  "ttl_days": 365
}
~~~

### 2.3 Plan (deterministic; supports HITL gates)
~~~json
{
  "plan_id": "<ulid>",
  "intent": {},
  "graph": [
    {
      "step": 1,
      "mode": "interactive|durable",
      "role": "Fetcher|Analyzer|Watcher|Resolver|Booker|Notifier",
      "uses": "<tool_id>",
      "call": "<operation>",
      "args": {},
      "after": [/* deps, optional */],
      "gate_id": "gate-A",
      "dry_run": true
    }
  ],
  "constraints": { "scopes": ["calendar.write"], "ttl_s": 900 },
  "plugins": ["<plugin_id>"],
  "meta": { "created_at": "<iso>", "author": "planner" }
}
~~~

### 2.4 Plan Signature (Ed25519 over canonicalized plan)
~~~json
{
  "algo": "Ed25519",
  "signer": "planner@system",
  "ts": "<iso>",
  "nonce": "<ulid>",
  "signature": "<base64>",
  "pubkey_id": "k1"
}
~~~

### 2.5 Preview Wrapper (read-only)
~~~json
{
  "normalized": {},
  "source": "preview",
  "can_execute": true,
  "evidence": []
}
~~~

### 2.6 Execute Wrapper (connector result)
~~~json
{
  "provider": "<connector_id>",
  "result": { "id": "<external_id>", "link": "<optional>" },
  "status": "created|updated|skipped|error"
}
~~~

### 2.7 Approval Token (binds user + plan hash; supports multi-gate)
~~~json
{
  "token": "<jwt|ulid>",
  "plan_hash": "<sha256>",
  "user_id": "<uuid>",
  "exp": "<iso>",
  "scopes": ["shopping.write"]
}
~~~

---

## 3) Context Tiers (Policy, not Model Feature)

- **Tier 1** — Session only (utterance, parsed Intent).  
- **Tier 2** — Stable preferences (work hours, duration, VC/location).  
- **Tier 3** — Recent history/patterns (e.g., “Tue 10–12 works with X”).  
- **Tier 4** — Live signals/cross-app (free/busy, People API).  
- **Tier 5** — Private content (derived facts only; explicit consent).

**ContextRAG** enforces the tier budget and emits ≤ ~2 KB of **`evidence[]`**; the Planner never pulls raw memory.

---

## 4) Components (What Each Does)

### 4.1 Intake & Reason (`components/Intake/`)
High-level goal shaping across turns; maintains a minimal Session/Goal summary and a Readiness Gate (when to plan). Emits Intent.

### 4.2 ContextRAG (`components/ContextRAG/`)
Curates just-enough context from Profile, History, Plan Library, Vector Index, Contacts; re-ranks and emits typed evidence[].

### 4.3 Profile Store / History / Vector Index
- **Profile:** durable prefs & consent flags (forget/export).  
- **History:** normalized outcomes (PII-light) — “what actually happened.”  
- **Vector Index:** semantic recall of exemplars/plans/facts (per-user namespaces).

### 4.4 Plan Library • Plan Retrieval
- **Plan Library:** canonical plans, signatures, outcomes; similarity search.  
- **Plan Retrieval:** propose successful prior plans with scores (hybrid retrieval).
- Ranks candidates via hybrid score (BM25 + vector + success + recency). Returns a short “why” rationale and suggested seeds.


### 4.5 Plugin Registry (n8n-aware) • Tool Selector
- **Registry:** Source of truth for logical tools → n8n connector bindings (node name/op, param maps, previewability, safety class, scopes, idempotency hints).
- **Selector:** choose minimal set of tools/ops per Intent (scope minimization, latency/cost/health).

### 4.6 Planner (Stateless) (`components/Planner/`)
- Deterministically emits the plan graph, labeling steps with {mode, role} and inserting HITL approval steps (with gate_id) where needed.

### 4.7 Signer (`components/Signer/`)
- Ed25519 sign/verify over the canonical plan; key rotation & audit; verification happens at Preview/Execute entry.

### 4.8 Preview Orchestrator (n8n) (`components/PreviewOrchestrator/`)
- Verifies signature, filters to previewable ops from the Registry, runs connectors (read-only) via Binding Resolver, and returns a Preview wrapper.

### 4.9 Approval Gate (`components/Approvals/`)
- Presents Preview/choices; on Approve, issues an approval token bound to {plan_hash, gate_id, user_id} with short TTL and single-use semantics.

### 4.10 Execute Orchestrator (n8n, short jobs) (`components/ExecuteOrchestrator/`)
- Verifies signature and the gate's approval token
- **Uses WorkflowBuilder** to convert plan → n8n workflow with parallel structure
- Executes n8n workflow (handles parallel branches internally based on `after` dependencies)
- For each step:
  - mode:"interactive" → already in workflow, executes via n8n parallel branches
  - mode:"durable" → hand off to Temporal (start workflow), await signal
- Enforces idempotency (Data Store: plan_id:step:arg_hash)
- Collects Execute wrappers, notifies user, and calls PlanWriter

### 4.11 Binding Resolver (components/BindingResolver/)
- Small mapping layer (can live inside n8n or as a helper service):
    {uses, call, args} + Registry vR → {node, operation, params} for the n8n node. Keeps flows generic; adding capabilities means editing the Registry, not flows.

### 4.12 WorkflowBuilder (`components/WorkflowBuilder/`)

**Purpose:** Convert plan dependency graph → n8n workflow JSON with parallel execution structure.

**Responsibilities:**
- Parse plan `graph[]` and analyze `after` dependencies
- Identify parallel-executable steps (no dependencies or shared dependencies)
- Generate n8n workflow with Split/Merge nodes for parallel branches
- Convert steps to n8n nodes (via Binding Resolver)
- Handle approval gates as n8n Wait nodes

**Input:** Signed plan + Registry snapshot
**Output:** n8n workflow JSON ready for execution

**Integration:** Execute Orchestrator uses WorkflowBuilder before running n8n workflows.

**Details:** See `components/WorkflowBuilder/LLD.md` (created during implementation phase)

### 4.13 Durable Orchestrator (Temporal, long jobs) (`components/DurableOrchestrator/`)
- Durable workflows for watchers/bookers/notifiers over days/weeks: deterministic core, Activities for I/O, retries/backoff, signals (approve/cancel/reconfigure), queries (status), ContinueAsNew, search attributes (user_id, plan_id, intent).

### 4.14 PlanWriter (`components/PlanWriter/`)
- **Purpose:** Persist outcomes back to **Plan Library** & **History**; trigger vector re-index.
- **Responsibilities:** map Execute wrappers → normalized facts; **idempotent** writes.

### 4.15 Audit & Observability (`components/Audit/`)
- **Purpose:** Structured logs, metrics, traces; user/system dashboards.
- **Responsibilities:** correlate by `plan_id`; SLOs (preview p95, execute p95), retries, error classes; summaries.

### 4.16 (Optional) MemoryHub Façade (`components/MemoryHub/`)
- **Purpose:** One door for memory ops (useful for n8n/tests).  
- **Endpoints (conceptual):** `GET evidence`, `GET/PUT prefs`, `POST fact`, `forget/export`.  
- **Note:** delegates to ContextRAG/Profile/History/Plan Library.

---

## 5) Runtime Agent Roles & Asynchronous Orchestration

### 5.1 Runtime Agent Model

**Runtime agents** are asynchronous execution instances that execute plan steps. Not just semantic labels—actual workers (n8n sub-workflows or Temporal activities) that:
- Execute independently and asynchronously
- Report completion to orchestrator
- Can run multiple instances concurrently
- Are classified by role for responsibility isolation

**Agent lifecycle:** spawn (per step) → execute → report → terminate

### 5.2 Six Agent Roles (Responsibility Classification)

- **Fetcher** — One-time read operations
  - Examples: Preview fetches, API calls, calendar slots, contact lookup
  - Implementation: n8n HTTP/connector nodes (interactive), Temporal activities (durable)

- **Analyzer** — Data processing, comparison, research
  - Examples: Compare flight options, rank restaurants, analyze expenses, find overlaps
  - Implementation: n8n Function/Code nodes, Temporal activities with compute logic

- **Watcher** — Long-running monitoring
  - Examples: Poll visa slots for days, monitor price drops, watch for emails
  - Implementation: Temporal workflows with ContinueAsNew, poll activities

- **Resolver** — Disambiguation, user interaction
  - Examples: "Which John?", choose from options, clarify preferences
  - Implementation: n8n Wait nodes with webhooks, interactive approval flows

- **Booker** — Writes with idempotency
  - Examples: Create calendar events, send emails, make purchases, book appointments
  - Implementation: n8n connector nodes (GCal, Gmail), Temporal activities with idempotency keys

- **Notifier** — Updates and alerts
  - Examples: Confirm booking, send summaries, progress updates, error notifications
  - Implementation: n8n Slack/email nodes, Temporal notification activities

### 5.3 Asynchronous Execution Model

**Plan-level concurrency:**
- Multiple plans execute simultaneously (different `plan_id` values)
- User A books meeting while User B monitors visa slots
- No blocking between independent plans

**Step-level concurrency (within a plan):**
- Steps with `after: []` execute immediately in parallel
- Steps with `after: [1, 2]` wait for both dependencies, then execute
- Dependency graph enables DAG-based parallel execution

**Example parallel execution:**
~~~json
{
  "graph": [
    {"step": 1, "role": "Fetcher", "call": "get_alice_calendar", "after": []},
    {"step": 2, "role": "Fetcher", "call": "get_bob_calendar", "after": []},
    {"step": 3, "role": "Analyzer", "call": "find_overlap", "after": [1, 2]},
    {"step": 4, "role": "Booker", "call": "create_event", "after": [3]}
  ]
}
~~~
Steps 1 and 2 execute in parallel; step 3 waits for both; step 4 runs after step 3.

### 5.4 Orchestration Mechanisms

**n8n (interactive mode):**
- WorkflowBuilder converts plan → n8n workflow JSON
- Parallel branches for steps with no dependencies
- Merge nodes wait for all branches to complete
- Split/Merge pattern enables concurrent execution

**Temporal (durable mode):**
- Child workflows spawned per step
- Fan-out pattern for parallel execution
- Signals for coordination and approval
- Activities for actual I/O operations

### 5.5 Coordination Rules

- Planner tags each step with `{mode, role}` for classification
- Orchestrators spawn agent instances per step (n8n sub-workflows or Temporal activities)
- Dependency graph (`after` field) determines execution order
- **Resource locks** prevent conflicting writes:
  - Fine-grained: `calendar.alice.write`, `calendar.bob.write` (can run parallel)
  - Read operations: no locks (parallel reads allowed)
  - Coarse locks only for rate-limited resources (e.g., `email.send`)
- **Saga pattern** for compensation on failure
- **Idempotency** enforced via `plan_id:step:arg_hash` keys
- Confidence below threshold → emit `needs[]` → n8n asks user or escalates Context tier → re-plan/resume

---

## 6) Human-in-the-Loop (HITL) Pattern (Multi-Gate)

Insert explicit approval steps with unique gate_id (“gate-A”, “gate-B”…).
Each gate issues a token bound to {plan_hash, gate_id, user_id}.
Subsequent write steps must present the matching gate token or they won’t run.
Typical shopping flow: shortlist → Gate A (choose) → add to cart → Gate B (final review) → purchase.

## 7) Dynamic Mapping to n8n Connectors

Registry stores connector bindings for each logical uses/call: node name, operation, arg/param maps, previewability, scopes, idempotency hints.
Binding Resolver turns plan steps into ready-to-run n8n node parameters.
n8n flows remain generic (Preview/Execute only); adding capabilities is a Registry edit, not a flow change.

## 8) Safety, Reliability, Governance

Signature verified at Preview/Execute; approval tokens required for writes (and per-gate where applicable).
Preview never mutates; only Registry-marked previewable ops run there.
Idempotency on every write (plan_id:step:arg_hash in n8n Data Store/DB).
Compensation when supported (cancel/delete ops declared in Registry).
Privacy: Tier-5 gated by consent; store derived facts; TTL/forget/export supported.
Observability: per-step logs with plan_id, op, latency_ms, retries, status.

## 9) Non-Functional Baselines

Preview p95 < 800 ms; short Execute steps p95 < 2 s.
ContextRAG p95 < 150 ms; Plan Retrieval p95 < 200 ms; Vector top-K < 100 ms.
Durable flows survive restarts; ContinueAsNew daily or on N events.
Availability: 99.9% (Intake/Preview), 99.5% (Execute/Durable).

## 10) Repository Mapping (Per Component Packet)

Each components/<Name>/ includes: SPEC.md, LLD.md, schemas/, tests/, code.
Additionally, usecases/<UseCase>/ includes: SPEC.md, LLD.md, plans/, tests/, fixtures/.
Intake, ContextRAG, Planner, Signer, PreviewOrchestrator, ExecuteOrchestrator, DurableOrchestrator, PluginRegistry, BindingResolver, PlanLibrary, PlanRetrieval, PlanWriter, Audit.
GLOBAL_SPEC.md: Intent, Evidence, Preview/Execute wrappers, plan step schema with {mode, role, after, gate_id}.

## 11) End-to-End Examples (Conceptual)
- 11.1 Meeting (interactive)

Intent → ContextRAG → Plan (list_free_busy → Gate A approve slot → create_event) → Preview (read) → Gate A token → Execute (write) → PlanWriter/Audit.

- 11.2 Shopping (HITL multi-gate)

Search/rank → Gate A (choose shortlist) → add_to_cart → Gate B (cart review) → purchase.
Each write phase requires the matching gate token; idempotency avoids duplicates.

- 11.3 Visa watcher (durable)

Watchers in Temporal; on slot hold → signal Gate for approval; on approve, book; notify; write-back.

- 11.4 Parallel execution (interactive with dependencies)

**Intent:** "Find meeting time with Alice and Bob"

**Plan:**
~~~json
{
  "plan_id": "01HX...",
  "graph": [
    {"step": 1, "mode": "interactive", "role": "Fetcher",
     "uses": "google.calendar", "call": "list_free_busy",
     "args": {"user": "alice@example.com"}, "after": []},
    {"step": 2, "mode": "interactive", "role": "Fetcher",
     "uses": "google.calendar", "call": "list_free_busy",
     "args": {"user": "bob@example.com"}, "after": []},
    {"step": 3, "mode": "interactive", "role": "Analyzer",
     "uses": "system.analyze", "call": "find_overlap",
     "args": {}, "after": [1, 2]},
    {"step": 4, "mode": "interactive", "role": "Resolver",
     "uses": "system.approval", "call": "confirm_slot",
     "gate_id": "gate-A", "after": [3]},
    {"step": 5, "mode": "interactive", "role": "Booker",
     "uses": "google.calendar", "call": "create_event",
     "args": {}, "after": [4]}
  ]
}
~~~

**Execution flow:**
1. WorkflowBuilder converts plan → n8n workflow
2. Steps 1 and 2 execute in parallel (n8n Split → [Fetch Alice || Fetch Bob] → Merge)
3. Step 3 executes after merge (Analyzer finds overlapping slots)
4. Step 4 presents options to user (Wait node for Gate A approval)
5. Step 5 creates event after approval (Booker with idempotency)

**Timeline:**
- t=0ms: Steps 1 & 2 start simultaneously
- t=200ms: Both complete, step 3 starts
- t=350ms: Step 3 completes, step 4 shows preview to user
- t=user: User approves (Gate A token issued)
- t=user+50ms: Step 5 creates event

---
