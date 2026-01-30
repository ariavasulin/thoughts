---
date: 2026-01-30T21:53:39Z
researcher: Claude
git_commit: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "Welcome Course Implementation Gap Analysis"
tags: [research, codebase, welcome-course, toml, modules, onboarding, architecture]
status: complete
last_updated: 2026-01-30
last_updated_by: Claude
---

# Research: Welcome Course Implementation Gap Analysis

**Date**: 2026-01-30T21:53:39Z
**Researcher**: Claude
**Git Commit**: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

What is the current state of affairs in the codebase to know what needs to be done to implement the first welcome course? Reference: `/Users/ariasulin/Desktop/welcome.toml`

## Summary

The welcome course requires a **complete course/module infrastructure** that currently exists only in the legacy YouLab Server (Letta-based) stack. The active Ralph backend has **none of the required systems** for courses, modules, TOML configuration, or onboarding flows. Here's what needs to be built:

| Component | Welcome.toml Requires | Ralph Has | Gap |
|-----------|----------------------|-----------|-----|
| TOML Parsing | Course/agent/block/task config | None | Complete rebuild |
| Module System | 3-phase progression (Presence → Patterns → Possibilities) | None | New system needed |
| First Message | Welcome content on new user | None | New trigger needed |
| Block Templates | 4 blocks with templates | Schema exists, no templates | Template initialization |
| Background Tasks | Idle-triggered gentle-return, cron graduation-reflection | Infrastructure exists | Task definition loading |
| Agent Selection | Route users to Welcome agent | Hardcoded single agent | Course routing needed |

## Detailed Findings

### 1. TOML Configuration System

#### What welcome.toml Expects

The welcome course TOML defines:

```toml
[agent]                      # Agent configuration
name = "Welcome"
model = "openai/gpt-4o"
system_prompt = "..."        # Multi-phase onboarding instructions
tools = ["memory_blocks", "honcho_tools", "file_tools"]
blocks = ["origin_story", "tech_relationship", "ai_partnership", "progress"]

[[block]]                    # Memory block schemas with templates
label = "origin_story"
title = "Origin Story"
template = "## Who I Am At My Best\n..."

[[task]]                     # Background tasks
name = "gentle-return"
trigger = { type = "idle", idle_minutes = 4320, cooldown_minutes = 10080 }
system_prompt = "..."
tools = ["honcho_tools", "memory_blocks"]
blocks = ["origin_story", "onboarding_progress"]

[first_message]              # Initial welcome content
content = "Welcome to YouLab..."
```

#### Current State in Ralph

**Ralph has NO TOML configuration parsing.** The entire stack uses hardcoded values:

| Aspect | Ralph Implementation | Location |
|--------|---------------------|----------|
| Agent config | Hardcoded per-request | `src/ralph/server.py:283-297` |
| System prompt | Static base instructions | `src/ralph/server.py:187-245` |
| Tools | Fixed list of 5 tools | `src/ralph/server.py:288-294` |
| Model | Environment variable | `RALPH_OPENROUTER_MODEL` |
| Block schemas | None (ad-hoc creation) | N/A |

**Legacy YouLab Server HAS TOML parsing** but is deprecated:

- Parser: `src/youlab_server/curriculum/loader.py:43-457`
- Schema definitions: `src/youlab_server/curriculum/schema.py:1-361`
- Existing courses: `config/courses/college-essay/`, `config/courses/default/`

#### Gap: TOML Loading for Ralph

Ralph needs:
1. TOML parser that loads course configurations
2. Agent factory that builds from TOML instead of hardcoding
3. Block schema registry for template initialization
4. Tool selection based on course config

### 2. Module/Course System

#### What welcome.toml Expects

Three-phase progression through onboarding:

```
Phase 1: Presence (First Conversation)
  - Who are you at your best?
  - Goals, energy sources
  → Update [origin_story] block

Phase 2: Patterns (Second Conversation)
  - Technology relationship
  - Intentionality concept
  → Update [tech_relationship] block

Phase 3: Possibilities (Third Conversation)
  - AI partnership vision
  - Guardrails definition
  → Update [ai_partnership] block + graduation
```

Progress tracked in `[onboarding_progress]` block with checklist.

#### Current State in Ralph

**Ralph has NO module/course system:**

