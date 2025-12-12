---
description: Review architectural decisions in SPEC/LLD for alignment with GLOBAL_SPEC and HLD principles.
---

Read first:
- `/docs/architecture/Project_HLD.md`
- `/docs/architecture/GLOBAL_SPEC.md`
- Target component's `SPEC.md` and `LLD.md`

## Architectural Review Checklist

### 1. Conformance to GLOBAL_SPEC
- [ ] Uses Preview/Execute wrapper pattern
- [ ] Preview mode is READ-ONLY (no mutations, no side effects)
- [ ] Execute mode has idempotency keys (`plan_id:step:arg_hash`)
- [ ] Intent input matches schema (intent, entities, constraints, tz)
- [ ] Normalized payload structure documented
- [ ] Error codes defined and descriptive

### 2. Layer Assignment
- [ ] Component fits into exactly one layer:
  - **Intake**: User request understanding (Intake, ContextRAG)
  - **Core**: Planning and memory (Planner, Signer, PlanLibrary, etc.)
  - **Runtime**: Execution orchestration (PreviewOrchestrator, ExecuteOrchestrator, DurableOrchestrator)
  - **Platform**: Infrastructure (Audit, API Gateway)
- [ ] No cross-layer violations (e.g., Intake calling ExecuteOrchestrator directly)

### 3. Preview-First Safety Model
- [ ] Preview operations use mocks/stubs for external calls
- [ ] Preview returns `can_execute: true/false` based on feasibility
- [ ] No writes to database/external APIs in preview mode
- [ ] Preview state caching declared if user makes selections
- [ ] Execute steps reference cached state with `{{preview.cached_state.field}}`

### 4. Idempotency & Compensation
- [ ] Write operations declare idempotency strategy
- [ ] Idempotency keys include: `plan_id`, `step`, `hash(args)`
- [ ] Compensation operation declared in PluginRegistry (if reversible)
- [ ] Non-reversible operations documented (e.g., "send_email": no compensation)
- [ ] Saga pattern for multi-step transactions

### 5. Resource Locking
- [ ] Fine-grained locks declared (e.g., `calendar.alice.write`)
- [ ] Read operations don't require locks
- [ ] Coarse locks only for rate-limited resources
- [ ] Lock scope minimal (avoid blocking unrelated operations)

### 6. Dependencies
- [ ] Dependencies on other components are minimal and documented
- [ ] No circular dependencies between components
- [ ] External APIs accessed through adapters (not direct imports)
- [ ] Shared contracts versioned (avoid breaking changes)

### 7. Scalability & Performance
- [ ] Can handle 100+ concurrent plans
- [ ] Preview latency target: p95 < 800ms
- [ ] Execute latency target: p95 < 2s
- [ ] Database queries use indexes
- [ ] N+1 query problems avoided
- [ ] Caching strategy defined for expensive operations

### 8. Observability
- [ ] Structured logs with `plan_id` correlation
- [ ] No secrets/PII in logs (API keys, passwords, email content)
- [ ] Metrics defined: latency, error rate, throughput
- [ ] Error scenarios documented with expected behavior
- [ ] Retry logic with exponential backoff

### 9. Privacy & Security
- [ ] Adheres to tier-based context policy (Tier 1-5)
- [ ] Raw PII not stored (only derived facts)
- [ ] TTL enforcement for temporal data
- [ ] Consent checks before accessing user data
- [ ] OAuth scopes minimal (principle of least privilege)

### 10. Testability
- [ ] Every acceptance criterion maps to a test
- [ ] Unit tests with mocked dependencies
- [ ] Integration tests with real database/Redis
- [ ] Contract tests validate SPEC.md schema
- [ ] Preview-execute consistency tests

## Review Output Format

```markdown
## Architectural Review: [Component Name]

### ✓ Strengths
- Strength 1: Description
- Strength 2: Description
- Strength 3: Description

### ⚠️ Warnings (Non-blocking)
- Warning 1: Issue + Recommendation
- Warning 2: Issue + Recommendation

### 🚨 Blockers (Must fix before implementation)
- Blocker 1: Issue + Required fix
- Blocker 2: Issue + Required fix

### Blast Radius Analysis
**Failure Scenario**: [What could go wrong]
**Impact**: [Scope of damage if this component fails]
**Containment**: [How failure is isolated]
**Example**: [Concrete timeline with numbers]

### Recommended Actions
1. [ ] Action 1 (Priority: High/Medium/Low)
2. [ ] Action 2 (Priority: High/Medium/Low)

### ADR Required?
[ ] Yes - Create ADR for: [Decision topic]
[ ] No - Changes are incremental and non-breaking
```

## Common Issues & Fixes

### Issue: Preview calls real external API
**Fix**: Add `mock=True` parameter to adapter calls in preview mode
```python
# Bad
result = adapter.create_event(args)

# Good
result = adapter.create_event(args, mock=(mode == "preview"))
```

### Issue: No idempotency key
**Fix**: Add idempotency check before write operations
```python
key = f"{plan_id}:{step}:{hash(args)}"
if redis.exists(key):
    return redis.get(key)
```

### Issue: Component spans multiple layers
**Fix**: Split into separate components, one per layer
- Example: Split "BookingService" into Intake (parse intent) + Runtime (execute booking)

### Issue: Circular dependency (A → B → A)
**Fix**: Extract shared logic into new component C, both A and B depend on C

### Issue: No compensation for reversible operation
**Fix**: Declare compensation in PluginRegistry
```json
{
  "create_event": {
    "compensation": "delete_event"
  }
}
```

### Issue: Logs contain secrets/PII
**Fix**: Redact sensitive fields, use plan_id for correlation
```python
# Bad
logger.info(f"Booking for {user.email} at {event.time}")

# Good
logger.info(f"Booking step completed", extra={"plan_id": plan_id, "step": 5})
```

## When to Escalate to Architect Agent

- **Breaking changes**: Component API changes affect multiple dependents
- **New infrastructure**: Adding database, message queue, or external service
- **Performance concerns**: Target latencies cannot be met with current design
- **Security/privacy**: New data handling or consent requirements
- **Blast radius unclear**: Failure scenarios span multiple components
