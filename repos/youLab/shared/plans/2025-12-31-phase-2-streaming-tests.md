# Phase 2: Streaming Implementation Test Plan

## Overview

Comprehensive test suite for the Phase 2 streaming implementation. This plan parallels `2025-12-31-phase-2-streaming-implementation.md` and provides automated tests for each component.

**Sister Plan**: `2025-12-31-phase-2-streaming-implementation.md`

## Test Strategy

### What We're Testing

| Component | Location | Test Focus |
|-----------|----------|------------|
| `stream_message()` | `server/agents.py` | SSE event generation, chunk parsing, error handling |
| `_chunk_to_sse_event()` | `server/agents.py` | Message type mapping, edge cases |
| `/chat/stream` endpoint | `server/main.py` | HTTP streaming, headers, error responses |
| `StreamChatRequest` | `server/schemas.py` | Pydantic validation |
| Pipe streaming | `pipelines/letta_pipe.py` | SSE consumption, event emission, error handling |

### What We're NOT Testing

- Integration with real Letta server (requires running server)
- OpenWebUI Pipe registration (requires OpenWebUI)
- Browser SSE rendering (manual testing)
- Token-level streaming latency (not implementing token streaming)

### Test Patterns Used

Following existing patterns from `tests/test_server/`:
- Class-based test organization (`TestStreamMessage`, `TestChatStreamEndpoint`)
- Fixtures in `conftest.py` for shared mocks
- `@patch` for external dependencies
- `MagicMock` for Letta client and streaming responses
- `pytest.mark.asyncio` for async tests
- FastAPI `TestClient` for endpoint tests

---

## Phase 1: Schema Tests

### Overview
Test `StreamChatRequest` Pydantic model validation.

### File: `tests/test_server/test_schemas.py`

Add after existing schema tests:

```python
class TestStreamChatRequest:
    """Tests for StreamChatRequest schema."""

    def test_minimal_request(self):
        """StreamChatRequest accepts minimal required fields."""
        from letta_starter.server.schemas import StreamChatRequest

        request = StreamChatRequest(agent_id="agent-123", message="Hello")
        assert request.agent_id == "agent-123"
        assert request.message == "Hello"
        assert request.chat_id is None
        assert request.chat_title is None
        assert request.enable_thinking is True

    def test_full_request(self):
        """StreamChatRequest accepts all optional fields."""
        from letta_starter.server.schemas import StreamChatRequest

        request = StreamChatRequest(
            agent_id="agent-123",
            message="Hello",
            chat_id="chat-456",
            chat_title="My Chat",
            enable_thinking=False,
        )
        assert request.chat_id == "chat-456"
        assert request.chat_title == "My Chat"
        assert request.enable_thinking is False

    def test_missing_agent_id(self):
        """StreamChatRequest requires agent_id."""
        from pydantic import ValidationError

        from letta_starter.server.schemas import StreamChatRequest

        with pytest.raises(ValidationError):
            StreamChatRequest(message="Hello")  # type: ignore[call-arg]

    def test_missing_message(self):
        """StreamChatRequest requires message."""
        from pydantic import ValidationError

        from letta_starter.server.schemas import StreamChatRequest

        with pytest.raises(ValidationError):
            StreamChatRequest(agent_id="agent-123")  # type: ignore[call-arg]
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make test-agent` passes

---

## Phase 2: Chunk-to-SSE Conversion Tests

### Overview
Test `_chunk_to_sse_event()` method for all Letta message types.

### File: `tests/test_server/test_agents.py`

Add new test class:

