# Phase 3: Honcho Message Persistence Implementation Plan

## Overview

Add Honcho message persistence to capture all OpenWebUI conversations for future theory-of-mind capabilities. Every message to/from any Letta agent is persisted to Honcho with 1:1 session mapping (OpenWebUI chat = Honcho session).

**Primary Goal**: Persist all messages to Honcho as foundation for future personalization features.

**Not In Scope**: Memory block updates, dialectic queries, working representations, cross-session context injection (all future phases).

## Current State Analysis

### What Exists
- FastAPI HTTP service on port 8100 (`src/letta_starter/server/main.py`)
- Per-user Letta agent management via `AgentManager`
- Streaming and non-streaming chat endpoints (`/chat`, `/chat/stream`)
- Full user context available: `user_id`, `chat_id`, `chat_title`, `agent_type`
- Langfuse tracing integration pattern to follow

### What's Missing
- Honcho SDK dependency
- Honcho configuration in settings
- HonchoClient module for message persistence
- Integration into chat endpoints

### Key Integration Points
- `/chat` endpoint (`main.py:151-206`) - non-streaming messages
- `/chat/stream` endpoint (`main.py:209-234`) - streaming messages
- `ServiceSettings` class (`settings.py:106-153`) - configuration
- Lifespan handler (`main.py:29-48`) - client initialization

## Desired End State

A working system where:
1. Every user message is persisted to Honcho before being sent to Letta
2. Every agent response is persisted to Honcho after receiving from Letta
3. OpenWebUI chats map 1:1 to Honcho sessions
4. Persistence is non-blocking (fire-and-forget)
5. Honcho unavailability doesn't break the tutoring flow

### Verification
- Send messages in OpenWebUI, verify they appear in Honcho dashboard
- Multiple chats create multiple Honcho sessions
- Service continues working if Honcho is unreachable
- No noticeable latency increase in chat responses

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Session naming | `chat_{chat_id}` | OpenWebUI chat_id is unique |
| Peer model | `student_{user_id}` + shared `tutor` | ToM accumulates on student peer |
| Message metadata | `chat_id`, `chat_title`, `agent_type` | Enables future filtering/context |
| Workspace | `youlab` | Single workspace for all data |
| Async model | Fire-and-forget via `asyncio.create_task()` | Non-blocking persistence |
| Error handling | Log warning, continue | Honcho is additive, not critical |
| Observation | Student: `observe_me=True`, Tutor: `observe_others=True` | We model students, not tutors |
| Flow position | Before AND after Letta | Capture all interactions |

## What We're NOT Doing

- Memory block updates from Honcho insights (Phase 6)
- Dialectic API queries (future)
- Working representation retrieval (future)
- Cross-session context injection into prompts (future)
- Honcho-based conversation summaries (future)
- Peer cards or profile extraction (future)

---

## Phase 1: Configuration & Dependencies

### Overview
Add Honcho SDK dependency and configuration settings.

### Changes Required

#### 1. Add Dependency
**File**: `pyproject.toml`

Add `honcho-ai` to dependencies (after line 17):

```toml
dependencies = [
    "letta>=0.6.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "python-dotenv>=1.0.0",
    "structlog>=24.1.0",
    "httpx>=0.27.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.32.0",
    "langfuse>=2.0.0",
    "httpx-sse>=0.4.0,<0.5.0",
    "honcho-ai>=2.0.0",
]
```

#### 2. Extend ServiceSettings
**File**: `src/letta_starter/config/settings.py`

Add Honcho configuration to `ServiceSettings` class (after line 153):

```python
class ServiceSettings(BaseSettings):
    # ... existing fields ...

    # Honcho configuration
    honcho_enabled: bool = Field(
        default=True,
        description="Enable Honcho message persistence",
    )
    honcho_workspace_id: str = Field(
        default="youlab",
        description="Honcho workspace identifier",
    )
    honcho_api_key: str | None = Field(
        default=None,
        description="Honcho API key (required for production)",
    )
    honcho_environment: str = Field(
        default="demo",
        description="Honcho environment: demo, local, or production",
    )
```

#### 3. Update Environment Example
**File**: `.env.example`

Add Honcho variables:

```bash
# Honcho (message persistence)
YOULAB_SERVICE_HONCHO_ENABLED=true
YOULAB_SERVICE_HONCHO_WORKSPACE_ID=youlab
YOULAB_SERVICE_HONCHO_API_KEY=
YOULAB_SERVICE_HONCHO_ENVIRONMENT=demo
```

