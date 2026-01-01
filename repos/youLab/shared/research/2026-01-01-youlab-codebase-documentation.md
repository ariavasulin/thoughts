---
date: 2026-01-01T04:20:00-08:00
researcher: ariasulin
git_commit: 70314e15449c8c3306b1da483a13e094c5d7f4f2
branch: main
repository: YouLab
topic: "YouLab Codebase Documentation - Complete Technical Reference"
tags: [documentation, codebase, architecture, reference, comprehensive]
status: complete
last_updated: 2026-01-01
last_updated_by: ariasulin
---

# YouLab Codebase Documentation

**Date**: 2026-01-01T04:20:00-08:00
**Researcher**: ariasulin
**Git Commit**: 70314e15449c8c3306b1da483a13e094c5d7f4f2
**Branch**: main
**Repository**: YouLab

---

## 1. Overview

YouLab is a course delivery platform with personalized AI tutoring. The first course is college essay coaching.

### Architecture

```
OpenWebUI (Pipe) → LettaStarter HTTP Service → Letta Server → Claude API
                                             ↘ Strategy Agent (RAG)
```

| Component | Technology | Purpose |
|-----------|------------|---------|
| Frontend | OpenWebUI | Chat interface with Pipe extension system |
| Pipeline | Python Pipe | Context extraction, user routing |
| HTTP Service | FastAPI (port 8100) | Agent management, chat endpoints |
| Agent Framework | Letta v0.16.1 | Persistent memory, agent execution |
| LLM | Claude API (via OpenAI compat) | Response generation |

### Implementation Status

| Phase | Status | Notes |
|-------|--------|-------|
| Phase 1: HTTP Service | **COMPLETE** | All endpoints + streaming + strategy agent |
| Phase 2: User Identity | **COMPLETE** | Absorbed into Phase 1 |
| Phase 3: Honcho | NOT STARTED | No code, no dependencies |
| Phase 4: Thread Context | NOT STARTED | Infrastructure only |
| Phase 5: Curriculum | NOT STARTED | No code exists |
| Phase 6: Background Worker | NOT STARTED | No code exists |
| Phase 7: Onboarding | NOT STARTED | No code exists |

---

## 2. Project Structure

```
src/letta_starter/
├── __init__.py
├── main.py                    # CLI entry point
├── agents/
│   ├── __init__.py
│   ├── base.py               # BaseAgent with memory + tracing
│   ├── default.py            # Factory functions + AgentRegistry
│   └── templates.py          # AgentTemplate + TUTOR_TEMPLATE
├── config/
│   ├── __init__.py
│   └── settings.py           # Pydantic settings (Settings, ServiceSettings)
├── memory/
│   ├── __init__.py
│   ├── blocks.py             # PersonaBlock, HumanBlock schemas
│   ├── manager.py            # MemoryManager orchestration
│   └── strategies.py         # Rotation strategies
├── observability/
│   ├── __init__.py
│   ├── logging.py            # Structured logging config
│   ├── metrics.py            # LLMMetrics dataclass
│   └── tracing.py            # Tracer context manager
├── pipelines/
│   ├── __init__.py
│   └── letta_pipe.py         # OpenWebUI Pipe integration
└── server/
    ├── __init__.py
    ├── cli.py                # letta-server entry point
    ├── main.py               # FastAPI app + endpoints
    ├── agents.py             # AgentManager class
    ├── schemas.py            # Request/response models
    ├── tracing.py            # Langfuse integration
    └── strategy/
        ├── __init__.py
        ├── manager.py        # StrategyManager singleton
        ├── router.py         # /strategy/* endpoints
        └── schemas.py        # Strategy request/response models

tests/
├── conftest.py
├── test_memory.py
├── test_pipe.py
├── test_templates.py
└── test_server/
    ├── conftest.py
    ├── test_agents.py
    ├── test_endpoints.py
    ├── test_schemas.py
    ├── test_tracing.py
    └── test_strategy/
        ├── conftest.py
        ├── test_endpoints.py
        └── test_manager.py
```

