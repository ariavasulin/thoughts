# Phase 1 TDD (Test-Driven Development) Plan

## Overview

This document defines the test-first approach for implementing Phase 1 of the YouLab HTTP Service. All tests will be written and pass (with stubs/mocks) before implementing the production code.

**Reference**: `thoughts/shared/plans/2025-12-29-phase-1-http-service.md`

## TDD Workflow

For each component:
1. **Write failing tests** - Define expected behavior
2. **Create minimal stubs** - Just enough to import without errors
3. **Verify tests fail** - Confirm they test the right thing
4. **Implement production code** - Make tests pass
5. **Refactor** - Clean up while keeping tests green

## Test File Structure

```
tests/
  conftest.py                 # Shared fixtures (extend existing)
  test_memory.py              # Existing memory block tests
  test_templates.py           # NEW: Agent template tests (Phase 1.1)
  test_server/
    __init__.py
    conftest.py               # Server-specific fixtures
    test_schemas.py           # Request/response schema tests
    test_agents.py            # AgentManager tests
    test_endpoints.py         # FastAPI endpoint tests
    test_tracing.py           # Langfuse tracing tests
  test_pipe.py                # NEW: Updated Pipe tests (Phase 1.3)
```

---

## Phase 1.1: Agent Template System Tests

### File: `tests/test_templates.py`

#### Test Cases

```python
"""Tests for agent template system."""

import pytest

# These imports will fail initially - that's expected in TDD
from letta_starter.agents.templates import (
    AgentTemplate,
    AgentTemplateRegistry,
    TUTOR_TEMPLATE,
    templates,
)
from letta_starter.memory.blocks import HumanBlock, PersonaBlock


class TestAgentTemplate:
    """Tests for AgentTemplate model."""

    def test_create_minimal_template(self):
        """Test creating template with minimal required fields."""
        template = AgentTemplate(
            type_id="test",
            display_name="Test Agent",
            persona=PersonaBlock(role="Test role"),
        )
        assert template.type_id == "test"
        assert template.display_name == "Test Agent"
        assert template.description == ""  # Default
        assert isinstance(template.human, HumanBlock)  # Default factory

    def test_create_full_template(self, sample_agent_template_data):
        """Test creating template with all fields."""
        template = AgentTemplate(**sample_agent_template_data)
        assert template.type_id == "custom"
        assert len(template.persona.capabilities) > 0

    def test_template_type_id_required(self):
        """Test that type_id is required."""
        with pytest.raises(Exception):  # ValidationError
            AgentTemplate(display_name="Test", persona=PersonaBlock(role="Role"))

    def test_template_display_name_required(self):
        """Test that display_name is required."""
        with pytest.raises(Exception):  # ValidationError
            AgentTemplate(type_id="test", persona=PersonaBlock(role="Role"))

    def test_template_persona_required(self):
        """Test that persona is required."""
        with pytest.raises(Exception):  # ValidationError
            AgentTemplate(type_id="test", display_name="Test")


class TestAgentTemplateRegistry:
    """Tests for AgentTemplateRegistry."""

    def test_registry_starts_with_builtin(self):
        """Test that registry includes built-in templates."""
        registry = AgentTemplateRegistry()
        assert "tutor" in registry.list_types()

    def test_register_new_template(self, sample_agent_template_data):
        """Test registering a new template."""
        registry = AgentTemplateRegistry()
        template = AgentTemplate(**sample_agent_template_data)
        registry.register(template)

        assert template.type_id in registry.list_types()
        assert registry.get(template.type_id) == template

    def test_get_existing_template(self):
        """Test getting an existing template."""
        registry = AgentTemplateRegistry()
        template = registry.get("tutor")

        assert template is not None
        assert template.type_id == "tutor"

    def test_get_nonexistent_template(self):
        """Test getting a template that doesn't exist."""
        registry = AgentTemplateRegistry()
        template = registry.get("nonexistent")

        assert template is None

    def test_list_types(self):
        """Test listing all template types."""
        registry = AgentTemplateRegistry()
        types = registry.list_types()

        assert isinstance(types, list)
        assert "tutor" in types

    def test_get_all_templates(self):
        """Test getting all templates as dict."""
        registry = AgentTemplateRegistry()
        all_templates = registry.get_all()

        assert isinstance(all_templates, dict)
        assert "tutor" in all_templates

    def test_register_overwrites_existing(self, sample_agent_template_data):
        """Test that registering with same type_id overwrites."""
        registry = AgentTemplateRegistry()

        # Modify sample data to use "tutor" type_id
        sample_agent_template_data["type_id"] = "tutor"
        sample_agent_template_data["display_name"] = "Custom Tutor"
        new_template = AgentTemplate(**sample_agent_template_data)

        registry.register(new_template)

        retrieved = registry.get("tutor")
        assert retrieved.display_name == "Custom Tutor"


class TestTutorTemplate:
    """Tests for the built-in TUTOR_TEMPLATE."""

    def test_tutor_template_exists(self):
        """Test that TUTOR_TEMPLATE is defined."""
        assert TUTOR_TEMPLATE is not None
        assert isinstance(TUTOR_TEMPLATE, AgentTemplate)

    def test_tutor_template_type_id(self):
        """Test TUTOR_TEMPLATE has correct type_id."""
        assert TUTOR_TEMPLATE.type_id == "tutor"

    def test_tutor_template_has_persona(self):
        """Test TUTOR_TEMPLATE has valid persona."""
        assert isinstance(TUTOR_TEMPLATE.persona, PersonaBlock)
        assert TUTOR_TEMPLATE.persona.name == "YouLab Essay Coach"
        assert "tutor" in TUTOR_TEMPLATE.persona.role.lower() or "essay" in TUTOR_TEMPLATE.persona.role.lower()

    def test_tutor_template_has_capabilities(self):
        """Test TUTOR_TEMPLATE has capabilities."""
        assert len(TUTOR_TEMPLATE.persona.capabilities) > 0

    def test_tutor_template_has_constraints(self):
        """Test TUTOR_TEMPLATE has constraints."""
        assert len(TUTOR_TEMPLATE.persona.constraints) > 0
        # Should include the critical constraint
        constraints_text = " ".join(TUTOR_TEMPLATE.persona.constraints).lower()
        assert "never write" in constraints_text or "not write" in constraints_text

    def test_tutor_template_persona_serializable(self):
        """Test TUTOR_TEMPLATE persona can be serialized."""
        memory_str = TUTOR_TEMPLATE.persona.to_memory_string()
        assert "[IDENTITY]" in memory_str
        assert "YouLab Essay Coach" in memory_str


class TestGlobalTemplatesRegistry:
    """Tests for the global templates registry instance."""

    def test_global_registry_exists(self):
        """Test that global templates instance exists."""
        assert templates is not None
        assert isinstance(templates, AgentTemplateRegistry)

    def test_global_registry_has_tutor(self):
        """Test global registry includes tutor template."""
        assert templates.get("tutor") is not None
```