```python
class TestChunkToSSEEvent:
    """Tests for AgentManager._chunk_to_sse_event()."""

    def test_reasoning_message(self, mock_agent_manager):
        """reasoning_message produces status event with thinking indicator."""
        chunk = MagicMock()
        chunk.message_type = "reasoning_message"
        chunk.reasoning = "Let me think about this..."

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        assert result is not None
        assert result.startswith("data: ")
        assert result.endswith("\n\n")

        import json
        data = json.loads(result[6:-2])
        assert data["type"] == "status"
        assert data["content"] == "Thinking..."
        assert data["reasoning"] == "Let me think about this..."

    def test_tool_call_message(self, mock_agent_manager):
        """tool_call_message produces status event with tool name."""
        chunk = MagicMock()
        chunk.message_type = "tool_call_message"
        chunk.tool_call = MagicMock()
        chunk.tool_call.name = "search_memory"

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        import json
        data = json.loads(result[6:-2])
        assert data["type"] == "status"
        assert data["content"] == "Using search_memory..."

    def test_tool_call_message_no_tool_call(self, mock_agent_manager):
        """tool_call_message handles missing tool_call gracefully."""
        chunk = MagicMock()
        chunk.message_type = "tool_call_message"
        chunk.tool_call = None

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        import json
        data = json.loads(result[6:-2])
        assert data["content"] == "Using tool..."

    def test_assistant_message_string(self, mock_agent_manager):
        """assistant_message with string content produces message event."""
        chunk = MagicMock()
        chunk.message_type = "assistant_message"
        chunk.content = "Hello! I'm your tutor."

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        import json
        data = json.loads(result[6:-2])
        assert data["type"] == "message"
        assert data["content"] == "Hello! I'm your tutor."

    def test_assistant_message_non_string(self, mock_agent_manager):
        """assistant_message with non-string content converts to string."""
        chunk = MagicMock()
        chunk.message_type = "assistant_message"
        chunk.content = ["Part 1", "Part 2"]  # List content

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        import json
        data = json.loads(result[6:-2])
        assert data["type"] == "message"
        assert "Part 1" in data["content"]

    def test_stop_reason(self, mock_agent_manager):
        """stop_reason produces done event."""
        chunk = MagicMock()
        chunk.message_type = "stop_reason"

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        import json
        data = json.loads(result[6:-2])
        assert data["type"] == "done"

    def test_ping(self, mock_agent_manager):
        """ping produces SSE comment for keep-alive."""
        chunk = MagicMock()
        chunk.message_type = "ping"

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        assert result == ": keepalive\n\n"

    def test_error_message(self, mock_agent_manager):
        """error_message produces error event."""
        chunk = MagicMock()
        chunk.message_type = "error_message"
        chunk.message = "Something went wrong"

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001

        import json
        data = json.loads(result[6:-2])
        assert data["type"] == "error"
        assert data["message"] == "Something went wrong"

    def test_ignored_message_types(self, mock_agent_manager):
        """Internal message types return None (ignored)."""
        ignored_types = [
            "tool_return_message",
            "usage_statistics",
            "hidden_reasoning_message",
            "system_message",
            "user_message",
        ]

        for msg_type in ignored_types:
            chunk = MagicMock()
            chunk.message_type = msg_type

            result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001
            assert result is None, f"{msg_type} should return None"

    def test_unknown_message_type(self, mock_agent_manager):
        """Unknown message types return None."""
        chunk = MagicMock()
        chunk.message_type = "future_message_type"

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001
        assert result is None

    def test_missing_message_type(self, mock_agent_manager):
        """Chunk without message_type returns None."""
        chunk = MagicMock(spec=[])  # No message_type attribute

        result = mock_agent_manager._chunk_to_sse_event(chunk)  # noqa: SLF001
        assert result is None
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make test-agent` passes

---

## Phase 3: Stream Message Tests

### Overview
Test `stream_message()` method yields correct SSE events.

### File: `tests/test_server/test_agents.py`

Add new test class:

```python
class TestStreamMessage:
    """Tests for AgentManager.stream_message()."""

    def test_stream_yields_events(self, mock_letta_client):
        """stream_message yields SSE events from Letta stream."""
        # Create mock chunks
        reasoning_chunk = MagicMock()
        reasoning_chunk.message_type = "reasoning_message"
        reasoning_chunk.reasoning = "Thinking..."

        message_chunk = MagicMock()
        message_chunk.message_type = "assistant_message"
        message_chunk.content = "Hello!"

        done_chunk = MagicMock()
        done_chunk.message_type = "stop_reason"

        # Mock the stream context manager
        mock_stream = MagicMock()
        mock_stream.__iter__ = MagicMock(
            return_value=iter([reasoning_chunk, message_chunk, done_chunk])
        )
        mock_stream.__enter__ = MagicMock(return_value=mock_stream)
        mock_stream.__exit__ = MagicMock(return_value=False)
        mock_letta_client.agents.messages.stream.return_value = mock_stream

        # Create manager
        from letta_starter.server.agents import AgentManager
        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client  # noqa: SLF001

        # Collect events
        events = list(manager.stream_message("agent-123", "Hello"))

        # Verify we got expected events
        assert len(events) == 3
        assert '"type": "status"' in events[0]
        assert '"type": "message"' in events[1]
        assert '"type": "done"' in events[2]

    def test_stream_filters_ignored_chunks(self, mock_letta_client):
        """stream_message filters out internal message types."""
        # Mix of visible and internal chunks
        visible_chunk = MagicMock()
        visible_chunk.message_type = "assistant_message"
        visible_chunk.content = "Hello!"

        internal_chunk = MagicMock()
        internal_chunk.message_type = "tool_return_message"

        mock_stream = MagicMock()
        mock_stream.__iter__ = MagicMock(
            return_value=iter([internal_chunk, visible_chunk, internal_chunk])
        )
        mock_stream.__enter__ = MagicMock(return_value=mock_stream)
        mock_stream.__exit__ = MagicMock(return_value=False)
        mock_letta_client.agents.messages.stream.return_value = mock_stream

        from letta_starter.server.agents import AgentManager
        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client  # noqa: SLF001

        events = list(manager.stream_message("agent-123", "Hello"))

        # Only the visible chunk should produce an event
        assert len(events) == 1
        assert '"type": "message"' in events[0]

    def test_stream_handles_exception(self, mock_letta_client):
        """stream_message yields error event on exception."""
        mock_letta_client.agents.messages.stream.side_effect = Exception("Connection failed")

        from letta_starter.server.agents import AgentManager
        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client  # noqa: SLF001

        events = list(manager.stream_message("agent-123", "Hello"))

        assert len(events) == 1
        assert '"type": "error"' in events[0]
        assert "Connection failed" in events[0]

    def test_stream_passes_enable_thinking(self, mock_letta_client):
        """stream_message passes enable_thinking to SDK."""
        mock_stream = MagicMock()
        mock_stream.__iter__ = MagicMock(return_value=iter([]))
        mock_stream.__enter__ = MagicMock(return_value=mock_stream)
        mock_stream.__exit__ = MagicMock(return_value=False)
        mock_letta_client.agents.messages.stream.return_value = mock_stream

        from letta_starter.server.agents import AgentManager
        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client  # noqa: SLF001

        # Test with thinking enabled
        list(manager.stream_message("agent-123", "Hello", enable_thinking=True))
        call_kwargs = mock_letta_client.agents.messages.stream.call_args.kwargs
        assert call_kwargs["enable_thinking"] == "true"

        # Test with thinking disabled
        list(manager.stream_message("agent-123", "Hello", enable_thinking=False))
        call_kwargs = mock_letta_client.agents.messages.stream.call_args.kwargs
        assert call_kwargs["enable_thinking"] == "false"
```

### Fixture Update

Add to `tests/test_server/conftest.py`:

```python
@pytest.fixture
def mock_letta_client_with_stream():
    """Mock Letta client with streaming support."""
    client = MagicMock()

    # Existing mock setup
    client.list_agents.return_value = []
    client.create_agent.return_value = MagicMock(
        id="agent-123",
        name="tutor_user123",
    )

    # Add agents.messages.stream mock
    client.agents = MagicMock()
    client.agents.messages = MagicMock()
    client.agents.messages.stream = MagicMock()

    return client
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make test-agent` passes

---

## Phase 4: Streaming Endpoint Tests

### Overview
Test `/chat/stream` endpoint HTTP behavior.

### File: `tests/test_server/test_endpoints.py`

Add new test class:

```python
class TestChatStreamEndpoint:
    """Tests for POST /chat/stream endpoint."""

    def test_stream_returns_sse(self, test_client, mock_agent_manager):
        """Streaming endpoint returns SSE content type."""
        # Mock stream_message to yield test events
        def mock_stream(*args, **kwargs):
            yield 'data: {"type": "message", "content": "Hello"}\n\n'
            yield 'data: {"type": "done"}\n\n'

        mock_agent_manager.stream_message = mock_stream
        mock_agent_manager.get_agent_info = lambda agent_id: {"agent_id": agent_id}

        response = test_client.post(
            "/chat/stream",
            json={"agent_id": "agent-123", "message": "Hello"},
        )

        assert response.status_code == 200
        assert response.headers["content-type"] == "text/event-stream; charset=utf-8"
        assert response.headers["cache-control"] == "no-cache"

    def test_stream_content(self, test_client, mock_agent_manager):
        """Streaming endpoint yields SSE events."""
        events = [
            'data: {"type": "status", "content": "Thinking..."}\n\n',
            'data: {"type": "message", "content": "Hello!"}\n\n',
            'data: {"type": "done"}\n\n',
        ]

        def mock_stream(*args, **kwargs):
            yield from events

        mock_agent_manager.stream_message = mock_stream
        mock_agent_manager.get_agent_info = lambda agent_id: {"agent_id": agent_id}

        response = test_client.post(
            "/chat/stream",
            json={"agent_id": "agent-123", "message": "Hello"},
        )

        content = response.text
        assert 'data: {"type": "status"' in content
        assert 'data: {"type": "message"' in content
        assert 'data: {"type": "done"' in content

    def test_stream_agent_not_found(self, test_client, mock_agent_manager):
        """Streaming endpoint returns 404 for unknown agent."""
        mock_agent_manager.get_agent_info = lambda agent_id: None

        response = test_client.post(
            "/chat/stream",
            json={"agent_id": "unknown-agent", "message": "Hello"},
        )

        assert response.status_code == 404
        assert "Agent not found" in response.json()["detail"]

    def test_stream_missing_agent_id(self, test_client):
        """Streaming endpoint requires agent_id."""
        response = test_client.post(
            "/chat/stream",
            json={"message": "Hello"},
        )

        assert response.status_code == 422  # Validation error

    def test_stream_missing_message(self, test_client):
        """Streaming endpoint requires message."""
        response = test_client.post(
            "/chat/stream",
            json={"agent_id": "agent-123"},
        )

        assert response.status_code == 422  # Validation error

    def test_stream_with_optional_fields(self, test_client, mock_agent_manager):
        """Streaming endpoint accepts optional fields."""
        def mock_stream(*args, **kwargs):
            yield 'data: {"type": "done"}\n\n'

        mock_agent_manager.stream_message = mock_stream
        mock_agent_manager.get_agent_info = lambda agent_id: {"agent_id": agent_id}

        response = test_client.post(
            "/chat/stream",
            json={
                "agent_id": "agent-123",
                "message": "Hello",
                "chat_id": "chat-456",
                "chat_title": "My Chat",
                "enable_thinking": False,
            },
        )

        assert response.status_code == 200
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make test-agent` passes

---

## Phase 5: Pipe Streaming Tests

### Overview
Test updated Pipe with streaming support.

### File: `tests/test_pipe.py`

Replace or update with streaming tests:

```python
"""Tests for YouLab OpenWebUI Pipeline with streaming."""

import json
from unittest.mock import AsyncMock, MagicMock, patch

import pytest


class TestPipelineInit:
    """Tests for Pipeline initialization."""

    def test_default_valves(self):
        """Pipeline initializes with default valve values."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()

        assert pipeline.valves.LETTA_SERVICE_URL == "http://localhost:8100"
        assert pipeline.valves.AGENT_TYPE == "tutor"
        assert pipeline.valves.ENABLE_LOGGING is True
        assert pipeline.valves.ENABLE_THINKING is True

    def test_custom_valves(self):
        """Pipeline valve values can be overridden."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        pipeline.valves.LETTA_SERVICE_URL = "http://custom:9000"
        pipeline.valves.ENABLE_THINKING = False

        assert pipeline.valves.LETTA_SERVICE_URL == "http://custom:9000"
        assert pipeline.valves.ENABLE_THINKING is False


class TestPipelineLifecycle:
    """Tests for Pipeline lifecycle methods."""

    @pytest.mark.asyncio
    async def test_on_startup(self, capsys):
        """on_startup logs startup message."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        await pipeline.on_startup()

        captured = capsys.readouterr()
        assert "YouLab Pipeline started" in captured.out

    @pytest.mark.asyncio
    async def test_on_shutdown(self, capsys):
        """on_shutdown logs shutdown message."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        await pipeline.on_shutdown()

        captured = capsys.readouterr()
        assert "YouLab Pipeline stopped" in captured.out


class TestPipeStreaming:
    """Tests for Pipeline streaming functionality."""

    @pytest.mark.asyncio
    async def test_pipe_no_message(self):
        """pipe returns error for empty message."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()

        result = await pipeline.pipe(
            body={"messages": []},
            __user__={"id": "user-123"},
        )

        assert "No message provided" in result

    @pytest.mark.asyncio
    async def test_pipe_no_user(self):
        """pipe emits error for missing user."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        result = await pipeline.pipe(
            body={"messages": [{"content": "Hello"}]},
            __user__=None,
            __event_emitter__=emitter,
        )

        assert result == ""
        emitter.assert_called_once()
        call_arg = emitter.call_args[0][0]
        assert "Could not identify user" in call_arg["data"]["content"]

    @pytest.mark.asyncio
    async def test_pipe_streams_messages(self):
        """pipe streams SSE events to event emitter."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        # Mock SSE events from server
        sse_events = [
            MagicMock(data=json.dumps({"type": "status", "content": "Thinking..."})),
            MagicMock(data=json.dumps({"type": "message", "content": "Hello!"})),
            MagicMock(data=json.dumps({"type": "done"})),
        ]

        mock_event_source = MagicMock()
        mock_event_source.aiter_sse = AsyncMock(return_value=AsyncIterator(sse_events))

        # Mock agent lookup response
        mock_get_response = MagicMock()
        mock_get_response.status_code = 200
        mock_get_response.json.return_value = {
            "agents": [{"agent_id": "agent-123", "agent_type": "tutor"}]
        }

        with patch("httpx.AsyncClient") as mock_client_class:
            mock_client = AsyncMock()
            mock_client.__aenter__.return_value = mock_client
            mock_client.__aexit__.return_value = None
            mock_client.get.return_value = mock_get_response
            mock_client_class.return_value = mock_client

            with patch("letta_starter.pipelines.letta_pipe.aconnect_sse") as mock_sse:
                mock_sse_ctx = MagicMock()
                mock_sse_ctx.__aenter__ = AsyncMock(return_value=mock_event_source)
                mock_sse_ctx.__aexit__ = AsyncMock(return_value=None)
                mock_sse.return_value = mock_sse_ctx

                result = await pipeline.pipe(
                    body={"messages": [{"content": "Hello"}]},
                    __user__={"id": "user-123", "name": "Test User"},
                    __metadata__={"chat_id": "chat-456"},
                    __event_emitter__=emitter,
                )

        assert result == ""
        # Verify status, message, and done events emitted
        assert emitter.call_count >= 3

    @pytest.mark.asyncio
    async def test_pipe_handles_timeout(self):
        """pipe handles timeout gracefully."""
        import httpx

        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        with patch("httpx.AsyncClient") as mock_client_class:
            mock_client = AsyncMock()
            mock_client.__aenter__.return_value = mock_client
            mock_client.__aexit__.return_value = None
            mock_client.get.side_effect = httpx.TimeoutException("Timeout")
            mock_client_class.return_value = mock_client

            result = await pipeline.pipe(
                body={"messages": [{"content": "Hello"}]},
                __user__={"id": "user-123"},
                __event_emitter__=emitter,
            )

        assert result == ""
        emitter.assert_called()
        call_arg = emitter.call_args[0][0]
        assert "timed out" in call_arg["data"]["content"]

    @pytest.mark.asyncio
    async def test_pipe_handles_connection_error(self):
        """pipe handles connection error gracefully."""
        import httpx

        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        with patch("httpx.AsyncClient") as mock_client_class:
            mock_client = AsyncMock()
            mock_client.__aenter__.return_value = mock_client
            mock_client.__aexit__.return_value = None
            mock_client.get.side_effect = httpx.ConnectError("Connection refused")
            mock_client_class.return_value = mock_client

            result = await pipeline.pipe(
                body={"messages": [{"content": "Hello"}]},
                __user__={"id": "user-123"},
                __event_emitter__=emitter,
            )

        assert result == ""
        emitter.assert_called()
        call_arg = emitter.call_args[0][0]
        assert "Could not connect" in call_arg["data"]["content"]


class TestHandleSSEEvent:
    """Tests for Pipeline._handle_sse_event()."""

    @pytest.mark.asyncio
    async def test_status_event(self):
        """status event emits status to OpenWebUI."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        await pipeline._handle_sse_event(  # noqa: SLF001
            json.dumps({"type": "status", "content": "Thinking..."}),
            emitter,
        )

        emitter.assert_called_once()
        call_arg = emitter.call_args[0][0]
        assert call_arg["type"] == "status"
        assert call_arg["data"]["description"] == "Thinking..."
        assert call_arg["data"]["done"] is False

    @pytest.mark.asyncio
    async def test_message_event(self):
        """message event emits message to OpenWebUI."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        await pipeline._handle_sse_event(  # noqa: SLF001
            json.dumps({"type": "message", "content": "Hello!"}),
            emitter,
        )

        emitter.assert_called_once()
        call_arg = emitter.call_args[0][0]
        assert call_arg["type"] == "message"
        assert call_arg["data"]["content"] == "Hello!"

    @pytest.mark.asyncio
    async def test_done_event(self):
        """done event emits completion status."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        await pipeline._handle_sse_event(  # noqa: SLF001
            json.dumps({"type": "done"}),
            emitter,
        )

        emitter.assert_called_once()
        call_arg = emitter.call_args[0][0]
        assert call_arg["type"] == "status"
        assert call_arg["data"]["done"] is True

    @pytest.mark.asyncio
    async def test_error_event(self):
        """error event emits error message."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        await pipeline._handle_sse_event(  # noqa: SLF001
            json.dumps({"type": "error", "message": "Something broke"}),
            emitter,
        )

        emitter.assert_called_once()
        call_arg = emitter.call_args[0][0]
        assert call_arg["type"] == "message"
        assert "Error:" in call_arg["data"]["content"]

    @pytest.mark.asyncio
    async def test_no_emitter(self):
        """handle_sse_event does nothing without emitter."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()

        # Should not raise
        await pipeline._handle_sse_event(  # noqa: SLF001
            json.dumps({"type": "message", "content": "Hello"}),
            None,
        )

    @pytest.mark.asyncio
    async def test_invalid_json(self, capsys):
        """handle_sse_event handles invalid JSON gracefully."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()
        emitter = AsyncMock()

        await pipeline._handle_sse_event(  # noqa: SLF001
            "not valid json",
            emitter,
        )

        emitter.assert_not_called()
        captured = capsys.readouterr()
        assert "Failed to parse SSE" in captured.out


class TestEnsureAgentExists:
    """Tests for Pipeline._ensure_agent_exists()."""

    @pytest.mark.asyncio
    async def test_finds_existing_agent(self):
        """Returns agent_id if agent exists."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()

        mock_response = MagicMock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "agents": [{"agent_id": "agent-123", "agent_type": "tutor"}]
        }

        mock_client = AsyncMock()
        mock_client.get.return_value = mock_response

        result = await pipeline._ensure_agent_exists(  # noqa: SLF001
            mock_client, "user-123"
        )

        assert result == "agent-123"

    @pytest.mark.asyncio
    async def test_creates_new_agent(self):
        """Creates agent if none exists."""
        from letta_starter.pipelines.letta_pipe import Pipeline

        pipeline = Pipeline()

        # First call returns no agents
        mock_get_response = MagicMock()
        mock_get_response.status_code = 200
        mock_get_response.json.return_value = {"agents": []}

        # Second call creates agent
        mock_post_response = MagicMock()
        mock_post_response.status_code = 201
        mock_post_response.json.return_value = {"agent_id": "new-agent-456"}

        mock_client = AsyncMock()
        mock_client.get.return_value = mock_get_response
        mock_client.post.return_value = mock_post_response

        result = await pipeline._ensure_agent_exists(  # noqa: SLF001
            mock_client, "user-123", "Test User"
        )

        assert result == "new-agent-456"
        mock_client.post.assert_called_once()


# Helper for async iteration in tests
class AsyncIterator:
    """Helper to make a list async-iterable."""

    def __init__(self, items):
        self.items = items
        self.index = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.index >= len(self.items):
            raise StopAsyncIteration
        item = self.items[self.index]
        self.index += 1
        return item
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make test-agent` passes

