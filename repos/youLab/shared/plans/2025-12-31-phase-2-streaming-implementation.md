# Phase 2: Full Streaming with Thinking Implementation Plan

## Overview

Implement full streaming with thinking indicators from OpenWebUI through LettaStarter HTTP Service to Letta agents. This eliminates the current 10-30 second wait for responses by streaming content in real-time and showing "thinking" indicators when the agent is reasoning.

## Current State Analysis

### What Exists Now

1. **HTTP Service** (`server/main.py:149-204`):
   - `/chat` endpoint is synchronous
   - Calls `manager.send_message()` which blocks until complete response
   - Returns full `ChatResponse` with all text

2. **AgentManager** (`server/agents.py:158-165`):
   - `send_message()` uses `client.send_message()` (non-streaming)
   - `_extract_response()` processes complete response

3. **Pipe** (`pipelines/letta_pipe.py:121-188`):
   - Sync `def pipe()` method
   - Uses `httpx.Client` (sync)
   - Returns full string response

### Key Constraints Discovered

- OpenWebUI Pipes support async with `__event_emitter__` callback
- Letta SDK has `client.agents.messages.stream()` returning typed stream
- Pipes have NO hot reload - must restart OpenWebUI or re-upload
- FastAPI needs `StreamingResponse` with async generator for SSE

## Desired End State

### Architecture Flow
```
Browser ← SSE ← OpenWebUI ← event_emitter ← Pipe ← SSE ← HTTP Service ← SDK Stream ← Letta
```

### User Experience
- Student sends message
- Sees "Thinking..." indicator immediately (from `ReasoningMessage`)
- Sees tool usage indicators (from `ToolCallMessage`)
- Response streams word-by-word (from `AssistantMessage`)
- No long waits

### Success Criteria
- [ ] Streaming endpoint works with curl: `curl -N http://localhost:8100/chat/stream`
- [ ] Pipe streams to OpenWebUI event emitter
- [ ] "Thinking" indicators appear in UI
- [ ] Response text streams incrementally
- [ ] Existing `/chat` endpoint still works (backwards compatible)

## What We're NOT Doing

- Token-level streaming (word-by-word is sufficient for MVP)
- Reconnection logic (simple error handling is fine)
- Background processing (sync-ish streaming is acceptable)
- Unit tests for streaming (integration test manually)
- OpenWebUI native dev setup (user prefers their existing Docker setup)

## Implementation Approach

We'll implement in 3 phases:
1. Add streaming endpoint to HTTP service
2. Add streaming method to AgentManager
3. Convert Pipe to async with event emitter

Each phase is independently testable.

---

## Phase 1: Add Dependency and Streaming Schemas

### Overview
Add `httpx-sse` dependency and define schemas for the streaming endpoint.

### Changes Required

#### 1. Add Dependency
**File**: `pyproject.toml`
**Changes**: Add `httpx-sse` to dependencies

```toml
dependencies = [
    # ... existing ...
    "httpx-sse>=0.4.0",
]
```

#### 2. Add Streaming Schemas
**File**: `src/letta_starter/server/schemas.py`
**Changes**: Add streaming request schema (same as ChatRequest for now)

```python
class StreamChatRequest(BaseModel):
    """Request to send a message with streaming response."""

    agent_id: str = Field(..., description="Letta agent ID")
    message: str = Field(..., description="User message content")
    chat_id: str | None = Field(default=None, description="OpenWebUI chat ID")
    chat_title: str | None = Field(default=None, description="OpenWebUI chat title")
    enable_thinking: bool = Field(default=True, description="Include reasoning in stream")
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Dependency installs: `uv sync`

---

## Phase 2: Add Streaming to AgentManager

### Overview
Add `stream_message()` method to AgentManager that yields SSE-formatted events.

### Changes Required

#### 1. Add Streaming Method
**File**: `src/letta_starter/server/agents.py`
**Changes**: Add async generator method

```python
from collections.abc import AsyncGenerator
import json

async def stream_message(
    self,
    agent_id: str,
    message: str,
    enable_thinking: bool = True,
) -> AsyncGenerator[str, None]:
    """Stream a message to an agent. Yields SSE-formatted events."""
    try:
        # Use the streaming API
        with self.client.agents.messages.stream(
            agent_id=agent_id,
            input=message,
            enable_thinking="true" if enable_thinking else "false",
            stream_tokens=False,  # Don't need token-by-token
            include_pings=True,   # Keep connection alive
        ) as stream:
            for chunk in stream:
                event = self._chunk_to_sse_event(chunk)
                if event:
                    yield event
    except Exception as e:
        # Yield error event
        yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"

