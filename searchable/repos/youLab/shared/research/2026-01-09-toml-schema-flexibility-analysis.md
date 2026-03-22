---
date: 2026-01-09T09:43:48+07:00
researcher: ariasulin
git_commit: 3d8dbffc4b38f61f2e099982610a355a7360926a
branch: main
repository: YouLab
topic: "TOML Schema Flexibility Analysis"
tags: [research, codebase, toml, configuration, schema, curriculum]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: TOML Schema Flexibility Analysis

**Date**: 2026-01-09T09:43:48+07:00
**Researcher**: ariasulin
**Git Commit**: 3d8dbffc4b38f61f2e099982610a355a7360926a
**Branch**: main
**Repository**: YouLab

## Research Question

Analyze the current TOML Schema, config and what is still managed directly in Python. What part of the schema is flexible and what part is hardcoded/inflexible?

## Summary

The YouLab codebase implements a dual-mode configuration system where TOML files define course-level configuration (agents, memory blocks, background agents, messages) while Python code handles runtime orchestration, serialization formats, and infrastructure settings. The TOML schema is highly flexible for defining *what* content goes into agents and memory blocks, but the Python code controls *how* that content is processed, serialized, and stored.

**Key finding**: There are two parallel pathways for agent creation - template-based (Python-defined) and curriculum-based (TOML-driven). The template-based path uses hardcoded model/embedding values while the curriculum-based path reads everything from TOML.

## Detailed Findings

### Flexible (TOML-Configured)

#### 1. Course Metadata
**File**: `config/courses/{course-id}/course.toml`
**Schema**: `src/letta_starter/curriculum/schema.py:357-362`

All course metadata is configurable:
- `id` - Course identifier (required)
- `name` - Display name (required)
- `version` - Semantic version (default: "1.0.0")
- `description` - Course description
- `modules` - List of module file references

#### 2. Agent Configuration
**Schema**: `src/letta_starter/curriculum/schema.py:132-151`
**Example**: `config/courses/college-essay/course.toml:17-33`

Fully configurable via `[agent]` table:
- `model` - LLM model identifier (e.g., "anthropic/claude-sonnet-4-20250514")
- `embedding` - Embedding model (e.g., "openai/text-embedding-3-small")
- `context_window` - Context window size (default: 128000)
- `max_response_tokens` - Max response tokens (default: 4096)
- `system` - Multi-line system prompt

#### 3. Tool Configuration
**Schema**: `src/letta_starter/curriculum/schema.py:105-124`
**Example**: `config/courses/college-essay/course.toml:35-46`

Tools are configurable via `[[agent.tools]]` arrays:
- `id` - Tool identifier
- `enabled` - Enable/disable flag
- `rules.type` - Exit behavior ("exit_loop", "continue_loop", "run_first")
- `rules.max_count` - Maximum invocation count

#### 4. Memory Block Schemas
**Schema**: `src/letta_starter/curriculum/schema.py:64-98`
**Example**: `config/courses/college-essay/course.toml:51-76`

This is the most flexible part of the schema. Any block with any fields can be defined:

```toml
[blocks.{block_name}]
label = "persona"  # Letta's internal label
description = "..."

[blocks.{block_name}.fields]
{field_name} = {
    type = "string|int|float|bool|list|datetime",
    default = ...,
    options = [...],  # For dropdowns
    max = 10,         # For lists
    description = "..."
}
```

The dynamic block generator at `src/letta_starter/curriculum/blocks.py:29-86` creates Pydantic models at runtime from these schemas.

#### 5. Background Agent Configuration
**Schema**: `src/letta_starter/curriculum/schema.py:159-206`
**Example**: `config/courses/college-essay/course.toml:81-120`

Fully configurable:
- `enabled` - Enable/disable flag
- `agent_types` - Agent types to process
- `batch_size` - Users per batch
- `triggers.schedule` - Cron expression (defined but not implemented)
- `triggers.manual` - Manual trigger flag
- `triggers.idle.*` - Idle trigger settings (defined but not implemented)

**Dialectic queries** (`[[background.{id}.queries]]`) are fully configurable:
- `id`, `question` - Query identification
- `session_scope` - "all", "recent", "current"
- `recent_limit` - Sessions to consider
- `target_block`, `target_field` - Memory target
- `merge_strategy` - "append", "replace", "llm_diff"

#### 6. UI Messages
**Schema**: `src/letta_starter/curriculum/schema.py:307-319`
**Example**: `config/courses/college-essay/course.toml:126-129`

All messages configurable:
- `welcome_first` - First-time user greeting
- `welcome_returning` - Returning user greeting
- `error_unavailable` - Error message

#### 7. Module and Lesson Definitions
**Schema**: `src/letta_starter/curriculum/schema.py:251-300`
**Location**: `config/courses/{course-id}/modules/{module-id}.toml`

