# Phase 1: LettaStarter HTTP Service Implementation Plan

## Overview

Convert LettaStarter from a library imported by the Pipe to a standalone HTTP service. The Pipe becomes a thin forwarder that extracts user/chat context from OpenWebUI and forwards requests to the service. This establishes the foundation for all subsequent phases.

## Current State Analysis

### What Exists
- `src/letta_starter/pipelines/letta_pipe.py` - Pipe that imports Letta directly
- `src/letta_starter/memory/blocks.py` - PersonaBlock/HumanBlock schemas
- `src/letta_starter/agents/` - BaseAgent, factory functions, AgentRegistry
- `src/letta_starter/config/settings.py` - Pydantic settings
- `src/letta_starter/observability/` - structlog, metrics, Langfuse

### What's Missing
- HTTP service (`server.py`)
- Agent template system
- Updated Pipe that calls HTTP service
- Service configuration (host, port, API key)

### Key Discoveries from OpenWebUI Research

**Chat title access** (from `OpenWebUI/open-webui/backend/open_webui/models/chats.py:706-719`):
```python
from open_webui.models.chats import Chats
chat = Chats.get_chat_by_id(chat_id)  # Returns ChatModel with .title
```

**Pipe receives** (from `OpenWebUI/open-webui/backend/open_webui/functions.py:254-278`):
- `body` - OpenAI-format request (messages, model, stream)
- `__user__` - Full user dict (id, email, name, role, settings)
- `__metadata__` - Request context (chat_id, message_id, session_id, user_id)
- `__request__` - FastAPI Request object

**Important**: Chats with IDs starting with `"local:"` are temporary and not in database.

## Desired End State

A working HTTP service where:
1. Pipe extracts user/chat context and forwards to service
2. Service manages agent lifecycle (create, lookup, message)
3. Agent templates define different agent types (tutor, etc.)
4. All state lives in Letta (service is stateless with in-memory cache)
5. Observability integrated (structlog + Langfuse)

### Verification
- Start service with `uv run letta-server`
- Pipe successfully forwards messages to service
- Agents created on demand via explicit endpoint
- Health endpoint returns Letta connection status
- Langfuse traces show request flow

## What We're NOT Doing

- Honcho integration (Phase 3)
- Thread context parsing (Phase 4) - we pass chat_title but don't parse it yet
- Curriculum loading (Phase 5)
- Background processing (Phase 6)
- Onboarding flow (Phase 7)
- Streaming responses (add later if needed)
- Authentication (trust localhost for pilot)

---

## Implementation Approach

### Architecture

```
OpenWebUI
    │
    ▼
┌─────────────────────────────────────────┐
│ Pipe (letta_pipe.py)                    │
│ - Extract __user__["id"], __metadata__  │
│ - Query chat title from Chats model     │
│ - HTTP POST to LettaStarter service     │
└─────────────────────────────────────────┘
    │
    ▼ HTTP (localhost:8100)
┌─────────────────────────────────────────┐
│ LettaStarter HTTP Service (server.py)   │
│ - POST /agents - Create agent           │
│ - GET /agents - List/lookup agents      │
│ - POST /chat - Send message             │
│ - GET /health - Service health          │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ Letta Server (localhost:8283)           │
│ - Agent storage                         │
│ - Memory blocks                         │
│ - LLM calls                             │
└─────────────────────────────────────────┘
```

### Agent Naming Convention

```
Agent name: youlab_{user_id}_{agent_type}
Agent metadata: {
    "youlab_user_id": "uuid-string",
    "youlab_agent_type": "tutor"
}
```

This allows:
- Discovering all YouLab agents by prefix
- Rebuilding cache from Letta on service restart
- Multiple agent types per user

---

## Phase 1.1: Agent Template System

### Overview
Create the template system that defines how different agent types are configured.

### Changes Required

#### 1. Agent Templates Module
**File**: `src/letta_starter/agents/templates.py` (new)

