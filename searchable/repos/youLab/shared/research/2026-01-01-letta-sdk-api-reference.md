---
date: 2026-01-01T04:28:17-08:00
researcher: ariasulin
git_commit: 70314e15449c8c3306b1da483a13e094c5d7f4f2
branch: main
repository: YouLab
topic: "Letta SDK & API Complete Reference for YouLab Project"
tags: [research, letta, sdk, api, agents, memory, streaming, reference]
status: complete
last_updated: 2026-01-01
last_updated_by: ariasulin
---

# Research: Letta SDK & API Complete Reference

**Date**: 2026-01-01T04:28:17-08:00
**Researcher**: ariasulin
**Git Commit**: 70314e15449c8c3306b1da483a13e094c5d7f4f2
**Branch**: main
**Repository**: YouLab

## Research Question

Create comprehensive documentation of the Letta SDK, Letta API, and all Letta components used in this codebase. This document serves as the definitive reference for AI agents working with Letta in this project.

## Summary

This document provides complete documentation for Letta as used in the YouLab project. It covers:
- Current project stack and versions
- Letta Python SDK (`letta-client`) reference
- REST API endpoints
- Agent architecture and concepts
- Memory systems (core blocks + archival)
- Streaming and message handling
- Tool system
- Version compatibility

**Key Versions:**
- Project dependency: `letta>=0.6.0`
- Latest Letta Server: v0.16.1 (December 2024)
- Latest Python SDK: `letta-client` v1.6.2 (December 2025)

---

## Table of Contents

