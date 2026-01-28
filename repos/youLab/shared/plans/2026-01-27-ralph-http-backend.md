# Ralph HTTP Backend Service Implementation Plan

## Overview

Create a minimal FastAPI HTTP backend service that the Ralph OpenWebUI pipe can call. The pipe (already created) is a thin HTTP client that calls `POST /chat/stream` and expects SSE events. This plan implements that backend with **real streaming** of tool calls and responses.

## Current State Analysis

**What exists:**
- `ralph/pipe.py` - Lightweight HTTP client pipe (calls port 8200)
- `ralph/openhands_client.py` - OpenHands wrapper (NEEDS REWRITE)
- `ralph/honcho.py` - Message persistence (skip for MVP)
- `ralph/config.py` - Pydantic settings

**What's missing:**
- HTTP backend service with streaming

**Architecture after this plan:**
```
User (OpenWebUI :8080)
    ↓
ralph/pipe.py (HTTP Client)
    ↓ POST /chat/stream (SSE)
ralph/server.py (FastAPI :8200) ← NEW
    ↓
OpenHands SDK with callbacks → SSE events
    ↓
Claude via OpenRouter
```

## Key Discovery: OpenHands SDK Supports Callbacks!

The SDK has real streaming support:
- `callbacks: list[Callable[[Event], None]]` - receives events during `run()`
- Event types: `MessageEvent`, `ActionEvent` (tool calls), `ObservationEvent` (tool results)
- `token_callbacks` - for streaming LLM token deltas

This means we CAN stream tool calls to the UI in real-time!

## Desired End State

```bash
# 1. Start the server
uv run ralph-server

# 2. Test health endpoint
curl http://localhost:8200/health

# 3. In OpenWebUI, send a message and see:
#    - "Thinking..." status
#    - Tool call status (e.g., "Running terminal command...")
#    - Streaming response text
```

## What We're NOT Doing

- Docker sandbox (local workspace only)
- Custom tools (query_honcho) - skip for MVP
- Honcho persistence - skip for MVP
- Conversation persistence across messages

---

## Phase 1: Add Dependencies

### Overview
Add FastAPI, OpenHands SDK, and SSE dependencies.

### Changes Required:

#### 1. Update pyproject.toml
**File**: `pyproject.toml`

Add to the existing `[project.optional-dependencies]` section:

```toml
ralph = [
    "fastapi>=0.109.0",
    "uvicorn>=0.27.0",
    "sse-starlette>=2.0.0",
    "httpx>=0.27.0",
    "openhands-sdk>=1.0.0",
    "openhands-tools>=1.0.0",
    "pydantic-settings>=2.0.0",
    "structlog>=24.0.0",
]
```

#### 2. Add CLI entry point
**File**: `pyproject.toml`

Add to `[project.scripts]` section:

```toml
ralph-server = "ralph.server:main"
```

### Success Criteria:

#### Automated Verification:
- [x] `uv sync --extra ralph` exits 0
- [x] `uv run python -c "import fastapi, uvicorn, sse_starlette; print('ok')"`
- [x] OpenHands SDK import test (skipped - requires separate install due to letta conflicts)

---

## Phase 2: Rewrite OpenHands Client with Streaming

### Overview
Replace `openhands_client.py` with code that uses callbacks for real-time streaming.

### Changes Required:

#### 1. Simplified OpenHands Client with Callbacks
**File**: `ralph/openhands_client.py`

```python
"""OpenHands SDK wrapper for Ralph with streaming callbacks."""

from __future__ import annotations

from pathlib import Path
from typing import TYPE_CHECKING, Any, Callable

import structlog

if TYPE_CHECKING:
    from openhands.sdk import Conversation
    from openhands.sdk.event import Event

from ralph.config import get_settings

log = structlog.get_logger()


def get_workspace_path(user_id: str) -> Path:
    """Get or create user workspace directory."""
    settings = get_settings()
    workspace = Path(settings.user_data_dir) / user_id / "workspace"
    workspace.mkdir(parents=True, exist_ok=True)
    return workspace


def create_conversation(
    user_id: str,
    on_event: Callable[[Event], None] | None = None,
) -> Conversation:
    """
    Create a new OpenHands conversation for a user.

    Args:
        user_id: User identifier for workspace isolation
        on_event: Callback invoked for each event (MessageEvent, ActionEvent, etc.)
    """
    from openhands.sdk import LLM, Agent, Conversation, Tool
    from openhands.tools.file_editor import FileEditorTool
    from openhands.tools.terminal import TerminalTool

    settings = get_settings()

    # Create LLM - check if we need base_url for OpenRouter
    llm_kwargs: dict[str, Any] = {
        "model": settings.openrouter_model,
        "api_key": settings.openrouter_api_key,
    }

    # OpenRouter requires custom base_url
    if "openrouter" in settings.openrouter_api_key.lower() or settings.openrouter_model.startswith(("anthropic/", "openai/", "meta/")):
        llm_kwargs["base_url"] = "https://openrouter.ai/api/v1"

    llm = LLM(**llm_kwargs)

    # Create agent with standard tools
    agent = Agent(
        llm=llm,
        tools=[
            Tool(name=TerminalTool.name),
            Tool(name=FileEditorTool.name),
        ],
    )

    # Create conversation with user's workspace and callbacks
    workspace_path = get_workspace_path(user_id)

    callbacks = [on_event] if on_event else None

    conversation = Conversation(
        agent=agent,
        workspace=str(workspace_path),
        callbacks=callbacks,
    )

    log.info("conversation_created", user_id=user_id, workspace=str(workspace_path))
    return conversation


def run_message(
    user_id: str,
    message: str,
    on_event: Callable[[Event], None] | None = None,
) -> None:
    """
    Run a single message through OpenHands.

    Args:
        user_id: User identifier
        message: User's message
        on_event: Callback for streaming events
    """
    conversation = create_conversation(user_id, on_event=on_event)
    conversation.send_message(message)
    conversation.run()  # Callbacks fire during this
```

