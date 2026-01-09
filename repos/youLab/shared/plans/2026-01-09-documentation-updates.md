# Documentation Updates Implementation Plan

## Overview

Update all documentation to accurately reflect the current codebase state. Phases 1-6 are implemented, the curriculum system is complete, and agent creation is unified on TOML configuration.

## Current State Analysis

Based on comprehensive verification (see `thoughts/shared/research/2026-01-09-documentation-verification.md`):

- **4 Critical Issues**: README phase status outdated, schema duplication (being resolved), Architecture.md missing curriculum, wrong config path
- **5 Moderate Issues**: Missing curriculum endpoints, missing AgentManager method, wrong line number, missing enum value, missing field
- **4 Minor Issues**: Missing tools field, get_all method, helper function, server files

### Key Discoveries:
- `docs/README.md:36-44` - Shows Phases 3-6 as "Not Started" but they're complete
- `docs/Architecture.md:218-269` - Missing entire `curriculum/` directory
- `docs/HTTP-Service.md` - Missing 5 curriculum endpoints
- `docs/Honcho.md:180` - Line reference says 220-272, actual is 310-363

## Desired End State

All documentation accurately reflects the implemented system:
- Phase status tables show Phases 1-6 as Complete
- README and Roadmap have identical status information
- Project structure includes curriculum system
- All HTTP endpoints documented
- Deprecated code clearly marked
- All line number references accurate

**Verification:**
1. Phase status tables match between README.md and Roadmap.md
2. All referenced files/lines exist at documented locations
3. All HTTP endpoints in docs match `src/letta_starter/server/` routers

## What We're NOT Doing

- **Not rewriting entire docs** - targeted updates only
- **Not adding new conceptual documentation** - just fixing inaccuracies
- **Not documenting future Phase 7** - only implemented features

## Implementation Approach

Update docs in logical order: high-level status first, then architecture, then specific component docs.

---

## Phase 1: Fix Phase Status Tables

### Overview
Update README.md and Roadmap.md to show Phases 1-6 as Complete with identical information.

### Changes Required:

#### 1. Update README.md Current Status table
**File**: `docs/README.md`
**Lines**: 36-44

Replace the current status table with:

```markdown
## Current Status

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1: HTTP Service | **Complete** | FastAPI service with agent management and streaming |
| Phase 2: User Identity | **Complete** | Per-user agents via OpenWebUI integration |
| Phase 3: Honcho | **Complete** | Message persistence for theory-of-mind modeling |
| Phase 4: Thread Context | **Complete** | Chat title extraction and metadata flow |
| Phase 5: Curriculum | **Complete** | TOML-based course definitions with hot-reload |
| Phase 6: Background Worker | **Complete** | Scheduled Honcho queries with memory enrichment |
| Phase 7: Onboarding | Not Started | Student setup flow |
```

#### 2. Update Roadmap.md Phase Overview table
**File**: `docs/Roadmap.md`
**Lines**: 41-50

Replace with identical information:

```markdown
## Phase Overview

| Phase | Name | Status | Dependencies |
|-------|------|--------|--------------|
| 1 | HTTP Service | **Complete** | - |
| 2 | User Identity & Routing | **Complete** | Phase 1 |
| 3 | Honcho Integration | **Complete** | Phase 1 |
| 4 | Thread Context | **Complete** | Phase 1 |
| 5 | Curriculum System | **Complete** | Phase 4 |
| 6 | Background Worker | **Complete** | Phase 3 |
| 7 | Student Onboarding | Not Started | Phase 5 |
```

#### 3. Update Roadmap.md Current Status ASCII diagram
**File**: `docs/Roadmap.md`
**Lines**: 19-35

Replace with:

```markdown
## Current Status

```
┌─────────────────────────────────────────────────────────────────┐
│                         Completed                                │
├─────────────────────────────────────────────────────────────────┤
│  Phase 1: HTTP Service ✓                                        │
│  Phase 2: User Identity ✓ (absorbed into Phase 1)               │
│  Phase 3: Honcho Integration ✓                                  │
│  Phase 4: Thread Context ✓                                      │
│  Phase 5: Curriculum System ✓                                   │
│  Phase 6: Background Worker ✓                                   │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────────┐
                    │     Next: Phase 7        │
                    └──────────────────────────┘
```
```

