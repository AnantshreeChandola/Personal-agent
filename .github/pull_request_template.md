<!--
Title format (recommended): feat/<area>: <short description>
Example: feat/FocusBlock: preview normalized payload and schema

Fill every section below. Delete any “N/A” lines before submitting.
-->

## Summary (what & why)
- 

## PRP (Plan/PR Proposal)
- Link: (optional)  <!-- or write “N/A” -->

## Components touched
- [ ] components/<Name>/SPEC.md → link:
- [ ] components/<Name>/LLD.md → link:
- (add more rows if needed)

## Contracts / Schemas
- [ ] Updated component schemas: components/<Name>/schemas/<file>.json
- [ ] Updated shared schemas (if applicable): plugins/schemas/<file>.json
- Validation strategy (tests, jsonschema, etc.): 

## Evidence (Preview artifacts)
- Links (payloads, screenshots, logs): 

## CI
- Checks: (link to GitHub Checks page)
- Notes on any flaky tests or retries:

## Risks / Rollback
- Risks:
- Rollback plan:

## ADRs (only if decision changed)
- docs/architecture/adr/NNNN-<title>.md → link:

---

### Checklist (required by the Constitution)
- [ ] Optional planning notes linked **or** “N/A”
- [ ] Components touched listed; SPEC/LLD links included
- [ ] Tests added/updated and **green** (component + system)
- [ ] Schemas updated and validated (if payloads changed)
- [ ] Preview uses stubs/mocks; **no secrets** committed
- [ ] Evidence/artifacts linked (if applicable)
- [ ] Reviewer(s) assigned

### Safety & Integrity (GLOBAL_SPEC)
- [ ] Plan signature verified at Preview and Execute
- [ ] Gate approval token/signature bound to {plan_hash, gate_id, user_id, exp}