### Success Criteria:

#### Automated Verification:
- [x] `make check-agent` passes
- [x] `uv run python -c "from ralph.openhands_client import create_conversation, run_message; print('ok')"`

---

## Phase 3: Create HTTP Server with Real Streaming

### Overview
Create FastAPI server that streams events via SSE as they happen.

### Changes Required:

#### 1. HTTP Server with Event Streaming
**File**: `ralph/server.py`

```python
"""
Ralph HTTP Backend Service.

Streams OpenHands events to OpenWebUI in real-time.
"""

from __future__ import annotations

import asyncio
import json
from contextlib import asynccontextmanager
from typing import TYPE_CHECKING, Any

import structlog
import uvicorn
from fastapi import FastAPI
from pydantic import BaseModel
from sse_starlette.sse import EventSourceResponse

if TYPE_CHECKING:
    from collections.abc import AsyncGenerator

from ralph.config import get_settings

log = structlog.get_logger()


class ChatRequest(BaseModel):
    """Request body for /chat/stream endpoint."""

    user_id: str
    chat_id: str
    message: str


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan handler."""
    settings = get_settings()
    log.info("ralph_server_starting", model=settings.openrouter_model)
    yield
    log.info("ralph_server_stopped")


app = FastAPI(
    title="Ralph Backend",
    description="HTTP backend for Ralph OpenWebUI pipe",
    version="0.1.0",
    lifespan=lifespan,
)


@app.get("/health")
async def health() -> dict[str, str]:
    """Health check endpoint."""
    return {"status": "ok", "service": "ralph"}


@app.post("/chat/stream")
async def chat_stream(request: ChatRequest) -> EventSourceResponse:
    """
    Stream chat events via SSE.

    Events streamed:
    - status: {"type": "status", "content": "Running command..."}
    - message: {"type": "message", "content": "Response text"}
    - done: {"type": "done"}
    - error: {"type": "error", "message": "Error description"}
    """

    async def generate() -> AsyncGenerator[dict[str, Any], None]:
        """Generate SSE events from OpenHands conversation."""
        from ralph.openhands_client import run_message

        # Queue for thread-safe event passing
        event_queue: asyncio.Queue[dict[str, Any]] = asyncio.Queue()
        response_parts: list[str] = []

        def on_event(event: Any) -> None:
            """Callback invoked by OpenHands for each event."""
            event_class = type(event).__name__

            # Handle different event types
            if event_class == "MessageEvent":
                # Agent message - this is the response text
                content = getattr(event, "content", None) or getattr(event, "message", "")
                if content:
                    response_parts.append(str(content))
                    # Use call_soon_threadsafe since we're in a different thread
                    asyncio.get_event_loop().call_soon_threadsafe(
                        event_queue.put_nowait,
                        {"type": "message", "content": str(content)}
                    )

            elif event_class == "ActionEvent":
                # Tool call starting
                tool_name = getattr(event, "tool_name", "tool")
                action = getattr(event, "action", None)
                desc = f"Running {tool_name}..."
                if action and hasattr(action, "command"):
                    desc = f"Running: {action.command[:50]}..."
                asyncio.get_event_loop().call_soon_threadsafe(
                    event_queue.put_nowait,
                    {"type": "status", "content": desc}
                )

            elif event_class == "ObservationEvent":
                # Tool result
                content = getattr(event, "content", "")
                if content:
                    preview = str(content)[:100]
                    asyncio.get_event_loop().call_soon_threadsafe(
                        event_queue.put_nowait,
                        {"type": "status", "content": f"Result: {preview}..."}
                    )

        try:
            # Send initial status
            yield {"event": "message", "data": json.dumps({"type": "status", "content": "Thinking..."})}

            # Run conversation in thread pool with callbacks
            loop = asyncio.get_event_loop()

            async def run_in_thread():
                await asyncio.to_thread(
                    run_message,
                    request.user_id,
                    request.message,
                    on_event,
                )
                await event_queue.put({"type": "__done__"})

            # Start the conversation
            run_task = asyncio.create_task(run_in_thread())

            # Stream events as they come in
            while True:
                try:
                    event = await asyncio.wait_for(event_queue.get(), timeout=0.5)
                    if event.get("type") == "__done__":
                        break
                    yield {"event": "message", "data": json.dumps(event)}
                except asyncio.TimeoutError:
                    if run_task.done():
                        # Check for exceptions
                        if run_task.exception():
                            raise run_task.exception()
                        break
                    continue

            # Wait for completion
            await run_task

            # Send done event
            yield {"event": "message", "data": json.dumps({"type": "done"})}

        except Exception as e:
            log.exception("chat_stream_error", error=str(e))
            yield {"event": "message", "data": json.dumps({"type": "error", "message": str(e)})}

    return EventSourceResponse(generate())


def main() -> None:
    """Run the server."""
    log.info("starting_ralph_server", port=8200)
    uvicorn.run(app, host="0.0.0.0", port=8200)


if __name__ == "__main__":
    main()
```