Complete curriculum structure is configurable:
- Module metadata (id, name, order, description)
- Lesson arrays with objectives
- Completion criteria (required_fields, min_turns, min_list_length)
- Per-lesson agent overrides (opening, focus, guidance, persona_overrides)

---

### Inflexible (Hardcoded in Python)

#### 1. Agent Naming Convention
**Location**: `src/letta_starter/server/agents.py:39`, `src/letta_starter/background/runner.py:144,225`

The agent naming pattern is hardcoded:
```python
f"youlab_{user_id}_{agent_type}"
```

This affects:
- Agent discovery in background runner
- Agent cache key format
- Agent lookup operations

#### 2. Memory Block Type System
**Location**: `src/letta_starter/curriculum/schema.py:24-32`, `src/letta_starter/curriculum/blocks.py:19-26`

Only 6 field types are supported:
- `string` → `str`
- `int` → `int`
- `float` → `float`
- `bool` → `bool`
- `list` → `list[str]` (always strings)
- `datetime` → `datetime | None`

Cannot define custom types, nested objects, or lists of non-string types.

#### 3. Serialization Format
**Location**: `src/letta_starter/curriculum/blocks.py:129-162`, `src/letta_starter/memory/blocks.py:75-131,180-274`

The memory string format is hardcoded:
```
[LABEL] value
```

- Labels are uppercase field names for dynamic blocks
- Static blocks use custom labels like `[IDENTITY]`, `[CAPABILITIES]`
- Lists use semicolon separator
- Booleans serialize as "yes"/"no"
- Datetimes use ISO format

#### 4. Template-Based Agent Creation Uses Hardcoded Models
**Location**: `src/letta_starter/server/agents.py:111-112`

When using `AgentManager.create_agent()` (template-based path):
```python
model="openai/gpt-4o-mini",           # Hardcoded
embedding="openai/text-embedding-ada-002",  # Hardcoded
```

This is different from `create_agent_from_curriculum()` which reads from TOML.

#### 5. Pre-configured Personas
**Location**: `src/letta_starter/memory/blocks.py:278-321`

Three static personas are hardcoded:
- `DEFAULT_PERSONA` - General-purpose assistant
- `CODING_ASSISTANT_PERSONA` - Coding specialist
- `RESEARCH_ASSISTANT_PERSONA` - Research specialist

#### 6. TUTOR_TEMPLATE
**Location**: `src/letta_starter/agents/templates.py:28-57`

The tutor template has hardcoded values:
- `name="YouLab Essay Coach"`
- `role="AI tutor specializing in college application essays"`
- Capabilities, expertise, constraints all hardcoded
- Tool list: `[query_honcho, edit_memory_block]`

#### 7. Memory Management Thresholds
**Location**: `src/letta_starter/memory/strategies.py:12-15,97,122,163-186`

Rotation strategy parameters are hardcoded:
- `ROTATION_HISTORY_MIN_SIZE = 5`
- `ROTATION_HISTORY_MAX_SIZE = 20`
- `ROTATION_INTERVAL_TOO_FAST = 0.5`
- `ROTATION_INTERVAL_TOO_SLOW = 0.9`
- AggressiveRotation threshold: `0.7`
- PreservativeRotation threshold: `0.9`
- AdaptiveRotation base: `0.8`, step: `0.05`, min: `0.6`, max: `0.95`

#### 8. Protected Memory Fields
**Location**: `src/letta_starter/tools/memory.py:19`

Cannot be edited by agents:
```python
PROTECTED_FIELDS = {"persona.name", "persona.role"}
```

#### 9. Audit Entry Format
**Location**: `src/letta_starter/memory/enricher.py:23-24,223-240`

Audit trail format is hardcoded:
- `AUDIT_CONTENT_MAX_CHARS = 200`
- `AUDIT_QUERY_MAX_CHARS = 100`
- Entry structure with timestamp, source, block, field, strategy

#### 10. Search Limits
**Location**: Multiple files

Default limit of 5 appears in multiple places:
- `src/letta_starter/memory/manager.py:271`
- `src/letta_starter/agents/base.py:201`
- `src/letta_starter/honcho/client.py:246`
- `src/letta_starter/server/strategy/router.py:84`

#### 11. Cost Calculation Constants
**Location**: `src/letta_starter/observability/metrics.py:120-130,248`

Model pricing is hardcoded:
- GPT-4: $0.03/$0.06 per 1K tokens
- GPT-4o-mini: $0.00015/$0.0006
- Claude models: Various prices
- Default fallback: GPT-4 prices

#### 12. Background Agent Limitations
**Location**: `src/letta_starter/background/runner.py:199`

- Only first agent_type from list is processed for enrichment
- Schedule/cron triggers defined in schema but not implemented
- Idle triggers defined in schema but not implemented
- Only manual triggers work

#### 13. Service Infrastructure
**Location**: `src/letta_starter/config/settings.py`, `src/letta_starter/server/main.py`

