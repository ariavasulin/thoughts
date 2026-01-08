# Phase 5: TOML Curriculum Format Implementation Plan

## Overview

Implement a TOML-based curriculum format where memory block schemas, agent configuration, and course structure are defined in configuration files rather than hardcoded Python. This enables course iteration without code changes and integrates with Phase 6 background agents.

## Current State Analysis

### What Exists

- **Agent Templates** (`templates.py`): Hardcoded `TUTOR_TEMPLATE` with `PersonaBlock`/`HumanBlock`
- **Memory Blocks** (`blocks.py`): Fixed Pydantic models with predefined fields
- **Phase 6 Background Agents**: TOML schema and runner infrastructure (in progress)
- **No curriculum files**: Course content embedded in Python code

### Key Files

- `src/letta_starter/agents/templates.py` - AgentTemplate, TUTOR_TEMPLATE
- `src/letta_starter/memory/blocks.py` - PersonaBlock, HumanBlock (hardcoded schemas)
- `src/letta_starter/server/agents.py` - AgentManager (creates agents from templates)
- `src/letta_starter/background/schema.py` - Phase 6 TOML schema for background agents

## Desired End State

1. **course.toml** defines:
   - Course metadata
   - Agent configuration (model, tools, system prompt)
   - Memory block schemas (fully flexible - any fields, any types)
   - Background agent references (defined in Phase 6 schema)
   - Module list

2. **module.toml** defines:
   - Module metadata
   - Lessons (sequential progression)
   - Background agent activation/overrides per module
   - Per-lesson agent customization

3. **Dynamic memory blocks**:
   - Pydantic models generated from TOML at runtime
   - No hardcoded PersonaBlock/HumanBlock field requirements
   - Progress tracking as a configurable memory block

4. **Integration with Phase 6**:
   - Modules activate background agents when current
   - Background agents reference course-defined memory blocks

### Verification

```bash
# Load course config
curl localhost:8100/curriculum/college-essay
# → Returns parsed course config

# Create agent from curriculum
curl -X POST localhost:8100/agents -d '{"user_id": "test", "course_id": "college-essay"}'
# → Agent created with TOML-defined memory blocks

# Reload curriculum
curl -X POST localhost:8100/curriculum/reload
# → Reloads all TOML without restart
```

## What We're NOT Doing

- Complex lesson state machines (completion is tracked, not enforced)
- Multi-course enrollment per user (one course at a time)
- Visual curriculum editor (TOML files edited directly)
- Versioned curriculum migrations (reload replaces config)

---

## Implementation Approach

```
Phase 1: TOML Schema & Pydantic Models  ─► Define the format
Phase 2: Dynamic Memory Block Generator  ─► Memory blocks from TOML
Phase 3: Curriculum Loader & Registry    ─► Parse and cache configs
Phase 4: AgentManager Integration        ─► Create agents from curriculum
Phase 5: HTTP Endpoints & Reload         ─► Management API
```

---

## Phase 1: TOML Schema & Pydantic Models

### Overview

Define the Pydantic models that represent the TOML structure.

### Changes Required

#### 1. Create Curriculum Schema Module

**File**: `src/letta_starter/curriculum/schema.py` (new)

