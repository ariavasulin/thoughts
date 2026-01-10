---
date: 2026-01-10T12:55:00+07:00
researcher: ariasulin
git_commit: 47ca115279c37e41f85e0dd3d7765dc2a1f7156d
branch: main
repository: YouLab
topic: "Do modules/lessons create their own Letta agents, or modify the existing one?"
tags: [research, codebase, curriculum, agents, letta, architecture]
status: complete
last_updated: 2026-01-10
last_updated_by: ariasulin
---

# Research: Curriculum Agent Architecture

**Date**: 2026-01-10T12:55:00+07:00
**Researcher**: ariasulin
**Git Commit**: 47ca115279c37e41f85e0dd3d7765dc2a1f7156d
**Branch**: main
**Repository**: YouLab

## Research Question

Do steps/lessons and modules create their own Letta agents, or do they just modify the existing Letta agent per student per course?

## Summary

**Modules and lessons do NOT create separate Letta agents.** The architecture is:

- **One Letta agent per user per course** - Created lazily on first interaction
- **Curriculum progression modifies the existing agent** - By updating the `journey` memory block
- **The `advance_lesson` tool** - Agent calls this to progress; it updates journey block with new module/lesson IDs

The single agent's behavior changes based on its memory state, not by spawning new agents.

## Detailed Findings

### Agent Creation: One Per User Per Course

Agents are created via `AgentManager.create_agent_from_curriculum()`:

```python
# src/letta_starter/server/agents.py:177-309
def create_agent_from_curriculum(
    self,
    user_id: str,
    course_id: str,
    user_name: str | None = None,
    block_overrides: dict[str, dict[str, Any]] | None = None,
) -> str:
    # Load course config
    course = curriculum.get(course_id)

    # Check for existing agent first
    existing = self.get_agent_id(user_id, agent_type)
    if existing:
        return existing  # Return existing, don't create new

    # Create single agent with curriculum config
    agent = self.client.agents.create(
        name=agent_name,  # "youlab_{user_id}_{course_id}"
        model=course.agent.model,
        system=course.agent.system,
        memory_blocks=memory_blocks,  # Per-agent blocks
        block_ids=shared_block_ids,   # Shared blocks
        tools=tool_names,
        metadata={
            "youlab_user_id": user_id,
            "youlab_agent_type": agent_type,
            "course_id": course_id,
        },
    )
```

**Key points**:
- Agent name: `"youlab_{user_id}_{course_id}"` (e.g., `youlab_alice_college-essay`)
- Cache key: `(user_id, agent_type)` tuple prevents duplicate creation
- All memory blocks created from course schema at agent creation time

### Journey Block: Curriculum State Machine

The `journey` block stores curriculum position and is the mechanism for progression:

```toml
# config/courses/college-essay/course.toml:94-103
[block.journey]
label = "journey"
description = "Curriculum progress and grader state"
field.module_id = { type = "string", default = "01-first-impression" }
field.lesson_id = { type = "string", default = "strengths-upload" }
field.status = { type = "string", default = "in_progress", options = ["not_started", "in_progress", "completed"] }
field.grader_notes = { type = "string", default = "" }
field.blockers = { type = "string", default = "" }
field.milestones = { type = "list", default = [], max = 30 }
```

**Runtime state example**:
```yaml
module_id: 01-first-impression
lesson_id: strengths-deepdive
status: in_progress
grader_notes: Student has engaged deeply with top strengths...
blockers:
milestones:
- 01-first-impression/strengths-upload
```

### Advance Lesson Tool: How Progression Works

The `advance_lesson` tool modifies the existing agent's memory:

```python
# src/letta_starter/tools/curriculum.py:27-85
def advance_lesson(
    reason: str,
    agent_state: dict[str, Any] | None = None,
) -> str:
    """
    Request advancement to the next lesson in the curriculum.
    Updates the journey block - does NOT create a new agent.
    """
    agent_id = agent_state.get("agent_id")
    course_id = _get_course_id_from_agent(agent_id)

    # Updates journey block and returns lesson opening
    result = _do_advance_lesson(agent_id, course_id, reason)
    return result
```

**Progression flow** (`curriculum.py:162-193`):
1. Find current position from journey block
2. Navigate to next lesson (same module) or first lesson (next module)
3. Update journey block with new position
4. Return opening message for new lesson

**Journey update** (`curriculum.py:264-302`):
```python
def _set_journey_and_get_opening(...):
    journey_data = _get_journey_block(agent_id)

    # Add milestone for completed lesson
    milestones = journey_data.get("milestones", [])
    milestone = f"{old_module}/{old_lesson}"
    milestones.append(milestone)

    # Update to new lesson
    journey_data["module_id"] = module_id
    journey_data["lesson_id"] = lesson_id
    journey_data["status"] = "in_progress"
    journey_data["grader_notes"] = ""  # Clear for new lesson
    journey_data["blockers"] = ""
    journey_data["milestones"] = milestones

    # Write back to agent's memory
    _update_journey_block(agent_id, journey_data)
```