### Success Criteria

#### Automated Verification
- [ ] Dependencies install: `uv sync`
- [ ] Type checking passes: `make check-agent`
- [ ] Settings load correctly: `python -c "from letta_starter.config.settings import ServiceSettings; s = ServiceSettings(); print(s.honcho_workspace_id)"`

#### Manual Verification
- [ ] `.env.example` documents all new variables

---

## Phase 2: HonchoClient Module

### Overview
Create the HonchoClient class that manages Honcho peers, sessions, and message persistence.

### Changes Required

#### 1. Create Module Directory
**Directory**: `src/letta_starter/honcho/`

Create `__init__.py`:

```python
"""Honcho integration for message persistence."""

from letta_starter.honcho.client import HonchoClient

__all__ = ["HonchoClient"]
```

#### 2. Create HonchoClient
**File**: `src/letta_starter/honcho/client.py`

```python
"""Honcho client for message persistence."""

from __future__ import annotations

import asyncio
from typing import TYPE_CHECKING

import structlog

if TYPE_CHECKING:
    from honcho import Honcho

log = structlog.get_logger()


class HonchoClient:
    """Manages Honcho peers and sessions for YouLab message persistence.

    Architecture:
    - One workspace: "youlab"
    - One peer per student: "student_{user_id}"
    - One shared tutor peer: "tutor"
    - One session per OpenWebUI chat: "chat_{chat_id}"
    """

    def __init__(
        self,
        workspace_id: str,
        api_key: str | None = None,
        environment: str = "demo",
    ) -> None:
        """Initialize Honcho client.

        Args:
            workspace_id: Honcho workspace identifier
            api_key: API key for production (None for demo)
            environment: "demo", "local", or "production"
        """
        self.workspace_id = workspace_id
        self.api_key = api_key
        self.environment = environment
        self._client: Honcho | None = None
        self._tutor_peer_id = "tutor"

    @property
    def client(self) -> Honcho:
        """Lazy-load Honcho client."""
        if self._client is None:
            from honcho import Honcho

            if self.environment == "demo":
                self._client = Honcho(
                    workspace_id=self.workspace_id,
                    environment="demo",
                )
            else:
                self._client = Honcho(
                    workspace_id=self.workspace_id,
                    api_key=self.api_key,
                    environment=self.environment,
                )
            log.info(
                "honcho_client_initialized",
                workspace_id=self.workspace_id,
                environment=self.environment,
            )
        return self._client

    def _get_student_peer_id(self, user_id: str) -> str:
        """Generate peer ID for a student."""
        return f"student_{user_id}"

    def _get_session_id(self, chat_id: str) -> str:
        """Generate session ID from OpenWebUI chat ID."""
        return f"chat_{chat_id}"

    async def persist_user_message(
        self,
        user_id: str,
        chat_id: str,
        message: str,
        chat_title: str | None = None,
        agent_type: str = "tutor",
    ) -> None:
        """Persist a user message to Honcho.

        Args:
            user_id: OpenWebUI user identifier
            chat_id: OpenWebUI chat identifier
            message: User message content
            chat_title: Optional chat title for metadata
            agent_type: Type of agent (for metadata)
        """
        try:
            student_peer = self.client.peer(self._get_student_peer_id(user_id))
            session = self.client.session(self._get_session_id(chat_id))

            metadata = {
                "chat_id": chat_id,
                "agent_type": agent_type,
            }
            if chat_title:
                metadata["chat_title"] = chat_title

            session.add_messages([
                student_peer.message(message, metadata=metadata)
            ])

            log.debug(
                "honcho_user_message_persisted",
                user_id=user_id,
                chat_id=chat_id,
                message_length=len(message),
            )
        except Exception as e:
            log.warning(
                "honcho_persist_failed",
                error=str(e),
                user_id=user_id,
                chat_id=chat_id,
                message_type="user",
            )

    async def persist_agent_message(
        self,
        user_id: str,
        chat_id: str,
        message: str,
        chat_title: str | None = None,
        agent_type: str = "tutor",
    ) -> None:
        """Persist an agent response to Honcho.

        Args:
            user_id: OpenWebUI user identifier (for session context)
            chat_id: OpenWebUI chat identifier
            message: Agent response content
            chat_title: Optional chat title for metadata
            agent_type: Type of agent that responded
        """
        try:
            tutor_peer = self.client.peer(self._tutor_peer_id)
            session = self.client.session(self._get_session_id(chat_id))

            metadata = {
                "chat_id": chat_id,
                "agent_type": agent_type,
                "user_id": user_id,  # Track which student this was for
            }
            if chat_title:
                metadata["chat_title"] = chat_title

            session.add_messages([
                tutor_peer.message(message, metadata=metadata)
            ])

            log.debug(
                "honcho_agent_message_persisted",
                user_id=user_id,
                chat_id=chat_id,
                message_length=len(message),
            )
        except Exception as e:
            log.warning(
                "honcho_persist_failed",
                error=str(e),
                user_id=user_id,
                chat_id=chat_id,
                message_type="agent",
            )

    def check_connection(self) -> bool:
        """Check if Honcho is reachable.

        Returns:
            True if connection successful, False otherwise.
        """
        try:
            # Attempt to access a peer (lazy operation that validates connection)
            _ = self.client.peer("connection_test")
            return True
        except Exception as e:
            log.warning("honcho_connection_check_failed", error=str(e))
            return False


def create_persist_task(
    honcho_client: HonchoClient | None,
    user_id: str,
    chat_id: str,
    message: str,
    is_user: bool,
    chat_title: str | None = None,
    agent_type: str = "tutor",
) -> None:
    """Fire-and-forget message persistence.

    Creates an asyncio task to persist the message without blocking.
    Safe to call even if honcho_client is None.

    Args:
        honcho_client: HonchoClient instance (or None if disabled)
        user_id: OpenWebUI user identifier
        chat_id: OpenWebUI chat identifier
        message: Message content
        is_user: True for user messages, False for agent responses
        chat_title: Optional chat title
        agent_type: Agent type for metadata
    """
    if honcho_client is None:
        return

    if not chat_id:
        log.debug("honcho_persist_skipped", reason="no_chat_id")
        return

    async def _persist() -> None:
        if is_user:
            await honcho_client.persist_user_message(
                user_id=user_id,
                chat_id=chat_id,
                message=message,
                chat_title=chat_title,
                agent_type=agent_type,
            )
        else:
            await honcho_client.persist_agent_message(
                user_id=user_id,
                chat_id=chat_id,
                message=message,
                chat_title=chat_title,
                agent_type=agent_type,
            )

    asyncio.create_task(_persist())
```

