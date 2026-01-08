# Honcho Documentation Updates Implementation Plan

## Overview

Fix documentation discrepancies introduced by commit 864e1dc (feat: add Honcho message persistence for theory-of-mind foundation) and add comprehensive Honcho integration documentation. The codebase now has full Honcho support but documentation hasn't been updated to reflect these changes.

## Current State Analysis

### Discrepancies Identified

| Category | Files Affected | Issue |
|----------|---------------|-------|
| Health schema | Schemas.md, API.md, HTTP-Service.md | Missing `honcho_connected` field |
| Configuration | Configuration.md, Settings.md | Missing Honcho env vars |
| Project structure | CLAUDE.md, Architecture.md | Missing `honcho/` directory |
| Roadmap status | Roadmap.md | Phase 3 shows "Planned" but is implemented |
| Architecture diagram | CLAUDE.md, Architecture.md | Missing Honcho in stack |
| Sidebar | _sidebar.md | No Honcho documentation link |

### Key Discoveries:
- `src/letta_starter/honcho/client.py:21-218` - Full HonchoClient implementation exists
- `src/letta_starter/config/settings.py:155-171` - ServiceSettings has 4 Honcho fields
- `src/letta_starter/server/schemas.py:70` - HealthResponse has `honcho_connected` field
- `src/letta_starter/server/main.py:39-50` - Honcho initialized in lifespan
- `pyproject.toml:18` - `honcho-ai>=1.0.0` dependency exists
- Tests exist: `tests/test_honcho.py`, `tests/test_server_honcho.py`

## Desired End State

All documentation accurately reflects the current Honcho integration:
1. API/Schema docs show `honcho_connected` in health responses
2. Configuration docs list all Honcho environment variables
3. New `Honcho.md` page documents the integration comprehensively
4. Project structure diagrams include `honcho/` module
5. Architecture shows Honcho in the system stack
6. Roadmap reflects Phase 3 completion status

### Verification:
- All code references in docs match actual file paths and line numbers
- Health response examples include `honcho_connected: true/false`
- Environment variable names match `ServiceSettings` field names with `YOULAB_SERVICE_` prefix
- Sidebar includes link to new Honcho.md

## What We're NOT Doing

- Updating `.env.example` (separate concern, may require user input on defaults)
- Documenting Honcho SDK internals (external dependency)
- Adding Honcho-specific tests documentation (covered in Testing.md patterns)
- Changing any code (documentation-only changes)

## Implementation Approach

Follow the established documentation patterns (modeled after Strategy-Agent.md). Update files in order of user-facing impact: API accuracy first, then configuration, then structural documentation.

---

## Phase 1: Schema & API Updates

### Overview
Fix high-priority inaccuracies in API documentation where the documented schema doesn't match the actual response.

### Changes Required:

#### 1. docs/Schemas.md
**File**: `docs/Schemas.md`
**Changes**: Add `honcho_connected` field to HealthResponse

Replace lines 80-87:
```markdown
## Health Schemas

### HealthResponse

```python
class HealthResponse(BaseModel):
    status: str
    letta_connected: bool
    honcho_connected: bool = False
    version: str = "0.1.0"
```
```

#### 2. docs/API.md
**File**: `docs/API.md`
**Changes**: Update /health response example and field table

