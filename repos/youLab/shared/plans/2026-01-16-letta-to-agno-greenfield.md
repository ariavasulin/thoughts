# Letta to Agno Greenfield Rebuild

## Overview

Replace Letta with Agno via greenfield rebuild, eliminating ~20% of the codebase (2,445 lines) that contains global state patterns, sandbox tool duplication, and Letta sync complexity. Replace with ~500 lines of clean Agno code.

**Net result:** 12,200 → 10,300 lines (~16% reduction), with the gnarliest complexity removed.

## Current State Analysis

### Pain Points (Why Rebuild)

1. **Global State Patterns** - `_letta_client`, `_honcho_client`, `_user_context` globals set at startup, used by tools. Breaks in sandboxed environments.

2. **Sandbox Tool Duplication** - `tools/sandbox.py` (200+ lines) duplicates `memory.py` and `dialectic.py` as HTTP-based tools because Letta's Docker sandbox can't import `youlab_server`.

3. **Letta Sync Complexity** - `UserBlockManager` has ~100 lines syncing git blocks → Letta blocks. Adds latency and failure modes.

4. **Three Streaming Layers** - Letta SDK → `AgentManager.stream_message()` → HTTP endpoint → Pipeline adapter. Each transforms events.

5. **Background Agent Factory** - Complex creation/deletion with `fresh=True` deleting agents to prevent message history confusion.

6. **Context Duplication** - OpenWebUI sends full thread, we append Honcho context, creating duplicate/conflicting context.

### What Works Well (Keep)

- Git-backed storage (`storage/git.py`) - versioned markdown files
- Pending diff system (`storage/diffs.py`) - agent-proposed changes
- Block schema system (`curriculum/`) - TOML-defined, Pydantic-generated
- Course configuration (`config/courses/`) - TOML structure
- Honcho client base (`honcho/client.py`) - just needs `get_context()`
- **OpenWebUI frontend** - Memory UI, diff approval, module view already built

### Files to Delete (2,445 lines)

| File | Lines | Reason |
|------|-------|--------|
| `server/agents.py` | 543 | Letta AgentManager, caching, shared blocks |
| `tools/sandbox.py` | 201 | HTTP tool duplication (eliminated entirely) |
| `tools/memory.py` | 114 | Global state version |
| `tools/dialectic.py` | 102 | Global state version |
| `tools/curriculum.py` | 325 | Global state + complex YAML parsing |
| `background/factory.py` | 318 | Letta background agent factory |
| `background/runner.py` | 572 | Complex Letta orchestration |
| `pipelines/letta_pipe.py` | 270 | Letta streaming adapter |

### Files to Keep (Minor Edits)

| File | Lines | Edit |
|------|-------|------|
| `storage/git.py` | 474 | Keep as-is |
| `storage/diffs.py` | 143 | Keep as-is |
| `storage/blocks.py` | 363 | Remove ~60 lines Letta sync |
| `curriculum/*` | 1,012 | Keep as-is |
| `honcho/client.py` | 362 | Add ~30 lines for `get_context()` |

## Desired End State

### Architecture

```
OpenWebUI (UI, threads, memory UI, module view)
    │
    ▼ Pipe receives full thread (USE AS-IS)
    │
YouLab Pipe (pipelines/agno_pipe.py)
    │ ✓ Forward thread messages to server
    │ ✓ Extract user_id, chat_id, course_id
    │
    ▼ POST /chat/stream
    │
YouLab Server (server/main.py)
    │
    ├─► Git Storage: get memory blocks
    │   Returns: student profile, journey, etc.
    │
    ├─► Build Agno Agent
    │   - Model: Claude (configurable)
    │   - Tools: [edit_memory_block, query_honcho, advance_lesson]
    │   - Session state: {user_id, chat_id, course_id, honcho_client, ...}
    │   - System prompt: course config + memory blocks
    │   - Messages: from OpenWebUI (thread history)
    │
    ├─► Stream response via Agno
    │
    └─► Fire-and-forget persist to Honcho (for dialectic queries)
```

**Note on thread management:** OpenWebUI handles thread/context natively. Long thread summarization is a future optimization - not needed for MVP.

### New Module Structure (~500 lines total)

