# Phase 1: Combined Implementation Plan (TDD + HTTP Service)

## Overview

This document combines the TDD plan and HTTP service plan into a single actionable implementation guide. Tests are written and stubs are in place. Implementation will make the failing tests pass.

**Source Plans:**
- `thoughts/shared/plans/2025-12-29-phase-1-tdd-plan.md` — Test structure and TDD workflow
- `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` — Implementation specifications

## Current State Summary

### What's Already Working (Tests Pass)
| Component | File | Status |
|-----------|------|--------|
| Memory blocks | `memory/blocks.py` | ✅ Fully implemented |
| Pipeline (Pipe) | `pipelines/letta_pipe.py` | ✅ Fully implemented |
| Schemas | `server/schemas.py` | ✅ Fully implemented |
| Tracing | `server/tracing.py` | ✅ Fully implemented |
| Config | `config/settings.py` | ✅ ServiceSettings added |
| AgentTemplate model | `agents/templates.py` | ✅ Model class works |

### What Needs Implementation (Tests Fail)
| Component | File | Failing Tests |
|-----------|------|---------------|
| TUTOR_TEMPLATE | `agents/templates.py` | 6 tests |
| AgentTemplateRegistry | `agents/templates.py` | 9 tests |
| AgentManager | `server/agents.py` | 18 tests |
| FastAPI endpoints | `server/main.py` | 16 tests |
| CLI entry point | `server/cli.py` | 0 tests (no tests) |

### Test Summary
- **Total tests:** 105
- **Passing:** 58
- **Failing:** 47 (all due to `NotImplementedError` stubs)

---

## Implementation Order

### Step 1: Agent Templates (`agents/templates.py`)
**Failing tests:** 15

Implement TUTOR_TEMPLATE and AgentTemplateRegistry methods.

#### Code to Implement

Replace the stubs with:

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
    human: HumanBlock = Field(
        default_factory=HumanBlock, description="Initial human block"
    )


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

#### Success Criteria
- [ ] `pytest tests/test_templates.py -v` — All 20 tests pass

---

### Step 2: Agent Manager (`server/agents.py`)
**Failing tests:** 18

Implement the AgentManager class that manages Letta agents.

#### Code to Implement

Replace the stubs with the full implementation from the HTTP service plan:

```python
"""Agent management for the HTTP service."""

import structlog
from letta import create_client

from letta_starter.agents.templates import templates

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

#### Success Criteria
- [ ] `pytest tests/test_server/test_agents.py -v` — All 18 tests pass

---

### Step 3: FastAPI Endpoints (`server/main.py`)
**Failing tests:** 16

Implement the full FastAPI application with all endpoints.

#### Code to Implement

Replace the stubs with the full implementation from the HTTP service plan:

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
from letta_starter.server.tracing import trace_chat, trace_generation

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

#### Success Criteria
- [ ] `pytest tests/test_server/test_endpoints.py -v` — All 16 tests pass

---

### Step 4: CLI Entry Point (`server/cli.py`)
**Failing tests:** 0 (no tests)

Implement the CLI entry point for running the server.

#### Code to Implement

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

#### Success Criteria
- [ ] `uv run letta-server` starts without error

---

### Step 5: Fix Minor Issues

#### 5.1 Fix Pipe Timeout Test
The test `test_timeout_handling` expects "timed out" but the error path goes through `_ensure_agent_exists` which returns a different message. Update the test or the implementation.

**Option A** (fix test): Change assertion to match actual behavior
**Option B** (fix impl): Add specific timeout handling in `_ensure_agent_exists`

#### 5.2 Fix Lint Issues
Run `make lint-fix` to auto-fix the lint issues in the codebase.

---

## Verification Checklist

### Automated Verification (run after each step)
```bash
# After Step 1
pytest tests/test_templates.py -v --no-cov

# After Step 2
pytest tests/test_server/test_agents.py -v --no-cov

# After Step 3
pytest tests/test_server/test_endpoints.py -v --no-cov

# After Step 4
uv run letta-server &
curl localhost:8100/health
kill %1

# Final verification
make verify
```

### Manual Verification (after all steps)
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
     -d '{"agent_id": "<agent_id>", "message": "Hello, who are you?"}'
   ```
5. Verify in Letta that agent exists with correct name and metadata

---

## Summary

| Step | Component | Tests | Estimated Complexity |
|------|-----------|-------|---------------------|
| 1 | Agent Templates | 15 failing → pass | Low |
| 2 | Agent Manager | 18 failing → pass | Medium |
| 3 | FastAPI Endpoints | 16 failing → pass | Medium |
| 4 | CLI Entry Point | N/A | Low |
| 5 | Fix Minor Issues | 1 failing + lint | Low |

**Implementation ready!** All tests are written, all stubs are in place. Just need to fill in the implementations following the code in each step.
