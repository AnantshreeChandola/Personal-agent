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
      "role": "Watcher|Resolver|Booker|Notifier",
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
- Verifies signature and the gate’s approval token; for each step:
- mode:"interactive" → run n8n connector nodes through the Binding Resolver; enforce idempotency (Data Store: plan_id:step:arg_hash).
- mode:"durable" → hand off to Temporal (start workflow), then continue when signaled.
    Collects Execute wrappers, notifies user, and calls PlanWriter.

### 4.11 Binding Resolver (components/BindingResolver/)
- Small mapping layer (can live inside n8n or as a helper service):
    {uses, call, args} + Registry vR → {node, operation, params} for the n8n node. Keeps flows generic; adding capabilities means editing the Registry, not flows.

### 4.11 Durable Orchestrator (Temporal, long jobs) (`components/DurableOrchestrator/`)
- Durable workflows for watchers/bookers/notifiers over days/weeks: deterministic core, Activities for I/O, retries/backoff, signals (approve/cancel/reconfigure), queries (status), ContinueAsNew, search attributes (user_id, plan_id, intent).

### 4.13 PlanWriter (`components/PlanWriter/`)
- **Purpose:** Persist outcomes back to **Plan Library** & **History**; trigger vector re-index.  
- **Responsibilities:** map Execute wrappers → normalized facts; **idempotent** writes.

### 4.14 Audit & Observability (`components/Audit/`)
- **Purpose:** Structured logs, metrics, traces; user/system dashboards.  
- **Responsibilities:** correlate by `plan_id`; SLOs (preview p95, execute p95), retries, error classes; summaries.

### 4.15 (Optional) MemoryHub Façade (`components/MemoryHub/`)
- **Purpose:** One door for memory ops (useful for n8n/tests).  
- **Endpoints (conceptual):** `GET evidence`, `GET/PUT prefs`, `POST fact`, `forget/export`.  
- **Note:** delegates to ContextRAG/Profile/History/Plan Library.

---

## 5) Multi-Agent Model (Roles & Parallelism)

Use **roles**, not heavyweight bespoke LLMs per task:

- **Watcher** — long polling/monitoring (Temporal workflow + read activities).  
- **Resolver** — disambiguation/low-confidence cases (ask user via n8n or escalate context tier).  
- **Booker** — writes with idempotency & compensation (activities).  
- **Notifier** — user/system updates, progress, summaries.

**Coordination rules**
- Planner tags each step with `{mode, role}`.  
- Durable Orchestrator assigns **child workflows/activities** per role; run in **parallel** where safe.  
- Only **one writer** (e.g., Booker) holds a resource write-lock; **Saga** compensates on failure.  
- Confidence below threshold → emit `needs[]` → n8n asks user or escalates Context tier → re-plan/resume.

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
---
