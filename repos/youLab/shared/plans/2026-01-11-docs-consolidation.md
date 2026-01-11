# Docs Consolidation Implementation Plan

## Overview

Consolidate the `/docs` directory from 29 files to 18 files by removing redundancy, trimming deprecated content, and improving organization for both human and AI agent navigation.

## Current State Analysis

| Category | Files | Issue |
|----------|-------|-------|
| Config | Configuration.md + Settings.md | Same env vars in different formats |
| API | HTTP-Service.md + API.md | ~400 lines duplicate endpoint docs |
| Agent/Memory | Agent-System.md + Memory-System.md | ~50-60% deprecated content |
| Dev | Development.md + Testing.md + Tooling.md | Overlapping content |
| Letta | 7 files (~55KB) | Duplicates official docs, will become stale |
| Project | Roadmap.md + Changelog.md | Phases 1-6 complete, mostly historical |

## Desired End State

```
docs/ (18 files, down from 29)
├── README.md              # Enhanced with "Start Here" paths
├── Architecture.md        # Unchanged
├── Quickstart.md          # Unchanged
├── HTTP-Service.md        # Implementation notes only (no endpoint duplication)
├── Agent-System.md        # Trimmed - current TOML approach + migration guide
├── Memory-System.md       # Trimmed - current approach + brief migration guide
├── Agent-Tools.md         # Unchanged
├── Background-Agents.md   # Unchanged
├── Pipeline.md            # Unchanged
├── Honcho.md              # Unchanged
├── Configuration.md       # Merged: env vars + settings classes
├── Development.md         # Merged: dev + testing + tooling
├── API.md                 # Unchanged - canonical endpoint reference
├── Schemas.md             # Unchanged
├── Letta-Integration.md   # New: YouLab-specific Letta patterns
├── Letta-Reference.md     # New: quick ref + links to official docs
├── Roadmap.md             # Compressed: current + next phases only
├── config-schema.md       # Unchanged
└── _sidebar.md            # Updated navigation
```

### Files to Delete
- Settings.md (→ Configuration.md)
- Testing.md (→ Development.md)
- Tooling.md (→ Development.md)
- Strategy-Agent.md (→ HTTP-Service.md)
- Changelog.md (→ Roadmap.md)
- Letta-Concepts.md (→ Letta-Integration.md)
- Letta-SDK.md (→ Letta-Integration.md)
- Letta-Streaming.md (→ Letta-Integration.md)
- Letta-REST-API.md (→ Letta-Reference.md)
- Letta-Tools.md (→ Letta-Reference.md)
- Letta-Archival.md (→ Letta-Reference.md)
- Letta-Versions.md (→ Letta-Reference.md)

## What We're NOT Doing

- Not changing the docs site engine (Docsify)
- Not reorganizing into subdirectories
- Not removing any currently-useful content
- Not changing config-schema.md (actively maintained, referenced by TOML configs)

## Implementation Approach

Execute in phases to maintain a working docs site throughout. Each phase is independently deployable.

---

## Phase 1: Configuration Merge

### Overview
Merge Settings.md into Configuration.md to create a single source of truth for all configuration documentation.

### Changes Required:

#### 1. Update Configuration.md
**File**: `docs/Configuration.md`

Add Settings class documentation after the environment variables section:

```markdown
---

## Settings Classes

YouLab uses two Pydantic settings classes for type-safe configuration.

**Location**: `src/letta_starter/config/settings.py`

| Class | Prefix | Use Case |
|-------|--------|----------|
| `Settings` | (none) | CLI and library |
| `ServiceSettings` | `YOULAB_SERVICE_` | HTTP service |

### Settings Class

General application settings, loaded from `.env`:

```python
from letta_starter.config import get_settings

settings = get_settings()
print(settings.letta_base_url)
```

### ServiceSettings Class

HTTP service-specific settings with `YOULAB_SERVICE_` prefix:

```python
from letta_starter.config.settings import ServiceSettings

settings = ServiceSettings()
uvicorn.run(app, host=settings.host, port=settings.port)
```

### Testing with Settings

```python
from unittest.mock import patch

def test_with_custom_settings():
    with patch.object(settings, "langfuse_enabled", False):
        result = get_langfuse()
        assert result is None
