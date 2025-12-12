# Directory Structure

Complete file organization for the Personal Agent project.

---

## Overview

```
Personal-agent/
├── .claude/                    # Claude Code configuration (commands, agents, skills)
├── .specify/                   # Git Spec Kit workspace (templates, scripts, workbench)
├── components/                 # Component implementations (16 components across 4 layers)
├── usecases/                   # Use case implementations
├── docs/                       # Architecture and development documentation
├── tests/                      # Test suites
├── COMPONENT_STATUS.md         # Implementation tracker (16 components)
├── DIRECTORY_STRUCTURE.md      # This file
├── DEVELOPMENT_WORKFLOW.md     # Development workflow integration guide
└── QUICK_REFERENCE.md          # Fast lookup (roles, layers, commands)
```

---

## `.claude/` - Claude Code Configuration

**Purpose**: Claude Code setup (commands, agents, skills only - no documentation)

```
.claude/
├── commands/                       # Custom slash commands (4 files)
│   ├── primer.md                   # Repo overview + propose next step
│   ├── specify.md                  # Create SPEC (component/use case templates)
│   ├── design.md                   # Create LLD + Mermaid flowchart
│   └── flow_orchestrate.md         # Run: planner → implementer → verifier → pr-manager
│
├── agents/                         # Specialized execution roles (5 files)
│   ├── planner.md                  # Maps SPEC → tasks (reads SPEC/LLD → writes tasks.md)
│   ├── implementer.md              # Writes code/tests/schemas
│   ├── verifier.md                 # Runs tests, validates BC/schemas
│   ├── pr-manager.md               # Creates PR with evidence
│   └── architect.md                # Makes architectural decisions (ADRs, trade-offs)
│
├── skills/                         # Quick helper skills (8 files)
│   ├── explain-component.md        # Explain component with examples
│   ├── review-plan-schema.md       # Validate plan JSON structure
│   ├── review-architecture.md      # Quick architectural review checklist
│   ├── quick-fix.md                # Fast bug fixes (< 3 files, < 10 lines)
│   ├── update-component-status.md  # Update implementation tracker
│   ├── create-component-spec.md    # Quick SPEC template
│   ├── create-component-lld.md     # Quick LLD template
│   └── add-test-cases.md           # Generate test templates
│
├── CLAUDE_SETUP.md                 # Main Claude Code setup guide
├── CLAUDE.md                       # Project rules & working context
├── README.md                       # Claude Code README
└── settings.json                   # Permissions and MCP configuration
```

---

## `.specify/` - Git Spec Kit Workspace

**Purpose**: Official Git Spec Kit toolkit (commands, templates, scripts, workbench)

```
.specify/
├── commands/                       # Spec Kit slash commands (8 files)
│   ├── speckit.constitution.md     # Define project governing principles
│   ├── speckit.clarify.md          # Address underspecified areas (optional)
│   ├── speckit.plan.md             # Create technical implementation plan
│   ├── speckit.analyze.md          # Validate consistency across artifacts (optional)
│   ├── speckit.tasks.md            # Generate actionable task breakdown
│   ├── speckit.implement.md        # Execute implementation in phases
│   ├── speckit.checklist.md        # Generate quality validation checklist (optional)
│   └── speckit.taskstoissues.md    # Convert tasks to GitHub issues (optional)
│   # Note: speckit.specify.md removed - use /specify instead (custom wrapper)
│
├── templates/                      # Spec Kit templates
│   ├── spec-template-component.md  # Component SPEC template
│   ├── spec-template-usecase.md    # Use case SPEC template
│   └── tasks-template.md           # Tasks breakdown template
│
├── scripts/bash/                   # Spec Kit bash scripts
│   ├── create-new-feature.sh       # Create new feature branch + spec
│   ├── check-prerequisites.sh      # Validate prerequisites before tasks
│   ├── common.sh                   # Shared utility functions
│   ├── setup-plan.sh               # Initialize plan structure
│   ├── update-agent-context.sh     # Update agent context
│   ├── check-task-prerequisites.sh # Validate task prerequisites
│   └── get-feature-paths.sh        # Get feature directory paths
│
├── memory/                         # Project constitution
│   └── constitution.md             # Project governing principles (template)
│
└── specs/                          # Workbench for feature specs
    └── 001-feature-name/           # Example feature directory
        ├── spec.md                 # Feature specification
        ├── plan.md                 # Implementation plan
        ├── tasks.md                # Task breakdown
        └── checklists/             # Quality checklists (optional)
            ├── test.md
            ├── security.md
            └── ux.md
```