---

## Phase 6: Fixture Cleanup

### Overview
Consolidate and clean up streaming fixtures.

### File: `tests/test_server/conftest.py`

Final fixture file:

```python
"""Fixtures for server tests."""

import pytest
from fastapi.testclient import TestClient
from unittest.mock import MagicMock

from letta_starter.server.agents import AgentManager
from letta_starter.server.main import app, get_strategy_manager


@pytest.fixture
def mock_letta_client():
    """Mock Letta client for testing."""
    client = MagicMock()

    # Basic operations
    client.list_agents.return_value = []
    client.create_agent.return_value = MagicMock(
        id="agent-123",
        name="tutor_user123",
    )
    client.send_message.return_value = MagicMock(
        messages=[MagicMock(text="Hello from agent!")],
    )

    # Streaming support
    client.agents = MagicMock()
    client.agents.messages = MagicMock()
    client.agents.messages.stream = MagicMock()

    return client


@pytest.fixture
def mock_agent_manager(mock_letta_client):
    """AgentManager with mocked Letta client."""
    manager = AgentManager.__new__(AgentManager)
    manager._client = mock_letta_client  # noqa: SLF001
    manager._cache = {}  # noqa: SLF001
    return manager


@pytest.fixture
def test_client(mock_agent_manager):
    """FastAPI test client with mocked dependencies."""
    # Set up app state
    app.state.agent_manager = mock_agent_manager

    # Mock strategy manager
    mock_strategy = MagicMock()
    mock_strategy.health_check.return_value = True
    mock_strategy.agent_id = "strategy-agent-id"
    app.dependency_overrides[get_strategy_manager] = lambda: mock_strategy

    yield TestClient(app)

    # Cleanup
    app.dependency_overrides.clear()
```

