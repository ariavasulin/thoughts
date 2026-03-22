---
date: 2026-01-09T17:50:56+07:00
researcher: ariasulin
git_commit: 566e1e278835bf7fa5caee1170995fd59b756914
branch: main
repository: YouLab
topic: "Strategy Agent Deprecation and TOML Course Configuration Migration"
tags: [research, strategy-agent, toml-config, deprecation, rag, curriculum]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: Strategy Agent Deprecation and TOML Course Configuration Migration

**Date**: 2026-01-09 17:50:56 +07
**Researcher**: ariasulin
**Git Commit**: 566e1e278835bf7fa5caee1170995fd59b756914
**Branch**: main
**Repository**: YouLab

## Research Question

How can the strategy agent be deprecated and converted to a TOML-based course configuration?

## Summary

The strategy agent is a specialized singleton RAG agent located at `src/letta_starter/server/strategy/` that provides project knowledge queries via HTTP endpoints. Unlike the per-user curriculum-based agents, it maintains a single shared Letta agent with archival memory. To deprecate it and convert to TOML-based configuration, you would follow the established deprecation patterns in the codebase: create a TOML course configuration defining the strategy agent's behavior, add deprecation warnings to the existing Python modules, and update the HTTP endpoints to delegate to the curriculum system.

## Detailed Findings

### Strategy Agent Current Implementation

The strategy agent lives in `src/letta_starter/server/strategy/` with four files:

| File | Purpose |
|------|---------|
| `__init__.py:1-30` | Package exports |
| `schemas.py:1-42` | Pydantic request/response models |
| `manager.py:1-218` | StrategyManager singleton with RAG logic |
| `router.py:1-108` | FastAPI endpoints mounted at `/strategy` |

**Key Components:**

1. **StrategyManager** (`manager.py:35-218`)
   - Singleton pattern for shared RAG agent
   - Agent name: `"YouLab-Support"` (line 37)
   - Persona instructs agent to search archival memory before answering (`manager.py:11-24`)
   - Uses Letta's built-in archival memory for document storage

2. **Core Methods:**
   - `upload_document()` (`manager.py:107-135`) - Inserts content into archival memory with optional tags
   - `ask()` (`manager.py:137-159`) - Sends question to agent, relies on persona to trigger archival search
   - `search_documents()` (`manager.py:161-184`) - Direct archival memory query

3. **HTTP Endpoints** (`router.py`):
   - `POST /strategy/documents` - Upload document to archival memory
   - `POST /strategy/ask` - Ask question, agent uses RAG
   - `GET /strategy/documents?query=X` - Direct document search
   - `GET /strategy/health` - Check agent status

### TOML Course Configuration System

The curriculum system provides declarative agent configuration via TOML files:

**Core Components:**

| Module | Location | Purpose |
|--------|----------|---------|
| Schema | `src/letta_starter/curriculum/schema.py` | Pydantic models for TOML validation |
| Loader | `src/letta_starter/curriculum/loader.py` | TOML parsing and caching |
| Blocks | `src/letta_starter/curriculum/blocks.py` | Dynamic Pydantic model generation |

**Course Structure:**
```
config/courses/{course-id}/
  course.toml        # Main configuration
  modules/           # Optional lesson modules
```

**TOML Configuration Sections** (from `docs/config-schema.md`):

```toml
[course]
id = "strategy"
name = "Strategy Agent"
version = "1.0.0"

[agent]
model = "anthropic/claude-sonnet-4-20250514"
system = "You are a project knowledge assistant..."

[[agent.tools]]
id = "send_message"
rules = { type = "exit_loop" }

[blocks.persona]
label = "persona"
[blocks.persona.fields]
name = { type = "string", default = "YouLab-Support" }
role = { type = "string", default = "Project knowledge assistant" }
```

### Established Deprecation Patterns

The codebase has a consistent approach to deprecation documented in multiple deprecated modules:

**Pattern 1: Module-Level Warning** (most common)
```python
# At top of file after imports
import warnings

warnings.warn(
    "letta_starter.agents.templates is deprecated. "
    "Use TOML course configuration instead. "
    "See config/courses/default/course.toml for the default template.",
    DeprecationWarning,
    stacklevel=2,
)
```

Used in:
- `agents/templates.py:15-21`
- `agents/default.py:17-23`
- `agents/base.py:17-23`
- `memory/manager.py:22-28`

**Pattern 2: Class Instantiation Warning**
```python
def model_post_init(self, _context: Any, /) -> None:
    warnings.warn(
        "PersonaBlock is deprecated. Use TOML-defined blocks instead.",
        DeprecationWarning,
        stacklevel=3,  # Higher due to Pydantic internals
    )
```