```python
"""Agent templates for YouLab."""

from pydantic import BaseModel, Field

from letta_starter.memory.blocks import HumanBlock, PersonaBlock


class AgentTemplate(BaseModel):
    """Template for creating agents of a specific type."""

    type_id: str = Field(..., description="Unique identifier for this template")
    display_name: str = Field(..., description="Human-readable name")
    description: str = Field(default="", description="Template description")
    persona: PersonaBlock = Field(..., description="Persona block for the agent")
    human: HumanBlock = Field(default_factory=HumanBlock, description="Initial human block")


# Default tutor template for college essay coaching
TUTOR_TEMPLATE = AgentTemplate(
    type_id="tutor",
    display_name="College Essay Coach",
    description="Primary tutor for college essay writing course",
    persona=PersonaBlock(
        name="YouLab Essay Coach",
        role="AI tutor specializing in college application essays",
        capabilities=[
            "Guide students through self-discovery exercises",
            "Help brainstorm and develop essay topics",
            "Provide constructive feedback on drafts",
            "Support emotional journey of college applications",
        ],
        expertise=[
            "College admissions",
            "Personal narrative",
            "Reflective writing",
            "Strengths-based coaching",
        ],
        tone="warm",
        verbosity="adaptive",
        constraints=[
            "Never write essays for students",
            "Always ask clarifying questions before giving advice",
            "Celebrate small wins and progress",
        ],
    ),
    human=HumanBlock(),  # Empty, filled during onboarding
)


class AgentTemplateRegistry:
    """Registry for agent templates."""

    def __init__(self) -> None:
        self._templates: dict[str, AgentTemplate] = {}
        # Register built-in templates
        self.register(TUTOR_TEMPLATE)

    def register(self, template: AgentTemplate) -> None:
        """Register an agent template."""
        self._templates[template.type_id] = template

    def get(self, type_id: str) -> AgentTemplate | None:
        """Get template by type ID."""
        return self._templates.get(type_id)

    def list_types(self) -> list[str]:
        """List all registered template type IDs."""
        return list(self._templates.keys())

    def get_all(self) -> dict[str, AgentTemplate]:
        """Get all registered templates."""
        return self._templates.copy()


# Global registry instance
templates = AgentTemplateRegistry()
```

**Note**: The existing `PersonaBlock` in `blocks.py` already has all needed fields:
- `name`, `role`, `capabilities`, `tone`, `verbosity`, `constraints`, `expertise`

No changes needed to `blocks.py`.

### Success Criteria

#### Automated Verification
- [ ] `from letta_starter.agents.templates import templates` imports successfully
- [ ] `templates.get("tutor")` returns TUTOR_TEMPLATE
- [ ] `templates.list_types()` returns `["tutor"]`
- [ ] `make check` passes

---

## Phase 1.2: HTTP Service Core

### Overview
Create the FastAPI service with core endpoints.

### Changes Required

#### 1. Add Dependencies
**File**: `pyproject.toml`

Add to dependencies (around line 7):
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
]
```

#### 2. Add Service Configuration
**File**: `src/letta_starter/config/settings.py`

Add new settings class (append to file):
```python
class ServiceSettings(BaseSettings):
    """Settings for the HTTP service."""

    model_config = SettingsConfigDict(
        env_prefix="YOULAB_SERVICE_",
        env_file=".env",
        extra="ignore",
    )

    host: str = "127.0.0.1"
    port: int = 8100
    api_key: str | None = None  # Optional, for future auth
    log_level: str = "INFO"

    # Letta connection (inherit from existing or define here)
    letta_base_url: str = "http://localhost:8283"
```

#### 3. Create Request/Response Schemas
**File**: `src/letta_starter/server/schemas.py` (new)

```python
"""Request and response schemas for the HTTP service."""

from pydantic import BaseModel, Field


# Agent schemas
class CreateAgentRequest(BaseModel):
    """Request to create a new agent."""

    user_id: str = Field(..., description="OpenWebUI user ID")
    agent_type: str = Field(default="tutor", description="Agent template type")
    user_name: str | None = Field(default=None, description="User's display name")


class AgentResponse(BaseModel):
    """Agent information response."""

    agent_id: str
    user_id: str
    agent_type: str
    agent_name: str
    created_at: int | None = None


class AgentListResponse(BaseModel):
    """List of agents response."""

    agents: list[AgentResponse]