- No course routing (`src/ralph/server.py:167` serves all users identically)
- No progression tracking (no journey block, no advance_lesson tool)
- No phase detection (agent doesn't know which phase user is in)
- All users get same agent with same instructions

**Legacy YouLab Server HAD module system:**

- Journey block: `field.module_id`, `field.lesson_id`, `field.status`, `field.milestones`
- Advance lesson tool: `src/youlab_server/tools/curriculum.py:27-326`
- Module loading: `src/youlab_server/curriculum/loader.py:403-456`

#### Gap: Module Progression for Ralph

Ralph needs:
1. Phase/module tracking (which phase is user in?)
2. Phase-aware instructions (base prompt changes per phase)
3. Phase advancement logic (when to transition)
4. Graduation detection (all phases complete)

### 3. First Message / Onboarding Trigger

#### What welcome.toml Expects

```toml
[first_message]
content = """
Welcome to YouLab.

Congratulations on taking the first step in reclaiming your relationship with AI...

Take a moment. Don't give me your LinkedIn bio. Tell me about a time recently
when you felt fully alive - energized, capable, in your element.

What were you doing?
"""
```

New users should receive this message **before** they send anything.

#### Current State in Ralph

**Ralph has NO first message handling:**

- No new user detection (`src/ralph/server.py:167-386`)
- No automatic greeting (agent waits for user message)
- Memory blocks load empty if user has none (`src/ralph/memory.py:34-35`)
- No initialization flow for new users

**Indirect "newness" signals exist but aren't used:**

| Signal | Location | Currently Used? |
|--------|----------|-----------------|
| Empty memory blocks | `memory.py:34-35` | No, returns empty string |
| No Honcho history | `honcho_tools.py:80-82` | Returns "No insights available" |
| Missing workspace | `server.py:55-56` | Creates silently |
| No user_activity row | `dolt.py:801-812` | Creates on first message |

#### Gap: First Message System

Ralph needs:
1. New user detection (check if user has blocks/history)
2. First message trigger (before user sends message)
3. Course-specific welcome content (from `[first_message]` in TOML)
4. OpenWebUI integration (how to push message to UI?)

### 4. Memory Block Templates

#### What welcome.toml Expects

Four memory blocks with rich templates:

```toml
[[block]]
label = "origin_story"
title = "Origin Story"
template = """
## Who I Am At My Best

[Moments when they feel most alive, capable, energized]

## What I'm Building Toward

[6-12 month vision, concrete goals, why these matter]

## My Superpowers

[Natural strengths, what comes easily, what others come to them for]

## My Kryptonite

[What drains them, patterns they fight against, blind spots]
"""
```

Templates have structured sections with placeholder guidance.

#### Current State in Ralph

**Memory blocks exist but without templates:**

- Dolt schema: `user_id`, `label`, `title`, `body`, `schema_ref`, `updated_at`
- `schema_ref` field exists but unused (intended for template reference)
- Blocks created empty when agent proposes edits
- No template initialization on new user

**Block operations work:**

- CRUD via `DoltClient` (`src/ralph/dolt.py:109-196`)
- Agent tools via `MemoryBlockTools` (`src/ralph/tools/memory_blocks.py`)
- REST API via `/users/{user_id}/blocks/*` (`src/ralph/api/blocks.py`)
- Proposal workflow via branch-based diffs

#### Gap: Template Initialization

Ralph needs:
1. Block schema registry (load `[[block]]` definitions from TOML)
2. Template instantiation (create blocks from templates for new users)
3. Schema validation (optional: validate body against field definitions)
4. Course-specific blocks (different courses define different blocks)

### 5. Background Task System

#### What welcome.toml Expects

Two background tasks:

```toml
[[task]]
name = "gentle-return"
trigger = { type = "idle", idle_minutes = 4320, cooldown_minutes = 10080 }  # 3 days, 1 week cooldown
system_prompt = "This person started onboarding but hasn't returned..."
tools = ["honcho_tools", "memory_blocks"]
blocks = ["origin_story", "onboarding_progress"]

[[task]]
name = "graduation-reflection"
trigger = { type = "cron", schedule = "0 10 * * 1" }  # Monday 10 AM
system_prompt = "For users who have completed onboarding..."
tools = ["memory_blocks", "file_tools"]
blocks = ["origin_story", "tech_relationship", "ai_partnership"]
```

#### Current State in Ralph

**Background task infrastructure EXISTS and is functional:**

| Component | Location | Status |
|-----------|----------|--------|
| Task models | `src/ralph/background/models.py:36-94` | Complete |
| Task registry | `src/ralph/background/registry.py:17-137` | Complete |
| Task executor | `src/ralph/background/executor.py:31-207` | Complete |
| Scheduler | `src/ralph/background/scheduler.py:26-161` | Complete |
| Dolt persistence | `src/ralph/dolt.py:564-883` | Complete |
| HTTP API | `src/ralph/api/background.py:24-303` | Complete |

**Trigger types supported:**

- `CronTrigger` with cron expressions
- `IdleTrigger` with idle_minutes and cooldown_minutes

**Tool factory has gaps:**

Current factory (`src/ralph/background/tools.py:35-59`) only supports:
- `"file_tools"` → FileTools
- `"shell_tools"` → ShellTools

Missing from factory:
- `"honcho_tools"` → HonchoTools
- `"memory_blocks"` → MemoryBlockTools
- `"latex_tools"` → LaTeXTools

#### Gap: Task Definition Loading

Ralph needs:
1. Load `[[task]]` definitions from course TOML files
2. Register tasks in registry on course load
3. Add missing tools to tool factory
4. Pass memory block labels to executor for context building

### 6. Agent Selection / Course Routing

#### What welcome.toml Expects

The Welcome agent is a distinct agent with:
- Specific system prompt (onboarding philosophy)
- Specific tools (memory_blocks, honcho_tools, file_tools)
- Specific blocks (origin_story, tech_relationship, ai_partnership, progress)
- Unique behavior (three-phase progression)

#### Current State in Ralph

**Ralph has ONE hardcoded agent for ALL users:**

```python
# src/ralph/server.py:283-297
agent = Agent(
    model=OpenRouter(id=settings.openrouter_model, ...),
    tools=[
        ShellTools(base_dir=workspace),
        FileTools(base_dir=workspace),
        HonchoTools(),
        MemoryBlockTools(),
        LaTeXTools(workspace=workspace),
    ],
    instructions=instructions,  # Same base instructions for everyone
    markdown=True,
)
```

**No course/agent selection:**

- No course_id in request
- No agent type routing
- No per-course configuration

**Legacy had course routing:**

- User initialization: `src/youlab_server/server/users.py:122-131`
- Agent creation: `src/youlab_server/server/agents.py:220-299`
- Course lookup: `curriculum.get(request.course_id)`

#### Gap: Course-Based Agent Factory

Ralph needs:
1. Course ID in chat request (from OpenWebUI metadata or routing)
2. Course configuration lookup (load from TOML)
3. Agent factory that builds from course config
4. Per-course tool selection, system prompt, and block schemas

## Architecture Documentation

### Current Ralph Request Flow

```
OpenWebUI User
      ↓
┌─────────────────┐
│   ralph/pipe.py │  Extracts user_id, chat_id
│   (HTTP client) │  POSTs to /chat/stream
└────────┬────────┘
         ↓
┌─────────────────┐
│ ralph/server.py │  Builds agent per-request:
│  /chat/stream   │  1. Get workspace path
│                 │  2. Load CLAUDE.md (if exists)
│                 │  3. Load memory blocks from Dolt
│                 │  4. Build base instructions
│                 │  5. Create Agno agent with tools
│                 │  6. Stream response
└────────┬────────┘
         ↓
┌─────────────────┐
│  Agno Agent     │  Executes with tools:
│                 │  - ShellTools, FileTools
│                 │  - HonchoTools, MemoryBlockTools
│                 │  - LaTeXTools
└─────────────────┘
```

### Proposed Course-Aware Flow

```
OpenWebUI User
      ↓
┌─────────────────┐
│   ralph/pipe.py │  Extracts user_id, chat_id, course_id
└────────┬────────┘
         ↓
┌─────────────────────────────────────────┐
│ Course Router (NEW)                      │
│  1. Is this a new user for this course? │
│     → Initialize blocks from templates   │
│     → Send first_message content         │
│  2. Load course config from TOML         │
│  3. Determine user's current phase       │
└────────┬────────────────────────────────┘
         ↓
┌─────────────────────────────────────────┐
│ Agent Factory (NEW)                      │
│  1. Select system prompt for phase       │
│  2. Select tools from course config      │
│  3. Load memory blocks (filtered by      │
│     course.blocks list)                  │
│  4. Create configured Agno agent         │
└────────┬────────────────────────────────┘
         ↓
┌─────────────────┐
│  Agno Agent     │  Course-specific execution
└─────────────────┘
```

## Code References

### Ralph Backend (Current State)

- `src/ralph/server.py:167-386` - Chat stream endpoint (no course awareness)
- `src/ralph/server.py:187-245` - Hardcoded base instructions
- `src/ralph/server.py:283-297` - Hardcoded agent creation
- `src/ralph/memory.py:11-44` - Generic block loading (no templates)
- `src/ralph/dolt.py:109-196` - Block CRUD (schema_ref unused)
- `src/ralph/background/tools.py:35-59` - Tool factory (missing honcho/memory)

### Legacy YouLab Server (Has Course System)

- `src/youlab_server/curriculum/loader.py:43-457` - TOML course loading
- `src/youlab_server/curriculum/schema.py:1-361` - Pydantic schemas
- `src/youlab_server/tools/curriculum.py:27-326` - Lesson advancement
- `src/youlab_server/server/users.py:122-131` - Course-based user init
- `src/youlab_server/server/agents.py:220-299` - Course-based agent creation

### Existing TOML Examples

- `config/courses/college-essay/course.toml` - v2 schema example
- `config/courses/college-essay/modules/*.toml` - Module definitions
- `config/courses/default/course.toml` - Simple fallback course

## Historical Context (from thoughts/)

### Architecture Decisions

- `thoughts/shared/research/2026-01-24-youlab-v2-greenfield.md` - Core philosophy for modules
- `thoughts/shared/research/2026-01-28-ralph-agno-architecture.md` - Ralph architecture design
- `thoughts/shared/plans/2026-01-26-ralph-wiggum-mvp.md` - Current implementation phases

### TOML Schema Research

- `thoughts/shared/research/ari-85-config-schema-research.md` - V1 vs V2 schema analysis
- `thoughts/shared/research/2026-01-13-ARI-82-course-module-config.md` - Course configuration
- `thoughts/shared/plans/2026-01-08-unified-toml-curriculum-schema.md` - Schema unification plan

### Onboarding Research

- `thoughts/shared/research/2025-01-12-ARI-78-user-onboarding-initialization.md` - Onboarding flow analysis

## Implementation Priorities

Based on the welcome.toml requirements, here's a suggested implementation order:

### Priority 1: TOML Configuration Loading

**Why first**: Everything else depends on being able to load course configuration.

**Components**:
1. Port `CurriculumLoader` to Ralph or create new loader
2. Add course TOML file for Welcome agent
3. Create agent factory that builds from config

### Priority 2: Block Template Initialization

**Why second**: Users need blocks initialized before agent can edit them.

**Components**:
1. Block schema registry from course config
2. Template initialization on new user detection
3. Schema_ref population for tracking source

### Priority 3: Module/Phase Progression

**Why third**: Core value of welcome course is phased onboarding.

**Components**:
1. Phase tracking (in progress block or separate state)
2. Phase-aware prompt composition
3. Phase advancement detection

### Priority 4: First Message Trigger

**Why fourth**: Needs block templates ready; enhances UX but not blocking.

**Components**:
1. New user detection logic
2. First message injection mechanism
3. OpenWebUI integration for push messaging

### Priority 5: Background Tasks Loading

**Why fifth**: Existing infrastructure works; just needs TOML loading.

**Components**:
1. Load `[[task]]` from course TOML
2. Register in task registry
3. Add missing tools to factory

## Open Questions

1. **Course routing mechanism**: How does OpenWebUI tell Ralph which course the user is in? URL path? Metadata? Model selection?

2. **First message delivery**: How to push first_message to OpenWebUI before user sends anything? Webhook? SSE event? Database injection?

3. **Phase persistence**: Should phase progress live in a memory block (onboarding_progress) or a separate database table?

4. **TOML location**: Should welcome.toml go in `config/courses/welcome/` following existing pattern?

5. **Shared vs isolated blocks**: Welcome course blocks (origin_story, etc.) - should they be visible to other courses after graduation?

6. **Migration strategy**: Port legacy loader or rewrite? Legacy has v1/v2 schema complexity that may not be needed.
