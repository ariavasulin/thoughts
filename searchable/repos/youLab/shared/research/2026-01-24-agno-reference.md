---
date: 2026-01-24T10:30:00-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Agno Agent Framework Reference for YouLab v2"
tags: [research, agno, framework, migration, tools, streaming, memory, letta-comparison]
status: complete
last_updated: 2026-01-24
last_updated_by: ariasulin
related: [2026-01-16-agno-vs-frameworkless-evaluation.md, 2026-01-16-letta-to-agno-migration.md]
---

# Research: Agno Agent Framework Reference

**Date**: 2026-01-24T10:30:00-08:00
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Comprehensive reference documentation for the Agno agent framework (formerly Phidata) for YouLab v2 migration. Covers agent creation, RunContext/DI, @tool decorator, streaming, session state, memory patterns, and error handling. Includes comparison with current Letta patterns.

## Summary

Agno is a high-performance Python framework for multi-agent systems with 529x faster instantiation than LangGraph. Key advantages for YouLab:

| Feature | Agno | Letta (Current) |
|---------|------|-----------------|
| **Tool Definition** | `@tool` decorator on plain functions | JSON schema + global registration |
| **Context Injection** | `RunContext` parameter auto-injected | `agent_state` dict from metadata |
| **Streaming** | `arun(stream=True)` returns AsyncIterator | `client.agents.messages.stream()` context manager |
| **Session State** | Built-in `session_state` dict | Manual metadata + global context |
| **Memory** | Agentic memory with DB persistence | Git blocks + Letta sync |
| **Error Handling** | `RetryAgentRun`/`StopAgentRun` exceptions | No built-in retry; manual try/except |

---

## 1. Agent Creation

### Basic Agent

```python
from agno.agent import Agent
from agno.models.anthropic import Claude

agent = Agent(
    model=Claude(id="claude-sonnet-4-5"),
    instructions="Use tables to display data.",
    markdown=True,
)

agent.print_response("Write a report", stream=True)
```

### Full Configuration

```python
from agno.agent import Agent
from agno.models.anthropic import Claude
from agno.db.sqlite import SqliteDb

agent = Agent(
    # Identity
    name="Tutor Agent",
    model=Claude(id="claude-sonnet-4-5"),

    # Instructions & Context
    instructions="You are a patient tutor. User name: {user_name}",
    description="A helpful writing coach",
    expected_output="Structured feedback with examples",

    # Tools
    tools=[edit_memory_block, query_honcho, advance_lesson],
    tool_call_limit=10,

    # Session & State
    db=SqliteDb(db_file="agno.db"),
    session_state={"user_name": "John", "course_id": "college-essay"},
    add_session_state_to_context=True,
    enable_agentic_state=True,

    # Memory
    enable_agentic_memory=True,
    enable_session_summaries=True,

    # History & Context
    add_history_to_context=True,
    add_datetime_to_context=True,

    # Reasoning (Claude extended thinking)
    reasoning=True,
    reasoning_model=Claude(id="claude-sonnet-4-5"),
    reasoning_max_steps=10,

    # Output
    markdown=True,
    structured_outputs=True,

    # Retry Logic
    retries=3,
    delay_between_retries=1.0,
    exponential_backoff=True,

    # Streaming
    stream=True,
    stream_events=True,

    # Debug
    debug_mode=True,
    debug_level=2,
)
```

### Key Methods

| Method | Description |
|--------|-------------|
| `run()` / `arun()` | Sync/async execution |
| `print_response()` / `aprint_response()` | Display formatted results |
| `continue_run()` / `acontinue_run()` | Resume interrupted operations |
| `get_session_state()` | Retrieve session state |
| `update_session_state()` | Modify session state |
| `get_user_memories()` | Retrieve stored memories |
| `cancel_run()` | Halt active execution |

### Comparison: Letta Agent Creation

