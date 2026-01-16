---
date: 2026-01-16T00:00:00+00:00
researcher: claude-sonnet-4.5
git_commit: 2611ba2
branch: main
repository: YouLab
topic: "Memory Block Schema - MD-Based System Research"
tags: [research, memory-blocks, schema, ARI-85, markdown, toml]
status: complete
parent_ticket: ARI-85
---

# Memory Block Schema Research (MD-Based System)

**Date**: 2026-01-16
**Researcher**: Claude Sonnet 4.5
**Git Commit**: 2611ba2
**Branch**: main
**Repository**: YouLab
**Parent Ticket**: ARI-85 - Create proof-of-concept single-module course with background agent

## Research Question

Investigate the memory block schema system, focusing on:
1. How memory blocks are defined (TOML schema → MD files)
2. The "MD-based schema" mentioned in the task
3. How blocks are loaded into agents
4. Memory block types and patterns
5. The dual-format system (TOML schema + MD storage)

## Executive Summary

YouLab implements a **dual-format memory block system**:

1. **TOML Schema Definition** (in `config/courses/*/course.toml`) - Defines the structure and fields
2. **Markdown Storage** (in `.data/users/*/memory-blocks/*.md`) - Stores actual user data with YAML frontmatter

This is NOT a "new MD-based schema" replacing TOML - rather, it's a **two-layer architecture**:
- **Schema layer (TOML)**: Defines what fields exist and their types
- **Data layer (Markdown)**: Stores the actual content per user

The system uses dynamic Pydantic model generation at runtime to bridge these layers.

## Detailed Findings

### 1. Schema Definition (TOML → Pydantic Models)

Memory block schemas are defined in course TOML files using the v2 `[block.{name}]` syntax:

**Location**: `config/courses/{course-id}/course.toml`

**Example from college-essay/course.toml** (lines 86-108):

```toml
[block.student]
label = "human"
shared = false
description = "Rich narrative understanding of who this student is"
field.profile = { type = "string", default = "", description = "1-2 paragraph narrative" }
field.insights = { type = "string", default = "", description = "Key observations" }

[block.engagement_strategy]
label = "persona"
shared = false
description = "Adapted coaching approach for this specific student"
field.approach = { type = "string", default = "I am YouLab Essay Coach...", description = "..." }

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

**Key Attributes**:
- `label`: Letta block type (e.g., "human", "persona", "journey")
- `shared`: Whether block is shared across agents (true) or per-agent (false)
- `description`: Human-readable description
- `field.*`: Field definitions with type, default, options, max, description, required

**Supported Field Types** (`src/youlab_server/curriculum/schema.py:45-54`):
- `string` → Python `str`
- `int` → Python `int`
- `float` → Python `float`
- `bool` → Python `bool`
- `list` → Python `list[str]`
- `datetime` → Python `datetime | None`

### 2. Dynamic Model Generation

The TOML schema is converted to Pydantic models at runtime:

**Implementation**: `src/youlab_server/curriculum/blocks.py`

**Flow**:
```
TOML Block Schema
    ↓
CurriculumLoader._parse_blocks_v2() (loader.py:294-323)
    ↓
BlockSchema Pydantic models (schema.py:208-214)
    ↓
curriculum.get_block_registry(course_id) (__init__.py:110-126)
    ↓
create_block_registry(blocks) (blocks.py:171-189)
    ↓
create_block_model(block_name, schema) (blocks.py:119-168)
    ↓
Pydantic.create_model() with DynamicBlock base
    ↓
Generated Model Class (e.g., StudentBlock, JourneyBlock)
```

**Generated Model Features** (`blocks.py:48-116`):
- Inherits from `DynamicBlock` base class
- `.to_memory_string()` - Serializes to YAML-like format for Letta
- `.from_memory_string()` - Parses memory string back to model
- Type validation via Pydantic
- Default values from schema

**Example**:
```python
# Schema defines:
field.profile = { type = "string", default = "" }
field.insights = { type = "string", default = "" }

# Generated class (conceptual):
class StudentBlock(DynamicBlock):
    profile: str = Field(default="")
    insights: str = Field(default="")

    def to_memory_string(self) -> str:
        return "profile: {}\n\ninsights: {}".format(self.profile, self.insights)