```python
"""TOML curriculum schema definitions."""

from __future__ import annotations

import sys
from enum import Enum
from typing import Any, Literal

from pydantic import BaseModel, Field

if sys.version_info >= (3, 11):
    import tomllib
else:
    import tomli as tomllib


class FieldType(str, Enum):
    """Supported field types for memory blocks."""
    STRING = "string"
    INT = "int"
    FLOAT = "float"
    BOOL = "bool"
    LIST = "list"
    DATETIME = "datetime"


class FieldSchema(BaseModel):
    """Schema for a single memory block field."""
    type: FieldType
    default: Any = None
    options: list[str] | None = None  # For enum-like fields
    max: int | None = None  # For lists
    description: str | None = None


class BlockSchema(BaseModel):
    """Schema for a memory block."""
    label: str  # Letta label (e.g., "persona", "human", "progress")
    description: str = ""
    fields: dict[str, FieldSchema] = Field(default_factory=dict)


class ToolRuleConfig(BaseModel):
    """Tool execution rule."""
    tool: str
    type: Literal["exit_loop", "run_first", "continue_loop", "max_count_per_step"]
    max_count: int | None = None


class AgentConfig(BaseModel):
    """Letta agent configuration."""
    model: str = "anthropic/claude-sonnet-4-20250514"
    embedding: str = "openai/text-embedding-3-small"
    system: str = ""
    tools: list[str] = Field(default_factory=lambda: ["query_honcho", "edit_memory_block"])
    tool_rules: list[ToolRuleConfig] = Field(default_factory=list)


class BackgroundAgentOverride(BaseModel):
    """Module-level background agent activation/override."""
    active: bool = True
    # Trigger overrides (None = use course defaults)
    schedule: str | None = None
    idle_minutes: int | None = None
    after_messages: int | None = None


class LessonAgentConfig(BaseModel):
    """Per-lesson agent customization."""
    opening: str | None = None
    focus: list[str] = Field(default_factory=list)
    constraints: list[str] = Field(default_factory=list)
    persona_overrides: dict[str, Any] = Field(default_factory=dict)


class LessonConfig(BaseModel):
    """Single lesson configuration."""
    id: str
    name: str
    objectives: list[str] = Field(default_factory=list)
    completion: list[str] = Field(default_factory=list)  # Completion criteria
    agent: LessonAgentConfig = Field(default_factory=LessonAgentConfig)


class ModuleConfig(BaseModel):
    """Module configuration (loaded from module.toml)."""
    id: str
    name: str
    order: int = 0
    description: str = ""
    background: dict[str, BackgroundAgentOverride] = Field(default_factory=dict)
    lessons: list[LessonConfig] = Field(default_factory=list)


class MessagesConfig(BaseModel):
    """UI message strings."""
    welcome_first: str = "Welcome!"
    welcome_returning: str = "Welcome back!"
    error_unavailable: str = "Service temporarily unavailable."


class CourseConfig(BaseModel):
    """Complete course configuration (loaded from course.toml)."""
    # Metadata
    id: str
    name: str
    version: str = "1.0.0"
    modules: list[str] = Field(default_factory=list)  # Module file references

    # Agent configuration
    agent: AgentConfig = Field(default_factory=AgentConfig)

    # Memory block schemas
    blocks: dict[str, BlockSchema] = Field(default_factory=dict)

    # Background agents (IDs reference Phase 6 definitions)
    background: dict[str, dict[str, Any]] = Field(default_factory=dict)

    # UI messages
    messages: MessagesConfig = Field(default_factory=MessagesConfig)

    # Loaded modules (populated after loading module files)
    loaded_modules: list[ModuleConfig] = Field(default_factory=list, exclude=True)
```

#### 2. Example course.toml

**File**: `config/courses/college-essay/course.toml` (new)

