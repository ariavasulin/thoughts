# TOML Course Configuration Implementation Plan

## Overview

Move all course designer-configurable content from Python code to a single TOML file per course (`config/{course_id}.toml`). This enables non-developers to customize tutor personas, messages, and behavior without touching Python code.

## Current State Analysis

All course configuration is currently embedded in Python:
- `src/letta_starter/agents/templates.py:19-47` - TUTOR_TEMPLATE with hardcoded persona
- `src/letta_starter/pipelines/letta_pipe.py:125-196` - Hardcoded error messages
- `src/letta_starter/memory/blocks.py:277-321` - Pre-configured personas (DEFAULT, CODING, RESEARCH)

### Key Discoveries:
- `AgentTemplateRegistry` (`templates.py:50-76`) already supports dynamic registration
- `PersonaBlock` and `HumanBlock` are Pydantic models with full validation
- Sleep agents TOML schema already exists in `thoughts/shared/plans/` as a reference
- Python 3.11+ includes `tomllib` in stdlib (read-only)

## Desired End State

```
config/
  college-essay.toml    # Complete config for essay coaching course
```

A course designer can edit `config/college-essay.toml` to change:
- Tutor name, role, tone, capabilities, constraints
- Welcome messages, error messages
- Background tasks (scheduled Honcho dialectic queries)

Changes take effect on service restart (no code changes needed).

### Verification:
- `make verify-agent` passes
- Service starts and loads config from TOML
- Agent created via API uses TOML-defined persona
- Error messages come from TOML

## What We're NOT Doing

- Hot-reloading (restart required for config changes)
- Admin UI for editing TOML
- Multiple tutors per course (one tutor per course file)
- Curriculum/lesson content (Phase 5 scope)
- Environment-specific config overrides

## Implementation Approach

Use Pydantic models to define the TOML schema, `tomllib` to parse, and modify `AgentTemplateRegistry` to load from config directory at startup.

---

## Phase 1: Define TOML Schema and Models

### Overview
Create the TOML file format and corresponding Pydantic models for validation.

### Changes Required:

#### 1. Create TOML Schema Models
**File**: `src/letta_starter/config/course_config.py` (new)

```python
"""Course configuration schema for TOML files."""

from pydantic import BaseModel, Field


class TutorConfig(BaseModel):
    """Tutor persona configuration."""

    name: str = Field(..., description="Tutor's display name")
    role: str = Field(..., description="Tutor's role description")
    tone: str = Field(default="warm", description="Communication tone")
    verbosity: str = Field(default="adaptive", description="Response verbosity")
    capabilities: list[str] = Field(default_factory=list)
    constraints: list[str] = Field(default_factory=list)
    expertise: list[str] = Field(default_factory=list)


class MessagesConfig(BaseModel):
    """User-facing messages configuration."""

    welcome_first: str = Field(
        default="Welcome! I'm here to help you with your college essays.",
        description="First-time user welcome message",
    )
    welcome_returning: str = Field(
        default="Welcome back! Ready to continue working on your essay?",
        description="Returning user welcome message",
    )
    error_not_logged_in: str = Field(
        default="Please log in to continue.",
        description="Error when user not identified",
    )
    error_no_message: str = Field(
        default="I didn't receive a message. Could you try again?",
        description="Error when no message provided",
    )
    error_service_unavailable: str = Field(
        default="The tutoring service is temporarily unavailable. Please try again shortly.",
        description="Error when service is down",
    )
    error_timeout: str = Field(
        default="The request timed out. Please try again.",
        description="Error on timeout",
    )


class IdleTrigger(BaseModel):
    """Idle-based trigger configuration."""

    idle_minutes: int = Field(..., description="Minutes of inactivity before trigger")
    cooldown: int = Field(default=60, description="Cooldown minutes between runs")


class TaskQuery(BaseModel):
    """A dialectic query that updates a memory block."""

    query: str = Field(..., description="Question to ask Honcho dialectic")
    field: str = Field(..., description="Memory block field to update")
    transform: str = Field(default="append", description="How to apply: append or replace")


class BackgroundTask(BaseModel):
    """A background task with trigger and memory updates."""

    trigger: str | IdleTrigger = Field(..., description="Cron string or idle trigger")
    human: list[TaskQuery] = Field(default_factory=list, description="Updates to human block")


class BackgroundTasksConfig(BaseModel):
    """Background tasks configuration."""

    scope: list[str] = Field(default_factory=lambda: ["tutor"])
    batch_size: int = Field(default=50)
    tasks: list[BackgroundTask] = Field(default_factory=list)


class CourseConfig(BaseModel):
    """Complete course configuration loaded from TOML."""

    # Course metadata
    id: str = Field(..., description="Course identifier (matches filename)")
    name: str = Field(..., description="Course display name")
    description: str = Field(default="")

    # Tutor configuration
    tutor: TutorConfig

    # Messages
    messages: MessagesConfig = Field(default_factory=MessagesConfig)

    # Background tasks (optional)
    background_tasks: BackgroundTasksConfig = Field(default_factory=BackgroundTasksConfig)
```