def _chunk_to_sse_event(self, chunk: Any) -> str | None:
    """Convert Letta streaming chunk to SSE event string."""
    import json

    msg_type = getattr(chunk, "message_type", None)

    if msg_type == "reasoning_message":
        return f"data: {json.dumps({'type': 'status', 'content': 'Thinking...', 'reasoning': chunk.reasoning})}\n\n"

    elif msg_type == "tool_call_message":
        tool_name = chunk.tool_call.name if hasattr(chunk.tool_call, "name") else "tool"
        return f"data: {json.dumps({'type': 'status', 'content': f'Using {tool_name}...'})}\n\n"

    elif msg_type == "assistant_message":
        content = chunk.content if isinstance(chunk.content, str) else str(chunk.content)
        return f"data: {json.dumps({'type': 'message', 'content': content})}\n\n"

    elif msg_type == "stop_reason":
        return f"data: {json.dumps({'type': 'done'})}\n\n"

    elif msg_type == "ping":
        return ": keepalive\n\n"  # SSE comment for keepalive

    elif msg_type == "error_message":
        return f"data: {json.dumps({'type': 'error', 'message': chunk.message})}\n\n"

    # Ignore other message types
    return None
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`

---

## Phase 3: Add Streaming Endpoint

### Overview
Add `/chat/stream` endpoint that returns `StreamingResponse` with SSE.

### Changes Required

#### 1. Add Streaming Endpoint
**File**: `src/letta_starter/server/main.py`
**Changes**: Add streaming endpoint after existing `/chat` endpoint

```python
from fastapi.responses import StreamingResponse
from letta_starter.server.schemas import StreamChatRequest

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

    async def event_generator() -> AsyncIterator[str]:
        """Generate SSE events from agent stream."""
        async for event in manager.stream_message(
            agent_id=request.agent_id,
            message=request.message,
            enable_thinking=request.enable_thinking,
        ):
            yield event

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )
```

#### 2. Add Import
**File**: `src/letta_starter/server/main.py`
**Changes**: Update imports at top of file

```python
from collections.abc import AsyncIterator
from fastapi.responses import StreamingResponse
from letta_starter.server.schemas import StreamChatRequest
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Server starts: `uv run letta-server &`

#### Manual Verification:
- [ ] Test with curl (requires Letta server running):
  ```bash
  # Start Letta server first: letta server
  # Start HTTP service: uv run letta-server
  # Create a test agent if needed, then:
  curl -N -X POST http://localhost:8100/chat/stream \
    -H "Content-Type: application/json" \
    -d '{"agent_id": "<agent_id>", "message": "Hello!", "enable_thinking": true}'
  ```
- [ ] Verify SSE events stream with proper format: `data: {...}\n\n`
- [ ] Verify "Thinking..." events appear before response
- [ ] Verify response content streams

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the streaming endpoint works with curl before proceeding.

---

## Phase 4: Convert Pipe to Async Streaming

### Overview
Update the OpenWebUI Pipe to use async and `__event_emitter__` for streaming.

### Changes Required

#### 1. Update Pipe Method Signature and Implementation
**File**: `src/letta_starter/pipelines/letta_pipe.py`
**Changes**: Convert to async with event emitter