```toml
# =============================================================================
# COURSE METADATA
# =============================================================================
[course]
id = "college-essay"
name = "College Essay Writing"
version = "1.0.0"

modules = [
    "01-self-discovery",
    "02-topic-development",
    "03-drafting",
]

# =============================================================================
# AGENT CONFIGURATION
# =============================================================================
[agent]
model = "anthropic/claude-sonnet-4-20250514"
embedding = "openai/text-embedding-3-small"

system = """
You are a college essay coach helping students craft authentic personal narratives.

GUIDELINES:
- Never write essays for students
- Use Socratic questioning to help students discover their own voice
- Celebrate progress and small wins
"""

tools = ["query_honcho", "edit_memory_block"]

[[agent.tool_rules]]
tool = "send_message"
type = "exit_loop"

# =============================================================================
# MEMORY BLOCKS (schema-defined, fully flexible)
# =============================================================================

[blocks.persona]
label = "persona"
description = "Agent identity and behavior"

[blocks.persona.fields.name]
type = "string"
default = "YouLab Essay Coach"

[blocks.persona.fields.role]
type = "string"
default = "AI tutor for college application essays"

[blocks.persona.fields.tone]
type = "string"
default = "warm"
options = ["warm", "professional", "encouraging"]

[blocks.persona.fields.capabilities]
type = "list"
default = ["Guide self-discovery", "Brainstorm topics", "Give feedback"]

[blocks.persona.fields.constraints]
type = "list"
default = ["Never write essays for students"]


[blocks.student]
label = "human"
description = "Student context"

[blocks.student.fields.name]
type = "string"
default = ""

[blocks.student.fields.essay_topics]
type = "list"
default = []

[blocks.student.fields.strengths]
type = "list"
default = []
max = 10

[blocks.student.fields.writing_level]
type = "string"
default = "unknown"
options = ["unknown", "developing", "proficient", "advanced"]

[blocks.student.fields.facts]
type = "list"
default = []
max = 20


[blocks.progress]
label = "progress"
description = "Module progression status"

[blocks.progress.fields.current_module]
type = "string"
default = "01-self-discovery"

[blocks.progress.fields.current_lesson]
type = "string"
default = ""

[blocks.progress.fields.completed_lessons]
type = "list"
default = []


# =============================================================================
# BACKGROUND AGENTS (references Phase 6 definitions)
# =============================================================================

[background.insight-harvester]
enabled = true
schedule = "0 3 * * *"
idle_minutes = 30

[background.progress-tracker]
enabled = true
after_messages = 5


# =============================================================================
# UI MESSAGES
# =============================================================================
[messages]
welcome_first = "Welcome to YouLab! I'm your personal essay coach."
welcome_returning = "Welcome back! Ready to continue your essay journey?"
error_unavailable = "I'm taking a short break. Please try again in a moment."
```

#### 3. Example module.toml

**File**: `config/courses/college-essay/modules/01-self-discovery.toml` (new)

```toml
[module]
id = "01-self-discovery"
name = "Self-Discovery"
order = 1
description = "Help students uncover their unique stories"

# =============================================================================
# BACKGROUND AGENT OVERRIDES
# =============================================================================

[background.insight-harvester]
active = true
idle_minutes = 15  # More frequent during onboarding

[background.progress-tracker]
active = true

# =============================================================================
# LESSONS
# =============================================================================

[[lessons]]
id = "strengths-assessment"
name = "Strengths Assessment"

objectives = [
    "Complete StrengthsFinder assessment",
    "Share top 5 strengths",
    "Identify most resonant strength",
]

completion = [
    "mentioned >= 3 strengths",
    "articulated why one resonates",
    "turns >= 3",
]

[lessons.agent]
opening = """
Welcome to your essay journey! Before we start writing,
let's discover what makes you uniquely you.
Have you completed your StrengthsFinder assessment?
"""

focus = ["strengths", "talents", "achievements"]

constraints = [
    "Ask about specific moments when strengths appeared",
]


[[lessons]]
id = "processing-results"
name = "Processing Your Results"

objectives = [
    "Deep dive into top 2-3 strengths",
    "Connect to specific life stories",
]

completion = [
    "shared >= 2 stories",
    "identified essay angle",
]

[lessons.agent]
focus = ["stories", "memories", "emotions", "details"]

[lessons.agent.persona_overrides]
tone = "encouraging"
```

### Success Criteria

#### Automated Verification:
- [ ] Schema models validate correctly: `make check-agent`
- [ ] Example TOML parses without errors
- [ ] Unit tests for schema validation pass

#### Manual Verification:
- [ ] TOML syntax is readable and intuitive
- [ ] All expected fields are representable

---

## Phase 2: Dynamic Memory Block Generator

### Overview

Generate Pydantic models from TOML block schemas at runtime, replacing hardcoded PersonaBlock/HumanBlock.

### Changes Required

#### 1. Create Memory Block Generator

**File**: `src/letta_starter/curriculum/blocks.py` (new)