```
src/youlab_server/
  agents/
    __init__.py
    agno.py          # ~100 lines - Agent factory with session state
  tools/
    __init__.py
    memory.py        # ~40 lines - edit_memory_block with RunContext
    dialectic.py     # ~35 lines - query_honcho with RunContext
    curriculum.py    # ~80 lines - advance_lesson with RunContext
  server/
    main.py          # ~200 lines - Simplified, no globals
  pipelines/
    agno_pipe.py     # ~80 lines - Simple SSE streaming
  background/
    runner.py        # ~150 lines - Simplified Agno runner
```

### Key Characteristics

1. **Zero global state** - All context via `RunContext.session_state`
2. **Zero tool duplication** - One version of each tool, runs in-process
3. **Zero Letta sync** - Git is sole source of truth for blocks
4. **Single streaming layer** - Agno → SSE directly
5. **Honcho for context** - `get_context()` with summarization
6. **OpenWebUI-first** - Backend serves the existing frontend

## What We're NOT Doing

- Rebuilding OpenWebUI frontend (already done)
- Changing course TOML structure
- Changing git storage format
- Changing pending diff workflow
- Adding new features (pure migration)
- Changing Honcho's role (still dialectic + context)

## Design Principles

### 1. OpenWebUI-First

> "The backend serves the frontend integration, not the other way around."

Every phase must answer:
- How does this appear in OpenWebUI?
- Does this work with the existing memory UI?
- Does this work with the existing module view?

**Mistake to avoid:** Building backend in isolation, then struggling to integrate.

### 2. Use OpenWebUI's Native Thread Management

OpenWebUI sends full thread history. We use it as-is:
- Pass messages directly to Agno agent
- Memory blocks injected into system prompt
- Honcho persists messages for dialectic queries (fire-and-forget)
- Long thread summarization is a future optimization

### 3. Stateless Agents, Stateful Context

Agno agents are stateless (created per-request). State lives in:
- **Honcho** - Conversation history, peer representations
- **Git** - Memory blocks, pending diffs
- **Session state** - Per-request context passed to tools

### 4. In-Process Tools

No sandboxing. Tools run in the same process as the server:
- No HTTP tool duplication needed
- No global state needed
- `RunContext.session_state` provides all context

### 5. Incremental, Testable Phases

Each phase produces a working system that can be tested against:
- Existing OpenWebUI frontend
- Existing HTTP API contracts
- Existing course configurations

---

## Phase 1: Agno Foundation + Tools

### Overview

Set up Agno agent creation and migrate tools to use `RunContext`. This phase establishes the core pattern without changing the HTTP layer.

### Changes Required

#### 1. Install Agno

**File**: `pyproject.toml`

Add dependency:
```toml
dependencies = [
    "agno",
    # ... existing deps
]
```

#### 2. Create Agno Agent Factory

**File**: `src/youlab_server/agents/agno.py` (new)

```python
"""Agno agent factory with session state injection."""

from agno.agent import Agent
from agno.models.anthropic import Claude

from youlab_server.tools import edit_memory_block, query_honcho, advance_lesson


def create_agent(
    user_id: str,
    chat_id: str,
    course_id: str,
    honcho_client: "HonchoClient | None",
    memory_blocks: dict[str, str],
    system_prompt: str,
    model: str = "claude-sonnet-4-20250514",
) -> Agent:
    """Create an Agno agent with session state for tools.

    Args:
        user_id: The user/student ID
        chat_id: The current chat/thread ID
        course_id: The course ID for curriculum tools
        honcho_client: Honcho client for dialectic queries
        memory_blocks: Dict of block_label -> content
        system_prompt: The agent's system prompt (from course config)
        model: Claude model ID

    Returns:
        Configured Agno Agent ready for chat
    """
    return Agent(
        model=Claude(id=model),
        tools=[edit_memory_block, query_honcho, advance_lesson],
        system_prompt=_build_system_prompt(system_prompt, memory_blocks),
        session_state={
            "user_id": user_id,
            "chat_id": chat_id,
            "course_id": course_id,
            "honcho_client": honcho_client,
        },
    )


def _build_system_prompt(base_prompt: str, blocks: dict[str, str]) -> str:
    """Build system prompt with embedded memory blocks."""
    if not blocks:
        return base_prompt

    block_context = "\n\n".join([
        f"## {label.replace('_', ' ').title()}\n{content}"
        for label, content in blocks.items()
    ])

    return f"""{base_prompt}

# Student Context (Memory Blocks)

{block_context}
"""
```