```

### 3. Markdown Storage Layer

User-specific memory blocks are stored as Markdown files with YAML frontmatter.

**Storage Location**: `.data/users/{user_id}/memory-blocks/{label}.md`

**Directory Structure** (`storage/git.py:89-107`):
```
.data/users/{user_id}/
    .git/                      # Git repo for version control
    memory-blocks/
        student.md             # "student" block (label="human")
        engagement_strategy.md # "engagement_strategy" block (label="persona")
        journey.md             # "journey" block (label="journey")
    pending_diffs/
        {diff_id}.json         # Pending edits from agents
    agent_threads/
        {agent_name}/
            {chat_id}.json
```

**File Format** (YAML frontmatter + Markdown body):

**Example** (`.data/users/.../memory-blocks/student.md`):
```markdown
---
block: student
updated_at: '2026-01-13T18:27:03.150665+00:00'
title: Student
---

## About Me

I'm a high school senior preparing for college applications. I'm interested in computer science.

## Goals

*   Write a compelling personal essay
*   Highlight my passion for technology
*   Show personal growth and resilience

## Background

*   GPA: 3.9 (unweighted), 4.2 (weighted)
*   Activities: Robotics club, coding bootcamp mentor
*   Interests: AI, game development, open source
```

**Frontmatter Functions** (`storage/git.py:21-65`):
- `parse_frontmatter(content)` - Extracts YAML frontmatter and body
- `format_frontmatter(metadata, body)` - Combines metadata + body into full file

**Metadata Fields**:
- `block`: Block label (e.g., "student")
- `updated_at`: ISO timestamp of last update
- `title`: Display title (optional, auto-generated from label if missing)
- `schema`: Optional schema reference (e.g., "college-essay/student")

### 4. Git Versioning

Every block edit creates a git commit for full version history.

**Implementation**: `src/youlab_server/storage/git.py:89-461`

**GitUserStorage** responsibilities:
- Initialize per-user git repository
- Read/write blocks with frontmatter
- Create git commits for each edit
- Track version history

**API**:
```python
storage = GitUserStorage(user_id="abc123", base_dir=".data/users")
storage.init()  # Initialize git repo

# Write block
storage.write_block(
    label="student",
    content="## About Me\n...",
    message="Update student profile",
    author="user",
    schema="college-essay/student",
    title="Student"
)  # Returns commit SHA

# Read block
content = storage.read_block("student")  # Full MD with frontmatter
body = storage.read_block_body("student")  # Body only
metadata = storage.read_block_metadata("student")  # Frontmatter only

# Version history
versions = storage.get_versions("student")  # List of VersionInfo
```

### 5. Letta Synchronization

Markdown blocks are synced to Letta in a memory string format.

**Sync Flow** (`storage/blocks.py:163-211`):

```
User edits block (via API)
    ↓
UserBlockManager.update_block()
    ↓
GitUserStorage.write_block() → Creates git commit
    ↓
UserBlockManager._sync_block_to_letta()
    ↓
_toml_to_memory_string() → Converts to Letta format
    ↓
letta.blocks.update() or letta.blocks.create()
```

**Memory String Format** (`storage/blocks.py:213-222`):

TOML/Pydantic model is converted to YAML-like string:

```python
# Input (Pydantic model fields):
{"profile": "I'm a student...", "insights": "Key observation 1\nKey observation 2"}

# Output (memory string):
"""
profile: I'm a student...

insights:
- Key observation 1
- Key observation 2
"""
```

**Letta Block Naming** (`storage/blocks.py:41-43`):
- User blocks: `youlab_user_{user_id}_{label}`
- Shared blocks: `youlab_shared_{course_id}_{label}`

### 6. Agent Creation & Block Attachment

When creating an agent, blocks are instantiated from schema and attached to Letta.

**Implementation**: `src/youlab_server/server/agents.py:240-351`

**Flow**:
```
AgentManager.create_agent_from_curriculum(user_id, course_id)
    ↓
Load course config: curriculum.get(course_id)
    ↓
Get block registry: curriculum.get_block_registry(course_id)
    ↓
For each block in course.blocks:
    ↓
    Get model class: block_registry.get(block_name)
    ↓
    Instantiate with defaults: model_class(**overrides)
    ↓
    Serialize: instance.to_memory_string()
    ↓
    If shared=true:
        _get_or_create_shared_block() → block_ids
    Else:
        Add to memory_blocks list
    ↓
