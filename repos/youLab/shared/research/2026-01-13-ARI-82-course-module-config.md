---
date: 2026-01-13T16:55:56+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "Course and Module Configuration Structure"
tags: [research, codebase, curriculum, configuration, memory-blocks, enrollment, hot-reload]
status: complete
last_updated: 2026-01-13
last_updated_by: ariasulin
linear_ticket: ARI-82
---

# Research: Course and Module Configuration Structure

**Date**: 2026-01-13T16:55:56+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Research course and module configuration structure. Focus on:
1. config/courses/ directory structure
2. Course TOML schema (course.toml, modules/*.toml)
3. How modules specify memory blocks and notes as working memory
4. Enrollment flow and hot reload mechanism
5. src/youlab_server/curriculum/ - loader, dynamic blocks

Goal: Understand course config and how it specifies memory blocks/notes.
Gap analysis: New spec has user repo with courses/ symlinks and automatic enrollment with hot reload.

## Summary

YouLab uses a TOML-based configuration system for courses, organized in `config/courses/{course-id}/` directories. Each course has a `course.toml` that defines agent settings, memory block schemas, background tasks, and UI messages. Module files in `modules/*.toml` define curriculum steps with completion criteria that reference memory block fields. The system supports two schema versions (v1 legacy, v2 current) with automatic detection.

Key architectural points:
- **Memory blocks** are defined as schemas in course.toml, instantiated dynamically via Pydantic
- **Modules** reference memory blocks through `block.field` format in completion criteria and background task queries
- **Enrollment** is agent-per-user, triggered on first chat via OpenWebUI Pipe
- **Hot reload** clears caches and rescans `config/courses/` without restarting

## Detailed Findings

### 1. Directory Structure

```
config/courses/
  {course-id}/
    course.toml           # Main course configuration
    modules/
      01-module-name.toml # Module with steps
      02-module-name.toml
```

Current courses in the project:
- `college-essay/` - Production course with 4 module files
- `default/` - Generic tutor baseline with no modules

**File**: `config/courses/college-essay/course.toml`
**File**: `config/courses/default/course.toml`

### 2. Course TOML Schema (v2 - Current)

The v2 schema merges course metadata into the `[agent]` section:

```toml
[agent]
id = "college-essay"
name = "College Essay Coaching"
version = "2.0.0"
modules = ["01-first-impression", "02-topic-development", "03-drafting"]
model = "openai/gpt-4o"
system = """..."""
tools = ["send_message", "edit_memory_block", "advance_lesson", "query_honcho"]

[agent.folders]
shared = ["college-essay-materials"]
private = true
```

**Key sections**:
- `[agent]` - Course metadata + agent configuration
- `[block.{name}]` - Memory block schemas with `field.*` dotted keys
- `[[task]]` - Background tasks array
- `[messages]` - UI messages (welcome_first, welcome_returning, error_unavailable)

**Schema detection** in `src/youlab_server/curriculum/loader.py:142-146`:
- v2: Has `[agent]` section with `id` field
- v1: Has separate `[course]` section

### 3. Memory Block Schema

Memory blocks define typed fields with defaults and constraints:

```toml
[block.student]
label = "human"              # Maps to Letta's "human" block
shared = false               # Per-user block
description = "..."

field.profile = { type = "string", default = "", description = "..." }
field.insights = { type = "string", default = "", description = "..." }

[block.journey]
label = "journey"
shared = false
field.module_id = { type = "string", default = "01-first-impression" }
field.lesson_id = { type = "string", default = "strengths-upload" }
field.status = { type = "string", default = "in_progress", options = ["not_started", "in_progress", "completed"] }
field.milestones = { type = "list", default = [], max = 30 }
```

**Field types** (`src/youlab_server/curriculum/schema.py:45-54`):
- `string`, `int`, `float`, `bool`, `list`, `datetime`
- Optional `options` for enum constraints
- Optional `max` for list length limits

**Shared blocks**: When `shared = true`, block is created once and reused across all agents for that course. See `AgentManager._get_or_create_shared_block()` at `src/youlab_server/server/agents.py:53-112`.

### 4. Module File Structure

```toml
[module]
id = "01-first-impression"
name = "First Impression"
order = 1
disabled_tools = ["query_honcho"]

[[steps]]
id = "strengths-upload"
name = "CliftonStrengths Upload"
order = 1
objectives = ["Create warm first impression", "Analyze CliftonStrengths PDF"]

[steps.completion]
required_fields = ["human.name"]   # References block.field
min_turns = 3
min_list_length = { "human.facts" = 2 }
auto_advance = false

[steps.agent]
opening = "Welcome to YouLab!..."
focus = ["rapport", "clifton_strengths"]
guidance = ["Update student.profile with insights"]
disabled_tools = ["advance_lesson"]

[steps.agent.persona_overrides]
tone = "encouraging"
```

**Key features**:
- `required_fields` uses `block.field` format for completion criteria
- `min_list_length` specifies minimum items in list fields
- `persona_overrides` temporarily changes persona block values for a step
- Tool disabling at both module and step level

### 5. How Modules Specify Memory Blocks/Notes

**A. Completion Criteria** (`src/youlab_server/curriculum/schema.py:222-229`):
```python
class StepCompletion(BaseModel):
    required_fields: list[str] = []  # "block.field" format
    min_turns: int | None = None
    min_list_length: dict[str, int] = {}
    auto_advance: bool = False
```

**B. Step Guidance** (natural language instructions):
```toml
guidance = [
    "Update student.profile with their strengths",
    "Capture potential essay themes in student.insights",
]
```

**C. Background Task Queries** (`[[task]]` sections):
```toml
queries = [
    { target = "journey.grader_notes", question = "...", scope = "all", merge = "replace" },
    { target = "student.insights", question = "...", scope = "all", merge = "append" },
]
```

**Target parsing** (`src/youlab_server/curriculum/schema.py:157-166`):
```python
class QueryConfig(BaseModel):
    target: str  # "block.field" format

    @property
    def target_block(self) -> str:
        return self.target.split(".")[0]
```

### 6. Curriculum Loader Implementation

**Entry point**: `src/youlab_server/curriculum/__init__.py:130` - `curriculum` singleton

**CurriculumLoader** (`src/youlab_server/curriculum/loader.py:43-456`):
- `list_courses()` - Scans `config/courses/` for directories with `course.toml`
- `load_course(course_id)` - Parses TOML, detects schema version, loads modules
- `reload()` - Clears cache and reloads all courses

**Dynamic block generation** (`src/youlab_server/curriculum/blocks.py:119-168`):
- `create_block_model(block_name, schema)` - Generates Pydantic class at runtime
- `create_block_registry(blocks)` - Returns dict of block_name -> model class

**Block serialization** (`src/youlab_server/curriculum/blocks.py:51-70`):
```python
def to_memory_string(self) -> str:
    # Serializes to YAML-like format for Letta memory
    # Lists: "field:\n- item1\n- item2"
    # Strings: "field: value"
```

### 7. Enrollment Flow

Enrollment is agent-per-user, triggered on first chat:

1. **User sends message** via OpenWebUI
2. **Pipe.pipe()** (`src/youlab_server/pipelines/letta_pipe.py:138`) extracts user_id
3. **_ensure_agent_exists()** (`letta_pipe.py:99-136`) checks for existing agent
4. **POST /agents** if no agent exists
5. **AgentManager.create_agent_from_curriculum()** (`src/youlab_server/server/agents.py:220-367`):
   - Loads CourseConfig from curriculum
   - Gets block registry, instantiates blocks with defaults
   - Creates Letta agent with memory blocks, tools, system prompt
   - Attaches shared/private folders if configured
6. **Agent ID cached** as `(user_id, agent_type)` -> `agent_id`

### 8. Hot Reload Mechanism

**Endpoint**: `POST /curriculum/reload` (`src/youlab_server/server/curriculum.py:147-173`)

**Implementation**:
```python
# In Curriculum.reload() at __init__.py:105-108
def reload(self) -> int:
    self._block_registries.clear()      # Clear cached Pydantic models
    return self._ensure_loader().reload()

# In CurriculumLoader.reload() at loader.py:123-133
def reload(self) -> int:
    self._cache.clear()                  # Clear cached CourseConfigs
    for course_id in self.list_courses():
        self.load_course(course_id)      # Reload from TOML
    return count
```

**What gets reloaded**:
- Course metadata, agent config, system prompts
- Memory block schemas
- Module definitions and step configurations
- Background task definitions
- Tool configurations

**What doesn't get reloaded**:
- Existing agent instances in Letta (keep old config until deleted)
- AgentManager cache (agent_id mappings persist)
- User storage / Git repositories

### 9. Startup Discovery

**In lifespan()** (`src/youlab_server/server/main.py:96-99`):
```python
curriculum.initialize(Path("config/courses"))
courses_loaded = curriculum.load_all()
```

**Discovery criteria** (`loader.py:61-69`):
- Scans immediate subdirectories of `config/courses/`
- Directory must contain `course.toml`
- Returns sorted list of directory names as course IDs

## Code References

| File | Lines | Purpose |
|------|-------|---------|
| `curriculum/schema.py` | 45-54 | FieldType enum |
| `curriculum/schema.py` | 197-214 | BlockSchema model |
| `curriculum/schema.py` | 222-262 | Step completion, agent, module config |
| `curriculum/schema.py` | 282-310 | AgentConfig model |
| `curriculum/loader.py` | 43-456 | CurriculumLoader class |
| `curriculum/loader.py` | 140-190 | Schema version detection |
| `curriculum/loader.py` | 294-323 | v2 block parsing |
| `curriculum/loader.py` | 402-455 | Module loading |
| `curriculum/blocks.py` | 119-168 | Dynamic model generation |
| `curriculum/__init__.py` | 57-130 | Curriculum singleton |
| `server/agents.py` | 220-367 | Agent creation from curriculum |
| `server/main.py` | 96-99 | Startup initialization |
| `server/curriculum.py` | 147-173 | Hot reload endpoint |
| `pipelines/letta_pipe.py` | 99-136 | Enrollment via Pipe |
| `tools/memory.py` | 27-108 | edit_memory_block tool |
| `tools/curriculum.py` | 264-302 | advance_lesson tool |

## Gap Analysis for ARI-82

The new spec requires:
1. **User repo with courses/ symlinks** - Currently courses are in `config/courses/` (shared), not per-user
2. **Automatic enrollment with hot reload** - Current enrollment is on-demand (first chat), not automatic

**Current state**:
- Course discovery: Scans `config/courses/` at startup and reload
- Enrollment: Agent-per-user, created on first message
- Hot reload: Clears caches, rescans directory, affects new agents only

**Gap**:
- No per-user courses/ symlink mechanism
- No automatic enrollment (watch for new courses and create agents)
- Existing agents don't pick up config changes

## Open Questions

1. How should per-user course symlinks interact with shared course configs?
2. Should hot reload update existing agents or just new ones?
3. What triggers automatic enrollment - webhook, filesystem watch, or scheduled scan?
4. How to handle course removal - delete associated agents?