---

## 3. HTTP Service

### FastAPI Application

**Location**: `src/letta_starter/server/main.py`

```python
app = FastAPI(
    title="LettaStarter Service",
    description="HTTP service for YouLab AI tutoring",
    version="0.1.0",
    lifespan=lifespan,
)
```

### Endpoints

| Method | Path | Handler | Purpose |
|--------|------|---------|---------|
| GET | `/health` | `main.py:67-75` | Health check with Letta connection status |
| POST | `/agents` | `main.py:79-108` | Create agent for user |
| GET | `/agents/{agent_id}` | `main.py:111-121` | Get agent by ID |
| GET | `/agents` | `main.py:124-147` | List agents (optional `user_id` filter) |
| POST | `/chat` | `main.py:151-206` | Send message (synchronous) |
| POST | `/chat/stream` | `main.py:209-234` | Send message (SSE streaming) |
| POST | `/strategy/documents` | `strategy/router.py:44-58` | Upload to archival |
| POST | `/strategy/ask` | `strategy/router.py:61-77` | Query strategy agent |
| GET | `/strategy/documents` | `strategy/router.py:80-95` | Search archival |
| GET | `/strategy/health` | `strategy/router.py:98-107` | Strategy agent status |

### Request/Response Schemas

**Location**: `src/letta_starter/server/schemas.py`

```python
# Agent Management
class CreateAgentRequest(BaseModel):
    user_id: str
    agent_type: str = "tutor"
    user_name: str | None = None

class AgentResponse(BaseModel):
    agent_id: str
    user_id: str
    agent_type: str
    agent_name: str
    created_at: datetime | None = None

# Chat
class ChatRequest(BaseModel):
    agent_id: str
    message: str
    chat_id: str | None = None
    chat_title: str | None = None

class ChatResponse(BaseModel):
    response: str
    agent_id: str

class StreamChatRequest(BaseModel):
    agent_id: str
    message: str
    chat_id: str | None = None
    chat_title: str | None = None
    enable_thinking: bool = True
```

### AgentManager

**Location**: `src/letta_starter/server/agents.py:15-303`

Manages per-user Letta agents with caching.

**Agent Naming**: `youlab_{user_id}_{agent_type}`

**Key Methods**:

| Method | Lines | Purpose |
|--------|-------|---------|
| `create_agent()` | 80-127 | Create agent from template |
| `get_agent_id()` | 62-78 | Lookup by user_id/type |
| `send_message()` | 162-168 | Sync message to agent |
| `stream_message()` | 170-199 | SSE streaming to agent |
| `rebuild_cache()` | 47-60 | Discover existing agents on startup |

**Streaming Implementation** (`agents.py:170-249`):

```python
# Letta chunk types → SSE events
reasoning_message  → {"type": "status", "content": "Thinking..."}
tool_call_message  → {"type": "status", "content": "Using {tool}..."}
assistant_message  → {"type": "message", "content": "..."}
stop_reason        → {"type": "done"}
```

Metadata stripping (`agents.py:251-284`) removes Letta-appended JSON (`follow_ups`, `title`, `tags`).

---

## 4. Memory System

### Memory Blocks

**Location**: `src/letta_starter/memory/blocks.py`

#### PersonaBlock (`blocks.py:28-131`)

Defines agent identity - stable across sessions.

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `name` | str | "Assistant" | Agent display name |
| `role` | str | (required) | Primary role |
| `capabilities` | list[str] | [] | What agent can do |
| `tone` | str | "professional" | Communication style |
| `verbosity` | str | "concise" | Response length |
| `constraints` | list[str] | [] | What NOT to do |
| `expertise` | list[str] | [] | Specializations |