# Chat schemas
class ChatRequest(BaseModel):
    """Request to send a message."""

    agent_id: str = Field(..., description="Letta agent ID")
    message: str = Field(..., description="User message content")
    chat_id: str | None = Field(default=None, description="OpenWebUI chat ID")
    chat_title: str | None = Field(default=None, description="OpenWebUI chat title")


class ChatResponse(BaseModel):
    """Chat response."""

    response: str
    agent_id: str


# Health schemas
class HealthResponse(BaseModel):
    """Health check response."""

    status: str
    letta_connected: bool
    version: str = "0.1.0"
```

#### 4. Create Agent Manager
**File**: `src/letta_starter/server/agents.py` (new)

```python
"""Agent management for the HTTP service."""

import structlog
from letta import create_client

from letta_starter.agents.templates import AgentTemplate, templates

log = structlog.get_logger()


class AgentManager:
    """Manages Letta agents for YouLab users."""

    def __init__(self, letta_base_url: str) -> None:
        self.letta_base_url = letta_base_url
        self._client = None
        # Cache: (user_id, agent_type) -> agent_id
        self._cache: dict[tuple[str, str], str] = {}

    @property
    def client(self):
        """Lazy-initialize Letta client."""
        if self._client is None:
            self._client = create_client(base_url=self.letta_base_url)
        return self._client

    def _agent_name(self, user_id: str, agent_type: str) -> str:
        """Generate agent name from user_id and type."""
        return f"youlab_{user_id}_{agent_type}"

    def _agent_metadata(self, user_id: str, agent_type: str) -> dict:
        """Generate agent metadata."""
        return {
            "youlab_user_id": user_id,
            "youlab_agent_type": agent_type,
        }

    async def rebuild_cache(self) -> int:
        """Rebuild cache from Letta on startup. Returns count of agents found."""
        agents = self.client.list_agents()
        count = 0
        for agent in agents:
            if agent.name and agent.name.startswith("youlab_"):
                meta = agent.metadata or {}
                user_id = meta.get("youlab_user_id")
                agent_type = meta.get("youlab_agent_type", "tutor")
                if user_id:
                    self._cache[(user_id, agent_type)] = agent.id
                    count += 1
        log.info("rebuilt_agent_cache", count=count)
        return count

    def get_agent_id(self, user_id: str, agent_type: str = "tutor") -> str | None:
        """Get agent ID from cache, or lookup in Letta."""
        cache_key = (user_id, agent_type)

        # Check cache first
        if cache_key in self._cache:
            return self._cache[cache_key]

        # Lookup in Letta by name
        agent_name = self._agent_name(user_id, agent_type)
        agents = self.client.list_agents()
        for agent in agents:
            if agent.name == agent_name:
                self._cache[cache_key] = agent.id
                return agent.id

        return None

    def create_agent(
        self,
        user_id: str,
        agent_type: str = "tutor",
        user_name: str | None = None,
    ) -> str:
        """Create a new agent from template. Returns agent_id."""
        # Check if already exists
        existing = self.get_agent_id(user_id, agent_type)
        if existing:
            log.info("agent_already_exists", user_id=user_id, agent_type=agent_type)
            return existing

        # Get template
        template = templates.get(agent_type)
        if template is None:
            raise ValueError(f"Unknown agent type: {agent_type}")

        # Create agent
        agent_name = self._agent_name(user_id, agent_type)
        metadata = self._agent_metadata(user_id, agent_type)

        # Customize human block with user name if provided
        human_block = template.human
        if user_name:
            human_block = template.human.model_copy()
            human_block.name = user_name

        agent = self.client.create_agent(
            name=agent_name,
            memory={
                "persona": template.persona.to_memory_string(),
                "human": human_block.to_memory_string(),
            },
            metadata=metadata,
        )

        # Update cache
        self._cache[(user_id, agent_type)] = agent.id
        log.info(
            "agent_created",
            agent_id=agent.id,
            user_id=user_id,
            agent_type=agent_type,
        )
        return agent.id

    def get_agent_info(self, agent_id: str) -> dict | None:
        """Get agent information by ID."""
        try:
            agent = self.client.get_agent(agent_id)
            meta = agent.metadata or {}
            return {
                "agent_id": agent.id,
                "user_id": meta.get("youlab_user_id", ""),
                "agent_type": meta.get("youlab_agent_type", "tutor"),
                "agent_name": agent.name,
                "created_at": getattr(agent, "created_at", None),
            }
        except Exception:
            return None

    def list_user_agents(self, user_id: str) -> list[dict]:
        """List all agents for a user."""
        results = []
        agents = self.client.list_agents()
        for agent in agents:
            meta = agent.metadata or {}
            if meta.get("youlab_user_id") == user_id:
                results.append({
                    "agent_id": agent.id,
                    "user_id": user_id,
                    "agent_type": meta.get("youlab_agent_type", "tutor"),
                    "agent_name": agent.name,
                    "created_at": getattr(agent, "created_at", None),
                })
        return results

    def send_message(self, agent_id: str, message: str) -> str:
        """Send a message to an agent. Returns response text."""
        response = self.client.send_message(
            agent_id=agent_id,
            message=message,
            role="user",
        )
        return self._extract_response(response)

    def _extract_response(self, response) -> str:
        """Extract text from Letta response object."""
        texts = []
        if hasattr(response, "messages"):
            for msg in response.messages:
                if hasattr(msg, "assistant_message") and msg.assistant_message:
                    texts.append(msg.assistant_message)
                elif hasattr(msg, "text") and msg.text:
                    texts.append(msg.text)
        return "\n".join(texts) if texts else ""

    def check_letta_connection(self) -> bool:
        """Check if Letta server is reachable."""
        try:
            self.client.list_agents()
            return True
        except Exception:
            return False
