# Ralph Wiggum MVP Implementation Plan

## Overview

Build a greenfield "Claude Code-like" experience: OpenWebUI chat → OpenHands sandbox with full streaming. Each user gets a persistent workspace (files survive across chats), each chat is a fresh conversation context. Agent has access to `query_honcho` tool to understand student history.

## Current State Analysis

**Existing YouLab**: Complex Letta-based system with curriculum configs, memory blocks, background agents, diff approval. Too heavy for MVP.

**What we're taking**:
- OpenWebUI Pipe pattern (`letta_pipe.py`) - proven streaming approach
- Honcho client (`honcho/client.py`) - message persistence + dialectic queries
- `query_honcho` tool concept - but simplified for OpenHands

**What we're leaving behind**:
- Letta, memory blocks, curriculum TOML, background agents, storage layer, agent factory

## Desired End State

```
ralph/
├── __init__.py
├── pipe.py              # OpenWebUI Pipe (entry point)
├── openhands_client.py  # OpenHands SDK wrapper
├── honcho.py            # Honcho persistence + dialectic
├── tools/
│   └── query_honcho.py  # Custom tool for OpenHands agent
└── config.py            # Pydantic settings
```

**User Experience**:
1. User opens new chat in OpenWebUI → fresh OpenHands conversation, same workspace
2. User sends message → streams through to Claude via OpenHands
3. Agent can run terminal commands, edit files in isolated Docker sandbox
4. Agent can call `query_honcho("What does this student struggle with?")`
5. Files persist across chats (like opening Claude Code in same directory)
6. All messages persisted to Honcho for future dialectic queries

**Verification**:
- `uv run python -c "from ralph import Pipe; print('ok')"`
- Copy `ralph/pipe.py` to OpenWebUI pipelines, send message, see streaming response
- Run command in chat, verify it executed in sandbox
- Create file in one chat, verify it exists in next chat (same user)
- Query honcho tool returns meaningful insights after conversation history exists

## What We're NOT Doing

- No curriculum/course system
- No memory blocks or Letta
- No background agents
- No diff approval system
- No multi-model orchestration
- No agent factory patterns
- No git-backed storage
- No complex configuration (just env vars)

---

## Phase 1: Project Setup

### Overview
Create greenfield `ralph/` directory with minimal dependencies and configuration.

### Changes Required:

#### 1. Create directory structure
**File**: `ralph/__init__.py`
```python
"""Ralph Wiggum MVP - OpenHands + OpenWebUI integration."""

from ralph.pipe import Pipe

__all__ = ["Pipe"]
```

#### 2. Configuration
**File**: `ralph/config.py`
```python
"""Configuration via environment variables."""

from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    """Ralph configuration."""

    # OpenRouter for model flexibility
    openrouter_api_key: str
    openrouter_model: str = "anthropic/claude-sonnet-4-20250514"

    # Honcho
    honcho_workspace_id: str = "ralph"
    honcho_environment: str = "demo"  # demo, local, production
    honcho_api_key: str | None = None

    # Workspace paths
    user_data_dir: str = "/data/ralph/users"
    conversations_dir: str = "/data/ralph/conversations"

    # Docker sandbox
    sandbox_base_image: str = "nikolaik/python-nodejs:python3.12-nodejs22"
    sandbox_timeout: int = 300  # 5 minutes

    class Config:
        env_prefix = "RALPH_"


settings = Settings()
```

#### 3. Add dependencies
**File**: `pyproject.toml` (add to existing)
```toml
# In [project.optional-dependencies]
ralph = [
    "openhands-sdk>=1.2.0",
    "openhands-tools>=1.2.0",
    "honcho>=0.1.0",
    "pydantic-settings>=2.0.0",
]
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv sync --extra ralph` installs dependencies
- [ ] `uv run python -c "from ralph.config import settings; print(settings.openrouter_model)"` works
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] Directory structure matches spec

---

## Phase 2: Honcho Integration

### Overview
Simplified Honcho client for message persistence and dialectic queries. Lighter than existing `youlab_server/honcho/client.py`.

### Changes Required:

#### 1. Honcho client
**File**: `ralph/honcho.py`
```python
"""Honcho message persistence and dialectic queries."""

from __future__ import annotations

import asyncio
from dataclasses import dataclass
from typing import TYPE_CHECKING

import structlog

if TYPE_CHECKING:
    from honcho import Honcho

from ralph.config import settings

log = structlog.get_logger()


@dataclass
class DialecticResponse:
    """Response from Honcho dialectic query."""
    insight: str
    query: str


class HonchoClient:
    """
    Minimal Honcho client for Ralph.

    Architecture:
    - Workspace: settings.honcho_workspace_id
    - Student peer: "student_{user_id}"
    - Tutor peer: "tutor"
    - Session: "chat_{chat_id}"
    """

    def __init__(self) -> None:
        self._client: Honcho | None = None
        self._initialized = False

    @property
    def client(self) -> Honcho | None:
        """Lazy-load Honcho client."""
        if self._client is None and not self._initialized:
            self._initialized = True
            try:
                from honcho import Honcho

                if settings.honcho_environment in ("demo", "local"):
                    self._client = Honcho(
                        workspace_id=settings.honcho_workspace_id,
                        environment=settings.honcho_environment,
                    )
                else:
                    self._client = Honcho(
                        workspace_id=settings.honcho_workspace_id,
                        api_key=settings.honcho_api_key,
                        environment=settings.honcho_environment,
                    )
                log.info("honcho_initialized", workspace=settings.honcho_workspace_id)
            except Exception as e:
                log.warning("honcho_init_failed", error=str(e))
        return self._client

    async def persist_message(
        self,
        user_id: str,
        chat_id: str,
        message: str,
        is_user: bool,
    ) -> None:
        """Persist a message to Honcho."""
        if self.client is None:
            return

        try:
            peer_id = f"student_{user_id}" if is_user else "tutor"
            peer = self.client.peer(peer_id)
            session = self.client.session(f"chat_{chat_id}")

            metadata = {"chat_id": chat_id, "user_id": user_id}
            session.add_messages([peer.message(message, metadata=metadata)])

            log.debug("message_persisted", user_id=user_id, chat_id=chat_id, is_user=is_user)
        except Exception as e:
            log.warning("persist_failed", error=str(e), user_id=user_id)

    async def query_dialectic(self, user_id: str, question: str) -> DialecticResponse | None:
        """Query Honcho for insights about a student."""
        if self.client is None:
            return None

        try:
            peer = self.client.peer(f"student_{user_id}")
            response = peer.chat(question)

            if response is None:
                return None

            insight = response if isinstance(response, str) else str(getattr(response, "content", response))

            log.info("dialectic_queried", user_id=user_id, question=question[:50])
            return DialecticResponse(insight=insight, query=question)
        except Exception as e:
            log.warning("dialectic_failed", error=str(e), user_id=user_id)
            return None


# Singleton instance
_honcho: HonchoClient | None = None


def get_honcho() -> HonchoClient:
    """Get or create Honcho client singleton."""
    global _honcho
    if _honcho is None:
        _honcho = HonchoClient()
    return _honcho


def persist_message_fire_and_forget(
    user_id: str,
    chat_id: str,
    message: str,
    is_user: bool,
) -> None:
    """Fire-and-forget message persistence."""
    if not chat_id:
        return

    async def _persist():
        await get_honcho().persist_message(user_id, chat_id, message, is_user)

    try:
        loop = asyncio.get_running_loop()
        loop.create_task(_persist())
    except RuntimeError:
        # No running loop, run synchronously
        asyncio.run(_persist())
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv run python -c "from ralph.honcho import get_honcho; print(get_honcho())"` works
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] With Honcho running, messages persist correctly
- [ ] Dialectic queries return insights

---

## Phase 3: OpenHands Client

### Overview
Wrapper around OpenHands SDK that manages workspaces per user and conversations per chat.

### Changes Required:

#### 1. OpenHands client
**File**: `ralph/openhands_client.py`
```python
"""OpenHands SDK wrapper for Ralph."""

from __future__ import annotations

import os
from pathlib import Path
from typing import TYPE_CHECKING, Callable

import structlog

if TYPE_CHECKING:
    from openhands.sdk import Conversation

from ralph.config import settings

log = structlog.get_logger()


class OpenHandsManager:
    """
    Manages OpenHands workspaces and conversations.

    Architecture:
    - One workspace per user: {user_data_dir}/{user_id}/workspace/
    - One conversation per chat: {conversations_dir}/{user_id}/{chat_id}/
    - Workspace persists across chats (files survive)
    - Conversation is fresh per chat (context resets)
    """

    def __init__(self) -> None:
        self._conversations: dict[str, Conversation] = {}  # chat_id -> Conversation

    def _get_workspace_path(self, user_id: str) -> Path:
        """Get or create user workspace directory."""
        workspace = Path(settings.user_data_dir) / user_id / "workspace"
        workspace.mkdir(parents=True, exist_ok=True)
        return workspace

    def _get_conversation_path(self, user_id: str, chat_id: str) -> Path:
        """Get conversation persistence directory."""
        conv_dir = Path(settings.conversations_dir) / user_id / chat_id
        conv_dir.mkdir(parents=True, exist_ok=True)
        return conv_dir

    def get_or_create_conversation(
        self,
        user_id: str,
        chat_id: str,
        callback: Callable | None = None,
    ) -> Conversation:
        """
        Get existing conversation or create new one.

        Each chat_id maps to one conversation. Workspace is shared across
        all conversations for a user.
        """
        if chat_id in self._conversations:
            return self._conversations[chat_id]

        from openhands.sdk import LLM, Agent, Conversation, Tool
        from openhands.tools.terminal import TerminalTool
        from openhands.tools.file_editor import FileEditorTool

        # Create LLM with OpenRouter
        llm = LLM(
            model=settings.openrouter_model,
            api_key=settings.openrouter_api_key,
            base_url="https://openrouter.ai/api/v1",
        )

        # Create agent with standard tools + custom query_honcho
        from ralph.tools.query_honcho import QueryHonchoTool

        agent = Agent(
            llm=llm,
            tools=[
                Tool(name=TerminalTool.name),
                Tool(name=FileEditorTool.name),
                Tool(name=QueryHonchoTool.name),
            ],
        )

        # Create conversation with persistence
        workspace_path = self._get_workspace_path(user_id)
        persistence_path = self._get_conversation_path(user_id, chat_id)

        callbacks = [callback] if callback else []

        conversation = Conversation(
            agent=agent,
            workspace=str(workspace_path),
            persistence_dir=str(persistence_path),
            conversation_id=chat_id,
            callbacks=callbacks,
        )

        # Store user context for query_honcho tool
        from ralph.tools.query_honcho import set_user_context
        set_user_context(chat_id, user_id)

        self._conversations[chat_id] = conversation
        log.info(
            "conversation_created",
            user_id=user_id,
            chat_id=chat_id,
            workspace=str(workspace_path),
        )

        return conversation

    def cleanup_conversation(self, chat_id: str) -> None:
        """Remove conversation from memory (persistence remains on disk)."""
        if chat_id in self._conversations:
            del self._conversations[chat_id]
            log.debug("conversation_cleaned", chat_id=chat_id)


# Singleton instance
_manager: OpenHandsManager | None = None


def get_manager() -> OpenHandsManager:
    """Get or create OpenHands manager singleton."""
    global _manager
    if _manager is None:
        _manager = OpenHandsManager()
    return _manager
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv run python -c "from ralph.openhands_client import get_manager; print(get_manager())"` works
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] Conversation creates workspace directory
- [ ] Files persist in workspace across conversations

---

## Phase 4: Query Honcho Tool

### Overview
Custom OpenHands tool that lets the agent query Honcho for student insights.

### Changes Required:

#### 1. Tool directory
**File**: `ralph/tools/__init__.py`
```python
"""Custom tools for Ralph OpenHands agent."""

from ralph.tools.query_honcho import QueryHonchoTool

__all__ = ["QueryHonchoTool"]
```