**Serialization Format**:
```
[IDENTITY] YouLab Essay Coach | AI tutor for essays
[CAPABILITIES] Guide self-discovery, Brainstorm topics, Provide feedback
[EXPERTISE] College admissions, Personal narrative, Reflective writing
[STYLE] warm, adaptive
[CONSTRAINTS] Never write essays; Ask clarifying questions first
```

#### HumanBlock (`blocks.py:134-274`)

User context - dynamic, frequently updated.

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `name` | str \| None | None | User's name |
| `role` | str \| None | None | User's profession |
| `current_task` | str \| None | None | Active work |
| `session_state` | SessionState | IDLE | Current state |
| `preferences` | list[str] | [] | Learned preferences |
| `context_notes` | list[str] | [] | Recent context (rolling) |
| `facts` | list[str] | [] | Important facts |

**SessionState Enum**:
- `IDLE`, `ACTIVE_TASK`, `WAITING_INPUT`, `THINKING`, `ERROR_RECOVERY`

### Memory Strategies

**Location**: `src/letta_starter/memory/strategies.py`

#### ContextMetrics (`strategies.py:18-50`)

```python
@dataclass
class ContextMetrics:
    persona_chars: int
    human_chars: int
    persona_max: int
    human_max: int

    @property
    def persona_usage(self) -> float  # 0-1
    @property
    def human_usage(self) -> float    # 0-1
    @property
    def total_usage(self) -> float    # 0-1
```

#### Rotation Strategies

| Strategy | Threshold | Best For |
|----------|-----------|----------|
| `AggressiveRotation` | 70% | Long sessions, many short tasks |
| `PreservativeRotation` | 90% | Complex multi-step tasks |
| `AdaptiveRotation` | 80% (adjusts) | General use (default) |

**AdaptiveRotation** learns from patterns:
- Threshold increases if rotating too often (avg < 0.5)
- Threshold decreases if rarely rotating (avg > 0.9)
- Maintains history of last 20 rotations

### MemoryManager

**Location**: `src/letta_starter/memory/manager.py:27-309`

Orchestrates memory lifecycle.

**Key Methods**:

| Method | Purpose |
|--------|---------|
| `get_persona_block()` | Retrieve and deserialize persona |
| `get_human_block()` | Retrieve and deserialize human |
| `update_persona(persona)` | Serialize and update in Letta |
| `update_human(human)` | Serialize, check rotation, update |
| `set_task(task, context)` | High-level task management |
| `learn_preference(pref)` | Record user preference |
| `learn_fact(fact)` | Record user fact |
| `search_archival(query)` | Search archival memory |

---

## 5. Pipeline Integration

**Location**: `src/letta_starter/pipelines/letta_pipe.py`

### Pipe Class (`letta_pipe.py:18-237`)

OpenWebUI extension for YouLab tutoring.

#### Configuration (Valves)

```python
class Valves(BaseModel):
    LETTA_SERVICE_URL: str = "http://host.docker.internal:8100"
    AGENT_TYPE: str = "tutor"
    ENABLE_LOGGING: bool = True
    ENABLE_THINKING: bool = True
```

#### Message Flow

1. **Extract Context** (`letta_pipe.py:120-143`)
   - `user_id` from `__user__["id"]`
   - `user_name` from `__user__["name"]`
   - `chat_id` from `__metadata__["chat_id"]`
   - `chat_title` via `Chats.get_chat_by_id()`

2. **Ensure Agent** (`letta_pipe.py:73-110`)
   - Query `/agents?user_id=X`
   - Create via `POST /agents` if not found

3. **Stream Response** (`letta_pipe.py:162-176`)
   - Connect to `POST /chat/stream` via SSE
   - Transform events for OpenWebUI

#### SSE Event Handling (`letta_pipe.py:199-236`)

| Event Type | OpenWebUI Action |
|------------|------------------|
| `status` | Show "Processing..." indicator |
| `message` | Stream content to chat |
| `done` | Mark complete |
| `error` | Display error message |

---

## 6. Strategy Agent

**Location**: `src/letta_starter/server/strategy/`

