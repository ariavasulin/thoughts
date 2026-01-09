# Unify to TOML-Only Agent Creation

## Overview

Simplify the agent creation system by removing the template-based path and unifying on curriculum-based (TOML-driven) configuration. This reduces code complexity, eliminates duplicate definitions, and makes all agent configuration visible in TOML files.

## Current State Analysis

The codebase has two parallel agent creation paths:

```
Template Path (DEPRECATED):
  templates.py → AgentManager.create_agent() → BaseAgent → MemoryManager
  - Hardcoded model: "openai/gpt-4o-mini"
  - Python-defined PersonaBlock/HumanBlock
  - External memory management

Curriculum Path (UNIFIED):
  course.toml → AgentManager.create_agent_from_curriculum() → Letta client
  - Configurable model from TOML
  - Dynamic blocks from TOML schema
  - Agent-driven memory via edit_memory_block tool
```

### Key Discoveries:
- `AgentManager.create_agent()` called from `server/main.py:141` (POST /agents endpoint)
- `MemoryManager` only used by `BaseAgent` - curriculum path doesn't use it
- Two `BackgroundAgentConfig` schemas exist with slightly different structures
- Static blocks use custom labels (`[IDENTITY]`), dynamic use field names (`[NAME]`)

## Desired End State

Single agent creation path where all configuration lives in TOML:

```
config/courses/{course-id}/course.toml → CurriculumLoader → AgentManager → Letta
```

**Verification:**
1. `AgentManager.create_agent("user1", "tutor")` creates agent from `config/courses/default/`
2. All legacy code has deprecation warnings
3. Tests pass using curriculum path
4. `background/schema.py` deleted, imports consolidated

## What We're NOT Doing

- **Not deleting legacy code** - marking as deprecated with warnings
- **Not migrating existing agents** - they can be recreated
- **Not refactoring MemoryManager internals** - just deprecating it
- **Not changing the curriculum path** - it already works correctly
- **Not updating Letta client calls** - API remains the same

## Implementation Approach

1. Create default course config that mirrors TUTOR_TEMPLATE
2. Make `create_agent()` delegate to `create_agent_from_curriculum()`
3. Consolidate schema files
4. Add deprecation warnings to legacy code
5. Update tests
6. Update documentation

---

## Phase 1: Create Default Course Configuration

### Overview
Create `config/courses/default/` with a course.toml that replicates TUTOR_TEMPLATE functionality.

### Changes Required:

#### 1. Create default course directory structure
**Directory**: `config/courses/default/`

```bash
mkdir -p config/courses/default/modules
```

#### 2. Create default course.toml
**File**: `config/courses/default/course.toml`

```toml
# =============================================================================
# DEFAULT COURSE - Base configuration for generic tutor agents
# =============================================================================

[course]
id = "default"
name = "Default Tutor"
version = "1.0.0"
description = "Default AI tutor configuration"
modules = []

# =============================================================================
# AGENT CONFIGURATION
# =============================================================================

[agent]
model = "anthropic/claude-sonnet-4-20250514"
embedding = "openai/text-embedding-3-small"
context_window = 128000
max_response_tokens = 4096
system = """You are a helpful AI assistant.

Your approach:
- Listen carefully to understand what the user needs
- Ask clarifying questions when requirements are unclear
- Provide clear, actionable guidance
- Be warm, encouraging, and patient"""

[[agent.tools]]
id = "send_message"
rules = { type = "exit_loop" }

[[agent.tools]]
id = "query_honcho"
rules = { type = "continue_loop" }

[[agent.tools]]
id = "edit_memory_block"
rules = { type = "continue_loop" }

# =============================================================================
# MEMORY BLOCK SCHEMAS
# =============================================================================

[blocks.persona]
label = "persona"
description = "Agent identity and behavior configuration"

[blocks.persona.fields]
name = { type = "string", default = "Assistant", description = "Agent's display name" }
role = { type = "string", default = "AI assistant", description = "Agent's primary role" }
capabilities = { type = "list", default = ["Answer questions", "Provide guidance", "Help with tasks"], max = 10, description = "What the agent can do" }
expertise = { type = "list", default = [], max = 10, description = "Areas of expertise" }
tone = { type = "string", default = "professional", options = ["warm", "professional", "friendly", "formal"], description = "Communication tone" }
verbosity = { type = "string", default = "adaptive", options = ["concise", "detailed", "adaptive"], description = "Response length style" }
constraints = { type = "list", default = [], max = 10, description = "Behavioral constraints" }

[blocks.human]
label = "human"
description = "User information and session context"

[blocks.human.fields]
name = { type = "string", default = "", description = "User's name" }
role = { type = "string", default = "", description = "User's role or context" }
current_task = { type = "string", default = "", description = "Current activity or request" }
session_state = { type = "string", default = "idle", options = ["idle", "active_task", "waiting_input", "thinking", "error_recovery"], description = "Current session state" }
preferences = { type = "list", default = [], max = 10, description = "Learned preferences" }
context_notes = { type = "list", default = [], max = 10, description = "Recent context notes" }
facts = { type = "list", default = [], max = 20, description = "Important facts about the user" }

# =============================================================================
# UI MESSAGES
# =============================================================================

[messages]
welcome_first = "Hello! I'm here to help. What can I assist you with today?"
welcome_returning = "Welcome back! How can I help you?"
error_unavailable = "I'm having a moment - please try again in a few seconds."
```