```

#### 5. Create Main Server
**File**: `src/letta_starter/server/main.py` (new)

```python
"""FastAPI HTTP service for LettaStarter."""

import structlog
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException, status

from letta_starter.config.settings import ServiceSettings
from letta_starter.server.agents import AgentManager
from letta_starter.server.schemas import (
    AgentListResponse,
    AgentResponse,
    ChatRequest,
    ChatResponse,
    CreateAgentRequest,
    HealthResponse,
)

log = structlog.get_logger()
settings = ServiceSettings()


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan handler."""
    # Startup
    log.info("starting_service", host=settings.host, port=settings.port)
    app.state.agent_manager = AgentManager(letta_base_url=settings.letta_base_url)

    # Rebuild cache from Letta
    try:
        count = await app.state.agent_manager.rebuild_cache()
        log.info("startup_complete", cached_agents=count)
    except Exception as e:
        log.warning("letta_not_available_at_startup", error=str(e))

    yield

    # Shutdown
    log.info("shutting_down_service")


app = FastAPI(
    title="LettaStarter Service",
    description="HTTP service for YouLab AI tutoring",
    version="0.1.0",
    lifespan=lifespan,
)


def get_agent_manager() -> AgentManager:
    """Get the agent manager from app state."""
    return app.state.agent_manager


# Health endpoint
@app.get("/health", response_model=HealthResponse)
async def health():
    """Check service health."""
    manager = get_agent_manager()
    letta_ok = manager.check_letta_connection()
    return HealthResponse(
        status="ok" if letta_ok else "degraded",
        letta_connected=letta_ok,
    )


# Agent endpoints
@app.post("/agents", response_model=AgentResponse, status_code=status.HTTP_201_CREATED)
async def create_agent(request: CreateAgentRequest):
    """Create a new agent for a user."""
    manager = get_agent_manager()

    try:
        agent_id = manager.create_agent(
            user_id=request.user_id,
            agent_type=request.agent_type,
            user_name=request.user_name,
        )
        info = manager.get_agent_info(agent_id)
        if info is None:
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Agent created but info retrieval failed",
            )
        return AgentResponse(**info)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e),
        )
    except Exception as e:
        log.exception("agent_creation_failed", error=str(e))
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Failed to create agent - Letta may be unavailable",
        )


@app.get("/agents/{agent_id}", response_model=AgentResponse)
async def get_agent(agent_id: str):
    """Get agent information by ID."""
    manager = get_agent_manager()
    info = manager.get_agent_info(agent_id)
    if info is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Agent not found: {agent_id}",
        )
    return AgentResponse(**info)


@app.get("/agents", response_model=AgentListResponse)
async def list_agents(user_id: str | None = None):
    """List agents, optionally filtered by user_id."""
    manager = get_agent_manager()

    if user_id:
        agents = manager.list_user_agents(user_id)
    else:
        # List all YouLab agents (admin use)
        agents = []
        for agent in manager.client.list_agents():
            if agent.name and agent.name.startswith("youlab_"):
                meta = agent.metadata or {}
                agents.append({
                    "agent_id": agent.id,
                    "user_id": meta.get("youlab_user_id", ""),
                    "agent_type": meta.get("youlab_agent_type", "tutor"),
                    "agent_name": agent.name,
                    "created_at": getattr(agent, "created_at", None),
                })

    return AgentListResponse(agents=[AgentResponse(**a) for a in agents])


# Chat endpoint
@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """Send a message to an agent."""
    manager = get_agent_manager()

    # Verify agent exists
    info = manager.get_agent_info(request.agent_id)
    if info is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Agent not found: {request.agent_id}",
        )

    try:
        # Log context (chat_title will be used in Phase 4)
        log.info(
            "chat_request",
            agent_id=request.agent_id,
            chat_id=request.chat_id,
            chat_title=request.chat_title,
            message_length=len(request.message),
        )

        response_text = manager.send_message(request.agent_id, request.message)

        return ChatResponse(
            response=response_text if response_text else "No response from agent.",
            agent_id=request.agent_id,
        )
    except Exception as e:
        log.exception("chat_failed", agent_id=request.agent_id, error=str(e))
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Failed to communicate with agent",
        )
```

#### 6. Create Server Entry Point
**File**: `src/letta_starter/server/__init__.py` (new)

```python
"""LettaStarter HTTP service."""

from letta_starter.server.main import app

__all__ = ["app"]
```

#### 7. Add CLI Entry Point
**File**: `pyproject.toml`

Update scripts section (around line 23):
```toml
[project.scripts]
letta-starter = "letta_starter.main:main"
letta-server = "letta_starter.server.cli:main"
```

**File**: `src/letta_starter/server/cli.py` (new)

```python
"""CLI entry point for the HTTP service."""

import uvicorn

from letta_starter.config.settings import ServiceSettings


def main() -> None:
    """Run the HTTP service."""
    settings = ServiceSettings()
    uvicorn.run(
        "letta_starter.server.main:app",
        host=settings.host,
        port=settings.port,
        log_level=settings.log_level.lower(),
        reload=False,
    )


if __name__ == "__main__":
    main()
```

### Success Criteria

#### Automated Verification
- [ ] `uv sync` installs fastapi and uvicorn
- [ ] `uv run letta-server` starts HTTP server on port 8100
- [ ] `curl localhost:8100/health` returns JSON with status
- [ ] `make check` passes (lint + typecheck)

#### Manual Verification
- [ ] Can POST to `/agents` with `{"user_id": "test123"}` and receive agent info
- [ ] Can GET `/agents?user_id=test123` and see the created agent
- [ ] Can POST to `/chat` with agent_id and message, receive response
- [ ] Server logs show structured output with request context

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the service works correctly before proceeding.

---

## Phase 1.3: Updated Pipe

### Overview
Update the OpenWebUI Pipe to be a thin forwarder to the HTTP service.

### Changes Required

#### 1. Replace Pipe Implementation
**File**: `src/letta_starter/pipelines/letta_pipe.py`

Replace entire file with:

```python
"""
OpenWebUI Pipeline for YouLab.