#### 2. Create Example TOML File
**File**: `config/college-essay.toml` (new)

```toml
# College Essay Coaching Course Configuration
# ============================================
# This file configures the AI tutor for the college essay course.
# Edit this file to customize tutor behavior without changing code.

[course]
id = "college-essay"
name = "College Essay Mastery"
description = "8-week college essay coaching program"

# =============================================================================
# TUTOR PERSONA
# =============================================================================
# Defines who the tutor is and how they interact with students.

[tutor]
name = "YouLab Essay Coach"
role = "AI tutor specializing in college application essays"
tone = "warm"           # Options: warm, professional, friendly, casual
verbosity = "adaptive"  # Options: concise, detailed, adaptive

capabilities = [
    "Guide students through self-discovery exercises",
    "Help brainstorm and develop essay topics",
    "Provide constructive feedback on drafts",
    "Support emotional journey of college applications",
]

expertise = [
    "College admissions",
    "Personal narrative",
    "Reflective writing",
    "Strengths-based coaching",
]

# IMPORTANT: These constraints define what the tutor should NEVER do
constraints = [
    "Never write essays for students",
    "Always ask clarifying questions before giving advice",
    "Celebrate small wins and progress",
]

# =============================================================================
# USER-FACING MESSAGES
# =============================================================================
# Customize the messages students see in various situations.

[messages]
welcome_first = """
Welcome to YouLab! I'm your personal college essay coach.

I'm here to help you discover your unique story and craft compelling essays.
I won't write your essays for you, but I'll guide you every step of the way.

What would you like to work on today?
"""

welcome_returning = "Welcome back! Ready to continue working on your essay journey?"

error_not_logged_in = "Please log in to access your personal essay coach."
error_no_message = "I didn't catch that. Could you try sending your message again?"
error_service_unavailable = "I'm taking a short break. Please try again in a moment."
error_timeout = "That took longer than expected. Let's try again."

# =============================================================================
# BACKGROUND TASKS (Scheduled Memory Enrichment)
# =============================================================================
# Tasks that run in the background to enrich agent memory via Honcho dialectic.

[background_tasks]
scope = ["tutor"]  # Which agent types to run on
batch_size = 50

# Daily insight gathering (3 AM)
[[background_tasks.task]]
trigger = "0 3 * * *"

[[background_tasks.task.human]]
query = "What learning style works best for this student?"
field = "context_notes"

[[background_tasks.task.human]]
query = "How engaged is this student? What motivates them?"
field = "context_notes"

# Idle-triggered mood check (30 min inactive)
[[background_tasks.task]]
trigger = { idle_minutes = 30, cooldown = 60 }

[[background_tasks.task.human]]
query = "Has this student's mood or engagement changed recently?"
field = "context_notes"

# Weekly progress summary (Monday 9 AM)
[[background_tasks.task]]
trigger = "0 9 * * 1"

[[background_tasks.task.human]]
query = "Summarize this student's progress over the past week."
field = "facts"
transform = "replace"
```

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] New models importable: `python -c "from letta_starter.config.course_config import CourseConfig"`

#### Manual Verification:
- [ ] TOML file is readable by non-developers
- [ ] Comments explain each section clearly

---

## Phase 2: Create Config Loader

### Overview
Add a loader that reads TOML files from `config/` directory and returns validated `CourseConfig` objects.

### Changes Required:

#### 1. Create Config Loader
**File**: `src/letta_starter/config/loader.py` (new)

