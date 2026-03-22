---
date: 2026-01-09T17:47:57+07:00
researcher: ariasulin
git_commit: 566e1e278835bf7fa5caee1170995fd59b756914
branch: main
repository: YouLab
topic: "Two-way sync between TOML course configs and OpenWebUI"
tags: [research, codebase, toml, openwebui, sync, curriculum, memory]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: Two-Way Sync Between TOML Courses and OpenWebUI

**Date**: 2026-01-09T17:47:57+07:00
**Researcher**: ariasulin
**Git Commit**: 566e1e278835bf7fa5caee1170995fd59b756914
**Branch**: main
**Repository**: YouLab

## Research Question

How can we achieve two-way synchronization between agent/course configurations in TOML files and OpenWebUI?

## Summary

The current architecture has **one-way flow**: TOML files → HTTP Service → Letta Agents. There is no mechanism for OpenWebUI to modify TOML configurations or for runtime agent state to persist back to TOML. Several approaches are possible depending on requirements:

1. **File-based sync with hot reload** - External tool writes TOML, triggers `/curriculum/reload`
2. **New HTTP write endpoints** - Add PATCH/PUT endpoints to modify course configs
3. **OpenWebUI Functions/Actions** - Use OpenWebUI's plugin system to call sync APIs
4. **Shared database** - Store configs in database instead of TOML files
5. **Git-based workflow** - TOML in Git, CI/CD deploys changes

## Detailed Findings

### Current Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Configuration Layer                          │
│  config/courses/{id}/course.toml → CurriculumLoader → CourseConfig     │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼ (read-only at runtime)
┌─────────────────────────────────────────────────────────────────────────┐
│                            HTTP Service Layer                           │
│  AgentManager.create_agent_from_curriculum() → Letta Agent + Memory    │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            Pipeline Layer                               │
│  OpenWebUI Pipe → POST /chat/stream → SSE Response                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Existing Sync Mechanisms

| Direction | Mechanism | Status |
|-----------|-----------|--------|
| TOML → Agent | Agent creation reads curriculum | ✓ Implemented |
| Agent → Honcho | Fire-and-forget message persistence | ✓ Implemented |
| Honcho → Agent | Background worker queries | Planned (Phase 6) |
| TOML → OpenWebUI | None | ❌ Not implemented |
| OpenWebUI → TOML | None | ❌ Not implemented |
| OpenWebUI → Agent Memory | Only user_name at creation | Minimal |

### Key Components

#### 1. TOML Configuration System
- **Location**: `config/courses/{course-id}/course.toml`
- **Schema**: `src/letta_starter/curriculum/schema.py`
- **Loader**: `src/letta_starter/curriculum/loader.py`
- **Hot reload**: `POST /curriculum/reload` triggers `curriculum.reload()`

The TOML system is **read-only at runtime**. Changes require:
1. Writing to disk
2. Calling reload endpoint
3. Creating new agents (existing agents keep old config)

#### 2. OpenWebUI Pipeline Integration
- **File**: `src/letta_starter/pipelines/letta_pipe.py`
- **Data passed**: user_id, user_name, chat_id, chat_title
- **No config access**: Pipeline cannot read or modify TOML configs
- **Title capability**: `_set_chat_title()` exists but unused for sync

#### 3. Export Endpoints
- `GET /curriculum/courses` - List course IDs
- `GET /curriculum/courses/{id}/full` - Complete course as JSON
- `GET /curriculum/courses/{id}/modules` - Module definitions

#### 4. No Import Endpoints
- No `POST /curriculum/courses` to create courses
- No `PATCH /curriculum/courses/{id}` to update courses
- No endpoints to modify running agent memory blocks

### Sync Options Analysis

#### Option 1: File-Based with Hot Reload

**How it works**:
```
External Tool → Writes TOML → POST /curriculum/reload → New agents use updated config
```

**Implementation**:
- No code changes needed
- External tool (script, admin UI) writes TOML files
- Calls `POST /curriculum/reload`
- Subsequent agent creations use new config

**Limitations**:
- Existing agents not updated
- No transactional guarantees
- Requires file system access

**Code path**:
- `src/letta_starter/curriculum/loader.py:287-298` - `reload()` method
- `src/letta_starter/server/curriculum.py:147-173` - `/curriculum/reload` endpoint

#### Option 2: HTTP Write Endpoints

**New endpoints needed**:
```python
# Course management
POST   /curriculum/courses                    # Create course
PATCH  /curriculum/courses/{id}               # Update course metadata
PUT    /curriculum/courses/{id}/blocks/{name} # Replace block schema
DELETE /curriculum/courses/{id}               # Delete course

# Runtime agent updates
PATCH  /agents/{id}/memory/{block_name}       # Update agent memory block
POST   /agents/{id}/sync                      # Sync agent to current curriculum
```

**Implementation effort**: Medium (new router, schema validation, TOML writer)

**Benefits**:
- RESTful API for course management
- Could trigger agent updates
- OpenWebUI can call directly

**Complexity**:
- TOML serialization from Pydantic models
- Handling conflicts between memory formats
- Agent migration for running agents

#### Option 3: OpenWebUI Functions

OpenWebUI has a "Functions" system that can extend capabilities.

**Approach**:
- Create OpenWebUI Function that calls HTTP API
- Admin interface to manage courses
- Pipeline passes course changes to backend

**Implementation**:
```python
# OpenWebUI Function
class CourseManager:
    async def list_courses(self) -> list[dict]:
        response = await httpx.get(f"{SERVICE_URL}/curriculum/courses")
        return response.json()

    async def update_block(self, course_id: str, block_name: str, data: dict):
        # Requires Option 2 endpoints
        pass
```

