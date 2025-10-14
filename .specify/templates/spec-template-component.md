# Component Specification: ⟨Name⟩

**Feature Branch**: `⟨NNN-name⟩`  
**Created**: ⟨DATE⟩  
**Status**: Draft  
**Input**: User description: "⟨component: Name⟩"

## ⚡ Quick Guidelines
- Focus on WHAT & WHY (no implementation details)
- Written for stakeholders; must be testable & unambiguous

### Section Requirements
- Mandatory sections below; remove anything not applicable

---

## User Scenarios & Testing *(mandatory)*

### Primary User Story
[Plain-language journey for this component]

### Acceptance Scenarios
1. **Given** … **When** … **Then** …
2. …

### Edge Cases
- …

## Requirements *(mandatory)*

### Functional Requirements
- **FR-001**: Interfaces — define handlers (thin) and service.preview/execute signatures (inputs/outputs)
- **FR-002**: Safety — preview has no external mutations; execute via adapters; least‑privilege scopes
- **FR-003**: Idempotency & Compensation — define idempotency key strategy and compensation ops (if any)
- **FR-004**: Schemas — JSON Schemas for payloads; wrappers adhere to GLOBAL_SPEC Preview/Execute
- **FR-005**: Observability — structured logs (no PII), correlation (plan_id/step/role), error classes
- **FR-006**: Determinism — same inputs → same preview outputs; no hidden state
- **FR-007**: NFRs — latency/availability budgets; resource limits
- **FR-008**: Backward Compatibility — versioning policy; ADR required for breaks
- **FR-009**: Registry Impacts — list `uses/call → node/op/params`, previewability, scopes, idempotency, compensation

### Key Entities *(include if data involved)*
- **⟨Entity⟩**: purpose and fields (no tech details)

---

## Review & Acceptance Checklist
- [ ] No implementation details
- [ ] Testable & unambiguous
- [ ] Scope bounded; assumptions listed
- [ ] GLOBAL_SPEC Preview/Execute wrappers respected
- [ ] Idempotency & (if any) compensation defined
- [ ] Registry entry impacts listed

---

## Conformance
This work conforms to docs/architecture/GLOBAL_SPEC.md v2 and docs/architecture/Project_HLD.md.