### Success Criteria:

#### Automated Verification:
- [ ] File exists: `config/courses/default/course.toml`
- [ ] TOML parses without errors: `python -c "import tomllib; tomllib.load(open('config/courses/default/course.toml', 'rb'))"`
- [ ] Curriculum loader loads it: `python -c "from letta_starter.curriculum import curriculum; curriculum.initialize('config/courses'); print(curriculum.get('default'))"`

#### Manual Verification:
- [ ] Review course.toml content matches intended defaults

---

## Phase 2: Unify Agent Creation Path

### Overview
Make `AgentManager.create_agent()` delegate to `create_agent_from_curriculum()` with "default" course.

### Changes Required:

#### 1. Update AgentManager.create_agent()
**File**: `src/letta_starter/server/agents.py`

Replace the `create_agent` method (lines 81-128) with:

```python
def create_agent(
    self,
    user_id: str,
    agent_type: str = "tutor",
    user_name: str | None = None,
) -> str:
    """
    Create a new agent for a user.

    This method now delegates to create_agent_from_curriculum() using
    the "default" course configuration.

    Args:
        user_id: User identifier
        agent_type: Agent type (used as course_id, defaults to "default" for "tutor")
        user_name: Optional user name to set in human block

    Returns:
        Agent ID

    """
    # Map legacy "tutor" type to "default" course
    course_id = "default" if agent_type == "tutor" else agent_type

    return self.create_agent_from_curriculum(
        user_id=user_id,
        course_id=course_id,
        user_name=user_name,
    )
```

#### 2. Remove templates import
**File**: `src/letta_starter/server/agents.py`

Delete line 10:
```python
from letta_starter.agents.templates import templates
```

#### 3. Update create_agent_from_curriculum cache key
**File**: `src/letta_starter/server/agents.py`

The method already uses `course_id` as `agent_type` for caching (line 158). No change needed.

### Success Criteria:

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Unit tests pass: `make test-agent`

#### Manual Verification:
- [ ] Create agent via API: `curl -X POST localhost:8100/agents -d '{"user_id": "test1"}'`
- [ ] Agent uses default course config (check model, system prompt in Letta)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding.

---

## Phase 3: Consolidate Schema Files

### Overview
Merge `background/schema.py` into `curriculum/schema.py` and delete the duplicate file.

### Changes Required:

#### 1. Add SPECIFIC to SessionScope enum
**File**: `src/letta_starter/curriculum/schema.py`

Update lines 35-40:

```python
class SessionScope(str, Enum):
    """Session scope for dialectic queries."""

    ALL = "all"
    RECENT = "recent"
    CURRENT = "current"
    SPECIFIC = "specific"  # Added from background/schema.py
```

#### 2. Delete background/schema.py
**File**: `src/letta_starter/background/schema.py`

Delete the entire file.

#### 3. Update background/__init__.py
**File**: `src/letta_starter/background/__init__.py`

Replace contents with:

```python
"""Background agent processing."""

from letta_starter.background.runner import BackgroundAgentRunner, RunResult

__all__ = [
    "BackgroundAgentRunner",
    "RunResult",
]
```