```python
"""Dynamic memory block generation from TOML schemas."""

from __future__ import annotations

from datetime import datetime
from typing import Any, get_type_hints

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
    """
    field_definitions: dict[str, tuple[type, Any]] = {}

    for field_name, field_schema in schema.fields.items():
        python_type = _get_python_type(field_schema)
        default = _get_default(field_schema)
        field_definitions[field_name] = (python_type, default)

    # Create the model dynamically
    model = create_model(
        name,
        __doc__=schema.description,
        **field_definitions,
    )

    # Add serialization methods
    model.to_memory_string = _create_to_memory_string(schema)
    model.from_memory_string = classmethod(_create_from_memory_string(schema))
    model._schema = schema
    model._label = schema.label

    return model


def _get_python_type(field_schema: FieldSchema) -> type:
    """Convert TOML field type to Python type."""
    base_type = TYPE_MAP.get(field_schema.type, str)

    if field_schema.type == FieldType.LIST:
        return list[str]  # Lists are always list[str] for simplicity

    return base_type


def _get_default(field_schema: FieldSchema) -> Any:
    """Get default value for a field."""
    if field_schema.default is None:
        if field_schema.type == FieldType.LIST:
            return Field(default_factory=list)
        elif field_schema.type == FieldType.STRING:
            return ""
        elif field_schema.type in (FieldType.INT, FieldType.FLOAT):
            return 0
        elif field_schema.type == FieldType.BOOL:
            return False
        else:
            return None

    if field_schema.type == FieldType.LIST:
        return Field(default_factory=lambda d=field_schema.default: list(d))

    return field_schema.default


def _create_to_memory_string(schema: BlockSchema):
    """Create a to_memory_string method for the generated model."""

    def to_memory_string(self, max_chars: int = 1500) -> str:
        """Serialize block to compact memory format."""
        lines = []

        for field_name, field_schema in schema.fields.items():
            value = getattr(self, field_name, None)
            if value is None or value == "" or value == []:
                continue

            label = field_name.upper()

            if isinstance(value, list):
                # Limit list items for token efficiency
                max_items = field_schema.max or 5
                value_str = "; ".join(str(v) for v in value[:max_items])
            else:
                value_str = str(value)

            lines.append(f"[{label}] {value_str}")

        result = "\n".join(lines)
        return result[:max_chars]

    return to_memory_string


def _create_from_memory_string(schema: BlockSchema):
    """Create a from_memory_string classmethod for the generated model."""

    def from_memory_string(cls, memory_str: str) -> BaseModel:
        """Parse memory string back into block instance."""
        data: dict[str, Any] = {}

        for line in memory_str.strip().split("\n"):
            if not line.startswith("["):
                continue

            # Parse [LABEL] value format
            bracket_end = line.find("]")
            if bracket_end == -1:
                continue

            label = line[1:bracket_end].lower()
            value = line[bracket_end + 1:].strip()

            # Find matching field
            for field_name, field_schema in schema.fields.items():
                if field_name.lower() == label:
                    if field_schema.type == FieldType.LIST:
                        data[field_name] = [v.strip() for v in value.split(";")]
                    elif field_schema.type == FieldType.INT:
                        data[field_name] = int(value) if value else 0
                    elif field_schema.type == FieldType.FLOAT:
                        data[field_name] = float(value) if value else 0.0
                    elif field_schema.type == FieldType.BOOL:
                        data[field_name] = value.lower() in ("true", "1", "yes")
                    else:
                        data[field_name] = value
                    break

        return cls(**data)

    return from_memory_string


class DynamicBlockRegistry:
    """Registry for dynamically generated memory block models."""

    def __init__(self) -> None:
        self._models: dict[str, type[BaseModel]] = {}

    def register_from_schema(
        self,
        block_name: str,
        schema: BlockSchema,
    ) -> type[BaseModel]:
        """Generate and register a block model from schema."""
        model_name = f"{block_name.title()}Block"
        model = generate_block_model(model_name, schema)
        self._models[block_name] = model
        return model

    def get(self, block_name: str) -> type[BaseModel] | None:
        """Get a registered block model."""
        return self._models.get(block_name)

    def get_all(self) -> dict[str, type[BaseModel]]:
        """Get all registered block models."""
        return self._models.copy()

    def clear(self) -> None:
        """Clear all registered models."""
        self._models.clear()
```

### Success Criteria

#### Automated Verification:
- [ ] Generated models serialize/deserialize correctly
- [ ] Type checking passes: `make check-agent`
- [ ] Unit tests for block generation pass

