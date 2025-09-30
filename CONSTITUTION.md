# Project Constitution (Non-Negotiables)

This file defines the guardrails for all contributors and agents. It is **binding**.

## 1) Branches & Pull Requests
- **Protected:** `main` — no direct pushes.
- All changes via **PRs** from branches named:
  - `feat/<area>-<desc>`, `feat/uc-<usecase>-<desc>`, `feat/comp-<component>-<desc>`, `fix/<area>-<desc>`, or `docs/<area>-<desc>`.
- **Human review required** on every PR.
- PR body must:
  - List **use case and components touched** and link their `usecases/<UseCase>/{SPEC.md,LLD.md}` and `components/<Name>/{SPEC.md,LLD.md}`.
  - Include the checklist in §10.

## 2) CI Is the Gate
- **Green CI is required** to merge. If CI fails, keep iterating on the same branch.
- CI must run at minimum:
  - Component tests: `components/**/tests`
  - Use‑case tests: `usecases/**/tests`
  - System tests: `tests`
  - (When present) schema validation (shared + components), plan/envelope schema checks, SAST, license checks

## 3) Safety Model — Preview vs Execute
- **Preview** paths use **stubs/mocks only**; never call real providers or mutate external state.
- **Execute** runs only after **explicit human approval** and **plan‑signature verification**.
- Preview should attach/link **evidence** (artifacts/payloads/screens) in the PR.

- Approval tokens are required for any write step; multi‑gate flows must present the matching gate’s token.
- All writes must be **idempotent** (key: `plan_id:step:arg_hash`), with compensation when declared in the Registry.

## 4) Secrets, Privacy, Logs
- No secrets in code, PRs, or logs. Use environment managers (e.g., GitHub Secrets).
- Default timezone: **America/Chicago**; avoid leaking precise geo/PII.
- Logs must be structured, minimal, and free of PII.

## 5) Source of Truth (Paths)
- Structure: `docs/architecture/PROJECT_STRUCTURE.md`
- Global HLD: `docs/architecture/Project_HLD.md`
- ADRs: `docs/architecture/adr/NNNN-title.md`
- **Use cases:** `usecases/<UseCase>/{SPEC.md, LLD.md, plans/, tests/, fixtures/}`
- **Components:** `components/<Name>/{SPEC.md, LLD.md, api/, service/, domain/, adapters/, schemas/, diagrams/, tests/}`
- **Shared contracts:** `plugins/{catalog.yaml, schemas/*}`
- **System tests:** `tests/`
- **CI workflows:** `.github/workflows/`

## 6) Component Packet Rules (must follow)
Every new or modified component **must** conform to this structure:
components/<Name>/
SPEC.md # WHAT & acceptance (requirements/ACs)
LLD.md # HOW (APIs, sequences, data models, failure modes)
api/ # routes/controllers/handlers; thin I/O only
service/ # component services; implements preview(), execute(); no cross‑component orchestration or connector bindings
domain/ # pure domain models & invariants; no external deps
adapters/ # external integrations (HTTP, DB, queues); called by service
schemas/ # component-specific JSON Schemas
tests/ # acceptance/contract tests for this component
diagrams/ # mermaid/plantuml as needed


**Layering rules**
- `api/handlers` **must not** contain business logic; call `service/*` only.
- `service/` implements **`preview(input, tz)`** and **`execute(approved_preview, creds)`**.
  - `preview()` **must not** call mutating adapters.
  - `execute()` may call adapters, subject to approvals in §3; enforce idempotency keys and support compensation when declared.
- `domain/` is framework- and provider-agnostic (pure types/validation).
- `adapters/` is the only place that touches external systems.
- Component-specific schemas live in `components/<Name>/schemas/`; **shared** schemas live in `plugins/schemas/`.
 - Emit audit fields (no PII): `plan_id, step, role, op, latency_ms, status`.

**Test rules**
- Each component must include at least **one contract/acceptance test** under `components/<Name>/tests/`.
- Each use case must include scenario/e2e tests under `usecases/<UseCase>/tests/`.
- Contract tests should validate payloads against the declared JSON Schemas.

## 7) Contracts & Verification
- Public payloads must be defined by **JSON Schemas** (component local or shared).
 - Tests must enforce **acceptance criteria** from the component’s or use case’s `SPEC.md`.
- **Definition of Done**:
  - Relevant SPEC/LLD linked in the PR (use case and components touched)
  - Tests passing (use‑case + component + system)
  - Schemas updated/validated if payloads changed
  - Preview evidence (if applicable) attached
  - Human review approved
 - Backward compatibility required for components; breaking changes demand an ADR + version bump.

## 8) Decision Records (ADRs)
- Record significant decisions under `docs/architecture/adr/NNNN-title.md`.
- Do not rewrite old ADRs; supersede with a new one if the decision changes.

## 9) Plans & Provenance (Optional but encouraged)
- When using plan signing or execution records:
  - Drafts: `plans/drafts/`
  - Signed: `plans/signed/`
  - Run records/evidence: `plans/records/`

## 10) PR Checklist (include in every PR body)
- [ ] Use case and components touched listed; **SPEC/LLD links included** (required)
- [ ] (Optional) Planning notes attached or “N/A”
- [ ] Tests added/updated and **green** (use‑case + component + system)
- [ ] Schemas updated and validated (if payloads changed)
- [ ] Plan has {mode, role, after, gate_id}; signature verified at Preview/Execute
- [ ] Approval tokens enforced for writes; idempotency keys present; compensation paths (if declared)
- [ ] Backward compatibility maintained OR ADR link + version bump noted
- [ ] Preview uses stubs/mocks; **no secrets** committed
- [ ] Evidence/artifacts linked (if applicable)
- [ ] Reviewer(s) assigned

## 11) Tooling & Agents
- Agents may use: git (clone/checkout/commit/open PR), shell to run tests/linters, editors.
- Agents **must not** alter branch protections, CI settings, or repository secrets.
- Multi-agent flows (planner/implementer/verifier/pr-manager) must satisfy §10 before requesting merge; a critic role confirms the checklist and CI status.

## 12) Violations
- PRs that bypass these rules will be closed or reverted.
- Repeated violations may lead to branch restrictions.

---
_Amend this constitution only via PR that updates this file and adds an ADR explaining why._
