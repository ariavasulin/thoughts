# Unified TOML Curriculum Schema Implementation Plan

## Overview

Implement a self-contained, explicit TOML configuration system where all YouLab course settings—agent configuration, memory block schemas, background agents, modules, and lessons—are defined in configuration files. The design prioritizes readability for both humans and AI agents.

**Key Design Principles:**
- **Self-contained**: Each course.toml is complete without external references
- **Explicit over implicit**: No inheritance, no magic—what you see is what you get
- **Two layers only**: course.toml + modules/*.toml
- **AI-friendly**: An agent can read one file and understand everything
- **UI-ready**: Schema is introspectable for future visual editors

## Current State Analysis

### What Exists

| Component | Location | Status |
|-----------|----------|--------|
| Background agent TOML | `src/letta_starter/background/schema.py` | Implemented |
| Background agent config | `config/courses/college-essay.toml` | In use (minimal) |
| Agent templates | `src/letta_starter/agents/templates.py` | Hardcoded Python |
| Memory blocks | `src/letta_starter/memory/blocks.py` | Hardcoded Python |

### Key Files to Modify

- `src/letta_starter/background/schema.py` → Expand to full curriculum schema
- `src/letta_starter/agents/templates.py` → Generate from TOML instead of hardcoded
- `src/letta_starter/memory/blocks.py` → Keep as fallback, add dynamic generation
- `src/letta_starter/server/agents.py` → Create agents from curriculum config
- `config/courses/college-essay.toml` → Expand to full course config

## Desired End State

### File Structure

```
config/
└── courses/
    └── college-essay/
        ├── course.toml              # Complete course configuration
        └── modules/
            ├── 01-self-discovery.toml
            ├── 02-topic-development.toml
            └── 03-drafting.toml

src/letta_starter/
├── curriculum/                      # NEW: Curriculum system
│   ├── __init__.py
│   ├── schema.py                    # Pydantic models for TOML
│   ├── blocks.py                    # Dynamic memory block generator
│   └── loader.py                    # Config loading and registry
└── server/
    ├── curriculum.py                # NEW: HTTP endpoints
    └── agents.py                    # Modified: Use curriculum config
```

### Verification

```bash
# Load course config
curl localhost:8100/curriculum/courses/college-essay
# → Returns complete parsed config

# Create agent from curriculum
curl -X POST localhost:8100/agents \
  -d '{"user_id": "test", "course_id": "college-essay"}'
# → Agent created with TOML-defined memory blocks

# Reload config without restart
curl -X POST localhost:8100/curriculum/reload
# → Reloads all TOML files
```

## What We're NOT Doing

- **No inheritance/extends** - Each config is self-contained
- **No custom DSL** - Completion criteria use simple field references
- **No template variables** - Static strings only (e.g., no `{{progress.lesson}}`)
- **No CLI tooling** - Configuration management is file-based for now
- **No visual editor** - TOML files edited directly (UI-ready schema for future)

---

## Implementation Approach

```
Phase 1: Schema Definitions          → Pydantic models for TOML structure
Phase 2: Dynamic Memory Blocks       → Generate Pydantic models from TOML
Phase 3: Curriculum Loader           → Parse and cache configurations
Phase 4: AgentManager Integration    → Create agents from curriculum
Phase 5: HTTP Endpoints              → Management API
Phase 6: Migration & Documentation   → Migrate existing config, write docs
```

---

## Phase 1: Schema Definitions

### Overview

Define Pydantic models that represent the complete TOML structure. These models serve as both validation and documentation.

### Changes Required

#### 1. Create Curriculum Schema Module

**File**: `src/letta_starter/curriculum/__init__.py` (new)

```python
"""Curriculum configuration system."""

from letta_starter.curriculum.schema import (
    AgentConfig,
    BackgroundAgentConfig,
    BlockSchema,
    CourseConfig,
    FieldSchema,
    LessonConfig,
    ModuleConfig,
)
from letta_starter.curriculum.blocks import DynamicBlockRegistry, generate_block_model
from letta_starter.curriculum.loader import CurriculumLoader, CurriculumRegistry, curriculum

__all__ = [
    "AgentConfig",
    "BackgroundAgentConfig",
    "BlockSchema",
    "CourseConfig",
    "CurriculumLoader",
    "CurriculumRegistry",
    "DynamicBlockRegistry",
    "FieldSchema",
    "LessonConfig",
    "ModuleConfig",
    "curriculum",
    "generate_block_model",
]
```

**File**: `src/letta_starter/curriculum/schema.py` (new)

```python
"""
TOML curriculum schema definitions.

This module defines Pydantic models that validate and parse course.toml
and module.toml files. The schema is designed to be:
- Self-contained: No inheritance or external references
- Explicit: All configuration is visible in the file
- AI-friendly: Easy to read and modify programmatically
- UI-ready: Schema is introspectable for visual editors
"""

from __future__ import annotations

from enum import Enum
from typing import Any

from pydantic import BaseModel, Field


# =============================================================================
# ENUMS
# =============================================================================


class FieldType(str, Enum):
    """Supported field types for memory blocks."""

    STRING = "string"
    INT = "int"
    FLOAT = "float"
    BOOL = "bool"
    LIST = "list"
    DATETIME = "datetime"


class SessionScope(str, Enum):
    """Session scope for dialectic queries."""

    ALL = "all"
    RECENT = "recent"
    CURRENT = "current"


class MergeStrategy(str, Enum):
    """Strategy for merging content into memory blocks."""

    APPEND = "append"
    REPLACE = "replace"
    LLM_DIFF = "llm_diff"


class ToolRuleType(str, Enum):
    """Tool execution rule types."""

    EXIT_LOOP = "exit_loop"
    CONTINUE_LOOP = "continue_loop"
    RUN_FIRST = "run_first"


# =============================================================================
# FIELD & BLOCK SCHEMAS
# =============================================================================


class FieldSchema(BaseModel):
    """
    Schema for a single memory block field.

    Example in TOML:
        name = { type = "string", default = "Assistant", description = "Agent name" }
        tone = { type = "string", default = "warm", options = ["warm", "professional"] }
        facts = { type = "list", default = [], max = 20 }
    """

    type: FieldType
    default: Any = None
    options: list[str] | None = None  # Valid values (renders as dropdown in UI)
    max: int | None = None  # Max items for lists
    description: str | None = None
    required: bool = False


class BlockSchema(BaseModel):
    """
    Schema for a memory block.

    Example in TOML:
        [blocks.persona]
        label = "persona"
        description = "Agent identity"

        [blocks.persona.fields]
        name = { type = "string", default = "Assistant" }
    """

    label: str  # Letta's internal block label (e.g., "persona", "human")
    description: str = ""
    fields: dict[str, FieldSchema] = Field(default_factory=dict)


# =============================================================================
# TOOL CONFIGURATION
# =============================================================================


class ToolRules(BaseModel):
    """Tool execution rules."""

    type: ToolRuleType = ToolRuleType.CONTINUE_LOOP
    max_count: int | None = None


class ToolConfig(BaseModel):
    """
    Configuration for a single tool.

    Example in TOML:
        [[agent.tools]]
        id = "send_message"
        rules = { type = "exit_loop" }
    """

    id: str
    enabled: bool = True
    rules: ToolRules = Field(default_factory=ToolRules)


# =============================================================================
# AGENT CONFIGURATION
# =============================================================================


class AgentConfig(BaseModel):
    """
    Letta agent configuration.

    Example in TOML:
        [agent]
        model = "anthropic/claude-sonnet-4-20250514"
        system = "You are a helpful assistant."

        [[agent.tools]]
        id = "send_message"
        rules = { type = "exit_loop" }
    """

    model: str = "anthropic/claude-sonnet-4-20250514"
    embedding: str = "openai/text-embedding-3-small"
    context_window: int = 128000
    max_response_tokens: int = 4096
    system: str = ""
    tools: list[ToolConfig] = Field(default_factory=list)


# =============================================================================
# BACKGROUND AGENTS
# =============================================================================


class IdleTrigger(BaseModel):
    """Idle-based trigger configuration."""

    enabled: bool = False
    threshold_minutes: int = 30
    cooldown_minutes: int = 60


class Triggers(BaseModel):
    """Trigger configuration for background agent."""

    schedule: str | None = None  # Cron expression
    idle: IdleTrigger = Field(default_factory=IdleTrigger)
    manual: bool = True
    after_messages: int | None = None  # Trigger after N messages


class DialecticQuery(BaseModel):
    """
    Single dialectic query configuration.

    Example in TOML:
        [[background.insight-harvester.queries]]
        id = "learning_style"
        question = "What learning style works best for this student?"
        target_block = "student"
        target_field = "context_notes"
        merge_strategy = "append"
    """

    id: str
    question: str
    session_scope: SessionScope = SessionScope.ALL
    recent_limit: int = 5
    target_block: str
    target_field: str
    merge_strategy: MergeStrategy = MergeStrategy.APPEND


class BackgroundAgentConfig(BaseModel):
    """Configuration for a background agent."""

    enabled: bool = True
    triggers: Triggers = Field(default_factory=Triggers)
    agent_types: list[str] = Field(default_factory=lambda: ["tutor"])
    user_filter: str = "all"
    batch_size: int = 50
    queries: list[DialecticQuery] = Field(default_factory=list)


# =============================================================================
# LESSONS & MODULES
# =============================================================================


class LessonCompletion(BaseModel):
    """
    Completion criteria for a lesson.

    Example in TOML:
        [lessons.completion]
        required_fields = ["student.name", "student.grade_level"]
        min_turns = 3
        min_list_length = { "student.strengths" = 3 }
    """

    required_fields: list[str] = Field(default_factory=list)
    min_turns: int | None = None
    min_list_length: dict[str, int] = Field(default_factory=dict)
    auto_advance: bool = False


class LessonAgentConfig(BaseModel):
    """
    Per-lesson agent customization.

    Example in TOML:
        [lessons.agent]
        opening = "Welcome! Let's get started."
        focus = ["strengths", "goals"]
        guidance = ["Ask follow-up questions"]

        [lessons.agent.persona_overrides]
        tone = "encouraging"
    """

    opening: str | None = None
    focus: list[str] = Field(default_factory=list)
    guidance: list[str] = Field(default_factory=list)
    persona_overrides: dict[str, Any] = Field(default_factory=dict)


class LessonConfig(BaseModel):
    """
    Single lesson configuration.

    Example in TOML:
        [[lessons]]
        id = "welcome"
        name = "Welcome & Onboarding"
        order = 1
        objectives = ["Learn student's name", "Set expectations"]

        [lessons.completion]
        required_fields = ["student.name"]

        [lessons.agent]
        opening = "Hello! What's your name?"
    """

    id: str
    name: str
    order: int = 0
    description: str = ""
    objectives: list[str] = Field(default_factory=list)
    completion: LessonCompletion = Field(default_factory=LessonCompletion)
    agent: LessonAgentConfig = Field(default_factory=LessonAgentConfig)


class ModuleConfig(BaseModel):
    """
    Module configuration (loaded from modules/*.toml).

    Example in TOML:
        [module]
        id = "01-self-discovery"
        name = "Self-Discovery"
        order = 1

        [[lessons]]
        id = "welcome"
        ...
    """

    id: str
    name: str
    order: int = 0
    description: str = ""
    lessons: list[LessonConfig] = Field(default_factory=list)
    # Module-level background agent overrides
    background: dict[str, dict[str, Any]] = Field(default_factory=dict)


# =============================================================================
# UI MESSAGES
# =============================================================================


class MessagesConfig(BaseModel):
    """
    UI message strings.

    Example in TOML:
        [messages]
        welcome_first = "Welcome! I'm your tutor."
        welcome_returning = "Welcome back!"
    """

    welcome_first: str = "Hello! How can I help you today?"
    welcome_returning: str = "Welcome back!"
    error_unavailable: str = "I'm temporarily unavailable. Please try again shortly."


# =============================================================================
# COURSE CONFIG (TOP-LEVEL)
# =============================================================================


class CourseConfig(BaseModel):
    """
    Complete course configuration (loaded from course.toml).

    This is the top-level schema. Each course.toml file should be self-contained
    with all configuration explicit—no inheritance or external references.

    Example structure:
        [course]
        id = "college-essay"
        name = "College Essay Writing"
        modules = ["01-self-discovery", "02-drafting"]

        [agent]
        model = "anthropic/claude-sonnet-4-20250514"
        system = "You are an essay coach."

        [blocks.persona]
        label = "persona"
        [blocks.persona.fields]
        name = { type = "string", default = "Essay Coach" }

        [background.insight-harvester]
        enabled = true
        ...

        [messages]
        welcome_first = "Welcome to essay coaching!"
    """

    # Course metadata
    id: str
    name: str
    version: str = "1.0.0"
    description: str = ""
    modules: list[str] = Field(default_factory=list)  # Module file references

    # Agent configuration
    agent: AgentConfig = Field(default_factory=AgentConfig)

    # Memory block schemas
    blocks: dict[str, BlockSchema] = Field(default_factory=dict)

    # Background agents (keyed by agent ID)
    background: dict[str, BackgroundAgentConfig] = Field(default_factory=dict)

    # UI messages
    messages: MessagesConfig = Field(default_factory=MessagesConfig)

    # Loaded modules (populated after loading module files, excluded from serialization)
    loaded_modules: list[ModuleConfig] = Field(default_factory=list, exclude=True)
```

### Success Criteria

#### Automated Verification:
- [x] Schema models validate correctly: `make check-agent`
- [x] All Pydantic models have proper type hints
- [ ] Unit tests for schema validation pass

#### Manual Verification:
- [ ] Schema docstrings are clear and complete
- [ ] Example TOML in docstrings is valid

**Implementation Note**: After completing this phase, pause for verification before proceeding.

---

## Phase 2: Dynamic Memory Block Generator

### Overview

Generate Pydantic models from TOML block schemas at runtime. This allows courses to define custom memory block fields without modifying Python code.

### Changes Required

#### 1. Create Memory Block Generator

**File**: `src/letta_starter/curriculum/blocks.py` (new)

```python
"""
Dynamic memory block generation from TOML schemas.

This module generates Pydantic models at runtime based on TOML field definitions.
Generated models are compatible with Letta's memory system and include
serialization methods for the compact memory string format.
"""

from __future__ import annotations

from datetime import datetime
from typing import Any

from pydantic import BaseModel, Field, create_model

from letta_starter.curriculum.schema import BlockSchema, FieldSchema, FieldType


# Type mapping from TOML field types to Python types
TYPE_MAP: dict[FieldType, type] = {
    FieldType.STRING: str,
    FieldType.INT: int,
    FieldType.FLOAT: float,
    FieldType.BOOL: bool,
    FieldType.LIST: list,
    FieldType.DATETIME: datetime,
}


def generate_block_model(
    name: str,
    schema: BlockSchema,
) -> type[BaseModel]:
    """
    Generate a Pydantic model from a block schema.

    Args:
        name: Model class name (e.g., "PersonaBlock")
        schema: Block schema from TOML

    Returns:
        Dynamically created Pydantic model class

    Example:
        >>> schema = BlockSchema(
        ...     label="persona",
        ...     fields={"name": FieldSchema(type=FieldType.STRING, default="Assistant")}
        ... )
        >>> PersonaBlock = generate_block_model("PersonaBlock", schema)
        >>> instance = PersonaBlock()
        >>> instance.name
        'Assistant'
    """
    field_definitions: dict[str, tuple[type, Any]] = {}

    for field_name, field_schema in schema.fields.items():
        python_type = _get_python_type(field_schema)
        default = _get_default(field_schema)
        field_definitions[field_name] = (python_type, default)

    # Create the model dynamically
    model = create_model(
        name,
        __doc__=schema.description or f"Dynamic memory block: {schema.label}",
        **field_definitions,
    )

    # Attach metadata and methods
    model._block_schema = schema  # type: ignore[attr-defined]
    model._block_label = schema.label  # type: ignore[attr-defined]

    # Add serialization methods as instance methods
    def to_memory_string(self: BaseModel, max_chars: int = 1500) -> str:
        return _to_memory_string(self, schema, max_chars)

    model.to_memory_string = to_memory_string  # type: ignore[attr-defined]

    # Add deserialization as classmethod
    @classmethod
    def from_memory_string(cls: type[BaseModel], memory_str: str) -> BaseModel:
        return _from_memory_string(cls, schema, memory_str)

    model.from_memory_string = from_memory_string  # type: ignore[attr-defined]

    return model


def _get_python_type(field_schema: FieldSchema) -> type:
    """Convert TOML field type to Python type."""
    base_type = TYPE_MAP.get(field_schema.type, str)

    if field_schema.type == FieldType.LIST:
        return list[str]  # Lists are always list[str] for simplicity

    if field_schema.type == FieldType.DATETIME:
        return datetime | None  # Datetimes can be None

    return base_type


def _get_default(field_schema: FieldSchema) -> Any:
    """Get default value with proper Field wrapper for Pydantic."""
    default = field_schema.default

    if field_schema.type == FieldType.LIST:
        if default is None:
            return Field(default_factory=list)
        # Create a factory that returns a copy of the default list
        default_list = list(default)
        return Field(default_factory=lambda d=default_list: list(d))

    if default is None:
        if field_schema.type == FieldType.STRING:
            return ""
        elif field_schema.type == FieldType.INT:
            return 0
        elif field_schema.type == FieldType.FLOAT:
            return 0.0
        elif field_schema.type == FieldType.BOOL:
            return False
        elif field_schema.type == FieldType.DATETIME:
            return None
        return None

    return default


def _to_memory_string(instance: BaseModel, schema: BlockSchema, max_chars: int) -> str:
    """
    Serialize block instance to compact memory format.

    Format: [LABEL] value
    Lists are joined with semicolons.
    Empty values are omitted.
    """
    lines = []

    for field_name, field_schema in schema.fields.items():
        value = getattr(instance, field_name, None)

        # Skip empty values
        if value is None or value == "" or value == []:
            continue

        label = field_name.upper()

        if isinstance(value, list):
            # Limit list items for token efficiency
            max_items = field_schema.max or 10
            value_str = "; ".join(str(v) for v in value[:max_items])
        elif isinstance(value, datetime):
            value_str = value.isoformat()
        elif isinstance(value, bool):
            value_str = "yes" if value else "no"
        else:
            value_str = str(value)

        lines.append(f"[{label}] {value_str}")

    result = "\n".join(lines)
    return result[:max_chars]


def _from_memory_string(
    cls: type[BaseModel],
    schema: BlockSchema,
    memory_str: str,
) -> BaseModel:
    """
    Parse memory string back into block instance.

    Best-effort parsing - unrecognized fields are ignored.
    """
    data: dict[str, Any] = {}

    for line in memory_str.strip().split("\n"):
        if not line.startswith("["):
            continue

        # Parse [LABEL] value format
        bracket_end = line.find("]")
        if bracket_end == -1:
            continue

        label = line[1:bracket_end].lower()
        value = line[bracket_end + 1 :].strip()

        # Find matching field
        for field_name, field_schema in schema.fields.items():
            if field_name.lower() == label:
                parsed = _parse_value(value, field_schema)
                if parsed is not None:
                    data[field_name] = parsed
                break

    return cls(**data)


def _parse_value(value: str, field_schema: FieldSchema) -> Any:
    """Parse a string value according to field type."""
    if not value:
        return None

    if field_schema.type == FieldType.LIST:
        return [v.strip() for v in value.split(";") if v.strip()]
    elif field_schema.type == FieldType.INT:
        try:
            return int(value)
        except ValueError:
            return None
    elif field_schema.type == FieldType.FLOAT:
        try:
            return float(value)
        except ValueError:
            return None
    elif field_schema.type == FieldType.BOOL:
        return value.lower() in ("true", "1", "yes")
    elif field_schema.type == FieldType.DATETIME:
        try:
            return datetime.fromisoformat(value)
        except ValueError:
            return None
    else:
        return value


class DynamicBlockRegistry:
    """
    Registry for dynamically generated memory block models.

    Each course gets its own registry with models generated from its block schemas.

    Example:
        >>> registry = DynamicBlockRegistry()
        >>> registry.register_from_schema("persona", persona_schema)
        >>> PersonaBlock = registry.get("persona")
        >>> instance = PersonaBlock(name="Coach")
    """

    def __init__(self) -> None:
        self._models: dict[str, type[BaseModel]] = {}
        self._schemas: dict[str, BlockSchema] = {}

    def register_from_schema(
        self,
        block_name: str,
        schema: BlockSchema,
    ) -> type[BaseModel]:
        """Generate and register a block model from schema."""
        model_name = f"{block_name.title().replace('_', '')}Block"
        model = generate_block_model(model_name, schema)
        self._models[block_name] = model
        self._schemas[block_name] = schema
        return model

    def get(self, block_name: str) -> type[BaseModel] | None:
        """Get a registered block model by name."""
        return self._models.get(block_name)

    def get_schema(self, block_name: str) -> BlockSchema | None:
        """Get the schema for a registered block."""
        return self._schemas.get(block_name)

    def get_all(self) -> dict[str, type[BaseModel]]:
        """Get all registered block models."""
        return self._models.copy()

    def list_blocks(self) -> list[str]:
        """List all registered block names."""
        return list(self._models.keys())

    def clear(self) -> None:
        """Clear all registered models."""
        self._models.clear()
        self._schemas.clear()

    def create_instances(
        self,
        overrides: dict[str, dict[str, Any]] | None = None,
    ) -> dict[str, BaseModel]:
        """
        Create instances of all registered blocks with optional overrides.

        Args:
            overrides: Dict mapping block_name -> field overrides

        Returns:
            Dict mapping block_name -> block instance
        """
        overrides = overrides or {}
        instances = {}

        for block_name, model in self._models.items():
            block_overrides = overrides.get(block_name, {})
            instances[block_name] = model(**block_overrides)

        return instances
```

### Success Criteria

#### Automated Verification:
- [ ] Generated models serialize/deserialize correctly
- [x] Type checking passes: `make check-agent`
- [ ] Unit tests for block generation pass
- [ ] Round-trip test: create → serialize → deserialize → compare

#### Manual Verification:
- [ ] Generated models work with Letta's memory system
- [ ] Serialization format is token-efficient

**Implementation Note**: After completing this phase, pause for verification before proceeding.

---

## Phase 3: Curriculum Loader

### Overview

Create the configuration loading system that parses TOML files, generates block models, and caches parsed configurations.

### Changes Required

#### 1. Create Curriculum Loader

**File**: `src/letta_starter/curriculum/loader.py` (new)

```python
"""
Curriculum loading and registry.

This module handles loading course.toml and module.toml files,
validating them against the schema, and caching parsed configurations.
"""

from __future__ import annotations

import sys
from pathlib import Path
from typing import Any

import structlog

if sys.version_info >= (3, 11):
    import tomllib
else:
    import tomli as tomllib

from letta_starter.curriculum.blocks import DynamicBlockRegistry
from letta_starter.curriculum.schema import (
    AgentConfig,
    BackgroundAgentConfig,
    BlockSchema,
    CourseConfig,
    DialecticQuery,
    FieldSchema,
    LessonConfig,
    MessagesConfig,
    ModuleConfig,
    Triggers,
)

log = structlog.get_logger()


class CurriculumLoader:
    """
    Loads and parses curriculum TOML files.

    Each course gets its own DynamicBlockRegistry for memory block models.
    """

    def __init__(self, base_dir: Path) -> None:
        self.base_dir = base_dir
        self._block_registries: dict[str, DynamicBlockRegistry] = {}

    def load_course(self, course_id: str) -> CourseConfig:
        """
        Load a course and its modules.

        Args:
            course_id: Course directory name (e.g., "college-essay")

        Returns:
            Fully loaded CourseConfig with modules

        Raises:
            FileNotFoundError: If course.toml doesn't exist
            tomllib.TOMLDecodeError: If TOML is invalid
            pydantic.ValidationError: If schema validation fails
        """
        course_dir = self.base_dir / course_id
        course_file = course_dir / "course.toml"

        if not course_file.exists():
            raise FileNotFoundError(f"Course config not found: {course_file}")

        # Load course.toml
        with open(course_file, "rb") as f:
            raw_data = tomllib.load(f)

        # Parse into CourseConfig
        course_data = self._flatten_course_data(raw_data)
        course = CourseConfig(**course_data)

        # Create block registry for this course
        registry = DynamicBlockRegistry()
        for block_name, block_schema in course.blocks.items():
            registry.register_from_schema(block_name, block_schema)
        self._block_registries[course.id] = registry

        # Load modules
        modules_dir = course_dir / "modules"
        if modules_dir.exists():
            for module_ref in course.modules:
                module_file = modules_dir / f"{module_ref}.toml"
                if module_file.exists():
                    try:
                        module = self._load_module(module_file)
                        course.loaded_modules.append(module)
                    except Exception as e:
                        log.warning(
                            "module_load_failed",
                            course_id=course.id,
                            module=module_ref,
                            error=str(e),
                        )

        # Sort modules by order
        course.loaded_modules.sort(key=lambda m: m.order)

        log.info(
            "course_loaded",
            course_id=course.id,
            modules=len(course.loaded_modules),
            blocks=list(course.blocks.keys()),
            background_agents=list(course.background.keys()),
        )

        return course

    def get_block_registry(self, course_id: str) -> DynamicBlockRegistry | None:
        """Get the block registry for a loaded course."""
        return self._block_registries.get(course_id)

    def _flatten_course_data(self, raw: dict[str, Any]) -> dict[str, Any]:
        """
        Flatten nested TOML structure for Pydantic parsing.

        TOML uses nested tables like [course], [agent], [blocks.persona].
        This flattens them into the structure CourseConfig expects.
        """
        result: dict[str, Any] = {}

        # Course metadata (from [course] table)
        if "course" in raw:
            result.update(raw["course"])

        # Agent config (from [agent] table)
        if "agent" in raw:
            result["agent"] = self._parse_agent_config(raw["agent"])

        # Blocks (from [blocks.*] tables)
        if "blocks" in raw:
            result["blocks"] = self._parse_blocks(raw["blocks"])

        # Background agents (from [background.*] tables)
        if "background" in raw:
            result["background"] = self._parse_background(raw["background"])

        # Messages (from [messages] table)
        if "messages" in raw:
            result["messages"] = raw["messages"]

        return result

    def _parse_agent_config(self, raw: dict[str, Any]) -> dict[str, Any]:
        """Parse agent configuration."""
        config = dict(raw)

        # Tools might be a list of dicts or inline tables
        if "tools" in config and isinstance(config["tools"], list):
            # Already in list format from [[agent.tools]]
            pass

        return config

    def _parse_blocks(self, raw: dict[str, Any]) -> dict[str, BlockSchema]:
        """Parse memory block definitions."""
        blocks = {}

        for block_name, block_data in raw.items():
            fields = {}

            # Fields can be in a nested "fields" key or inline
            field_source = block_data.get("fields", {})

            for field_name, field_def in field_source.items():
                if isinstance(field_def, dict):
                    fields[field_name] = FieldSchema(**field_def)

            blocks[block_name] = BlockSchema(
                label=block_data.get("label", block_name),
                description=block_data.get("description", ""),
                fields=fields,
            )

        return blocks

    def _parse_background(
        self, raw: dict[str, Any]
    ) -> dict[str, BackgroundAgentConfig]:
        """Parse background agent configurations."""
        agents = {}

        for agent_id, agent_data in raw.items():
            # Parse triggers
            triggers_data = agent_data.get("triggers", {})
            triggers = Triggers(**triggers_data)

            # Parse queries
            queries = []
            for query_data in agent_data.get("queries", []):
                queries.append(DialecticQuery(**query_data))

            agents[agent_id] = BackgroundAgentConfig(
                enabled=agent_data.get("enabled", True),
                triggers=triggers,
                agent_types=agent_data.get("agent_types", ["tutor"]),
                user_filter=agent_data.get("user_filter", "all"),
                batch_size=agent_data.get("batch_size", 50),
                queries=queries,
            )

        return agents

    def _load_module(self, path: Path) -> ModuleConfig:
        """Load a single module file."""
        with open(path, "rb") as f:
            raw = tomllib.load(f)

        # Get module metadata
        module_data = raw.get("module", {})

        # Parse lessons
        lessons = []
        for lesson_data in raw.get("lessons", []):
            lessons.append(LessonConfig(**lesson_data))

        # Sort lessons by order
        lessons.sort(key=lambda l: l.order)

        return ModuleConfig(
            id=module_data.get("id", path.stem),
            name=module_data.get("name", path.stem),
            order=module_data.get("order", 0),
            description=module_data.get("description", ""),
            lessons=lessons,
            background=raw.get("background", {}),
        )


class CurriculumRegistry:
    """
    Registry for loaded curriculum configurations.

    Provides a central access point for course configs and supports hot reload.
    """

    def __init__(self) -> None:
        self._courses: dict[str, CourseConfig] = {}
        self._loader: CurriculumLoader | None = None
        self._base_dir: Path | None = None

    def initialize(self, base_dir: Path) -> None:
        """Initialize the registry with a base directory."""
        self._base_dir = base_dir
        self._loader = CurriculumLoader(base_dir)

    def load_all(self) -> int:
        """
        Load all courses from base directory.

        Returns:
            Number of courses loaded successfully
        """
        if self._loader is None or self._base_dir is None:
            raise RuntimeError("Registry not initialized - call initialize() first")

        count = 0
        for course_dir in self._base_dir.iterdir():
            if course_dir.is_dir() and (course_dir / "course.toml").exists():
                try:
                    course = self._loader.load_course(course_dir.name)
                    self._courses[course.id] = course
                    count += 1
                except Exception as e:
                    log.error(
                        "course_load_failed",
                        course_dir=str(course_dir),
                        error=str(e),
                    )

        log.info("curriculum_loaded", courses=count, course_ids=list(self._courses.keys()))
        return count

    def get(self, course_id: str) -> CourseConfig | None:
        """Get a loaded course by ID."""
        return self._courses.get(course_id)

    def get_block_registry(self, course_id: str) -> DynamicBlockRegistry | None:
        """Get the block registry for a course."""
        if self._loader is None:
            return None
        return self._loader.get_block_registry(course_id)

    def list_courses(self) -> list[str]:
        """List all loaded course IDs."""
        return list(self._courses.keys())

    def reload(self) -> int:
        """
        Reload all courses from disk.

        Returns:
            Number of courses loaded
        """
        self._courses.clear()
        if self._loader:
            self._loader._block_registries.clear()
        return self.load_all()

    def is_initialized(self) -> bool:
        """Check if the registry has been initialized."""
        return self._loader is not None


# Global registry instance
curriculum = CurriculumRegistry()
```

### Success Criteria

#### Automated Verification:
- [ ] Loader parses example course.toml correctly
- [ ] Loader parses module.toml files correctly
- [ ] Registry loads multiple courses
- [ ] Reload clears and reloads correctly
- [x] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Courses load without errors on server startup
- [ ] Module ordering matches TOML order field
- [ ] Block models are generated correctly

**Implementation Note**: After completing this phase, pause for verification before proceeding.

---

## Phase 4: AgentManager Integration

### Overview

Update AgentManager to create Letta agents from curriculum configuration instead of hardcoded templates.

### Changes Required

#### 1. Update AgentManager

**File**: `src/letta_starter/server/agents.py`

**Changes**: Add `create_agent_from_curriculum` method

```python
# Add imports at top
from letta_starter.curriculum import curriculum

# Add new method to AgentManager class
class AgentManager:
    # ... existing code ...

    def create_agent_from_curriculum(
        self,
        user_id: str,
        course_id: str,
        user_name: str | None = None,
        block_overrides: dict[str, dict[str, Any]] | None = None,
    ) -> str:
        """
        Create an agent based on curriculum configuration.

        Args:
            user_id: User identifier
            course_id: Course to use for agent configuration
            user_name: Optional user name to set in student/human block
            block_overrides: Optional field overrides per block

        Returns:
            Agent ID

        Raises:
            ValueError: If course not found
        """
        course = curriculum.get(course_id)
        if course is None:
            raise ValueError(f"Unknown course: {course_id}")

        # Use course_id as agent_type for caching
        agent_type = course_id
        agent_name = self._agent_name(user_id, agent_type)

        # Check for existing agent
        existing = self.get_agent_id(user_id, agent_type)
        if existing:
            return existing

        # Get block registry for this course
        block_registry = curriculum.get_block_registry(course_id)
        if block_registry is None:
            raise ValueError(f"Block registry not found for course: {course_id}")

        # Build memory blocks from course schema
        overrides = block_overrides or {}
        memory_blocks = []

        for block_name, block_schema in course.blocks.items():
            model_class = block_registry.get(block_name)
            if model_class is None:
                continue

            # Get block-specific overrides
            block_override = overrides.get(block_name, {})

            # Inject user name into human/student block
            if block_schema.label == "human" and user_name:
                block_override.setdefault("name", user_name)

            # Create instance with overrides
            instance = model_class(**block_override)

            memory_blocks.append({
                "label": block_schema.label,
                "value": instance.to_memory_string(),
            })

        # Build tool list
        tool_names = [tool.id for tool in course.agent.tools if tool.enabled]

        # Build tool rules
        tool_rules = []
        for tool in course.agent.tools:
            if tool.enabled and tool.rules:
                rule = {"tool_name": tool.id, "type": tool.rules.type.value}
                if tool.rules.max_count:
                    rule["max_count"] = tool.rules.max_count
                tool_rules.append(rule)

        # Create agent with curriculum config
        agent = self.client.agents.create(
            name=agent_name,
            model=course.agent.model,
            embedding=course.agent.embedding,
            system=course.agent.system if course.agent.system else None,
            memory_blocks=memory_blocks,
            tools=tool_names,
            tool_rules=tool_rules if tool_rules else None,
            metadata={
                **self._agent_metadata(user_id, agent_type),
                "course_id": course_id,
                "course_version": course.version,
            },
        )

        self._cache[(user_id, agent_type)] = agent.id

        log.info(
            "agent_created_from_curriculum",
            user_id=user_id,
            course_id=course_id,
            agent_id=agent.id,
            blocks=list(course.blocks.keys()),
            tools=tool_names,
        )

        return agent.id
```

#### 2. Update Server Lifespan

**File**: `src/letta_starter/server/main.py`

**Changes**: Initialize curriculum registry on startup

```python
# Add import
from letta_starter.curriculum import curriculum

# In lifespan function, after existing initialization
@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... existing Letta/Honcho initialization ...

    # Initialize curriculum
    curriculum.initialize(Path("config/courses"))
    courses_loaded = curriculum.load_all()
    log.info("curriculum_initialized", courses=courses_loaded)

    yield

    # ... existing cleanup ...
```

### Success Criteria

#### Automated Verification:
- [x] Agent creation uses curriculum config
- [x] Memory blocks match TOML schema
- [x] Tools match TOML configuration
- [x] Type checking passes: `make check-agent`
- [x] Integration tests pass

#### Manual Verification:
- [ ] Agents created with correct model
- [ ] Memory blocks serialize correctly
- [ ] Agent metadata includes course_id and version

**Implementation Note**: After completing this phase, pause for verification before proceeding.

---

## Phase 5: HTTP Endpoints

### Overview

Add curriculum management endpoints for listing courses, viewing config, and hot-reloading.

### Changes Required

#### 1. Create Curriculum Router

**File**: `src/letta_starter/server/curriculum.py` (new)

```python
"""Curriculum management HTTP endpoints."""

from __future__ import annotations

from pathlib import Path
from typing import Any

import structlog
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

from letta_starter.curriculum import curriculum

log = structlog.get_logger()

router = APIRouter(prefix="/curriculum", tags=["curriculum"])


# =============================================================================
# RESPONSE MODELS
# =============================================================================


class CourseListResponse(BaseModel):
    """Response for course list."""

    courses: list[str]
    count: int


class ModuleSummary(BaseModel):
    """Summary of a module."""

    id: str
    name: str
    order: int
    lesson_count: int


class BlockSummary(BaseModel):
    """Summary of a memory block."""

    name: str
    label: str
    field_count: int
    fields: list[str]


class CourseDetailResponse(BaseModel):
    """Detailed course information."""

    id: str
    name: str
    version: str
    description: str
    modules: list[ModuleSummary]
    blocks: list[BlockSummary]
    background_agents: list[str]
    tool_count: int


class ReloadResponse(BaseModel):
    """Response for config reload."""

    success: bool
    courses_loaded: int
    courses: list[str]
    message: str


# =============================================================================
# ENDPOINTS
# =============================================================================


@router.get("/courses", response_model=CourseListResponse)
async def list_courses() -> CourseListResponse:
    """List all available courses."""
    courses = curriculum.list_courses()
    return CourseListResponse(courses=courses, count=len(courses))


@router.get("/courses/{course_id}", response_model=CourseDetailResponse)
async def get_course(course_id: str) -> CourseDetailResponse:
    """Get detailed course configuration."""
    course = curriculum.get(course_id)
    if course is None:
        raise HTTPException(status_code=404, detail=f"Course not found: {course_id}")

    modules = [
        ModuleSummary(
            id=m.id,
            name=m.name,
            order=m.order,
            lesson_count=len(m.lessons),
        )
        for m in course.loaded_modules
    ]

    blocks = [
        BlockSummary(
            name=name,
            label=schema.label,
            field_count=len(schema.fields),
            fields=list(schema.fields.keys()),
        )
        for name, schema in course.blocks.items()
    ]

    return CourseDetailResponse(
        id=course.id,
        name=course.name,
        version=course.version,
        description=course.description,
        modules=modules,
        blocks=blocks,
        background_agents=list(course.background.keys()),
        tool_count=len(course.agent.tools),
    )


@router.get("/courses/{course_id}/full")
async def get_course_full(course_id: str) -> dict[str, Any]:
    """
    Get complete course configuration as JSON.

    This returns the full parsed configuration, useful for debugging
    or building UI editors.
    """
    course = curriculum.get(course_id)
    if course is None:
        raise HTTPException(status_code=404, detail=f"Course not found: {course_id}")

    # Return full config (Pydantic handles serialization)
    return course.model_dump(exclude={"loaded_modules"})


@router.get("/courses/{course_id}/modules")
async def get_course_modules(course_id: str) -> list[dict[str, Any]]:
    """Get all modules for a course with full lesson details."""
    course = curriculum.get(course_id)
    if course is None:
        raise HTTPException(status_code=404, detail=f"Course not found: {course_id}")

    return [m.model_dump() for m in course.loaded_modules]


@router.post("/reload", response_model=ReloadResponse)
async def reload_curriculum() -> ReloadResponse:
    """
    Reload all curriculum configurations from disk.

    This allows updating course configurations without restarting the server.
    """
    try:
        count = curriculum.reload()
        courses = curriculum.list_courses()

        log.info("curriculum_reloaded", courses=count)

        return ReloadResponse(
            success=True,
            courses_loaded=count,
            courses=courses,
            message=f"Successfully reloaded {count} course(s)",
        )
    except Exception as e:
        log.error("curriculum_reload_failed", error=str(e))
        return ReloadResponse(
            success=False,
            courses_loaded=0,
            courses=[],
            message=f"Reload failed: {e}",
        )
```

#### 2. Register Router

**File**: `src/letta_starter/server/main.py`

**Changes**: Add curriculum router

```python
# Add import
from letta_starter.server.curriculum import router as curriculum_router

# After app creation, add router
app.include_router(curriculum_router)
```

### Success Criteria

#### Automated Verification:
- [x] All tests pass: `make verify-agent`
- [x] OpenAPI schema includes curriculum endpoints
- [x] Response models serialize correctly

#### Manual Verification:
- [ ] `GET /curriculum/courses` lists courses
- [ ] `GET /curriculum/courses/{id}` returns config summary
- [ ] `GET /curriculum/courses/{id}/full` returns complete config
- [ ] `POST /curriculum/reload` reloads without restart

**Implementation Note**: After completing this phase, pause for verification before proceeding.

---

## Phase 6: Migration & Documentation

### Overview

Migrate the existing college-essay.toml to the new schema and create documentation.

### Changes Required

#### 1. Migrate Existing Config

**File**: `config/courses/college-essay/course.toml` (rewrite)

Create the full course.toml as shown in the schema design section above.

#### 2. Create Module Files

**File**: `config/courses/college-essay/modules/01-self-discovery.toml` (new)

Create module files as shown in the schema design section.

#### 3. Create Schema Documentation

**File**: `docs/config-schema.md` (new)

```markdown
# Course Configuration Schema

This document describes the TOML configuration schema for YouLab courses.

## Design Principles

- **Self-contained**: Each course.toml is complete without external references
- **Explicit**: All configuration is visible in the file
- **AI-friendly**: Easy to read and modify programmatically
- **UI-ready**: Schema is introspectable for visual editors

## File Structure

\`\`\`
config/courses/{course-id}/
├── course.toml          # Main course configuration
└── modules/
    ├── 01-module-name.toml
    └── 02-module-name.toml
\`\`\`

## course.toml Reference

### [course] - Course Metadata

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| id | string | yes | - | Unique identifier (kebab-case) |
| name | string | yes | - | Display name |
| version | string | no | "1.0.0" | Semantic version |
| description | string | no | "" | Course description |
| modules | list[string] | yes | - | Module file names to load |

### [agent] - Agent Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| model | string | "anthropic/claude-sonnet-4-20250514" | LLM model identifier |
| embedding | string | "openai/text-embedding-3-small" | Embedding model |
| context_window | int | 128000 | Context window size |
| system | string | "" | System prompt |

### [[agent.tools]] - Tool Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| id | string | - | Tool identifier |
| enabled | bool | true | Whether tool is enabled |
| rules.type | enum | "continue_loop" | "exit_loop", "continue_loop", "run_first" |

### [blocks.{name}] - Memory Block Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| label | string | yes | Letta's internal block label |
| description | string | no | Block description |

### [blocks.{name}.fields.{field}] - Field Schema

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| type | enum | - | "string", "int", "float", "bool", "list", "datetime" |
| default | any | type-specific | Default value |
| options | list[string] | null | Valid values (for dropdowns) |
| max | int | null | Max items for lists |
| description | string | null | Field description |
| required | bool | false | Whether field is required |

... [continue with full schema documentation]
\`\`\`

#### 4. Update CLAUDE.md

Add reference to new config system:

```markdown
## Configuration

Course configuration is in `config/courses/{course-id}/`:
- `course.toml` - Main configuration (agent, memory blocks, background agents)
- `modules/*.toml` - Module and lesson definitions

See `docs/config-schema.md` for full schema reference.
```

### Success Criteria

#### Automated Verification:
- [x] Migrated config loads without errors
- [x] All tests pass: `make verify-agent`
- [x] Server starts successfully with new config

#### Manual Verification:
- [ ] Documentation is clear and complete
- [ ] Example config is well-commented
- [ ] Schema matches implementation

---

## Testing Strategy

### Unit Tests

**New test files:**
- `tests/test_curriculum_schema.py` - Schema validation
- `tests/test_curriculum_blocks.py` - Dynamic block generation
- `tests/test_curriculum_loader.py` - TOML loading
- `tests/test_curriculum_integration.py` - AgentManager integration

### Integration Tests

- Load course → create agent → verify memory blocks match schema
- Modify TOML → reload → verify new config is active
- Create agent → chat → verify tools work

### Manual Testing Steps

1. Start server: `uv run letta-server`
2. `GET /curriculum/courses` - verify course listed
3. `GET /curriculum/courses/college-essay` - verify config returned
4. `POST /agents` with `course_id` - verify agent created
5. Check agent has correct memory blocks and tools
6. Modify course.toml, call `POST /curriculum/reload`
7. Create new agent - verify uses updated config

---

## Migration Notes

### New Dependencies

```toml
# pyproject.toml - for Python 3.10 compatibility
tomli = { version = "^2.0", python = "<3.11" }
```

### Backward Compatibility

- Existing `BackgroundAgentConfig` in `background/schema.py` will be replaced
- Existing `TUTOR_TEMPLATE` in `templates.py` remains as fallback
- `AgentManager.create_agent()` still works, new method is `create_agent_from_curriculum()`

### Environment Variables

```bash
# Optional: Override default config directory
YOULAB_CURRICULUM_DIR=/path/to/config/courses
```

---

## References

- Original Phase 5 plan: `thoughts/shared/plans/2026-01-08-phase-5-toml-curriculum.md`
- Background agents implementation: `src/letta_starter/background/`
- Memory blocks: `src/letta_starter/memory/blocks.py`
- Agent templates: `src/letta_starter/agents/templates.py`
