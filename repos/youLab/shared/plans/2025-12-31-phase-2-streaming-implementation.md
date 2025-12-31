# Phase 2: Full Streaming with Thinking Implementation Plan

## Overview

Implement full streaming with thinking indicators from OpenWebUI through LettaStarter HTTP Service to Letta agents. This eliminates the current 10-30 second wait for responses by streaming content in real-time and showing "thinking" indicators when the agent is reasoning.

## Current State Analysis

### What Exists Now

1. **HTTP Service** (`server/main.py:149-204`):
   - `/chat` endpoint is synchronous
   - Calls `manager.send_message()` which blocks until complete response
   - Returns full `ChatResponse` with all text

2. **AgentManager** (`server/agents.py`):
   - Uses sync `Letta` client (not `AsyncLetta`)
   - `send_message()` uses `client.send_message()` (non-streaming)
   - All methods except `rebuild_cache()` are synchronous

3. **Pipe** (`pipelines/letta_pipe.py:121-188`):
   - **BUG**: Incorrect method signature - uses `user_message, model_id, messages` parameters that OpenWebUI doesn't inject
   - Sync `def pipe()` method
   - Uses `httpx.Client` (sync)
   - Returns full string response

### Critical Findings from Verification

| Component | Issue | Impact |
|-----------|-------|--------|
| **Letta SDK** | Sync `Stream` class doesn't support `async for` - must use `AsyncLetta` with `AsyncStream` | Must refactor AgentManager to use async client |
| **Pipe Signature** | Current signature is **wrong** - OpenWebUI only injects `body`, `__user__`, `__metadata__`, `__event_emitter__` | Existing pipe may be broken or working by accident |
| **AgentManager** | Uses sync `Letta` client throughout | Cannot simply add async method - need async client |

### Letta SDK Architecture (Verified)

```
Sync Path (CANNOT use with async for):
  Letta → AgentsResource → MessagesResource.stream() → Stream[T]

Async Path (REQUIRED for FastAPI streaming):
  AsyncLetta → AsyncAgentsResource → AsyncMessagesResource.stream() → AsyncStream[T]
```

**Key imports**:
```python
from letta_client import AsyncLetta  # Async client
from letta_client import Letta       # Sync client (current)
```

### OpenWebUI Pipe Signature (Verified)

**Correct signature** (from `OpenWebUI/open-webui/backend/open_webui/functions.py:206-209`):
```python
async def pipe(
    self,
    body: dict,                    # Contains messages, model, etc.
    __user__: dict | None = None,
    __metadata__: dict | None = None,
    __event_emitter__: Callable[[dict], Awaitable[None]] | None = None,
) -> str
```

**Current WRONG signature** in `letta_pipe.py:121`:
```python
def pipe(self, user_message: str, model_id: str, messages: list, body: dict, ...)
```

## Desired End State

### Architecture Flow
```
Browser ← SSE ← OpenWebUI ← event_emitter ← Pipe ← SSE ← HTTP Service ← AsyncStream ← Letta
```

### User Experience
- Student sends message
- Sees "Thinking..." indicator immediately (from `ReasoningMessage`)
- Sees tool usage indicators (from `ToolCallMessage`)
- Response streams incrementally (from `AssistantMessage`)
- No long waits

### Success Criteria
- [ ] Streaming endpoint works with curl: `curl -N http://localhost:8100/chat/stream`
- [ ] Pipe streams to OpenWebUI event emitter
- [ ] "Thinking" indicators appear in UI
- [ ] Response text streams incrementally
- [ ] Existing `/chat` endpoint still works (backwards compatible)

## What We're NOT Doing

- Token-level streaming (message-level is sufficient for MVP)
- Reconnection logic (simple error handling is fine)
- Converting ALL AgentManager methods to async (only add streaming)
- Unit tests for streaming (integration test manually)
- Full async refactor of AgentManager (keep sync methods for backwards compat)

## Implementation Approach

We'll implement in 5 phases:
1. Add dependencies
2. Add async Letta client support to AgentManager
3. Add streaming endpoint to HTTP service
4. Fix and convert Pipe to async with event emitter
5. Test end-to-end

Each phase is independently testable.

---

## Phase 1: Add Dependencies

### Overview
Add `httpx-sse` dependency for Pipe SSE consumption.

### Changes Required

#### 1. Add Dependency
**File**: `pyproject.toml`
**Changes**: Add `httpx-sse` to dependencies

```python
# Add after "httpx>=0.27.0"
"httpx-sse>=0.4.0",
```

### Success Criteria

#### Automated Verification:
- [ ] Dependency installs: `uv sync`
- [ ] Linting passes: `make lint-fix`

---

## Phase 2: Add Async Client Support to AgentManager

### Overview
Add `AsyncLetta` client and async streaming method to AgentManager while preserving existing sync methods for backwards compatibility.

### Changes Required

#### 1. Add Async Client Property
**File**: `src/letta_starter/server/agents.py`
**Changes**: Add async client alongside existing sync client

