---
description: Plan & scaffold a component implementation that conforms to GLOBAL_SPEC and this repo’s Constitution, without touching executable code.
---

/system
Act as a planner/scaffolder. You MAY write only documentation, schemas, tests, and diagrams for the targeted component packet. You MUST NOT modify executable code (api/, service/, domain/, adapters/) and MUST NOT push to main. Respect the repo’s sources of truth.

/user
## Inputs (ask if missing)
- Component Name: <Name>                 (e.g., FocusBlock)
- Branch: feat/<area>-<short-desc>       (e.g., feat/focus-block)
- One-line Summary: <what/why>
- Optional: SPEC_SOURCE (absolute path to a Spec Kit spec to promote)

## Read first (canonical)
1) CONSTITUTION.md
2) docs/architecture/PROJECT_STRUCTURE.md
3) docs/architecture/GLOBAL_SPEC.md       (Intent + Preview/Execute envelopes, NFRs)
4) docs/architecture/Project_HLD.md
5) components/<Name>/SPEC.md              (must exist; see Promotion note below)

## Promotion note (only if SPEC.md is missing)
- If components/<Name>/SPEC.md does not exist AND SPEC_SOURCE was provided (absolute path from Spec Kit),
  copy SPEC_SOURCE → components/<Name>/SPEC.md and proceed.
- If neither exists, STOP and recommend running /specify first. Do not fabricate a SPEC.

/assistant
## Steps (plan, scaffold, and prepare for Flow)

1) Prepare the component packet (no code)
   - Ensure directories exist (create if missing):
     components/<Name>/
       api/        (no writes in this command)
       service/    (no writes in this command)
       domain/     (no writes in this command)
       adapters/   (no writes in this command)
       schemas/
       tests/
       diagrams/

2) Produce/refresh LLD (design, not code)
   - Write or update: components/<Name>/LLD.md
   - Content must include:
     - Purpose & Scope (link to SPEC.md)
     - Public Interfaces (thin handlers) → map inputs to service calls; no inline business logic
     - Service entry points and behaviors:
       - preview(input, tz="America/Chicago") → normalized (no external mutations; stubs/mocks only)
       - execute(approved_preview, creds)     → provider_result (via adapters; least privilege)
     - Domain models (entities, value objects) with invariants
     - Adapters (what providers, what minimal scopes, idempotency strategy)
     - Sequences (happy/error), retries/backoff, idempotency keys
     - Data contracts: reference schema(s) in schemas/
     - Observability: what to log (structured), no secrets/PII, correlation ids
     - Risks & Open Questions
     - Conformance line: “Conforms to docs/architecture/GLOBAL_SPEC.md v1.”

3) Define the normalized payload contract (schema-first)
   - Write or update: components/<Name>/schemas/response.normalized.json
   - JSON Schema (draft 2020-12) describing the component’s normalized payload returned in Preview.
   - Include: title, type=object, required fields, properties with types/examples, additionalProperties=false.

4) Create a minimal contract test (safe, self-contained)
   - Write or update: components/<Name>/tests/test_contract.py
   - The test should:
     - Import pytest and attempt to import jsonschema; skip the test suite if jsonschema is missing.
     - Load response.normalized.json.
     - Build a minimal valid sample normalized payload (dict) that satisfies the schema.
     - Assert wrapper requirements from GLOBAL_SPEC Preview envelope:
       - wrapper.source == "preview"
       - wrapper.can_execute is bool
       - wrapper.normalized matches the JSON Schema
     - Do not import production code; the sample is inline and uses stubs only.