```

---

## Related Pages

- [[Development]] - Local setup
- [[Quickstart]] - Getting started
```

#### 2. Delete Settings.md
**File**: `docs/Settings.md`
**Action**: Delete file

#### 3. Update _sidebar.md
**File**: `docs/_sidebar.md`
**Changes**: Remove Settings.md entry, keep Configuration.md

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds: `npx serve docs` and visit http://localhost:3000
- [ ] No broken links in Configuration.md
- [ ] Settings.md is deleted

#### Manual Verification:
- [ ] Configuration.md contains all env var tables
- [ ] Configuration.md contains Settings class usage patterns
- [ ] Navigation works correctly

---

## Phase 2: Development Docs Merge

### Overview
Merge Testing.md and Tooling.md into Development.md for a single comprehensive development guide.

### Changes Required:

#### 1. Update Development.md
**File**: `docs/Development.md`

Restructure to include all development content:

```markdown
# Development Guide

[[README|← Back to Overview]]

Complete guide for developing and contributing to YouLab.

## Prerequisites

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) package manager
- Running Letta server (for integration testing)

---

## Quick Start

```bash
# Clone and setup
git clone <repo-url>
cd YouLab
make setup  # Installs deps + pre-commit hooks
```

---

## Development Workflow

1. **Make changes** in `src/letta_starter/`
2. **Run lint fix** frequently: `make lint-fix`
3. **Verify before commit**: `make verify-agent`

Pre-commit hooks run `make verify` automatically.

---

## Commands Reference

### Verification (Agent-Optimized)

| Command | Description |
|---------|-------------|
| `make check-agent` | Quick: lint + typecheck, minimal output |
| `make test-agent` | Tests only, fail-fast |
| `make verify-agent` | Full: lint + typecheck + tests |

### Standard Commands

| Command | Description |
|---------|-------------|
| `make lint` | Run ruff check |
| `make lint-fix` | Auto-fix lint issues |
| `make typecheck` | Run basedpyright |
| `make test` | Run pytest with coverage |
| `make coverage-html` | Generate HTML coverage report |

---

## Tooling Configuration

All tools are configured in `pyproject.toml`.

### uv (Package Manager)

```bash
uv sync              # Install dependencies
uv run <command>     # Run command in venv
uv add <package>     # Add dependency
```

### Ruff (Linter + Formatter)

```bash
make lint-fix        # Auto-fix and format
```

Key settings:
- Line length: 100
- Target: Python 3.11
- Uses ALL rules with sensible ignores

### Basedpyright (Type Checker)

```bash
make typecheck
```

Strict mode with external library handling disabled.

### Pre-commit

Hooks run `make verify` before each commit. Bypass with `--no-verify` (emergency only).

---

## Testing

### Running Tests

```bash
make test-agent      # Quick, fail-fast
make test            # Full with coverage
make coverage-html   # Generate HTML report
```

### Test Structure

```
tests/
├── conftest.py           # Shared fixtures
├── test_memory.py        # Memory block tests
├── test_templates.py     # Agent template tests
├── test_pipe.py          # Pipeline tests
└── test_server/          # HTTP service tests
```

### Writing Tests

```python
def test_persona_block_serialization(sample_persona_data):
    """Test PersonaBlock serializes to memory string."""
    persona = PersonaBlock(**sample_persona_data)
    result = persona.to_memory_string()
    assert "[IDENTITY]" in result

@pytest.mark.asyncio
async def test_agent_manager_cache():
    """Test AgentManager caches agent IDs."""
    manager = AgentManager(mock_client)
    agent_id = await manager.get_or_create_agent("user1", "tutor")
    cached_id = await manager.get_or_create_agent("user1", "tutor")
    assert agent_id == cached_id
```

### Mocking Letta

```python
@pytest.fixture
def mock_letta_client():
    client = MagicMock()
    client.agents.create = AsyncMock(return_value=MagicMock(id="agent-123"))
    return client
