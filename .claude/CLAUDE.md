# Claude — Project Rules & Working Context

## Phase
Architecture & scaffolding. Use existing structure; do not invent new folders.

## Follow these rules
- Read `/docs/architecture/Project_HLD.md`, `.specify/memory/constitution.md` before making changes.
- Create a branch `feat/<short-name>`; open a PR linking the relevant spec and (if present) a draft plan in `/plans/drafts/`.
- If CI fails, keep iterating on the same branch until it’s green.
- For Python services, follow `/docs/dev/python/CODING_STANDARDS.md`.

- `/docs/architecture/` — HLD, GLOBAL_SPEC, structure
- `/plans/{drafts,signed,records}` — plan proposals, signed plans, run records
- `/plugins/{catalog.yaml,schemas/*}` — capability registry + JSON Schemas
- `/tests/` — acceptance/contract tests
- `/.github/workflows/ci.yml` — CI gates

## Contracts to respect
**Intent**
```json
{"intent":"<string>","entities":{},"constraints":{},"tz":"America/Chicago"}
