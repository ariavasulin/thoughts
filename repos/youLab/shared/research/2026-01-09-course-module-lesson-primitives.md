---
date: 2026-01-09T17:52:14+07:00
researcher: ariasulin
git_commit: 566e1e278835bf7fa5caee1170995fd59b756914
branch: main
repository: YouLab
topic: "Current Course/Module/Lesson Primitives Architecture"
tags: [research, codebase, curriculum, primitives, schema]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: Current Course/Module/Lesson Primitives Architecture

**Date**: 2026-01-09T17:52:14+07:00
**Researcher**: ariasulin
**Git Commit**: 566e1e278835bf7fa5caee1170995fd59b756914
**Branch**: main
**Repository**: YouLab

## Research Question

What are the current "course" and "module" primitives in the codebase? How are they structured and used? This research documents the existing system to support understanding before any changes to primitives.

## Summary

The YouLab codebase uses a **3-tier TOML-based curriculum system**:

```
Course → Module → Lesson
```

- **Course** (`CourseConfig`): Top-level container with agent configuration, memory block schemas, background agents, and UI messages. Contains a list of module references.
- **Module** (`ModuleConfig`): Organizational container for lessons within a curriculum phase. Contains ordered lessons and optional background agent overrides.
- **Lesson** (`LessonConfig`): Individual instructional unit with objectives, completion criteria, and per-lesson agent customization.

The agent configuration lives at the **Course level**, not as an independent entity. The current design is course-centric—the course defines the agent, and modules/lessons structure the curriculum progression.

## Detailed Findings

### 1. Schema Definitions

All three primitives are Pydantic models defined in `src/letta_starter/curriculum/schema.py`.

#### CourseConfig (`schema.py:328-379`)

The top-level configuration. Each course.toml file is self-contained.

```python
class CourseConfig(BaseModel):
    # Course metadata
    id: str                                    # Unique identifier (e.g., "college-essay")
    name: str                                  # Display name
    version: str = "1.0.0"                     # Semantic version
    description: str = ""                      # Course description
    modules: list[str] = []                    # Module file references (without .toml)

    # Agent configuration (embedded in course)
    agent: AgentConfig                         # Model, system prompt, tools

    # Memory block schemas
    blocks: dict[str, BlockSchema] = {}        # Dynamic memory block definitions

    # Background agents
    background: dict[str, BackgroundAgentConfig] = {}

    # UI messages
    messages: MessagesConfig                   # welcome_first, welcome_returning, etc.

    # Runtime only (not serialized)
    loaded_modules: list[ModuleConfig] = []    # Populated after loading
```

#### ModuleConfig (`schema.py:279-301`)

Container for related lessons. Loaded from `modules/*.toml` files.

```python
class ModuleConfig(BaseModel):
    id: str                                    # Module identifier (e.g., "01-self-discovery")
    name: str                                  # Display name
    order: int = 0                             # Sort order among modules
    description: str = ""                      # Module description
    lessons: list[LessonConfig] = []           # Ordered list of lessons
    background: dict[str, dict[str, Any]] = {} # Module-level background overrides
```

#### LessonConfig (`schema.py:252-277`)

Individual instructional unit with completion criteria and agent customization.

```python
class LessonConfig(BaseModel):
    id: str                                    # Lesson identifier (e.g., "welcome")
    name: str                                  # Display name
    order: int = 0                             # Sort order within module
    description: str = ""                      # Lesson description
    objectives: list[str] = []                 # Learning objectives
    completion: LessonCompletion               # When is lesson complete?
    agent: LessonAgentConfig                   # Per-lesson agent customization
```

**LessonCompletion** (`schema.py:215-230`):
```python
class LessonCompletion(BaseModel):
    required_fields: list[str] = []            # Fields that must be non-empty
    min_turns: int | None = None               # Minimum conversation turns
    min_list_length: dict[str, int] = {}       # Minimum items in list fields
    auto_advance: bool = False                 # Auto-advance when complete
```

**LessonAgentConfig** (`schema.py:232-250`):
```python
class LessonAgentConfig(BaseModel):
    opening: str | None = None                 # Opening message for lesson
    focus: list[str] = []                      # Topics to emphasize
    guidance: list[str] = []                   # Instructions for agent behavior
    persona_overrides: dict[str, Any] = {}     # Temporary persona field changes
```

### 2. File Structure