5) Tasks for Implementer (no code in this command)
   - Emit a short ordered task list mapped 1:1 to SPEC.md Acceptance Criteria.
   - Each task names exact files to touch under components/<Name>/**.
   - Save to components/<Name>/tasks.md (optional) OR append to LLD.md under “Tasks”.

6) Branch & PR prep (do not open PR here)
   - Ensure on feature branch: <Branch> (create/switch if needed).
   - Stage only docs/schemas/tests created above.
   - Prepare a PR title and body using .github/pull_request_template.md:
     - Required links: components/<Name>/SPEC.md and components/<Name>/LLD.md
     - Mention that Preview path has no external mutations; Execute via adapters with least privilege.
   - Hand off to Flow: next command is /flow_orchestrate (Planner → Implementer → Verifier → PR Manager).

## Deliverables (print in this order)
1) Path to LLD: components/<Name>/LLD.md
2) Path to schema: components/<Name>/schemas/response.normalized.json
3) Path to test: components/<Name>/tests/test_contract.py
4) Tasks (bullet list)
5) Suggested branch, PR title, and PR body (ready to paste)

/assistant
## Templates to write (exact file contents)

### A) components/<Name>/LLD.md  (overwrite or create)
# <Name> — Low Level Design (LLD)

## Purpose & Scope
Link: components/<Name>/SPEC.md  
One-line summary: <One-line Summary>

## Public Interfaces (API/Handlers — thin)
- Describe each endpoint/handler and its request/response shapes.
- Handlers must: parse/validate → call service → wrap per GLOBAL_SPEC.

## Service Entry Points
- preview(input, tz="America/Chicago") -> NormalizedPayload
  - No external mutations; uses stubs/mocks and static fixtures only.
  - Returns wrapper: { "normalized": <NormalizedPayload>, "source": "preview", "can_execute": <bool>, "evidence": [] }
- execute(approved_preview, creds) -> ProviderResult
  - Calls adapters only; least-privilege credentials; idempotent where possible.
  - Returns wrapper per GLOBAL_SPEC Execute envelope.

## Domain
- Entities, value objects, invariants relevant to <Name>.

## Adapters
- External systems/providers and exact operations (execute-only).
- Permissions/scopes and idempotency strategy.

## Sequences
- Happy path and error path (step-by-step).
- Retries, backoff, and compensation if needed.

## Data Contracts
- Normalized payload schema: schemas/response.normalized.json
- Wrapper contracts: see docs/architecture/GLOBAL_SPEC.md

## Observability & Safety
- Structured logs (no PII/secrets), correlation ids.
- Preview path guarantees no external mutations.
- Execute path requires explicit approval and minimal scopes.

## Risks & Open Questions
- …

## Conformance
This component conforms to docs/architecture/GLOBAL_SPEC.md v1.

---

### B) components/<Name>/schemas/response.normalized.json  (overwrite or create)
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "<Name> Normalized Payload",
  "type": "object",
  "additionalProperties": false,
  "required": ["id", "summary"],
  "properties": {
    "id":      { "type": "string", "description": "Stable preview item id" },
    "summary": { "type": "string", "description": "Short human summary of the action/result" }
  }
}

---

### C) components/<Name>/tests/test_contract.py  (overwrite or create)
import json
import os
import pytest

try:
    import jsonschema  # type: ignore
    from jsonschema import validate
except Exception:  # pragma: no cover
    pytest.skip("jsonschema not installed; skipping contract tests", allow_module_level=True)

SCHEMA_PATH = os.path.join(os.path.dirname(__file__), "..", "schemas", "response.normalized.json")

def load_schema():
    with open(SCHEMA_PATH, "r", encoding="utf-8") as f:
        return json.load(f)

def test_preview_wrapper_and_normalized_contract():
    schema = load_schema()

    # Minimal valid normalized payload (adjust as SPEC evolves)
    normalized = {
        "id": "preview-001",
        "summary": "Example <Name> preview summary"
    }

    # GLOBAL_SPEC Preview envelope assertions
    wrapper = {
        "normalized": normalized,
        "source": "preview",
        "can_execute": True,
        "evidence": []
    }

    # Validate normalized payload against schema
    validate(instance=normalized, schema=schema)

    # Wrapper checks per GLOBAL_SPEC
    assert wrapper["source"] == "preview"
    assert isinstance(wrapper["can_execute"], bool)
    assert isinstance(wrapper["evidence"], list)

---

### D) components/<Name>/tasks.md  (optional; overwrite or create)
# <Name> — Implementation Tasks (Mapped to SPEC Acceptance Criteria)

## AC → Tasks
- AC1: <copy AC from SPEC.md>
  - [ ] Implement API handler stub in api/ (no business logic)
  - [ ] Implement service.preview() stub returning schema-valid normalized payload
  - [ ] Add/adjust schema fields in schemas/response.normalized.json
  - [ ] Extend test_contract.py sample to cover new fields
- AC2: <copy AC from SPEC.md>
  - [ ] …
- AC3: <copy AC from SPEC.md>
  - [ ] …

## Notes
- No external mutations in preview path.
- Execute path via adapters with least privilege and idempotency.
- Link PRs to SPEC.md and LLD.md.

/assistant
## Final Output Format (print this summary)
- LLD: components/<Name>/LLD.md
- Schema: components/<Name>/schemas/response.normalized.json
- Test: components/<Name>/tests/test_contract.py
- Tasks: components/<Name>/tasks.md (or LLD Tasks section)
- Suggested branch: <Branch>
- Next command: /flow_orchestrate for <Name> on <Branch> (Planner → Implementer → Verifier → PR Manager)

## Constraints Recap
- Only write files listed above under components/<Name>/**.
- Do not modify executable code (api/, service/, domain/, adapters/) in this command.
- Do not push to main. Use feature branch only.
- Follow docs/architecture/GLOBAL_SPEC.md v1 and CONSTITUTION.md.