```

---

## Code Style

- Type annotations everywhere
- Prefer composition over inheritance
- Keep functions focused and small

| Type | Convention | Example |
|------|------------|---------|
| Classes | PascalCase | `AgentManager` |
| Functions | snake_case | `get_agent` |
| Constants | UPPER_SNAKE | `DEFAULT_PORT` |
| Private | _prefix | `_cache` |

---

## IDE Setup

### VS Code

Recommended extensions: Python, Pylance, Ruff

```json
{
    "python.defaultInterpreterPath": ".venv/bin/python",
    "[python]": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "charliermarsh.ruff"
    }
}
```

---

## Common Issues

| Issue | Solution |
|-------|----------|
| `Letta connection failed` | Ensure `letta server` is running |
| `Type errors` | Run `make typecheck` for details |
| `Import errors` | Run `uv sync` to update deps |

---

## Related Pages

- [[Configuration]] - Environment variables
- [[Quickstart]] - Getting started
```

#### 2. Delete Testing.md and Tooling.md
**Files**: `docs/Testing.md`, `docs/Tooling.md`
**Action**: Delete both files

#### 3. Update _sidebar.md
**File**: `docs/_sidebar.md`
**Changes**: Remove Testing.md and Tooling.md entries

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds without errors
- [ ] Testing.md and Tooling.md deleted
- [ ] No broken links

#### Manual Verification:
- [ ] Development.md is comprehensive but navigable
- [ ] All make commands documented
- [ ] Test patterns included

---

## Phase 3: Agent-System.md Trim

### Overview
Remove deprecated template/factory documentation, keeping only the current TOML-based approach and migration guide.

### Changes Required:

#### 1. Rewrite Agent-System.md
**File**: `docs/Agent-System.md`

Replace with focused content (~150 lines instead of ~530):

```markdown
# Agent System

[[README|← Back to Overview]]

YouLab uses Letta agents managed through `AgentManager` with TOML-based configuration.

## AgentManager

The `AgentManager` class manages per-user agents.

**Location**: `src/letta_starter/server/agents.py`

### Key Methods

```python
class AgentManager:
    async def create_agent(
        self,
        user_id: str,
        course_config: CourseConfig,
        user_name: str | None = None,
    ) -> AgentState:
        """Create a new agent from TOML course configuration."""

    async def get_or_create_agent(
        self,
        user_id: str,
        course_config: CourseConfig,
    ) -> AgentState:
        """Get existing agent or create new one."""

    async def send_message(
        self,
        user_id: str,
        message: str,
        course_config: CourseConfig,
    ) -> str:
        """Send message to user's agent."""
```

### Agent Naming

Agents use the pattern: `youlab_{user_id}_{course_id}`

### Creating Agents

Agents are created from TOML course configurations:

```python
from letta_starter.curriculum.loader import load_course_config
from letta_starter.server.agents import AgentManager