letta.agents.create(
    memory_blocks=[...],  # Per-agent blocks
    block_ids=[...]       # Shared blocks
)
```

**Key Distinction** (lines 289-305):
- **`memory_blocks` param**: Letta creates NEW blocks for this agent (per-agent)
- **`block_ids` param**: Letta attaches EXISTING shared blocks (cross-agent)

### 7. Memory Block Patterns

**Three Block Types** (from college-essay example):

1. **Student Block** (`label="human"`)
   - Purpose: Rich narrative understanding of the student
   - Fields: `profile`, `insights`
   - Pattern: Free-form narrative, updated as agent learns
   - Shared: No (per-student)

2. **Engagement Strategy Block** (`label="persona"`)
   - Purpose: Adapted coaching approach
   - Fields: `approach`
   - Pattern: Agent's self-model, how to work with this student
   - Shared: No (per-student)

3. **Journey Block** (`label="journey"`)
   - Purpose: Curriculum progress tracking
   - Fields: `module_id`, `lesson_id`, `status`, `grader_notes`, `blockers`, `milestones`
   - Pattern: Structured state machine, updated by tools
   - Shared: No (per-student)

**Shared Blocks** (not in current example):
- Use `shared = true` in schema
- Created once per course, attached to all agents
- Example use case: Course syllabus, operating manual, team info

### 8. Background Agent Memory Updates

Background agents can update memory blocks via dialectic queries.

**Configuration** (`course.toml:114-136`):

```toml
[[task]]
name = "progression-grader"
on_idle = true
queries = [
    { target = "journey.grader_notes", question = "...", scope = "all", merge = "replace" },
    { target = "journey.blockers", question = "...", scope = "all", merge = "replace" },
    { target = "student.insights", question = "...", scope = "all", merge = "append" },
    { target = "engagement_strategy.approach", question = "...", scope = "all", merge = "llm_diff" }
]
```

**Target Format**: `{block_name}.{field_name}`

**Merge Strategies** (`schema.py:29-34`):
- `append`: Add to existing content
- `replace`: Overwrite existing content
- `llm_diff`: LLM-generated diff/merge

## Code References

| File | Lines | Purpose |
|------|-------|---------|
| `config/courses/college-essay/course.toml` | 86-108 | Block schema definitions (v2 format) |
| `src/youlab_server/curriculum/schema.py` | 45-54 | FieldType enum |
| `src/youlab_server/curriculum/schema.py` | 197-214 | BlockSchema model |
| `src/youlab_server/curriculum/loader.py` | 294-323 | TOML v2 block parsing |
| `src/youlab_server/curriculum/blocks.py` | 119-168 | Dynamic model creation |
| `src/youlab_server/curriculum/blocks.py` | 48-116 | DynamicBlock base class |
| `src/youlab_server/curriculum/__init__.py` | 110-126 | Block registry caching |
| `src/youlab_server/storage/git.py` | 89-461 | GitUserStorage implementation |
| `src/youlab_server/storage/git.py` | 21-65 | Frontmatter parsing/formatting |
| `src/youlab_server/storage/blocks.py` | 163-211 | Letta sync logic |
| `src/youlab_server/server/agents.py` | 240-351 | Agent creation with blocks |

## How to Define Memory Blocks (MD Format)

### Step 1: Define Schema in TOML

In `config/courses/{course-id}/course.toml`:

```toml
[block.my_block]
label = "custom_label"          # Letta block type
shared = false                  # true for shared, false for per-user
description = "What this block is for"

# Define fields with dotted notation
field.field_name = { type = "string", default = "", description = "..." }
field.another_field = { type = "list", default = [], max = 10 }
field.counter = { type = "int", default = 0 }
```

### Step 2: Initialize User Storage

The system automatically creates `.data/users/{user_id}/memory-blocks/{label}.md` files when:
1. User is initialized via `/users/init` API
2. Agent edits the block via `edit_memory_block` tool
3. Manual creation via `GitUserStorage.write_block()`

### Step 3: Example Generated File

`.data/users/abc123/memory-blocks/my_block.md`:

```markdown
---
block: my_block
updated_at: '2026-01-16T00:00:00+00:00'
title: My Block
schema: college-essay/my_block
---

## Field Name

Content here...

## Another Field

*   Item 1
*   Item 2

## Counter

5
```

### Step 4: Access from Code

```python
from youlab_server.curriculum import curriculum

# Get block registry
registry = curriculum.get_block_registry("college-essay")
MyBlockClass = registry["my_block"]

# Create instance
block = MyBlockClass(field_name="value", counter=42)

# Serialize for Letta
memory_str = block.to_memory_string()

