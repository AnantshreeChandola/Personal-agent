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
  IN --> REG[Plugin Registry]
  REG --> SEL[Select Plugins]
  SEL --> PLAN[Planner (dry_run plan)]
  PLAN --> SIG[Signer (Ed25519)]
  SIG --> PREV[Preview Orchestrator (n8n, read-only)]
  PREV -->|Preview card + evidence| U
  U -->|Approve| GATE[Approval Gate]
  GATE --> EXE[n8n Execute (short jobs)]
  GATE --> DUR[Temporal Durable Orchestrator (long jobs)]
  EXE --> ADP[Adapters / Activities]
  DUR --> ADP
  ADP --> AUD[Audit & Metrics]
  ADP --> PW[PlanWriter → Plan Library + History]
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
- **Separation of concerns:** Orchestrators orchestrate; **Adapters/Activities** perform side-effects behind JSON-schema’d contracts.  
- **Durability & parallelism:** Long jobs and multi-agent roles execute in **Temporal** with signals, idempotency, and compensation.  
- **Auditability:** Correlate everything on `plan_id`; store outcomes and derived facts, not raw private content.

---

## 2) Canonical Contracts (Source of Truth)

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

### 2.3 Plan (deterministic; steps labeled for runtime + role)
~~~json
{
  "plan_id": "<ulid>",
  "intent": {},
  "graph": [
    {
      "step": 1,
      "mode": "interactive|durable",
      "role": "Watcher|Resolver|Booker|Notifier",
      "uses": "<plugin_id>",
      "call": "<operation>",
      "args": {},
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

### 2.6 Execute Wrapper (side-effects via adapters/activities)
~~~json
{
  "provider": "<adapter>",
  "result": { "id": "<external_id>", "link": "<optional>" },
  "status": "created|updated|skipped|error"
}
~~~

### 2.7 Approval Token (binds user + plan hash, short TTL)
~~~json
{
  "token": "<jwt|ulid>",
  "plan_hash": "<sha256>",
  "user_id": "<uuid>",
  "exp": "<iso>"
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
- **Purpose:** Convert user text + channel metadata into an Intent.  
- **Responsibilities:** classification, entity extraction, tz/user binding, early safety checks.  
- **I/O:** text → **Intent** (see §2.1).  
- **Notes:** may call ContextRAG for minor lookups (optional).

### 4.2 ContextRAG (`components/ContextRAG/`)
- **Purpose:** Curate just-enough context under a tier budget.  
- **Responsibilities:** per-tier allowlists; fetch from **Profile**, **History**, **Plan Library**, **Vector Index**, Contacts; re-rank (semantic + recency + success + personal-fit); dedupe/compress; emit **`evidence[]`**.  
- **Data:** `schemas/evidence.json`; caches (LRU/TTL); decay/TTL of history; consent flags.

### 4.3 Profile Store / History / Vector Index
- **Profile:** durable prefs & consent flags (forget/export).  
- **History:** normalized outcomes (PII-light) — “what actually happened.”  
- **Vector Index:** semantic recall of exemplars/plans/facts (per-user namespaces).

### 4.4 Plan Library • Plan Retrieval
- **Plan Library:** canonical plans, signatures, outcomes; similarity search.  
- **Plan Retrieval:** propose successful prior plans with scores (hybrid retrieval).

### 4.5 Plugin Registry • Plugin Selector
- **Registry:** tool catalog (JSON Schemas, scopes, examples, safety class).  
- **Selector:** choose minimal, safe tools per Intent (scope minimization, budget/latency).

### 4.6 Planner (Stateless) (`components/Planner/`)
- **Purpose:** Deterministically produce a plan graph (dry-run), labeling each step with `{mode, role}`.  
- **Responsibilities:** consume **Intent + evidence + registry**; emit **plan** + optional `needs[]` for low-confidence gaps.  
- **Notes:** no datastore access; no side-effects; pure function of inputs.

### 4.7 Signer (`components/Signer/`)
- **Purpose:** Ed25519 sign/verify over the canonical plan.  
- **Responsibilities:** key storage (KMS), rotation, audit, verification at Preview/Execute entry points.

### 4.8 Preview Orchestrator (n8n) (`components/PreviewOrchestrator/`)
- **Purpose:** Execute plan **read-only** to produce a Preview wrapper.  
- **Responsibilities:** verify signature; call only read endpoints/adapters; assemble **Preview** (`source:"preview"`, `can_execute`); include evidence.

### 4.9 Approval Gate (`components/Approvals/`)
- **Purpose:** Present Preview, collect approval, issue **approval_token**.  
- **Responsibilities:** TTL and single-use guarantees; revocation; bind token to `plan_hash` + `user_id`.

### 4.10 Execute Orchestrator (n8n, short jobs) (`components/ExecuteOrchestrator/`)
- **Purpose:** Execute approved **interactive** steps (`mode:"interactive"`).  
- **Responsibilities:** verify signature + token; call write adapters; collect **Execute** wrappers; notify; audit.

### 4.11 Durable Orchestrator (Temporal, long jobs) (`components/DurableOrchestrator/`)
- **Purpose:** Run **durable** steps (`mode:"durable"`) over days/weeks.  
- **Capabilities:**  
  - **Workflows** (deterministic; no I/O).  
  - **Activities** (your adapters; retries, rate limits, **idempotency keys**, compensation).  
  - **Child workflows** (parallel fan-out).  
  - **Signals** (`approval`, `cancel`, `reconfigure`).  
  - **Queries** (`status` for UI).  
  - **Timers** (durable sleep/backoff), **ContinueAsNew** (truncate history).  
  - **Search attributes**: index by `user_id`, `plan_id`, `intent`.

### 4.12 Adapters / Activities (`components/Adapters/*`)
- **Purpose:** Provider-specific integrations (Calendar, People, Messaging, WebAutomation, Payments).  
- **Responsibilities:** JSON-schema’d I/O; least-privilege tokens; PII-safe logs; **idempotency** on writes; retries/backoff; compensations.

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

## 6) End-to-End Examples

### 6.1 Simple (interactive) — “Arrange a meeting next Tuesday with X”
1. **Intake** → Intent (entities: attendee_hint “X”, date), tz, user_id, `context_budget=2`.  
2. **ContextRAG** (Tier-2/3) → evidence: duration 30m, Tue 10–12 window, X’s email (confidence 0.88).  
3. **Planner** → plan (`mode:"interactive"` steps only); **Signer** signs.  
4. **Preview (n8n, read-only)** → card: title/date/proposed 10:30–11:00; evidence shown; `can_execute:true`.  
5. **Approval** → token issued.  
6. **Execute (n8n)** → CalendarAdapter.create_event; **PlanWriter** + **Audit**; ContextRAG learns the successful window.

### 6.2 Long-running (durable multi-agent) — “Watch visa slots 45 days; auto-book earliest morning”
- **Planner** emits a plan with roles:
  ~~~json
  {
    "graph": [
      { "step": 1, "mode": "durable", "role": "Watcher", "uses": "VisaAdapter", "call": "watch_slots", "args": { "consulates": ["IN-DEL","IN-MUM"], "window_days": 45, "hold_minutes": 10 } },
      { "step": 2, "mode": "interactive", "role": "Resolver", "uses": "Approvals", "call": "request_user_approval" },
      { "step": 3, "mode": "durable", "role": "Booker", "uses": "VisaAdapter", "call": "book", "args": { "idempotency_key": "plan:<id>:slot" } },
      { "step": 4, "mode": "durable", "role": "Notifier", "uses": "MessagingAdapter", "call": "notify" }
    ]
  }
  ~~~
- **n8n** starts **Temporal** workflow `VisaSlotWatch(plan_id, params)` (Watcher).  
- Watcher fans out to child workflows per consulate; polls with backoff; on slot → **place_hold** (activity) → **signal** n8n for approval (Resolver).  
- On approval signal → Booker runs **book** (idempotent); Notifier sends confirmation; **PlanWriter** persists; **History** learns preferred windows.

---

## 7) Safety, Reliability, Governance

- **Preview vs Execute** is a hard wall; Preview never mutates providers.  
- **Signatures** verified at Preview and Execute entry points.  
- **Approval token** required before any write.  
- **Idempotency** on every write activity (`plan_id:step:arg_hash`).  
- **Compensation** paths for holds/carts/resources (Saga).  
- **Rate limits** centralized in activities (token bucket).  
- **Privacy:** Tier-5 requires consent; store only derived facts; TTL/forget/export supported.  
- **Determinism:** workflows are pure; all I/O in activities.

---

## 8) Non-Functional Baselines

- Preview p95 < **800 ms**; short Execute steps p95 < **2 s**.  
- Durable workflows survive restarts; use **ContinueAsNew** every 24 h or N events.  
- Availability: 99.9% (Intake/Preview), 99.5% (Execute/Durable).  
- Observability: traces across plan/preview/execute; searchable by `plan_id`, `user_id`, `intent`.

---

## 9) Repository Mapping (Per Component Packet)

Each `components/<Name>/` includes: `SPEC.md`, `LLD.md`, `schemas/`, `tests/`, code.

- **Intake, ContextRAG, Planner, Signer, PreviewOrchestrator, ExecuteOrchestrator, DurableOrchestrator, PluginRegistry, PlanLibrary, PlanRetrieval, Adapters/*, PlanWriter, Audit**.  
- **GLOBAL_SPEC.md**: Intent, Evidence, Preview/Execute, **plan step `{mode, role}`** schema.

---

## 10) Next Steps

1. Add `{ "mode", "role" }` to plan-step schema in **GLOBAL_SPEC.md**.  
2. Create `components/DurableOrchestrator/` (SPEC/LLD): workflows, signals (`approval|cancel|reconfigure`), queries (`status`), child-workflow patterns, ContinueAsNew policy.  
3. Wrap first adapters as **Temporal activities** (Calendar/People/Messaging) with JSON Schemas + idempotency + tests.  
4. Stand up two n8n flows (**Preview**, **Execute**) and wire a Temporal starter (Watcher sample).  
5. Seed ContextRAG with 3 prefs + 1 pattern; wire **PlanWriter** write-back.
