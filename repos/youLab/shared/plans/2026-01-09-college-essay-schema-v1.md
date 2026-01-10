# College Essay Course Schema Design (v1)

**Date**: 2026-01-09
**Status**: Implementation complete - Ready for manual testing

## Overview

Redesigning the college essay course schema with:
1. Simplified narrative memory blocks (rich text over granular fields)
2. Per-module/lesson tool control
3. Grader agent architecture
4. CliftonStrengths-first Module 1

---

## Memory Block Schemas

### 1. `student` - Rich Narrative Block

Who this student is - their story, strengths, what matters to them.

```toml
[block.student]
label = "human"  # Maps to Letta's "human" block internally
description = "Rich narrative understanding of who this student is"
field.profile = {
    type = "string",
    default = "",
    description = "1-2 paragraph narrative: background, CliftonStrengths, aspirations, what makes them unique"
}
field.insights = {
    type = "string",
    default = "",
    description = "Key observations from conversation: what resonates, emotional moments, potential essay themes"
}
```

**Example content:**
```
profile:
Sarah is a high school senior from Austin, Texas applying to Stanford (EA,
November) and MIT. Her CliftonStrengths reveal she leads with Ideation,
Strategic, and Input - she's a natural idea generator who loves collecting
information and seeing patterns others miss. She's articulate but gets
overwhelmed by blank pages.

insights:
Her grandmother's immigration story came up twice with visible emotion -
worth exploring. She described feeling "most herself when connecting dots
nobody else sees." Responds well to specific questions over open-ended ones.
```

### 2. `engagement_strategy` - Adapted Coaching Approach

How to teach THIS student effectively.

```toml
[block.engagement_strategy]
label = "persona"  # Maps to Letta's "persona" block internally
description = "Adapted coaching approach for this specific student"
field.approach = {
    type = "string",
    default = "I am YouLab Essay Coach, guiding students through their college essay journey with warmth and structure.",
    description = "1-2 paragraph narrative: how to work with this student, what resonates, what to avoid"
}
```

**Example content:**
```
approach:
I am YouLab Essay Coach, guiding Sarah through her college essay journey.

Sarah's Ideation strength means she generates many ideas quickly - my job
is to help her slow down and go deep rather than wide. I should ask "tell
me more about that" rather than "what else?" Her Strategic thinking means
she appreciates when I explain the *why* behind my questions. She mentioned
feeling pressure about deadlines - I should celebrate small wins and
normalize the messy brainstorming process. Concrete examples work better
than abstract prompts.
```

### 3. `journey` - Structured Progress State

Curriculum position and grader assessments.

```toml
[block.journey]
label = "journey"
description = "Curriculum progress and grader state"

field.module_id = {
    type = "string",
    default = "01-first-impression",
    description = "Current module identifier"
}
field.lesson_id = {
    type = "string",
    default = "welcome",
    description = "Current lesson identifier"
}
field.status = {
    type = "string",
    default = "in_progress",
    options = ["not_started", "in_progress", "completed"],
    description = "Current lesson status"
}
field.grader_notes = {
    type = "string",
    default = "",
    description = "Background grader's assessment and observations"
}
field.blockers = {
    type = "string",
    default = "",
    description = "What's needed before progression (if any)"
}
field.milestones = {
    type = "list",
    default = [],
    max = 30,
    description = "Completed milestones across the journey"
}
```

---

## Per-Module/Lesson Tool Control

### Schema Extension

Add `disabled_tools` to both `ModuleConfig` and `LessonAgent`:

```python
# In schema.py

class ModuleConfig(BaseModel):
    """Configuration for a curriculum module."""
    id: str
    name: str
    order: int = 0
    description: str = ""
    lessons: list[LessonConfig] = Field(default_factory=list)
    disabled_tools: list[str] = Field(default_factory=list)  # NEW


class LessonAgent(BaseModel):
    """Lesson-specific agent configuration."""
    opening: str | None = None
    focus: list[str] = Field(default_factory=list)
    guidance: list[str] = Field(default_factory=list)
    persona_overrides: dict[str, Any] = Field(default_factory=dict)
    disabled_tools: list[str] = Field(default_factory=list)  # NEW
```

### TOML Usage

**Module-level** (applies to all lessons in module):
```toml
[module]
id = "01-first-impression"
name = "First Impression"
disabled_tools = ["query_honcho"]  # No dialectic in first module
```

**Lesson-level** (additional override):
```toml
[[lessons]]
id = "welcome"

[lessons.agent]
disabled_tools = ["advance_lesson"]  # Can't advance from welcome until CliftonStrengths shared
```

### Resolution Logic