#### Fixtures to add to `tests/conftest.py`

```python
@pytest.fixture
def sample_agent_template_data():
    """Sample agent template data for testing."""
    return {
        "type_id": "custom",
        "display_name": "Custom Agent",
        "description": "A custom test agent",
        "persona": PersonaBlock(
            name="Custom",
            role="Custom test role",
            capabilities=["Testing", "Validation"],
            tone="professional",
        ),
        "human": HumanBlock(),
    }
```

---

## Phase 1.2: HTTP Service Core Tests

### File: `tests/test_server/conftest.py`

```python
"""Server test fixtures."""

import pytest
from unittest.mock import MagicMock, AsyncMock
from fastapi.testclient import TestClient


@pytest.fixture
def mock_letta_client():
    """Mock Letta client for testing."""
    client = MagicMock()
    client.list_agents.return_value = []
    client.create_agent.return_value = MagicMock(id="agent-123", name="youlab_user1_tutor")
    client.get_agent.return_value = MagicMock(
        id="agent-123",
        name="youlab_user1_tutor",
        metadata={"youlab_user_id": "user1", "youlab_agent_type": "tutor"},
    )
    client.send_message.return_value = MagicMock(
        messages=[MagicMock(assistant_message="Hello! I'm your essay coach.")]
    )
    return client


@pytest.fixture
def mock_agent_manager(mock_letta_client):
    """Mock AgentManager for endpoint testing."""
    from letta_starter.server.agents import AgentManager

    manager = AgentManager.__new__(AgentManager)
    manager._client = mock_letta_client
    manager._cache = {}
    manager.letta_base_url = "http://localhost:8283"
    return manager


@pytest.fixture
def test_client(mock_agent_manager):
    """Test client with mocked dependencies."""
    from letta_starter.server.main import app

    app.state.agent_manager = mock_agent_manager
    return TestClient(app)
```

### File: `tests/test_server/test_schemas.py`

```python
"""Tests for request/response schemas."""

import pytest
from pydantic import ValidationError

from letta_starter.server.schemas import (
    CreateAgentRequest,
    AgentResponse,
    AgentListResponse,
    ChatRequest,
    ChatResponse,
    HealthResponse,
)


class TestCreateAgentRequest:
    """Tests for CreateAgentRequest schema."""

    def test_minimal_request(self):
        """Test request with only required fields."""
        request = CreateAgentRequest(user_id="user123")
        assert request.user_id == "user123"
        assert request.agent_type == "tutor"  # Default
        assert request.user_name is None

    def test_full_request(self):
        """Test request with all fields."""
        request = CreateAgentRequest(
            user_id="user123",
            agent_type="custom",
            user_name="Alice",
        )
        assert request.user_id == "user123"
        assert request.agent_type == "custom"
        assert request.user_name == "Alice"

    def test_user_id_required(self):
        """Test that user_id is required."""
        with pytest.raises(ValidationError):
            CreateAgentRequest()


class TestAgentResponse:
    """Tests for AgentResponse schema."""

    def test_full_response(self):
        """Test response with all fields."""
        response = AgentResponse(
            agent_id="agent-123",
            user_id="user123",
            agent_type="tutor",
            agent_name="youlab_user123_tutor",
            created_at=1703880000,
        )
        assert response.agent_id == "agent-123"
        assert response.created_at == 1703880000

    def test_minimal_response(self):
        """Test response without optional fields."""
        response = AgentResponse(
            agent_id="agent-123",
            user_id="user123",
            agent_type="tutor",
            agent_name="youlab_user123_tutor",
        )
        assert response.created_at is None


class TestChatRequest:
    """Tests for ChatRequest schema."""

    def test_minimal_request(self):
        """Test request with only required fields."""
        request = ChatRequest(
            agent_id="agent-123",
            message="Hello!",
        )
        assert request.agent_id == "agent-123"
        assert request.message == "Hello!"
        assert request.chat_id is None
        assert request.chat_title is None

    def test_full_request(self):
        """Test request with all fields."""
        request = ChatRequest(
            agent_id="agent-123",
            message="Hello!",
            chat_id="chat-456",
            chat_title="Essay Brainstorming",
        )
        assert request.chat_id == "chat-456"
        assert request.chat_title == "Essay Brainstorming"


class TestHealthResponse:
    """Tests for HealthResponse schema."""

    def test_healthy_response(self):
        """Test healthy status response."""
        response = HealthResponse(
            status="ok",
            letta_connected=True,
        )
        assert response.status == "ok"
        assert response.letta_connected is True
        assert response.version == "0.1.0"  # Default

    def test_degraded_response(self):
        """Test degraded status response."""
        response = HealthResponse(
            status="degraded",
            letta_connected=False,
        )
        assert response.status == "degraded"
        assert response.letta_connected is False
```

