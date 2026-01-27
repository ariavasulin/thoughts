# Ralph HTTP Backend Service Implementation Plan

## Overview

Create a minimal FastAPI HTTP backend service that the Ralph OpenWebUI pipe can call. The pipe (already created) is a thin HTTP client that calls `POST /chat/stream` and expects SSE events. This plan implements that backend.

## Current State Analysis

**What exists:**
- `ralph/pipe.py` - Lightweight HTTP client pipe (calls port 8200)
- `ralph/openhands_client.py` - OpenHands workspace/conversation manager (NEEDS REWRITE)
- `ralph/honcho.py` - Message persistence + dialectic queries
- `ralph/config.py` - Pydantic settings

**What's missing:**
- HTTP backend service that receives requests from pipe and orchestrates OpenHands

**Architecture after this plan:**
```
User (OpenWebUI :8080)
    ↓
ralph/pipe.py (HTTP Client)
    ↓ POST /chat/stream (SSE)
ralph/server.py (FastAPI :8200) ← NEW
    ↓
OpenHands SDK (simple agent loop)
    ↓
Claude via OpenRouter
```

## Desired End State

**Verification:**
```bash
# 1. Start the server
uv run ralph-server

# 2. Test health endpoint
curl http://localhost:8200/health

# 3. In OpenWebUI, select "Ralph Wiggum" pipe and send a message
#    → See streaming response from Claude via OpenHands
```

## What We're NOT Doing

- Docker sandbox (local workspace only)
- Custom tools (query_honcho) - too complex for MVP
- Honcho persistence - skip for MVP
- Streaming callbacks - OpenHands SDK `run()` is blocking, we'll get response after completion
- Multi-model orchestration
- Complex error recovery

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

### Success Criteria:

#### Automated Verification:
- [ ] `uv sync --extra ralph` exits 0
- [ ] `uv run python -c "import fastapi, uvicorn, sse_starlette; print('ok')"`
- [ ] `uv run python -c "from openhands.sdk import LLM, Agent, Conversation; print('ok')"`

---

## Phase 2: Rewrite OpenHands Client

### Overview
Replace the speculative `openhands_client.py` with code that uses the real OpenHands SDK API.

### Changes Required:

#### 1. Simplified OpenHands Client
**File**: `ralph/openhands_client.py`

```python
"""OpenHands SDK wrapper for Ralph - simplified for MVP."""

from __future__ import annotations

import os
from pathlib import Path
from typing import TYPE_CHECKING

import structlog

if TYPE_CHECKING:
    from openhands.sdk import Conversation

from ralph.config import get_settings

log = structlog.get_logger()


def get_workspace_path(user_id: str) -> Path:
    """Get or create user workspace directory."""
    settings = get_settings()
    workspace = Path(settings.user_data_dir) / user_id / "workspace"
    workspace.mkdir(parents=True, exist_ok=True)
    return workspace


def create_conversation(user_id: str) -> Conversation:
    """
    Create a new OpenHands conversation for a user.

    Each conversation uses the user's persistent workspace directory.
    """
    from openhands.sdk import LLM, Agent, Conversation, Tool
    from openhands.tools.file_editor import FileEditorTool
    from openhands.tools.terminal import TerminalTool

    settings = get_settings()

    # Create LLM with OpenRouter
    llm = LLM(
        model=settings.openrouter_model,
        api_key=settings.openrouter_api_key,
        base_url="https://openrouter.ai/api/v1",
    )

    # Create agent with standard tools
    agent = Agent(
        llm=llm,
        tools=[
            Tool(name=TerminalTool.name),
            Tool(name=FileEditorTool.name),
        ],
    )

    # Create conversation with user's workspace
    workspace_path = get_workspace_path(user_id)

    conversation = Conversation(
        agent=agent,
        workspace=str(workspace_path),
    )

    log.info("conversation_created", user_id=user_id, workspace=str(workspace_path))
    return conversation


def run_conversation(user_id: str, message: str) -> str:
    """
    Run a single conversation turn and return the response.

    This is a simple blocking call - no streaming for MVP.
    """
    conversation = create_conversation(user_id)
    conversation.send_message(message)
    conversation.run()  # Blocking call

    # Get the response from conversation state
    # The SDK stores messages in conversation.state.messages
    state = getattr(conversation, 'state', None)
    if state and hasattr(state, 'messages'):
        # Find the last assistant message
        for msg in reversed(state.messages):
            if getattr(msg, 'role', None) == 'assistant':
                return getattr(msg, 'content', str(msg))

    return "Conversation completed but no response found."
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] `uv run python -c "from ralph.openhands_client import create_conversation; print('ok')"`

#### Manual Verification:
- [ ] With API key set, `run_conversation("test", "Say hello")` returns a response

---

## Phase 3: Create HTTP Server

### Overview
Create a minimal FastAPI server with `/chat/stream` SSE endpoint.

### Changes Required:

#### 1. HTTP Server
**File**: `ralph/server.py`

```python
"""
Ralph HTTP Backend Service.

Receives chat requests from the OpenWebUI pipe and streams responses.
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
    log.info(
        "ralph_server_starting",
        model=settings.openrouter_model,
    )
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
    Stream a chat response via SSE.

    For MVP, this runs the conversation synchronously and streams
    the complete response at once (no true streaming yet).
    """

    async def generate() -> AsyncGenerator[dict[str, Any], None]:
        """Generate SSE events."""
        try:
            # Send initial status
            yield {"event": "message", "data": json.dumps({"type": "status", "content": "Thinking..."})}

            # Import here to avoid startup issues
            from ralph.openhands_client import run_conversation

            # Run conversation in thread pool (it's blocking)
            response = await asyncio.to_thread(
                run_conversation,
                request.user_id,
                request.message,
            )

            # Send the response
            yield {"event": "message", "data": json.dumps({"type": "message", "content": response})}

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

