# Use Case Specification: ⟨UseCase⟩

**Feature Branch**: `⟨NNN-usecase⟩`  
**Created**: ⟨DATE⟩  
**Status**: Draft  
**Input**: User description: "⟨use case⟩"

## Execution Flow (main)
```
1. Parse user description from Input
2. Extract actors, actions, data, constraints
3. Mark ambiguities with [NEEDS CLARIFICATION: …]
4. Fill User Scenarios & Testing
5. Generate Acceptance Criteria (testable)
6. Map to components that will change
7. Run Review Checklist
8. SUCCESS (spec ready for design)
```

---

## ⚡ Quick Guidelines
- Focus on WHAT & WHY (avoid implementation)
- Must be testable & unambiguous; include ACs

### Section Requirements
- Mandatory sections below; remove anything not applicable

---

## User Scenarios & Testing *(mandatory)*

### Primary User Story
[Plain-language journey for this use case]

### Acceptance Scenarios
1. **Given** … **When** … **Then** …
2. …

### Edge Cases
- …

## Requirements *(mandatory)*
- **AC-001**: …
- **AC-002**: …

## Intent & Gates
- Intent shape (GLOBAL_SPEC 2.1): action, entities, constraints, tz, context_budget
- HITL gates: gate_ids, when/why; per‑gate approval required before writes

## Evidence Needs (ContextRAG)
- Typed keys to request (prefs/history/contacts); budget; what not to fetch

## Scopes & Safety
- Minimal provider scopes by phase (Preview read‑only; Execute write)
- Idempotency expectations for writes; compensation steps if declared

## Component Mapping
- components/⟨Name⟩/: files expected to change (api/service/domain/adapters/schemas/tests)
- …

## Dependencies & Risks
- …

## Conformance
This work conforms to docs/architecture/GLOBAL_SPEC.md v2 and docs/architecture/Project_HLD.md.