```python
"""Load course configurations from TOML files."""

import tomllib
from functools import lru_cache
from pathlib import Path

import structlog

from letta_starter.config.course_config import CourseConfig

log = structlog.get_logger()

# Default config directory relative to project root
CONFIG_DIR = Path(__file__).parent.parent.parent.parent / "config"


class ConfigLoader:
    """Loads and caches course configurations from TOML files."""

    def __init__(self, config_dir: Path | None = None) -> None:
        self.config_dir = config_dir or CONFIG_DIR
        self._cache: dict[str, CourseConfig] = {}

    def load(self, course_id: str) -> CourseConfig | None:
        """Load a course configuration by ID."""
        if course_id in self._cache:
            return self._cache[course_id]

        config_path = self.config_dir / f"{course_id}.toml"
        if not config_path.exists():
            log.warning("config_not_found", course_id=course_id, path=str(config_path))
            return None

        try:
            with config_path.open("rb") as f:
                data = tomllib.load(f)

            # Flatten nested structure for Pydantic
            config_data = {
                "id": data.get("course", {}).get("id", course_id),
                "name": data.get("course", {}).get("name", course_id),
                "description": data.get("course", {}).get("description", ""),
                "tutor": data.get("tutor", {}),
                "messages": data.get("messages", {}),
                "background_tasks": data.get("background_tasks", {}),
            }

            config = CourseConfig(**config_data)
            self._cache[course_id] = config
            log.info("config_loaded", course_id=course_id)
            return config

        except Exception as e:
            log.error("config_load_failed", course_id=course_id, error=str(e))
            raise

    def load_all(self) -> dict[str, CourseConfig]:
        """Load all course configurations from the config directory."""
        configs = {}
        if not self.config_dir.exists():
            log.warning("config_dir_not_found", path=str(self.config_dir))
            return configs

        for config_file in self.config_dir.glob("*.toml"):
            course_id = config_file.stem
            config = self.load(course_id)
            if config:
                configs[course_id] = config

        return configs

    def clear_cache(self) -> None:
        """Clear the configuration cache."""
        self._cache.clear()


@lru_cache
def get_config_loader() -> ConfigLoader:
    """Get the global config loader instance."""
    return ConfigLoader()


def load_course_config(course_id: str) -> CourseConfig | None:
    """Convenience function to load a course config."""
    return get_config_loader().load(course_id)
```

#### 2. Update config __init__.py
**File**: `src/letta_starter/config/__init__.py`

```python
"""Configuration management."""

from letta_starter.config.course_config import (
    BackgroundTask,
    BackgroundTasksConfig,
    CourseConfig,
    MessagesConfig,
    TaskQuery,
    TutorConfig,
)
from letta_starter.config.loader import ConfigLoader, get_config_loader, load_course_config
from letta_starter.config.settings import ServiceSettings, Settings, get_settings

__all__ = [
    "BackgroundTask",
    "BackgroundTasksConfig",
    "ConfigLoader",
    "CourseConfig",
    "MessagesConfig",
    "ServiceSettings",
    "Settings",
    "TaskQuery",
    "TutorConfig",
    "get_config_loader",
    "get_settings",
    "load_course_config",
]
```

#### 3. Add Tests
**File**: `tests/test_config_loader.py` (new)

```python
"""Tests for course configuration loader."""

from pathlib import Path
from tempfile import TemporaryDirectory

import pytest

from letta_starter.config.course_config import CourseConfig
from letta_starter.config.loader import ConfigLoader


class TestConfigLoader:
    """Tests for ConfigLoader."""

    def test_load_valid_config(self, tmp_path: Path) -> None:
        """Test loading a valid TOML config."""
        config_content = '''
[course]
id = "test-course"
name = "Test Course"

[tutor]
name = "Test Tutor"
role = "Test role"
'''
        (tmp_path / "test-course.toml").write_text(config_content)

        loader = ConfigLoader(config_dir=tmp_path)
        config = loader.load("test-course")

        assert config is not None
        assert config.id == "test-course"
        assert config.tutor.name == "Test Tutor"

    def test_load_missing_config(self, tmp_path: Path) -> None:
        """Test loading a non-existent config returns None."""
        loader = ConfigLoader(config_dir=tmp_path)
        config = loader.load("nonexistent")
        assert config is None

    def test_load_caches_config(self, tmp_path: Path) -> None:
        """Test that configs are cached."""
        config_content = '''
[course]
id = "cached"
name = "Cached"

[tutor]
name = "Tutor"
role = "Role"
'''
        (tmp_path / "cached.toml").write_text(config_content)

        loader = ConfigLoader(config_dir=tmp_path)
        config1 = loader.load("cached")
        config2 = loader.load("cached")

        assert config1 is config2  # Same object (cached)

    def test_load_all(self, tmp_path: Path) -> None:
        """Test loading all configs from directory."""
        for name in ["course1", "course2"]:
            content = f'''
[course]
id = "{name}"
name = "{name}"

[tutor]
name = "Tutor"
role = "Role"
'''
            (tmp_path / f"{name}.toml").write_text(content)

        loader = ConfigLoader(config_dir=tmp_path)
        configs = loader.load_all()

        assert len(configs) == 2
        assert "course1" in configs
        assert "course2" in configs
```

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Loader successfully reads `config/college-essay.toml`