#### 4. Update background/runner.py imports
**File**: `src/letta_starter/background/runner.py`

Update line 12 (inside TYPE_CHECKING block):

```python
if TYPE_CHECKING:
    from letta_starter.curriculum.schema import BackgroundAgentConfig, DialecticQuery
```

#### 5. Update server/background.py imports
**File**: `src/letta_starter/server/background.py`

Update line 16:

```python
from letta_starter.curriculum.schema import BackgroundAgentConfig
from letta_starter.curriculum import curriculum
```

And update the `initialize_background` function to use curriculum loader instead of the deleted `load_all_course_configs`.

Replace lines 30-45:

```python
def initialize_background(
    letta_client: Any,
    honcho_client: HonchoClient | None,
    config_dir: Path,
) -> None:
    """
    Initialize background agent system.

    Args:
        letta_client: Letta client instance
        honcho_client: Honcho client (optional)
        config_dir: Directory containing course configs

    """
    global _runner, _initialized

    _runner = BackgroundAgentRunner(letta_client, honcho_client)

    # Curriculum is already initialized by server startup
    # Background agents are accessed via curriculum.get(course_id).background

    _initialized = True
    log.info("background_system_initialized")
```

Update `run_background_agent` endpoint (lines 87-131) to get config from curriculum:

```python
@router.post("/{agent_id}/run", response_model=RunResponse)
async def run_background_agent(
    agent_id: str,
    request: RunRequest | None = None,
) -> RunResponse:
    """Run a background agent manually."""
    if not _initialized or _runner is None:
        raise HTTPException(status_code=503, detail="Background system not initialized")

    # Find the agent config across all courses
    agent_config = None
    for course_id in curriculum.list():
        course = curriculum.get(course_id)
        if course and agent_id in course.background:
            bg_config = course.background[agent_id]
            # Create a config object with id from dict key
            agent_config = bg_config
            break

    if agent_config is None:
        raise HTTPException(status_code=404, detail=f"Background agent not found: {agent_id}")

    user_ids = request.user_ids if request else None
    result = await _runner.run_agent(agent_config, user_ids)

    return RunResponse(
        agent_id=agent_id,
        started_at=result.started_at,
        completed_at=result.completed_at,
        users_processed=result.users_processed,
        queries_executed=result.queries_executed,
        enrichments_applied=result.enrichments_applied,
        errors=result.errors[:MAX_ERRORS_IN_RESPONSE],
    )
```

Update `list_background_agents` endpoint:

```python
@router.get("/agents", response_model=list[AgentInfo])
async def list_background_agents() -> list[AgentInfo]:
    """List all configured background agents."""
    agents = []
    for course_id in curriculum.list():
        course = curriculum.get(course_id)
        if course:
            for agent_id, config in course.background.items():
                agents.append(
                    AgentInfo(
                        id=agent_id,
                        name=agent_id,
                        course_id=course_id,
                        enabled=config.enabled,
                        triggers=f"schedule={config.triggers.schedule}, manual={config.triggers.manual}",
                        query_count=len(config.queries),
                    )
                )
    return agents
```

#### 6. Update BackgroundAgentRunner to work with curriculum schema
**File**: `src/letta_starter/background/runner.py`

The runner expects `config.id` but curriculum's BackgroundAgentConfig doesn't have an id field (it uses dict keys). Update the runner to accept agent_id as a parameter:

Update `run_agent` method signature and usage:

```python
async def run_agent(
    self,
    config: BackgroundAgentConfig,
    user_ids: list[str] | None = None,
    agent_id: str = "unknown",  # Add agent_id parameter
) -> RunResult:
```

And update logging/result to use the passed `agent_id` instead of `config.id`.

### Success Criteria:

#### Automated Verification:
- [ ] File deleted: `! test -f src/letta_starter/background/schema.py`
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] All tests pass: `make test-agent`

#### Manual Verification:
- [ ] Background agent list endpoint works: `curl localhost:8100/background/agents`

---

## Phase 4: Deprecate Legacy Code

### Overview
Add deprecation warnings to all legacy code without deleting it.

### Changes Required:

#### 1. Deprecate templates.py
**File**: `src/letta_starter/agents/templates.py`

