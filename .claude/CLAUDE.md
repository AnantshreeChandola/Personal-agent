# Claude — Project Rules & Working Context

## Phase
Architecture & scaffolding. Use existing structure; do not invent new folders.

## Follow these rules
- Read `/docs/architecture/PersonalAgent_HLD.md`, `/spec/product/agent-product-spec.md`, and `.specify/memory/constitution.md` before making changes.
- Create a branch `feat/<short-name>`; open a PR linking the relevant spec and (if present) a draft plan in `/plans/drafts/`.
- If CI fails, keep iterating on the same branch until it’s green.
- For Python services, follow `/docs/dev/python/CODING_STANDARDS.md`.

## Paths of truth (use these exactly)
- `/spec/product/` — global & capability specs
- `/plans/{drafts,signed,records}` — plan proposals, signed plans, run records
- `/plugins/{catalog.yaml,schemas/*}` — capability registry + JSON Schemas
- `/tests/` — acceptance/contract tests
- `/.github/workflows/ci.yml` — CI gates

## Contracts to respect
**Intent**
```json
{"intent":"<string>","entities":{},"constraints":{},"tz":"America/Chicago"}