---

## Phase 3: Integrate with Agent Templates

### Overview
Modify `AgentTemplateRegistry` to load templates from TOML configs instead of hardcoded Python.

### Changes Required:

#### 1. Update AgentTemplateRegistry
**File**: `src/letta_starter/agents/templates.py`

Replace the current implementation:

```python
"""Agent templates for YouLab."""

from pydantic import BaseModel, Field

from letta_starter.config.course_config import CourseConfig
from letta_starter.config.loader import get_config_loader
from letta_starter.memory.blocks import HumanBlock, PersonaBlock


class AgentTemplate(BaseModel):
    """Template for creating agents of a specific type."""

    type_id: str = Field(..., description="Unique identifier for this template")
    display_name: str = Field(..., description="Human-readable name")
    description: str = Field(default="", description="Template description")
    persona: PersonaBlock = Field(..., description="Persona block for the agent")
    human: HumanBlock = Field(default_factory=HumanBlock, description="Initial human block")


def _course_config_to_template(config: CourseConfig) -> AgentTemplate:
    """Convert a CourseConfig to an AgentTemplate."""
    return AgentTemplate(
        type_id=config.id,
        display_name=config.name,
        description=config.description,
        persona=PersonaBlock(
            name=config.tutor.name,
            role=config.tutor.role,
            tone=config.tutor.tone,
            verbosity=config.tutor.verbosity,
            capabilities=config.tutor.capabilities,
            constraints=config.tutor.constraints,
            expertise=config.tutor.expertise,
        ),
        human=HumanBlock(),
    )


class AgentTemplateRegistry:
    """Registry for agent templates."""

    def __init__(self) -> None:
        self._templates: dict[str, AgentTemplate] = {}
        self._load_from_config()

    def _load_from_config(self) -> None:
        """Load templates from TOML config files."""
        loader = get_config_loader()
        configs = loader.load_all()

        for course_id, config in configs.items():
            template = _course_config_to_template(config)
            self._templates[template.type_id] = template

    def register(self, template: AgentTemplate) -> None:
        """Register an agent template (for programmatic additions)."""
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

    def reload(self) -> None:
        """Reload templates from config (clears cache)."""
        get_config_loader().clear_cache()
        self._templates.clear()
        self._load_from_config()


# Global registry instance
templates = AgentTemplateRegistry()
```

#### 2. Update Tests
**File**: `tests/test_templates.py`

Update tests to work with TOML-loaded templates:

```python
"""Tests for agent templates."""

from pathlib import Path

import pytest

from letta_starter.agents.templates import (
    AgentTemplate,
    AgentTemplateRegistry,
    _course_config_to_template,
)
from letta_starter.config.course_config import CourseConfig, TutorConfig
from letta_starter.config.loader import ConfigLoader
from letta_starter.memory.blocks import HumanBlock, PersonaBlock


class TestAgentTemplate:
    """Tests for AgentTemplate model."""

    def test_create_template(self) -> None:
        """Test creating an agent template."""
        template = AgentTemplate(
            type_id="test",
            display_name="Test Agent",
            description="A test agent",
            persona=PersonaBlock(name="Test", role="Tester"),
        )
        assert template.type_id == "test"
        assert template.display_name == "Test Agent"
        assert isinstance(template.human, HumanBlock)


class TestCourseConfigToTemplate:
    """Tests for config to template conversion."""

    def test_converts_tutor_config(self) -> None:
        """Test converting CourseConfig to AgentTemplate."""
        config = CourseConfig(
            id="test-course",
            name="Test Course",
            description="Test description",
            tutor=TutorConfig(
                name="Test Tutor",
                role="Test role",
                tone="warm",
                capabilities=["cap1", "cap2"],
            ),
        )

        template = _course_config_to_template(config)

        assert template.type_id == "test-course"
        assert template.display_name == "Test Course"
        assert template.persona.name == "Test Tutor"
        assert template.persona.tone == "warm"
        assert template.persona.capabilities == ["cap1", "cap2"]


class TestAgentTemplateRegistry:
    """Tests for AgentTemplateRegistry."""

    def test_loads_from_config_dir(self, tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> None:
        """Test registry loads templates from config directory."""
        # Create test config
        config_content = '''
[course]
id = "test"
name = "Test"

[tutor]
name = "Test Tutor"
role = "Test role"
'''
        (tmp_path / "test.toml").write_text(config_content)

        # Patch the config loader
        from letta_starter.config import loader
        monkeypatch.setattr(loader, "CONFIG_DIR", tmp_path)
        loader.get_config_loader.cache_clear()

        registry = AgentTemplateRegistry()
        template = registry.get("test")

        assert template is not None
        assert template.persona.name == "Test Tutor"

    def test_register_programmatic_template(self) -> None:
        """Test registering a template programmatically."""
        registry = AgentTemplateRegistry()
        template = AgentTemplate(
            type_id="custom",
            display_name="Custom",
            persona=PersonaBlock(name="Custom", role="Custom role"),
        )

        registry.register(template)

        assert registry.get("custom") is not None
```

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Lint passes: `make lint-fix`