This Pipe forwards requests to the LettaStarter HTTP service.
It extracts user context from OpenWebUI and chat title from the database.
"""

from collections.abc import Generator, Iterator
from typing import Any

import httpx
from pydantic import BaseModel, Field


class Pipeline:
    """
    OpenWebUI Pipeline for YouLab tutoring.

    Forwards messages to the LettaStarter HTTP service.
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
            # Not running inside OpenWebUI
            return None
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to get chat title: {e}")
            return None

    def _ensure_agent_exists(
        self,
        client: httpx.Client,
        user_id: str,
        user_name: str | None = None,
    ) -> str | None:
        """Ensure agent exists for user, create if needed. Returns agent_id."""
        # Check if agent exists
        try:
            response = client.get(
                f"{self.valves.LETTA_SERVICE_URL}/agents",
                params={"user_id": user_id},
            )
            if response.status_code == 200:
                agents = response.json().get("agents", [])
                # Find agent of correct type
                for agent in agents:
                    if agent.get("agent_type") == self.valves.AGENT_TYPE:
                        return agent.get("agent_id")
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to check existing agents: {e}")

        # Create new agent
        try:
            response = client.post(
                f"{self.valves.LETTA_SERVICE_URL}/agents",
                json={
                    "user_id": user_id,
                    "agent_type": self.valves.AGENT_TYPE,
                    "user_name": user_name,
                },
            )
            if response.status_code == 201:
                return response.json().get("agent_id")
            else:
                if self.valves.ENABLE_LOGGING:
                    print(f"Failed to create agent: {response.text}")
                return None
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"Failed to create agent: {e}")
            return None

    def pipe(
        self,
        user_message: str,
        model_id: str,
        messages: list[dict[str, Any]],
        body: dict[str, Any],
        __user__: dict[str, Any] | None = None,
        __metadata__: dict[str, Any] | None = None,
    ) -> str | Generator[str, None, None] | Iterator[str]:
        """
        Process a message through the YouLab tutor.

        Args:
            user_message: The user's message text
            model_id: The selected model ID (from OpenWebUI)
            messages: Full conversation history
            body: Additional request body data
            __user__: User information from OpenWebUI
            __metadata__: Request metadata (chat_id, message_id, etc.)

        Returns:
            Agent response as string
        """
        # Extract user info
        user_id = __user__.get("id") if __user__ else None
        user_name = __user__.get("name") if __user__ else None

        if not user_id:
            return "Error: Could not identify user. Please log in."

        # Extract chat context
        chat_id = __metadata__.get("chat_id") if __metadata__ else None
        chat_title = self._get_chat_title(chat_id)

        if self.valves.ENABLE_LOGGING:
            print(f"YouLab: user={user_id}, chat={chat_id}, title={chat_title}")

        try:
            with httpx.Client(timeout=120.0) as client:
                # Ensure agent exists
                agent_id = self._ensure_agent_exists(client, user_id, user_name)
                if not agent_id:
                    return "Error: Could not create or find your tutor agent."

                # Send message
                response = client.post(
                    f"{self.valves.LETTA_SERVICE_URL}/chat",
                    json={
                        "agent_id": agent_id,
                        "message": user_message,
                        "chat_id": chat_id,
                        "chat_title": chat_title,
                    },
                )

                if response.status_code == 200:
                    return response.json().get("response", "No response from tutor.")
                else:
                    if self.valves.ENABLE_LOGGING:
                        print(f"YouLab chat error: {response.status_code} - {response.text}")
                    return f"Error communicating with tutor: {response.status_code}"

        except httpx.TimeoutException:
            return "Error: Request timed out. Please try again."
        except Exception as e:
            if self.valves.ENABLE_LOGGING:
                print(f"YouLab error: {e}")
            return f"Error: {str(e)}"