Singleton RAG-enabled agent for project knowledge.

### StrategyManager (`strategy/manager.py:29-218`)

Unlike AgentManager (per-user), StrategyManager maintains one shared agent.

**Agent Name**: `"YouLab-Support"`

**Persona** (instructs archival search):
```
You are a YouLab project strategist with comprehensive knowledge stored in your archival memory.

CRITICAL: Before answering ANY question about YouLab:
1. Use archival_memory_search to find relevant documentation
2. Cite the sources you found in your response
3. If no relevant documents found, say so explicitly
```

**Key Methods**:

| Method | Purpose |
|--------|---------|
| `ensure_agent()` | Get or create singleton agent |
| `upload_document(content, tags)` | Add to archival memory |
| `ask(question)` | Query with RAG |
| `search_documents(query, limit)` | Direct archival search |

### Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /strategy/documents` | Upload document |
| `POST /strategy/ask` | Ask question |
| `GET /strategy/documents?query=X` | Search documents |
| `GET /strategy/health` | Check agent status |

---

## 7. Agent Templates

**Location**: `src/letta_starter/agents/templates.py`

### AgentTemplate Class

```python
class AgentTemplate(BaseModel):
    type_id: str         # "tutor"
    display_name: str    # "College Essay Coach"
    description: str     # Template description
    persona: PersonaBlock
    human: HumanBlock    # Usually empty, filled during onboarding
```

### TUTOR_TEMPLATE (`templates.py:19-47`)

```python
TUTOR_TEMPLATE = AgentTemplate(
    type_id="tutor",
    display_name="College Essay Coach",
    persona=PersonaBlock(
        name="YouLab Essay Coach",
        role="AI tutor specializing in college application essays",
        capabilities=[
            "Guide students through self-discovery exercises",
            "Help brainstorm and develop essay topics",
            "Provide constructive feedback on drafts",
            "Support emotional journey of college applications",
        ],
        expertise=["College admissions", "Personal narrative", "Reflective writing"],
        tone="warm",
        verbosity="adaptive",
        constraints=[
            "Never write essays for students",
            "Always ask clarifying questions before giving advice",
            "Celebrate small wins and progress",
        ],
    ),
)
```

### AgentTemplateRegistry (`templates.py:50-76`)

Global registry with register/get/list operations.

```python
templates = AgentTemplateRegistry()  # Global instance
templates.get("tutor")  # → TUTOR_TEMPLATE
```

---

## 8. Agent Factories

**Location**: `src/letta_starter/agents/default.py`

### Factory Functions

| Function | Persona | Use Case |
|----------|---------|----------|
| `create_default_agent()` | DEFAULT_PERSONA | General assistant |
| `create_coding_agent()` | CODING_ASSISTANT_PERSONA | Development |
| `create_research_agent()` | RESEARCH_ASSISTANT_PERSONA | Research |
| `create_custom_agent()` | Custom | Fully configurable |

### Pre-configured Personas (`blocks.py:278-321`)

**DEFAULT_PERSONA**: General-purpose, friendly, adaptive verbosity

**CODING_ASSISTANT_PERSONA**: CodeHelper, professional, detailed, Python/JS/Testing expertise

**RESEARCH_ASSISTANT_PERSONA**: Researcher, professional, detailed, methodology expertise

### AgentRegistry (`default.py:160-261`)

Multi-agent coordination for managing multiple agent instances.

```python
registry = AgentRegistry(client, tracer)
registry.create_and_register("my-agent", "coding")
agent = registry.get("my-agent")
```

---

## 9. BaseAgent

**Location**: `src/letta_starter/agents/base.py:17-216`

Base class with integrated memory and observability.

### Initialization

```python
agent = BaseAgent(
    name="my-agent",
    persona=my_persona,
    human=my_human,
    client=letta_client,
    tracer=optional_tracer,
    max_memory_chars=1500,
)
```

Creates agent in Letta with serialized memory blocks. Falls back to existing agent if creation fails.