### Success Criteria

#### Automated Verification
- [ ] Module imports: `python -c "from letta_starter.honcho import HonchoClient"`
- [ ] Type checking passes: `make check-agent`
- [ ] Lint passes: `make lint-fix`

#### Manual Verification
- [ ] Code review confirms fire-and-forget pattern
- [ ] Error handling logs warnings but doesn't raise

---

## Phase 3: Server Integration

### Overview
Integrate HonchoClient into the FastAPI server, initializing it during startup and calling it from chat endpoints.

### Changes Required

#### 1. Update Imports
**File**: `src/letta_starter/server/main.py`

Add import at top (after line 23):

```python
from letta_starter.honcho import HonchoClient
from letta_starter.honcho.client import create_persist_task
```

#### 2. Initialize HonchoClient in Lifespan
**File**: `src/letta_starter/server/main.py`

Update lifespan handler (replace lines 29-48):

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    """Application lifespan handler."""
    # Startup
    log.info("starting_service", host=settings.host, port=settings.port)
    app.state.agent_manager = AgentManager(letta_base_url=settings.letta_base_url)
    init_strategy_manager(letta_base_url=settings.letta_base_url)

    # Initialize Honcho client (if enabled)
    if settings.honcho_enabled:
        app.state.honcho_client = HonchoClient(
            workspace_id=settings.honcho_workspace_id,
            api_key=settings.honcho_api_key,
            environment=settings.honcho_environment,
        )
        honcho_ok = app.state.honcho_client.check_connection()
        log.info("honcho_initialized", connected=honcho_ok)
    else:
        app.state.honcho_client = None
        log.info("honcho_disabled")

    # Rebuild cache from Letta
    try:
        count = await app.state.agent_manager.rebuild_cache()
        log.info("startup_complete", cached_agents=count)
    except Exception as e:
        log.warning("letta_not_available_at_startup", error=str(e))

    yield

    # Shutdown
    log.info("shutting_down_service")
```

#### 3. Add Helper Function
**File**: `src/letta_starter/server/main.py`

Add after `get_agent_manager()` function (after line 64):

```python
def get_honcho_client() -> HonchoClient | None:
    """Get the Honcho client from app state (may be None if disabled)."""
    return getattr(app.state, "honcho_client", None)