```
config/courses/
└── {course-id}/                    # e.g., "college-essay"
    ├── course.toml                 # Main configuration
    └── modules/
        ├── 01-self-discovery.toml
        ├── 02-topic-development.toml
        └── 03-drafting.toml
```

**Example course.toml structure:**
```toml
[course]
id = "college-essay"
name = "College Essay Coaching"
version = "1.0.0"
modules = ["01-self-discovery", "02-topic-development", "03-drafting"]

[agent]
model = "anthropic/claude-sonnet-4-20250514"
system = "You are an essay coach..."

[[agent.tools]]
id = "send_message"
rules = { type = "exit_loop" }

[blocks.persona]
label = "persona"
[blocks.persona.fields]
name = { type = "string", default = "Essay Coach" }

[blocks.human]
label = "human"
[blocks.human.fields]
name = { type = "string", default = "" }

[background.insight-harvester]
enabled = true
# ...queries...

[messages]
welcome_first = "Welcome!"
```

**Example module.toml structure:**
```toml
[module]
id = "01-self-discovery"
name = "Self-Discovery"
order = 1

[[lessons]]
id = "welcome"
name = "Welcome & Onboarding"
order = 1
objectives = ["Learn student's name", "Set expectations"]

[lessons.completion]
required_fields = ["human.name"]
min_turns = 3

[lessons.agent]
opening = "Welcome! What's your name?"
focus = ["name", "goals"]
```

### 3. Data Flow

```
                    TOML Files
                        │
                        ▼
            ┌───────────────────────┐
            │   CurriculumLoader    │   loader.py:32-225
            │   - load_course()     │
            │   - _load_module()    │
            └───────────────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  CurriculumRegistry   │   loader.py:228-306
            │  - _courses dict      │
            │  - _block_registries  │
            │  (global singleton)   │
            └───────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
    AgentManager   HTTP Endpoints  Background Runner
    agents.py      curriculum.py   runner.py
```

#### Loading Flow (`loader.py:43-107`)

1. `CurriculumLoader.load_course(course_id)` reads `{course_id}/course.toml`
2. Parses TOML into `CourseConfig` instance
3. Creates `DynamicBlockRegistry` with Pydantic models from block schemas
4. Loads each module from `modules/{module_ref}.toml`
5. Appends `ModuleConfig` instances (containing `LessonConfig` lists) to `course.loaded_modules`
6. Sorts modules and lessons by `order` field

#### Agent Creation Flow (`agents.py:110-225`)

1. `AgentManager.create_agent_from_curriculum(user_id, course_id)` retrieves course
2. Gets block registry for course
3. Builds memory blocks from `course.blocks` with user overrides
4. Builds tool list and rules from `course.agent.tools`
5. Creates Letta agent with `course.agent.model`, `course.agent.system`, memory blocks, tools
6. Stores `course_id` and `course.version` in agent metadata

### 4. Where Primitives Are Used

| Location | Usage |
|----------|-------|
| `curriculum/schema.py:252-379` | Schema definitions (CourseConfig, ModuleConfig, LessonConfig) |
| `curriculum/loader.py:43-225` | Loading TOML into Python objects |
| `curriculum/loader.py:228-306` | CurriculumRegistry singleton |
| `curriculum/__init__.py:5-28` | Public exports |
| `server/main.py:82-84` | Initialize curriculum on startup |
| `server/agents.py:110-225` | Create agents from course config |
| `server/curriculum.py:75-173` | HTTP endpoints for curriculum management |
| `server/background.py:96-201` | Background agent endpoints access `course.background` |
| `config/courses/college-essay/course.toml` | College essay course configuration |
| `config/courses/college-essay/modules/*.toml` | Module and lesson definitions |
| `config/courses/default/course.toml` | Default course configuration |
| `docs/config-schema.md:1-326` | Schema documentation |

### 5. HTTP Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /curriculum/courses` | List all course IDs |
| `GET /curriculum/courses/{id}` | Get course summary with module list |
| `GET /curriculum/courses/{id}/full` | Get complete config as JSON |
| `GET /curriculum/courses/{id}/modules` | Get all modules with lessons |
| `POST /curriculum/reload` | Hot-reload all configurations |

### 6. Key Relationships

