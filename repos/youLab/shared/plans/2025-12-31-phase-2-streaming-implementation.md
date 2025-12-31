# Phase 2: Full Streaming with Thinking Implementation Plan

## Overview

Implement streaming responses with thinking indicators from OpenWebUI through LettaStarter HTTP Service to Letta agents. Students will see real-time "Thinking..." indicators and incremental response text instead of 10-30 second waits.

## Current State

1. **HTTP Service** (`server/main.py:149-204`): Sync `/chat` endpoint returns full response
2. **AgentManager** (`server/agents.py:158-165`): Sync `send_message()` using sync `Letta` client
3. **Pipe** (`pipelines/letta_pipe.py:121-188`): **BUG** - wrong method signature, sync, returns full string

## Desired End State

```
Browser ← SSE ← OpenWebUI ← event_emitter ← Pipe ← SSE ← HTTP Service ← Stream ← Letta
```

- Student sends message
- "Thinking..." indicator appears immediately
- Response streams incrementally
- No long waits

## Key Design Decision: Sync Streaming

We're using **sync streaming** throughout the server:
- Existing codebase is sync
- Uvicorn handles ~40 concurrent streaming requests (plenty for early stage)
- No need for `AsyncLetta` - simpler architecture
- FastAPI's `StreamingResponse` accepts sync iterators

**Note**: The Pipe must be async because OpenWebUI's `__event_emitter__` is async. This is an OpenWebUI requirement, not a complexity we're adding.

## What We're NOT Doing

- Token-level streaming (message-level is fine)
- `AsyncLetta` client (unnecessary complexity)
- Converting existing sync methods to async
- Unit tests for streaming (manual integration testing)

---

## Phase 1: Add Dependency

### Overview
Add `httpx-sse` for Pipe SSE consumption.

### Changes

**File**: `pyproject.toml`

```toml
dependencies = [
    # ... existing ...
    "httpx-sse>=0.4.0",
]
```

### Success Criteria
- [ ] `uv sync` completes
- [ ] `make lint-fix` passes

---

## Phase 2: Add Streaming to AgentManager

### Overview
Add sync `stream_message()` method that yields SSE-formatted events.

### Changes

**File**: `src/letta_starter/server/agents.py`

Add imports at top:
```python
import json
from collections.abc import Iterator
```

Add method after `send_message()`:
```python
def stream_message(
    self,
    agent_id: str,
    message: str,
    enable_thinking: bool = True,
) -> Iterator[str]:
    """Stream a message to an agent. Yields SSE-formatted events."""
    try:
        with self.client.agents.messages.stream(
            agent_id=agent_id,
            input=message,
            enable_thinking="true" if enable_thinking else "false",
            stream_tokens=False,
            include_pings=True,
        ) as stream:
            for chunk in stream:
                event = self._chunk_to_sse_event(chunk)
                if event:
                    yield event
    except Exception as e:
        log.exception("stream_message_failed", agent_id=agent_id, error=str(e))
        yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"

def _chunk_to_sse_event(self, chunk: Any) -> str | None:
    """Convert Letta streaming chunk to SSE event string."""
    msg_type = getattr(chunk, "message_type", None)

    if msg_type == "reasoning_message":
        reasoning = getattr(chunk, "reasoning", "")
        return f"data: {json.dumps({'type': 'status', 'content': 'Thinking...', 'reasoning': reasoning})}\n\n"

    elif msg_type == "tool_call_message":
        tool_call = getattr(chunk, "tool_call", None)
        tool_name = getattr(tool_call, "name", "tool") if tool_call else "tool"
        return f"data: {json.dumps({'type': 'status', 'content': f'Using {tool_name}...'})}\n\n"

    elif msg_type == "assistant_message":
        content = getattr(chunk, "content", "")
        if not isinstance(content, str):
            content = str(content)
        return f"data: {json.dumps({'type': 'message', 'content': content})}\n\n"

    elif msg_type == "stop_reason":
        return f"data: {json.dumps({'type': 'done'})}\n\n"

    elif msg_type == "ping":
        return ": keepalive\n\n"

    elif msg_type == "error_message":
        error_msg = getattr(chunk, "message", "Unknown error")
        return f"data: {json.dumps({'type': 'error', 'message': error_msg})}\n\n"

    return None
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make check-agent` passes

---

## Phase 3: Add Streaming Endpoint

### Overview
Add `/chat/stream` endpoint returning `StreamingResponse`.

### Changes

**File**: `src/letta_starter/server/schemas.py`

Add after `ChatRequest`:
```python
class StreamChatRequest(BaseModel):
    """Request to send a message with streaming response."""

    agent_id: str = Field(..., description="Letta agent ID")
    message: str = Field(..., description="User message content")
    chat_id: str | None = Field(default=None, description="OpenWebUI chat ID")
    chat_title: str | None = Field(default=None, description="OpenWebUI chat title")
    enable_thinking: bool = Field(default=True, description="Include reasoning in stream")
```

