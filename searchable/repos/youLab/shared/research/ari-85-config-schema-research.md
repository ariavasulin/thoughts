# ARI-85 Config Schema Research

**Research Date**: 2026-01-16
**Objective**: Thoroughly document the current config schema for courses to support creation of proof-of-concept single-module course with background agent
**Status**: Complete

---

## Executive Summary

The YouLab course configuration system is based on **TOML files** with **v2 schema** currently in production. There is **NO MD-based configuration** - all course/module/step configuration is TOML-based. The system has migrated from v1 to v2 schema successfully, with backwards compatibility maintained in the loader.

**Key Finding**: To create a new course with background agent for ARI-85, you'll use the v2 TOML schema with `[[task]]` entries, NOT markdown configuration.

---

## Table of Contents

1. [Schema Version History](#schema-version-history)
2. [File Structure](#file-structure)
3. [V2 Schema Components](#v2-schema-components)
4. [Pydantic Models (Schema Validation)](#pydantic-models-schema-validation)
5. [Config Loading Mechanism](#config-loading-mechanism)
6. [Dynamic Block Generation](#dynamic-block-generation)
7. [Background Tasks (Critical for ARI-85)](#background-tasks-critical-for-ari-85)
8. [Module Metadata (OpenWebUI Only)](#module-metadata-openwebui-only)
9. [Creating a New Course - Step by Step](#creating-a-new-course---step-by-step)
10. [Verification](#verification)

---

## Schema Version History

### V1 Schema (Legacy - Deprecated)
- Separate `[course]` and `[agent]` sections
- Nested block syntax: `[blocks.x.fields.y]`
- Explicit tool configuration: `[[agent.tools]]` with full rules
- Named background agents: `[background.name]`

**Status**: Supported for backwards compatibility, emits deprecation warnings

### V2 Schema (Current - Production)
- Merged `[agent]` section (combines course + agent metadata)
- Flattened block syntax: `[block.x]` with `field.*` dotted keys
- Simple tool list: `tools = ["name"]` with registry defaults
- Task array: `[[task]]` for background agents
- **New feature**: `shared = true` flag for cross-agent memory blocks

**Implementation Status**: ✅ Fully implemented and deployed
- All phases from plan `2025-01-09-config-schema-v2.md` are complete
- Both `college-essay` and `default` courses migrated to v2
- Loader supports both v1 and v2 for backwards compatibility

**Files**:
- Schema: `src/youlab_server/curriculum/schema.py:1-356`
- Loader: `src/youlab_server/curriculum/loader.py:1-456`
- Documentation: `docs/config-schema.md:1-453`

---

## File Structure

```
config/courses/{course-id}/
├── course.toml              # Main course configuration (v2 schema)
└── modules/
    ├── 01-module-name.toml  # Module with steps
    └── 02-module-name.toml
```

**Current Courses**:
- `config/courses/college-essay/` - Primary course with 3 modules
- `config/courses/default/` - Default fallback course
- `config/course-new.toml` - Root-level file (purpose unclear, appears to be v1 format)

**Module Files Found**:
- `college-essay/modules/01-first-impression.toml` (v2 format, CliftonStrengths-focused)
- `college-essay/modules/01-self-discovery.toml` (older format)
- `college-essay/modules/02-topic-development.toml`
- `college-essay/modules/03-drafting.toml`

---

## V2 Schema Components

### 1. Agent Configuration (`[agent]`)

**Merged course + agent metadata in single section**

```toml
[agent]
id = "college-essay"                    # Unique course identifier
name = "College Essay Coaching"          # Display name
version = "2.0.0"                        # Semantic version
description = "AI-powered tutoring..."   # Course description
modules = ["01-first-impression", ...]   # Module load order
model = "openai/gpt-4o"                  # LLM model
embedding = "openai/text-embedding-3-small"
context_window = 128000
max_response_tokens = 4096
system = """..."""                       # System prompt
tools = ["send_message", "edit_memory_block", "advance_lesson", "query_honcho"]
```

**Key Fields**:
- `id`: Must match course directory name
- `modules`: List of module file names (without `.toml`)
- `tools`: Simple string list (uses registry defaults, can override with `:rule`)

**Folder Configuration** (optional):
```toml
[agent.folders]
shared = ["college-essay-materials"]  # Shared course materials
private = true                         # User gets private folder
```

**Schema Location**: `src/youlab_server/curriculum/schema.py:282-310`

---

### 2. Memory Blocks (`[block.{name}]`)

**Flattened dotted-key syntax for fields**

```toml
[block.student]
label = "human"
shared = false
description = "Rich narrative understanding of who this student is"
field.profile = { type = "string", default = "", description = "1-2 paragraph narrative..." }
field.insights = { type = "string", default = "", description = "Key observations..." }

[block.engagement_strategy]
label = "persona"
shared = false
description = "Adapted coaching approach for this specific student"
field.approach = { type = "string", default = "...", description = "..." }

[block.journey]
label = "journey"
shared = false
description = "Curriculum progress and grader state"
field.module_id = { type = "string", default = "01-first-impression" }
field.lesson_id = { type = "string", default = "strengths-upload" }
field.status = { type = "string", default = "in_progress", options = ["not_started", "in_progress", "completed"] }
field.grader_notes = { type = "string", default = "" }
field.blockers = { type = "string", default = "" }
field.milestones = { type = "list", default = [], max = 30 }
```

**Field Types** (from `FieldType` enum):
- `string` - Text field
- `int` - Integer
- `float` - Floating point
- `bool` - Boolean
- `list` - List of strings
- `datetime` - Datetime (nullable)

**FieldSchema Properties**:
- `type`: Required, one of above types
- `default`: Default value (type-specific)
- `options`: List of valid values (for dropdowns)
- `max`: Max items (for lists)
- `description`: Field description
- `required`: Whether field is required (default: false)

**Shared Blocks**:
- `shared = true` - Block created once per course, attached to all agents
- Use case: Team knowledge, organization policies
- Implementation: `AgentManager._get_or_create_shared_block()` in `src/youlab_server/server/agents.py:53-112`

**Schema Location**: `src/youlab_server/curriculum/schema.py:197-215`

---

### 3. Background Tasks (`[[task]]`)

**Array of task configurations for background agents**

```toml
[[task]]
name = "progression-grader"              # Task name (optional)
on_idle = true                           # Trigger on user idle
idle_threshold_minutes = 5               # Minutes idle before trigger
idle_cooldown_minutes = 30               # Cooldown between runs
manual = true                            # Allow manual trigger
agent_types = ["tutor", "college-essay"] # Agent types to process
user_filter = "all"                      # User filter
batch_size = 50                          # Users per batch
system = """You are a curriculum grader..."""  # System prompt for task agent

queries = [
    {
        target = "journey.grader_notes",
        question = "Assess student's progress...",
        scope = "all",
        merge = "replace"
    },
    {
        target = "journey.blockers",
        question = "What gaps prevent progression?",
        scope = "all",
        merge = "replace"
    },
    {
        target = "student.insights",
        question = "What new insights emerged?",
        scope = "all",
        merge = "append"
    }
]
```

**Task Scheduling Options**:
- `schedule`: Cron expression (e.g., `"0 3 * * *"` for 3 AM daily)
- `manual`: Allow manual trigger via API (default: true)
- `on_idle`: Trigger when user is idle
- `idle_threshold_minutes`: Minutes of inactivity before trigger
- `idle_cooldown_minutes`: Cooldown between idle triggers

**Query Configuration**:
- `target`: Block.field format (e.g., `"journey.grader_notes"`)
- `question`: Question to ask about conversation history
- `scope`: Session scope - `"all"`, `"recent"`, `"current"`, `"specific"`
- `recent_limit`: Number of recent sessions (if scope = "recent")
- `merge`: Merge strategy - `"append"`, `"replace"`, `"llm_diff"`

**Advanced Features**:
- `system`: Custom system prompt (full agent capabilities)
- `tools`: List of tools for task agent (for complex operations)

**Schema Location**: `src/youlab_server/curriculum/schema.py:169-190`

---

### 4. Modules (`[module]`) and Steps (`[[steps]]`)

**Module file structure**

```toml
[module]
id = "01-first-impression"
name = "First Impression"
order = 1
description = "Make a memorable first impression..."
disabled_tools = ["query_honcho"]  # Module-level tool restrictions

[[steps]]
id = "strengths-upload"
name = "CliftonStrengths Upload"
order = 1
description = "Welcome the student and analyze their CliftonStrengths PDF"
objectives = [
    "Create warm, engaging first impression",
    "Get student to upload their CliftonStrengths PDF",
    "Analyze the PDF thoroughly and reflect back insights"
]

[steps.completion]
min_turns = 3

[steps.agent]
opening = """Welcome to YouLab! I'm thrilled..."""
focus = ["rapport", "clifton_strengths", "pdf_analysis"]
guidance = [
    "Be warm, enthusiastic, genuinely curious",
    "If no CliftonStrengths: explain why it matters...",
    "When PDF uploaded: analyze thoroughly..."
]
disabled_tools = ["advance_lesson"]  # Step-level tool restrictions
```

**Step Completion Criteria**:
- `required_fields`: List of fields that must be non-empty (e.g., `["human.name"]`)
- `min_turns`: Minimum conversation turns
- `min_list_length`: Dict of field -> min items (e.g., `{"human.facts" = 3}`)
- `auto_advance`: Auto-advance when complete (default: false)

**Schema Location**:
- Module: `src/youlab_server/curriculum/schema.py:253-262`
- Step: `src/youlab_server/curriculum/schema.py:241-251`

---

### 5. UI Messages (`[messages]`)

```toml
[messages]
welcome_first = "Welcome to YouLab!..."
welcome_returning = "Welcome back!..."
error_unavailable = "I'm having a moment..."
```

**Schema Location**: `src/youlab_server/curriculum/schema.py:269-275`

---

## Pydantic Models (Schema Validation)

All TOML configurations are validated against Pydantic models defined in `src/youlab_server/curriculum/schema.py`.

### Core Models

**1. `CourseConfig`** (Root model)
```python
class CourseConfig(BaseModel):
    agent: AgentConfig
    blocks: dict[str, BlockSchema]
    background: dict[str, BackgroundAgentConfig]  # v1 legacy
    tasks: list[TaskConfig]                       # v2
    messages: MessagesConfig
    loaded_modules: list[ModuleConfig]
```
Location: `src/youlab_server/curriculum/schema.py:316-356`

**2. `AgentConfig`** (Agent + Course metadata)
```python
class AgentConfig(BaseModel):
    id: str
    name: str
    version: str = "1.0.0"
    description: str = ""
    modules: list[str]
    model: str
    embedding: str
    context_window: int
    max_response_tokens: int
    system: str
    folders: AgentFoldersConfig
    tools: list[ToolConfig]  # v1 or v2 format
```
Location: `src/youlab_server/curriculum/schema.py:282-310`

**3. `BlockSchema`** (Memory block definition)
```python
class BlockSchema(BaseModel):
    label: str
    description: str = ""
    shared: bool = False
    fields: dict[str, FieldSchema]
```
Location: `src/youlab_server/curriculum/schema.py:208-215`

**4. `FieldSchema`** (Block field definition)
```python
class FieldSchema(BaseModel):
    type: FieldType
    default: Any = None
    options: list[str] | None = None
    max: int | None = None
    description: str | None = None
    required: bool = False
```
Location: `src/youlab_server/curriculum/schema.py:197-206`

**5. `TaskConfig`** (Background task - v2)
```python
class TaskConfig(BaseModel):
    schedule: str | None = None
    manual: bool = True
    on_idle: bool = False
    idle_threshold_minutes: int = 30
    idle_cooldown_minutes: int = 60
    agent_types: list[str]
    user_filter: str = "all"
    batch_size: int = 50
    queries: list[QueryConfig]
    system: str | None = None
    tools: list[str]
```
Location: `src/youlab_server/curriculum/schema.py:169-190`

**6. `QueryConfig`** (Dialectic query within task)
```python
class QueryConfig(BaseModel):
    target: str              # "block.field" format
    question: str
    scope: SessionScope      # Enum: ALL, RECENT, CURRENT, SPECIFIC
    recent_limit: int = 5
    merge: MergeStrategy     # Enum: APPEND, REPLACE, LLM_DIFF

    @property
    def target_block(self) -> str:
        return self.target.split(".")[0]

    @property
    def target_field(self) -> str:
        parts = self.target.split(".")
        return parts[1] if len(parts) > 1 else ""
```
Location: `src/youlab_server/curriculum/schema.py:148-167`

**7. `ModuleConfig`** (Module metadata)
```python
class ModuleConfig(BaseModel):
    id: str
    name: str
    order: int = 0
    description: str = ""
    steps: list[StepConfig]
    disabled_tools: list[str]
```
Location: `src/youlab_server/curriculum/schema.py:253-262`

**8. `StepConfig`** (Step metadata)
```python
class StepConfig(BaseModel):
    id: str
    name: str
    order: int = 0
    description: str = ""
    objectives: list[str]
    completion: StepCompletion
    agent: StepAgent
```
Location: `src/youlab_server/curriculum/schema.py:241-251`

### Enums

```python
class FieldType(str, Enum):
    STRING = "string"
    INT = "int"
    FLOAT = "float"
    BOOL = "bool"
    LIST = "list"
    DATETIME = "datetime"

class SessionScope(str, Enum):
    ALL = "all"
    RECENT = "recent"
    CURRENT = "current"
    SPECIFIC = "specific"

class MergeStrategy(str, Enum):
    APPEND = "append"
    REPLACE = "replace"
    LLM_DIFF = "llm_diff"

class ToolRuleType(str, Enum):
    EXIT_LOOP = "exit_loop"
    CONTINUE_LOOP = "continue_loop"
    RUN_FIRST = "run_first"
```
Location: `src/youlab_server/curriculum/schema.py:20-54`

---

## Config Loading Mechanism

### CurriculumLoader

**File**: `src/youlab_server/curriculum/loader.py:43-456`

**Initialization**:
```python
loader = CurriculumLoader(config_dir="config/courses")
```
Defaults to `config/courses/` from project root.

**Key Methods**:

1. **`list_courses() -> list[str]`**
   - Returns list of available course IDs (directory names)
   - Only includes directories with `course.toml`

2. **`load_course(course_id: str, force: bool = False) -> CourseConfig`**
   - Loads and caches course configuration
   - Raises `FileNotFoundError` if course.toml doesn't exist
   - Raises `ValueError` if TOML is invalid
   - Auto-detects v1 vs v2 schema

3. **`get(course_id: str) -> CourseConfig | None`**
   - Safe getter, returns None if not found

4. **`reload() -> int`**
   - Clears cache and reloads all courses
   - Returns count of courses loaded successfully

### Schema Version Detection

**Location**: `src/youlab_server/curriculum/loader.py:140-190`

```python
def _parse_course_config(self, data: dict[str, Any]) -> CourseConfig:
    # Detect schema version
    has_v2_agent = "agent" in data and "id" in data.get("agent", {})
    has_v1_course = "course" in data
    has_v2_blocks = "block" in data
    has_v1_blocks = "blocks" in data

    # Parse agent config (v2 or v1)
    if has_v2_agent:
        agent = self._parse_agent_config(data["agent"], is_v2=True)
    elif has_v1_course:
        # Merge [course] + [agent]
        merged = {**data.get("agent", {})}
        merged.update({
            "id": data["course"]["id"],
            "name": data["course"]["name"],
            # ... other course fields
        })
        agent = self._parse_agent_config(merged, is_v2=False)

    # Parse blocks (v2 or v1)
    if has_v2_blocks:
        blocks = self._parse_blocks_v2(data["block"])
    elif has_v1_blocks:
        blocks = self._parse_blocks_v1(data["blocks"])

    # Parse tasks
    tasks = self._parse_tasks(data.get("task", []))

    return CourseConfig(agent=agent, blocks=blocks, tasks=tasks, ...)
```

### Block Parsing (V2)

**Location**: `src/youlab_server/curriculum/loader.py:294-323`

Handles TOML's dotted-key syntax: `field.name = {...}` is parsed as `{"field": {"name": {...}}}`

```python
def _parse_blocks_v2(self, blocks_data: dict[str, Any]) -> dict[str, BlockSchema]:
    blocks = {}
    for block_name, block_data in blocks_data.items():
        fields = {}

        # TOML parses field.name as {"field": {"name": {...}}}
        field_dict = block_data.get("field", {})
        for field_name, field_value in field_dict.items():
            fields[field_name] = FieldSchema(
                type=FieldType(field_value.get("type", "string")),
                default=field_value.get("default"),
                options=field_value.get("options"),
                max=field_value.get("max"),
                description=field_value.get("description"),
                required=field_value.get("required", False),
            )

        blocks[block_name] = BlockSchema(
            label=block_data.get("label", block_name),
            description=block_data.get("description", ""),
            shared=block_data.get("shared", False),
            fields=fields,
        )
    return blocks
```

### Module Loading

**Location**: `src/youlab_server/curriculum/loader.py:402-456`

Modules are loaded from `{course_dir}/modules/{module_name}.toml`:

```python
def _load_modules(self, course_dir: Path, module_names: list[str]) -> list[ModuleConfig]:
    modules = []
    modules_dir = course_dir / "modules"

    for module_name in module_names:
        module_file = modules_dir / f"{module_name}.toml"
        if not module_file.exists():
            log.warning("module_not_found", module=module_name)
            continue

        data = self._load_toml(module_file)
        module_data = data.get("module", {})

        steps = []
        for step_data in data.get("steps", []):
            steps.append(StepConfig(...))

        modules.append(ModuleConfig(
            id=module_data.get("id", module_name),
            name=module_data.get("name", module_name),
            steps=steps,
            disabled_tools=module_data.get("disabled_tools", [])
        ))

    return modules
```

---

## Dynamic Block Generation

**File**: `src/youlab_server/curriculum/blocks.py:1-190`

Pydantic models are generated **at runtime** from `BlockSchema` definitions.

### Block Creation

```python
from youlab_server.curriculum.blocks import create_block_model

schema = BlockSchema(
    label="persona",
    fields={
        "name": FieldSchema(type=FieldType.STRING, default="Bot"),
        "role": FieldSchema(type=FieldType.STRING, default="Assistant")
    }
)

PersonaBlock = create_block_model("persona", schema)
block = PersonaBlock(name="MyBot", role="Helper")
```

### Memory String Serialization

Blocks serialize to Letta's memory format:

```python
block.to_memory_string()
# Output:
# name: MyBot
# role: Helper
```

For lists:
```python
# field.items = { type = "list", default = ["a", "b"] }
block.to_memory_string()
# Output:
# items:
# - a
# - b
```

### Block Registry

```python
from youlab_server.curriculum.blocks import create_block_registry

registry = create_block_registry(course.blocks)
# registry = {
#     "persona": PersonaBlock,
#     "human": HumanBlock,
#     "journey": JourneyBlock
# }
```

**Usage**: Agent creation dynamically generates block models from course config.

---

## Background Tasks (Critical for ARI-85)

### Task Execution Flow

**Runner**: `src/youlab_server/background/runner.py:46-237`

```python
class BackgroundAgentRunner:
    def __init__(self, letta_client, honcho_client):
        self.letta = letta_client
        self.honcho = honcho_client
        self.enricher = MemoryEnricher(letta_client)

    async def run_agent(
        self,
        config: BackgroundAgentConfig,  # NOTE: Uses v1 BackgroundAgentConfig
        user_ids: list[str] | None = None,
        agent_id: str = "unknown",
    ) -> RunResult:
        # Get users to process
        target_users = user_ids or await self._get_target_users(config)

        # Process users in batches
        for batch in batches(target_users, config.batch_size):
            for user_id in batch:
                await self._process_user(config, user_id, result, agent_id)

        return result
```

### Query Execution

For each query in the task:

1. **Query Honcho Dialectic**
   ```python
   response = await self.honcho.query_dialectic(
       user_id=user_id,
       question=query.question,
       session_scope=SessionScope(query.session_scope),
       recent_limit=query.recent_limit
   )
   ```

2. **Apply Enrichment**
   ```python
   strategy = MergeStrategy(query.merge_strategy)
   enrich_result = self.enricher.enrich(
       agent_id=target_agent_id,
       block=query.target_block,
       field=query.target_field,
       content=response.insight,
       strategy=strategy,
       source=f"background:{agent_id}",
       source_query=query.question
   )
   ```

### Merge Strategies

**Location**: `src/youlab_server/memory/enricher.py` (referenced in runner)

1. **`append`**: Add to list field
2. **`replace`**: Replace entire field value
3. **`llm_diff`**: Use LLM to merge intelligently

### Current Limitation

**IMPORTANT**: The runner currently uses **v1 `BackgroundAgentConfig`** schema, not v2 `TaskConfig`.

**Evidence**: `src/youlab_server/background/runner.py:69` parameter type is `BackgroundAgentConfig`

**Implication**: To use background tasks from v2 `[[task]]` config, there may need to be a conversion layer or the runner needs updating.

**TODO for ARI-85**: Verify if runner has been updated to accept `TaskConfig` or if conversion is needed.

---

## Module Metadata (OpenWebUI Only)

**IMPORTANT**: This is **separate** from course TOML configuration. It's for OpenWebUI's visual UI only.

**File**: `docs/module-metadata-schema.md:1-263`

### Purpose

Enables course-based navigation in OpenWebUI sidebar via `youlab_module` metadata field on models.

### Schema

```json
{
  "youlab_module": {
    "course_id": "college-essay",
    "module_index": 0,
    "status": "available",  // locked | available | in_progress | completed
    "welcome_message": "Hi! I'm excited...",
    "unlock_criteria": {
      "previous_module": "college-essay-intro",
      "min_interactions": 5
    }
  }
}
```

### Current Limitations

- **Static status**: Not per-user (Phase A limitation)
- **No automatic unlocking**: `unlock_criteria` not enforced
- **No progression tracking**: Will be added in Phase B via `journey` memory block

### Relationship to TOML Config

- **TOML config**: Defines agent behavior, memory, tools, background tasks
- **Module metadata**: Defines OpenWebUI UI presentation only

**They are independent systems** - creating a course requires both:
1. TOML config for agent runtime
2. Module metadata for UI (if using OpenWebUI)

---

## Creating a New Course - Step by Step

### For ARI-85: Single-Module Course with Background Agent

**1. Create Course Directory**
```bash
mkdir -p config/courses/my-course/modules
```

**2. Create `course.toml` (V2 Schema)**

```toml
# =============================================================================
# AGENT CONFIGURATION
# =============================================================================
[agent]
id = "my-course"
name = "My Course Name"
version = "1.0.0"
description = "Course description"
modules = ["01-module-one"]
model = "openai/gpt-4o"
embedding = "openai/text-embedding-3-small"
context_window = 128000
max_response_tokens = 4096
system = """You are a tutor for my course.

Your approach:
- Be helpful and encouraging
- Guide students through exercises
"""
tools = ["send_message", "edit_memory_block"]

# =============================================================================
# MEMORY BLOCKS
# =============================================================================
[block.student]
label = "human"
shared = false
description = "Student information"
field.name = { type = "string", default = "", description = "Student's name" }
field.notes = { type = "string", default = "", description = "Learning notes" }

[block.tutor]
label = "persona"
shared = false
description = "Tutor persona"
field.approach = { type = "string", default = "I am a helpful tutor", description = "Teaching approach" }

# =============================================================================
# BACKGROUND TASKS
# =============================================================================
[[task]]
name = "insight-gatherer"
on_idle = true
idle_threshold_minutes = 10
idle_cooldown_minutes = 30
manual = true
agent_types = ["my-course"]
user_filter = "all"
batch_size = 50
system = """You are an insight gatherer analyzing student conversations."""
queries = [
    {
        target = "student.notes",
        question = "What key insights emerged from this conversation?",
        scope = "recent",
        recent_limit = 3,
        merge = "append"
    }
]

# =============================================================================
# UI MESSAGES
# =============================================================================
[messages]
welcome_first = "Welcome to my course!"
welcome_returning = "Welcome back!"
error_unavailable = "I'm having a moment - please try again."
```

**3. Create Module File: `modules/01-module-one.toml`**

```toml
[module]
id = "01-module-one"
name = "Module One"
order = 1
description = "Introduction to the course"

[[steps]]
id = "welcome"
name = "Welcome"
order = 1
description = "Welcome and introduction"
objectives = [
    "Learn student's name",
    "Set expectations"
]

[steps.completion]
min_turns = 2

[steps.agent]
opening = "Hello! Welcome to the course. What's your name?"
focus = ["introduction", "rapport"]
guidance = [
    "Be warm and welcoming",
    "Learn the student's name",
    "Explain what to expect"
]
```

**4. Verify Configuration**

```bash
# Start Python REPL
uv run python

from youlab_server.curriculum.loader import CurriculumLoader

loader = CurriculumLoader()
course = loader.load_course("my-course")

print(f"Course: {course.name}")
print(f"Blocks: {list(course.blocks.keys())}")
print(f"Modules: {len(course.loaded_modules)}")
print(f"Tasks: {len(course.tasks)}")
```

**5. Test via HTTP API** (if server running)

```bash
curl http://localhost:8000/curriculum/courses/my-course
```

---

## Verification

### Code Analysis Verification

✅ **V2 Schema Fully Implemented**
- All Pydantic models in `schema.py` support v2 format
- Loader handles both v1 and v2 with auto-detection
- `college-essay/course.toml` uses v2 schema (confirmed via file read)

✅ **No MD-Based Configuration**
- No `.md` files in `config/` directory
- `module-metadata-schema.md` is **documentation only**, not config
- All course config is TOML-based

✅ **Background Tasks Schema**
- `TaskConfig` model exists and is complete (`schema.py:169-190`)
- `[[task]]` array parsing implemented (`loader.py:369-400`)
- `QueryConfig` with `target` in `block.field` format (`schema.py:148-167`)

⚠️ **Background Runner Compatibility**
- Runner uses v1 `BackgroundAgentConfig` type (`runner.py:69`)
- May need conversion layer or runner update for v2 `TaskConfig`
- **Action**: Verify runner compatibility before ARI-85 implementation

✅ **Dynamic Block Generation**
- `create_block_model()` generates Pydantic models at runtime
- Memory serialization to Letta format implemented
- Used by agent creation

### Documentation Verification

✅ **`docs/config-schema.md` Accuracy**
- Matches current implementation
- Clearly documents v1 (legacy) vs v2 (current)
- Examples match actual config files

✅ **Module Metadata Separate**
- `docs/module-metadata-schema.md` clearly scoped to OpenWebUI UI
- Not confused with TOML course config

### Migration Status

✅ **All Phases from Plan Complete**
- Phase 1: Tool defaults ✅
- Phase 2: Flatten agent config ✅
- Phase 3: Flatten block schema ✅
- Phase 4: Shared blocks ✅
- Phase 5: New task schema ✅
- Phase 6: Config migration ✅

**Evidence**: Both `college-essay` and `default` courses use v2 schema

---

## Key Discoveries for ARI-85

### 1. No MD Migration Needed
The plan title mentions "TOML + MD Migration" but **there is no MD-based configuration**. All config is TOML.

### 2. Background Task System Ready
The `[[task]]` schema is implemented and production-ready. You can:
- Use `on_idle = true` for idle-based triggering
- Specify `queries` for simple dialectic-based enrichment
- Use `system` + `tools` for full agent capabilities (if runner supports it)

### 3. Runner Update Status Unknown
The background runner may still use v1 schema internally. Check if conversion layer exists or if runner needs updating.

### 4. Course Creation is Straightforward
To create a POC course with background agent:
1. Copy v2 template (use `college-essay/course.toml` as reference)
2. Define memory blocks with `[block.name]` and `field.*` syntax
3. Add `[[task]]` entry with `queries` for background agent
4. Create single module file with steps
5. Load via `CurriculumLoader` to validate

### 5. Shared Blocks Available
If your POC needs cross-agent memory (e.g., shared course materials), use `shared = true` on block definition.

---

## Recommended Next Steps for ARI-85

1. **Verify background runner compatibility**
   - Check if `BackgroundAgentRunner.run_agent()` has been updated to accept `TaskConfig`
   - If not, determine conversion approach

2. **Choose memory block structure**
   - What blocks does your POC course need?
   - Will any blocks be shared across agents?

3. **Define background task queries**
   - What insights should the background agent gather?
   - What merge strategy (append/replace/llm_diff)?
   - What trigger mechanism (idle/schedule/manual)?

4. **Create minimal module structure**
   - Single module with 1-2 steps for POC
   - Simple completion criteria

5. **Test end-to-end**
   - Load course via loader
   - Create agent with curriculum
   - Trigger background task manually
   - Verify memory enrichment

---

## References

**Primary Source Files**:
- `src/youlab_server/curriculum/schema.py` - Schema definitions
- `src/youlab_server/curriculum/loader.py` - Config loading
- `src/youlab_server/curriculum/blocks.py` - Dynamic block generation
- `src/youlab_server/background/runner.py` - Background task execution
- `docs/config-schema.md` - Official schema documentation
- `thoughts/shared/plans/2025-01-09-config-schema-v2.md` - Migration plan

**Example Configs**:
- `config/courses/college-essay/course.toml` - Production v2 course
- `config/courses/college-essay/modules/01-first-impression.toml` - Production module

**Related Documentation**:
- `docs/module-metadata-schema.md` - OpenWebUI module metadata (UI only)