#### Manual Verification:
- [ ] Generated model instances work with MemoryManager
- [ ] Serialization format is token-efficient

---

## Phase 3: Curriculum Loader & Registry

### Overview

Create the curriculum loading system that parses TOML files and caches parsed configs.

### Changes Required

#### 1. Create Curriculum Loader

**File**: `src/letta_starter/curriculum/loader.py` (new)

```python
"""Curriculum loading and registry."""

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
    BlockSchema,
    CourseConfig,
    FieldSchema,
    ModuleConfig,
)

log = structlog.get_logger()


class CurriculumLoader:
    """Loads and parses curriculum TOML files."""

    def __init__(self, base_dir: Path) -> None:
        self.base_dir = base_dir
        self.block_registry = DynamicBlockRegistry()

    def load_course(self, course_id: str) -> CourseConfig:
        """
        Load a course and its modules.

        Args:
            course_id: Course directory name

        Returns:
            Fully loaded CourseConfig with modules
        """
        course_dir = self.base_dir / course_id
        course_file = course_dir / "course.toml"

        if not course_file.exists():
            raise FileNotFoundError(f"Course config not found: {course_file}")

        # Load course.toml
        with open(course_file, "rb") as f:
            raw_data = tomllib.load(f)

        # Flatten nested structure for Pydantic
        course_data = self._flatten_course_data(raw_data)
        course = CourseConfig(**course_data)

        # Generate memory block models
        for block_name, block_schema in course.blocks.items():
            self.block_registry.register_from_schema(block_name, block_schema)

        # Load modules
        modules_dir = course_dir / "modules"
        if modules_dir.exists():
            for module_ref in course.modules:
                module_file = modules_dir / f"{module_ref}.toml"
                if module_file.exists():
                    module = self._load_module(module_file)
                    course.loaded_modules.append(module)

        # Sort modules by order
        course.loaded_modules.sort(key=lambda m: m.order)

        log.info(
            "course_loaded",
            course_id=course.id,
            modules=len(course.loaded_modules),
            blocks=list(course.blocks.keys()),
        )

        return course

    def _flatten_course_data(self, raw: dict[str, Any]) -> dict[str, Any]:
        """Flatten nested TOML structure for Pydantic parsing."""
        result: dict[str, Any] = {}

        # Course metadata
        if "course" in raw:
            result.update(raw["course"])

        # Agent config
        if "agent" in raw:
            result["agent"] = raw["agent"]

        # Blocks - need to convert nested field definitions
        if "blocks" in raw:
            blocks = {}
            for block_name, block_data in raw["blocks"].items():
                fields = {}
                for key, value in block_data.items():
                    if key in ("label", "description"):
                        continue
                    if key == "fields":
                        for field_name, field_data in value.items():
                            fields[field_name] = FieldSchema(**field_data)
                    elif isinstance(value, dict) and "type" in value:
                        # Inline field definition
                        fields[key] = FieldSchema(**value)

                blocks[block_name] = BlockSchema(
                    label=block_data.get("label", block_name),
                    description=block_data.get("description", ""),
                    fields=fields,
                )
            result["blocks"] = blocks

        # Background agents
        if "background" in raw:
            result["background"] = raw["background"]

        # Messages
        if "messages" in raw:
            result["messages"] = raw["messages"]

        return result

    def _load_module(self, path: Path) -> ModuleConfig:
        """Load a single module file."""
        with open(path, "rb") as f:
            raw = tomllib.load(f)

        # Flatten module structure
        module_data = raw.get("module", {})
        module_data["background"] = raw.get("background", {})
        module_data["lessons"] = raw.get("lessons", [])

        return ModuleConfig(**module_data)


class CurriculumRegistry:
    """Registry for loaded curriculum configurations."""

    def __init__(self) -> None:
        self._courses: dict[str, CourseConfig] = {}
        self._loader: CurriculumLoader | None = None

    def initialize(self, base_dir: Path) -> None:
        """Initialize the registry with a base directory."""
        self._loader = CurriculumLoader(base_dir)

    def load_all(self) -> int:
        """Load all courses from base directory."""
        if self._loader is None:
            raise RuntimeError("Registry not initialized")

        count = 0
        for course_dir in self._loader.base_dir.iterdir():
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

        return count

    def get(self, course_id: str) -> CourseConfig | None:
        """Get a loaded course by ID."""
        return self._courses.get(course_id)

    def get_block_registry(self, course_id: str) -> DynamicBlockRegistry | None:
        """Get the block registry for a course."""
        if self._loader is None:
            return None
        return self._loader.block_registry

    def list_courses(self) -> list[str]:
        """List all loaded course IDs."""
        return list(self._courses.keys())

    def reload(self) -> int:
        """Reload all courses."""
        self._courses.clear()
        if self._loader:
            self._loader.block_registry.clear()
        return self.load_all()


# Global registry instance
curriculum = CurriculumRegistry()
```

