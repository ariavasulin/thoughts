# Config Schema V2 Implementation Plan

## Overview

Migrate the course configuration schema from the current verbose structure to a cleaner, flatter design that supports shared blocks and sophisticated background tasks.

## Current State Analysis

**Current schema location**: `src/letta_starter/curriculum/schema.py`
**Current configs**: `config/courses/college-essay/`, `config/courses/default/`

### Current Structure
```toml
[course]
id = "college-essay"
name = "College Essay Coaching"
modules = ["01-self-discovery"]

[agent]
model = "anthropic/claude-sonnet-4-20250514"
system = "..."

[[agent.tools]]
id = "send_message"
rules = { type = "exit_loop" }

[blocks.persona]
label = "persona"
[blocks.persona.fields]
name = { type = "string", default = "..." }

[background.insight-harvester]
enabled = true
[background.insight-harvester.triggers]
schedule = "0 3 * * *"
[[background.insight-harvester.queries]]
id = "learning_style"
question = "..."
target_block = "human"
target_field = "context_notes"
```

### Pain Points
- Verbose nested structure (`[blocks.persona.fields]`)
- Separate `[course]` and `[agent]` sections (redundant)
- Tool rules require full config per tool
- Background agent naming is arbitrary
- No support for shared blocks across agents
- Background tasks limited to simple queries

## Desired End State

### New Schema Structure
```toml
# =============================================================================
# AGENT CONFIGURATION (combines course + agent)
# =============================================================================
[agent]
id = "college-essay"
name = "College Essay Coaching"
version = "1.0.0"
description = "AI-powered tutoring for college application essays"
model = "anthropic/claude-sonnet-4-20250514"
embedding = "openai/text-embedding-3-small"
context_window = 128000
max_response_tokens = 4096
system = """
You are YouLab Essay Coach...
"""
modules = ["01-self-discovery", "02-topic-development"]
tools = ["send_message", "query_honcho", "edit_memory_block"]

# =============================================================================
# MEMORY BLOCKS (flattened with dotted keys)
# =============================================================================
[block.persona]
label = "persona"
shared = false  # default
description = "Agent identity and behavior"
field.name = { type = "string", default = "YouLab Essay Coach" }
field.role = { type = "string", default = "AI tutor..." }
field.tone = { type = "string", default = "warm", options = ["warm", "professional"] }
field.capabilities = { type = "list", default = [], max = 10 }

[block.human]
label = "human"
shared = false
description = "User context"
field.name = { type = "string", default = "" }
field.bio = { type = "string", default = "" }
field.top_of_mind = { type = "string", default = "" }
field.top_3_priorities = { type = "string", default = "" }

[block.team]
label = "team"
shared = true  # Attached to all agents using this course
description = "Shared team context"
field.decisions = { type = "string", default = "" }
field.open_questions = { type = "string", default = "" }

# =============================================================================
# BACKGROUND TASKS (array of batches)
# =============================================================================

# Simple query-based task
[[task]]
schedule = "0 3 * * *"
batch_size = 50
queries = [
    { target = "human.context_notes", question = "What learning style?", merge = "append" },
    { target = "human.facts", question = "How engaged?", merge = "append" }
]

# Complex agent task
[[task]]
schedule = "0 4 * * 0"
system = """
You are a document organizer...
"""
tools = ["manage_files", "edit_memory_block"]

# =============================================================================
# UI MESSAGES
# =============================================================================
[messages]
welcome_first = "Welcome to YouLab!"
welcome_returning = "Welcome back!"
error_unavailable = "I'm having a moment..."
```

### Key Changes Summary

| Aspect | Current | New |
|--------|---------|-----|
| Course metadata | `[course]` + `[agent]` | `[agent]` combined |
| Tools | `[[agent.tools]]` with rules | `tools = ["name", "name:rule"]` |
| Tool defaults | In config | In tool files |
| Block fields | `[blocks.x.fields]` section | `field.x = {...}` dotted |
| Shared blocks | Not supported | `shared = true` flag |
| Background tasks | Named `[background.name]` | Anonymous `[[task]]` array |
| Task capabilities | Queries only | Queries + full agent |