#### 2. Query Honcho tool
**File**: `ralph/tools/query_honcho.py`
```python
"""Query Honcho tool for OpenHands agent."""

from __future__ import annotations

import asyncio
from typing import ClassVar

import structlog
from openhands.tools.base import BaseTool

from ralph.honcho import get_honcho

log = structlog.get_logger()

# Chat ID -> User ID mapping (set when conversation created)
_user_context: dict[str, str] = {}


def set_user_context(chat_id: str, user_id: str) -> None:
    """Set user context for a conversation."""
    _user_context[chat_id] = user_id


class QueryHonchoTool(BaseTool):
    """
    Query Honcho for insights about the current student.

    Use this to understand:
    - Student learning patterns and preferences
    - Historical context from past conversations
    - What approaches have worked before
    - Student's current struggles or goals
    """

    name: ClassVar[str] = "query_honcho"
    description: ClassVar[str] = """Query for insights about the current student based on their conversation history.

    Args:
        question: Natural language question about the student.
                  Examples:
                  - "What learning style works best for this student?"
                  - "What has this student been working on recently?"
                  - "What does this student struggle with?"
                  - "What motivates this student?"

    Returns:
        Insight about the student based on their conversation history,
        or an error message if unavailable.
    """

    def __call__(self, question: str, **kwargs) -> str:
        """Execute the tool."""
        # Get user_id from context
        # OpenHands passes conversation context, we need to extract chat_id
        chat_id = kwargs.get("conversation_id")
        user_id = _user_context.get(chat_id) if chat_id else None

        if not user_id:
            # Fallback: try to find any user context (single user case)
            if len(_user_context) == 1:
                user_id = list(_user_context.values())[0]
            else:
                return "Unable to identify current student. Cannot query history."

        # Query Honcho
        try:
            loop = asyncio.get_event_loop()
        except RuntimeError:
            loop = asyncio.new_event_loop()
            asyncio.set_event_loop(loop)

        result = loop.run_until_complete(
            get_honcho().query_dialectic(user_id, question)
        )

        if result is None:
            return "No conversation history available yet. This may be a new student."

        return result.insight
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv run python -c "from ralph.tools import QueryHonchoTool; print(QueryHonchoTool.name)"` works
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] Tool returns insights when conversation history exists
- [ ] Tool gracefully handles new students with no history

---

## Phase 5: OpenWebUI Pipe

### Overview
The main entry point - OpenWebUI Pipe that routes messages to OpenHands with full streaming.

### Changes Required:

#### 1. OpenWebUI Pipe
**File**: `ralph/pipe.py`
```python
"""
title: Ralph Wiggum
description: Claude Code-like experience with OpenHands sandbox
version: 0.1.0
"""

from collections.abc import AsyncGenerator, Awaitable, Callable
from typing import Any

import structlog

log = structlog.get_logger()


class Pipe:
    """OpenWebUI Pipe for Ralph - OpenHands integration with streaming."""

    class Valves:
        """Configuration exposed in OpenWebUI admin."""

        def __init__(self):
            self.ENABLE_LOGGING: bool = True

    def __init__(self) -> None:
        self.name = "Ralph Wiggum"
        self.valves = self.Valves()
        self._response_buffer: str = ""

    async def on_startup(self) -> None:
        """Initialize on startup."""
        if self.valves.ENABLE_LOGGING:
            print("Ralph Pipe started")

    async def on_shutdown(self) -> None:
        """Cleanup on shutdown."""
        if self.valves.ENABLE_LOGGING:
            print("Ralph Pipe stopped")

    async def pipe(
        self,
        body: dict[str, Any],
        __user__: dict[str, Any] | None = None,
        __metadata__: dict[str, Any] | None = None,
        __event_emitter__: Callable[[dict[str, Any]], Awaitable[None]] | None = None,
    ) -> str:
        """Process a message with streaming."""
        from ralph.openhands_client import get_manager
        from ralph.honcho import persist_message_fire_and_forget

        # Extract message
        messages = body.get("messages", [])
        user_message = messages[-1].get("content", "") if messages else ""

        if not user_message:
            return "Error: No message provided."

        # Extract user info
        user_id = __user__.get("id") if __user__ else None

        if not user_id:
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": "Error: Could not identify user. Please log in."},
                })
            return ""

        # Get chat context
        chat_id = __metadata__.get("chat_id") if __metadata__ else None

        if not chat_id:
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": "Error: No chat context available."},
                })
            return ""

        if self.valves.ENABLE_LOGGING:
            print(f"Ralph: user={user_id}, chat={chat_id}")

        # Persist user message
        persist_message_fire_and_forget(user_id, chat_id, user_message, is_user=True)

        # Reset response buffer
        self._response_buffer = ""

        # Create streaming callback
        async def stream_callback(event):
            """Handle OpenHands events and stream to OpenWebUI."""
            if __event_emitter__ is None:
                return

            event_type = getattr(event, "type", None)

            if event_type == "message" and hasattr(event, "message"):
                # Agent message - stream it
                content = event.message
                self._response_buffer += content
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": content},
                })

            elif event_type == "observation":
                # Tool output - show as status
                if hasattr(event, "content"):
                    await __event_emitter__({
                        "type": "status",
                        "data": {
                            "description": f"Running: {event.content[:100]}...",
                            "done": False,
                        },
                    })

        try:
            # Get or create conversation
            manager = get_manager()
            conversation = manager.get_or_create_conversation(
                user_id=user_id,
                chat_id=chat_id,
                callback=stream_callback,
            )

            # Send message and run
            if __event_emitter__:
                await __event_emitter__({
                    "type": "status",
                    "data": {"description": "Thinking...", "done": False},
                })

            conversation.send_message(user_message)

            # Run conversation (this will trigger callbacks)
            import asyncio
            await asyncio.to_thread(conversation.run)

            # Mark complete
            if __event_emitter__:
                await __event_emitter__({
                    "type": "status",
                    "data": {"description": "Complete", "done": True},
                })

            # Persist agent response
            if self._response_buffer:
                persist_message_fire_and_forget(
                    user_id, chat_id, self._response_buffer, is_user=False
                )

        except Exception as e:
            error_msg = str(e)
            if self.valves.ENABLE_LOGGING:
                print(f"Ralph error: {error_msg}")
            if __event_emitter__:
                await __event_emitter__({
                    "type": "message",
                    "data": {"content": f"Error: {error_msg}"},
                })

        return ""
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv run python -c "from ralph import Pipe; p = Pipe(); print(p.name)"` works
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] Copy `ralph/pipe.py` to OpenWebUI pipelines
- [ ] Send message, see streaming response
- [ ] Run command (e.g., "list files in current directory"), verify execution
- [ ] Create file in one chat, start new chat, verify file exists
- [ ] After some conversation, ask agent to use query_honcho, verify it works