# For standalone testing
if __name__ == "__main__":
    import asyncio

    async def test() -> None:
        pipeline = Pipeline()
        await pipeline.on_startup()

        # Simulate OpenWebUI context
        response = pipeline.pipe(
            user_message="Hello! What can you help me with?",
            model_id="youlab",
            messages=[],
            body={},
            __user__={"id": "test-user-123", "name": "Test User"},
            __metadata__={"chat_id": "local:test"},
        )
        print(f"Response: {response}")

        await pipeline.on_shutdown()

    asyncio.run(test())
```

### Success Criteria

#### Automated Verification
- [ ] Pipe imports successfully: `from letta_starter.pipelines.letta_pipe import Pipeline`
- [ ] `make check` passes

#### Manual Verification
- [ ] Upload Pipe to OpenWebUI
- [ ] Send message through OpenWebUI
- [ ] Verify request flows through service (check service logs)
- [ ] Verify response appears in OpenWebUI
- [ ] Verify agent created in Letta (check via Letta API)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the full flow works end-to-end.

---

## Phase 1.4: Langfuse Integration

### Overview
Add Langfuse tracing to the HTTP service for observability.

### Changes Required

#### 1. Add Langfuse Dependency
**File**: `pyproject.toml`

Add to dependencies:
```toml
"langfuse>=2.0.0",
```

#### 2. Add Langfuse Settings
**File**: `src/letta_starter/config/settings.py`

Add to ServiceSettings:
```python
class ServiceSettings(BaseSettings):
    # ... existing fields ...

    # Langfuse (optional)
    langfuse_public_key: str | None = None
    langfuse_secret_key: str | None = None
    langfuse_host: str = "https://cloud.langfuse.com"
    langfuse_enabled: bool = True