```python
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
            description="Agent type to use (tutor, etc.)",
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
        """Initialize the pipeline."""
        self.name = "YouLab Tutor"
        self.valves = self.Valves()

    async def on_startup(self) -> None:
        """Called when the pipeline starts."""
        if self.valves.ENABLE_LOGGING:
            print(f"YouLab Pipeline started. Service URL: {self.valves.LETTA_SERVICE_URL}")

    async def on_shutdown(self) -> None:
        """Called when the pipeline stops."""
        if self.valves.ENABLE_LOGGING:
            print("YouLab Pipeline stopped")

    async def on_valves_updated(self) -> None:
        """Called when valves are updated via the UI."""
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
        """Ensure agent exists for user, create if needed. Returns agent_id."""
        try:
            response = await client.get(
                f"{self.valves.LETTA_SERVICE_URL}/agents",
                params={"user_id": user_id},
            )
            if response.status_code == 200:
                agents = response.json().get("agents", [])
                for agent in agents:
                    if agent.get("agent_type") == self.valves.AGENT_TYPE:
                        return agent.get("agent_id")
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to check existing agents: {e}")

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
            return None
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
        """
        Process a message through the YouLab tutor with streaming.

        Args:
            body: Request body containing messages
            __user__: User information from OpenWebUI
            __metadata__: Request metadata (chat_id, message_id, etc.)
            __event_emitter__: Async callback for streaming events

        Returns:
            Empty string (content is streamed via event_emitter)

        """
        # Extract user message from body
        messages = body.get("messages", [])
        user_message = messages[-1].get("content", "") if messages else ""

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

        # Extract chat context
        chat_id = __metadata__.get("chat_id") if __metadata__ else None
        chat_title = self._get_chat_title(chat_id)

        if self.valves.ENABLE_LOGGING:
            print(f"YouLab: user={user_id}, chat={chat_id}, title={chat_title}")

        try:
            async with httpx.AsyncClient(timeout=120.0) as client:
                # Ensure agent exists
                agent_id = await self._ensure_agent_exists(client, user_id, user_name)
                if not agent_id:
                    if __event_emitter__:
                        await __event_emitter__({
                            "type": "message",
                            "data": {"content": "Error: Could not create or find your tutor agent."}
                        })
                    return ""

                # Stream response
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
                    "data": {"content": "Error: Request timed out. Please try again."}
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
        event_emitter: Callable[[dict[str, Any]], Awaitable[None]] | None,
    ) -> None:
        """Handle an SSE event from the HTTP service."""
        import json

        if not event_emitter:
            return

        try:
            event = json.loads(data)
            event_type = event.get("type")

            if event_type == "status":
                # Show status indicator (thinking, tool use)
                await event_emitter({
                    "type": "status",
                    "data": {
                        "description": event.get("content", "Processing..."),
                        "done": False,
                    }
                })

            elif event_type == "message":
                # Stream message content
                await event_emitter({
                    "type": "message",
                    "data": {"content": event.get("content", "")}
                })

            elif event_type == "done":
                # Mark complete
                await event_emitter({
                    "type": "status",
                    "data": {"description": "Complete", "done": True}
                })

            elif event_type == "error":
                await event_emitter({
                    "type": "message",
                    "data": {"content": f"Error: {event.get('message', 'Unknown error')}"}
                })

        except json.JSONDecodeError:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to parse SSE event: {data}")
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Upload updated Pipe to OpenWebUI (Admin > Functions > Pipes)
- [ ] Restart OpenWebUI if needed (Pipes have no hot reload)
- [ ] Test chat in OpenWebUI
- [ ] Verify "Thinking..." indicator appears
- [ ] Verify response streams word-by-word
- [ ] Verify no regressions in error handling

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the full end-to-end streaming works in OpenWebUI.

---

## Testing Strategy

### Integration Tests (Curl)

```bash
# Test streaming endpoint
curl -N -X POST http://localhost:8100/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "<id>", "message": "Hello"}'

# Expected output (SSE format):
# data: {"type": "status", "content": "Thinking..."}
# data: {"type": "message", "content": "Hello! I'm your tutor..."}
# data: {"type": "done"}
```

### Manual Testing Steps

1. **Start services**:
   ```bash
   letta server                    # Terminal 1
   uv run letta-server            # Terminal 2
   docker compose up open-webui   # Terminal 3 (if using Docker)
   ```

2. **Create test agent**:
   ```bash
   curl -X POST http://localhost:8100/agents \
     -H "Content-Type: application/json" \
     -d '{"user_id": "test-user", "agent_type": "tutor"}'
   ```

3. **Test streaming endpoint**:
   ```bash
   curl -N -X POST http://localhost:8100/chat/stream \
     -H "Content-Type: application/json" \
     -d '{"agent_id": "<from step 2>", "message": "Hello!", "enable_thinking": true}'
   ```

4. **Test in OpenWebUI**:
   - Log in as test user
   - Start new chat
   - Send message
   - Verify streaming and thinking indicators

## Performance Considerations

- **Keepalive**: SSE pings prevent connection timeout during long responses
- **Buffering**: `X-Accel-Buffering: no` header prevents nginx from buffering SSE
- **Timeout**: 120 second timeout allows for complex agent responses
- **Memory**: Streaming avoids loading full response into memory

## Migration Notes

- Existing `/chat` endpoint is preserved for backwards compatibility
- No database changes required
- Pipe upload replaces existing Pipe (no versioning needed)

## References

- Research: `thoughts/shared/research/2025-12-31-phase-2-streaming-research.md`
- Handoff: `thoughts/shared/handoffs/general/2025-12-31_15-11-36_phase-2-streaming-research.md`
- Letta SDK: `.venv/lib/python3.12/site-packages/letta_client/resources/agents/messages.py`
- OpenWebUI Pipes: `thoughts/global/shared/reference/open-web-ui-pipes.md`