---

## `components/` - Component Implementations

**Purpose**: 16 components across 4 architectural layers

```
components/
├── Memory Layer (4 components)
│   ├── ProfileStore/
│   │   ├── SPEC.md              # Component specification (source of truth)
│   │   ├── LLD.md               # Low-level design
│   │   ├── tasks.md             # Implementation tasks
│   │   ├── diagrams/
│   │   │   └── flow.mmd         # Mermaid flowchart
│   │   ├── api/                 # API handlers
│   │   ├── service/             # Business logic (preview/execute)
│   │   ├── domain/              # Domain models
│   │   ├── adapters/            # External integrations
│   │   ├── schemas/             # JSON schemas
│   │   │   └── response.normalized.json
│   │   └── tests/               # Tests
│   │       └── test_contract.py
│   ├── ScheduleStore/
│   ├── ContextRAG/
│   └── RateLimiter/
│
├── Domain Layer (6 components)
│   ├── IntentParser/
│   ├── Resolver/
│   ├── Fetcher/
│   ├── Analyzer/
│   ├── Watcher/
│   └── Booker/
│
├── Orchestration Layer (3 components)
│   ├── PreviewOrchestrator/
│   ├── ExecuteOrchestrator/
│   └── Notifier/
│
└── Platform Layer (3 components)
    ├── n8nRuntime/
    ├── TemporalRuntime/
    └── LLMGateway/
```

---

## `usecases/` - Use Case Implementations

**Purpose**: End-to-end user flows

```
usecases/
└── schedule-meeting/
    ├── SPEC.md              # Use case specification
    ├── LLD.md               # Low-level design
    ├── tasks.md             # Implementation tasks
    └── tests/               # Integration tests
```

---

## `docs/` - Documentation

**Purpose**: Architecture and development guides

```
docs/
├── architecture/
│   ├── Project_HLD.md              # High-level design (system context, layers)
│   ├── GLOBAL_SPEC.md              # Universal contracts (Intent, Preview, Execute)
│   ├── PROJECT_STRUCTURE.md        # Component-first structure rules
│   └── MODULAR_ARCHITECTURE.md     # Modular design principles
│
└── dev/
    └── PYTHON_GUIDE.md             # Python development standards
```

---

## `tests/` - Test Suites

**Purpose**: Acceptance and contract tests

```
tests/
├── acceptance/          # End-to-end tests
└── contract/            # Component contract tests
```

---

## Key File Locations Quick Reference

### **Root Documentation Files**

| File | Purpose | Location |
|------|---------|----------|
| Component Status | Implementation tracker (16 components) | `COMPONENT_STATUS.md` |
| Directory Structure | Complete file organization | `DIRECTORY_STRUCTURE.md` |
| Integration Guide | Three-tier system explanation | `INTEGRATION_GUIDE.md` |
| Quick Reference | Fast lookup (roles, layers, commands) | `QUICK_REFERENCE.md` |
| README | Project overview | `README.md` |

---

### **Claude Code Files** (`.claude/`)

| File | Purpose | Location |
|------|---------|----------|
| Commands | Custom slash commands | `.claude/commands/*.md` |
| Agents | Specialized execution roles | `.claude/agents/*.md` |
| Skills | Quick helper skills | `.claude/skills/*.md` |
| Setup Guide | Claude Code setup documentation | `.claude/CLAUDE_SETUP.md` |
| Project Rules | Working context for Claude | `.claude/CLAUDE.md` |
| Settings | Permissions and MCP config | `.claude/settings.json` |

---

### **Git Spec Kit Files** (`.specify/`)