```

#### 3. Add Tracing Middleware
**File**: `src/letta_starter/server/tracing.py` (new)

```python
"""Langfuse tracing for the HTTP service."""

import structlog
from contextlib import contextmanager
from typing import Any, Generator
from uuid import uuid4

from langfuse import Langfuse

from letta_starter.config.settings import ServiceSettings

log = structlog.get_logger()
settings = ServiceSettings()

# Initialize Langfuse client (lazy)
_langfuse: Langfuse | None = None


def get_langfuse() -> Langfuse | None:
    """Get or create Langfuse client."""
    global _langfuse

    if not settings.langfuse_enabled:
        return None

    if not settings.langfuse_public_key or not settings.langfuse_secret_key:
        return None

    if _langfuse is None:
        _langfuse = Langfuse(
            public_key=settings.langfuse_public_key,
            secret_key=settings.langfuse_secret_key,
            host=settings.langfuse_host,
        )

    return _langfuse


@contextmanager
def trace_chat(
    user_id: str,
    agent_id: str,
    chat_id: str | None = None,
    metadata: dict[str, Any] | None = None,
) -> Generator[dict[str, Any], None, None]:
    """Context manager for tracing a chat request."""
    trace_id = str(uuid4())
    context = {"trace_id": trace_id}

    langfuse = get_langfuse()
    trace = None

    if langfuse:
        try:
            trace = langfuse.trace(
                id=trace_id,
                name="chat",
                user_id=user_id,
                session_id=chat_id,
                metadata={
                    "agent_id": agent_id,
                    **(metadata or {}),
                },
            )
            context["langfuse_trace"] = trace
        except Exception as e:
            log.warning("langfuse_trace_failed", error=str(e))

    try:
        yield context
    finally:
        if trace:
            try:
                langfuse.flush()
            except Exception:
                pass


def trace_generation(
    trace_context: dict[str, Any],
    name: str,
    input_text: str,
    output_text: str,
    model: str = "letta",
) -> None:
    """Record a generation span within a trace."""
    trace = trace_context.get("langfuse_trace")
    if trace:
        try:
            trace.generation(
                name=name,
                input=input_text,
                output=output_text,
                model=model,
            )
        except Exception as e:
            log.warning("langfuse_generation_failed", error=str(e))
```

#### 4. Update Chat Endpoint with Tracing
**File**: `src/letta_starter/server/main.py`

Update the chat endpoint:
```python
from letta_starter.server.tracing import trace_chat, trace_generation

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """Send a message to an agent."""
    manager = get_agent_manager()

    # Verify agent exists
    info = manager.get_agent_info(request.agent_id)
    if info is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Agent not found: {request.agent_id}",
        )

    user_id = info.get("user_id", "unknown")

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
            )
```

### Success Criteria

#### Automated Verification
- [ ] `make check` passes
- [ ] Service starts without Langfuse credentials (graceful fallback)

#### Manual Verification
- [ ] With Langfuse credentials in `.env`, traces appear in Langfuse dashboard
- [ ] Traces show user_id, agent_id, chat_id
- [ ] Generation spans show input/output text

---

## Testing Strategy

### Unit Tests
**File**: `tests/test_server.py` (new)

```python
"""Tests for the HTTP service."""

import pytest
from fastapi.testclient import TestClient

from letta_starter.server.main import app


@pytest.fixture
def client():
    """Create test client."""
    return TestClient(app)


def test_health_endpoint(client):
    """Test health endpoint returns expected structure."""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert "status" in data
    assert "letta_connected" in data
    assert "version" in data