Replace lines 25-38:
```markdown
**Response**:
```json
{
  "status": "ok",
  "letta_connected": true,
  "honcho_connected": true,
  "version": "0.1.0"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` or `"degraded"` |
| `letta_connected` | boolean | Letta server reachable |
| `honcho_connected` | boolean | Honcho service reachable |
| `version` | string | Service version (e.g., `"0.1.0"`) |
```

#### 3. docs/HTTP-Service.md
**File**: `docs/HTTP-Service.md`
**Changes**: Update /health response example and add status explanation

Replace lines 38-51:
```markdown
**Response**:
```json
{
  "status": "ok",
  "letta_connected": true,
  "honcho_connected": true,
  "version": "0.1.0"
}
```

| Status | Meaning |
|--------|---------|
| `ok` | Service healthy, Letta connected |
| `degraded` | Service running, Letta unavailable |

> **Note**: `honcho_connected` indicates whether Honcho message persistence is available. The service functions without Honcho (messages won't be persisted for ToM analysis).
```

### Success Criteria:

#### Automated Verification:
- [ ] No broken markdown links: `npx markdownlint docs/*.md`
- [ ] Documentation builds without errors: `npx serve docs` (manual check that it loads)

#### Manual Verification:
- [ ] Health response examples match actual `/health` endpoint output
- [ ] Field descriptions are accurate and helpful

---

## Phase 2: Configuration Documentation

### Overview
Document all Honcho-related environment variables and settings class fields.

### Changes Required:

#### 1. docs/Configuration.md
**File**: `docs/Configuration.md`
**Changes**: Add new Honcho Configuration section after HTTP Service Variables

Insert after line 100 (after the HTTP Service Variables table):
```markdown
### Honcho Message Persistence

| Variable | Default | Description |
|----------|---------|-------------|
| `YOULAB_SERVICE_HONCHO_ENABLED` | `true` | Enable Honcho message persistence |
| `YOULAB_SERVICE_HONCHO_WORKSPACE_ID` | `youlab` | Honcho workspace identifier |
| `YOULAB_SERVICE_HONCHO_API_KEY` | `null` | Honcho API key (required for production) |
| `YOULAB_SERVICE_HONCHO_ENVIRONMENT` | `demo` | Environment: `demo`, `local`, or `production` |

---
```

Also update the Production example (around line 119) to include Honcho:
```markdown
### Production

```bash
# .env for production
LETTA_BASE_URL=http://letta-server:8283
OPENAI_API_KEY=sk-prod-key-here

LOG_LEVEL=INFO
LOG_JSON=true
SERVICE_NAME=youlab-production

LANGFUSE_ENABLED=true
LANGFUSE_PUBLIC_KEY=pk-lf-xxx
LANGFUSE_SECRET_KEY=sk-lf-xxx

# Honcho (Theory of Mind)
YOULAB_SERVICE_HONCHO_ENABLED=true
YOULAB_SERVICE_HONCHO_ENVIRONMENT=production
YOULAB_SERVICE_HONCHO_API_KEY=your-honcho-api-key

YOULAB_SERVICE_HOST=0.0.0.0
YOULAB_SERVICE_PORT=8100
```
```

#### 2. docs/Settings.md
**File**: `docs/Settings.md`
**Changes**: Add Honcho fields to ServiceSettings section

Insert after line 138 (after langfuse_host field):
```markdown
# Honcho (message persistence)
honcho_enabled: bool = True
honcho_workspace_id: str = "youlab"
honcho_api_key: str | None = None
honcho_environment: str = "demo"
```

### Success Criteria:

#### Automated Verification:
- [ ] No broken markdown links: `npx markdownlint docs/Configuration.md docs/Settings.md`

#### Manual Verification:
- [ ] Environment variable names match actual `ServiceSettings` fields with `YOULAB_SERVICE_` prefix
- [ ] Default values match code in `src/letta_starter/config/settings.py:155-171`

---

## Phase 3: Create Honcho.md

### Overview
Create comprehensive Honcho integration documentation following the Strategy-Agent.md pattern.

### Changes Required:

#### 1. docs/Honcho.md (New File)
**File**: `docs/Honcho.md`
**Changes**: Create new documentation page

```markdown
# Honcho Integration

[[README|← Back to Overview]]

Honcho provides message persistence for theory-of-mind (ToM) modeling in YouLab.

## Overview

Honcho captures all chat messages for long-term analysis:
- **User messages** - What students say
- **Agent responses** - What the tutor replies
- **Session context** - Chat IDs, titles, agent types

This enables future ToM features like:
- Student behavior modeling
- Learning pattern analysis
- Personalized recommendations

```
┌─────────────────────────────────────────────────────────────┐
│                      Chat Flow                               │
│                                                              │
│  User Message ──► HTTP Service ──► Letta Server             │
│                        │                                     │
│                        ▼                                     │
│                   HonchoClient                               │
│                   (fire-and-forget)                          │
│                        │                                     │
│                        ▼                                     │
│                  Honcho Service                              │
│              (message persistence)                           │
└─────────────────────────────────────────────────────────────┘
```

## Architecture

### Honcho Concepts

| Concept | YouLab Mapping | Example |
|---------|---------------|---------|
| Workspace | Application | `youlab` |
| Peer | Message sender | `student_{user_id}`, `tutor` |
| Session | Chat thread | `chat_{chat_id}` |
| Message | Individual message | User or agent content |

### Data Model

```
Workspace: "youlab"
├── Peer: "student_user123"
│   └── Messages from this student
├── Peer: "student_user456"
│   └── Messages from this student
├── Peer: "tutor"
│   └── All agent responses
└── Session: "chat_abc123"
    └── Messages in this chat thread
```

---

## HonchoClient

**Location**: `src/letta_starter/honcho/client.py`

### Initialization

```python
from letta_starter.honcho import HonchoClient

client = HonchoClient(
    workspace_id="youlab",
    api_key=None,  # Required for production
    environment="demo",  # demo, local, or production
)
```

### Lazy Loading

The Honcho SDK client is lazily initialized on first use:

```python
@property
def client(self) -> Honcho | None:
    if self._client is None and not self._initialized:
        self._initialized = True
        # Initialize Honcho SDK...
    return self._client
```

If initialization fails (network error, invalid credentials), `client` returns `None` and persistence is silently skipped.

### Methods

#### persist_user_message()

Persist a user's message:

```python
await client.persist_user_message(
    user_id="user123",
    chat_id="chat456",
    message="Help me brainstorm essay topics",
    chat_title="Essay Brainstorming",
    agent_type="tutor",
)
```

#### persist_agent_message()

Persist an agent's response:

```python
await client.persist_agent_message(
    user_id="user123",  # Which student this was for
    chat_id="chat456",
    message="Great! Let's explore some topics...",
    chat_title="Essay Brainstorming",
    agent_type="tutor",
)
```

#### check_connection()

Verify Honcho is reachable:

```python
if client.check_connection():
    print("Honcho is available")
```

---

## Fire-and-Forget Pattern

**Location**: `src/letta_starter/honcho/client.py:220-272`

Messages are persisted asynchronously without blocking the chat response:

```python
from letta_starter.honcho.client import create_persist_task

# In chat endpoint - doesn't block response
create_persist_task(
    honcho_client=honcho,
    user_id="user123",
    chat_id="chat456",
    message="User's message",
    is_user=True,
    chat_title="My Chat",
    agent_type="tutor",
)
```

### Graceful Degradation

- If `honcho_client` is `None`, persistence is skipped
- If `chat_id` is empty, persistence is skipped
- If Honcho is unreachable, errors are logged but not raised
- Chat functionality continues regardless of Honcho status

---

## HTTP Service Integration

**Location**: `src/letta_starter/server/main.py`

### Initialization

Honcho is initialized during service startup:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... other initialization ...

    if settings.honcho_enabled:
        app.state.honcho_client = HonchoClient(
            workspace_id=settings.honcho_workspace_id,
            api_key=settings.honcho_api_key,
            environment=settings.honcho_environment,
        )
        honcho_ok = app.state.honcho_client.check_connection()
        log.info("honcho_initialized", connected=honcho_ok)
    else:
        app.state.honcho_client = None
        log.info("honcho_disabled")
```

### Health Endpoint

The `/health` endpoint reports Honcho status:

```json
{
  "status": "ok",
  "letta_connected": true,
  "honcho_connected": true,
  "version": "0.1.0"
}
```

### Chat Endpoints

Both `/chat` and `/chat/stream` persist messages:

1. **User message** - Persisted before sending to Letta
2. **Agent response** - Persisted after receiving from Letta

---

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `YOULAB_SERVICE_HONCHO_ENABLED` | `true` | Enable persistence |
| `YOULAB_SERVICE_HONCHO_WORKSPACE_ID` | `youlab` | Workspace ID |
| `YOULAB_SERVICE_HONCHO_API_KEY` | `null` | API key (production) |
| `YOULAB_SERVICE_HONCHO_ENVIRONMENT` | `demo` | Environment |

### Environments

| Environment | Use Case | API Key |
|-------------|----------|---------|
| `demo` | Development/testing | Not required |
| `local` | Local Honcho server | Not required |
| `production` | Production deployment | Required |

---

## Metadata

Messages include metadata for context:

```python
metadata = {
    "chat_id": "chat456",
    "agent_type": "tutor",
    "chat_title": "Essay Brainstorming",  # Optional
    "user_id": "user123",  # Agent messages only
}
```

---

## Related Pages

- [[Architecture]] - System overview with Honcho
- [[HTTP-Service]] - Chat endpoint details
- [[Configuration]] - Environment variables
- [[Roadmap]] - ToM integration plans
```

#### 2. docs/_sidebar.md
**File**: `docs/_sidebar.md`
**Changes**: Add Honcho link to Core Systems section

Update lines 6-12:
```markdown
- **Core Systems**
  - [HTTP Service](HTTP-Service.md)
  - [Memory System](Memory-System.md)
  - [Agent System](Agent-System.md)
  - [Pipeline Integration](Pipeline.md)
  - [Strategy Agent](Strategy-Agent.md)
  - [Honcho Integration](Honcho.md)
```

### Success Criteria:

#### Automated Verification:
- [ ] No broken markdown links: `npx markdownlint docs/Honcho.md`
- [ ] File exists at correct path

#### Manual Verification:
- [ ] Documentation follows Strategy-Agent.md pattern
- [ ] Code examples match actual implementation
- [ ] Sidebar link appears in correct section

---

## Phase 4: Structural Updates

### Overview
Update project structure documentation and architecture diagrams to include Honcho.

### Changes Required:

#### 1. CLAUDE.md
**File**: `CLAUDE.md`
**Changes**: Update Current Stack, Project Structure, and Key Files

**Current Stack** (replace lines 5-13):
```markdown
## Current Stack

```
OpenWebUI → Pipeline → HTTP Service → Letta Server → Claude API
                            ↘ Honcho (message persistence)
```

- **OpenWebUI**: Chat frontend with Pipe extension system
- **LettaStarter**: Python library providing memory management, observability, and agent factories
- **Letta**: Agent framework with persistent memory (currently single shared agent)
- **Honcho**: Message persistence for theory-of-mind modeling
```

**Project Structure** (replace lines 15-28):
```markdown
## Project Structure

```
src/letta_starter/       # Python backend
  agents/                # BaseAgent + factory functions + AgentRegistry + templates
  memory/                # Memory blocks, rotation strategies, manager
  pipelines/             # OpenWebUI Pipe integration
  server/                # FastAPI HTTP service (agent management, chat endpoints)
    strategy/            # RAG-enabled strategy agent (project knowledge queries)
  honcho/                # Honcho client for message persistence
  observability/         # Logging, metrics, tracing (Langfuse)
  config/                # Pydantic settings from env
  main.py                # CLI entry point
tests/                   # Pytest suite (including tests/test_server/)
```
```

**Key Files** (add after line 85, before Roadmap):
```markdown
- `src/letta_starter/honcho/client.py` - HonchoClient for message persistence
```

**Roadmap section** (replace lines 87-94):
```markdown
## Roadmap

Current architecture with Honcho integration:

```
OpenWebUI (Pipe) → LettaStarter HTTP Service → Letta Server → Claude API
                                             → Honcho (ToM layer)
```

**Full plan**: `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md`
```

#### 2. docs/Architecture.md
**File**: `docs/Architecture.md`
**Changes**: Update system diagram and project structure

**System Diagram** (replace lines 9-48):
```markdown
## System Overview

YouLab is built as a layered architecture where each component has a single responsibility:

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenWebUI                               │
│                   (Chat Interface)                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Pipeline (Pipe)                           │
│  • Extract user_id, chat_id from OpenWebUI                  │
│  • Ensure agent exists for user                              │
│  • Stream SSE responses back to UI                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              HTTP Service (FastAPI :8100)                    │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  AgentManager    │  │ StrategyManager  │                 │
│  │  (per-user)      │  │ (singleton RAG)  │                 │
│  └────────┬─────────┘  └────────┬─────────┘                 │
│           │                     │                            │
│           └──────────┬──────────┘                            │
│                      │                                       │
│           ┌──────────┴──────────┐                            │
│           │    HonchoClient     │                            │
│           │ (message persist)   │                            │
│           └──────────┬──────────┘                            │
└──────────────────────┼──────────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
┌─────────────────────┐   ┌─────────────────────┐
│  Letta Server       │   │   Honcho Service    │
│  (:8283)            │   │   (ToM Layer)       │
│  • Agent lifecycle  │   │   • Message store   │
│  • Core memory      │   │   • Session mgmt    │
│  • Archival memory  │   │   • Peer tracking   │
│  • Tool execution   │   │                     │
└─────────┬───────────┘   └─────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Claude API                                │
│              (via OpenAI compatibility)                      │
└─────────────────────────────────────────────────────────────┘
```
```

**Project Structure** (replace lines 209-242, add honcho directory):
```markdown
## Project Structure

```
src/letta_starter/
├── agents/              # Agent creation and management
│   ├── base.py          # BaseAgent class
│   ├── default.py       # Factory functions, AgentRegistry
│   └── templates.py     # AgentTemplate, TUTOR_TEMPLATE
│
├── config/              # Configuration
│   └── settings.py      # Settings, ServiceSettings
│
├── honcho/              # Message persistence
│   ├── __init__.py      # Exports HonchoClient
│   └── client.py        # HonchoClient, create_persist_task
│
├── memory/              # Memory system
│   ├── blocks.py        # PersonaBlock, HumanBlock
│   ├── manager.py       # MemoryManager
│   └── strategies.py    # Rotation strategies
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
│   ├── schemas.py       # Request/response models
│   ├── tracing.py       # Langfuse integration
│   └── strategy/        # Strategy agent subsystem
│
└── main.py              # CLI entry point
```
```

### Success Criteria:

#### Automated Verification:
- [ ] No broken markdown links: `npx markdownlint CLAUDE.md docs/Architecture.md`

#### Manual Verification:
- [ ] System diagram accurately reflects data flow
- [ ] Project structure matches actual directory layout
- [ ] Key files section includes honcho module

---

## Phase 5: Roadmap & Status Updates

### Overview
Update project status documentation to reflect Honcho implementation completion.

### Changes Required:

#### 1. docs/Roadmap.md
**File**: `docs/Roadmap.md`
**Changes**: Update Phase 3 status and phase table

**Phase Table** (replace lines 46-57):
```markdown
## Phase Overview

| Phase | Name | Status | Dependencies |
|-------|------|--------|--------------|
| 1 | HTTP Service | **Complete** | - |
| 2 | User Identity & Routing | Planned | Phase 1 |
| 3 | Honcho Integration | **Complete** | Phase 1 |
| 4 | Thread Context | Planned | Phase 2 |
| 5 | Curriculum Parser | Planned | Phase 4 |
| 6 | Background Worker | Planned | Phase 3 |
| 7 | Student Onboarding | Planned | Phase 5 |
```

**Phase 3 Section** (replace lines 124-148):
```markdown
## Phase 3: Honcho Integration (Complete)

Persist all messages to Honcho for theory-of-mind modeling.

### Deliverables

- [x] HonchoClient with lazy initialization
- [x] Fire-and-forget message persistence
- [x] Integration with `/chat` endpoint
- [x] Integration with `/chat/stream` endpoint
- [x] Health endpoint reports Honcho status
- [x] Graceful degradation when Honcho unavailable
- [x] Configuration via environment variables
- [x] Unit and integration tests

### Key Files

- `src/letta_starter/honcho/client.py`
- `src/letta_starter/config/settings.py` (ServiceSettings)
- `src/letta_starter/server/main.py` (lifespan, endpoints)
- `tests/test_honcho.py`
- `tests/test_server_honcho.py`

### What's NOT Included (Future Work)

- Dialectic queries from Honcho (Phase 6)
- Working representation updates (Phase 6)
- ToM-informed agent behavior (Phase 6)
```

#### 2. docs/Changelog.md
**File**: `docs/Changelog.md`
**Changes**: Add Honcho integration entry

Add at the top of the changelog (after frontmatter/title):
```markdown
## [Unreleased]

### Added
- **Honcho Integration**: Message persistence for theory-of-mind modeling
  - `HonchoClient` for async message persistence
  - Fire-and-forget pattern for non-blocking chat
  - Health endpoint reports Honcho connection status
  - Configuration via `YOULAB_SERVICE_HONCHO_*` environment variables
  - Graceful degradation when Honcho unavailable
```

### Success Criteria:

#### Automated Verification:
- [ ] No broken markdown links: `npx markdownlint docs/Roadmap.md docs/Changelog.md`

#### Manual Verification:
- [ ] Phase status accurately reflects implementation state
- [ ] Deliverables list matches actual implementation
- [ ] Changelog entry follows existing format

---

## Testing Strategy

### Automated Tests:
- Markdown linting for all modified files
- Documentation site builds without errors

### Manual Testing Steps:
1. Run `npx serve docs` and verify all pages load
2. Click through sidebar links to verify navigation
3. Compare health response example with actual `/health` output
4. Verify environment variable names match code
5. Check that code examples are syntactically correct

## References

- Verification research: `thoughts/shared/research/2026-01-06-docs-verification-honcho-integration.md`
- HonchoClient implementation: `src/letta_starter/honcho/client.py`
- ServiceSettings with Honcho: `src/letta_starter/config/settings.py:155-171`
- Server integration: `src/letta_starter/server/main.py:39-50`
- Pattern reference: `docs/Strategy-Agent.md`