**File**: `src/letta_starter/server/main.py`

Add import:
```python
from fastapi.responses import StreamingResponse

from letta_starter.server.schemas import (
    # ... existing ...
    StreamChatRequest,
)
```

Add endpoint after `/chat`:
```python
@app.post("/chat/stream")
async def chat_stream(request: StreamChatRequest) -> StreamingResponse:
    """Send a message to an agent with streaming response (SSE)."""
    manager = get_agent_manager()

    # Verify agent exists
    info = manager.get_agent_info(request.agent_id)
    if info is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Agent not found: {request.agent_id}",
        )

    return StreamingResponse(
        manager.stream_message(
            agent_id=request.agent_id,
            message=request.message,
            enable_thinking=request.enable_thinking,
        ),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

### Success Criteria

#### Automated:
- [ ] `make lint-fix` passes
- [ ] `make check-agent` passes

#### Manual:
```bash
# Terminal 1
letta server

# Terminal 2
uv run letta-server

# Terminal 3 - create agent
curl -X POST http://localhost:8100/agents \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test-user", "agent_type": "tutor"}'

# Note the agent_id, then test streaming
curl -N -X POST http://localhost:8100/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "<agent_id>", "message": "Hello!"}'
```

Expected output:
```
data: {"type": "status", "content": "Thinking...", "reasoning": "..."}

data: {"type": "message", "content": "Hello! I'm your tutor..."}

data: {"type": "done"}
```

**Pause here** for manual verification before Phase 4.

---

## Phase 4: Fix and Update Pipe

### Overview
Fix incorrect pipe signature and add streaming support.

**Bug being fixed**: Current signature uses `user_message, model_id, messages` parameters that OpenWebUI doesn't inject. Correct signature uses `body` containing messages.

**Why async**: OpenWebUI's `__event_emitter__` is an async function, so `pipe()` must be async to call it. This is an OpenWebUI requirement.

### Changes

**File**: `src/letta_starter/pipelines/letta_pipe.py`

Replace entire file:
```python
"""
OpenWebUI Pipeline for YouLab.

Forwards requests to the LettaStarter HTTP service with streaming support.
"""

import json
from collections.abc import Awaitable, Callable
from typing import Any

import httpx
from httpx_sse import aconnect_sse
from pydantic import BaseModel, Field


