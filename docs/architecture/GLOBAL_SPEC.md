# GLOBAL SPEC — Operating Contract (v2)

**Status:** Active  
**Applies to:** All components in this repository  
**Default timezone:** America/Chicago

---

## 0) Purpose
Define the universal rules every component must follow:
- The **safety model** (Preview vs Execute vs Durable)  
- Canonical **I/O contracts** (Intent, Evidence, Plan, Signature, Preview, Execute, Approvals)  
- Baseline **non-functional requirements**  
- **Schemas & validation** expectations  
- **Determinism & auditability** across all planning/execution  

Component `SPEC.md` files **inherit** these rules and may only deviate if explicitly stated (with rationale).

---

## 1) Safety Model (applies to all components)

### Preview
- **Side-effect free**: stubs/mocks only, no writes or external mutations.  
- Runs **n8n connectors** only in previewable/read-only mode.  
- Returns a **Preview wrapper** with normalized payload + optional evidence.  

### Execute (Short Jobs, via n8n)
- Allowed **only after explicit human approval** with a valid approval token and verified plan signature.  
- Calls real providers under **least-privilege** credentials.  
- Idempotency required (`plan_id:step:arg_hash`).  
- Returns an **Execute wrapper**.  

### Durable (Long Jobs, via Temporal)
- Handles long-running/stateful work (poll, retry, signals, compensation).  
- Deterministic workflow core, Activities for I/O.  
- Resilient to restarts; must support **ContinueAsNew** and compensation.  

---

## 2) Canonical Contracts

### 2.1 Intent (input)
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

### 2.2 Evidence Item
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
      "after": [/* deps */],
      "gate_id": "gate-A",
      "dry_run": true
    }
  ],
  "constraints": { "scopes": ["calendar.write"], "ttl_s": 900 },
  "plugins": ["<plugin_id>"],
  "meta": { "created_at": "<iso>", "author": "planner" }
}
~~~

### 2.4 Plan Signature
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

### 2.5 Preview Wrapper
~~~json
{
  "normalized": {},
  "source": "preview",
  "can_execute": true,
  "evidence": []
}
~~~

### 2.6 Execute Wrapper
~~~json
{
  "provider": "<connector_id>",
  "result": { "id": "<external_id>", "link": "<optional>" },
  "status": "created|updated|skipped|error"
}
~~~

### 2.7 Approval Token
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

## 3) Non-Functional Requirements
- **Preview latency:** p95 < 800 ms  
- **Short Execute latency:** p95 < 2 s  
- **ContextRAG:** p95 < 150 ms  
- **Plan Retrieval:** p95 < 200 ms  
- **Vector search:** < 100 ms  
- **Durable flows:** survive restarts; ContinueAsNew daily  
- **Availability:** 99.9% Intake/Preview, 99.5% Execute/Durable  
- **Observability:** structured logs, correlated by `plan_id`; no raw secrets/PII  

---

## 4) Schemas & Validation
- **Component-specific schemas** in `components/<Name>/schemas/`  
- **Shared contracts** in `plugins/schemas/` (Intent, Evidence, Plan, Signature, Wrappers)  
- **Tests must validate** against schemas; **no schema drift**  

---

## 5) Conformance
- Each `SPEC.md` must declare conformance to `GLOBAL_SPEC.md v2` and list deltas.  
- Handlers are thin: validate Intent → call service → return wrapped Preview/Execute.  
- `preview()` must never mutate; `execute()` only after valid approval & signature.  

---

## 6) Versioning
- Breaking changes require version bump and ADR.  
- Components must indicate which version they conform to.  

---

## 7) Context Policy
- **Tier 1:** session only  
- **Tier 2:** stable prefs  
- **Tier 3:** recent history  
- **Tier 4:** live signals (cross-app)  
- **Tier 5:** private content (derived facts only, explicit consent)  
- ContextRAG enforces tier budgets and produces typed `evidence[]`.  

---

## 8) Safety & Governance
- **Signature verification** required at Preview/Execute.  
- **Approval tokens** required for writes (per gate).  
- **Idempotency** enforced via datastore.  
- **Compensation** supported when declared in Registry.  
- **Privacy:** derived facts only; TTL/forget/export enforced.  
- **Observability:** plan_id correlation, latency/error metrics.  

---

## 9) Repository Mapping
Each `components/<Name>/` includes: SPEC.md, LLD.md, schemas/, tests/, code.  
Global contracts live in this file.  

---

## 10) End-to-End Examples
- **Meeting flow:** Intent → ContextRAG → Plan → Preview → Gate A → Execute → Audit/PlanWriter  
- **Shopping flow:** multi-gate approval before cart/purchase  
- **Visa watcher:** Temporal watcher signals approval gate → execute booking → notify  