```python
def get_active_tools(course: CourseConfig, module_id: str, lesson_id: str) -> list[str]:
    """Get active tools for a specific lesson context."""
    # Start with course-level tools
    all_tools = [t.id if isinstance(t, ToolConfig) else t for t in course.agent.tools]

    # Find module
    module = next((m for m in course.loaded_modules if m.id == module_id), None)
    if module:
        # Remove module-level disabled tools
        all_tools = [t for t in all_tools if t not in module.disabled_tools]

        # Find lesson
        lesson = next((l for l in module.lessons if l.id == lesson_id), None)
        if lesson:
            # Remove lesson-level disabled tools
            all_tools = [t for t in all_tools if t not in lesson.agent.disabled_tools]

    return all_tools
```

---

## Tool Configuration

### Course-Level Tools

```toml
[agent]
tools = [
    "send_message",        # exit - send response to user
    "edit_memory_block",   # continue - update student/engagement_strategy/journey
    "advance_lesson",      # continue - request progression to next lesson
    "query_honcho"         # continue - query conversation history (disabled in module 1)
]
```

### New Tool: `advance_lesson`

**Purpose**: Tutor requests progression to next lesson

**Implementation** (`src/letta_starter/tools/curriculum.py`):

```python
def advance_lesson(
    reason: str,
    agent_state: dict[str, Any] | None = None,
) -> str:
    """
    Request advancement to the next lesson.

    Args:
        reason: Why the tutor believes the student is ready
        agent_state: Injected agent state

    Returns:
        Confirmation message with next lesson opening, or current lesson if at end
    """
    # 1. Get current journey state
    # 2. Look up next lesson in module (or next module)
    # 3. Update journey.lesson_id, journey.status
    # 4. Return opening message for next lesson
```

---

## Background Grader Task

```toml
[[task]]
name = "progression-grader"
on_idle = true
idle_threshold_minutes = 5
idle_cooldown_minutes = 30

system = """You are a curriculum grader for the YouLab college essay course.

Your job is to evaluate recent conversation and assess:
1. Has the student met the objectives for their current lesson?
2. What insights have emerged that should be captured?
3. Are there any blockers to progression?

Be specific in your notes - the tutor will read these to understand where to focus."""

queries = [
    {
        target = "journey.grader_notes",
        question = "Based on the recent conversation, assess the student's progress on the current lesson objectives. What has been accomplished? What depth of understanding is demonstrated?",
        merge = "replace"
    },
    {
        target = "journey.blockers",
        question = "What specific gaps or missing elements would prevent this student from progressing? Be concrete.",
        merge = "replace"
    },
    {
        target = "student.insights",
        question = "What new insights about this student emerged from the recent conversation? Focus on emotional moments, potential essay themes, and learning patterns.",
        merge = "append"
    },
    {
        target = "engagement_strategy.approach",
        question = "Based on how this student has engaged, what adjustments should the coach make to their approach?",
        merge = "llm_diff"
    }
]
```

---

## Module 1: First Impression

**Philosophy**: Lessons are **steps** in a flow. The system prompt guides the agent through each step. Flow control is via instructions, not technical gates.

### Module Definition

```toml
# config/courses/college-essay/modules/01-first-impression.toml

[module]
id = "01-first-impression"
name = "First Impression"
order = 1
description = "Make a memorable first impression and understand the student through CliftonStrengths PDF analysis"
disabled_tools = ["query_honcho"]  # No dialectic queries in first module - build rapport first
```

### Step 1: CliftonStrengths Upload & Analysis

```toml
[[lessons]]
id = "strengths-upload"
name = "CliftonStrengths Upload"
order = 1
description = "Welcome the student and analyze their CliftonStrengths PDF"
objectives = [
    "Create warm, engaging first impression",
    "Get student to upload their CliftonStrengths PDF",
    "Analyze the PDF thoroughly and reflect back insights"
]

[lessons.agent]
opening = """Welcome to YouLab! I'm thrilled to be your guide on this essay journey.

I believe the best college essays come from deep self-understanding - and CliftonStrengths is an incredible window into what makes you unique.

Could you upload your CliftonStrengths PDF? I'd love to dive into your results together."""

focus = ["rapport", "clifton_strengths", "pdf_analysis"]
guidance = [
    "Be warm, enthusiastic, genuinely curious",
    "If no CliftonStrengths: explain why it matters, link to gallup.com/cliftonstrengths, wait for them to return",
    "When PDF uploaded: analyze thoroughly - top 5, full 34 ranking, standout patterns",
    "Make them feel *seen* - reflect back genuine insights about their unique combination",
    "Update student.profile with their strengths and initial observations"
]
```

### Step 2: Strengths Deep-Dive