class Pipeline:
    """OpenWebUI Pipeline for YouLab tutoring with streaming."""

    class Valves(BaseModel):
        """Configuration options exposed in OpenWebUI admin."""

        LETTA_SERVICE_URL: str = Field(
            default="http://localhost:8100",
            description="URL of the LettaStarter HTTP service",
        )
        AGENT_TYPE: str = Field(
            default="tutor",
            description="Agent type to use",
        )
        ENABLE_LOGGING: bool = Field(
            default=True,
            description="Enable detailed logging",
        )
        ENABLE_THINKING: bool = Field(
            default=True,
            description="Show thinking indicators",
        )

    def __init__(self) -> None:
        self.name = "YouLab Tutor"
        self.valves = self.Valves()

    async def on_startup(self) -> None:
        if self.valves.ENABLE_LOGGING:
            print(f"YouLab Pipeline started. Service: {self.valves.LETTA_SERVICE_URL}")

    async def on_shutdown(self) -> None:
        if self.valves.ENABLE_LOGGING:
            print("YouLab Pipeline stopped")

    async def on_valves_updated(self) -> None:
        if self.valves.ENABLE_LOGGING:
            print("YouLab Pipeline valves updated")

    def _get_chat_title(self, chat_id: str | None) -> str | None:
        """Get chat title from OpenWebUI database."""
        if not chat_id or chat_id.startswith("local:"):
            return None
        try:
            from open_webui.models.chats import Chats
            chat = Chats.get_chat_by_id(chat_id)
            return chat.title if chat else None
        except ImportError:
            return None
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to get chat title: {e}")
            return None

    async def _ensure_agent_exists(
        self,
        client: httpx.AsyncClient,
        user_id: str,
        user_name: str | None = None,
    ) -> str | None:
        """Ensure agent exists for user, create if needed."""
        try:
            response = await client.get(
                f"{self.valves.LETTA_SERVICE_URL}/agents",
                params={"user_id": user_id},
            )
            if response.status_code == 200:
                for agent in response.json().get("agents", []):
                    if agent.get("agent_type") == self.valves.AGENT_TYPE:
                        return agent.get("agent_id")
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to check agents: {e}")

        # Create new agent
        try:
            response = await client.post(
                f"{self.valves.LETTA_SERVICE_URL}/agents",
                json={
                    "user_id": user_id,
                    "agent_type": self.valves.AGENT_TYPE,
                    "user_name": user_name,
                },
            )
            if response.status_code == 201:
                return response.json().get("agent_id")
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to create agent: {response.text}")
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to create agent: {e}")
        return None

    async def pipe(
        self,
        body: dict[str, Any],
        __user__: dict[str, Any] | None = None,
        __metadata__: dict[str, Any] | None = None,
        __event_emitter__: Callable[[dict[str, Any]], Awaitable[None]] | None = None,
    ) -> str:
        """Process a message with streaming."""
        # Extract message from body
        messages = body.get("messages", [])
        user_message = messages[-1].get("content", "") if messages else ""

        if not user_message:
            return "Error: No message provided."

        # Extract user info
        user_id = __user__.get("id") if __user__ else None
        user_name = __user__.get("name") if __user__ else None

        if not user_id:
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": "Error: Could not identify user. Please log in."}
                })
            return ""

        # Get chat context
        chat_id = __metadata__.get("chat_id") if __metadata__ else None
        chat_title = self._get_chat_title(chat_id)

        if self.valves.ENABLE_LOGGING:
            print(f"YouLab: user={user_id}, chat={chat_id}")

        try:
            async with httpx.AsyncClient(timeout=120.0) as client:
                agent_id = await self._ensure_agent_exists(client, user_id, user_name)
                if not agent_id:
                    if __event_emitter__:
                        await __event_emitter__({
                            "type": "message",
                            "data": {"content": "Error: Could not find or create tutor agent."}
                        })
                    return ""

                # Stream from HTTP service
                async with aconnect_sse(
                    client,
                    "POST",
                    f"{self.valves.LETTA_SERVICE_URL}/chat/stream",
                    json={
                        "agent_id": agent_id,
                        "message": user_message,
                        "chat_id": chat_id,
                        "chat_title": chat_title,
                        "enable_thinking": self.valves.ENABLE_THINKING,
                    },
                ) as event_source:
                    async for sse in event_source.aiter_sse():
                        if sse.data:
                            await self._handle_sse_event(sse.data, __event_emitter__)

        except httpx.TimeoutException:
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": "Error: Request timed out."}
                })
        except httpx.ConnectError:
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": "Error: Could not connect to tutor service."}
                })
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"YouLab error: {e}")
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": f"Error: {e!s}"}
                })

        return ""

    async def _handle_sse_event(
        self,
        data: str,
        emitter: Callable[[dict[str, Any]], Awaitable[None]] | None,
    ) -> None:
        """Handle SSE event from HTTP service."""
        if not emitter:
            return

        try:
            event = json.loads(data)
            event_type = event.get("type")

            if event_type == "status":
                await emitter({
                    "type": "status",
                    "data": {
                        "description": event.get("content", "Processing..."),
                        "done": False,
                    }
                })
            elif event_type == "message":
                await emitter({
                    "type": "message",
                    "data": {"content": event.get("content", "")}
                })
            elif event_type == "done":
                await emitter({
                    "type": "status",
                    "data": {"description": "Complete", "done": True}
                })
            elif event_type == "error":
                await emitter({
                    "type": "message",
                    "data": {"content": f"Error: {event.get('message', 'Unknown')}"}
                })

        except json.JSONDecodeError:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to parse SSE: {data}")
```

### Success Criteria

#### Automated:
- [ ] `make lint-fix` passes
- [ ] `make check-agent` passes

#### Manual:
- [ ] Upload Pipe to OpenWebUI (Admin > Functions > Pipes)
- [ ] Restart OpenWebUI (no hot reload for Pipes)
- [ ] Send message in chat
- [ ] Verify "Thinking..." indicator appears
- [ ] Verify response streams incrementally

---

## Phase 5: End-to-End Testing

### Test Checklist

- [ ] Streaming works with curl
- [ ] Streaming works in OpenWebUI
- [ ] Thinking indicators visible
- [ ] Errors handled gracefully
- [ ] Original `/chat` endpoint still works
- [ ] Agent creation still works

### Error Scenarios to Test

1. Invalid agent_id → error message
2. Letta server down → connection error
3. Long response → streams without timeout

---

## Summary

| Phase | What | Files |
|-------|------|-------|
| 1 | Add `httpx-sse` dependency | `pyproject.toml` |
| 2 | Add `stream_message()` method | `server/agents.py` |
| 3 | Add `/chat/stream` endpoint | `server/schemas.py`, `server/main.py` |
| 4 | Fix and update Pipe | `pipelines/letta_pipe.py` |
| 5 | End-to-end testing | Manual |

## References

- Letta SDK Stream: `.venv/lib/python3.12/site-packages/letta_client/_streaming.py:21-101`
- OpenWebUI Pipe signature: `OpenWebUI/open-webui/backend/open_webui/functions.py:206-209`
- OpenWebUI event emitter: `OpenWebUI/open-webui/backend/open_webui/socket/main.py:693-810`