#### 4. Update Roadmap.md Phase 5 section
**File**: `docs/Roadmap.md`
**Lines**: 168-203

Update section header and content:

```markdown
## Phase 5: Curriculum System (Complete)

Load course definitions from TOML files.

### Deliverables

- [x] Define curriculum in TOML with course.toml and modules/
- [x] Parse into Pydantic schemas (CourseConfig, ModuleConfig, LessonConfig)
- [x] Hot-reload on API endpoint
- [x] Dynamic memory block generation from schema
- [x] HTTP endpoints for curriculum management

### Key Files

- `src/letta_starter/curriculum/schema.py`
- `src/letta_starter/curriculum/loader.py`
- `src/letta_starter/curriculum/blocks.py`
- `src/letta_starter/server/curriculum.py`
- `config/courses/college-essay/course.toml`
```

#### 5. Update Roadmap.md Phase 6 section
**File**: `docs/Roadmap.md`
**Lines**: 205-222

Update section:

```markdown
## Phase 6: Background Worker (Complete)

Query Honcho dialectic and update agent memory on schedule or manual trigger.

### Deliverables

- [x] BackgroundAgentRunner execution engine
- [x] MemoryEnricher for external memory updates
- [x] Audit trails in archival memory
- [x] HTTP endpoints for manual triggers
- [x] TOML configuration for background agents

### Key Files

- `src/letta_starter/background/runner.py`
- `src/letta_starter/memory/enricher.py`
- `src/letta_starter/server/background.py`
```

### Success Criteria:

#### Automated Verification:
- [ ] No markdown syntax errors: `npx markdownlint docs/README.md docs/Roadmap.md`

#### Manual Verification:
- [ ] README.md and Roadmap.md phase tables are identical
- [ ] ASCII diagram shows correct phases

---

## Phase 2: Update Architecture Documentation

### Overview
Fix Architecture.md to include curriculum directory and correct config paths.

### Changes Required:

#### 1. Add curriculum to project structure
**File**: `docs/Architecture.md`
**Lines**: 218-269

Update the project structure to include all directories:

```markdown
## Project Structure

```
src/letta_starter/
├── agents/              # Agent creation and management
│   ├── base.py          # BaseAgent class (deprecated)
│   ├── default.py       # Factory functions (deprecated)
│   └── templates.py     # AgentTemplate (deprecated)
│
├── background/          # Background agent system
│   └── runner.py        # BackgroundAgentRunner execution engine
│
├── config/              # Configuration
│   └── settings.py      # Settings, ServiceSettings
│
├── curriculum/          # Curriculum system
│   ├── schema.py        # Full Pydantic schemas (CourseConfig, etc.)
│   ├── loader.py        # TOML loading and caching
│   └── blocks.py        # Dynamic memory block generation
│
├── honcho/              # Message persistence + dialectic
│   ├── __init__.py      # Exports HonchoClient
│   └── client.py        # HonchoClient, query_dialectic
│
├── memory/              # Memory system
│   ├── blocks.py        # PersonaBlock, HumanBlock (deprecated)
│   ├── manager.py       # MemoryManager (deprecated)
│   ├── strategies.py    # Rotation strategies (deprecated)
│   └── enricher.py      # MemoryEnricher for external updates
│
├── observability/       # Logging and tracing
│   ├── logging.py       # Structured logging
│   ├── metrics.py       # LLMMetrics
│   └── tracing.py       # Tracer context manager
│
├── pipelines/           # OpenWebUI integration
│   └── letta_pipe.py    # Pipe class
│
├── server/              # HTTP service
│   ├── main.py          # FastAPI app
│   ├── agents.py        # AgentManager
│   ├── background.py    # Background agent endpoints
│   ├── curriculum.py    # Curriculum endpoints
│   ├── schemas.py       # Request/response models
│   ├── tracing.py       # Langfuse integration
│   └── strategy/        # Strategy agent subsystem
│       ├── manager.py   # StrategyManager
│       ├── router.py    # FastAPI router
│       └── schemas.py   # Strategy schemas
│
├── tools/               # Agent tools
│   ├── dialectic.py     # query_honcho tool
│   └── memory.py        # edit_memory_block tool
│
└── main.py              # CLI entry point