#### Manual Verification:
- [ ] Start service: `uv run letta-server`
- [ ] Create agent via API: `curl -X POST http://localhost:8100/agents -d '{"user_id": "test", "agent_type": "college-essay"}'`
- [ ] Verify agent uses persona from TOML

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that agents are created with TOML-configured personas before proceeding to the next phase.

---

## Phase 4: Integrate Messages

### Overview
Make the pipeline use messages from TOML config instead of hardcoded strings.

### Changes Required:

#### 1. Add Messages Access to Config
**File**: `src/letta_starter/config/loader.py`

Add convenience function:

```python
def get_messages(course_id: str) -> MessagesConfig | None:
    """Get messages config for a course."""
    config = load_course_config(course_id)
    return config.messages if config else None
```

#### 2. Update Pipeline to Use Config Messages
**File**: `src/letta_starter/pipelines/letta_pipe.py`

Update error handling to use configured messages:

```python
# At top of file, add import
from letta_starter.config.loader import load_course_config

# In the Pipe class, add method to get messages
def _get_message(self, key: str, default: str) -> str:
    """Get a message from config, with fallback to default."""
    config = load_course_config(self.valves.AGENT_TYPE)
    if config and hasattr(config.messages, key):
        return getattr(config.messages, key)
    return default

# Replace hardcoded error messages like:
# "Error: Could not identify user. Please log in."
# With:
# self._get_message("error_not_logged_in", "Error: Could not identify user. Please log in.")
```

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Modify `config/college-essay.toml` messages section
- [ ] Restart service
- [ ] Trigger error condition
- [ ] Verify custom message appears

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that messages from TOML appear correctly before proceeding to the next phase.

---

## Phase 5: Documentation and Cleanup

### Overview
Document the TOML configuration system and remove legacy code.

### Changes Required:

#### 1. Update Documentation
**File**: `docs/Configuration.md` (new)

Create documentation for course designers explaining:
- TOML file structure
- Each configuration section
- How to create a new course
- Common customizations

#### 2. Remove Legacy Hardcoded Template
**File**: `src/letta_starter/agents/templates.py`

Remove `TUTOR_TEMPLATE` constant (now loaded from TOML).

#### 3. Update CLAUDE.md
Add note about `config/` directory and TOML configuration.

### Success Criteria:

#### Automated Verification:
- [ ] `make verify-agent` passes
- [ ] No references to removed TUTOR_TEMPLATE constant

#### Manual Verification:
- [ ] Documentation is clear for non-developers
- [ ] Course designer can create new course by copying and editing TOML

---

## Testing Strategy

### Unit Tests:
- `test_config_loader.py` - TOML parsing, caching, error handling
- `test_templates.py` - Config to template conversion, registry loading
- `test_course_config.py` - Pydantic model validation

### Integration Tests:
- Service startup loads configs
- Agent creation uses TOML personas
- Messages come from TOML

### Manual Testing Steps:
1. Edit `config/college-essay.toml` tutor name
2. Restart service
3. Create new agent
4. Verify agent has new tutor name
5. Edit messages section
6. Trigger error, verify custom message

## Performance Considerations

- Configs are cached at startup (no per-request file reads)
- `@lru_cache` on `get_config_loader()` ensures single instance
- `load_all()` called once during `AgentTemplateRegistry.__init__`

## Migration Notes

- Existing agents keep their current persona (stored in Letta)
- New agents get TOML-configured persona
- No migration needed for existing data

## References

- TOML configuration research: `thoughts/shared/research/2026-01-07-toml-configuration-opportunities.md`
- Background tasks TOML schema: `thoughts/shared/plans/2026-01-07-background-agents-schema.toml`
- Current templates: `src/letta_starter/agents/templates.py`
- Roadmap Phase 5: `docs/Roadmap.md` (curriculum parser)