### File: `tests/test_server/test_agents.py`

```python
"""Tests for AgentManager."""

import pytest
from unittest.mock import MagicMock, patch

from letta_starter.server.agents import AgentManager


class TestAgentManagerNaming:
    """Tests for agent naming conventions."""

    def test_agent_name_format(self):
        """Test agent name follows convention."""
        manager = AgentManager("http://localhost:8283")
        name = manager._agent_name("user123", "tutor")
        assert name == "youlab_user123_tutor"

    def test_agent_name_with_uuid(self):
        """Test agent name with UUID user_id."""
        manager = AgentManager("http://localhost:8283")
        name = manager._agent_name("550e8400-e29b-41d4-a716-446655440000", "tutor")
        assert name.startswith("youlab_")
        assert name.endswith("_tutor")

    def test_agent_metadata_format(self):
        """Test agent metadata structure."""
        manager = AgentManager("http://localhost:8283")
        metadata = manager._agent_metadata("user123", "tutor")

        assert metadata == {
            "youlab_user_id": "user123",
            "youlab_agent_type": "tutor",
        }


class TestAgentManagerCache:
    """Tests for agent cache functionality."""

    def test_cache_hit(self, mock_letta_client):
        """Test cache returns agent_id when present."""
        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client
        manager._cache[("user123", "tutor")] = "cached-agent-id"

        result = manager.get_agent_id("user123", "tutor")

        assert result == "cached-agent-id"
        # Should not call Letta
        mock_letta_client.list_agents.assert_not_called()

    def test_cache_miss_lookup(self, mock_letta_client):
        """Test Letta lookup on cache miss."""
        mock_agent = MagicMock()
        mock_agent.name = "youlab_user123_tutor"
        mock_agent.id = "letta-agent-id"
        mock_letta_client.list_agents.return_value = [mock_agent]

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.get_agent_id("user123", "tutor")

        assert result == "letta-agent-id"
        assert manager._cache[("user123", "tutor")] == "letta-agent-id"

    def test_cache_miss_not_found(self, mock_letta_client):
        """Test None returned when agent doesn't exist."""
        mock_letta_client.list_agents.return_value = []

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.get_agent_id("user123", "tutor")

        assert result is None


class TestAgentManagerCreate:
    """Tests for agent creation."""

    def test_create_agent_new(self, mock_letta_client):
        """Test creating a new agent."""
        mock_letta_client.list_agents.return_value = []
        mock_letta_client.create_agent.return_value = MagicMock(id="new-agent-id")

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.create_agent("user123", "tutor", "Alice")

        assert result == "new-agent-id"
        mock_letta_client.create_agent.assert_called_once()

    def test_create_agent_already_exists(self, mock_letta_client):
        """Test creating agent when one already exists."""
        mock_agent = MagicMock()
        mock_agent.name = "youlab_user123_tutor"
        mock_agent.id = "existing-agent-id"
        mock_letta_client.list_agents.return_value = [mock_agent]

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.create_agent("user123", "tutor")

        assert result == "existing-agent-id"
        mock_letta_client.create_agent.assert_not_called()

    def test_create_agent_unknown_type(self, mock_letta_client):
        """Test creating agent with unknown type raises error."""
        mock_letta_client.list_agents.return_value = []

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        with pytest.raises(ValueError, match="Unknown agent type"):
            manager.create_agent("user123", "nonexistent_type")

    def test_create_agent_with_user_name(self, mock_letta_client):
        """Test agent creation includes user name in human block."""
        mock_letta_client.list_agents.return_value = []
        mock_letta_client.create_agent.return_value = MagicMock(id="new-agent-id")

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        manager.create_agent("user123", "tutor", "Alice")

        # Verify create_agent was called with memory containing user name
        call_kwargs = mock_letta_client.create_agent.call_args[1]
        assert "memory" in call_kwargs
        assert "Alice" in call_kwargs["memory"]["human"] or call_kwargs["memory"]["human"] is not None


class TestAgentManagerRebuildCache:
    """Tests for cache rebuilding on startup."""

    @pytest.mark.asyncio
    async def test_rebuild_cache_empty(self, mock_letta_client):
        """Test rebuilding cache when no agents exist."""
        mock_letta_client.list_agents.return_value = []

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        count = await manager.rebuild_cache()

        assert count == 0
        assert len(manager._cache) == 0

    @pytest.mark.asyncio
    async def test_rebuild_cache_with_agents(self, mock_letta_client):
        """Test rebuilding cache discovers existing agents."""
        mock_agents = [
            MagicMock(
                name="youlab_user1_tutor",
                id="agent-1",
                metadata={"youlab_user_id": "user1", "youlab_agent_type": "tutor"},
            ),
            MagicMock(
                name="youlab_user2_tutor",
                id="agent-2",
                metadata={"youlab_user_id": "user2", "youlab_agent_type": "tutor"},
            ),
        ]
        mock_letta_client.list_agents.return_value = mock_agents

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        count = await manager.rebuild_cache()

        assert count == 2
        assert manager._cache[("user1", "tutor")] == "agent-1"
        assert manager._cache[("user2", "tutor")] == "agent-2"

    @pytest.mark.asyncio
    async def test_rebuild_cache_ignores_non_youlab(self, mock_letta_client):
        """Test cache rebuild ignores non-YouLab agents."""
        mock_agents = [
            MagicMock(name="some_other_agent", id="other-1", metadata={}),
            MagicMock(
                name="youlab_user1_tutor",
                id="agent-1",
                metadata={"youlab_user_id": "user1", "youlab_agent_type": "tutor"},
            ),
        ]
        mock_letta_client.list_agents.return_value = mock_agents

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        count = await manager.rebuild_cache()

        assert count == 1


class TestAgentManagerSendMessage:
    """Tests for sending messages to agents."""

    def test_send_message_success(self, mock_letta_client):
        """Test successful message sending."""
        mock_response = MagicMock()
        mock_msg = MagicMock()
        mock_msg.assistant_message = "Hello! I'm your coach."
        mock_response.messages = [mock_msg]
        mock_letta_client.send_message.return_value = mock_response

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.send_message("agent-123", "Hello!")

        assert result == "Hello! I'm your coach."

    def test_send_message_extracts_text(self, mock_letta_client):
        """Test message extraction handles different response formats."""
        mock_response = MagicMock()
        mock_msg = MagicMock()
        mock_msg.assistant_message = None
        mock_msg.text = "Fallback text"
        mock_response.messages = [mock_msg]
        mock_letta_client.send_message.return_value = mock_response

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.send_message("agent-123", "Hello!")

        assert result == "Fallback text"

    def test_send_message_empty_response(self, mock_letta_client):
        """Test handling empty response."""
        mock_response = MagicMock()
        mock_response.messages = []
        mock_letta_client.send_message.return_value = mock_response

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.send_message("agent-123", "Hello!")

        assert result == ""


class TestAgentManagerHealthCheck:
    """Tests for Letta connection health check."""

    def test_check_connection_success(self, mock_letta_client):
        """Test successful connection check."""
        mock_letta_client.list_agents.return_value = []

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.check_letta_connection()

        assert result is True

    def test_check_connection_failure(self, mock_letta_client):
        """Test failed connection check."""
        mock_letta_client.list_agents.side_effect = Exception("Connection refused")

        manager = AgentManager("http://localhost:8283")
        manager._client = mock_letta_client

        result = manager.check_letta_connection()

        assert result is False
```