### Key Methods

| Method | Purpose |
|--------|---------|
| `send_message(message, session_id)` | Send with tracing |
| `update_context(task, note)` | Update memory context |
| `learn(preference, fact)` | Record user info |
| `get_memory_summary()` | Get memory stats |
| `search_memory(query, limit)` | Search archival |

---

## 10. Configuration

**Location**: `src/letta_starter/config/settings.py`

### Settings Class (`settings.py:9-98`)

General application settings (no prefix).

| Variable | Default | Purpose |
|----------|---------|---------|
| `LETTA_BASE_URL` | `http://localhost:8283` | Letta server |
| `LLM_PROVIDER` | `openai` | LLM provider |
| `LLM_MODEL` | `gpt-4` | Default model |
| `LOG_LEVEL` | `INFO` | Logging level |
| `LANGFUSE_ENABLED` | `false` | Tracing toggle |
| `CORE_MEMORY_MAX_CHARS` | `1500` | Memory limit |

### ServiceSettings Class (`settings.py:106-154`)

HTTP service settings with `YOULAB_SERVICE_` prefix.

| Variable | Default | Purpose |
|----------|---------|---------|
| `YOULAB_SERVICE_HOST` | `127.0.0.1` | Bind host |
| `YOULAB_SERVICE_PORT` | `8100` | Bind port |
| `YOULAB_SERVICE_LETTA_BASE_URL` | `http://localhost:8283` | Letta server |
| `YOULAB_SERVICE_LANGFUSE_ENABLED` | `true` | Tracing |

---

## 11. Observability

### Structured Logging

**Location**: `src/letta_starter/observability/logging.py`

Uses structlog with JSON output for production.

```python
log = structlog.get_logger()
log.info("event_name", key="value", count=42)
```

### LLM Metrics

**Location**: `src/letta_starter/observability/metrics.py`

```python
@dataclass
class LLMMetrics:
    prompt_tokens: int = 0
    completion_tokens: int = 0
    total_tokens: int = 0
    latency_ms: float = 0
    model: str = ""
```

### Tracer

**Location**: `src/letta_starter/observability/tracing.py`

Context manager for LLM call tracing.

```python
tracer = get_tracer()
with tracer.trace_llm_call("model", "operation", agent_name="test") as metrics:
    response = call_llm()
    metrics.prompt_tokens = response.usage.prompt_tokens
```

### Langfuse Integration

**Location**: `src/letta_starter/server/tracing.py`

```python
@contextmanager
def trace_chat(user_id, agent_id, chat_id, metadata):
    # Creates Langfuse trace with UUID
    # Records generation spans
    # Auto-flushes on exit
```

---

## 12. Testing

### Test Structure

| Test File | Lines | Coverage |
|-----------|-------|----------|
| `test_memory.py` | 210 | Memory blocks, strategies |
| `test_templates.py` | 165 | Agent templates |
| `test_pipe.py` | 373 | Pipeline integration |
| `test_server/test_endpoints.py` | 379 | HTTP endpoints |
| `test_server/test_agents.py` | 521 | AgentManager |
| `test_server/test_schemas.py` | 151 | Request/response models |
| `test_server/test_tracing.py` | 181 | Langfuse tracing |
| `test_server/test_strategy/*` | 427 | Strategy agent |

### Running Tests

```bash
make verify-agent  # Full: lint + typecheck + tests (minimal output)
make check-agent   # Quick: lint + typecheck only
make test-agent    # Tests only
```

Agent-optimized scripts implement "swallow success, show failure" pattern.

---

## 13. Dependencies

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `letta` | >=0.6.0 | Agent framework |
| `pydantic` | >=2.0.0 | Data validation |
| `pydantic-settings` | >=2.0.0 | Env config |
| `fastapi` | >=0.115.0 | HTTP framework |
| `uvicorn` | >=0.32.0 | ASGI server |
| `httpx` | >=0.27.0 | HTTP client |
| `httpx-sse` | >=0.4.0,<0.5.0 | SSE streaming |
| `structlog` | >=24.1.0 | Logging |
| `langfuse` | >=2.0.0 | LLM tracing |