Hardcoded defaults:
- Default host: `127.0.0.1`
- Default port: `8100`
- Letta URL: `http://localhost:8283`
- Honcho workspace: `youlab`
- Config directory: `config/courses`
- Service version: `0.1.0`

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Configuration Sources                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  TOML (Flexible)                    │  Python (Hardcoded)            │
│  ═══════════════                    │  ═════════════════             │
│                                     │                                │
│  course.toml                        │  settings.py                   │
│  ├── [course] metadata              │  ├── Service ports/URLs        │
│  ├── [agent] model, prompt          │  ├── Default models            │
│  ├── [blocks.*] schemas             │  └── Memory limits             │
│  ├── [background.*] config          │                                │
│  └── [messages] strings             │  blocks.py                     │
│                                     │  ├── PersonaBlock fields       │
│  modules/*.toml                     │  ├── HumanBlock fields         │
│  ├── [module] metadata              │  └── Pre-configured personas   │
│  └── [[lessons]] curriculum         │                                │
│                                     │  templates.py                  │
│                                     │  └── TUTOR_TEMPLATE            │
│                                     │                                │
│                                     │  strategies.py                 │
│                                     │  └── Rotation thresholds       │
│                                     │                                │
│                                     │  agents.py                     │
│                                     │  ├── Agent naming pattern      │
│                                     │  └── Template model defaults   │
│                                     │                                │
└──────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          Agent Creation Paths                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Path 1: Template-Based              Path 2: Curriculum-Based        │
│  ═════════════════════              ═══════════════════════         │
│                                                                      │
│  AgentManager.create_agent()        AgentManager.create_agent_       │
│                                     from_curriculum()                │
│  ┌─────────────────────┐            ┌─────────────────────┐         │
│  │ Uses templates.py   │            │ Uses course.toml    │         │
│  │ TUTOR_TEMPLATE      │            │ config              │         │
│  ├─────────────────────┤            ├─────────────────────┤         │
│  │ Model: HARDCODED    │            │ Model: FROM TOML    │         │
│  │ (gpt-4o-mini)       │            │ (configurable)      │         │
│  ├─────────────────────┤            ├─────────────────────┤         │
│  │ Blocks: Python      │            │ Blocks: Dynamic     │         │
│  │ PersonaBlock        │            │ from TOML schema    │         │
│  │ HumanBlock          │            │                     │         │
│  └─────────────────────┘            └─────────────────────┘         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Code References

### TOML Schema Files
- `src/letta_starter/curriculum/schema.py:327-378` - CourseConfig top-level schema
- `src/letta_starter/curriculum/schema.py:64-98` - FieldSchema and BlockSchema
- `src/letta_starter/curriculum/schema.py:132-151` - AgentConfig
- `src/letta_starter/curriculum/schema.py:176-206` - BackgroundAgentConfig
- `src/letta_starter/background/schema.py:60-70` - Alternate BackgroundAgentConfig (simpler)

### Configuration Loading
- `src/letta_starter/curriculum/loader.py:43-107` - CurriculumLoader.load_course()
- `src/letta_starter/curriculum/loader.py:159-179` - _parse_blocks() TOML → BlockSchema
- `src/letta_starter/curriculum/blocks.py:29-86` - generate_block_model() dynamic model creation

### Agent Creation
- `src/letta_starter/server/agents.py:81-128` - Template-based create_agent() (hardcoded model)
- `src/letta_starter/server/agents.py:130-245` - Curriculum-based create_agent_from_curriculum() (TOML model)

### Hardcoded Values
- `src/letta_starter/server/agents.py:111-112` - Hardcoded model/embedding in template path
- `src/letta_starter/server/agents.py:39` - Agent naming pattern
- `src/letta_starter/memory/strategies.py:12-15,97,122,163` - Rotation thresholds
- `src/letta_starter/tools/memory.py:19` - Protected fields
- `src/letta_starter/memory/blocks.py:278-321` - Pre-configured personas

---

## Historical Context (from thoughts/)

No prior research documents specifically about TOML schema flexibility were found. This is the first comprehensive analysis of the configuration system architecture.

---

## Related Research

None found in thoughts/shared/research/ directory.

---

## Open Questions

1. **Template vs Curriculum Path**: Should the template-based path be deprecated in favor of curriculum-based configuration? Currently they serve different use cases but create maintenance burden.

2. **Hardcoded Model in Template Path**: The hardcoded `gpt-4o-mini` model in `create_agent()` appears to be an oversight since the curriculum path reads from TOML. Should this be unified?

3. **Background Trigger Implementation**: The schema supports `schedule` (cron) and `idle` triggers, but only `manual` is implemented. Are these planned?

4. **Memory Type System**: The current 6-type system doesn't support nested structures or custom types. Is this limitation by design or a future enhancement?

5. **Agent Naming Convention**: The `youlab_` prefix is hardcoded throughout. Should this be configurable for multi-tenant deployments?