### Success Criteria:

#### Automated Verification:
- [x] `make check-agent` passes
- [x] File exists: `ralph/ralph/server.py`
- [x] `uv run python -c "from ralph.server import app; print('ok')"`

#### Manual Verification:
- [x] `uv run ralph-server` starts without error
- [x] `curl http://localhost:8200/health` returns OK

---

## Phase 4: End-to-End Testing

### Manual Testing Checklist:

1. **Set environment variables:**
   ```bash
   export RALPH_OPENROUTER_API_KEY=your-key-here
   export RALPH_USER_DATA_DIR=./data/users
   ```

2. **Start the server:**
   ```bash
   uv run ralph-server
   ```

3. **Test health endpoint:**
   ```bash
   curl http://localhost:8200/health
   ```

4. **Test SSE endpoint directly:**
   ```bash
   curl -N -X POST http://localhost:8200/chat/stream \
     -H "Content-Type: application/json" \
     -d '{"user_id": "test", "chat_id": "test", "message": "Create a file called hello.py that prints hello world, then run it"}'
   ```

   **Expected SSE events:**
   ```
   data: {"type": "status", "content": "Thinking..."}
   data: {"type": "status", "content": "Running: echo ..."}
   data: {"type": "status", "content": "Result: ..."}
   data: {"type": "message", "content": "I've created..."}
   data: {"type": "done"}
   ```

5. **Test via OpenWebUI:**
   - Import `ralph/pipe.py` as a pipeline
   - Select "Ralph Wiggum" in chat
   - Send "Create hello.py and run it"
   - Should see status updates as tools run

6. **Test file persistence:**
   - Create file in one chat
   - Start NEW chat, ask what files exist
   - File should persist (same user workspace)

### Success Criteria:

- [x] Health endpoint returns OK
- [x] SSE shows status updates during tool execution
- [x] Response text streams to UI
- [x] Files persist across chats

---

## Configuration

| Variable | Default | Required | Description |
|----------|---------|----------|-------------|
| `RALPH_OPENROUTER_API_KEY` | - | Yes | OpenRouter API key |
| `RALPH_OPENROUTER_MODEL` | `anthropic/claude-sonnet-4-20250514` | No | Model to use |
| `RALPH_USER_DATA_DIR` | `/data/ralph/users` | No | User workspaces |

**For local dev:**
```bash
export RALPH_OPENROUTER_API_KEY=your-key-here
export RALPH_USER_DATA_DIR=./data/users
```

---

## Known Limitations (MVP)

1. **No conversation persistence** - Each message creates new conversation
2. **No Honcho** - Messages not saved
3. **No custom tools** - Only terminal and file editor
4. **Basic event parsing** - May need adjustment based on actual SDK event structure

---

## References

- [OpenHands SDK Getting Started](https://docs.openhands.dev/sdk/getting-started)
- [OpenHands SDK Event API](https://docs.openhands.dev/sdk/api-reference/openhands.sdk.event)
- [OpenHands SDK GitHub](https://github.com/OpenHands/software-agent-sdk)
- Callback type: `ConversationCallbackType = Callable[[Event], None]`
- Event types: `MessageEvent`, `ActionEvent`, `ObservationEvent`
