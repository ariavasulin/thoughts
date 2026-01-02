---
date: 2025-12-25T19:41:00-08:00
author: ariasulin
type: project-context
status: active
last_updated: 2025-12-25
---

# YouLab Platform — Project Context

Course delivery platform with personalized AI tutoring. First course: college essay coaching.

## Vision

**Key Insight**: "What we are really building is a platform and novel delivery system for online courses."

The platform is course-agnostic — YouLab college essay coaching is just the first course. The architecture should support any curriculum defined in markdown.

## Background

- User has been exploring Letta vs Honcho for months
- Working with Joey Gonzalez (MemGPT/Letta creator) at Berkeley
- Building over Christmas break 2024, aiming to test with real students in January 2025
- Mom is a college counselor, potential pilot facilitator
- User's background: transferred from CC, former carpenter, senior at Berkeley

## Architecture Decisions Made

### Letta vs Honcho — NOT Redundant

This was a key clarification. They serve different purposes:

| Layer | Letta | Honcho |
|-------|-------|--------|
| Role | Agent framework (like LangGraph, Agno) | Theory of Mind layer |
| Owns | Structured curriculum state, memory blocks, assessments | Emergent psychological picture, dynamic identity modeling |
| Data | Module progress, Clifton strengths results, explicit facts | Inferred preferences, personality model, engagement patterns |
| API | Agent runtime, memory CRUD | Dialectic API for deep queries |

### Stack

```
OpenWebUI (Pipe) → LettaStarter (Python) → Letta Server → Claude API
                                        → Honcho (planned)
```

- **OpenWebUI**: Chat frontend with Pipe extension system for custom backends
- **LettaStarter**: Python wrapper adding memory management, observability, and curriculum logic
- **Letta Server**: Agent runtime with persistent memory (core + archival)
- **Honcho**: Additive ToM layer (not yet integrated)

### Curriculum as Markdown

Multi-file structure:
- One file per module, plus a `course.md` manifest
- Lessons defined inline within module files
- YAML frontmatter for metadata
- Agent instructions, completion criteria, memory block references all in markdown

## What Exists (Implemented)

### LettaStarter Framework (`src/letta_starter/`)