config/
└── courses/             # TOML course configurations
    ├── default/         # Default agent configuration
    │   └── course.toml
    └── college-essay/   # College essay course
        ├── course.toml
        └── modules/
            ├── 01-self-discovery.toml
            ├── 02-topic-development.toml
            └── 03-drafting.toml
```
```

### Success Criteria:

#### Automated Verification:
- [ ] All documented directories exist: `test -d src/letta_starter/curriculum`

#### Manual Verification:
- [ ] Project structure matches actual filesystem

---

## Phase 3: Add Curriculum Endpoints to HTTP-Service.md

### Overview
Document the 5 curriculum endpoints and the `create_agent_from_curriculum` method.

### Changes Required:

#### 1. Add Curriculum Endpoints section
**File**: `docs/HTTP-Service.md`

Add after Background Endpoints section (around line 436):

```markdown
---

## Curriculum Endpoints

The curriculum system provides course configuration management.

**Location**: `src/letta_starter/server/curriculum.py`

### List Courses

```http
GET /curriculum/courses
```

Lists all available courses.

**Response**:
```json
{
  "courses": ["default", "college-essay"],
  "count": 2
}
```

---

### Get Course Details

```http
GET /curriculum/courses/{course_id}
```

Returns course metadata and structure.

**Response**:
```json
{
  "id": "college-essay",
  "name": "College Essay Coaching",
  "version": "1.0.0",
  "description": "AI-powered tutoring for college application essays",
  "modules": [
    {"id": "01-self-discovery", "name": "Self-Discovery", "lesson_count": 3}
  ],
  "blocks": [
    {"name": "persona", "label": "persona", "field_count": 7}
  ],
  "background_agents": ["insight-harvester"],
  "tool_count": 3
}
```

---

### Get Full Course Config

```http
GET /curriculum/courses/{course_id}/full
```

Returns complete course configuration as JSON (useful for debugging or UI editors).

**Response**: Full `CourseConfig` serialized to JSON.

---

### Get Course Modules

```http
GET /curriculum/courses/{course_id}/modules
```

Returns all modules with full lesson details.

**Response**: List of `ModuleConfig` serialized to JSON.

---

### Reload Configuration

```http
POST /curriculum/reload
```

Hot-reloads all curriculum configurations from disk.

**Response**:
```json
{
  "success": true,
  "courses_loaded": 2,
  "courses": ["default", "college-essay"],
  "message": "Configuration reloaded successfully"
}
```

---
```

#### 2. Document create_agent_from_curriculum in AgentManager section
**File**: `docs/HTTP-Service.md`

Add to AgentManager section (around line 460):

```markdown
### Curriculum-Based Agent Creation

Agents can be created from TOML course configurations:

```python
agent_id = manager.create_agent_from_curriculum(
    user_id="user123",
    course_id="college-essay",
    user_name="Alice",
    block_overrides={"human": {"name": "Alice"}},
)
```

This method:
- Loads configuration from `config/courses/{course_id}/course.toml`
- Builds memory blocks from TOML schema definitions
- Configures tools and tool rules from course config
- Uses course-specified model and embedding
- Adds course metadata to agent
```

### Success Criteria:

#### Automated Verification:
- [ ] All documented endpoints exist in router

#### Manual Verification:
- [ ] Endpoint signatures match implementation

---

## Phase 4: Fix Line Number References

### Overview
Correct the fire-and-forget line number reference in Honcho.md.

### Changes Required:

#### 1. Fix Honcho.md line reference
**File**: `docs/Honcho.md`
**Line**: ~180

Change:
```markdown
**Location**: `src/letta_starter/honcho/client.py:220-272`
```

To:
```markdown
**Location**: `src/letta_starter/honcho/client.py:310-363`
```

### Success Criteria:

#### Automated Verification:
- [ ] Function exists at documented lines: `sed -n '310,363p' src/letta_starter/honcho/client.py | grep -q "create_persist_task"`

---

## Phase 5: Add Missing Schema Documentation

### Overview
Add missing fields and methods to component documentation.

### Changes Required:

#### 1. Add SessionScope.SPECIFIC to config-schema.md
**File**: `docs/config-schema.md`
**Line**: ~177

Update session_scope documentation:

```markdown
| `session_scope` | enum | `"all"` | Scope: `"all"`, `"recent"`, `"current"`, `"specific"` |
```