Add at the top of the file (after imports):

```python
import warnings

warnings.warn(
    "letta_starter.agents.templates is deprecated. "
    "Use TOML course configuration instead. "
    "See config/courses/default/course.toml for the default template.",
    DeprecationWarning,
    stacklevel=2,
)
```

#### 2. Deprecate static block classes
**File**: `src/letta_starter/memory/blocks.py`

Add deprecation to PersonaBlock and HumanBlock classes:

```python
class PersonaBlock(BaseModel):
    """
    Agent identity and behavior.

    .. deprecated::
        Use dynamic blocks from TOML configuration instead.
        See config/courses/*/course.toml for block schema definitions.
    """

    def __init__(self, **data: Any) -> None:
        warnings.warn(
            "PersonaBlock is deprecated. Use TOML-defined blocks via curriculum loader.",
            DeprecationWarning,
            stacklevel=2,
        )
        super().__init__(**data)
```

Same pattern for `HumanBlock`.

#### 3. Deprecate pre-configured personas
**File**: `src/letta_starter/memory/blocks.py`

Wrap the persona definitions (lines 278-321):

```python
def _get_default_persona() -> PersonaBlock:
    """Get default persona (deprecated)."""
    warnings.warn(
        "DEFAULT_PERSONA is deprecated. Use config/courses/default/course.toml instead.",
        DeprecationWarning,
        stacklevel=2,
    )
    return PersonaBlock(
        name="Assistant",
        role="AI assistant",
        # ... rest of fields
    )

# Keep for backwards compatibility
DEFAULT_PERSONA = _get_default_persona()
```

Or simpler: just add a module-level warning that fires on import:

```python
# At end of file, after persona definitions:
warnings.warn(
    "Pre-configured personas (DEFAULT_PERSONA, CODING_ASSISTANT_PERSONA, "
    "RESEARCH_ASSISTANT_PERSONA) are deprecated. Use TOML configuration instead.",
    DeprecationWarning,
    stacklevel=2,
)
```

#### 4. Deprecate factory functions
**File**: `src/letta_starter/agents/default.py`

Add at the top:

```python
import warnings

warnings.warn(
    "letta_starter.agents.default is deprecated. "
    "Use AgentManager.create_agent_from_curriculum() with TOML configuration instead.",
    DeprecationWarning,
    stacklevel=2,
)
```

#### 5. Deprecate BaseAgent
**File**: `src/letta_starter/agents/base.py`

Add deprecation warning in `__init__`:

```python
def __init__(self, ...):
    warnings.warn(
        "BaseAgent is deprecated. Use AgentManager with TOML course configuration instead.",
        DeprecationWarning,
        stacklevel=2,
    )
    # ... rest of init
```

#### 6. Deprecate MemoryManager
**File**: `src/letta_starter/memory/manager.py`

Add deprecation warning in `__init__`:

```python
def __init__(self, ...):
    warnings.warn(
        "MemoryManager is deprecated. "
        "The curriculum-based agent path uses agent-driven memory via edit_memory_block tool.",
        DeprecationWarning,
        stacklevel=2,
    )
    # ... rest of init
```

### Success Criteria:

#### Automated Verification:
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Tests pass (with deprecation warnings): `make test-agent`

#### Manual Verification:
- [ ] Importing deprecated modules shows warnings: `python -W default -c "from letta_starter.agents.templates import templates"`

---

## Phase 5: Update Tests

### Overview
Update test files to use curriculum path and handle deprecation warnings.

### Changes Required:

#### 1. Update test_server/test_agents.py
**File**: `tests/test_server/test_agents.py`

Update tests for `create_agent` to expect curriculum behavior:

```python
def test_create_agent_new(self, manager, mock_letta_client):
    """Test creating a new agent uses curriculum path."""
    # Mock curriculum to return default course
    with patch("letta_starter.server.agents.curriculum") as mock_curriculum:
        mock_course = MagicMock()
        mock_course.agent.model = "anthropic/claude-sonnet-4-20250514"
        mock_course.agent.embedding = "openai/text-embedding-3-small"
        mock_course.agent.system = "You are helpful."
        mock_course.agent.tools = []
        mock_course.blocks = {}
        mock_course.version = "1.0.0"
        mock_curriculum.get.return_value = mock_course
        mock_curriculum.get_block_registry.return_value = MagicMock(get=lambda x: None)

        result = manager.create_agent("user123", "tutor", "Alice")

        mock_curriculum.get.assert_called_with("default")
```