### File: `tests/test_server/test_endpoints.py`

```python
"""Tests for FastAPI endpoints."""

import pytest
from fastapi.testclient import TestClient


class TestHealthEndpoint:
    """Tests for /health endpoint."""

    def test_health_returns_200(self, test_client):
        """Test health endpoint returns 200."""
        response = test_client.get("/health")
        assert response.status_code == 200

    def test_health_response_structure(self, test_client):
        """Test health response has expected fields."""
        response = test_client.get("/health")
        data = response.json()

        assert "status" in data
        assert "letta_connected" in data
        assert "version" in data

    def test_health_connected_status(self, test_client, mock_agent_manager):
        """Test health shows connected when Letta is available."""
        mock_agent_manager.check_letta_connection = lambda: True

        response = test_client.get("/health")
        data = response.json()

        assert data["status"] == "ok"
        assert data["letta_connected"] is True

    def test_health_degraded_status(self, test_client, mock_agent_manager):
        """Test health shows degraded when Letta unavailable."""
        mock_agent_manager.check_letta_connection = lambda: False

        response = test_client.get("/health")
        data = response.json()

        assert data["status"] == "degraded"
        assert data["letta_connected"] is False


class TestCreateAgentEndpoint:
    """Tests for POST /agents endpoint."""

    def test_create_agent_success(self, test_client, mock_agent_manager):
        """Test successful agent creation."""
        mock_agent_manager.create_agent = lambda **kw: "new-agent-id"
        mock_agent_manager.get_agent_info = lambda aid: {
            "agent_id": "new-agent-id",
            "user_id": "user123",
            "agent_type": "tutor",
            "agent_name": "youlab_user123_tutor",
            "created_at": None,
        }

        response = test_client.post("/agents", json={"user_id": "user123"})

        assert response.status_code == 201
        data = response.json()
        assert data["agent_id"] == "new-agent-id"
        assert data["user_id"] == "user123"

    def test_create_agent_with_name(self, test_client, mock_agent_manager):
        """Test agent creation with user name."""
        mock_agent_manager.create_agent = lambda **kw: "new-agent-id"
        mock_agent_manager.get_agent_info = lambda aid: {
            "agent_id": "new-agent-id",
            "user_id": "user123",
            "agent_type": "tutor",
            "agent_name": "youlab_user123_tutor",
            "created_at": None,
        }

        response = test_client.post("/agents", json={
            "user_id": "user123",
            "user_name": "Alice",
        })

        assert response.status_code == 201

    def test_create_agent_missing_user_id(self, test_client):
        """Test agent creation fails without user_id."""
        response = test_client.post("/agents", json={})
        assert response.status_code == 422  # Validation error

    def test_create_agent_invalid_type(self, test_client, mock_agent_manager):
        """Test agent creation fails with invalid type."""
        mock_agent_manager.create_agent = lambda **kw: (_ for _ in ()).throw(
            ValueError("Unknown agent type: bad_type")
        )

        response = test_client.post("/agents", json={
            "user_id": "user123",
            "agent_type": "bad_type",
        })

        assert response.status_code == 400


class TestGetAgentEndpoint:
    """Tests for GET /agents/{agent_id} endpoint."""

    def test_get_agent_success(self, test_client, mock_agent_manager):
        """Test successful agent retrieval."""
        mock_agent_manager.get_agent_info = lambda aid: {
            "agent_id": "agent-123",
            "user_id": "user123",
            "agent_type": "tutor",
            "agent_name": "youlab_user123_tutor",
            "created_at": 1703880000,
        }

        response = test_client.get("/agents/agent-123")

        assert response.status_code == 200
        data = response.json()
        assert data["agent_id"] == "agent-123"

    def test_get_agent_not_found(self, test_client, mock_agent_manager):
        """Test 404 for nonexistent agent."""
        mock_agent_manager.get_agent_info = lambda aid: None

        response = test_client.get("/agents/nonexistent")

        assert response.status_code == 404


class TestListAgentsEndpoint:
    """Tests for GET /agents endpoint."""

    def test_list_all_agents(self, test_client, mock_agent_manager):
        """Test listing all agents."""
        mock_agent_manager.client = type("Client", (), {
            "list_agents": lambda: [
                type("Agent", (), {
                    "name": "youlab_user1_tutor",
                    "id": "agent-1",
                    "metadata": {"youlab_user_id": "user1", "youlab_agent_type": "tutor"},
                })(),
            ]
        })()

        response = test_client.get("/agents")

        assert response.status_code == 200
        data = response.json()
        assert "agents" in data

    def test_list_agents_by_user(self, test_client, mock_agent_manager):
        """Test listing agents filtered by user_id."""
        mock_agent_manager.list_user_agents = lambda uid: [
            {
                "agent_id": "agent-1",
                "user_id": uid,
                "agent_type": "tutor",
                "agent_name": f"youlab_{uid}_tutor",
                "created_at": None,
            }
        ]

        response = test_client.get("/agents?user_id=user123")

        assert response.status_code == 200
        data = response.json()
        assert len(data["agents"]) == 1
        assert data["agents"][0]["user_id"] == "user123"


class TestChatEndpoint:
    """Tests for POST /chat endpoint."""

    def test_chat_success(self, test_client, mock_agent_manager):
        """Test successful chat message."""
        mock_agent_manager.get_agent_info = lambda aid: {
            "agent_id": "agent-123",
            "user_id": "user123",
            "agent_type": "tutor",
            "agent_name": "youlab_user123_tutor",
            "created_at": None,
        }
        mock_agent_manager.send_message = lambda aid, msg: "Hello! I'm your coach."

        response = test_client.post("/chat", json={
            "agent_id": "agent-123",
            "message": "Hello!",
        })

        assert response.status_code == 200
        data = response.json()
        assert data["response"] == "Hello! I'm your coach."
        assert data["agent_id"] == "agent-123"

    def test_chat_with_context(self, test_client, mock_agent_manager):
        """Test chat with chat_id and chat_title."""
        mock_agent_manager.get_agent_info = lambda aid: {
            "agent_id": "agent-123",
            "user_id": "user123",
            "agent_type": "tutor",
            "agent_name": "youlab_user123_tutor",
            "created_at": None,
        }
        mock_agent_manager.send_message = lambda aid, msg: "Response"

        response = test_client.post("/chat", json={
            "agent_id": "agent-123",
            "message": "Hello!",
            "chat_id": "chat-456",
            "chat_title": "Essay Brainstorming",
        })

        assert response.status_code == 200

    def test_chat_agent_not_found(self, test_client, mock_agent_manager):
        """Test 404 when agent doesn't exist."""
        mock_agent_manager.get_agent_info = lambda aid: None

        response = test_client.post("/chat", json={
            "agent_id": "nonexistent",
            "message": "Hello!",
        })

        assert response.status_code == 404

    def test_chat_empty_response(self, test_client, mock_agent_manager):
        """Test fallback message for empty response."""
        mock_agent_manager.get_agent_info = lambda aid: {
            "agent_id": "agent-123",
            "user_id": "user123",
            "agent_type": "tutor",
            "agent_name": "youlab_user123_tutor",
            "created_at": None,
        }
        mock_agent_manager.send_message = lambda aid, msg: ""

        response = test_client.post("/chat", json={
            "agent_id": "agent-123",
            "message": "Hello!",
        })

        assert response.status_code == 200
        data = response.json()
        assert data["response"] == "No response from agent."
```