| File | Purpose | Location |
|------|---------|----------|
| Spec Kit Commands | Official workflow commands | `.specify/commands/speckit.*.md` |
| Templates | SPEC/task templates | `.specify/templates/*.md` |
| Bash Scripts | Workflow automation | `.specify/scripts/bash/*.sh` |
| Constitution | Project principles | `.specify/memory/constitution.md` |
| Workbench | Feature specs | `.specify/specs/###-name/` |

---

### **Component Files** (`components/`)

| File | Purpose | Location |
|------|---------|----------|
| SPEC.md | Component specification (source of truth) | `components/<Name>/SPEC.md` |
| LLD.md | Low-level design | `components/<Name>/LLD.md` |
| tasks.md | Implementation tasks | `components/<Name>/tasks.md` |
| Flowchart | Mermaid diagram | `components/<Name>/diagrams/flow.mmd` |
| API handlers | Thin handlers | `components/<Name>/api/` |
| Service | Business logic (preview/execute) | `components/<Name>/service/` |
| Domain | Domain models | `components/<Name>/domain/` |
| Adapters | External integrations | `components/<Name>/adapters/` |
| Schemas | JSON schemas | `components/<Name>/schemas/` |
| Tests | Contract tests | `components/<Name>/tests/` |

---

## Migration Summary

### **What Changed**

**Before restructuring**:
- Spec Kit commands were in `.claude/commands/speckit.*.md`
- Mixed Claude Code and Spec Kit files in same directory

**After restructuring**:
- Spec Kit commands moved to `.specify/commands/speckit.*.md`
- Clean separation: `.claude/` for Claude Code, `.specify/` for Spec Kit

### **Commands Moved**

```bash
# Moved from .claude/commands/ to .specify/commands/
speckit.constitution.md
speckit.specify.md
speckit.clarify.md
speckit.plan.md
speckit.analyze.md
speckit.tasks.md
speckit.implement.md
speckit.checklist.md
speckit.taskstoissues.md
```

### **Commands Remaining in `.claude/commands/`**

```bash
# Custom Claude commands (stay in .claude/commands/)
primer.md
specify.md
design.md
flow_orchestrate.md
```

---

## File Count Summary

| Directory | File Count | Purpose |
|-----------|-----------|---------|
| `.claude/commands/` | 4 | Custom Claude commands |
| `.claude/agents/` | 5 | Specialized execution roles |
| `.claude/skills/` | 8 | Quick helper skills |
| `.specify/commands/` | 9 | Spec Kit workflow commands |
| `.specify/templates/` | 3 | SPEC/task templates |
| `.specify/scripts/bash/` | 7 | Bash automation scripts |

**Total**: 36 organizational files for workflow automation

---

## Design Principles

### 1. **Clean Separation**
- `.claude/` = Claude Code configuration
- `.specify/` = Git Spec Kit workspace
- `components/` = Implementation (source of truth for SPEC/LLD)

### 2. **No Overlap**
- Spec Kit commands only in `.specify/commands/`
- Custom commands only in `.claude/commands/`
- Agents only in `.claude/agents/`
- Skills only in `.claude/skills/`

### 3. **Source of Truth**
- Component SPEC: `components/<Name>/SPEC.md` (not `.specify/specs/`)
- Component LLD: `components/<Name>/LLD.md` (not `.specify/specs/`)
- Workbench: `.specify/specs/` is for drafting, must promote to `components/`

### 4. **Consistent Structure**
- Every component has same structure: SPEC, LLD, tasks, diagrams, api, service, domain, adapters, schemas, tests
- Every use case has same structure: SPEC, LLD, tasks, tests
- Templates enforce consistency

---

## Quick Navigation

**To work on a component**:
1. SPEC: `components/<Name>/SPEC.md`
2. LLD: `components/<Name>/LLD.md`
3. Tasks: `components/<Name>/tasks.md`
4. Code: `components/<Name>/{api,service,domain,adapters,schemas,tests}/`

**To run a workflow**:
1. Custom: `/specify` → `/design` → `/flow_orchestrate`
2. Spec Kit: `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement`

**To get help**:
1. Setup: `.claude/CLAUDE_SETUP.md`
2. Integration: `.claude/INTEGRATION_GUIDE.md`
3. Quick ref: `.claude/QUICK_REFERENCE.md`
4. Component status: `.claude/COMPONENT_STATUS.md`