1. [Project Stack Overview](#1-project-stack-overview)
2. [Letta Python SDK Reference](#2-letta-python-sdk-reference)
3. [REST API Reference](#3-rest-api-reference)
4. [Agent Architecture](#4-agent-architecture)
5. [Memory System](#5-memory-system)
6. [Streaming & Messages](#6-streaming--messages)
7. [Tool System](#7-tool-system)
8. [Codebase Integration](#8-codebase-integration)
9. [Version Compatibility](#9-version-compatibility)
10. [Quick Reference](#10-quick-reference)

---

## 1. Project Stack Overview

### Current Architecture

```
OpenWebUI → LettaStarter HTTP Service → Letta Server → Claude/OpenAI API
            (FastAPI on port 8100)     (port 8283)
```

### Dependencies (`pyproject.toml`)

```python
dependencies = [
    "letta>=0.6.0",           # Letta server package (includes client)
    "pydantic>=2.0.0",        # Schema validation
    "pydantic-settings>=2.0.0",
    "fastapi>=0.115.0",       # HTTP service
    "uvicorn>=0.32.0",
    "httpx>=0.27.0",          # HTTP client
    "httpx-sse>=0.4.0,<0.5.0", # SSE streaming
    "langfuse>=2.0.0",        # Observability
]
```

### Environment Configuration (`.env.example`)

```bash
# Letta Connection
LETTA_BASE_URL=http://localhost:8283  # Letta server URL

# LLM Provider
LLM_PROVIDER=openai
LLM_MODEL=gpt-4
OPENAI_API_KEY=...

# Memory Settings
CORE_MEMORY_MAX_CHARS=1500
ARCHIVAL_ROTATION_THRESHOLD=0.8
```

---

## 2. Letta Python SDK Reference

### Package Installation

```bash
# Modern SDK (recommended)
pip install letta-client

# Legacy package (also used in this project for server)
pip install letta
```

### Client Initialization

```python
from letta_client import Letta

# Self-hosted (YouLab default)
client = Letta(base_url="http://localhost:8283")

# Letta Cloud
client = Letta(token="LETTA_API_KEY")

# Async client
from letta_client import AsyncLetta
async_client = AsyncLetta(base_url="http://localhost:8283")
```

### Core Services

| Service | Access | Description |
|---------|--------|-------------|
| Agents | `client.agents` | Create, list, retrieve, update, delete agents |
| Messages | `client.agents.messages` | Send messages, stream responses |
| Blocks | `client.agents.blocks` / `client.blocks` | Memory block management |
| Tools | `client.tools` / `client.agents.tools` | Tool management |
| Passages | `client.agents.passages` | Archival memory operations |

### Agent Operations

```python
# Create agent
agent = client.agents.create(
    name="youlab_user123_tutor",
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-ada-002",
    memory_blocks=[
        {"label": "persona", "value": "I am a tutor..."},
        {"label": "human", "value": "User info..."},
    ],
    metadata={"youlab_user_id": "user123", "youlab_agent_type": "tutor"},
)

# List agents
agents = client.agents.list()

# Retrieve agent
agent = client.agents.retrieve(agent_id="agent-xxx")

# Delete agent
client.agents.delete(agent_id="agent-xxx")
```

### Memory Block Operations

```python
# List blocks for agent
blocks = client.agents.blocks.list(agent_id=agent.id)

# Retrieve specific block
block = client.agents.blocks.retrieve(agent_id=agent.id, block_label="human")

# Update block (replaces entire value)
client.agents.blocks.update(
    agent_id=agent.id,
    block_label="human",
    value="Updated user info..."
)

# Create standalone block
block = client.blocks.create(
    label="shared_context",
    value="Shared information...",
    description="Context shared across agents"
)

# Attach/detach blocks
client.agents.blocks.attach(agent_id=agent.id, block_id=block.id)
client.agents.blocks.detach(agent_id=agent.id, block_id=block.id)
```

### Archival Memory (Passages)

```python
# Insert passage
client.agents.passages.insert(
    agent_id=agent.id,
    content="Important fact to remember",
    tags=["category", "important"]
)

# Search passages (semantic + keyword)
results = client.agents.passages.search(
    agent_id=agent.id,
    query="search query",
    tags=["category"],  # Optional filter
    page=0
)

# List all passages
passages = client.agents.passages.list(agent_id=agent.id, limit=100)

# Delete passage
client.agents.passages.delete(agent_id=agent.id, passage_id=passage.id)
```

---

## 3. REST API Reference

### Base URL

| Environment | URL |
|-------------|-----|
| Self-hosted (default) | `http://localhost:8283` |
| Letta Cloud | `https://api.letta.com` |

### Authentication

```http
Authorization: Bearer <token>
```
Self-hosted: No auth required by default.

### Complete Endpoint Reference

#### Agents

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/agents/` | List all agents |
| POST | `/v1/agents/` | Create agent |
| GET | `/v1/agents/{agent_id}` | Retrieve agent |
| PATCH | `/v1/agents/{agent_id}` | Update agent |
| DELETE | `/v1/agents/{agent_id}` | Delete agent |

#### Messages

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/agents/{agent_id}/messages` | List messages |
| POST | `/v1/agents/{agent_id}/messages` | Send message (sync) |
| POST | `/v1/agents/{agent_id}/messages/stream` | Send message (SSE) |
| PATCH | `/v1/agents/{agent_id}/reset-messages` | Reset history |

#### Memory Blocks

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/agents/{agent_id}/core-memory/blocks` | List blocks |
| GET | `/v1/agents/{agent_id}/core-memory/blocks/{label}` | Get block |
| PATCH | `/v1/agents/{agent_id}/core-memory/blocks/{label}` | Update block |
| PATCH | `/v1/agents/{agent_id}/core-memory/blocks/attach/{block_id}` | Attach |
| PATCH | `/v1/agents/{agent_id}/core-memory/blocks/detach/{block_id}` | Detach |

#### Archival Memory

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/agents/{agent_id}/archival` | Insert passage |
| GET | `/v1/agents/{agent_id}/archival` | List passages |
| DELETE | `/v1/agents/{agent_id}/archival/{passage_id}` | Delete passage |
| POST | `/v1/passages/search` | Search passages |

#### Tools

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/tools/` | List tools |
| POST | `/v1/tools/` | Create tool |
| GET | `/v1/tools/{tool_id}` | Get tool |
| PATCH | `/v1/tools/{tool_id}` | Update tool |
| DELETE | `/v1/tools/{tool_id}` | Delete tool |

#### Health

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/health` | Health check |

---

## 4. Agent Architecture

### Core Concepts

Letta agents are **stateful AI services** based on the MemGPT research:

- **Persistent State**: All agent state persists to database at each step
- **Self-Managing Memory**: Agents autonomously decide what to store/retrieve
- **Single Thread**: No "sessions" - one perpetual conversation per agent
- **Tool-Based Actions**: All agent actions (including memory) via tool calls

### Agent Creation Schema

```python
agent = client.agents.create(
    name="agent_name",                    # Unique identifier
    model="provider/model-name",          # e.g., "openai/gpt-4o-mini"
    embedding="provider/model-name",      # e.g., "openai/text-embedding-ada-002"
    memory_blocks=[                       # Initial memory
        {"label": "persona", "value": "...", "limit": 5000},
        {"label": "human", "value": "...", "limit": 5000},
    ],
    tools=["tool_name"],                  # Tool names to attach
    tool_ids=["tool-xxx"],                # Tool IDs to attach
    metadata={...},                       # Custom metadata
    tags=["tag1", "tag2"],                # Organization tags
)
```

### Agent State (AgentState)

```python
{
    "id": "agent-xxx",
    "name": "my-agent",
    "model": "openai/gpt-4o-mini",
    "system": "System prompt...",
    "blocks": [...],      # Memory blocks
    "tools": [...],       # Attached tools
    "tags": [...],
    "metadata": {...},
    "created_at": "2025-01-01T00:00:00Z"
}
```

### LLM Configuration

Models are specified as `provider/model-name`:

| Provider | Model Examples |
|----------|----------------|
| OpenAI | `openai/gpt-4o-mini`, `openai/gpt-4.1` |
| Anthropic | `anthropic/claude-3-5-sonnet`, `anthropic/claude-3-opus` |
| Ollama | `ollama/llama2:7b-q6_K` |

**Embedding Models:**
- `openai/text-embedding-3-small` (recommended)
- `openai/text-embedding-ada-002`
- `ollama/mxbai-embed-large`

---

## 5. Memory System

### Two-Tier Architecture

| Tier | Location | Access | Best For |
|------|----------|--------|----------|
| **Core Memory (Blocks)** | In-context | Always visible | Preferences, persona, current state |
| **Archival Memory** | Vector DB | Semantic search | Documents, history, facts |

### Memory Blocks

Memory blocks are structured sections always visible in the context window.

**Block Structure:**
```python
{
    "label": "human",                    # Unique identifier
    "description": "User information",   # Guides agent behavior
    "value": "User's name is Alice...",  # Actual content
    "limit": 2000,                       # Character limit
    "read_only": False                   # Optional: prevent agent edits
}
```

**Default Blocks:**
- `persona` - Agent identity and behavior
- `human` - Information about the user

**Agent Memory Tools (during conversation):**

| Tool | Purpose |
|------|---------|
| `memory_insert` | Add text at specific line |
| `memory_replace` | Find and replace text |
| `memory_rethink` | Rewrite entire block |
| `memory_finish_edits` | Signal editing complete |

### Archival Memory

Long-term storage with semantic search capabilities.

**Agent Archival Tools:**

| Tool | Purpose |
|------|---------|
| `archival_memory_insert` | Store content with tags |
| `archival_memory_search` | Semantic search |
| `conversation_search` | Search chat history |

**Search Features:**
- Hybrid search (semantic + keyword)
- Reciprocal Rank Fusion (RRF) scoring
- Tag-based filtering
- Date range filtering

### Memory Block Serialization (YouLab Custom)

The codebase uses a custom compact format for memory blocks:

**PersonaBlock Format:**
```
[IDENTITY] YouLab Essay Coach | AI tutor
[CAPABILITIES] Guide self-discovery, brainstorm topics, provide feedback
[EXPERTISE] College admissions, personal narrative, reflective writing
[STYLE] warm, adaptive
[CONSTRAINTS] Never write essays; always ask clarifying questions
```

**HumanBlock Format:**
```
[USER] Alice | Student
[TASK] Draft personal statement intro
[STATE] ACTIVE_TASK
[PREFS] prefers evening sessions; likes detailed feedback
[FACTS] senior at Lincoln High; interested in computer science
[CONTEXT] brainstormed 3 essay topics; chose volunteering experience
```

---

## 6. Streaming & Messages

### Message Sending

**Non-Streaming:**
```python
response = client.agents.messages.create(
    agent_id=agent.id,
    messages=[{"role": "user", "content": "Hello!"}],
    # OR shorthand:
    input="Hello!"
)

for message in response.messages:
    print(message)
```

**Streaming:**
```python
with client.agents.messages.stream(
    agent_id=agent.id,
    input="Hello!",
    enable_thinking="true",   # Include reasoning
    stream_tokens=False,      # Step-level streaming
    include_pings=True,       # Keepalive pings
) as stream:
    for chunk in stream:
        print(chunk.message_type, chunk)
```

### Message Types

| Type | Description | Key Fields |
|------|-------------|------------|
| `user_message` | User input | `content` |
| `system_message` | System context | `content` |
| `reasoning_message` | Chain-of-thought | `reasoning` |
| `assistant_message` | Agent response | `content` |
| `tool_call_message` | Tool request | `tool_call.name`, `tool_call.arguments` |
| `tool_return_message` | Tool result | `tool_return`, `status` |
| `stop_reason` | Stream end | `stop_reason` |
| `usage_statistics` | Token counts | `prompt_tokens`, `completion_tokens` |
| `ping` | Keepalive | - |

### SSE Event Format

```
data: {"id":"msg-123","message_type":"assistant_message","content":"Hello!"}

data: {"message_type":"stop_reason","stop_reason":"end_turn"}

data: {"message_type":"usage_statistics","total_tokens":500}
```

### Response Extraction Pattern

```python
def extract_response(response) -> str:
    """Extract text from Letta response object."""
    texts = []
    for msg in response.messages:
        if hasattr(msg, "assistant_message") and msg.assistant_message:
            texts.append(msg.assistant_message)
        elif hasattr(msg, "text") and msg.text:
            texts.append(msg.text)
    return "\n".join(texts)
```

---

## 7. Tool System

### Built-in Tools

**Memory Tools:**
- `memory_insert`, `memory_replace`, `memory_rethink`, `memory_finish_edits`
- `archival_memory_insert`, `archival_memory_search`
- `conversation_search`, `conversation_search_date`

**Utility Tools:**
- `web_search` - Semantic web search
- `run_code` - Code execution sandbox

### Custom Tools

**Method 1: From Function**
```python
def roll_dice(num_dice: int, num_sides: int) -> str:
    """
    Roll dice and return results.

    Args:
        num_dice: Number of dice to roll
        num_sides: Number of sides per die

    Returns:
        String with roll results
    """
    import random
    results = [random.randint(1, num_sides) for _ in range(num_dice)]
    return f"Rolled: {results}"

tool = client.tools.upsert_from_function(func=roll_dice)
```

**Method 2: With Pydantic Schema**
```python
from pydantic import BaseModel, Field

class DiceArgs(BaseModel):
    num_dice: int = Field(description="Number of dice")
    num_sides: int = Field(description="Sides per die")

tool = client.tools.upsert_from_function(
    func=roll_dice,
    args_schema=DiceArgs
)
```

### Attaching Tools to Agents

```python
# At creation
agent = client.agents.create(
    model="openai/gpt-4o-mini",
    memory_blocks=[...],
    tools=["tool_name"],      # By name
    tool_ids=[tool.id]        # By ID
)

# After creation
client.agents.tools.attach(agent_id=agent.id, tool_id=tool.id)
client.agents.tools.detach(agent_id=agent.id, tool_id=tool.id)
```

---

## 8. Codebase Integration

### File Locations

| Component | Path |
|-----------|------|
| Agent Templates | `src/letta_starter/agents/templates.py` |
| AgentManager | `src/letta_starter/server/agents.py` |
| Memory Blocks | `src/letta_starter/memory/blocks.py` |
| Memory Strategies | `src/letta_starter/memory/strategies.py` |
| Memory Manager | `src/letta_starter/memory/manager.py` |
| HTTP Service | `src/letta_starter/server/main.py` |
| OpenWebUI Pipe | `src/letta_starter/pipelines/letta_pipe.py` |

### Letta Imports Used

```python
# New SDK (HTTP Service)
from letta_client import Letta

# Legacy SDK (BaseAgent)
from letta import create_client as letta_create_client
```

### AgentManager API (server/agents.py)

```python
class AgentManager:
    def __init__(self, letta_base_url: str):
        self._client: Letta | None = None  # Lazy-initialized

    @property
    def client(self) -> Letta:
        if self._client is None:
            self._client = Letta(base_url=self.letta_base_url)
        return self._client

    # Agent Operations
    def create_agent(user_id: str, agent_type: str = "tutor") -> str
    def get_agent_id(user_id: str, agent_type: str) -> str | None
    def get_agent_info(agent_id: str) -> dict
    def list_user_agents(user_id: str) -> list[dict]

    # Messaging
    def send_message(agent_id: str, message: str) -> str
    def stream_message(agent_id: str, message: str) -> Iterator[str]

    # Health
    def check_letta_connection() -> bool
```

### Agent Naming Convention

```python
def _agent_name(user_id: str, agent_type: str) -> str:
    return f"youlab_{user_id}_{agent_type}"
    # Example: "youlab_user123_tutor"

def _agent_metadata(user_id: str, agent_type: str) -> dict:
    return {
        "youlab_user_id": user_id,
        "youlab_agent_type": agent_type,
    }
```

### Agent Creation (Current Implementation)

```python
agent = self.client.agents.create(
    name=agent_name,                              # "youlab_user123_tutor"
    model="openai/gpt-4o-mini",                   # Hardcoded
    embedding="openai/text-embedding-ada-002",    # Hardcoded
    memory_blocks=[
        {"label": "persona", "value": template.persona.to_memory_string()},
        {"label": "human", "value": human_block.to_memory_string()},
    ],
    metadata=metadata,
)
```

### Streaming Implementation

```python
def stream_message(self, agent_id: str, message: str) -> Iterator[str]:
    with self.client.agents.messages.stream(
        agent_id=agent_id,
        input=message,
        enable_thinking="true",
        stream_tokens=False,
        include_pings=True,
    ) as stream:
        for chunk in stream:
            event = self._chunk_to_sse_event(chunk)
            if event:
                yield event
```

### SSE Event Conversion

```python
def _chunk_to_sse_event(self, chunk) -> str | None:
    msg_type = getattr(chunk, "message_type", None)

    if msg_type == "reasoning_message":
        return json.dumps({"type": "status", "reasoning": chunk.reasoning})
    elif msg_type == "tool_call_message":
        return json.dumps({"type": "status", "content": f"Using {chunk.tool_call.name}..."})
    elif msg_type == "assistant_message":
        content = self._strip_letta_metadata(chunk.content)
        return json.dumps({"type": "message", "content": content})
    elif msg_type == "stop_reason":
        return json.dumps({"type": "done"})
    elif msg_type == "ping":
        return ": keepalive\n\n"
    # ... error handling
```

---

## 9. Version Compatibility

### Letta Server Versions

| Version | Date | Notable Changes |
|---------|------|-----------------|
| v0.16.1 | Dec 2024 | Current stable, OpenAI-proxy fix |
| v0.16.0 | Dec 2024 | Architecture-specific OpenTelemetry |
| v0.12.0 | Oct 2024 | New `letta_v1_agent` architecture |
| v0.6.4 | 2024 | Python 3.13 support, TypeScript SDK |
| v0.5.0 | 2024 | Dynamic model listing, Alembic migrations |

### Python SDK Versions (`letta-client`)

| Version | Date | Notable Changes |
|---------|------|-----------------|
| 1.6.2 | Dec 2024 | Latest stable |
| 1.6.0 | Dec 2024 | Compaction response feature |
| 1.5.0 | Dec 2024 | Message ID in search |
| 1.3.0 | Nov 2024 | Messages and passages in config |

### Breaking Changes

**v0.12 (Major):**
- New agent architecture (no `send_message` tool required)
- Heartbeat system removed
- `sources` routes renamed to `folders`

**v0.5:**
- `letta configure` command deprecated
- Environment variable configuration required

### Current Project Compatibility

```python
# pyproject.toml
"letta>=0.6.0"  # Minimum version requirement
```

The project uses both:
- `letta` package (for server and legacy client)
- `letta_client.Letta` class (for modern SDK calls)

---

## 10. Quick Reference

### Common Operations

```python
from letta_client import Letta

client = Letta(base_url="http://localhost:8283")

# Create agent
agent = client.agents.create(
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-ada-002",
    memory_blocks=[
        {"label": "persona", "value": "I am a helpful assistant."},
        {"label": "human", "value": "User info here."},
    ]
)

# Send message
response = client.agents.messages.create(
    agent_id=agent.id,
    input="Hello!"
)

# Stream message
with client.agents.messages.stream(
    agent_id=agent.id,
    input="Hello!",
) as stream:
    for chunk in stream:
        if chunk.message_type == "assistant_message":
            print(chunk.content)

# Update memory
client.agents.blocks.update(
    agent_id=agent.id,
    block_label="human",
    value="Updated info..."
)

# Insert to archival
client.agents.passages.insert(
    agent_id=agent.id,
    content="Important fact",
    tags=["category"]
)

# Search archival
results = client.agents.passages.search(
    agent_id=agent.id,
    query="search query"
)

# List agents
agents = client.agents.list()

# Health check
try:
    client.agents.list()
    print("Connected")
except:
    print("Not connected")
```

### Error Handling

```python
from letta_client.core.api_error import ApiError

try:
    response = client.agents.messages.create(...)
except ApiError as e:
    print(f"Status: {e.status_code}")
    print(f"Message: {e.message}")
    print(f"Body: {e.body}")
```

### Environment Variables

```bash
# Required for provider
OPENAI_API_KEY=sk-...
# OR
ANTHROPIC_API_KEY=sk-ant-...

# Letta connection (self-hosted)
LETTA_BASE_URL=http://localhost:8283

# Optional
LETTA_API_KEY=...  # For Letta Cloud or auth-enabled server
```

---

## Code References

- `src/letta_starter/server/agents.py:28` - Letta client initialization
- `src/letta_starter/server/agents.py:108-117` - Agent creation with memory blocks
- `src/letta_starter/server/agents.py:186-192` - Streaming message API call
- `src/letta_starter/server/agents.py:201-249` - SSE event conversion
- `src/letta_starter/memory/blocks.py:28-132` - PersonaBlock schema
- `src/letta_starter/memory/blocks.py:134-274` - HumanBlock schema
- `src/letta_starter/agents/templates.py:19-47` - TUTOR_TEMPLATE definition

---

## Historical Context (from thoughts/)

- `thoughts/global/shared/reference/letta-archival-memory.md` - Comprehensive archival memory guide
- `thoughts/shared/research/2025-12-31-phase-2-streaming-research.md` - Streaming implementation research
- `thoughts/shared/research/2025-12-31-phase-2-consolidated-reference.md` - Phase 2 reference
- `thoughts/shared/handoffs/general/2025-12-31_16-53-13_letta-v0.16-compatibility.md` - v0.16 compatibility notes

---

## Related Research

- `thoughts/shared/research/2026-01-01-youlab-codebase-documentation.md` - Codebase overview
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Technical roadmap

---

## Open Questions

1. **Agent Type Migration**: Should legacy `agent_type` specification be removed in favor of the new architecture (no agent_type)?

2. **Memory Block Limits**: Current hardcoded limit is 1500 chars (`to_memory_string()`). Is this optimal for the tutoring use case?

3. **Embedding Model**: Currently using `text-embedding-ada-002`. Consider upgrading to `text-embedding-3-small` for better performance.

4. **SDK Version Alignment**: The `letta>=0.6.0` dependency may pull older server. Consider pinning to match SDK version.

---

## External Links

### Official Documentation
- [Letta Docs Home](https://docs.letta.com)
- [Python SDK Reference](https://docs.letta.com/api/python/)
- [API Reference](https://docs.letta.com/api/)
- [Developer Quickstart](https://docs.letta.com/quickstart/)

### Guides
- [Building Stateful Agents](https://docs.letta.com/guides/agents/overview/)
- [Memory Overview](https://docs.letta.com/guides/agents/memory/)
- [Memory Blocks Guide](https://docs.letta.com/guides/agents/memory-blocks/)
- [Archival Memory](https://docs.letta.com/guides/agents/archival-memory/)
- [Streaming Guide](https://docs.letta.com/guides/agents/streaming/)
- [Custom Tools](https://docs.letta.com/guides/agents/custom-tools)
- [Self-hosting](https://docs.letta.com/guides/selfhosting/)

### GitHub
- [letta-ai/letta](https://github.com/letta-ai/letta) - Main repository
- [letta-ai/letta-python](https://github.com/letta-ai/letta-python) - Python SDK
- [Releases](https://github.com/letta-ai/letta/releases) - Version history

### Provider Configuration
- [OpenAI Provider](https://docs.letta.com/guides/server/providers/openai)
- [Anthropic Provider](https://docs.letta.com/guides/server/providers/anthropic/)
- [Ollama Provider](https://docs.letta.com/guides/server/providers/ollama/)

---

*This document was compiled from official Letta documentation, the YouLab codebase, and thoughts/ directory research. Last updated: 2026-01-01.*