config = load_course_config("college-essay")
manager = AgentManager(letta_client)
agent = await manager.create_agent(user_id="user123", course_config=config)
```

See [[config-schema]] for TOML configuration format.

---

## Migration from Legacy APIs

> **Note**: The template-based agent creation is deprecated. Existing code should migrate to TOML-based configuration.

### Deprecated Modules

| Deprecated | Replacement |
|------------|-------------|
| `agents/templates.py` | `config/courses/*/course.toml` |
| `agents/default.py` | `AgentManager.create_agent()` |
| `agents/base.py` | Direct Letta agents via AgentManager |
| `memory/blocks.py` | TOML-defined blocks |

### Migration Steps

1. Define agent persona in `[agent.blocks.persona]` in course TOML
2. Use `AgentManager.create_agent()` with `CourseConfig`
3. Remove imports from deprecated modules

### Before (Deprecated)

```python
from letta_starter.agents.templates import TUTOR_TEMPLATE
from letta_starter.agents.default import create_custom_agent

agent = create_custom_agent(client, name="tutor", ...)
```

### After (Current)

```toml
# config/courses/my-course/course.toml
[agent]
type = "tutor"
system_prompt = "You are an AI tutor..."

[agent.blocks.persona]
label = "persona"
value = """
[IDENTITY] Course Coach | AI tutor
[CAPABILITIES] Guide students, Provide feedback
"""
```

```python
config = load_course_config("my-course")
agent = await manager.create_agent(user_id, config)
```

---

## Related Pages

- [[config-schema]] - TOML configuration format
- [[HTTP-Service]] - Agent endpoints
- [[Memory-System]] - Memory block concepts
- [[Letta-Integration]] - Letta SDK patterns
```

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds
- [ ] No broken internal links

#### Manual Verification:
- [ ] Current AgentManager API is clear
- [ ] Migration guide is preserved
- [ ] ~70% size reduction achieved

---

## Phase 4: Memory-System.md Trim

### Overview
Remove deprecated MemoryManager documentation, keeping current TOML-based block approach.

### Changes Required:

#### 1. Rewrite Memory-System.md
**File**: `docs/Memory-System.md`

Replace with focused content (~120 lines instead of ~390):

```markdown
# Memory System

[[README|← Back to Overview]]

YouLab agents use Letta's two-tier memory architecture with blocks defined in TOML configuration.

## Architecture

```
┌─────────────────────────────────────┐
│        Core Memory (In-Context)     │
│  Always visible in every prompt     │
│  - persona block (agent identity)   │
│  - human block (user context)       │
│  - custom blocks from TOML          │
└─────────────────────────────────────┘
                  │
                  │ Agent-driven rotation
                  ▼
┌─────────────────────────────────────┐
│     Archival Memory (Vector DB)     │
│  Semantic search on-demand          │
│  - Unlimited capacity               │
│  - Historical context               │
└─────────────────────────────────────┘
```

## Defining Memory Blocks

Memory blocks are defined in course TOML files:

```toml
# config/courses/my-course/course.toml

[agent.blocks.persona]
label = "persona"
description = "Agent identity and capabilities"
value = """
[IDENTITY] Essay Coach | AI tutor
[CAPABILITIES] Guide discovery, Brainstorm topics
[STYLE] warm, adaptive
"""

[agent.blocks.human]
label = "human"
description = "User context, updated during conversations"
value = """
[USER] {name} | Student
[TASK] None
[STATE] idle
"""
```

See [[config-schema]] for full TOML schema.

## Agent-Driven Memory

Agents update their own memory using the `edit_memory_block` tool:

```python
# Agent can call this tool during conversations
edit_memory_block(
    block_label="human",
    action="replace",
    old_content="[TASK] None",
    new_content="[TASK] Essay brainstorming"
)
```

## Memory Block Format

YouLab uses a token-efficient bracket notation:

```
[IDENTITY] YouLab Coach | AI tutor for college essays
[CAPABILITIES] Guide discovery; Brainstorm topics; Provide feedback
[STYLE] warm, adaptive
[CONSTRAINTS] Never write essays for students
```

This format:
- Maximizes information density
- Is easily parseable by agents
- Fits within Letta's ~2000 char block limit

---

## Migration from Legacy APIs

> **Note**: `MemoryManager`, `PersonaBlock`, and `HumanBlock` classes are deprecated.

| Deprecated | Replacement |
|------------|-------------|
| `memory/manager.py` | Agent-driven via `edit_memory_block` tool |
| `memory/blocks.py` | TOML-defined blocks |
| `memory/strategies.py` | Not needed with agent-driven memory |

---

## Related Pages

- [[config-schema]] - TOML block definitions
- [[Agent-System]] - Agent management
- [[Letta-Integration]] - Letta memory concepts
```

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds
- [ ] No broken internal links

#### Manual Verification:
- [ ] Current TOML-based approach is clear
- [ ] Migration guide preserved
- [ ] ~70% size reduction achieved

---

## Phase 5: HTTP-Service.md Deduplication

### Overview
Remove duplicate endpoint documentation (keep in API.md), merge Strategy-Agent.md content, focus on implementation details.

### Changes Required:

#### 1. Rewrite HTTP-Service.md
**File**: `docs/HTTP-Service.md`

Restructure to focus on implementation (~300 lines instead of ~800):

```markdown
# HTTP Service

[[README|← Back to Overview]]

The HTTP service is a FastAPI application providing agent management and chat functionality.

## Quick Reference

| Property | Value |
|----------|-------|
| Default Host | `127.0.0.1` |
| Default Port | `8100` |
| Framework | FastAPI |
| Entry Point | `src/letta_starter/server/main.py` |

```bash
# Start the service
uv run letta-server

# With custom settings
YOULAB_SERVICE_HOST=0.0.0.0 YOULAB_SERVICE_PORT=8000 uv run letta-server
```

For complete API reference, see [[API]].

---

## Service Architecture

```
HTTP Service (FastAPI)
├── /health           - Health check
├── /agents/*         - Agent CRUD (AgentManager)
├── /chat/*           - Chat endpoints
├── /strategy/*       - Strategy agent (StrategyManager)
├── /background/*     - Background agents
├── /curriculum/*     - Course management
└── /sync/*           - File sync (OpenWebUI → Letta)
```

---

## AgentManager

Manages per-user agents with caching.

**Location**: `src/letta_starter/server/agents.py`

### Naming Convention

```python
agent_name = f"youlab_{user_id}_{course_id}"
# Example: youlab_user123_college-essay
```

### Caching

```python
# Cache structure
_cache: dict[tuple[str, str], str]  # (user_id, course_id) -> agent_id

# On startup, cache is rebuilt from Letta
async def lifespan(app):
    count = await app.state.agent_manager.rebuild_cache()
```

---

## Strategy Agent

A singleton RAG-enabled agent for project-wide knowledge queries.

**Location**: `src/letta_starter/server/strategy/`

| Aspect | User Agents | Strategy Agent |
|--------|-------------|----------------|
| Count | 1 per user | 1 per system |
| Name | `youlab_{user_id}_{course}` | `YouLab-Support` |
| Purpose | Tutoring | Project knowledge |
| Endpoints | `/agents/*`, `/chat/*` | `/strategy/*` |

### StrategyManager

```python
manager = StrategyManager(letta_base_url="http://localhost:8283")
agent_id = manager.ensure_agent()  # Gets or creates singleton
manager.upload_document(content="...", tags=["docs"])
response = manager.ask("How does streaming work?")
```

---

## Streaming Implementation

The service translates Letta streaming chunks to SSE events.

### Chunk Mapping

| Letta Type | SSE Event |
|------------|-----------|
| `reasoning_message` | `{"type": "status", "content": "Thinking..."}` |
| `tool_call_message` | `{"type": "status", "content": "Using {tool}..."}` |
| `assistant_message` | `{"type": "message", "content": "..."}` |
| `stop_reason` | `{"type": "done"}` |
| `ping` | `": keepalive\n\n"` |

### Metadata Stripping

Letta appends JSON metadata to messages. The service strips it:

```python
# Before: "Hello!{"follow_ups": ["Tell me more"]}"
# After:  "Hello!"
```

---

## Tracing

All chat requests are traced via Langfuse:

```python
with trace_chat(
    user_id=user_id,
    agent_id=request.agent_id,
    chat_id=request.chat_id,
) as trace_ctx:
    response = manager.send_message(...)
```

See [[Configuration]] for Langfuse settings.

---

## Related Pages

- [[API]] - Complete endpoint reference
- [[Schemas]] - Request/response models
- [[Configuration]] - Service settings
- [[Background-Agents]] - Background agent system
```

#### 2. Delete Strategy-Agent.md
**File**: `docs/Strategy-Agent.md`
**Action**: Delete file

#### 3. Update _sidebar.md
Remove Strategy-Agent.md entry.

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds
- [ ] Strategy-Agent.md deleted
- [ ] No broken links

#### Manual Verification:
- [ ] Implementation details preserved
- [ ] No endpoint duplication with API.md
- [ ] Strategy agent documented inline

---

## Phase 6: Letta Consolidation

### Overview
Consolidate 7 Letta reference files into 2 focused documents.

### Changes Required:

#### 1. Create Letta-Integration.md
**File**: `docs/Letta-Integration.md`

New file focusing on YouLab-specific Letta patterns:

```markdown
# Letta Integration

[[README|← Back to Overview]]

How YouLab integrates with the Letta agent framework.

## Overview

Letta provides stateful AI agents with persistent memory. YouLab leverages:

- **Persistent state** - Agent state survives restarts
- **Two-tier memory** - Core (in-context) + Archival (vector search)
- **Streaming** - Real-time response delivery
- **Tool system** - Custom tools for memory editing

For complete Letta documentation, see [docs.letta.com](https://docs.letta.com).

---

## Client Initialization

```python
from letta_client import Letta

# Self-hosted (YouLab default)
client = Letta(base_url="http://localhost:8283")

# From settings
from letta_starter.config import get_settings
settings = get_settings()
client = Letta(base_url=str(settings.letta_base_url))
```

---

## Agent Creation

YouLab creates agents with course-specific configuration:

```python
agent = client.agents.create(
    name="youlab_user123_college-essay",
    memory_blocks=[
        {"label": "persona", "value": persona_string},
        {"label": "human", "value": human_string},
    ],
    model="anthropic/claude-sonnet-4-20250514",
    embedding="openai/text-embedding-3-small",
    metadata={
        "youlab_user_id": "user123",
        "youlab_course_id": "college-essay",
    },
)
```

---

## Memory Patterns

### Core Memory Blocks

Always visible in agent context:

```python
blocks = client.agents.core_memory.list(agent_id)
human_block = next(b for b in blocks if b.label == "human")

# Update block
client.agents.core_memory.update(
    agent_id=agent_id,
    block_id=human_block.id,
    value=new_value,
)
```

### Archival Memory

Long-term storage with semantic search:

```python
# Insert
client.agents.archival_memory.insert(
    agent_id=agent_id,
    text="[ARCHIVED 2025-01-01]\nContext...",
)

# Search
results = client.agents.archival_memory.search(
    agent_id=agent_id,
    query="essay topics",
    limit=5,
)
```

---

## Streaming

### SDK Pattern

```python
with client.agents.messages.stream(
    agent_id=agent.id,
    input="Hello!",
    stream_tokens=False,
    include_pings=True,
) as stream:
    for chunk in stream:
        if chunk.message_type == "assistant_message":
            print(chunk.content)
```

### Message Types

| Type | Purpose |
|------|---------|
| `reasoning_message` | Agent thinking |
| `tool_call_message` | Tool invocation |
| `assistant_message` | Response to user |
| `stop_reason` | Stream complete |
| `usage_statistics` | Token counts |

---

## Tool System

### Creating Tools

```python
def my_tool(arg: str) -> str:
    """
    Tool description.

    Args:
        arg: Input argument

    Returns:
        Result string
    """
    return f"Processed: {arg}"

tool = client.tools.upsert_from_function(func=my_tool)
```

### Attaching to Agent

```python
client.agents.tools.attach(agent_id=agent.id, tool_id=tool.id)
```

---

## Best Practices

### 1. Reuse Client Instances

```python
# Good
client = Letta()
for user in users:
    agent = client.agents.retrieve(...)

# Bad - creates overhead
for user in users:
    client = Letta()  # Don't do this
```

### 2. Cache Agent IDs

```python
_cache: dict[str, str] = {}

async def get_agent_id(user_id: str) -> str:
    if user_id not in _cache:
        agent = await find_agent(user_id)
        _cache[user_id] = agent.id
    return _cache[user_id]
```

### 3. Use Structured Memory

```python
# Good - parseable
"[USER] Alice | Student\n[TASK] Essay brainstorming"

# Bad - unstructured
"Alice is a student working on essay brainstorming"
```

---

## Related Pages

- [[Letta-Reference]] - Quick reference and links
- [[Agent-System]] - YouLab agent management
- [[Memory-System]] - Memory block patterns
```

#### 2. Create Letta-Reference.md
**File**: `docs/Letta-Reference.md`

Quick reference with links to official docs:

```markdown
# Letta Quick Reference

[[README|← Back to Overview]]

Quick reference for Letta APIs. For detailed documentation, see [docs.letta.com](https://docs.letta.com).

## Installation

```bash
pip install letta         # Server
pip install letta-client  # Python SDK
```

## Server

```bash
letta server              # Start on :8283
```

## Client

```python
from letta_client import Letta

client = Letta(base_url="http://localhost:8283")
```

---

## Common Operations

### Agents

```python
# Create
agent = client.agents.create(
    name="my-agent",
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-3-small",
    memory_blocks=[
        {"label": "persona", "value": "I am..."},
        {"label": "human", "value": "User info..."},
    ],
)

# List
agents = client.agents.list()

# Get
agent = client.agents.retrieve(agent_id)

# Delete
client.agents.delete(agent_id)
```

### Messages

```python
# Synchronous
response = client.agents.messages.create(
    agent_id=agent.id,
    input="Hello!"
)

# Streaming
with client.agents.messages.stream(
    agent_id=agent.id,
    input="Hello!",
) as stream:
    for chunk in stream:
        print(chunk)
```

### Memory

```python
# Core memory
blocks = client.agents.core_memory.list(agent_id)
client.agents.core_memory.update(agent_id, block_id, value="...")

# Archival memory
client.agents.archival_memory.insert(agent_id, text="...")
results = client.agents.archival_memory.search(agent_id, query="...")
```

### Tools

```python
# Create from function
tool = client.tools.upsert_from_function(func=my_function)

# Attach to agent
client.agents.tools.attach(agent_id, tool_id)
```

---

## Model Formats

```python
# LLM models
"openai/gpt-4o-mini"
"anthropic/claude-3-5-sonnet"
"ollama/llama2:7b"

# Embedding models
"openai/text-embedding-3-small"
"letta/letta-free"
```

---

## Environment Variables

```bash
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
LETTA_BASE_URL=http://localhost:8283
```

---

## Official Documentation

| Topic | Link |
|-------|------|
| Getting Started | [docs.letta.com/quickstart](https://docs.letta.com/quickstart) |
| Core Concepts | [docs.letta.com/core-concepts](https://docs.letta.com/core-concepts) |
| Agent Memory | [docs.letta.com/guides/agents/memory](https://docs.letta.com/guides/agents/memory) |
| Streaming | [docs.letta.com/guides/agents/streaming](https://docs.letta.com/guides/agents/streaming) |
| Tools | [docs.letta.com/guides/agents/tools](https://docs.letta.com/guides/agents/tools) |
| REST API | [docs.letta.com/api-reference](https://docs.letta.com/api-reference) |

---

## Version Compatibility

YouLab requires `letta>=0.6.0`. Current stable: 0.10.x

See [GitHub releases](https://github.com/letta-ai/letta/releases) for changelog.

---

## Related Pages

- [[Letta-Integration]] - YouLab-specific patterns
- [[Configuration]] - Environment setup
```

#### 3. Delete Old Letta Files
**Files to delete**:
- `docs/Letta-Concepts.md`
- `docs/Letta-SDK.md`
- `docs/Letta-REST-API.md`
- `docs/Letta-Streaming.md`
- `docs/Letta-Tools.md`
- `docs/Letta-Archival.md`
- `docs/Letta-Versions.md`

#### 4. Update _sidebar.md
Replace 7 Letta entries with 2 new ones.

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds
- [ ] All 7 old Letta files deleted
- [ ] 2 new Letta files created
- [ ] No broken links

#### Manual Verification:
- [ ] Letta-Integration.md covers YouLab-specific patterns
- [ ] Letta-Reference.md provides quick lookup + official doc links
- [ ] ~55KB reduced to ~15KB

---

## Phase 7: Roadmap Compression

### Overview
Merge Changelog.md into Roadmap.md and compress completed phases.

### Changes Required:

#### 1. Rewrite Roadmap.md
**File**: `docs/Roadmap.md`

Compress to focus on current/future (~150 lines instead of ~430 combined):

```markdown
# Roadmap

[[README|← Back to Overview]]

Implementation roadmap for YouLab's AI tutoring platform.

## Current Status

| Phase | Status |
|-------|--------|
| 1-6 | **Complete** - HTTP Service, Honcho, Curriculum, Background Workers |
| 7: Onboarding | Not Started |

---

## Next: Phase 7 - Student Onboarding

Handle new student first-time experience.

### Goals

- Detect new students
- Initialize agent with onboarding context
- Guide through setup flow
- Transition to Module 1

### Onboarding Flow

1. Welcome message
2. Collect basic info (name, goals)
3. Explain system capabilities
4. First step setup

---

## Completed Phases Summary

### Phase 1: HTTP Service
FastAPI service with agent management, streaming chat, strategy agent.

### Phase 2: User Identity
Per-user agents via OpenWebUI integration, agent naming convention.

### Phase 3: Honcho Integration
Message persistence for theory-of-mind modeling, graceful degradation.

### Phase 4: Thread Context
Chat title extraction and metadata flow.

### Phase 5: Curriculum System
TOML-based course definitions with hot-reload.

### Phase 6: Background Worker
Scheduled Honcho queries with memory enrichment.

---

## Out of Scope

- Full user management UI
- Multi-facilitator support
- Course marketplace
- Production deployment infrastructure
- Automated pedagogical testing

---

## Version History

| Version | Date | Milestone |
|---------|------|-----------|
| 0.1.0 | 2025-12-31 | Phase 1: HTTP Service |
| - | 2026-01 | Phases 2-6 complete |

---

## Related Pages

- [[Architecture]] - System design
- [[HTTP-Service]] - Service implementation
```

#### 2. Delete Changelog.md
**File**: `docs/Changelog.md`
**Action**: Delete file

#### 3. Update _sidebar.md
Remove Changelog.md entry.

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds
- [ ] Changelog.md deleted
- [ ] No broken links

#### Manual Verification:
- [ ] Current phase is prominent
- [ ] Completed phases are summarized
- [ ] Version history preserved

---

## Phase 8: Sidebar and README Update

### Overview
Update navigation and enhance README with "Start Here" guidance.

### Changes Required:

#### 1. Update _sidebar.md
**File**: `docs/_sidebar.md`

```markdown
- **Getting Started**
  - [Overview](README.md)
  - [Architecture](Architecture.md)
  - [Quick Start](Quickstart.md)

- **Core Systems**
  - [HTTP Service](HTTP-Service.md)
  - [Agent System](Agent-System.md)
  - [Memory System](Memory-System.md)
  - [Agent Tools](Agent-Tools.md)
  - [Background Agents](Background-Agents.md)
  - [Pipeline](Pipeline.md)
  - [Honcho](Honcho.md)

- **Configuration**
  - [Configuration](Configuration.md)
  - [Course Schema](config-schema.md)

- **Development**
  - [Development Guide](Development.md)

- **API Reference**
  - [API Endpoints](API.md)
  - [Schemas](Schemas.md)

- **Letta**
  - [Integration Patterns](Letta-Integration.md)
  - [Quick Reference](Letta-Reference.md)

- **Project**
  - [Roadmap](Roadmap.md)
```

#### 2. Enhance README.md
**File**: `docs/README.md`

Add "Start Here" section:

```markdown
# YouLab

> AI-powered course delivery platform with state of the art theory of mind

[... existing content ...]

## Start Here

| I want to... | Read |
|--------------|------|
| Get running quickly | [[Quickstart]] |
| Understand the system | [[Architecture]] |
| Add/modify a course | [[config-schema]] |
| Contribute code | [[Development]] |
| Use the API | [[API]] |
| Understand Letta | [[Letta-Integration]] |

[... rest of existing content ...]
```

### Success Criteria:

#### Automated Verification:
- [ ] Docs site builds
- [ ] All sidebar links work

#### Manual Verification:
- [ ] Navigation is logical and scannable
- [ ] "Start Here" helps different user types
- [ ] No orphaned pages

---

## Testing Strategy

### Automated Tests
- Docs site builds: `npx serve docs`
- Link checker: `npx linkinator http://localhost:3000`

### Manual Testing
1. Navigate through sidebar - all links work
2. Search for common terms - content is findable
3. Read key pages - content is coherent
4. Check cross-references - no broken [[links]]

---

## Verification Summary

After all phases complete:

- [ ] File count: 29 → 18 (11 files deleted)
- [ ] Total size reduction: ~55KB removed
- [ ] No broken links
- [ ] Sidebar updated
- [ ] All deprecated content removed or summarized
- [ ] All current content preserved

### Final File List

```
docs/
├── README.md
├── Architecture.md
├── Quickstart.md
├── HTTP-Service.md
├── Agent-System.md
├── Memory-System.md
├── Agent-Tools.md
├── Background-Agents.md
├── Pipeline.md
├── Honcho.md
├── Configuration.md
├── Development.md
├── API.md
├── Schemas.md
├── Letta-Integration.md
├── Letta-Reference.md
├── Roadmap.md
├── config-schema.md
└── _sidebar.md
```

## References

- Linear ticket context (docs consolidation request)
- Current docs directory: `/docs/`
- Docsify site: `npx serve docs`