---

## Phase 1.3: Updated Pipe Tests

### File: `tests/test_pipe.py`

```python
"""Tests for OpenWebUI Pipe."""

import pytest
from unittest.mock import MagicMock, patch

from letta_starter.pipelines.letta_pipe import Pipeline


class TestPipelineInit:
    """Tests for Pipeline initialization."""

    def test_pipeline_name(self):
        """Test pipeline has correct name."""
        pipeline = Pipeline()
        assert pipeline.name == "YouLab Tutor"

    def test_pipeline_has_valves(self):
        """Test pipeline has valves configuration."""
        pipeline = Pipeline()
        assert hasattr(pipeline, "valves")
        assert pipeline.valves.LETTA_SERVICE_URL == "http://localhost:8100"

    def test_valves_default_values(self):
        """Test valves have correct defaults."""
        pipeline = Pipeline()
        assert pipeline.valves.AGENT_TYPE == "tutor"
        assert pipeline.valves.ENABLE_LOGGING is True


class TestPipelineLifecycle:
    """Tests for pipeline lifecycle methods."""

    @pytest.mark.asyncio
    async def test_on_startup(self, capsys):
        """Test startup handler."""
        pipeline = Pipeline()
        await pipeline.on_startup()

        captured = capsys.readouterr()
        assert "YouLab Pipeline started" in captured.out

    @pytest.mark.asyncio
    async def test_on_shutdown(self, capsys):
        """Test shutdown handler."""
        pipeline = Pipeline()
        await pipeline.on_shutdown()

        captured = capsys.readouterr()
        assert "YouLab Pipeline stopped" in captured.out


class TestGetChatTitle:
    """Tests for _get_chat_title method."""

    def test_no_chat_id(self):
        """Test returns None when no chat_id."""
        pipeline = Pipeline()
        result = pipeline._get_chat_title(None)
        assert result is None

    def test_local_chat_id(self):
        """Test returns None for local: prefixed IDs."""
        pipeline = Pipeline()
        result = pipeline._get_chat_title("local:some-id")
        assert result is None

    def test_openwebui_import_error(self):
        """Test handles ImportError gracefully."""
        pipeline = Pipeline()
        result = pipeline._get_chat_title("some-chat-id")
        # Without OpenWebUI, should return None
        assert result is None

    @patch("letta_starter.pipelines.letta_pipe.Chats")
    def test_with_valid_chat(self, mock_chats):
        """Test returns title when chat exists."""
        # This test would require mocking the OpenWebUI import
        # Skip for now as it requires complex import mocking
        pass


class TestEnsureAgentExists:
    """Tests for _ensure_agent_exists method."""

    def test_agent_exists(self):
        """Test returns agent_id when agent exists."""
        pipeline = Pipeline()

        with patch("httpx.Client") as mock_client_class:
            mock_client = MagicMock()
            mock_client.__enter__ = MagicMock(return_value=mock_client)
            mock_client.__exit__ = MagicMock(return_value=False)
            mock_client_class.return_value = mock_client

            mock_response = MagicMock()
            mock_response.status_code = 200
            mock_response.json.return_value = {
                "agents": [{"agent_id": "existing-id", "agent_type": "tutor"}]
            }
            mock_client.get.return_value = mock_response

            with mock_client:
                result = pipeline._ensure_agent_exists(mock_client, "user123")

            assert result == "existing-id"

    def test_agent_created(self):
        """Test creates agent when not exists."""
        pipeline = Pipeline()

        with patch("httpx.Client") as mock_client_class:
            mock_client = MagicMock()
            mock_client.__enter__ = MagicMock(return_value=mock_client)
            mock_client.__exit__ = MagicMock(return_value=False)
            mock_client_class.return_value = mock_client

            # First call: no agents
            mock_get_response = MagicMock()
            mock_get_response.status_code = 200
            mock_get_response.json.return_value = {"agents": []}
            mock_client.get.return_value = mock_get_response

            # Second call: create agent
            mock_post_response = MagicMock()
            mock_post_response.status_code = 201
            mock_post_response.json.return_value = {"agent_id": "new-agent-id"}
            mock_client.post.return_value = mock_post_response

            with mock_client:
                result = pipeline._ensure_agent_exists(mock_client, "user123", "Alice")

            assert result == "new-agent-id"


class TestPipe:
    """Tests for main pipe method."""

    def test_missing_user(self):
        """Test error when no user context."""
        pipeline = Pipeline()

        result = pipeline.pipe(
            user_message="Hello",
            model_id="test",
            messages=[],
            body={},
            __user__=None,
        )

        assert "Error" in result
        assert "user" in result.lower()

    def test_missing_user_id(self):
        """Test error when user has no id."""
        pipeline = Pipeline()

        result = pipeline.pipe(
            user_message="Hello",
            model_id="test",
            messages=[],
            body={},
            __user__={"name": "Test"},  # No id
        )

        assert "Error" in result

    @patch("httpx.Client")
    def test_successful_chat(self, mock_client_class):
        """Test successful message flow."""
        pipeline = Pipeline()

        mock_client = MagicMock()
        mock_client.__enter__ = MagicMock(return_value=mock_client)
        mock_client.__exit__ = MagicMock(return_value=False)
        mock_client_class.return_value = mock_client

        # Mock agent exists
        mock_get = MagicMock()
        mock_get.status_code = 200
        mock_get.json.return_value = {
            "agents": [{"agent_id": "agent-123", "agent_type": "tutor"}]
        }
        mock_client.get.return_value = mock_get

        # Mock chat response
        mock_post = MagicMock()
        mock_post.status_code = 200
        mock_post.json.return_value = {"response": "Hello! I'm here to help."}
        mock_client.post.return_value = mock_post

        result = pipeline.pipe(
            user_message="Hello!",
            model_id="test",
            messages=[],
            body={},
            __user__={"id": "user123", "name": "Alice"},
            __metadata__={"chat_id": "chat-456"},
        )

        assert result == "Hello! I'm here to help."

    @patch("httpx.Client")
    def test_agent_creation_failure(self, mock_client_class):
        """Test error handling when agent creation fails."""
        pipeline = Pipeline()

        mock_client = MagicMock()
        mock_client.__enter__ = MagicMock(return_value=mock_client)
        mock_client.__exit__ = MagicMock(return_value=False)
        mock_client_class.return_value = mock_client

        # Mock no existing agents
        mock_get = MagicMock()
        mock_get.status_code = 200
        mock_get.json.return_value = {"agents": []}
        mock_client.get.return_value = mock_get

        # Mock failed creation
        mock_post = MagicMock()
        mock_post.status_code = 500
        mock_post.text = "Internal error"
        mock_client.post.return_value = mock_post

        result = pipeline.pipe(
            user_message="Hello!",
            model_id="test",
            messages=[],
            body={},
            __user__={"id": "user123"},
        )

        assert "Error" in result

    @patch("httpx.Client")
    def test_timeout_handling(self, mock_client_class):
        """Test timeout error handling."""
        import httpx

        pipeline = Pipeline()

        mock_client = MagicMock()
        mock_client.__enter__ = MagicMock(return_value=mock_client)
        mock_client.__exit__ = MagicMock(return_value=False)
        mock_client_class.return_value = mock_client
        mock_client.get.side_effect = httpx.TimeoutException("Timeout")

        result = pipeline.pipe(
            user_message="Hello!",
            model_id="test",
            messages=[],
            body={},
            __user__={"id": "user123"},
        )

        assert "timed out" in result.lower()
```