#### 3. Migrate edit_memory_block Tool

**File**: `src/youlab_server/tools/memory.py` (rewrite)

```python
"""Memory block editing tool using Agno RunContext."""

from agno.tools import tool
from agno.run.response import RunContext


@tool(
    name="edit_memory_block",
    description="Propose changes to a student's memory block",
)
def edit_memory_block(
    ctx: RunContext,
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
) -> str:
    """Propose changes to a memory block field.

    Args:
        block: Memory block label (e.g., "student", "journey")
        field: Specific field to update within the block
        content: Content to add or replace
        strategy: Merge strategy - "append", "replace", or "full_replace"
        reasoning: Explanation for the change

    Returns:
        Confirmation message with diff ID for user review.
    """
    from youlab_server.storage import get_storage_manager
    from youlab_server.storage.blocks import UserBlockManager

    user_id = ctx.session_state["user_id"]

    storage_manager = get_storage_manager()
    user_storage = storage_manager.get(user_id)

    if not user_storage:
        return f"Error: No storage found for user {user_id}"

    manager = UserBlockManager(user_id, user_storage)

    diff = manager.propose_edit(
        agent_id="agno_tutor",
        block_label=block,
        field=field,
        proposed_value=content,
        operation=strategy,
        reasoning=reasoning,
    )

    return f"Proposed change to {block}.{field} (diff: {diff.id[:8]}). Awaiting user review."
```

#### 4. Migrate query_honcho Tool

**File**: `src/youlab_server/tools/dialectic.py` (rewrite)

```python
"""Honcho dialectic query tool using Agno RunContext."""

import asyncio
from agno.tools import tool
from agno.run.response import RunContext


@tool(
    name="query_honcho",
    description="Query conversation history for insights about the student",
)
def query_honcho(
    ctx: RunContext,
    question: str,
    session_scope: str = "all",
) -> str:
    """Query conversation history to understand the student better.

    Args:
        question: Natural language question about the student
        session_scope: Which sessions to query - "all", "recent", or "current"

    Returns:
        Insight about the student based on conversation history.
    """
    honcho_client = ctx.session_state.get("honcho_client")
    user_id = ctx.session_state["user_id"]

    if not honcho_client:
        return "Honcho not available for conversation queries."

    # Run async query synchronously
    try:
        loop = asyncio.get_event_loop()
    except RuntimeError:
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)

    result = loop.run_until_complete(
        honcho_client.query_dialectic(user_id, question)
    )

    if not result:
        return "Unable to retrieve insights from conversation history."

    return result.insight
```

#### 5. Migrate advance_lesson Tool

**File**: `src/youlab_server/tools/curriculum.py` (rewrite)

```python
"""Curriculum advancement tool using Agno RunContext."""

from agno.tools import tool
from agno.run.response import RunContext


@tool(
    name="advance_lesson",
    description="Advance the student to the next lesson in the curriculum",
)
def advance_lesson(
    ctx: RunContext,
    reason: str,
) -> str:
    """Request advancement to the next lesson.

    Args:
        reason: Explanation for why the student is ready to advance

    Returns:
        Next lesson opening message or status update.
    """
    from youlab_server.curriculum import get_curriculum_loader
    from youlab_server.storage import get_storage_manager
    from youlab_server.storage.blocks import UserBlockManager

    user_id = ctx.session_state["user_id"]
    course_id = ctx.session_state["course_id"]

    # Get curriculum and storage
    loader = get_curriculum_loader()
    course = loader.get_course(course_id)

    if not course:
        return f"Error: Course {course_id} not found"

    storage_manager = get_storage_manager()
    user_storage = storage_manager.get(user_id)
    manager = UserBlockManager(user_id, user_storage)

    # Read current journey
    journey_content = manager.get_block_body("journey")
    current_position = _parse_journey(journey_content)

    # Find next lesson
    next_lesson = _find_next_lesson(course, current_position)

    if not next_lesson:
        return "Congratulations! You have completed all lessons in this course."

    # Propose journey update
    new_position = f"module_id: {next_lesson.module_id}\nlesson_id: {next_lesson.lesson_id}\nstatus: in_progress"

    manager.propose_edit(
        agent_id="agno_tutor",
        block_label="journey",
        field="position",
        proposed_value=new_position,
        operation="full_replace",
        reasoning=reason,
    )

    return f"Advanced to: {next_lesson.title}\n\n{next_lesson.opening_message}"


def _parse_journey(content: str) -> dict:
    """Parse journey block content into position dict."""
    result = {}
    for line in content.strip().split("\n"):
        if ":" in line:
            key, value = line.split(":", 1)
            result[key.strip()] = value.strip()
    return result


def _find_next_lesson(course, current: dict):
    """Find the next lesson after current position."""
    # TODO: Implement based on course structure
    # This is simplified - full implementation will iterate modules/lessons
    return None
```

