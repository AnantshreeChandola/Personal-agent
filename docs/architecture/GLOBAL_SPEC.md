# GLOBAL SPEC — Operating Contract (v1)

**Status:** Active  
**Applies to:** All components in this repository  
**Default timezone:** America/Chicago

---

## 0) Purpose
Define the universal rules every component must follow:
- The **safety model** (Preview vs Execute)
- Canonical **I/O contracts** (Intent input, Preview wrapper, Execute wrapper)
- Baseline **non-functional requirements**
- **Schemas & validation** expectations

Component `SPEC.md` files **inherit** these rules and may only deviate if explicitly stated (with rationale).

---

## 1) Safety Model (applies to all components)

### Preview
- Uses **stubs/mocks only**; **no calls** to real external providers.
- **No external mutations** (no writes, no side effects).
- Returns a **normalized payload** and optional **evidence** (links, payload dumps, screenshots).

### Execute
- Allowed **only after explicit human approval** (and, if enabled, verified plan signature).
- May call real providers via adapters under **least-privilege** credentials.
- Returns a provider **result** and **status**.

---

## 2) Universal Contracts

### 2.1 Intent (input)
Every entry point that handles user intent must accept (or normalize to) this shape:
~~~json
{
  "intent": "<string>",
  "entities": {},
  "constraints": {},
  "tz": "America/Chicago"
}
~~~
- `intent`: machine name of the action (e.g., "create_focus_block").
- `entities`: concrete parameters (times, titles, ids…).
- `constraints`: guardrails/preferences (limits, caps, policies…).
- `tz`: IANA timezone string; default is **America/Chicago**.

### 2.2 Preview (normalized output wrapper)
All Preview responses wrap the component-specific `normalized` payload in this envelope:
~~~json
{
  "normalized": {},
  "source": "preview",
  "can_execute": true,
  "evidence": ["preview://..."]
}
~~~
- `normalized`: component-specific object (see that component’s schema).
- `source`: must be `"preview"`.
- `can_execute`: boolean; whether the Preview is eligible for Execute (subject to approval).
- `evidence`: optional URIs/strings referencing preview artifacts.

### 2.3 Execute (result wrapper)
All Execute responses use this envelope:
~~~json
{
  "provider": "<string>",
  "result": { "id": "<id>" },
  "status": "created|updated|error"
}
~~~
- `provider`: canonical name of the external system acted upon.
- `result`: provider-specific object; **must** include an identifier (`id`).
- `status`: one of `"created"`, `"updated"`, or `"error"`.

> The **envelopes** above are universal. Each component defines its own `normalized` **payload schema** (see §4).

---

## 3) Non-Functional Requirements (baseline)
- **Preview latency:** p95 < **800 ms** (component + model time; excludes large external calls, which aren’t allowed in Preview).
- **Execute latency:** p95 < **2 s**, subject to provider SLOs.
- **Observability:** structured, minimal logs; **no secrets/PII**.
- **Reliability:** CI must pass on the default branch; contract tests must guard critical flows.

---

## 4) Schemas & Validation
- **Component-specific schemas** live in:  
  `components/<Name>/schemas/`  
  (e.g., `request.json`, `response.normalized.json`).

- **Shared/cross-component schemas** live in:  
  `plugins/schemas/`  
  (e.g., `intent.schema.json`, `preview.wrapper.schema.json`, `execute.wrapper.schema.json` if you centralize the envelopes).

- **Tests must validate**:
  1) Preview outputs against the component’s **normalized schema**, and  
  2) *(Optionally)* against shared **wrapper schemas** for Preview/Execute if maintained centrally.

- **No schema drift:** when payloads change, update schemas **and** tests in the **same PR**.

---

## 5) Conformance
- Each `components/<Name>/SPEC.md` should include:  
  _“This component conforms to `docs/architecture/GLOBAL_SPEC.md` v1.”_  
  and list any explicit deltas.

- Handlers (in `api/`) are **thin**: validate **Intent**, call **service** (`preview()` / `execute()`), and return the wrapped response.

- `preview()` **must not** call mutating adapters; `execute()` may, **after approval** (see §1).

---

## 6) Versioning
- This document is versioned. Breaking changes require a version bump (v2, v3…) **and** an ADR explaining why.
- Components must indicate which version they conform to in their `SPEC.md`.

---

## 7) References
- **Constitution (repo laws):** `CONSTITUTION.md`  
- **Project structure:** `docs/architecture/PROJECT_STRUCTURE.md`  
- **Global HLD:** `docs/architecture/Project_HLD.md`  
- **Shared contracts (optional):** `plugins/schemas/`
