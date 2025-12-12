---
description: Update the component implementation status tracker with current progress.
---

Read first:
- `.claude/COMPONENT_STATUS.md` (the tracker file)
- `components/` directory to check what exists

## Update Process

### 1. Scan Components Directory
```bash
# Find all component directories
find components -maxdepth 1 -type d -not -path components

# For each component, check for required files
ls components/<Name>/SPEC.md 2>/dev/null
ls components/<Name>/LLD.md 2>/dev/null
ls components/<Name>/*.py 2>/dev/null
```

### 2. Check File Existence
For each component, verify:
- [ ] SPEC.md exists
- [ ] LLD.md exists
- [ ] Code files exist (Python/TS)
- [ ] Tests exist
- [ ] Schema files exist

### 3. Update Status File
Edit `.claude/COMPONENT_STATUS.md`:
- Mark `✓` for completed items
- Mark `✗` for missing items
- Mark `WIP` for work in progress

### Status Symbols
- `✓` - Completed and verified
- `✗` - Not started
- `WIP` - Work in progress
- `⚠️` - Needs attention (issues found)

## Component Status Template

```markdown
## [Layer Name]

### ComponentName
- SPEC.md: [✓/✗/WIP]
- LLD.md: [✓/✗/WIP]
- Code: [✓/✗/WIP]
- Tests: [✓/✗/WIP]
- Schemas: [✓/✗/WIP]
- Notes: [Any relevant notes or blockers]
```

## Example Update

**Before**:
```markdown
## Memory Layer

### ProfileStore
- SPEC.md: ✗
- LLD.md: ✗
- Code: ✗
- Tests: ✗
```

**After** (after creating SPEC):
```markdown
## Memory Layer

### ProfileStore
- SPEC.md: ✓ (created 2025-01-08)
- LLD.md: WIP (in review)
- Code: ✗
- Tests: ✗
- Notes: Waiting for LLD approval before implementation
```

## Bulk Status Check

Use this command to quickly check all components:

```bash
# Create a status report
for component in components/*/; do
  name=$(basename "$component")
  echo "## $name"
  echo "- SPEC: $([ -f "$component/SPEC.md" ] && echo '✓' || echo '✗')"
  echo "- LLD: $([ -f "$component/LLD.md" ] && echo '✓' || echo '✗')"
  echo "- Code: $(find "$component" -name '*.py' | head -1 >/dev/null && echo '✓' || echo '✗')"
  echo "- Tests: $([ -d "$component/tests" ] && echo '✓' || echo '✗')"
  echo ""
done
```

## Integration with Workflow

**When to update**:
- After completing SPEC with /specify
- After generating LLD with /design
- After implementer finishes code
- After verifier validates tests
- After PR is merged

**Automated trigger** (optional):
Add to git hooks or CI to auto-update on merge to main.