#### 6. Update Tool Exports

**File**: `src/youlab_server/tools/__init__.py` (rewrite)

```python
"""YouLab agent tools using Agno RunContext."""

from youlab_server.tools.memory import edit_memory_block
from youlab_server.tools.dialectic import query_honcho
from youlab_server.tools.curriculum import advance_lesson

__all__ = [
    "edit_memory_block",
    "query_honcho",
    "advance_lesson",
]
```

### Success Criteria

#### Automated Verification:
- [ ] `uv add agno` installs successfully
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] New tool modules import without error: `python -c "from youlab_server.tools import edit_memory_block, query_honcho, advance_lesson"`
- [ ] Agent factory imports: `python -c "from youlab_server.agents.agno import create_agent"`

#### Manual Verification:
- [ ] Create agent in Python REPL and verify session_state is accessible
- [ ] Tools receive RunContext with correct session_state values

---

## Phase 2: Remove Letta Sync from Storage

### Overview

Remove all Letta-specific code from `UserBlockManager`. Git becomes the sole source of truth for blocks.

### Changes Required

#### 1. Clean UserBlockManager

**File**: `src/youlab_server/storage/blocks.py`

Remove:
- Line 12: `from letta_client import Letta` import
- Line 33: `letta_client: Letta | None = None` parameter
- Line 37: `self.letta = letta_client` assignment
- Lines 41-43: `_letta_block_name()` method
- Lines 73, 85, 100-101: `sync_to_letta` parameter and logic in `update_block()`
- Lines 147, 152-153: `sync_to_letta` parameter and logic in `restore_version()`
- Lines 158-203: Entire `_sync_block_to_letta()` and `get_or_create_letta_block_id()` methods
- Line 319: `sync_to_letta=True` in `approve_diff()`

**Simplified `__init__`:**
```python
def __init__(self, user_id: str, storage: GitUserStorage):
    """Initialize block manager for a user.

    Args:
        user_id: The user ID
        storage: Git storage instance for this user
    """
    self.user_id = user_id
    self.storage = storage
    self.diffs = PendingDiffStore(storage.diffs_dir)
```

**Simplified `update_block`:**
```python
def update_block(
    self,
    label: str,
    content: str,
    message: str = "Update block",
    author: str = "system",
    schema: str | None = None,
    title: str | None = None,
) -> str:
    """Update a block's content and commit to git.

    Returns:
        The commit SHA
    """
    return self.storage.write_block(
        label=label,
        content=content,
        message=message,
        author=author,
        schema=schema,
        title=title,
    )
```

#### 2. Update All Instantiation Sites

**Files to update:**
- `src/youlab_server/server/users.py` - Remove `letta_client` parameter
- `src/youlab_server/server/blocks.py` - Remove `letta_client` parameter
- `src/youlab_server/background/runner.py` - Remove `letta_client` parameter
- `src/youlab_server/tools/memory.py` - Already updated in Phase 1