```python
from collections.abc import AsyncGenerator
import json
from typing import Any

import structlog
from letta_client import AsyncLetta, Letta

from letta_starter.agents.templates import templates

log = structlog.get_logger()


class AgentManager:
    """Manages Letta agents for YouLab users."""

    def __init__(self, letta_base_url: str) -> None:
        self.letta_base_url = letta_base_url
        self._client: Letta | None = None
        self._async_client: AsyncLetta | None = None
        # Cache: (user_id, agent_type) -> agent_id
        self._cache: dict[tuple[str, str], str] = {}

    @property
    def client(self) -> Letta:
        """Lazy-initialize sync Letta client."""
        if self._client is None:
            self._client = Letta(base_url=self.letta_base_url)
        return self._client

    @client.setter
    def client(self, value: Letta) -> None:
        """Set the Letta client (for testing)."""
        self._client = value

    @property
    def async_client(self) -> AsyncLetta:
        """Lazy-initialize async Letta client."""
        if self._async_client is None:
            self._async_client = AsyncLetta(base_url=self.letta_base_url)
        return self._async_client

    @async_client.setter
    def async_client(self, value: AsyncLetta) -> None:
        """Set the async Letta client (for testing)."""
        self._async_client = value
```

#### 2. Add Streaming Method
**File**: `src/letta_starter/server/agents.py`
**Changes**: Add after existing `send_message()` method

```python
async def stream_message(
    self,
    agent_id: str,
    message: str,
    enable_thinking: bool = True,
) -> AsyncGenerator[str, None]:
    """Stream a message to an agent. Yields SSE-formatted events."""
    try:
        async with self.async_client.agents.messages.stream(
            agent_id=agent_id,
            input=message,
            enable_thinking="true" if enable_thinking else "false",
            stream_tokens=False,
            include_pings=True,
        ) as stream:
            async for chunk in stream:
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
        return f"data: {json.dumps({'type': 'status', 'content': 'Thinking...', 'reasoning': getattr(chunk, 'reasoning', '')})}\n\n"

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
        return f"data: {json.dumps({'type': 'error', 'message': getattr(chunk, 'message', 'Unknown error')})}\n\n"

    return None
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Tests pass: `make test-agent`

---

## Phase 3: Add Streaming Schemas and Endpoint

### Overview
Add `/chat/stream` endpoint that returns `StreamingResponse` with SSE.

### Changes Required

#### 1. Add Streaming Schema
**File**: `src/letta_starter/server/schemas.py`
**Changes**: Add after `ChatRequest` class

```python
class StreamChatRequest(BaseModel):
    """Request to send a message with streaming response."""

    agent_id: str = Field(..., description="Letta agent ID")
    message: str = Field(..., description="User message content")
    chat_id: str | None = Field(default=None, description="OpenWebUI chat ID")
    chat_title: str | None = Field(default=None, description="OpenWebUI chat title")
    enable_thinking: bool = Field(default=True, description="Include reasoning in stream")
```

#### 2. Add Streaming Endpoint
**File**: `src/letta_starter/server/main.py`
**Changes**: Add imports and endpoint after existing `/chat` endpoint

Add to imports:
```python
from fastapi.responses import StreamingResponse

from letta_starter.server.schemas import (
    # ... existing imports ...
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
            "X-Accel-Buffering": "no",
        },
    )
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Server starts: `uv run letta-server &`

#### Manual Verification:
- [ ] Test with curl (requires Letta server running):
  ```bash
  # Terminal 1: letta server
  # Terminal 2: uv run letta-server
  # Terminal 3:
  curl -X POST http://localhost:8100/agents \
    -H "Content-Type: application/json" \
    -d '{"user_id": "test-user", "agent_type": "tutor"}'
  # Note the agent_id, then:
  curl -N -X POST http://localhost:8100/chat/stream \
    -H "Content-Type: application/json" \
    -d '{"agent_id": "<agent_id>", "message": "Hello!", "enable_thinking": true}'
  ```
- [ ] Verify SSE events stream with format: `data: {...}\n\n`
- [ ] Verify "Thinking..." status events appear
- [ ] Verify message content streams

**Implementation Note**: Pause here for manual confirmation that the streaming endpoint works with curl before proceeding to Phase 4.

---

## Phase 4: Fix and Convert Pipe to Async Streaming

### Overview
Fix the incorrect pipe signature and convert to async with `__event_emitter__` for streaming.

**Note**: This is also a bug fix - the current pipe signature is incorrect and may not work properly with all OpenWebUI versions.

### Changes Required

#### 1. Complete Pipe Rewrite
**File**: `src/letta_starter/pipelines/letta_pipe.py`
**Changes**: Replace entire file

