---
description: Generate a concrete, AC-mapped implementation task plan and write it to components/<Name>/tasks.md.
---

/system
Act as a planner. You MAY write exactly one file: `components/<Name>/tasks.md`. Do NOT modify any other files, do NOT run installs, and do NOT push to main. All tasks must map 1:1 to Acceptance Criteria in the component’s SPEC. Be precise with file paths under the component packet (`api/`, `service/`, `domain/`, `adapters/`, `schemas/`, `tests/`).

/user
## Inputs (ask if missing)
- Component Name: <Name>                 (e.g., FocusBlock)
- Branch: feat/<area>-<short-desc>       (e.g., feat/focus-block)
- Optional: Owners map (e.g., {"impl":"me","review":"me"})

## Read first (read-only, no edits)
1) .specify/memory/constitution.md
2) docs/architecture/PROJECT_STRUCTURE.md
3) docs/architecture/GLOBAL_SPEC.md
4) docs/architecture/Project_HLD.md
5) components/<Name>/SPEC.md
6) components/<Name>/LLD.md (if present)

/assistant
## Rules
- Every task MUST reference a concrete file path that will change under `components/<Name>/**`.
- Each task MUST tie to exactly one SPEC Acceptance Criterion (AC).
- Include test-first tasks (schema/test updates precede code).
- Preview path tasks MUST avoid external mutations; Execute path tasks use adapters only.
- No secrets; no edits outside the component packet; no pushes to main.

## What to produce (write to components/<Name>/tasks.md)
Write a single Markdown document with these sections, filled with concrete content derived from SPEC/LLD:

1) Title & Links
   - `# <Name> — Implementation Tasks (AC-mapped)`
   - Links to `components/<Name>/SPEC.md` and (if present) `components/<Name>/LLD.md`
   - Branch: `<Branch>`

2) AC → Tasks Mapping
   - For each AC (`AC1`, `AC2`, ...), list an ordered checklist of tasks that will satisfy it.
   - Include exact file paths per task (e.g., `components/<Name>/schemas/response.normalized.json`).

3) Backlog Table
   - A table with columns: `ID`, `Task`, `Files`, `AC`, `Type`, `Estimate`, `Owner`
     - Type ∈ {schema, test, api, service, domain, adapters, docs, ci}
     - Estimates are small (S/M/L or hours), bias toward S.

4) Dependency Graph (Text)
   - Express as `T1 -> T3`, `T2 -> T3`, etc. Keep it minimal and readable.

5) Definition of Done (DoD)
   - Bullets that reflect repo rules (green CI, schema valid, preview has no external mutations, PR links SPEC/LLD, etc.).

6) Risk & Mitigations
   - 3–5 bullets max, tied to tasks where relevant.

7) Execution Checklist
   - Mini runbook (create/switch branch, implement in order, run `pytest -q components/**/tests tests`, open PR using template).

8) PR Preparation
   - Suggested PR title and a short PR body stub (must link SPEC and LLD; mention contract tests).

## Constraints
- Only write `components/<Name>/tasks.md`. Do not write code, schemas, or tests in this command.
- Never push to main; assume work occurs on `<Branch>`.
- Use exact, component-local paths.
- Keep content specific and actionable (no vague “update logic” tasks).

## Output
- After writing the file, print:
  - `Tasks file: components/<Name>/tasks.md`
  - `Next step: /flow_orchestrate for <Name> on <Branch> (Planner → Implementer → Verifier → PR Manager)` OR `/plan` if LLD/schema/tests don’t exist yet.

/assistant
## Template to write (fill in; then save as components/<Name>/tasks.md)

# <Name> — Implementation Tasks (AC-mapped)

**SPEC:** components/<Name>/SPEC.md  
**LLD:** components/<Name>/LLD.md  
**Branch:** <Branch>

---

## AC → Tasks Mapping

### AC1 — <paste AC text from SPEC>
- [ ] Update schema: `components/<Name>/schemas/response.normalized.json` (add <field>, examples)
- [ ] Extend contract test: `components/<Name>/tests/test_contract.py` (cover <field>)
- [ ] Implement service.preview() stub behavior: `components/<Name>/service/<name>_service.py` (no external mutations)
- [ ] Implement api handler glue: `components/<Name>/api/<route>.py` (thin: validate → call service → wrap per GLOBAL_SPEC)
- [ ] Execute path adapter call (scaffold only): `components/<Name>/adapters/<provider>.py` (no secrets; least privilege)
- [ ] Docs: add example payload to `components/<Name>/LLD.md` (Data Contracts)

### AC2 — <paste AC text from SPEC>
- [ ] …
- [ ] …

### AC3 — <paste AC text from SPEC>
- [ ] …
- [ ] …

---

## Backlog

| ID | Task | Files | AC | Type | Estimate | Owner |
|----|------|-------|----|------|----------|-------|
| T1 | Update normalized schema with `<field>` | `components/<Name>/schemas/response.normalized.json` | AC1 | schema | S | impl:@me |
| T2 | Extend contract test to validate `<field>` | `components/<Name>/tests/test_contract.py` | AC1 | test | S | impl:@me |
| T3 | Service.preview returns `<field>` | `components/<Name>/service/<name>_service.py` | AC1 | service | M | impl:@me |
| T4 | API handler wraps preview per GLOBAL_SPEC | `components/<Name>/api/<route>.py` | AC1 | api | S | impl:@me |
| T5 | Adapter scaffold for execute | `components/<Name>/adapters/<provider>.py` | AC1 | adapters | M | impl:@me |
| T6 | LLD data contracts updated | `components/<Name>/LLD.md` | AC1 | docs | S | impl:@me |
| … | … | … | … | … | … | … |

**Legend:** Type ∈ schema, test, api, service, domain, adapters, docs, ci

---

## Dependencies (Text Graph)
T1 -> T2  
T2 -> T3 -> T4  
T3 -> T5

---

## Definition of Done
- All ACs satisfied; every task in AC mapping is checked.
- Contract tests pass locally: `pytest -q components/**/tests tests`.
- Preview path performs **no external mutations**; evidence is stubbed.
- Execute path uses adapters only, least-privilege credentials (documented, not embedded).
- JSON Schema and examples are consistent with SPEC and LLD.
- PR opened from `<Branch>` using `.github/pull_request_template.md`, **linking SPEC.md and LLD.md**.

---

## Risks & Mitigations
- Provider API drift → Mitigation: isolate via adapters and contract tests.
- Timezone edge cases → Mitigation: enforce `"tz": "America/Chicago"` defaults; add test sample.
- Data shape ambiguity → Mitigation: finalize schema fields before service code; keep examples in LLD.

---

## Execution Checklist
- [ ] `git checkout -b <Branch>` (if not on it)
- [ ] Implement tasks in dependency order
- [ ] Run `pytest -q components/**/tests tests`
- [ ] Open PR with title: `feat(<Name>): implement ACs for <short summary>`
- [ ] Request review; wait for green CI

---

## PR Preparation (stub)
**Title:** feat(<Name>): implement ACs for <short summary>  
**Body:**
- Links: `components/<Name>/SPEC.md`, `components/<Name>/LLD.md`  
- Summary: <2–3 lines>  
- Evidence: contract test output summary  
- Checklist: schema updated ✅, tests added ✅, preview safe ✅, execute via adapters ✅