Search and replace pattern:
```python
# Old
UserBlockManager(user_id, user_storage, letta_client=self.letta)

# New
UserBlockManager(user_id, user_storage)
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] No references to `letta` in `storage/blocks.py`: `grep -c "letta" src/youlab_server/storage/blocks.py` returns 0
- [ ] All tests pass: `make test-agent`

#### Manual Verification:
- [ ] Create a pending diff via API, verify it saves to git
- [ ] Approve a diff via API, verify git commit is created
- [ ] Block content is correct after approval

---

## Phase 3: Streaming Pipeline

### Overview

Replace the three-layer Letta streaming with a single Agno streaming layer. OpenWebUI thread history is used as-is.

### Changes Required

#### 1. Create Agno Streaming Endpoint

**File**: `src/youlab_server/server/main.py`

Replace the existing `/chat/stream` endpoint:

```python
@app.post("/chat/stream")
async def chat_stream(request: StreamChatRequest):
    """Stream chat response using Agno agent.

    OpenWebUI thread history is passed through directly.
    Memory blocks are loaded from git and injected into system prompt.
    """
    from youlab_server.agents.agno import create_agent
    from youlab_server.curriculum import get_curriculum_loader
    from youlab_server.storage import get_storage_manager
    from youlab_server.storage.blocks import UserBlockManager

    user_id = request.user_id
    chat_id = request.chat_id
    course_id = request.course_id or "college-essay"
    messages = request.messages  # Full thread from OpenWebUI

    # Load memory blocks from git storage
    memory_blocks = {}
    storage_manager = get_storage_manager()
    user_storage = storage_manager.get(user_id)
    if user_storage:
        manager = UserBlockManager(user_id, user_storage)
        for label in manager.list_blocks():
            memory_blocks[label] = manager.get_block_body(label)

    # Get system prompt from course config
    loader = get_curriculum_loader()
    course = loader.get_course(course_id)
    system_prompt = course.agent.system_prompt if course else ""

    # Create agent with memory blocks in system prompt
    agent = create_agent(
        user_id=user_id,
        chat_id=chat_id,
        course_id=course_id,
        honcho_client=app.state.honcho_client,
        memory_blocks=memory_blocks,
        system_prompt=system_prompt,
    )

    # Fire-and-forget: persist latest user message to Honcho
    if app.state.honcho_client and chat_id and messages:
        latest_user_msg = messages[-1].get("content", "") if messages[-1].get("role") == "user" else ""
        if latest_user_msg:
            create_persist_task(
                app.state.honcho_client,
                user_id, chat_id, latest_user_msg,
                is_user=True,
            )

    async def stream_response():
        full_response = []

        # Pass full message history to Agno
        async for chunk in agent.arun_response_stream(messages):
            # Convert to SSE format
            if chunk.content:
                full_response.append(chunk.content)
                yield f"data: {json.dumps({'type': 'message', 'content': chunk.content})}\n\n"

            if hasattr(chunk, 'tool_calls') and chunk.tool_calls:
                for tc in chunk.tool_calls:
                    yield f"data: {json.dumps({'type': 'status', 'content': f'Using {tc.name}...'})}\n\n"

        yield f"data: {json.dumps({'type': 'done'})}\n\n"

        # Fire-and-forget: persist agent response
        response_text = "".join(full_response)
        if app.state.honcho_client and chat_id and response_text:
            create_persist_task(
                app.state.honcho_client,
                user_id, chat_id, response_text,
                is_user=False,
            )

    return StreamingResponse(
        stream_response(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

#### 2. Create Agno Pipe for OpenWebUI

**File**: `src/youlab_server/pipelines/agno_pipe.py` (new)

```python
"""OpenWebUI Pipe for YouLab using Agno agents.

This pipe forwards the full message history from OpenWebUI to the YouLab server.
Memory blocks are injected server-side into the system prompt.
"""

import json
import httpx
from typing import Iterator, Union
from pydantic import BaseModel, Field


class Pipe:
    """YouLab Agno Pipe for OpenWebUI."""

    class Valves(BaseModel):
        youlab_server_url: str = Field(
            default="http://localhost:8100",
            description="YouLab server URL"
        )

    def __init__(self):
        self.valves = self.Valves()

    def pipe(
        self,
        body: dict,
        __user__: dict | None = None,
        __event_emitter__=None,
    ) -> Union[str, Iterator[str]]:
        """Process chat request.

        Forwards full message history to YouLab server.
        Server handles memory block injection and Honcho persistence.
        """
        messages = body.get("messages", [])
        if not messages:
            return "No message provided"

        # Get user info
        user_id = __user__.get("id", "anonymous") if __user__ else "anonymous"
        chat_id = body.get("chat_id")

        # Determine course from model name (e.g., "youlab/college-essay")
        model = body.get("model", "")
        course_id = model.split("/")[-1] if "/" in model else "college-essay"

        # Stream from YouLab server
        return self._stream_chat(
            user_id=user_id,
            chat_id=chat_id,
            course_id=course_id,
            messages=messages,
            event_emitter=__event_emitter__,
        )

    def _stream_chat(
        self,
        user_id: str,
        chat_id: str | None,
        course_id: str,
        messages: list,
        event_emitter,
    ) -> Iterator[str]:
        """Stream chat response from YouLab server."""

        with httpx.stream(
            "POST",
            f"{self.valves.youlab_server_url}/chat/stream",
            json={
                "user_id": user_id,
                "chat_id": chat_id,
                "course_id": course_id,
                "messages": messages,  # Full thread history
            },
            timeout=120.0,
        ) as response:
            for line in response.iter_lines():
                if not line.startswith("data: "):
                    continue

                data = json.loads(line[6:])
                event_type = data.get("type")

                if event_type == "message":
                    yield data.get("content", "")
                elif event_type == "status" and event_emitter:
                    event_emitter({"type": "status", "data": {"description": data.get("content")}})
                elif event_type == "done":
                    break
```

#### 3. Simplified Server Startup

**File**: `src/youlab_server/server/main.py`

Remove from lifespan startup:
- Letta client initialization
- Global state setters (`set_letta_client`, `set_honcho_client`, `set_user_context`)
- Tool registration with Letta

Keep:
- Honcho client initialization
- Storage manager initialization
- Curriculum loader initialization

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan manager."""
    settings = ServiceSettings()

    # Initialize Honcho client (optional)
    if settings.honcho_enabled:
        app.state.honcho_client = HonchoClient(
            workspace_id=settings.honcho_workspace_id,
            api_key=settings.honcho_api_key,
            environment=settings.honcho_environment,
        )
    else:
        app.state.honcho_client = None

    # Initialize storage manager
    app.state.storage_manager = GitUserStorageManager(
        base_dir=Path(settings.data_dir) / "users"
    )

    # Initialize curriculum loader
    app.state.curriculum_loader = CurriculumLoader(
        config_dir=Path("config/courses")
    )

    yield

    # Cleanup (if needed)
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Server starts without Letta: `uv run youlab-server` (should not require Letta server)
- [ ] `/chat/stream` endpoint responds: `curl -X POST localhost:8100/chat/stream -H "Content-Type: application/json" -d '{"user_id": "test", "message": "hello"}'`

#### Manual Verification:
- [ ] Stream a message through OpenWebUI with the new pipe
- [ ] Verify response streams correctly
- [ ] Verify messages are persisted to Honcho
- [ ] Verify memory blocks are included in agent context
- [ ] Verify tool calls appear as status updates in UI

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the OpenWebUI integration works correctly before proceeding.

---

## Phase 4: Module Sync

### Overview

Auto-sync course modules as OpenWebUI "models" via the OpenWebUI API. Each course appears as a selectable model in the UI.

### Changes Required

#### 1. Create Module Sync Service

**File**: `src/youlab_server/server/module_sync.py` (new)

```python
"""Sync YouLab courses as OpenWebUI models."""

import httpx
from pathlib import Path

from youlab_server.curriculum import get_curriculum_loader


async def sync_courses_to_openwebui(
    openwebui_url: str,
    api_key: str,
    base_pipe_id: str = "youlab-pipe",
) -> dict[str, bool]:
    """Sync all courses as OpenWebUI models.

    Args:
        openwebui_url: OpenWebUI API URL
        api_key: OpenWebUI API key
        base_pipe_id: The base pipe ID to use

    Returns:
        Dict of course_id -> success status
    """
    loader = get_curriculum_loader()
    results = {}

    async with httpx.AsyncClient() as client:
        for course_id in loader.list_courses():
            course = loader.get_course(course_id)
            if not course:
                results[course_id] = False
                continue

            # Create/update model in OpenWebUI
            model_data = {
                "id": f"youlab/{course_id}",
                "name": course.title,
                "description": course.description,
                "base_model_id": base_pipe_id,
                "meta": {
                    "course_id": course_id,
                    "version": course.version,
                },
            }

            try:
                response = await client.post(
                    f"{openwebui_url}/api/models",
                    json=model_data,
                    headers={"Authorization": f"Bearer {api_key}"},
                )
                results[course_id] = response.status_code in (200, 201)
            except Exception:
                results[course_id] = False

    return results
```

#### 2. Add Sync Endpoint

**File**: `src/youlab_server/server/main.py`

```python
@app.post("/admin/sync-modules")
async def sync_modules(
    openwebui_url: str = "http://localhost:3000",
    api_key: str = "",
):
    """Sync courses to OpenWebUI as models."""
    from youlab_server.server.module_sync import sync_courses_to_openwebui

    results = await sync_courses_to_openwebui(
        openwebui_url=openwebui_url,
        api_key=api_key,
    )

    return {"synced": results}
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Module sync imports: `python -c "from youlab_server.server.module_sync import sync_courses_to_openwebui"`

#### Manual Verification:
- [ ] Call `/admin/sync-modules` with valid OpenWebUI credentials
- [ ] Verify courses appear as models in OpenWebUI model selector
- [ ] Verify selecting a course model routes to correct course config

---

## Phase 5: Background Agents

### Overview

Replace the complex Letta background agent system with simple Agno agents. Same tools, simpler lifecycle.

### Changes Required

#### 1. Simplified Background Runner

**File**: `src/youlab_server/background/runner.py` (rewrite)

```python
"""Simplified background agent runner using Agno."""

import asyncio
from dataclasses import dataclass
from pathlib import Path

from youlab_server.agents.agno import create_agent
from youlab_server.agents.context import load_agent_context
from youlab_server.storage import get_storage_manager
from youlab_server.honcho.client import HonchoClient


@dataclass
class BackgroundTaskResult:
    """Result from a background task run."""
    user_id: str
    task_name: str
    success: bool
    message: str
    diffs_created: int = 0


class BackgroundRunner:
    """Run background tasks for users using Agno agents."""

    def __init__(
        self,
        honcho_client: HonchoClient | None = None,
    ):
        self.honcho = honcho_client
        self.storage_manager = get_storage_manager()

    async def run_task(
        self,
        task_name: str,
        user_id: str,
        course_id: str,
        instruction: str,
    ) -> BackgroundTaskResult:
        """Run a background task for a user.

        Args:
            task_name: Name of the task (for logging)
            user_id: Target user ID
            course_id: Course context
            instruction: What the agent should do

        Returns:
            BackgroundTaskResult with outcome
        """
        try:
            # Load context (no chat_id for background tasks)
            context = await load_agent_context(
                user_id=user_id,
                chat_id=None,
                course_id=course_id,
                honcho_client=self.honcho,
            )

            # Create agent with background-specific prompt
            agent = create_agent(
                user_id=user_id,
                chat_id=None,
                course_id=course_id,
                honcho_client=self.honcho,
                memory_blocks=context["memory_blocks"],
                system_prompt=self._background_system_prompt(),
            )

            # Run the task
            response = await agent.arun(instruction)

            return BackgroundTaskResult(
                user_id=user_id,
                task_name=task_name,
                success=True,
                message=response.content,
                diffs_created=self._count_new_diffs(user_id),
            )

        except Exception as e:
            return BackgroundTaskResult(
                user_id=user_id,
                task_name=task_name,
                success=False,
                message=str(e),
            )

    def _background_system_prompt(self) -> str:
        return """You are a background analysis agent for YouLab.

Your job is to analyze conversation history and update student memory blocks.

Use these tools:
- query_honcho: Ask questions about the student's conversation history
- edit_memory_block: Propose updates to memory blocks

Be concise. Only propose meaningful updates based on evidence from conversations."""

    def _count_new_diffs(self, user_id: str) -> int:
        """Count pending diffs for user (rough metric of work done)."""
        from youlab_server.storage.blocks import UserBlockManager

        storage = self.storage_manager.get(user_id)
        if not storage:
            return 0

        manager = UserBlockManager(user_id, storage)
        return sum(manager.count_pending_diffs().values())
```

#### 2. Background Task Endpoint

**File**: `src/youlab_server/server/main.py`

```python
@app.post("/background/{task_name}/run")
async def run_background_task(
    task_name: str,
    user_id: str,
    course_id: str = "college-essay",
    instruction: str = "Analyze recent conversations and update memory blocks as needed.",
):
    """Run a background task for a user."""
    from youlab_server.background.runner import BackgroundRunner

    runner = BackgroundRunner(honcho_client=app.state.honcho_client)
    result = await runner.run_task(
        task_name=task_name,
        user_id=user_id,
        course_id=course_id,
        instruction=instruction,
    )

    return {
        "success": result.success,
        "message": result.message,
        "diffs_created": result.diffs_created,
    }
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Background runner imports: `python -c "from youlab_server.background.runner import BackgroundRunner"`

#### Manual Verification:
- [ ] Run background task via API for a test user
- [ ] Verify agent queries Honcho and proposes memory updates
- [ ] Verify pending diffs are created

---

## Phase 6: Cleanup

### Overview

Delete all Letta-specific code, update dependencies, and verify the complete system.

### Changes Required

#### 1. Delete Letta Files

```bash
rm src/youlab_server/server/agents.py
rm src/youlab_server/tools/sandbox.py
rm src/youlab_server/background/factory.py
rm src/youlab_server/pipelines/letta_pipe.py
```

#### 2. Remove Letta Dependencies

**File**: `pyproject.toml`

Remove:
```toml
"letta-client",
```

#### 3. Update Imports

Search for any remaining imports from deleted modules and update:

```bash
grep -r "from youlab_server.server.agents import" src/
grep -r "from youlab_server.tools.sandbox import" src/
grep -r "from youlab_server.background.factory import" src/
grep -r "from youlab_server.pipelines.letta_pipe import" src/
```

#### 4. Update Tests

- Remove tests for deleted modules
- Add tests for new Agno modules
- Update integration tests to use new endpoints

#### 5. Update Documentation

- Update `CLAUDE.md` to reflect new architecture
- Update `docs/` if any Letta-specific docs exist
- Update README if needed

### Success Criteria

#### Automated Verification:
- [ ] `make verify-agent` passes (full lint + typecheck + tests)
- [ ] No Letta imports remain: `grep -r "letta" src/youlab_server/ | grep -v "# letta" | wc -l` returns 0
- [ ] Server starts: `uv run youlab-server`
- [ ] All endpoints respond correctly

#### Manual Verification:
- [ ] Full conversation flow works in OpenWebUI
- [ ] Memory blocks display correctly in UI
- [ ] Diff approval works in UI
- [ ] Module selection works in UI
- [ ] Background tasks run successfully

**Implementation Note**: This is the final phase. After all verification passes, the migration is complete.

---

## Testing Strategy

### Unit Tests

- `tests/test_agents/test_agno.py` - Agent creation, session state
- `tests/test_tools/test_memory.py` - edit_memory_block with mock RunContext
- `tests/test_tools/test_dialectic.py` - query_honcho with mock RunContext
- `tests/test_honcho/test_context.py` - get_context integration

### Integration Tests

- `tests/integration/test_chat_flow.py` - Full chat with Honcho + blocks
- `tests/integration/test_streaming.py` - SSE streaming verification
- `tests/integration/test_background.py` - Background task execution

### Manual Testing Checklist

1. [ ] Start fresh conversation in OpenWebUI
2. [ ] Verify agent responds with context from memory blocks
3. [ ] Trigger a memory update (agent proposes diff)
4. [ ] Approve diff in UI
5. [ ] Verify memory block updated
6. [ ] Continue conversation, verify updated context used
7. [ ] Switch to different course module
8. [ ] Verify course-specific behavior
9. [ ] Run background task
10. [ ] Verify background-created diffs appear in UI

---

## Migration Notes

### Data Migration

**No data migration needed:**
- Git storage format unchanged
- Pending diff format unchanged
- Honcho data unchanged
- Course TOML format unchanged

### Rollback Plan

If issues arise:
1. Keep Letta server running during migration
2. Old code preserved in git history
3. Can revert by checking out pre-migration commit
4. No data format changes means no data migration to reverse

### Parallel Running

During migration, both systems can run:
- Old Letta system on port 8100
- New Agno system on port 8101
- Switch OpenWebUI pipe URL to test new system
- Switch back if issues

---

## References

- Research document: `thoughts/shared/research/2026-01-16-letta-to-agno-migration.md`
- Honcho reference: `thoughts/global/shared/reference/honcho-reference.md`
- Agno documentation: https://docs.agno.com
- Current Letta integration: `src/youlab_server/server/agents.py`