### Success Criteria

#### Automated Verification:
- [ ] Loader parses example TOML correctly
- [ ] Registry loads multiple courses
- [ ] Unit tests pass: `make test-agent`

#### Manual Verification:
- [ ] Courses load without errors
- [ ] Module ordering is correct
- [ ] Block models are generated

---

## Phase 4: AgentManager Integration

### Overview

Update AgentManager to create agents from curriculum config instead of hardcoded templates.

### Changes Required

#### 1. Update AgentManager

**File**: `src/letta_starter/server/agents.py`

**Changes**: Add curriculum-based agent creation

```python
from letta_starter.curriculum.loader import curriculum
from letta_starter.curriculum.blocks import DynamicBlockRegistry


class AgentManager:
    # ... existing code ...

    def create_agent_from_curriculum(
        self,
        user_id: str,
        course_id: str,
        user_name: str | None = None,
    ) -> str:
        """
        Create an agent based on curriculum configuration.

        Args:
            user_id: User identifier
            course_id: Course to use for agent config
            user_name: Optional user name for student block

        Returns:
            Agent ID
        """
        course = curriculum.get(course_id)
        if course is None:
            raise ValueError(f"Unknown course: {course_id}")

        agent_type = course_id  # Use course_id as agent_type
        agent_name = self._agent_name(user_id, agent_type)

        # Check for existing agent
        existing = self.get_agent_id(user_id, agent_type)
        if existing:
            return existing

        # Get block registry for this course
        block_registry = curriculum.get_block_registry(course_id)

        # Build memory blocks from course schema
        memory_blocks = []
        for block_name, block_schema in course.blocks.items():
            model_class = block_registry.get(block_name)
            if model_class is None:
                continue

            # Create instance with defaults
            instance = model_class()

            # Inject user name into student block
            if block_schema.label == "human" and user_name:
                if hasattr(instance, "name"):
                    instance.name = user_name

            memory_blocks.append({
                "label": block_schema.label,
                "value": instance.to_memory_string(),
            })

        # Create agent with curriculum config
        agent = self.client.agents.create(
            name=agent_name,
            model=course.agent.model,
            embedding=course.agent.embedding,
            system=course.agent.system if course.agent.system else None,
            memory_blocks=memory_blocks,
            tools=course.agent.tools,
            tool_rules=[
                {"tool_name": r.tool, "type": r.type, "max_count": r.max_count}
                for r in course.agent.tool_rules
            ] if course.agent.tool_rules else None,
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
        )

        return agent.id
```

#### 2. Update Server Lifespan

**File**: `src/letta_starter/server/main.py`

**Changes**: Initialize curriculum registry

```python
from letta_starter.curriculum.loader import curriculum

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... existing initialization ...

    # Initialize curriculum
    curriculum.initialize(Path("config/courses"))
    courses_loaded = curriculum.load_all()
    log.info("curriculum_initialized", courses=courses_loaded)

    yield
```

### Success Criteria

#### Automated Verification:
- [ ] Agent creation uses curriculum config
- [ ] Memory blocks match TOML schema
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Agents created with correct model/tools
- [ ] Memory blocks serialize correctly
- [ ] Metadata includes course_id

---

## Phase 5: HTTP Endpoints & Reload

### Overview

Add curriculum management endpoints.

### Changes Required

#### 1. Create Curriculum Router