```

#### 4. Update /chat Endpoint
**File**: `src/letta_starter/server/main.py`

Update chat endpoint (replace lines 151-206):

```python
@app.post("/chat")
async def chat(request: ChatRequest) -> ChatResponse:
    """Send a message to an agent."""
    manager = get_agent_manager()
    honcho = get_honcho_client()

    # Verify agent exists
    info = manager.get_agent_info(request.agent_id)
    if info is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Agent not found: {request.agent_id}",
        )

    user_id = info.get("user_id", "unknown")
    agent_type = info.get("agent_type", "tutor")

    # Persist user message to Honcho (fire-and-forget)
    if request.chat_id:
        create_persist_task(
            honcho_client=honcho,
            user_id=user_id,
            chat_id=request.chat_id,
            message=request.message,
            is_user=True,
            chat_title=request.chat_title,
            agent_type=agent_type,
        )

    with trace_chat(
        user_id=user_id,
        agent_id=request.agent_id,
        chat_id=request.chat_id,
        metadata={"chat_title": request.chat_title},
    ) as trace_ctx:
        try:
            log.info(
                "chat_request",
                trace_id=trace_ctx.get("trace_id"),
                agent_id=request.agent_id,
                chat_id=request.chat_id,
                chat_title=request.chat_title,
                message_length=len(request.message),
            )

            response_text = manager.send_message(request.agent_id, request.message)

            # Record generation in Langfuse
            trace_generation(
                trace_ctx,
                name="agent_response",
                input_text=request.message,
                output_text=response_text,
            )

            # Persist agent response to Honcho (fire-and-forget)
            if request.chat_id and response_text:
                create_persist_task(
                    honcho_client=honcho,
                    user_id=user_id,
                    chat_id=request.chat_id,
                    message=response_text,
                    is_user=False,
                    chat_title=request.chat_title,
                    agent_type=agent_type,
                )

            return ChatResponse(
                response=response_text if response_text else "No response from agent.",
                agent_id=request.agent_id,
            )
        except Exception as e:
            log.exception(
                "chat_failed",
                trace_id=trace_ctx.get("trace_id"),
                agent_id=request.agent_id,
                error=str(e),
            )
            raise HTTPException(
                status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
                detail="Failed to communicate with agent",
            ) from None