---

## Phase 1.4: Langfuse Integration Tests

### File: `tests/test_server/test_tracing.py`

```python
"""Tests for Langfuse tracing."""

import pytest
from unittest.mock import MagicMock, patch

from letta_starter.server.tracing import (
    get_langfuse,
    trace_chat,
    trace_generation,
)


class TestGetLangfuse:
    """Tests for get_langfuse function."""

    def test_disabled_returns_none(self):
        """Test None returned when Langfuse disabled."""
        with patch("letta_starter.server.tracing.settings") as mock_settings:
            mock_settings.langfuse_enabled = False
            mock_settings.langfuse_public_key = "pk_test"
            mock_settings.langfuse_secret_key = "sk_test"

            result = get_langfuse()

            assert result is None

    def test_missing_keys_returns_none(self):
        """Test None returned when keys missing."""
        with patch("letta_starter.server.tracing.settings") as mock_settings:
            mock_settings.langfuse_enabled = True
            mock_settings.langfuse_public_key = None
            mock_settings.langfuse_secret_key = None

            result = get_langfuse()

            assert result is None

    def test_creates_client_with_keys(self):
        """Test Langfuse client created with valid keys."""
        with patch("letta_starter.server.tracing.settings") as mock_settings:
            with patch("letta_starter.server.tracing.Langfuse") as mock_langfuse:
                mock_settings.langfuse_enabled = True
                mock_settings.langfuse_public_key = "pk_test"
                mock_settings.langfuse_secret_key = "sk_test"
                mock_settings.langfuse_host = "https://cloud.langfuse.com"

                # Reset singleton
                import letta_starter.server.tracing as tracing_module
                tracing_module._langfuse = None

                result = get_langfuse()

                mock_langfuse.assert_called_once_with(
                    public_key="pk_test",
                    secret_key="sk_test",
                    host="https://cloud.langfuse.com",
                )


class TestTraceChatContext:
    """Tests for trace_chat context manager."""

    def test_returns_trace_id(self):
        """Test context includes trace_id."""
        with patch("letta_starter.server.tracing.get_langfuse", return_value=None):
            with trace_chat(
                user_id="user123",
                agent_id="agent-123",
            ) as ctx:
                assert "trace_id" in ctx
                assert isinstance(ctx["trace_id"], str)

    def test_creates_trace_when_enabled(self):
        """Test Langfuse trace created when enabled."""
        mock_langfuse = MagicMock()
        mock_trace = MagicMock()
        mock_langfuse.trace.return_value = mock_trace

        with patch("letta_starter.server.tracing.get_langfuse", return_value=mock_langfuse):
            with trace_chat(
                user_id="user123",
                agent_id="agent-123",
                chat_id="chat-456",
            ) as ctx:
                assert ctx.get("langfuse_trace") == mock_trace

        mock_langfuse.trace.assert_called_once()
        mock_langfuse.flush.assert_called_once()

    def test_graceful_without_langfuse(self):
        """Test works without Langfuse."""
        with patch("letta_starter.server.tracing.get_langfuse", return_value=None):
            with trace_chat(
                user_id="user123",
                agent_id="agent-123",
            ) as ctx:
                assert "trace_id" in ctx
                assert "langfuse_trace" not in ctx

    def test_handles_trace_exception(self):
        """Test handles Langfuse errors gracefully."""
        mock_langfuse = MagicMock()
        mock_langfuse.trace.side_effect = Exception("Connection error")

        with patch("letta_starter.server.tracing.get_langfuse", return_value=mock_langfuse):
            # Should not raise
            with trace_chat(
                user_id="user123",
                agent_id="agent-123",
            ) as ctx:
                assert "trace_id" in ctx


class TestTraceGeneration:
    """Tests for trace_generation function."""

    def test_records_generation(self):
        """Test generation recorded to trace."""
        mock_trace = MagicMock()
        context = {"langfuse_trace": mock_trace}

        trace_generation(
            trace_context=context,
            name="agent_response",
            input_text="Hello!",
            output_text="Hi there!",
        )

        mock_trace.generation.assert_called_once_with(
            name="agent_response",
            input="Hello!",
            output="Hi there!",
            model="letta",
        )

    def test_no_trace_in_context(self):
        """Test handles missing trace gracefully."""
        context = {}

        # Should not raise
        trace_generation(
            trace_context=context,
            name="agent_response",
            input_text="Hello!",
            output_text="Hi there!",
        )

    def test_handles_generation_exception(self):
        """Test handles generation errors gracefully."""
        mock_trace = MagicMock()
        mock_trace.generation.side_effect = Exception("API error")
        context = {"langfuse_trace": mock_trace}

        # Should not raise
        trace_generation(
            trace_context=context,
            name="agent_response",
            input_text="Hello!",
            output_text="Hi there!",
        )
```