**File**: `src/letta_starter/server/curriculum.py` (new)

```python
"""Curriculum management endpoints."""

from pathlib import Path

import structlog
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

from letta_starter.curriculum.loader import curriculum
from letta_starter.curriculum.schema import CourseConfig

log = structlog.get_logger()

router = APIRouter(prefix="/curriculum", tags=["curriculum"])


class CourseListResponse(BaseModel):
    """Response for course list."""
    courses: list[str]
    count: int


class ReloadResponse(BaseModel):
    """Response for config reload."""
    success: bool
    courses_loaded: int
    courses: list[str]
    message: str


@router.get("/courses", response_model=CourseListResponse)
async def list_courses() -> CourseListResponse:
    """List all available courses."""
    courses = curriculum.list_courses()
    return CourseListResponse(courses=courses, count=len(courses))


@router.get("/courses/{course_id}")
async def get_course(course_id: str) -> dict:
    """Get course configuration."""
    course = curriculum.get(course_id)
    if course is None:
        raise HTTPException(404, f"Course not found: {course_id}")

    return {
        "id": course.id,
        "name": course.name,
        "version": course.version,
        "modules": [
            {
                "id": m.id,
                "name": m.name,
                "order": m.order,
                "lessons": len(m.lessons),
            }
            for m in course.loaded_modules
        ],
        "blocks": list(course.blocks.keys()),
        "background_agents": list(course.background.keys()),
    }


@router.post("/reload", response_model=ReloadResponse)
async def reload_curriculum() -> ReloadResponse:
    """Reload all curriculum configurations."""
    try:
        count = curriculum.reload()
        courses = curriculum.list_courses()

        log.info("curriculum_reloaded", courses=count)

        return ReloadResponse(
            success=True,
            courses_loaded=count,
            courses=courses,
            message="Curriculum reloaded successfully",
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
from letta_starter.server.curriculum import router as curriculum_router

# After app creation
app.include_router(curriculum_router)
```

### Success Criteria

#### Automated Verification:
- [ ] All tests pass: `make verify-agent`
- [ ] OpenAPI schema includes curriculum endpoints

#### Manual Verification:
- [ ] `GET /curriculum/courses` lists courses
- [ ] `GET /curriculum/courses/{id}` returns config
- [ ] `POST /curriculum/reload` reloads without restart

---

## Testing Strategy

### Unit Tests

**New test files:**
- `tests/test_curriculum_schema.py` - Schema validation
- `tests/test_curriculum_blocks.py` - Dynamic block generation
- `tests/test_curriculum_loader.py` - TOML loading
- `tests/test_curriculum_integration.py` - AgentManager integration

### Integration Tests

- Load course → create agent → verify memory blocks
- Modify TOML → reload → verify new config
- Module progression updates progress block

### Manual Testing Steps

1. Start service with example curriculum
2. `GET /curriculum/courses` - verify course listed
3. `POST /agents` with `course_id` - verify agent created
4. `GET /agents/{id}` - verify memory blocks match schema
5. Modify course.toml, call `/curriculum/reload`
6. Create new agent - verify uses new config

---

## Migration Notes

### New Dependencies

```toml
# pyproject.toml - Python 3.10 compatibility
tomli = { version = "^2.0", python = "<3.11" }
```

### New Directory Structure

```
config/
  courses/
    college-essay/
      course.toml
      modules/
        01-self-discovery.toml
        02-topic-development.toml
        03-drafting.toml

src/letta_starter/
  curriculum/
    __init__.py
    schema.py
    blocks.py
    loader.py
  server/
    curriculum.py  (new router)
```

### Environment Variables

```bash
YOULAB_CURRICULUM_DIR=/path/to/config/courses  # Optional override
```

---

## References

- Phase 6 Background Agents: `thoughts/shared/plans/2026-01-08-phase-6-background-agents.md`
- Letta AF Research: `thoughts/shared/research/2026-01-08-letta-af-yaml-integration.md`
- Sleep Agent TOML Schema: `thoughts/shared/plans/2026-01-07-sleep-agents-toml-schema.toml`
- Roadmap Phase 5: `docs/Roadmap.md:169-202`