Used in:
- `memory/blocks.py:109-116` (PersonaBlock)
- `memory/blocks.py:259-266` (HumanBlock)

**Pattern 3: Documentation Table**

From `CLAUDE.md:109-122`:
```markdown
## Deprecated Code

| Module | Replacement |
|--------|-------------|
| `agents/templates.py` | `config/courses/*/course.toml` |
| `agents/default.py` | `AgentManager.create_agent()` with curriculum |
```

### Key Differences: Strategy Agent vs Curriculum Agents

| Aspect | Strategy Agent | Curriculum Agents |
|--------|----------------|-------------------|
| Cardinality | Single shared agent | Per-user agents |
| Memory | Archival (RAG documents) | Working memory blocks |
| Purpose | Project knowledge queries | User tutoring sessions |
| State | Stateless (no conversation history) | Stateful with Honcho |
| Creation | At server startup | On-demand per user |

### Migration Approach from Existing Plan

The `thoughts/shared/plans/2026-01-09-unify-toml-only-agent-creation.md` document provides a comprehensive template:

**Phase 1: Create TOML Configuration**
- Create `config/courses/strategy/course.toml`
- Define agent model, system prompt, and memory blocks
- Mirror existing `STRATEGY_PERSONA` and `STRATEGY_HUMAN` from `manager.py`

**Phase 2: Unify Agent Creation**
- Create new manager that uses curriculum loader
- Delegate existing endpoints to curriculum-based agent

**Phase 3: Add Deprecation Warnings**
- Add module-level warning to `server/strategy/__init__.py`
- Add class-level warning to `StrategyManager.__init__()`

**Phase 4: Update Tests and Documentation**
- Suppress deprecation warnings in legacy tests
- Update docs with migration guide

## Code References

- `src/letta_starter/server/strategy/manager.py:35-218` - StrategyManager implementation
- `src/letta_starter/server/strategy/router.py:22-37` - Router initialization and singleton
- `src/letta_starter/curriculum/schema.py:328-378` - CourseConfig model
- `src/letta_starter/curriculum/loader.py:228-306` - CurriculumRegistry singleton
- `src/letta_starter/curriculum/blocks.py:29-86` - Dynamic block generation
- `src/letta_starter/server/agents.py:80-150` - AgentManager.create_agent() delegation pattern
- `config/courses/default/course.toml` - Default course configuration example
- `config/courses/college-essay/course.toml` - Complete course example with background agents

## Architecture Documentation

### Current Strategy Agent Architecture
```
Client → POST /strategy/ask → router.py → StrategyManager → Letta Agent
                                                          → Archival Memory (RAG)
                                                          → Claude API
```

### Proposed TOML-Based Architecture
```
config/courses/strategy/course.toml → CurriculumLoader → StrategyManager
                                                       → Letta Agent
                                                       → Archival Memory
```

### Integration Points

1. **Server Startup** (`server/main.py:38-96`):
   - Line 44: `init_strategy_manager()` called during lifespan
   - Line 82-83: Curriculum already initialized before strategy

2. **Router Mounting** (`server/main.py:107`):
   - Strategy router mounted at `/strategy` prefix

3. **No Honcho Integration**:
   - Strategy agent does not persist conversations
   - Each query is independent

## Historical Context (from thoughts/)

Relevant documents discovered:

| Document | Relevance |
|----------|-----------|
| `thoughts/shared/plans/2026-01-09-unify-toml-only-agent-creation.md` | Comprehensive migration template |
| `thoughts/shared/research/2025-12-30-strategy-rag-agent-integration.md` | Original strategy agent research |
| `thoughts/shared/handoffs/general/2025-12-30_01-08-56_strategy-rag-agent.md` | Implementation handoff |
| `thoughts/shared/plans/2026-01-08-unified-toml-curriculum-schema.md` | Unified TOML schema plan |

## Related Research

- `thoughts/shared/research/2026-01-09-toml-schema-flexibility-analysis.md` - TOML schema flexibility
- `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Memory architecture decisions

## Open Questions

1. **Document Upload Mechanism**: How should document upload (`POST /strategy/documents`) integrate with the curriculum system? Options:
   - Keep as separate API (archival memory is Letta-managed)
   - Add documents section to TOML for static content
   - Create a dedicated document management system

2. **Singleton vs Per-Course**: Should strategy agents be:
   - Global singleton (current behavior)
   - Per-course (each course has its own knowledge base)
   - Hybrid (shared base + course-specific extensions)

3. **Migration Path**: Should existing archival memory content be:
   - Preserved (agent continues with same name)
   - Migrated (new agent, import documents)
   - Reset (fresh start with new TOML config)