**Letta (Current)**:
```python
# src/youlab_server/server/agents.py:324-338
agent = self.client.agents.create(
    name=_agent_name(user_id, agent_type),
    model=course.agent.model,
    embedding=course.agent.embedding,
    system=course.agent.system,
    memory_blocks=[{"label": "human", "value": "..."}],
    block_ids=shared_block_ids,
    tools=["edit_memory_block", "query_honcho"],
    tool_rules=[{"type": "exit", "tool_name": "send_message"}],
    metadata={"youlab_user_id": user_id, "course_id": course_id},
)
```

**Agno Equivalent**:
```python
agent = Agent(
    name=f"tutor_{user_id}_{course_id}",
    model=Claude(id="claude-sonnet-4-5"),
    instructions=course.agent.system + format_blocks(user_blocks),
    tools=[edit_memory_block, query_honcho, advance_lesson],
    session_state={
        "user_id": user_id,
        "course_id": course_id,
        "memory_blocks": load_user_blocks(user_id),
    },
    add_session_state_to_context=True,
)
```

---

## 2. RunContext / Dependency Injection

### Overview

`RunContext` is Agno's dependency injection mechanism. When a tool function declares a `run_context: RunContext` parameter, Agno automatically injects it with:

| Property | Description |
|----------|-------------|
| `run_id` | Unique ID for this execution |
| `session_id` | Session identifier |
| `user_id` | User identifier |
| `session_state` | Persistent state dictionary |
| `dependencies` | Injected dependencies |
| `metadata` | Runtime metadata |
| `knowledge_filters` | RAG filters |
| `tool_call_id` | Current tool call ID (as of Jan 2026) |

### Basic Usage

```python
from agno.run import RunContext
from agno.tools import tool

@tool
def get_shopping_list(run_context: RunContext) -> str:
    """Get the shopping list from session state."""
    return run_context.session_state.get("shopping_list", [])

@tool
def greet_user(run_context: RunContext, greeting: str) -> str:
    """Greet the user by name."""
    user_name = run_context.dependencies.get("user_name", "Guest")
    return f"{greeting}, {user_name}!"

agent = Agent(
    tools=[get_shopping_list, greet_user],
    session_state={"shopping_list": ["milk", "bread", "eggs"]},
)

# Pass dependencies at runtime
agent.print_response(
    "What's on my shopping list?",
    dependencies={"user_name": "John"},
    stream=True,
)
```

### Comparison: Letta agent_state Injection

**Letta (Current)**:
```python
# src/youlab_server/tools/sandbox.py:63-67
def query_honcho(
    question: str,
    session_scope: str = "all",
    agent_state: dict[str, Any] | None = None,  # Injected by Letta
) -> str:
    user_id = agent_state.get("user_id")
    if not user_id:
        metadata = agent_state.get("metadata", {})
        user_id = metadata.get("youlab_user_id")
```

**Agno Equivalent**:
```python
@tool
def query_honcho(
    run_context: RunContext,  # Injected by Agno
    question: str,
    session_scope: str = "all",
) -> str:
    user_id = run_context.session_state.get("user_id")
    # or
    user_id = run_context.user_id
```

### Key Differences

| Aspect | Letta | Agno |
|--------|-------|------|
| **Parameter Name** | `agent_state: dict` | `run_context: RunContext` |
| **User ID Access** | `agent_state["metadata"]["youlab_user_id"]` | `run_context.user_id` or `run_context.session_state["user_id"]` |
| **State Persistence** | Manual (global `_user_context` dict) | Built-in (session_state + DB) |
| **Dependencies** | Global modules (`_honcho_client`) | `run_context.dependencies` |

---

## 3. @tool Decorator

### Basic Definition

```python
from agno.tools import tool

@tool
def get_weather(city: str) -> str:
    """Get weather for a city.

    Args:
        city: The city name to get weather for.

    Returns:
        Weather description string.
    """
    return f"Weather in {city}: Sunny, 72F"
```

Agno automatically generates JSON schema from the function signature and docstring.

### Full Decorator Options