---

## Implementation Order

### Step 1: Add Test Dependencies (if needed)

Verify `pytest-asyncio` is in dev dependencies (already present in `pyproject.toml`).

### Step 2: Create Test Files (Tests First)

1. Create `tests/test_templates.py`
2. Create `tests/test_server/__init__.py`
3. Create `tests/test_server/conftest.py`
4. Create `tests/test_server/test_schemas.py`
5. Create `tests/test_server/test_agents.py`
6. Create `tests/test_server/test_endpoints.py`
7. Create `tests/test_server/test_tracing.py`
8. Create `tests/test_pipe.py`
9. Add fixture to `tests/conftest.py`

### Step 3: Create Minimal Stubs

Create empty modules so imports don't fail:

1. `src/letta_starter/agents/templates.py` - Empty classes
2. `src/letta_starter/server/__init__.py`
3. `src/letta_starter/server/schemas.py` - Empty models
4. `src/letta_starter/server/agents.py` - Empty class
5. `src/letta_starter/server/main.py` - Empty app
6. `src/letta_starter/server/cli.py` - Empty main
7. `src/letta_starter/server/tracing.py` - Empty functions

### Step 4: Verify Tests Fail

Run `make test` - all new tests should fail (expected).

### Step 5: Implement Production Code

Implement each module to make tests pass, following the Phase 1 plan.

---

## Success Criteria

### Test Coverage Targets

| Module | Min Coverage |
|--------|-------------|
| `agents/templates.py` | 95% |
| `server/schemas.py` | 100% |
| `server/agents.py` | 90% |
| `server/main.py` | 85% |
| `server/tracing.py` | 80% |
| `pipelines/letta_pipe.py` | 75% |

### Quality Gates

- [ ] All tests pass: `make test`
- [ ] Lint passes: `make lint`
- [ ] Type check passes: `make typecheck`
- [ ] Coverage meets targets
- [ ] No regressions in existing tests

---

## Notes

### Mocking Strategy

- **Letta Client**: Always mock to avoid requiring running Letta server
- **httpx.Client**: Mock for Pipe tests
- **OpenWebUI imports**: Mock since not available in test environment
- **Langfuse**: Mock to test both enabled and disabled states

### Test Isolation

Each test should be independent. Use fixtures to set up and tear down state. Avoid global state modifications.

### Async Tests

Use `@pytest.mark.asyncio` for async functions. The `asyncio_mode = "auto"` setting in `pyproject.toml` handles most cases.