**agents/**
- `BaseAgent` — Core agent class with integrated memory + tracing
- Factory functions: `create_default_agent()`, `create_coding_agent()`, `create_research_agent()`
- `AgentRegistry` — Multi-agent management

**memory/**
- `PersonaBlock` / `HumanBlock` — Pydantic schemas that serialize to token-efficient strings
- Rotation strategies: `AggressiveRotation` (70%), `PreservativeRotation` (90%), `AdaptiveRotation` (80% adaptive)
- `MemoryManager` — Orchestrates lifecycle, handles archival rotation

**pipelines/**
- `Pipeline` class for OpenWebUI Pipe integration
- Configurable via OpenWebUI admin (Valves)

**observability/**
- Structured logging with structlog
- Metrics collection (token usage, latency, cost estimation)
- Langfuse tracing integration

**config/**
- Pydantic settings from environment variables

### Tests

- Memory block serialization/deserialization
- Rotation strategy thresholds
- Context metrics calculations

## What Doesn't Exist (Planned)

### Curriculum Layer

- `courses/` directory structure
- Course manifest parsing (`course.md`)
- Module/lesson parsing (`module-*.md`)
- Lesson schema (triggers, objectives, completion criteria)
- Curriculum state machine (progression logic)

### Honcho Integration

- Dialectic API client
- Personalization hooks in agent instructions
- ToM context injection points
- Session-level vs long-term modeling

### Student Data

- Per-student state persistence
- Memory block schemas for course-specific data
- Assessment result storage

## Open Questions

### 1. Lesson Schema Design

What frontmatter fields does each lesson need?

**Proposed fields:**
```yaml
## Lesson: lesson-name
trigger: previous-lesson.complete | module_start | manual
objectives:
  - Objective 1
  - Objective 2
completion_criteria:
  - type: llm_judged | explicit | turns_minimum
    value: ...
memory_blocks:
  - block_name
honcho_hooks:
  - pre_session | mid_conversation | background
```

**Unresolved:**
- How are triggers expressed? Simple strings? Expressions?
- How does LLM-judged completion work? What prompt template?
- How are memory block references resolved at runtime?

### 2. Memory Block Schemas

What structured data does Letta hold per student?

**Proposed blocks:**
- `clifton_strengths` — Top 5 strengths, full ranking, student reactions
- `module_progress` — Current module, completed lessons, timestamps
- `assessment_results` — Quiz scores, essay drafts, feedback history
- `student_profile` — Name, goals, background, preferences

**Unresolved:**
- Exact field definitions
- How do blocks reference each other?
- Size limits per block?

### 3. Honcho Integration Points

When/how does the agent query Dialectic?

**Options:**
- **Pre-session context hydration** — Query Honcho before conversation starts
- **Mid-conversation tool call** — Agent can call Honcho as a tool
- **Background only** — Honcho runs async, updates context periodically

**Unresolved:**
- Which approach(es) to implement first?
- What queries does the agent make?
- How is Honcho context injected into system prompt?

### 4. Student Data Storage

Where does per-student state live?

**Options:**
- **Letta only** — All state in memory blocks (core + archival)
- **Postgres/SQLite** — Separate student table, Letta for conversation memory
- **Honcho only** — Honcho stores everything, Letta is stateless

**Leaning toward:** Hybrid — Letta for conversation memory + structured course state, Postgres for user accounts + metadata

### 5. Multi-Agent Architecture

Does the course need multiple specialized agents?

**Possibilities:**
- Single agent per student, persona changes per module
- Specialized agents per module type (reflection, drafting, feedback)
- Orchestrator agent that delegates to specialists

## Proposed Curriculum Format

### Course File (`course.md`)

```markdown
---
name: College Essay Mastery
version: 1.0
description: 8-week guided self-discovery and essay writing
modules:
  - module-1-discovery
  - module-2-story-mining
  - module-3-drafting
default_pedagogy: reflective
---

# College Essay Mastery

Overview content here...
```

### Module File (`module-1-discovery.md`)

```markdown
---
name: Self-Discovery
duration: 2 weeks
pedagogy: reflective, exploratory
memory_blocks:
  - clifton_strengths
  - time_inventory
lessons:
  - strengths-assessment
  - processing-results
  - time-inventory
  - pattern-recognition
---

# Module 1: Self-Discovery

## Lesson: strengths-assessment
trigger: module_start
objectives:
  - Complete Clifton StrengthsFinder
  - Initial reaction conversation

### Agent Instructions

You have access to their strengths results. Start by asking
how the results landed. Listen for resistance or surprise.

### Completion Criteria

- Articulated one resonant strength
- Named one thing assessment missed
- Minimum 3 turns

---

## Lesson: processing-results
trigger: strengths-assessment.complete
...
```

## Next Steps (Priority Order)

1. **Nail down lesson schema** — This is where agent behavior gets defined. Need to finalize frontmatter fields before writing the parser.

2. **Define memory block schemas** — Clifton strengths format, module progress tracking, assessment results.

3. **Build curriculum parser** — Python module that loads `courses/` and creates in-memory course graph.

4. **Implement curriculum state machine** — Track student position, evaluate completion criteria, trigger transitions.

5. **Sketch Honcho integration** — Minimal proof-of-concept with Dialectic API.

## References

### External Resources

- OpenWebUI Pipes: https://docs.openwebui.com/features/plugin/functions/pipe/
- Honcho GitHub: https://github.com/plastic-labs/honcho
- Letta (MemGPT): https://github.com/letta-ai/letta

### Past Conversations (Claude.ai)

- Honcho vs Letta decision: https://claude.ai/chat/821015f0-4174-4d3f-a53a-7aa71a798739
- YouLab business model / architecture: https://claude.ai/chat/d3337810-f3c1-4f66-857f-83f7875dcf8b
- Nested learning architectures: https://claude.ai/chat/653be51d-f7cb-474e-b704-a450e3919587

## Linear

- Team: Ariav (`dd52dd50-3e8c-43c2-810f-d79c71933dc9`)
- Project: YouLab (`eac4c2fe-bee6-4784-9061-05aaba98409f`)