#### 2. Add CLI entry point
**File**: `pyproject.toml`

Add to `[project.scripts]` section:

```toml
ralph-server = "ralph.server:main"
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] File exists: `ralph/server.py`
- [ ] `uv run python -c "from ralph.server import app; print('ok')"`

#### Manual Verification:
- [ ] `uv run ralph-server` starts without error
- [ ] `curl http://localhost:8200/health` returns `{"status": "ok", "service": "ralph"}`

---

## Phase 4: End-to-End Testing

### Overview
Test the full flow: OpenWebUI → Pipe → Server → OpenHands → Response.

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
   # Expected: {"status": "ok", "service": "ralph"}
   ```

4. **Test SSE endpoint directly:**
   ```bash
   curl -N -X POST http://localhost:8200/chat/stream \
     -H "Content-Type: application/json" \
     -d '{"user_id": "test", "chat_id": "test-chat", "message": "Say hello in exactly 5 words"}'
   # Expected: SSE events with response
   ```

5. **Test via OpenWebUI:**
   - Import `ralph/pipe.py` as a pipeline in OpenWebUI admin
   - Select "Ralph Wiggum" in chat
   - Send "Hello, who are you?"
   - Expected: Response from Claude

6. **Test file persistence:**
   - Send "Create a file called hello.py with print('Hello World')"
   - Start a NEW chat
   - Send "What files exist in the workspace?"
   - Expected: hello.py should be listed (workspace persists across chats)

### Success Criteria:

#### Manual Verification:
- [ ] Health endpoint returns OK
- [ ] SSE endpoint returns response
- [ ] OpenWebUI pipe shows response
- [ ] Files persist across chats (same user)

---

## Configuration

All configuration via environment variables with `RALPH_` prefix:

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

1. **No true streaming** - Response comes all at once after `conversation.run()` completes
2. **No Honcho persistence** - Messages not saved to Honcho
3. **No custom tools** - Only terminal and file editor
4. **New conversation per message** - No conversation persistence across messages in same chat

These can be added in future iterations.

---

## Implementation Order

1. **Phase 1**: Add dependencies
2. **Phase 2**: Rewrite openhands_client.py with correct SDK API
3. **Phase 3**: Create server.py and CLI entry point
4. **Phase 4**: End-to-end testing

---

## References

- OpenHands SDK docs: https://docs.openhands.dev/sdk/getting-started
- OpenHands SDK GitHub: https://github.com/OpenHands/software-agent-sdk
- Existing pipe (HTTP client): `ralph/pipe.py`
- OpenRouter: https://openrouter.ai/docs