**Benefits**:
- Integrated into OpenWebUI UI
- Admin access control via OpenWebUI

**Requirements**:
- Option 2 endpoints must exist first
- OpenWebUI function development

#### Option 4: Shared Database

**Architecture change**:
```
OpenWebUI DB (courses table) ← → HTTP Service reads/writes
```

**Instead of TOML**:
- Store CourseConfig as JSON in database
- HTTP service reads from DB instead of files
- OpenWebUI can modify directly

**Benefits**:
- True bidirectional sync
- Single source of truth
- Real-time updates

**Drawbacks**:
- Major architectural change
- Loses file-based version control
- Database dependency
- Migration complexity

#### Option 5: Git-Based Workflow

**Approach**:
```
Developer edits TOML → Git push → CI/CD → Server pulls → Reload
```

**Benefits**:
- Version control for courses
- Review process for changes
- Audit trail

**For OpenWebUI integration**:
- OpenWebUI admin could trigger webhook
- Webhook creates PR or direct commit
- CI/CD deploys

**Complexity**:
- CI/CD pipeline setup
- Git authentication from OpenWebUI

### Memory Layer Considerations

The architecture distinguishes three memory layers (from `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md`):

1. **Letta (Structured)**: Curriculum state, memory blocks, assessments
2. **Honcho (Emergent)**: Psychological model, theory-of-mind
3. **Background Workers (Adaptive)**: Query Honcho → Update Letta

**Sync implications**:
- TOML defines initial block schemas
- Runtime memory lives in Letta
- Syncing TOML ↔ Letta memory requires translation

**Agent-driven updates** (already implemented):
- Tool: `edit_memory_block` (`src/letta_starter/tools/memory.py`)
- Allows agent to update its own memory
- Changes persist in Letta, not TOML

### OpenWebUI Capabilities

From `src/letta_starter/pipelines/letta_pipe.py`:

**Current access**:
- `__user__`: id, name, email, role
- `__metadata__`: chat_id, message_id
- `Chats.get_chat_by_id()` - Direct DB access
- `Chats.update_chat_title_by_id()` - Write capability exists

**Extension points**:
- Valves for configuration
- Could add more valves for course selection
- Could add metadata for curriculum context

**Valve additions that could help**:
```python
class Valves(BaseModel):
    COURSE_ID: str = Field(default="college-essay", description="Course to use")
    ALLOW_COURSE_SWITCHING: bool = Field(default=False)
```

## Recommendations

### For Read-Only Sync (Course Export)

Already implemented:
```bash
# Export course config
curl http://localhost:8100/curriculum/courses/college-essay/full | jq .
```

### For Write Sync (Minimal Implementation)

Add endpoint to update memory blocks at runtime:

```python
# In src/letta_starter/server/main.py
@app.patch("/agents/{agent_id}/memory/{block_label}")
async def update_memory_block(
    agent_id: str,
    block_label: str,
    request: UpdateBlockRequest,
) -> UpdateBlockResponse:
    """Update a specific memory block on an agent."""
    # Validate agent exists
    # Get current block
    # Merge updates
    # Call Letta SDK to update
```

### For Full Bidirectional Sync

Phase approach:
1. Add export/import endpoints for TOML
2. Build OpenWebUI admin panel using those endpoints
3. Add runtime memory update endpoints
4. Implement agent migration for config changes

## Code References

- `src/letta_starter/curriculum/schema.py:328-378` - CourseConfig model
- `src/letta_starter/curriculum/loader.py:43-107` - load_course() method
- `src/letta_starter/curriculum/loader.py:287-298` - reload() method
- `src/letta_starter/server/curriculum.py:121-134` - GET /courses/{id}/full endpoint
- `src/letta_starter/server/curriculum.py:147-173` - POST /curriculum/reload endpoint
- `src/letta_starter/server/agents.py:110-225` - create_agent_from_curriculum()
- `src/letta_starter/pipelines/letta_pipe.py:73-97` - _set_chat_title() unused capability
- `src/letta_starter/tools/memory.py` - edit_memory_block tool

## Architecture Documentation

### Current Data Flow
```
TOML Files (source of truth)
    ↓ read at startup + reload
CurriculumRegistry (in-memory cache)
    ↓ on agent creation
AgentManager → Letta Agent (runtime state)
    ↓ every message
Honcho (observation layer)
    ↓ planned
Background Workers → Memory enrichment
```

### Gap: No Write Path
```
OpenWebUI UI
    ✗ Cannot modify courses
    ✗ Cannot update agent memory directly
    ✓ Can only trigger chat which may cause agent self-updates
```

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-08-phase-4-thread-context.md` - Added `_set_chat_title()` but deferred memory sync
- `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Defines tri-layer memory architecture
- `thoughts/shared/research/2026-01-06-http-service-architectural-significance.md` - Explains HTTP service as control plane
- `thoughts/shared/plans/2026-01-08-phase-6-background-agents.md` - Plans for Honcho → Letta memory enrichment

## Related Research

- `thoughts/shared/research/2026-01-09-toml-schema-flexibility-analysis.md` - TOML schema design
- `thoughts/shared/plans/2026-01-09-unify-toml-only-agent-creation.md` - Consolidating agent creation through TOML

## Open Questions

1. **Which sync option to prioritize?** File-based is lowest effort, database is most flexible
2. **Should existing agents update when TOML changes?** Currently no, requires migration mechanism
3. **Who should own course editing?** Developers via TOML, or admins via UI?
4. **Is OpenWebUI the right admin UI?** Or should there be a separate course management system?
5. **How to handle conflicts?** If agent memory diverges from TOML schema