### Success Criteria
- [ ] `make lint-fix` passes
- [ ] `make test-agent` passes
- [ ] All tests pass with consolidated fixtures

---

## Summary

| Phase | What | Files | Tests Added |
|-------|------|-------|-------------|
| 1 | Schema tests | `test_schemas.py` | 4 tests |
| 2 | Chunk conversion | `test_agents.py` | 12 tests |
| 3 | Stream message | `test_agents.py` | 4 tests |
| 4 | Endpoint tests | `test_endpoints.py` | 6 tests |
| 5 | Pipe streaming | `test_pipe.py` | 15 tests |
| 6 | Fixture cleanup | `conftest.py` | - |

**Total: ~41 new tests**

## Implementation Order

This test plan is designed to be implemented in parallel with the main implementation plan:

| Implementation Phase | Test Phase(s) |
|---------------------|---------------|
| Phase 1: Dependency | - (no tests needed) |
| Phase 2: AgentManager | Phase 1 (schemas), Phase 2, Phase 3 |
| Phase 3: Endpoint | Phase 4 |
| Phase 4: Pipe | Phase 5 |
| Phase 5: E2E | Phase 6 (cleanup) |

## References

- **Implementation Plan**: `2025-12-31-phase-2-streaming-implementation.md`
- **Existing Tests**: `tests/test_server/` for patterns
- **Test Configuration**: `pyproject.toml` lines 146-176

---

## Validation Notes

**Created**: 2025-12-31

### Test Coverage Goals
- All new public methods have tests
- Edge cases covered (missing fields, exceptions, invalid data)
- Error handling paths tested
- Mock isolation prevents external dependencies