## What We're NOT Doing

- Migrating module files (they stay as-is for now)
- Changing the loader caching mechanism
- Modifying the HTTP endpoints
- Changing how agents are created in Letta (that's a follow-up)

## Implementation Approach

Implement schema changes incrementally with backwards compatibility during transition. Each phase is independently deployable.

---

## Phase 1: Tool Default Rules

### Overview
Move tool rule defaults into tool files so config only needs overrides.

### Changes Required

#### 1. Tool Metadata Schema
**File**: `src/letta_starter/tools/__init__.py` (new)

```python
"""Tool registry with default rules."""

from enum import Enum
from typing import NamedTuple


class ToolRule(str, Enum):
    EXIT_LOOP = "exit"
    CONTINUE_LOOP = "continue"
    RUN_FIRST = "first"


class ToolMetadata(NamedTuple):
    id: str
    default_rule: ToolRule
    description: str = ""


# Registry of known tools with their defaults
TOOL_REGISTRY: dict[str, ToolMetadata] = {
    "send_message": ToolMetadata("send_message", ToolRule.EXIT_LOOP),
    "query_honcho": ToolMetadata("query_honcho", ToolRule.CONTINUE_LOOP),
    "edit_memory_block": ToolMetadata("edit_memory_block", ToolRule.CONTINUE_LOOP),
}


def get_tool_rule(tool_spec: str) -> tuple[str, ToolRule]:
    """Parse tool spec like 'send_message' or 'custom:exit' into (id, rule)."""
    if ":" in tool_spec:
        tool_id, rule_str = tool_spec.split(":", 1)
        rule = ToolRule(rule_str)
    else:
        tool_id = tool_spec
        metadata = TOOL_REGISTRY.get(tool_id)
        rule = metadata.default_rule if metadata else ToolRule.CONTINUE_LOOP
    return tool_id, rule
```

#### 2. Update Schema
**File**: `src/letta_starter/curriculum/schema.py`

Remove `ToolConfig` and `ToolRules` classes. Update `AgentConfig`:

```python
class AgentConfig(BaseModel):
    """Agent configuration."""

    model: str = "anthropic/claude-sonnet-4-20250514"
    embedding: str = "openai/text-embedding-3-small"
    context_window: int = 128000
    max_response_tokens: int = 4096
    system: str = ""
    # Changed: simple list with optional :rule suffix
    tools: list[str] = Field(default_factory=list)
```

### Success Criteria

#### Automated Verification
- [ ] Lint passes: `make lint-fix`
- [ ] Typecheck passes: `make check-agent`
- [ ] Tests pass: `make test-agent`
- [ ] Tool parsing works: `get_tool_rule("send_message") == ("send_message", EXIT_LOOP)`
- [ ] Override works: `get_tool_rule("custom:exit") == ("custom", EXIT_LOOP)`

#### Manual Verification
- [ ] Existing configs still load correctly

---

## Phase 2: Flatten Agent Config

### Overview
Merge `[course]` and `[agent]` into single `[agent]` section.

### Changes Required

#### 1. Update Schema
**File**: `src/letta_starter/curriculum/schema.py`

Merge `CourseConfig` fields into `AgentConfig`:

```python
class AgentConfig(BaseModel):
    """Combined agent and course configuration."""

    # Course metadata (moved from CourseConfig)
    id: str
    name: str
    version: str = "1.0.0"
    description: str = ""
    modules: list[str] = Field(default_factory=list)

    # Agent settings
    model: str = "anthropic/claude-sonnet-4-20250514"
    embedding: str = "openai/text-embedding-3-small"
    context_window: int = 128000
    max_response_tokens: int = 4096
    system: str = ""
    tools: list[str] = Field(default_factory=list)


class CourseConfig(BaseModel):
    """Complete course configuration."""

    # Now just wraps AgentConfig
    agent: AgentConfig
    blocks: dict[str, BlockSchema] = Field(default_factory=dict)
    background: dict[str, BackgroundAgentConfig] = Field(default_factory=dict)  # Deprecated
    tasks: list[TaskConfig] = Field(default_factory=list)  # New
    messages: MessagesConfig = Field(default_factory=MessagesConfig)
    loaded_modules: list[ModuleConfig] = Field(default_factory=list, exclude=True)

    # Convenience accessors
    @property
    def id(self) -> str:
        return self.agent.id

    @property
    def name(self) -> str:
        return self.agent.name
```

#### 2. Update Loader
**File**: `src/letta_starter/curriculum/loader.py`

Support both old `[course]` + `[agent]` and new `[agent]` only:

```python
def _parse_course_config(self, data: dict) -> CourseConfig:
    """Parse TOML data, supporting both v1 and v2 schemas."""

    # V2: [agent] contains everything
    if "agent" in data and "id" in data["agent"]:
        return CourseConfig(**data)

    # V1: [course] + [agent] separate (backwards compat)
    if "course" in data:
        # Merge course into agent
        agent_data = data.get("agent", {})
        agent_data.update(data["course"])
        data["agent"] = agent_data
        del data["course"]
        return CourseConfig(**data)

    raise ValueError("Config must have [agent] section with 'id' field")
```

#### 3. Migrate Existing Configs
**Files**: `config/courses/college-essay/course.toml`, `config/courses/default/course.toml`

Before:
```toml
[course]
id = "college-essay"
name = "College Essay Coaching"
modules = [...]

[agent]
model = "..."
system = "..."
```

After:
```toml
[agent]
id = "college-essay"
name = "College Essay Coaching"
modules = [...]
model = "..."
system = "..."
```

### Success Criteria

#### Automated Verification
- [ ] Lint passes: `make lint-fix`
- [ ] Typecheck passes: `make check-agent`
- [ ] Tests pass: `make test-agent`
- [ ] Old configs load: loader handles `[course]` + `[agent]`
- [ ] New configs load: loader handles merged `[agent]`

#### Manual Verification
- [ ] `GET /curriculum/courses` returns correct data
- [ ] Agent creation still works with curriculum

---

## Phase 3: Flatten Block Schema

### Overview
Change from `[blocks.x.fields]` section to `[block.x]` with `field.y` dotted keys.

### Changes Required

#### 1. Update Schema
**File**: `src/letta_starter/curriculum/schema.py`

```python
class BlockSchema(BaseModel):
    """Memory block schema with dotted field syntax."""

    label: str
    description: str = ""
    shared: bool = False  # NEW: for cross-agent sharing

    # Fields as dict, populated from field.* dotted keys
    fields: dict[str, FieldSchema] = Field(default_factory=dict, exclude=True)

    def model_post_init(self, __context) -> None:
        """Extract field.* keys into fields dict."""
        # This is handled in the loader, not here
        pass
```

#### 2. Update Loader
**File**: `src/letta_starter/curriculum/loader.py`

Parse dotted `field.*` keys:

```python
def _parse_blocks(self, data: dict) -> dict[str, BlockSchema]:
    """Parse block configs, supporting both v1 and v2 syntax."""
    blocks = {}

    # V2: [block.x] with field.* dotted keys
    if "block" in data:
        for block_name, block_data in data["block"].items():
            fields = {}
            # Extract field.* keys
            for key, value in list(block_data.items()):
                if key.startswith("field."):
                    field_name = key[6:]  # Remove "field." prefix
                    fields[field_name] = FieldSchema(**value)
                    del block_data[key]

            block_data["fields"] = fields
            blocks[block_name] = BlockSchema(**block_data)
        return blocks

    # V1: [blocks.x.fields] nested (backwards compat)
    if "blocks" in data:
        for block_name, block_data in data["blocks"].items():
            blocks[block_name] = BlockSchema(**block_data)
        return blocks

    return blocks
```

#### 3. Migrate Existing Configs

Before:
```toml
[blocks.persona]
label = "persona"

[blocks.persona.fields]
name = { type = "string", default = "..." }
role = { type = "string", default = "..." }
```

After:
```toml
[block.persona]
label = "persona"
shared = false
field.name = { type = "string", default = "..." }
field.role = { type = "string", default = "..." }
```

### Success Criteria

#### Automated Verification
- [ ] Lint passes: `make lint-fix`
- [ ] Typecheck passes: `make check-agent`
- [ ] Tests pass: `make test-agent`
- [ ] Old `[blocks.x.fields]` syntax loads
- [ ] New `[block.x]` with `field.*` loads
- [ ] `shared` flag is parsed correctly

#### Manual Verification
- [ ] Curriculum endpoints return correct block schemas
- [ ] Agent creation uses block schemas correctly

---

## Phase 4: Implement Shared Blocks

### Overview
Add support for `shared = true` blocks that use Letta's global blocks API.

### Changes Required

#### 1. Update AgentManager
**File**: `src/letta_starter/server/agents.py`

```python
class AgentManager:
    def __init__(self, ...):
        self._shared_blocks: dict[str, str] = {}  # course:label -> block_id

    async def _ensure_shared_block(self, course_id: str, block_schema: BlockSchema) -> str:
        """Get or create a shared block, return block_id."""
        cache_key = f"{course_id}:{block_schema.label}"

        if cache_key in self._shared_blocks:
            return self._shared_blocks[cache_key]

        # Check if block exists
        block_name = f"{course_id}-{block_schema.label}"
        existing = self.client.blocks.list(name=block_name)

        if existing:
            block_id = existing[0].id
        else:
            # Create global block
            block = self.client.blocks.create(
                label=block_schema.label,
                value=self._render_block_default(block_schema),
                name=block_name,
            )
            block_id = block.id

        self._shared_blocks[cache_key] = block_id
        return block_id

    async def create_agent(self, user_id: str, course_id: str, ...) -> AgentState:
        course = self.loader.load_course(course_id)

        # Separate shared vs agent-specific blocks
        memory_blocks = []
        shared_block_ids = []

        for block_name, block_schema in course.blocks.items():
            if block_schema.shared:
                block_id = await self._ensure_shared_block(course_id, block_schema)
                shared_block_ids.append(block_id)
            else:
                memory_blocks.append({
                    "label": block_schema.label,
                    "value": self._render_block_default(block_schema),
                })

        agent = self.client.agents.create(
            name=agent_name,
            memory_blocks=memory_blocks,
            # Attach shared blocks after creation
            ...
        )

        # Attach shared blocks
        for block_id in shared_block_ids:
            self.client.agents.core_memory.attach_block(agent.id, block_id)

        return agent
```

### Success Criteria

#### Automated Verification
- [ ] Lint passes: `make lint-fix`
- [ ] Typecheck passes: `make check-agent`
- [ ] Tests pass: `make test-agent`
- [ ] Unit test: shared block created once, reused for second agent
- [ ] Unit test: agent has shared block attached

#### Manual Verification
- [ ] Create two agents with same course
- [ ] Update shared block via one agent
- [ ] Verify other agent sees the update

**Implementation Note**: This requires a running Letta server for full testing.

---

## Phase 5: New Task Schema

### Overview
Replace `[background.name]` with `[[task]]` array supporting both queries and full agents.

### Changes Required

#### 1. Update Schema
**File**: `src/letta_starter/curriculum/schema.py`

```python
class QueryConfig(BaseModel):
    """Single dialectic query within a task."""
    target: str  # "block.field" format
    question: str
    scope: SessionScope = SessionScope.ALL
    recent_limit: int = 5
    merge: MergeStrategy = MergeStrategy.APPEND

    @property
    def target_block(self) -> str:
        return self.target.split(".")[0]

    @property
    def target_field(self) -> str:
        return self.target.split(".")[1]


class TaskConfig(BaseModel):
    """Background task configuration."""

    # Scheduling
    schedule: str | None = None  # Cron expression
    manual: bool = True
    on_idle: bool = False
    idle_threshold_minutes: int = 30
    idle_cooldown_minutes: int = 60

    # Scope
    agent_types: list[str] = Field(default_factory=lambda: ["tutor"])
    user_filter: str = "all"
    batch_size: int = 50

    # Simple queries (optional)
    queries: list[QueryConfig] = Field(default_factory=list)

    # Full agent capabilities (optional)
    system: str | None = None
    tools: list[str] = Field(default_factory=list)
```

#### 2. Update Loader
**File**: `src/letta_starter/curriculum/loader.py`

```python
def _parse_tasks(self, data: dict) -> list[TaskConfig]:
    """Parse [[task]] array."""
    tasks = []

    # V2: [[task]] array
    if "task" in data and isinstance(data["task"], list):
        for task_data in data["task"]:
            tasks.append(TaskConfig(**task_data))
        return tasks

    # V1: [background.name] dict (backwards compat)
    if "background" in data:
        for name, bg_data in data["background"].items():
            # Convert old format to new
            task = self._convert_legacy_background(bg_data)
            tasks.append(task)
        return tasks

    return tasks

def _convert_legacy_background(self, bg_data: dict) -> TaskConfig:
    """Convert [background.x] to TaskConfig."""
    triggers = bg_data.get("triggers", {})
    queries = []

    for q in bg_data.get("queries", []):
        queries.append(QueryConfig(
            target=f"{q['target_block']}.{q['target_field']}",
            question=q["question"],
            scope=q.get("session_scope", "all"),
            merge=q.get("merge_strategy", "append"),
        ))

    return TaskConfig(
        schedule=triggers.get("schedule"),
        manual=triggers.get("manual", True),
        on_idle=triggers.get("idle", {}).get("enabled", False),
        queries=queries,
    )
```

#### 3. Update Background Runner
**File**: `src/letta_starter/background/runner.py`

Add support for full agent tasks (system prompt + tools).

#### 4. Migrate Existing Configs

Before:
```toml
[background.insight-harvester]
enabled = true
[background.insight-harvester.triggers]
schedule = "0 3 * * *"
[[background.insight-harvester.queries]]
id = "learning_style"
question = "What learning style?"
target_block = "human"
target_field = "context_notes"
merge_strategy = "append"
```

After:
```toml
[[task]]
schedule = "0 3 * * *"
batch_size = 50
queries = [
    { target = "human.context_notes", question = "What learning style?", merge = "append" }
]
```

### Success Criteria

#### Automated Verification
- [ ] Lint passes: `make lint-fix`
- [ ] Typecheck passes: `make check-agent`
- [ ] Tests pass: `make test-agent`
- [ ] Old `[background.x]` format loads
- [ ] New `[[task]]` format loads
- [ ] Task with `queries` parses correctly
- [ ] Task with `system` + `tools` parses correctly

#### Manual Verification
- [ ] Background task endpoint lists tasks correctly
- [ ] Simple query task executes
- [ ] Full agent task executes (if implemented)

---

## Phase 6: Migrate Existing Configs

### Overview
Convert all existing configs to v2 schema.

### Changes Required

#### 1. college-essay/course.toml
Full migration to new schema.

#### 2. default/course.toml
Full migration to new schema.

#### 3. Update Documentation
**File**: `docs/config-schema.md`

Document new v2 schema, mark v1 as deprecated.

### Success Criteria

#### Automated Verification
- [ ] All configs use v2 schema
- [ ] `make verify-agent` passes
- [ ] No deprecation warnings from loader

#### Manual Verification
- [ ] Documentation accurately reflects new schema
- [ ] Curriculum endpoints work correctly

---

## Testing Strategy

### Unit Tests
- Tool rule parsing (with/without override)
- Schema validation for all new models
- Loader backwards compatibility (v1 → v2)
- Shared block detection

### Integration Tests
- Full config loading (new schema)
- Agent creation with shared blocks
- Background task execution

### Manual Testing Steps
1. Load college-essay course, verify all fields parsed
2. Create two agents, verify shared block attached to both
3. Update shared block, verify both agents see change
4. Trigger background task, verify queries execute

## Migration Notes

- Loader supports both v1 and v2 schemas during transition
- v1 configs emit deprecation warning
- After all configs migrated, v1 support can be removed

## References

- Current schema: `src/letta_starter/curriculum/schema.py`
- Current loader: `src/letta_starter/curriculum/loader.py`
- Letta shared blocks: https://docs.letta.com/guides/agents/multi-agent-shared-memory
- Existing configs: `config/courses/`