```
CourseConfig
├── id, name, version, description
├── modules: list[str]  ─────────────────────┐
├── agent: AgentConfig                       │
│   ├── model, embedding, context_window     │
│   ├── system (prompt)                      │
│   └── tools: list[ToolConfig]              │
├── blocks: dict[str, BlockSchema]           │
│   ├── persona → fields                     │
│   └── human → fields                       │
├── background: dict[str, BackgroundAgentConfig]
├── messages: MessagesConfig                 │
└── loaded_modules: list[ModuleConfig] ◄─────┘
                         │
                         ├── id, name, order, description
                         ├── lessons: list[LessonConfig]
                         │              │
                         │              ├── id, name, order
                         │              ├── objectives
                         │              ├── completion: LessonCompletion
                         │              │   ├── required_fields
                         │              │   ├── min_turns
                         │              │   └── min_list_length
                         │              └── agent: LessonAgentConfig
                         │                  ├── opening
                         │                  ├── focus
                         │                  ├── guidance
                         │                  └── persona_overrides
                         │
                         └── background: dict (module-level overrides)
```

### 7. Current Behavior Observations

1. **Agent is embedded in Course**: The `AgentConfig` lives inside `CourseConfig`, not as a separate entity. The course defines the agent.

2. **Modules are containers**: Modules group lessons and provide ordering. They don't have their own agent configuration—only background overrides.

3. **Lessons have agent customization**: Each lesson can override agent behavior (opening, focus, guidance, persona_overrides) but not the core agent config (model, tools, etc.).

4. **No progression tracking**: The completion criteria schema exists (`LessonCompletion`) but no implementation tracks current lesson or progress state. The schema is defined but not actively used.

5. **Course-centric design**: The system is organized around courses. You create an agent "from a curriculum" (course_id), not the other way around.

6. **Self-contained TOML**: Each course is self-contained—no inheritance, no external references. This is an explicit design principle.

## Code References

- `src/letta_starter/curriculum/schema.py:252-277` - LessonConfig definition
- `src/letta_starter/curriculum/schema.py:279-301` - ModuleConfig definition
- `src/letta_starter/curriculum/schema.py:328-379` - CourseConfig definition
- `src/letta_starter/curriculum/loader.py:43-107` - Course loading logic
- `src/letta_starter/curriculum/loader.py:204-225` - Module loading logic
- `src/letta_starter/curriculum/loader.py:228-306` - CurriculumRegistry
- `src/letta_starter/server/agents.py:110-225` - Agent creation from curriculum
- `src/letta_starter/server/curriculum.py:75-173` - HTTP endpoints
- `config/courses/college-essay/course.toml:1-129` - Example course config
- `config/courses/college-essay/modules/01-self-discovery.toml:1-111` - Example module

## Architecture Documentation

### Design Principles (from `docs/config-schema.md`)

- **Self-contained**: Each course.toml is complete without external references
- **Explicit**: All configuration is visible in the file
- **AI-friendly**: Easy to read and modify programmatically
- **UI-ready**: Schema is introspectable for visual editors
- **Two layers only**: course.toml + modules/*.toml

### Why This Structure?

The 3-tier hierarchy (Course → Module → Lesson) maps to educational domain concepts:
- **Course**: The complete learning experience (e.g., "College Essay Coaching")
- **Module**: A phase of the curriculum (e.g., "Self-Discovery", "Drafting")
- **Lesson**: A single instructional interaction (e.g., "Welcome", "Values Exploration")

The agent configuration is embedded in the course because each course type has different tutoring needs (different tools, personas, memory schemas).

## Historical Context (from thoughts/)

### Related Plans

- `thoughts/shared/plans/2026-01-08-unified-toml-curriculum-schema.md` - The implementation plan for this TOML-based system
- `thoughts/shared/plans/2026-01-08-phase-5-toml-curriculum.md` - Phase 5 implementation details
- `thoughts/shared/plans/2026-01-09-unify-toml-only-agent-creation.md` - Unifying to TOML-only creation
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Master technical plan

### Evolution

The system evolved from hardcoded Python (`agents/templates.py`, `memory/blocks.py`) to TOML-based configuration. The deprecated modules emit warnings but remain for backwards compatibility.

## Open Questions

1. **Lesson progression**: How will the system track which lesson a user is on? The completion criteria schema exists but isn't actively used.

2. **Agent-first vs Course-first**: The current design is course-centric. Changing to "agent and steps" would invert this relationship—making the agent the primary entity.

3. **Module utility**: Modules provide grouping but no distinct agent behavior. If flattening to "agent and steps", modules could be eliminated or become tags/categories.

4. **Lesson agent customization scope**: Currently, lessons can only override specific agent behaviors (opening, focus). A "steps" model might need different customization options.