```python
from agno.tools import tool
from agno.run import RunContext

@tool(
    # Naming
    name="edit_memory_block",              # Override function name
    description="Propose changes to memory", # Override docstring

    # Execution Control
    stop_after_tool_call=True,             # End agent run after this tool
    requires_confirmation=True,             # User must confirm
    requires_user_input=True,               # Prompt for input
    user_input_fields=["approval"],         # Fields requiring input
    external_execution=True,                # Execute outside agent

    # Result Handling
    show_result=True,                       # Show output in response

    # Caching
    cache_results=True,                     # Cache results
    cache_dir="/tmp/agno_cache",            # Cache directory
    cache_ttl=3600,                         # TTL in seconds

    # Hooks
    pre_hook=my_pre_hook,                   # Before tool call
    post_hook=my_post_hook,                 # After tool call
    tool_hooks=[logger_hook],               # General hooks
)
def edit_memory_block(
    run_context: RunContext,
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
) -> str:
    """Propose changes to a memory block field."""
    user_id = run_context.session_state.get("user_id")
    # ... implementation
    return f"Proposed change to {block}.{field}"
```

### Automatically Injected Parameters

These parameters are automatically provided by Agno (don't include in tool schema):

| Parameter | Type | Description |
|-----------|------|-------------|
| `run_context` | `RunContext` | Session state, dependencies, metadata |
| `agent` | `Agent` | The Agent instance |
| `team` | `Team` | Team object (for team agents) |
| `images` | `list` | Attached images |
| `videos` | `list` | Attached videos |
| `audio` | `list` | Attached audio |
| `files` | `list` | Attached files |

### Async Tools

```python
import asyncio
from agno.tools import tool

@tool
async def fetch_data(run_context: RunContext, url: str) -> str:
    """Fetch data from URL asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
```

### Tool Hooks

```python
from agno.run import RunContext
from typing import Any, Callable, Dict

def logger_hook(
    run_context: RunContext,
    function_name: str,
    function_call: Callable,
    arguments: Dict[str, Any],
):
    """Log tool calls before/after execution."""
    print(f"Calling {function_name} with {arguments}")
    result = function_call(**arguments)
    print(f"Result: {result}")
    return result

agent = Agent(
    tools=[edit_memory_block],
    tool_hooks=[logger_hook],
)
```

### Comparison: Letta Tool Registration

**Letta (Current)**:
```python
# src/youlab_server/server/main.py:104-108
# Global registration at startup
client.tools.upsert_from_function(sandbox_query_honcho)
client.tools.upsert_from_function(sandbox_edit_memory_block)
client.tools.upsert_from_function(advance_lesson)
```

**Agno Equivalent**:
```python
# Tools are just functions - no global registration needed
agent = Agent(
    tools=[query_honcho, edit_memory_block, advance_lesson],
)
```

---

## 4. Streaming (arun_response_stream)

### Synchronous Streaming

```python
from agno.agent import Agent, RunOutputEvent, RunEvent

agent = Agent(model=Claude(id="claude-sonnet-4-5"))

# Basic streaming
for chunk in agent.run("Tell me a story", stream=True):
    if chunk.event == RunEvent.run_content:
        print(chunk.content, end="")
```

### Async Streaming

```python
import asyncio

async def stream_chat():
    async for event in agent.arun("Tell me a story", stream=True):
        if event.event == RunEvent.run_content:
            print(event.content, end="")

asyncio.run(stream_chat())
```

### Stream All Events

```python
# Get all events including tool calls, reasoning, etc.
async for event in agent.arun(
    "Analyze this",
    stream=True,
    stream_events=True,  # Exposes internal steps
):
    if event.event == RunEvent.run_content:
        print(f"Content: {event.content}")
    elif event.event == RunEvent.tool_call_started:
        print(f"Tool started: {event.tool.tool_name}")
    elif event.event == RunEvent.tool_call_completed:
        print(f"Tool completed: {event.tool.tool_result}")
    elif event.event == RunEvent.reasoning_step:
        print(f"Reasoning: {event.reasoning_content}")
```

### Event Types

| Category | Events |
|----------|--------|
| **Core** | `RunStarted`, `RunContent`, `RunCompleted`, `RunError`, `RunCancelled` |
| **Tool** | `ToolCallStarted`, `ToolCallCompleted` |
| **Reasoning** | `ReasoningStarted`, `ReasoningStep`, `ReasoningCompleted` |
| **Memory** | `MemoryUpdateStarted`, `SessionSummaryStarted` |
| **Hooks** | `PreHookStarted`, `PostHookStarted` |

### SSE Conversion for OpenWebUI

```python
import json
from typing import AsyncIterator
from agno.agent import Agent, RunEvent

async def agno_to_sse(agent: Agent, message: str) -> AsyncIterator[str]:
    """Convert Agno streaming to SSE format for OpenWebUI."""
    async for chunk in agent.arun(message, stream=True, stream_events=True):
        if chunk.event == RunEvent.tool_call_started:
            yield f"data: {json.dumps({'type': 'status', 'content': f'Using {chunk.tool.tool_name}...'})}\n\n"

        elif chunk.event == RunEvent.reasoning_step:
            yield f"data: {json.dumps({'type': 'status', 'content': 'Thinking...', 'reasoning': chunk.reasoning_content})}\n\n"

        elif chunk.event == RunEvent.run_content:
            yield f"data: {json.dumps({'type': 'message', 'content': chunk.content})}\n\n"

        elif chunk.event == RunEvent.run_completed:
            yield f"data: {json.dumps({'type': 'done'})}\n\n"

        elif chunk.event == RunEvent.run_error:
            yield f"data: {json.dumps({'type': 'error', 'message': str(chunk.error)})}\n\n"
```

### Comparison: Letta Streaming

**Letta (Current)**:
```python
# src/youlab_server/server/agents.py:426-439
with self.client.agents.messages.stream(
    agent_id=agent_id,
    input=message,
    stream_tokens=False,
    include_pings=True,
) as stream:
    for chunk in stream:
        event = self._chunk_to_sse_event(chunk)
        yield event
```

**Agno Equivalent**:
```python
async for chunk in agent.arun(message, stream=True, stream_events=True):
    event = agno_chunk_to_sse(chunk)
    yield event
```

---

## 5. Session State

### Basic Usage

```python
from agno.agent import Agent
from agno.db.sqlite import SqliteDb

agent = Agent(
    model=Claude(id="claude-sonnet-4-5"),
    db=SqliteDb(db_file="sessions.db"),  # Persistence
    session_state={"cart": [], "user_name": ""},  # Defaults
    add_session_state_to_context=True,  # Include in prompt
    enable_agentic_state=True,  # Agent can auto-update
)

# Run with session
agent.print_response(
    "My name is John",
    session_id="session_123",
    user_id="user_456",
    session_state={"user_name": "John"},  # Override defaults
)

# Access state programmatically
state = agent.get_session_state(session_id="session_123")

# Update state
agent.update_session_state(
    session_id="session_123",
    session_state_updates={"cart": ["item1", "item2"]},
)
```

### Template Variables

State values can be used in instructions:

```python
agent = Agent(
    instructions="User's name is {user_name} and they are studying {course_id}.",
    session_state={"user_name": "John", "course_id": "college-essay"},
    add_session_state_to_context=True,
)
```

### Persistence Options

```python
from agno.db.postgres import PostgresDb
from agno.db.sqlite import SqliteDb
from agno.db.dynamodb import DynamoDb

# PostgreSQL
db = PostgresDb(
    db_url="postgresql+psycopg://user:pass@localhost:5432/db",
    session_table="agno_sessions",
)

# SQLite
db = SqliteDb(db_file="agno.db")

# DynamoDB
db = DynamoDb(table_name="agno_sessions", region="us-west-2")
```

### Comparison: Letta State Management

**Letta (Current)**:
```python
# Global context mapping (src/youlab_server/tools/dialectic.py:25-36)
_user_context: dict[str, str] = {}

def set_user_context(agent_id: str, user_id: str) -> None:
    _user_context[agent_id] = user_id

# Called before each chat (src/youlab_server/server/main.py:302)
set_user_context(agent_id, request.user_id)

# Access in tool
user_id = _user_context.get(agent_id)
```

**Agno Equivalent**:
```python
# No global state needed - session_state is automatically available
agent = Agent(
    session_state={"user_id": user_id, "course_id": course_id},
)

@tool
def my_tool(run_context: RunContext) -> str:
    user_id = run_context.session_state["user_id"]
```

---

## 6. Memory Patterns

### Three Memory Layers

Agno provides three memory layers:

| Layer | Purpose | Scope |
|-------|---------|-------|
| **User Memory** | Personalization across sessions | User-specific |
| **Session Storage** | Short-term conversation context | Per-session |
| **Culture** | Shared organizational knowledge | All agents |

### User Memory (Agentic)

```python
from agno.agent import Agent
from agno.memory import Memory
from agno.db.sqlite import SqliteDb

agent = Agent(
    model=Claude(id="claude-sonnet-4-5"),
    memory=Memory(db=SqliteDb(db_file="memory.db")),
    enable_agentic_memory=True,  # Agent creates/updates/deletes memories
)

# Agent automatically stores: "John prefers bullet points"
agent.print_response("I prefer bullet point summaries")

# Later session - agent remembers
agent.print_response("Summarize this article", user_id="john")

# Retrieve memories programmatically
memories = agent.get_user_memories(user_id="john")
```

### Session Summaries

```python
agent = Agent(
    db=SqliteDb(db_file="sessions.db"),
    add_history_to_context=True,  # Include chat history
    enable_session_summaries=True,  # Summarize for context
)
```

### Culture (Experimental)

```python
agent = Agent(
    db=SqliteDb(db_file="culture.db"),
    add_culture_to_context=True,      # Read shared knowledge
    update_cultural_knowledge=True,    # Update culture after runs
)
```

### YouLab Memory Migration

**Current Git Blocks Pattern**:
```python
# Load blocks into system prompt
blocks = storage.get_blocks(user_id)
system_prompt = course.agent.system + format_blocks(blocks)
```

**Agno Equivalent Options**:

**Option A: Session State**
```python
agent = Agent(
    session_state={"memory_blocks": load_user_blocks(user_id)},
    instructions="Use memory_blocks for student context: {memory_blocks}",
    add_session_state_to_context=True,
)
```

**Option B: System Prompt Injection**
```python
def build_system_prompt(course, user_blocks):
    block_context = "\n".join([
        f"## {label}\n{content}"
        for label, content in user_blocks.items()
    ])
    return f"{course.agent.system}\n\n# Memory Blocks\n{block_context}"

agent = Agent(
    system_prompt=build_system_prompt(course, user_blocks),
)
```

**Option C: Agentic Memory with Hooks**
```python
# Use Agno's memory system, sync to Git on changes
def git_sync_hook(run_context, function_name, function_call, arguments):
    result = function_call(**arguments)
    if function_name == "update_memory":
        sync_to_git(run_context.user_id, result)
    return result

agent = Agent(
    memory=Memory(db=SqliteDb(db_file="memory.db")),
    enable_agentic_memory=True,
    tool_hooks=[git_sync_hook],
)
```

---

## 7. Error Handling

### Exception Types

| Exception | Behavior |
|-----------|----------|
| `RetryAgentRun` | Feedback to model; model can retry |
| `StopAgentRun` | Ends run gracefully; status = COMPLETED |

### RetryAgentRun Example

```python
from agno.tools import tool
from agno.tools.exceptions import RetryAgentRun

@tool
def add_items(run_context: RunContext, items: list[str]) -> str:
    """Add items to list. Requires at least 3 items."""
    if len(items) < 3:
        raise RetryAgentRun(
            f"Please add {3 - len(items)} more items."
        )
    run_context.session_state["items"] = items
    return f"Added {len(items)} items."
```

### StopAgentRun Example

```python
from agno.tools.exceptions import StopAgentRun

@tool
def check_limit(value: int) -> str:
    """Stop if value exceeds limit."""
    if value > 100:
        raise StopAgentRun("Value exceeded limit. Stopping.")
    return f"Value {value} OK"
```

### Retry Configuration

```python
agent = Agent(
    model=Claude(id="claude-sonnet-4-5"),
    retries=3,                    # Number of attempts
    delay_between_retries=1.0,    # Delay in seconds
    exponential_backoff=True,     # Exponential delay
)
```

### Top-Level Error Handling

```python
from agno.agent import Agent

agent = Agent(model=Claude(id="claude-sonnet-4-5"), tools=[my_tool])

try:
    response = agent.run("Do something risky")
except Exception as e:
    print(f"Agent failed: {e}")
```

### Comparison: Letta Error Handling

**Letta (Current)**:
```python
# src/youlab_server/server/agents.py:437-439
try:
    # ... streaming logic
except Exception as e:
    log.exception("Error streaming")
    yield f'data: {json.dumps({"type": "error", "message": str(e)})}\n\n'

# src/youlab_server/tools/memory.py:106-114
try:
    diff = manager.propose_edit(...)
except Exception as e:
    log.error(f"Failed to create proposal: {e}")
    return f"Failed to create proposal: {e}"
```

**Agno Equivalent**:
```python
@tool
def edit_memory_block(run_context: RunContext, ...) -> str:
    try:
        diff = manager.propose_edit(...)
    except ValidationError as e:
        raise RetryAgentRun(f"Invalid input: {e}")
    except StorageError as e:
        raise StopAgentRun(f"Storage unavailable: {e}")
    return f"Created proposal {diff.id}"
```

---

## 8. Complete Migration Example

### Current Letta Pattern

```python
# src/youlab_server/server/main.py (simplified)
@app.post("/chat/stream")
async def chat_stream(request: StreamChatRequest):
    manager = get_agent_manager()

    # Set global context
    set_user_context(request.agent_id, request.user_id)

    # Persist user message
    create_persist_task(user_id, chat_id, message, is_user=True)

    async def stream():
        full_response = []
        for chunk in manager.stream_message(agent_id, message):
            # Parse SSE, accumulate response
            full_response.append(chunk.content)
            yield chunk

        # Persist response
        create_persist_task(user_id, chat_id, "".join(full_response), is_user=False)

    return StreamingResponse(stream(), media_type="text/event-stream")
```

### Agno Equivalent

```python
from agno.agent import Agent, RunEvent
from agno.models.anthropic import Claude
from agno.db.sqlite import SqliteDb
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

# Agent per session (or cache)
def get_agent(user_id: str, course_id: str) -> Agent:
    course = curriculum.get(course_id)
    user_blocks = load_user_blocks(user_id, course_id)

    return Agent(
        name=f"tutor_{user_id}",
        model=Claude(id=course.agent.model),
        instructions=build_system_prompt(course, user_blocks),
        tools=[edit_memory_block, query_honcho, advance_lesson],
        db=SqliteDb(db_file="sessions.db"),
        session_state={
            "user_id": user_id,
            "course_id": course_id,
            "honcho_client": get_honcho_client(),
        },
        add_session_state_to_context=True,
        add_history_to_context=True,
    )

@app.post("/chat/stream")
async def chat_stream(request: StreamChatRequest):
    agent = get_agent(request.user_id, request.course_id)

    # Persist user message (fire-and-forget)
    create_persist_task(request.user_id, request.chat_id, request.message, is_user=True)

    async def stream():
        full_response = []
        async for event in agent.arun(
            request.message,
            session_id=request.chat_id,
            user_id=request.user_id,
            stream=True,
            stream_events=True,
        ):
            if event.event == RunEvent.run_content:
                full_response.append(event.content)
                yield f"data: {json.dumps({'type': 'message', 'content': event.content})}\n\n"
            elif event.event == RunEvent.tool_call_started:
                yield f"data: {json.dumps({'type': 'status', 'content': f'Using {event.tool.tool_name}...'})}\n\n"
            elif event.event == RunEvent.run_completed:
                yield f"data: {json.dumps({'type': 'done'})}\n\n"
            elif event.event == RunEvent.run_error:
                yield f"data: {json.dumps({'type': 'error', 'message': str(event.error)})}\n\n"

        # Persist response
        create_persist_task(request.user_id, request.chat_id, "".join(full_response), is_user=False)

    return StreamingResponse(stream(), media_type="text/event-stream")
```

---

## Code References (Current Letta Implementation)

### Agent Management
- `src/youlab_server/server/agents.py:19` - `AgentManager` class
- `src/youlab_server/server/agents.py:324-338` - Agent creation
- `src/youlab_server/server/agents.py:410-439` - Streaming

### Tool Registration
- `src/youlab_server/server/main.py:92-115` - Global tool registration
- `src/youlab_server/tools/sandbox.py` - Sandbox-compatible tools
- `src/youlab_server/tools/memory.py` - In-process tools

### Streaming
- `src/youlab_server/server/main.py:371-453` - HTTP endpoint
- `src/youlab_server/server/agents.py:441-489` - SSE conversion
- `src/youlab_server/pipelines/letta_pipe.py:138-231` - OpenWebUI Pipe

### State Management
- `src/youlab_server/tools/dialectic.py:25-36` - Global user context
- `src/youlab_server/storage/blocks.py:161-203` - Block sync to Letta

### Error Handling
- `src/youlab_server/server/agents.py:437-439` - Stream error handling
- `src/youlab_server/tools/memory.py:106-114` - Tool error handling

---

## Related Research

- [2026-01-16-agno-vs-frameworkless-evaluation.md](./2026-01-16-agno-vs-frameworkless-evaluation.md) - Decision to go frameworkless
- [2026-01-16-letta-to-agno-migration.md](./2026-01-16-letta-to-agno-migration.md) - Detailed migration plan

---

## Sources

### Agno Documentation
- [Agno Official Documentation](https://docs.agno.com/)
- [Agno GitHub Repository](https://github.com/agno-agi/agno) (37,000+ stars)
- [Agno Cookbook](https://github.com/agno-agi/agno/tree/main/cookbook)

### Specific Topics
- [Agent Creation](https://docs.agno.com/basics/agents/overview)
- [Tool Decorator Reference](https://docs.agno.com/reference/tools/decorator)
- [Python Functions as Tools](https://docs.agno.com/basics/tools/creating-tools/python-functions)
- [Tool Hooks](https://docs.agno.com/basics/tools/hooks)
- [Running Agents](https://docs.agno.com/basics/agents/running-agents)
- [Agent Context](https://docs.agno.com/examples/getting-started/08-agent-context)
- [Session State](https://docs.agno.com/basics/state/agent/overview)
- [Agentic Memory](https://docs.agno.com/basics/memory/agent/usage/agentic-memory)
- [Culture](https://docs.agno.com/basics/culture/overview)
- [Exceptions & Retries](https://docs.agno.com/basics/tools/exceptions)

### External Resources
- [WorkOS Blog: Agno for Python Teams](https://workos.com/blog/agno-the-agent-framework-for-python-teams)
- [DigitalOcean: Understanding Agno](https://www.digitalocean.com/community/conceptual-articles/agno-fast-scalable-multi-agent-framework)
- [Groq + Agno Integration](https://console.groq.com/docs/agno)

---

## Open Questions

1. **Agent Caching**: Should we recreate agents per request (stateless) or cache them?
   - Agno agents are lightweight (~3us instantiation)
   - Session state persists independently in DB

2. **Sandbox Compatibility**: Current sandbox tools use HTTP endpoints
   - Agno can run tools in-process (simpler)
   - Or keep HTTP pattern for consistency

3. **Block Sync Strategy**: How to sync Git blocks with Agno memory?
   - Option A: Inject via session_state (current plan)
   - Option B: Use Agno's agentic memory + Git sync hooks
   - Option C: Hybrid (session_state for read, agentic for write)

4. **Background Agents**: Migrate to Agno or keep as direct Claude API calls?
   - Agno provides same tool execution patterns
   - But background agents may not need full framework

5. **Known Agno Issues**:
   - Session state in prompts may be stale during multi-step tool calls ([#3916](https://github.com/agno-agi/agno/issues/3916))
   - Tool hooks can bypass agent parameter injection ([#3856](https://github.com/agno-agi/agno/issues/3856))
