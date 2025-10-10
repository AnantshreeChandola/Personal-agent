---
description: Generate LLD and a flow diagram from a SPEC. Writes LLD.md and a Mermaid flowchart; no code edits.
---

/system
Act as a design generator. Read a SPEC (component or use case) and produce:
- LLD.md with responsibilities, interfaces, sequences, data contracts, safety, observability, and NFRs
- A Mermaid flowchart saved alongside the LLD (or under docs/architecture/diagrams/ for use cases)
Do not modify code, schemas, or tests.

/user
## Inputs (ask if missing)
- Target: component/<Name> or usecases/<UseCase>
- SPEC path: absolute path to SPEC.md
- Output dir: components/<Name>/ (for component) or specs/<NNN-…>/ (for use case)

## Read first (canonical)
- .specify/memory/constitution.md
- docs/architecture/PROJECT_STRUCTURE.md
- docs/architecture/GLOBAL_SPEC.md
- docs/architecture/Project_HLD.md
- <SPEC path>

/assistant
## LLD.md sections (write)
- Purpose & Scope; Conformance to GLOBAL_SPEC
- Interfaces (handlers, service.preview/execute signatures)
- Adapters (providers, scopes, idempotency, compensation)
- Sequences (happy/error, retries/backoff)
- Data Contracts (reference schemas)
- Observability & Safety (logs, PII, gates)
- Risks & Open Questions

## Diagram (write)
- Mermaid flowchart capturing key steps/gates; save as diagrams/flow.mmd or specs/<NNN-…>/flow.mmd

## Constraints
- Only write LLD.md and flow.mmd; no code/schemas/tests in this command.
- Use existing directories; create diagrams/ if missing for components.