#### 2. Add ModuleConfig.background to config-schema.md
**File**: `docs/config-schema.md`

Add to Module File Reference section:

```markdown
### [module.background.{agent-id}] - Module-Level Background Overrides

Modules can override background agent configuration for module-specific behavior.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| enabled | bool | inherited | Override enabled status |
| queries | array | inherited | Override or extend queries |
```

#### 3. Add tools field to Agent-System.md AgentTemplate
**File**: `docs/Agent-System.md`

Update AgentTemplate class documentation:

```markdown
### AgentTemplate Class

```python
class AgentTemplate(BaseModel):
    type_id: str          # Unique identifier ("tutor")
    display_name: str     # Human-readable ("College Essay Coach")
    description: str      # Template description
    persona: PersonaBlock # Agent identity
    human: HumanBlock     # Initial user context (usually empty)
    tools: list[Callable] # Tool functions available to agent
```
```

#### 4. Add get_all() to AgentTemplateRegistry
**File**: `docs/Agent-System.md`

Add to registry usage:

```markdown
# Get all templates
all_templates = templates.get_all()  # {"tutor": AgentTemplate(...)}
```

#### 5. Add set_letta_client() to Agent-Tools.md
**File**: `docs/Agent-Tools.md`

Add to Context Requirements section:

```markdown
### Context Requirements

The tools require client context to be set before use:

```python
from letta_starter.tools.dialectic import set_honcho_client, set_user_context
from letta_starter.tools.memory import set_letta_client

# During service initialization
set_honcho_client(honcho_client)
set_letta_client(letta_client)

# Before each conversation
set_user_context(agent_id="agent-abc", user_id="user123")
```
```

### Success Criteria:

#### Automated Verification:
- [ ] Linting passes on all markdown files

#### Manual Verification:
- [ ] All documented fields exist in code

---

## Phase 6: Add Deprecation Notes

### Overview
Document deprecated code now that TOML unification is complete.

### Changes Required:

#### 1. Add Deprecated Code section to CLAUDE.md
**File**: `CLAUDE.md`

Add after Key Files section:

```markdown
## Deprecated Code

The following modules are deprecated in favor of TOML-based configuration:

| Module | Replacement |
|--------|-------------|
| `agents/templates.py` | `config/courses/*/course.toml` |
| `agents/default.py` | `AgentManager.create_agent()` with curriculum |
| `agents/base.py` | Letta agents via `AgentManager` |
| `memory/manager.py` | Agent-driven memory via `edit_memory_block` tool |
| `memory/blocks.py` PersonaBlock/HumanBlock | Dynamic blocks from TOML schema |
| `memory/strategies.py` | Not needed with agent-driven memory |

These modules remain for backwards compatibility but emit `DeprecationWarning` on import.
```

#### 2. Update Agent-System.md with deprecation notice
**File**: `docs/Agent-System.md`

Add notice at top:

```markdown
> **Note**: The template-based agent creation (`AgentTemplate`, factory functions, `BaseAgent`) is deprecated. New code should use `AgentManager.create_agent()` which loads configuration from TOML files in `config/courses/`.
```

#### 3. Update Memory-System.md with deprecation notice
**File**: `docs/Memory-System.md`

Add notice at top:

```markdown
> **Note**: The `MemoryManager`, `PersonaBlock`, and `HumanBlock` classes are deprecated. The curriculum-based agent path uses agent-driven memory via the `edit_memory_block` tool. Memory block schemas are now defined in TOML course configurations.
```

### Success Criteria:

#### Automated Verification:
- [ ] Documentation files exist and are valid markdown

#### Manual Verification:
- [ ] Deprecation notices are clear and point to correct replacements

---

## Testing Strategy

### Automated:
- Markdown lint all changed files
- Verify referenced files exist at documented paths
- Verify line number references are accurate

### Manual Testing Steps:
1. Review README.md and Roadmap.md phase tables match
2. Compare Architecture.md structure to actual `ls -R src/letta_starter/`
3. Test each documented curl command against running server
4. Verify all code examples are syntactically correct

---

## References

- Research document: `thoughts/shared/research/2026-01-09-documentation-verification.md`
- TOML unification plan: `thoughts/shared/plans/2026-01-09-unify-toml-only-agent-creation.md`
- Actual implementation files in `src/letta_starter/`