### Lesson-Specific Configuration

Lessons define opening messages and behavior hints, but don't create agents:

```toml
# config/courses/college-essay/modules/01-first-impression.toml:32-46
[[lessons]]
id = "strengths-upload"
objectives = ["Create warm first impression", "Get PDF upload", "Analyze thoroughly"]

[lessons.agent]
opening = """Welcome to YouLab! I'm thrilled to be your guide..."""
focus = ["rapport", "clifton_strengths"]
guidance = ["Be warm, enthusiastic", "Analyze PDF thoroughly"]
disabled_tools = ["advance_lesson"]  # Can't advance until analyzed
```

**How this affects behavior**:
- `opening` - Returned by `advance_lesson` tool for agent to use
- `focus` and `guidance` - Available in curriculum config for prompts
- `disabled_tools` - Prevents certain actions per lesson (not yet implemented in tool validation)

### Background Graders: Modify Agent Memory

Background tasks also modify the single agent's memory, not create new agents:

```toml
# config/courses/college-essay/course.toml:109-131
[[task]]
name = "progression-grader"
on_idle = true
queries = [
    { target = "journey.grader_notes", question = "Assess progress...", merge = "replace" },
    { target = "student.insights", question = "What insights emerged?", merge = "append" }
]
```

**Flow**:
1. Background runner finds idle agents
2. Uses Honcho dialectic queries to analyze conversation
3. Updates agent's memory blocks with insights
4. Agent reads updated memory on next message

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Per User Per Course                       │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │            One Letta Agent (youlab_alice_college-essay)  │ │
│ │                                                          │ │
│ │  Memory Blocks:                                          │ │
│ │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │ │
│ │  │   student    │  │   persona    │  │   journey    │   │ │
│ │  │   (human)    │  │  (persona)   │  │  (journey)   │   │ │
│ │  │              │  │              │  │              │   │ │
│ │  │ profile:     │  │ approach:    │  │ module_id:   │   │ │
│ │  │ insights:    │  │              │  │ lesson_id:   │   │ │
│ │  └──────────────┘  └──────────────┘  │ status:      │   │ │
│ │                                       │ grader_notes:│   │ │
│ │                                       │ milestones:  │   │ │
│ │                                       └──────────────┘   │ │
│ └─────────────────────────────────────────────────────────┘ │
│                            │                                 │
│                    ┌───────┴───────┐                        │
│                    ▼               ▼                        │
│          ┌─────────────┐   ┌─────────────┐                 │
│          │advance_lesson│   │edit_memory │                 │
│          │   (tool)     │   │   (tool)    │                 │
│          └──────┬──────┘   └─────────────┘                 │
│                 │                                           │
│                 ▼                                           │
│    ┌────────────────────────────────────┐                  │
│    │ Updates journey block:              │                  │
│    │ - module_id → next module          │                  │
│    │ - lesson_id → next lesson          │                  │
│    │ - milestones += completed          │                  │
│    │ - grader_notes = ""                │                  │
│    │                                    │                  │
│    │ Returns: lesson.agent.opening      │                  │
│    └────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## Code References

- `src/letta_starter/server/agents.py:177-309` - Agent creation from curriculum
- `src/letta_starter/server/agents.py:147-175` - Legacy create_agent (delegates to curriculum)
- `src/letta_starter/tools/curriculum.py:27-85` - advance_lesson tool
- `src/letta_starter/tools/curriculum.py:162-193` - Navigation logic (next lesson/module)
- `src/letta_starter/tools/curriculum.py:264-302` - Journey block update
- `src/letta_starter/tools/curriculum.py:196-249` - Journey block reading/parsing
- `src/letta_starter/curriculum/schema.py:219-228` - StepConfig/LessonConfig schema
- `config/courses/college-essay/course.toml:94-103` - Journey block definition
- `config/courses/college-essay/modules/01-first-impression.toml:32-46` - Lesson configuration

## Historical Context

This architecture follows the pattern established in the curriculum system redesign:
- Single agent per user enables persistent memory and relationship building
- Memory blocks serve as the "state machine" for curriculum progression
- Tools (`advance_lesson`, `edit_memory_block`) are the primary mutation interface
- Background graders update memory asynchronously based on conversation analysis

## Open Questions

None - the architecture is clear: one agent per user per course, modified by tools and background tasks rather than spawning new agents.