# Parse from string
parsed = MyBlockClass.from_memory_string(memory_str)
```

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    TOML Schema Layer                        │
│  config/courses/college-essay/course.toml                   │
│  [block.student]                                            │
│  field.profile = { type = "string", ... }                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ↓ CurriculumLoader.load_course()
                       ↓ _parse_blocks_v2()
                       ↓
┌──────────────────────┴──────────────────────────────────────┐
│              Pydantic Schema Models (Runtime)               │
│  BlockSchema(label="human", fields={"profile": ...})        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ↓ create_block_registry()
                       ↓ create_block_model()
                       ↓
┌──────────────────────┴──────────────────────────────────────┐
│           Dynamic Pydantic Models (Runtime)                 │
│  StudentBlock(DynamicBlock)                                 │
│    profile: str                                             │
│    insights: str                                            │
│    .to_memory_string() → YAML-like                          │
│    .from_memory_string() → parse back                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         ↓                           ↓
┌────────────────────┐      ┌────────────────────┐
│  Markdown Storage  │      │  Letta Memory      │
│  (per-user)        │      │  (agent runtime)   │
│                    │      │                    │
│  .data/users/      │←─────│  blocks.create()   │
│  {user_id}/        │ sync │  blocks.update()   │
│  memory-blocks/    │─────→│  agents.blocks.*   │
│    student.md      │      │                    │
│    journey.md      │      │  youlab_user_      │
│                    │      │  {id}_{label}      │
│  YAML frontmatter  │      │                    │
│  + MD body         │      │  Memory string     │
│                    │      │  (YAML-like)       │
│  Git versioned     │      │                    │
└────────────────────┘      └────────────────────┘
```

## Key Insights

1. **NOT a "new MD-based schema"** - It's a dual-layer system where TOML defines structure, MD stores data

2. **Dynamic models are powerful** - Runtime Pydantic generation enables flexible schemas without hardcoded classes

3. **Git provides free versioning** - Every block edit is a commit, enabling full history/rollback

4. **YAML frontmatter bridges formats** - Metadata in frontmatter, content in markdown body

5. **Memory string format is simple** - YAML-like key: value format for Letta consumption

6. **Shared vs per-agent blocks** - Schema-level decision impacts storage and attachment strategy

7. **Background agents use target notation** - `block.field` format for precise updates

## Related Research

- `thoughts/searchable/shared/research/2026-01-13-ARI-82-letta-memory-integration.md` - Letta blocks API and sync patterns
- `thoughts/searchable/shared/research/2026-01-12-ARI-78-unified-user-memory-blocks.md` - User-level memory blocks
- `thoughts/searchable/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md` - Memory System MVP implementation

## Open Questions

1. **Schema evolution**: How to handle schema changes when users have existing blocks?
2. **Migration**: If a field is added to schema, how to backfill existing user blocks?
3. **Validation**: Should MD files be validated against schema on read?
4. **Block deletion**: What happens when a block is removed from schema but files exist?
5. **Shared block updates**: If schema for shared block changes, how to update all agents?

## Recommendations for ARI-85

For creating a proof-of-concept single-module course with background agent:

1. **Define minimal block schema** in TOML:
   - One "student" block (label="human")
   - One "progress" block (label="journey")
   - Keep fields simple (2-3 per block)

2. **Use existing patterns**:
   - Copy structure from `college-essay/course.toml:86-108`
   - Use v2 `[block.name]` syntax with `field.*` notation

3. **Test block instantiation**:
   - Verify `curriculum.get_block_registry()` works
   - Confirm `.to_memory_string()` produces valid format

4. **Background agent query**:
   - Use `target = "progress.status"` format
   - Start with `merge = "replace"` for simplicity

5. **Validate Git storage**:
   - Check `.data/users/{user_id}/memory-blocks/` files created
   - Verify frontmatter parses correctly
   - Test git commit history

## Conclusion

YouLab's memory block system is a sophisticated dual-layer architecture:
- **TOML schemas** define structure and types
- **Markdown files** store actual user data with git versioning
- **Dynamic Pydantic models** bridge the two layers at runtime
- **Letta sync** keeps agent memory in sync with git storage

The "MD-based schema" is actually the **storage format**, not a replacement for TOML schemas. The system leverages the strengths of each format:
- TOML for declarative, typed schemas
- Markdown for human-readable, version-controlled data storage
- Pydantic for runtime validation and serialization

This design enables flexible course configuration while maintaining data integrity and full version history.