---

## Phase 6: Docker Sandbox (Optional Enhancement)

### Overview
Switch from LocalWorkspace to DockerWorkspace for true isolation. This is optional for MVP but recommended for multi-user.

### Changes Required:

#### 1. Update OpenHands client for Docker
**File**: `ralph/openhands_client.py` (modify `get_or_create_conversation`)

```python
# Add to imports
from openhands.workspace import DockerWorkspace

# In get_or_create_conversation, replace workspace creation:

# For Docker sandbox (production)
if settings.use_docker_sandbox:
    workspace = DockerWorkspace(
        base_image=settings.sandbox_base_image,
        host_port=settings.sandbox_port_base + hash(user_id) % 1000,
        volumes={
            str(self._get_workspace_path(user_id)): "/workspace:rw",
        },
    )
else:
    workspace = str(self._get_workspace_path(user_id))
```

#### 2. Add config options
**File**: `ralph/config.py` (add):
```python
use_docker_sandbox: bool = False
sandbox_port_base: int = 9000
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] With `RALPH_USE_DOCKER_SANDBOX=true`, commands run in container
- [ ] User cannot access files outside their workspace
- [ ] Container cleanup happens properly

---

## Testing Strategy

### Unit Tests:
- HonchoClient initialization and error handling
- OpenHandsManager workspace path generation
- QueryHonchoTool user context lookup

### Integration Tests:
- Full message flow: Pipe → OpenHands → response
- Honcho persistence and retrieval
- Workspace persistence across conversations

### Manual Testing Steps:
1. Start OpenWebUI with Ralph pipe installed
2. Login as test user
3. Send "Hello, who are you?"
4. Send "Create a file called test.py with a hello world program"
5. Verify file exists: "List files in current directory"
6. Start new chat
7. Send "What files exist in my workspace?" - verify test.py is there
8. Send "What have I been working on?" - agent should use query_honcho
9. Verify Honcho has conversation history

---

## Migration Notes

This is greenfield - no migration needed. The `ralph/` directory is completely independent from existing `src/youlab_server/`.

To switch from existing YouLab to Ralph:
1. Update OpenWebUI pipeline to use `ralph/pipe.py` instead of `letta_pipe.py`
2. Set environment variables (RALPH_OPENROUTER_API_KEY, etc.)
3. Ensure Docker is available if using sandbox mode

---

## References

- OpenHands SDK: https://docs.openhands.dev/sdk
- OpenHands Conversation API: https://docs.openhands.dev/sdk/api-reference/openhands.sdk.conversation
- Honcho Python SDK: https://github.com/plastic-labs/honcho
- Existing pipe: `src/youlab_server/pipelines/letta_pipe.py`
- Existing Honcho client: `src/youlab_server/honcho/client.py`