def test_create_agent_missing_user_id(client):
    """Test agent creation fails without user_id."""
    response = client.post("/agents", json={})
    assert response.status_code == 422  # Validation error


def test_get_nonexistent_agent(client):
    """Test getting nonexistent agent returns 404."""
    response = client.get("/agents/nonexistent-id")
    assert response.status_code == 404


def test_chat_nonexistent_agent(client):
    """Test chat with nonexistent agent returns 404."""
    response = client.post("/chat", json={
        "agent_id": "nonexistent-id",
        "message": "Hello",
    })
    assert response.status_code == 404
```

### Integration Tests (Manual)

1. Start Letta server: `letta server`
2. Start LettaStarter service: `uv run letta-server`
3. Create agent:
   ```bash
   curl -X POST http://localhost:8100/agents \
     -H "Content-Type: application/json" \
     -d '{"user_id": "test-user", "user_name": "Test User"}'
   ```
4. Send message:
   ```bash
   curl -X POST http://localhost:8100/chat \
     -H "Content-Type: application/json" \
     -d '{"agent_id": "<agent_id_from_step_3>", "message": "Hello, who are you?"}'
   ```
5. Verify in Letta that agent exists with correct name and metadata

---

## Environment Variables

**File**: `.env.example`

Add:
```bash
# LettaStarter Service
YOULAB_SERVICE_HOST=127.0.0.1
YOULAB_SERVICE_PORT=8100
YOULAB_SERVICE_API_KEY=  # Optional, for future auth
YOULAB_SERVICE_LOG_LEVEL=INFO
YOULAB_SERVICE_LETTA_BASE_URL=http://localhost:8283

# Langfuse (optional)
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com
```

---

## File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `pyproject.toml` | Edit | Add fastapi, uvicorn, langfuse deps + letta-server script |
| `src/letta_starter/config/settings.py` | Edit | Add ServiceSettings class |
| `src/letta_starter/agents/templates.py` | Create | Agent template system |
| `src/letta_starter/server/__init__.py` | Create | Server package |
| `src/letta_starter/server/schemas.py` | Create | Request/response models |
| `src/letta_starter/server/agents.py` | Create | Agent manager |
| `src/letta_starter/server/main.py` | Create | FastAPI app |
| `src/letta_starter/server/cli.py` | Create | Server CLI entry point |
| `src/letta_starter/server/tracing.py` | Create | Langfuse integration |
| `src/letta_starter/pipelines/letta_pipe.py` | Replace | Thin HTTP forwarder |
| `.env.example` | Edit | Add service config vars |
| `tests/test_server.py` | Create | Server unit tests |

**Total**: 3 edited files, 9 new files

---

## Implementation Order

1. **Phase 1.1**: Agent template system
2. **Phase 1.2**: HTTP service core (deps → config → schemas → agents → main → cli)
3. **Phase 1.3**: Updated Pipe
4. **Phase 1.4**: Langfuse integration

Each sub-phase should be tested before proceeding to the next.

---

## Impact on Downstream Phases

### Phase 2: User Identity & Routing
- Already handled: user_id extraction, agent creation
- Remaining: first-interaction onboarding trigger

### Phase 3: Honcho Integration
- Service is ready to add Honcho client
- `/chat` endpoint can persist messages to Honcho
- Add `honcho_connected` to health check

### Phase 4: Thread Context Management
- `chat_title` already passed to `/chat`
- Add context parser that reads chat_title
- Update agent memory blocks based on context

### Phase 5: Curriculum Parser
- Add `/admin/reload-curriculum` endpoint
- Context updater pulls from curriculum

### Phase 6: Background Worker
- Add `/background/run` endpoint
- Add idle detection scheduler

### Phase 7: Onboarding
- Detect first interaction in `/chat`
- Set initial context to onboarding module

---

## References

- OpenWebUI Chats model: `OpenWebUI/open-webui/backend/open_webui/models/chats.py`
- OpenWebUI Pipe interface: `OpenWebUI/open-webui/backend/open_webui/functions.py:254-278`
- Existing memory blocks: `src/letta_starter/memory/blocks.py`
- Existing agent registry: `src/letta_starter/agents/default.py`
- Project context: `thoughts/shared/youlab-project-context.md`
- Original 7-phase plan: `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md`