#### 2. Suppress deprecation warnings in legacy tests
**File**: `tests/test_templates.py`

Add pytest marker to suppress expected deprecation warnings:

```python
import pytest

pytestmark = pytest.mark.filterwarnings("ignore::DeprecationWarning")
```

#### 3. Update test_background_schema.py
**File**: `tests/test_background_schema.py`

Update imports to use curriculum schema:

```python
from letta_starter.curriculum.schema import (
    BackgroundAgentConfig,
    DialecticQuery,
    MergeStrategy,
    SessionScope,
    Triggers,
    IdleTrigger,
)
```

Remove tests for deleted `CourseConfig` from background schema (the curriculum version is different).

#### 4. Update test fixtures
**File**: `tests/conftest.py`

Add deprecation warning filter for fixtures using legacy blocks:

```python
@pytest.fixture
def sample_persona_data():
    """Sample persona data for testing."""
    with warnings.catch_warnings():
        warnings.simplefilter("ignore", DeprecationWarning)
        # ... existing fixture code
```

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `make test-agent`
- [ ] No unexpected deprecation warnings in test output

#### Manual Verification:
- [ ] Review test output for any failures

---

## Phase 6: Update Documentation

### Overview
Update documentation to reflect the unified TOML-only architecture.

### Changes Required:

#### 1. Update CLAUDE.md
**File**: `CLAUDE.md`

Update the Key Files section to remove deprecated files and add default course:

```markdown
## Key Files

- `config/courses/default/course.toml` - Default agent configuration
- `config/courses/college-essay/course.toml` - College essay course configuration
- `src/letta_starter/curriculum/schema.py` - All TOML configuration schemas
- `src/letta_starter/curriculum/loader.py` - TOML loading and caching
- `src/letta_starter/curriculum/blocks.py` - Dynamic block generation from TOML
- `src/letta_starter/server/agents.py` - AgentManager for per-user agents
- `src/letta_starter/server/main.py` - FastAPI HTTP service
...
```

Add deprecation note:

```markdown
## Deprecated Code

The following modules are deprecated and will be removed in a future version:
- `src/letta_starter/agents/templates.py` - Use TOML configuration
- `src/letta_starter/agents/default.py` - Use AgentManager.create_agent()
- `src/letta_starter/agents/base.py` - Use curriculum-based agents
- `src/letta_starter/memory/manager.py` - Use agent-driven memory
- `src/letta_starter/memory/blocks.py` PersonaBlock/HumanBlock - Use TOML blocks
```

#### 2. Update docs/Agent-System.md

Remove references to templates and factory functions. Update to show TOML-based agent creation.

#### 3. Update docs/config-schema.md

Add documentation for default course and note that it's the base configuration.

### Success Criteria:

#### Automated Verification:
- [ ] Documentation files updated
- [ ] No broken links in docs

#### Manual Verification:
- [ ] Review updated documentation for accuracy

---

## Testing Strategy

### Unit Tests:
- `test_server/test_agents.py` - Agent creation via curriculum
- `test_curriculum/` - TOML loading and block generation
- Legacy tests with deprecation filters

### Integration Tests:
- Create agent via POST /agents → verify uses default course
- Create agent via POST /agents/curriculum → verify custom course
- Background agent execution with curriculum config

### Manual Testing Steps:
1. Start server: `uv run letta-server`
2. Create agent: `curl -X POST localhost:8100/agents -H "Content-Type: application/json" -d '{"user_id": "test1"}'`
3. Verify agent in Letta uses correct model and system prompt
4. Send message and verify response

## Migration Notes

- Existing agents created with old template path will continue to work (Letta stores the config)
- New agents will use TOML configuration
- No data migration required - agents can be recreated if needed

## References

- Research document: `thoughts/shared/research/2026-01-09-toml-schema-flexibility-analysis.md`
- Existing college-essay config: `config/courses/college-essay/course.toml`
- Curriculum schema: `src/letta_starter/curriculum/schema.py`
