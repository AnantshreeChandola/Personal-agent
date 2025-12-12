---
description: Generate a Low-Level Design (LLD.md) for a component after SPEC.md is approved.
---

Read first:
- Component's `SPEC.md`
- `/docs/architecture/Project_HLD.md`
- `/docs/architecture/GLOBAL_SPEC.md`
- For Python services: `/docs/dev/PYTHON_GUIDE.md`

## LLD.md Template

```markdown
# [Component Name] — LLD

## Architecture

### Data Flow
```
Step 1: Input validation
  ↓
Step 2: Business logic
  ↓
Step 3: Storage/API call (preview: mock, execute: real)
  ↓
Step 4: Response formatting (Preview/Execute wrapper)
```

### Class Diagram
```
[Handler] → [Service] → [Repository/Adapter]
    ↓
[Schemas] (Pydantic models)
```

## File Structure
```
components/<Name>/
├── __init__.py
├── api/
│   └── handlers.py      # FastAPI routes
├── service/
│   └── service.py       # preview() and execute()
├── domain/
│   └── models.py        # Domain entities
├── adapters/
│   └── provider.py      # External API/DB calls
├── schemas/
│   ├── input.py         # Intent schema
│   ├── response.normalized.json
│   └── output.py        # Preview/Execute wrappers
└── tests/
    ├── test_service.py
    ├── test_contract.py
    └── test_integration.py
```

## Key Classes

### Handler (API Layer)
**Purpose**: Parse request, delegate to service, wrap response
**Methods**:
- `preview_handler(intent: Intent) -> PreviewWrapper`
- `execute_handler(plan: Plan, approval: ApprovalToken) -> ExecuteWrapper`

### Service (Business Logic)
**Purpose**: Core logic for preview and execute modes
**Methods**:
- `preview(intent: Intent) -> NormalizedPayload`
  - READ-ONLY: No mutations
  - Mocks/stubs for external calls
  - Returns normalized data
- `execute(approved_preview: ApprovedPreview) -> ProviderResult`
  - WRITE operations
  - Idempotency: `plan_id:step:arg_hash`
  - Compensation if declared in Registry

### Repository/Adapter
**Purpose**: External I/O (database, APIs)
**Methods**:
- Provider-specific calls
- Scopes/auth handling
- Retry logic with exponential backoff

## Database Schema (if applicable)
```sql
CREATE TABLE table_name (
  id UUID PRIMARY KEY,
  field1 VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_field1 ON table_name(field1);
```

## API Endpoints
```
POST /api/v1/component/preview
POST /api/v1/component/execute
```

## Error Handling
- `ValidationError`: Invalid input → 400
- `NotFoundError`: Resource not found → 404
- `ExecutionError`: External API failure → 502
- Retry policy: 3 retries, exponential backoff (100ms, 200ms, 400ms)

## Idempotency
```python
key = f"{plan_id}:{step}:{hash(args)}"
if redis.exists(key):
    return redis.get(key)  # Return cached result

result = provider.execute(...)
redis.setex(key, 3600, result)  # Cache for 1 hour
```

## Compensation (Saga Pattern)
```python
# Registry declares compensation
{
  "create_resource": {
    "compensation": "delete_resource"
  }
}

# On failure, undo in reverse order
```

## Observability
**Logs** (no secrets/PII):
```json
{
  "plan_id": "01HX...",
  "step": 5,
  "role": "Booker",
  "operation": "create_event",
  "latency_ms": 234,
  "status": "success"
}
```

**Metrics**:
- Preview latency: p95 < 800ms
- Execute latency: p95 < 2s
- Error rate by operation

## Testing Strategy

### Unit Tests
Mock all external dependencies (database, APIs)
```python
def test_preview_happy_path():
    """AC1: [Acceptance criterion from SPEC]"""
    service = MyService()
    mock_repo = Mock()
    result = service.preview(intent)
    assert result.normalized.field == expected
```

### Integration Tests
Real database (testcontainers), real Redis
```python
def test_execute_with_real_db(postgres_container):
    """AC2: [Acceptance criterion from SPEC]"""
    # Test with real database
```

### Contract Tests
Validate against SPEC.md acceptance criteria
```python
def test_preview_wrapper_schema():
    """Validate Preview wrapper matches GLOBAL_SPEC"""
    result = service.preview(intent)
    assert result.source == "preview"
    assert result.can_execute in [True, False]
```

## Implementation Phases
1. **Phase 1**: Core service logic + unit tests
2. **Phase 2**: Database/adapter integration + integration tests
3. **Phase 3**: API handlers + contract tests
4. **Phase 4**: Observability (logs, metrics, dashboards)

## Conformance
Conforms to `/docs/architecture/GLOBAL_SPEC.md` v2.

## Open Technical Decisions
- [ ] Decision 1?
- [ ] Decision 2?
```

Map every acceptance criterion from SPEC.md to test cases. Follow existing patterns from similar components.