```

#### 5. Update /chat/stream Endpoint
**File**: `src/letta_starter/server/main.py`

Update streaming endpoint (replace lines 209-234):

```python
@app.post("/chat/stream")
async def chat_stream(request: StreamChatRequest) -> StreamingResponse:
    """Send a message to an agent with streaming response (SSE)."""
    manager = get_agent_manager()
    honcho = get_honcho_client()

    # Verify agent exists
    info = manager.get_agent_info(request.agent_id)
    if info is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Agent not found: {request.agent_id}",
        )

    user_id = info.get("user_id", "unknown")
    agent_type = info.get("agent_type", "tutor")

    # Persist user message to Honcho (fire-and-forget)
    if request.chat_id:
        create_persist_task(
            honcho_client=honcho,
            user_id=user_id,
            chat_id=request.chat_id,
            message=request.message,
            is_user=True,
            chat_title=request.chat_title,
            agent_type=agent_type,
        )

    async def stream_with_persistence() -> AsyncIterator[str]:
        """Wrap streaming to capture full response for Honcho."""
        full_response: list[str] = []

        async for chunk in manager.stream_message(
            agent_id=request.agent_id,
            message=request.message,
            enable_thinking=request.enable_thinking,
        ):
            # Capture message content for persistence
            if chunk and "\"type\": \"message\"" in chunk:
                import json
                try:
                    # Extract content from SSE data
                    data_start = chunk.find("data: ") + 6
                    data_end = chunk.find("\n\n")
                    if data_start > 5 and data_end > data_start:
                        data = json.loads(chunk[data_start:data_end])
                        if data.get("type") == "message" and data.get("content"):
                            full_response.append(data["content"])
                except (json.JSONDecodeError, KeyError):
                    pass
            yield chunk

        # Persist complete response after stream ends
        if request.chat_id and full_response:
            create_persist_task(
                honcho_client=honcho,
                user_id=user_id,
                chat_id=request.chat_id,
                message="".join(full_response),
                is_user=False,
                chat_title=request.chat_title,
                agent_type=agent_type,
            )

    return StreamingResponse(
        stream_with_persistence(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

#### 6. Update Health Endpoint
**File**: `src/letta_starter/server/main.py`

Update health endpoint to include Honcho status (replace lines 67-75):

```python
@app.get("/health")
async def health() -> HealthResponse:
    """Check service health."""
    manager = get_agent_manager()
    honcho = get_honcho_client()

    letta_ok = manager.check_letta_connection()
    honcho_ok = honcho.check_connection() if honcho else False

    # Service is "ok" if Letta works, "degraded" otherwise
    # Honcho status is informational (not required for core function)
    health_status = "ok" if letta_ok else "degraded"

    return HealthResponse(
        status=health_status,
        letta_connected=letta_ok,
        honcho_connected=honcho_ok,
    )
```

#### 7. Update HealthResponse Schema
**File**: `src/letta_starter/server/schemas.py`

Add `honcho_connected` field to HealthResponse:

```python
class HealthResponse(BaseModel):
    """Health check response."""

    status: str
    letta_connected: bool
    honcho_connected: bool = False
    version: str = "0.1.0"
```

### Success Criteria

#### Automated Verification
- [ ] Type checking passes: `make check-agent`
- [ ] All tests pass: `make test-agent`
- [ ] Server starts: `uv run letta-server` (check startup logs)
- [ ] Health endpoint works: `curl http://localhost:8100/health`

#### Manual Verification
- [ ] Startup logs show "honcho_initialized"
- [ ] Chat requests log "honcho_user_message_persisted" and "honcho_agent_message_persisted" at DEBUG level
- [ ] Service continues working if Honcho demo server is unreachable

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Testing & Verification

### Overview
Add unit tests for HonchoClient and integration tests for the message persistence flow.

### Changes Required

#### 1. Create Test File
**File**: `tests/test_honcho.py`

```python
"""Tests for Honcho integration."""

from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from letta_starter.honcho.client import HonchoClient, create_persist_task


class TestHonchoClient:
    """Tests for HonchoClient class."""

    def test_peer_id_generation(self) -> None:
        """Test peer ID generation."""
        client = HonchoClient(workspace_id="test", environment="demo")
        assert client._get_student_peer_id("user123") == "student_user123"

    def test_session_id_generation(self) -> None:
        """Test session ID generation."""
        client = HonchoClient(workspace_id="test", environment="demo")
        assert client._get_session_id("chat456") == "chat_chat456"

    @patch("letta_starter.honcho.client.Honcho")
    def test_client_lazy_init(self, mock_honcho_class: MagicMock) -> None:
        """Test that Honcho client is lazily initialized."""
        client = HonchoClient(workspace_id="test", environment="demo")
        assert client._client is None

        # Access client property
        _ = client.client

        mock_honcho_class.assert_called_once()
        assert client._client is not None

    @patch("letta_starter.honcho.client.Honcho")
    def test_production_client_requires_api_key(
        self, mock_honcho_class: MagicMock
    ) -> None:
        """Test production environment uses API key."""
        client = HonchoClient(
            workspace_id="test",
            api_key="secret-key",
            environment="production",
        )
        _ = client.client

        mock_honcho_class.assert_called_once_with(
            workspace_id="test",
            api_key="secret-key",
            environment="production",
        )


class TestPersistTask:
    """Tests for fire-and-forget persistence."""

    def test_persist_skips_when_client_none(self) -> None:
        """Test that persistence is skipped when client is None."""
        # Should not raise any exceptions
        create_persist_task(
            honcho_client=None,
            user_id="user123",
            chat_id="chat456",
            message="test",
            is_user=True,
        )

    def test_persist_skips_when_no_chat_id(self) -> None:
        """Test that persistence is skipped when chat_id is empty."""
        mock_client = MagicMock(spec=HonchoClient)

        create_persist_task(
            honcho_client=mock_client,
            user_id="user123",
            chat_id="",  # Empty chat_id
            message="test",
            is_user=True,
        )

        # No async task should be created for persistence
        mock_client.persist_user_message.assert_not_called()


class TestHonchoClientPersistence:
    """Tests for message persistence methods."""

    @pytest.mark.asyncio
    async def test_persist_user_message_logs_on_error(self) -> None:
        """Test that errors are logged but not raised."""
        client = HonchoClient(workspace_id="test", environment="demo")

        # Mock to raise an exception
        with patch.object(client, "client") as mock_client:
            mock_client.peer.side_effect = Exception("Connection failed")

            # Should not raise
            await client.persist_user_message(
                user_id="user123",
                chat_id="chat456",
                message="test message",
            )

    @pytest.mark.asyncio
    async def test_persist_agent_message_includes_metadata(self) -> None:
        """Test that agent messages include correct metadata."""
        client = HonchoClient(workspace_id="test", environment="demo")

        with patch.object(client, "client") as mock_client:
            mock_peer = MagicMock()
            mock_session = MagicMock()
            mock_client.peer.return_value = mock_peer
            mock_client.session.return_value = mock_session

            await client.persist_agent_message(
                user_id="user123",
                chat_id="chat456",
                message="response",
                chat_title="Test Chat",
                agent_type="tutor",
            )

            # Verify peer and session were accessed
            mock_client.peer.assert_called_with("tutor")
            mock_client.session.assert_called_with("chat_chat456")

            # Verify message was added
            mock_session.add_messages.assert_called_once()
```

#### 2. Add Integration Test
**File**: `tests/test_server_honcho.py`

```python
"""Integration tests for Honcho with server endpoints."""

from unittest.mock import MagicMock, patch

import pytest
from fastapi.testclient import TestClient


class TestServerHonchoIntegration:
    """Test Honcho integration in server endpoints."""

    @pytest.fixture
    def mock_agent_manager(self) -> MagicMock:
        """Create mock agent manager."""
        manager = MagicMock()
        manager.get_agent_info.return_value = {
            "agent_id": "agent-123",
            "user_id": "user-456",
            "agent_type": "tutor",
            "agent_name": "youlab_user-456_tutor",
        }
        manager.send_message.return_value = "Hello! I'm here to help."
        manager.check_letta_connection.return_value = True
        return manager

    @pytest.fixture
    def mock_honcho_client(self) -> MagicMock:
        """Create mock Honcho client."""
        client = MagicMock()
        client.check_connection.return_value = True
        return client

    def test_health_includes_honcho_status(
        self,
        mock_agent_manager: MagicMock,
        mock_honcho_client: MagicMock,
    ) -> None:
        """Test health endpoint includes Honcho connection status."""
        with patch("letta_starter.server.main.get_agent_manager") as mock_get_manager:
            with patch("letta_starter.server.main.get_honcho_client") as mock_get_honcho:
                mock_get_manager.return_value = mock_agent_manager
                mock_get_honcho.return_value = mock_honcho_client

                from letta_starter.server.main import app
                client = TestClient(app)

                response = client.get("/health")

                assert response.status_code == 200
                data = response.json()
                assert "honcho_connected" in data
```

### Success Criteria

#### Automated Verification
- [ ] All tests pass: `make test-agent`
- [ ] Coverage includes new honcho module: check coverage report

#### Manual Verification
- [ ] Run `uv run letta-server` and send messages via OpenWebUI
- [ ] Check Honcho demo dashboard at https://demo.honcho.dev for persisted messages
- [ ] Verify messages appear with correct metadata (chat_id, chat_title, agent_type)
- [ ] Verify user messages and agent responses are both captured
- [ ] Test multiple chats create separate sessions in Honcho

---

## Testing Strategy

### Unit Tests
- HonchoClient initialization (demo vs production)
- Peer and session ID generation
- Fire-and-forget task creation
- Error handling (warnings logged, no exceptions raised)

### Integration Tests
- Health endpoint with Honcho status
- Chat endpoint triggers persistence
- Streaming endpoint captures full response

### Manual Testing Steps
1. Start server with Honcho enabled: `uv run letta-server`
2. Verify startup logs show "honcho_initialized"
3. Send a message in OpenWebUI
4. Check server logs for persistence debug messages
5. Visit https://demo.honcho.dev and verify:
   - Session created with `chat_{chat_id}` naming
   - User message from `student_{user_id}` peer
   - Agent response from `tutor` peer
   - Metadata includes chat_title and agent_type
6. Test graceful degradation:
   - Set `YOULAB_SERVICE_HONCHO_ENABLED=false`
   - Verify chat still works without Honcho

## Performance Considerations

- **Non-blocking**: All Honcho calls use `asyncio.create_task()` for fire-and-forget
- **No latency impact**: User sees response immediately; persistence happens in background
- **Graceful degradation**: Honcho failures log warnings but don't affect core functionality

## Migration Notes

- No data migration needed (new feature)
- Existing conversations won't be backfilled to Honcho
- Future enhancement could import historical data if needed

## References

- Research document: `thoughts/shared/research/2026-01-06-phase-3-honcho-integration-research.md`
- Technical foundation plan: `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md`
- Honcho SDK documentation: https://docs.honcho.dev/v2/documentation/reference/sdk
- Honcho Python SDK: https://github.com/plastic-labs/honcho-python-core