```python
"""
OpenWebUI Pipeline for YouLab.

This Pipe forwards requests to the LettaStarter HTTP service with streaming support.
It extracts user context from OpenWebUI and chat title from the database.
"""

import json
from collections.abc import Awaitable, Callable
from typing import Any

import httpx
from httpx_sse import aconnect_sse
from pydantic import BaseModel, Field


class Pipeline:
    """
    OpenWebUI Pipeline for YouLab tutoring with streaming.

    Forwards messages to the LettaStarter HTTP service and streams responses.
    Configure using Valves in the OpenWebUI admin panel.
    """

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
            description="Show thinking indicators in UI",
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
            body: Request body containing messages, model, etc.
            __user__: User information from OpenWebUI
            __metadata__: Request metadata (chat_id, message_id, etc.)
            __event_emitter__: Async callback for streaming events to UI

        Returns:
            Empty string (content is streamed via event_emitter)

        """
        # Extract user message from body
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

        # Extract chat context from metadata
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

                # Stream response from HTTP service
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
        except httpx.ConnectError:
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": "Error: Could not connect to tutor service. Please try again later."}
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


# For standalone testing
if __name__ == "__main__":
    import asyncio

    async def mock_emitter(event: dict[str, Any]) -> None:
        print(f"Event: {event}")

    async def test() -> None:
        pipeline = Pipeline()
        await pipeline.on_startup()

        # Simulate OpenWebUI request
        await pipeline.pipe(
            body={
                "messages": [{"role": "user", "content": "Hello! What can you help me with?"}],
                "model": "youlab",
            },
            __user__={"id": "test-user-123", "name": "Test User"},
            __metadata__={"chat_id": "local:test"},
            __event_emitter__=mock_emitter,
        )

        await pipeline.on_shutdown()

    asyncio.run(test())
```

### Success Criteria

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Upload updated Pipe to OpenWebUI (Admin > Functions > Pipes)
- [ ] Restart OpenWebUI (Pipes have no hot reload)
- [ ] Test chat in OpenWebUI
- [ ] Verify "Thinking..." indicator appears during processing
- [ ] Verify response streams incrementally (not all at once)
- [ ] Verify errors are displayed properly

**Implementation Note**: Pause here for manual confirmation that end-to-end streaming works in OpenWebUI.

---

## Phase 5: End-to-End Testing

### Overview
Comprehensive testing of the full streaming pipeline.

### Test Scenarios

#### 1. Basic Streaming Flow
```bash
# Start all services
letta server                    # Terminal 1
uv run letta-server            # Terminal 2

# Test HTTP service streaming
curl -N -X POST http://localhost:8100/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"agent_id": "<id>", "message": "Explain what you can help me with"}'
```

Expected output:
```
data: {"type": "status", "content": "Thinking...", "reasoning": "..."}

data: {"type": "message", "content": "I'm your college essay tutor..."}

data: {"type": "done"}

```

#### 2. OpenWebUI Integration
1. Log in to OpenWebUI
2. Select YouLab Tutor pipe
3. Send message: "Help me brainstorm essay topics"
4. Observe:
   - "Thinking..." status appears immediately
   - Response text streams incrementally
   - Final status shows "Complete"

#### 3. Error Handling
- Test with invalid agent_id → should show error message
- Test with Letta server down → should show connection error
- Test with very long response → should stream without timeout

### Success Criteria

#### All Must Pass:
- [ ] Streaming works end-to-end in curl
- [ ] Streaming works in OpenWebUI
- [ ] Thinking indicators visible in UI
- [ ] Errors handled gracefully
- [ ] Existing `/chat` endpoint still works (backwards compat)
- [ ] No regressions in agent creation

---

## Performance Considerations

- **Keepalive**: SSE pings (`message_type: "ping"`) prevent connection timeout
- **Buffering**: `X-Accel-Buffering: no` header prevents nginx buffering
- **Timeout**: 120 second timeout for long agent responses
- **Memory**: Streaming avoids loading full response into memory
- **Connection pooling**: `AsyncClient` should be reused where possible

## Migration Notes

- Existing `/chat` endpoint preserved for backwards compatibility
- Sync `AgentManager.send_message()` preserved for existing callers
- Pipe signature fix may affect any custom integrations using old signature
- No database changes required

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Letta SDK async API differs from documented | Verified against actual SDK source code |
| OpenWebUI event format changes | Verified against OpenWebUI source in local clone |
| httpx-sse incompatibility | Library is stable, well-maintained |
| Existing tests break | Keeping sync methods, only adding async |

## References

- Research: `thoughts/shared/research/2025-12-31-phase-2-streaming-research.md`
- Handoff: `thoughts/shared/handoffs/general/2025-12-31_15-11-36_phase-2-streaming-research.md`
- Letta SDK AsyncStream: `.venv/lib/python3.12/site-packages/letta_client/_streaming.py:103-183`
- OpenWebUI Pipe signature: `OpenWebUI/open-webui/backend/open_webui/functions.py:206-209`
- OpenWebUI event emitter: `OpenWebUI/open-webui/backend/open_webui/socket/main.py:693-810`