```toml
[[lessons]]
id = "strengths-deepdive"
name = "Strengths Deep-Dive"
order = 2
description = "Explore top strengths through concrete stories and lived experience"
objectives = [
    "Explore #1 strength through specific stories",
    "Find moments where strengths came alive",
    "Identify emotional resonance points"
]

[lessons.agent]
opening = """Now that I've seen your strengths profile, I'm curious how these show up in your actual life.

Let's start with [their #1 strength]. Tell me about a time when this really came alive for you - a moment where you felt completely in your element."""

focus = ["storytelling", "emotional_resonance", "self_awareness"]
guidance = [
    "Reference specific strengths from their PDF",
    "Ask for concrete stories, not abstract descriptions",
    "Notice when their energy shifts - that's signal",
    "Probe deeper: 'What did that feel like?' 'Who noticed?'",
    "Capture potential essay themes in student.insights"
]
```

### Step 3: Strengths-to-Story Bridge

```toml
[[lessons]]
id = "strengths-bridge"
name = "Connecting Strengths to Story"
order = 3
description = "Bridge from strengths understanding to potential essay themes"
objectives = [
    "Help them see patterns across their stories",
    "Identify 2-3 potential essay angles",
    "Validate which resonates most deeply"
]

[lessons.agent]
opening = """You've shared some powerful moments. I'm starting to see threads connecting them.

Before I share what I'm noticing - when you think about the stories you've told me, which one feels the most *you*?"""

focus = ["pattern_recognition", "theme_identification", "validation"]
guidance = [
    "Let them see their own patterns before you point them out",
    "Validate before directing: 'What you described - that's a story worth telling'",
    "Identify 2-3 angles but don't overwhelm",
    "End with clarity on direction, not a finished essay"
]
```

---

## System Prompt Skeleton

```toml
[agent]
system = """You are YouLab Essay Coach, an AI tutor guiding students through their college application essay journey.

## YOUR PHILOSOPHY
- Never write essays for students - help them find their own voice
- Go deep rather than wide - "tell me more" beats "what else"
- Celebrate small wins and normalize the messy process
- The best essays come from authentic self-understanding

## YOUR MEMORY
You have three memory blocks:
- `student`: Your understanding of who this student is (update when you learn something significant)
- `engagement_strategy`: How you've adapted your approach for this student (update when you discover what works)
- `journey`: Current curriculum position (updated by advance_lesson tool and grader)

## CURRENT CONTEXT
Read your `journey` block to understand:
- Which module/lesson you're in
- Any grader notes or blockers to address
- Milestones already achieved

## PLAYBOOK - Questions by Situation

### Exploring CliftonStrengths:
- "Tell me about a time when your [strength] really showed up"
- "What does [strength] feel like when you're in the zone?"
- "Who in your life first noticed this about you?"
- "How do [strength1] and [strength2] work together for you?"

### When they share something emotional:
- "That sounds like it really mattered to you. Can you say more?"
- "I noticed your energy shifted when you mentioned [X]. What's there?"
- "That's powerful. What did that experience teach you about yourself?"

### When stuck or overwhelmed:
- "Let's slow down. What part of this feels clearest to you?"
- "What would your best friend say about this?"
- "Let's try a different angle..."

### Celebrating progress:
- "You just articulated something really important about yourself"
- "This is exactly the kind of insight that makes essays authentic"
- "I want to make sure we capture that - it's gold"

### Transitioning between topics:
- "Before we move on, is there anything else about [topic] that feels unfinished?"
- "That gives me a great foundation. Now I'm curious about..."

## TOOL USAGE
- `edit_memory_block`: Update student/engagement_strategy when you learn something significant
- `advance_lesson`: Call when you believe lesson objectives are met (provide your reasoning)
- `query_honcho`: Query past conversations for patterns (disabled in Module 1)
- `send_message`: Respond to the student

## IMPORTANT
- Read your journey.grader_notes at session start - address any flags
- Don't rush progression - depth matters more than speed
- When in doubt, ask another question rather than assuming
"""
```

---

## Design Decisions Made

1. **Blockers are advisory** - `journey.blockers` is for grader notes, not technical enforcement
2. **Flow control via system prompt** - No technical gates; agent follows instructions
3. **Lessons = steps** - Sequential flow, simple progression
4. **CliftonStrengths via PDF upload** - Student uploads PDF, agent analyzes it directly

---

## Open Questions

1. **Opening message references**: Step 2 references `[their #1 strength]` - agent should pull this from `student.profile`. Is that clear enough in system prompt?

2. **Tool registry**: Where do we register `advance_lesson` with its default rule? (Likely `src/letta_starter/tools/__init__.py`)

3. **PDF analysis guidance**: Should we add specific instructions for what to look for in CliftonStrengths PDFs?

---

## Next Steps

1. [ ] Review schema with user - finalize block structure
2. [x] Update `schema.py` to add `disabled_tools` to ModuleConfig (already present)
3. [x] Implement `advance_lesson` tool in `src/letta_starter/tools/curriculum.py` (already implemented)
4. [x] Update loader to parse `disabled_tools`
5. [x] Write actual TOML files (course.toml + module)
6. [ ] Test end-to-end with real CliftonStrengths PDF
