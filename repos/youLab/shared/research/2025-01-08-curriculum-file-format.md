---
date: 2026-01-08T13:04:03+07:00
researcher: ARI
git_commit: cc9a73ffffd3322c9943a1f77d0c4f7abf0c3aa5
branch: main
repository: YouLab
topic: "Curriculum File Format - Current Implementation and Planned Structure"
tags: [research, codebase, curriculum, templates, yaml, markdown]
status: complete
last_updated: 2026-01-08
last_updated_by: ARI
---

# Research: Curriculum File Format

**Date**: 2026-01-08T13:04:03+07:00
**Researcher**: ARI
**Git Commit**: cc9a73ffffd3322c9943a1f77d0c4f7abf0c3aa5
**Branch**: main
**Repository**: YouLab

## Research Question

What is the current state of curriculum files in the codebase? How are they structured, and what format is used?

## Summary

**Current State**: The curriculum system is **planned for Phase 5 but NOT YET IMPLEMENTED**. Currently, course content is hardcoded in Python using Pydantic models.

**Existing Implementation**:
- `AgentTemplate` - Pydantic model for agent configuration
- `PersonaBlock` / `HumanBlock` - Structured memory blocks with serialization
- `TUTOR_TEMPLATE` - Single hardcoded template for college essay coaching

**Planned Format (Phase 5)**: Markdown files with YAML frontmatter, stored in `courses/` directory.

## Detailed Findings

### Current Template System

The codebase uses a Pydantic-based template system defined in `src/letta_starter/agents/templates.py`:

```python
class AgentTemplate(BaseModel):
    type_id: str           # Unique identifier (e.g., "tutor")
    display_name: str      # Human-readable name
    description: str       # Template description
    persona: PersonaBlock  # Agent persona configuration
    human: HumanBlock      # Initial user context
```

### PersonaBlock Structure (from `blocks.py`)

The `PersonaBlock` defines agent personality:

```python
class PersonaBlock(BaseModel):
    name: str              # Agent's name
    role: str              # Primary role/purpose
    capabilities: list[str]  # What the agent can do
    expertise: list[str]     # Domain expertise areas
    tone: str              # Communication style
    verbosity: str         # Response length preference
    constraints: list[str]   # Behavioral restrictions
```

### Current College Essay Coach (TUTOR_TEMPLATE)

The only course content that exists is hardcoded in `templates.py:19-47`:

```python
TUTOR_TEMPLATE = AgentTemplate(
    type_id="tutor",
    display_name="College Essay Coach",
    description="Primary tutor for college essay writing course",
    persona=PersonaBlock(
        name="YouLab Essay Coach",
        role="AI tutor specializing in college application essays",
        capabilities=[
            "Guide students through self-discovery exercises",
            "Help brainstorm and develop essay topics",
            "Provide constructive feedback on drafts",
            "Support emotional journey of college applications",
        ],
        expertise=[
            "College admissions",
            "Personal narrative",
            "Reflective writing",
            "Strengths-based coaching",
        ],
        tone="warm",
        verbosity="adaptive",
        constraints=[
            "Never write essays for students",
            "Always ask clarifying questions before giving advice",
            "Celebrate small wins and progress",
        ],
    ),
)
```

### Planned Phase 5 Format (from Roadmap.md)

The planned curriculum format uses **Markdown with YAML frontmatter**:

```markdown
# courses/college-essay/module-1.md
---
name: Self-Discovery
lessons:
  - strengths-assessment
  - processing-results
---

## Lesson: strengths-assessment
trigger: module_start
objectives:
  - Complete Clifton StrengthsFinder
  - Initial reaction conversation

### Agent Instructions
[Instructions for agent behavior]

### Completion Criteria
- Articulated one resonant strength
- Minimum 3 turns
```

### AgentTemplateRegistry

Templates are managed via a registry pattern (`templates.py:50-76`):

```python
class AgentTemplateRegistry:
    def register(template: AgentTemplate) -> None
    def get(type_id: str) -> AgentTemplate | None
    def list_types() -> list[str]
    def get_all() -> dict[str, AgentTemplate]
```

The global registry is instantiated at module load with `TUTOR_TEMPLATE` pre-registered.

## Code References

- `src/letta_starter/agents/templates.py:8-16` - AgentTemplate class definition
- `src/letta_starter/agents/templates.py:19-47` - TUTOR_TEMPLATE (current college essay coach)
- `src/letta_starter/agents/templates.py:50-76` - AgentTemplateRegistry
- `src/letta_starter/memory/blocks.py:28-131` - PersonaBlock with serialization
- `src/letta_starter/memory/blocks.py:134-274` - HumanBlock with serialization
- `docs/Roadmap.md:169-202` - Phase 5 Curriculum Parser specification

## Architecture Documentation

### Current Data Flow

```
Python code (templates.py)
    ↓
AgentTemplate (Pydantic model)
    ↓
AgentTemplateRegistry.register()
    ↓
AgentManager.get_or_create_agent() uses template
    ↓
Letta Agent created with PersonaBlock
```

### Memory Serialization

PersonaBlock and HumanBlock both have `to_memory_string()` and `from_memory_string()` methods for compact serialization:

```
[IDENTITY] YouLab Essay Coach | AI tutor specializing in college application essays
[CAPABILITIES] Guide students through self-discovery exercises, Help brainstorm...
[EXPERTISE] College admissions, Personal narrative, Reflective writing, Strengths-based coaching
[STYLE] warm, adaptive
[CONSTRAINTS] Never write essays for students; Always ask clarifying questions...
```

### What Does NOT Exist

- No `courses/` directory
- No `curriculum/` directory
- No markdown curriculum files
- No YAML curriculum files
- No curriculum parser code
- No hot-reload functionality
- No lesson/module data structures beyond what's in templates

## Open Questions

1. **Format Decision**: The planned format is Markdown with YAML frontmatter. The user is considering pure YAML instead. No existing implementation constrains this choice.

2. **Schema Definition**: Would need to define what fields a curriculum YAML file contains (modules, lessons, triggers, objectives, completion criteria, agent instructions).

3. **Parser Implementation**: Would need to build a parser that loads YAML/Markdown files and converts them to AgentTemplate or a new CurriculumTemplate structure.

4. **Hot-Reload Mechanism**: How to detect file changes and reload curriculum without restarting the server.