### Dev Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pytest` | >=8.0.0 | Testing |
| `pytest-asyncio` | >=0.23.0 | Async tests |
| `pytest-cov` | >=4.1.0 | Coverage |
| `ruff` | >=0.4.0 | Linting |
| `basedpyright` | >=1.22.0 | Type checking |
| `pre-commit` | >=3.7 | Git hooks |

### Entry Points

```bash
letta-starter   # Interactive CLI
letta-server    # HTTP service
```

---

## 14. Tooling

### Makefile Targets

| Target | Purpose |
|--------|---------|
| `make setup` | Install deps + pre-commit hooks |
| `make lint-fix` | Auto-fix lint issues |
| `make verify-agent` | Full verification (agent-optimized) |
| `make check-agent` | Quick lint + typecheck |
| `make test-agent` | Tests only |

### Pre-commit Hooks

1. `ruff-check` - Lint with auto-fix
2. `ruff-format` - Code formatting
3. `basedpyright` - Type checking
4. `pytest` - Full test suite

### Ruff Configuration

- Line length: 100
- Target: Python 3.11
- Select: ALL rules
- Ignores: 40+ opinionated rules

### Basedpyright Configuration

- Mode: strict
- Relaxed: Missing imports/stubs for Letta, Langfuse

---

## 15. Letta SDK Patterns

The codebase uses Letta v0.16.1 API patterns:

```python
# Client
from letta_client import Letta
client = Letta(base_url="http://localhost:8283")

# Create agent
agent = client.agents.create(
    name="agent-name",
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-ada-002",
    memory_blocks=[
        {"label": "persona", "value": "..."},
        {"label": "human", "value": "..."},
    ],
    metadata={"key": "value"},
)

# Send message
response = client.agents.messages.create(
    agent_id=agent.id,
    input="Hello",
)

# Stream message
with client.agents.messages.stream(
    agent_id=agent.id,
    input="Hello",
    enable_thinking="true",
) as stream:
    for chunk in stream:
        process(chunk)

# Archival memory
client.insert_archival_memory(agent_id, memory="content")
results = client.get_archival_memory(agent_id, query="search", limit=5)
```

---

## 16. Thoughts Directory

Documentation stored separately via `humanlayer thoughts`.

```
thoughts/
├── shared/
│   ├── youlab-project-context.md      # Central project doc
│   ├── plans/                          # Implementation plans
│   │   └── 2025-12-26-youlab-technical-foundation.md
│   ├── research/                       # Research docs
│   └── handoffs/general/               # Session handoffs
└── global/shared/reference/            # API references
    ├── honcho-reference.md
    └── open-web-ui-pipes.md
```

### Key Documents

- **youlab-project-context.md**: Architecture decisions, open questions
- **2025-12-26-youlab-technical-foundation.md**: Master 7-phase plan
- **honcho-reference.md**: Honcho API reference (for Phase 3)

---

## Related Research

- `thoughts/shared/research/2026-01-01-technical-foundation-progress-analysis.md` - Implementation status
- `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` - Phase 1 details
- `thoughts/shared/research/2025-12-31-phase-2-consolidated-reference.md` - Phase 2-4 specs

---

## Quick Reference

### Start HTTP Service

```bash
# Start Letta server first
letta server

# Start YouLab service
uv run letta-server
```

### Create Agent

```bash
curl -X POST http://localhost:8100/agents \
  -H "Content-Type: application/json" \
  -d '{"user_id": "user123", "agent_type": "tutor"}'
```

### Chat

```bash
curl -X POST http://localhost:8100/chat \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "...", "message": "Hello"}'
```

### Stream Chat

```bash
curl -N -X POST http://localhost:8100/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "...", "message": "Hello"}'
```
