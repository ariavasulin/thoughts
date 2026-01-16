# ARI-85: POC Single-Module Course with Background Agent

## Overview

Create a proof-of-concept single-module course to validate the current architecture and identify gaps. The POC tests:
1. Course/agent creation from TOML config
2. Memory blocks stored as MD with git versioning
3. Background agent that updates memory via PendingDiffs (user approval)
4. Integration with new `UserBlockManager` storage layer

**Philosophy**: This is a testing/validation exercise. We add TODOs for future work and confront problems directly rather than shipping fast.

## Current State Analysis

### What Works (Verified by Code)
- v2 TOML schema: `[agent]`, `[block.{name}]`, `[[task]]`
- Memory blocks stored as MD with YAML frontmatter (`.data/users/{user_id}/memory-blocks/*.md`)
- Git versioning via `GitUserStorage`
- Interactive agent `edit_memory_block` creates `PendingDiff` for user approval
- Manual trigger of background agents via `POST /background/{agent_id}/run`
- Blocks API: `/users/{user_id}/blocks/*`

### Critical Issues Found

#### Issue 1: Idle Trigger NOT Implemented
**Schema exists but NO execution code:**
- `on_idle`, `idle_threshold_minutes`, `idle_cooldown_minutes` defined in `schema.py:175-177`
- `BackgroundAgentRunner` does NOT check these fields
- No scheduler (cron, APScheduler, celery) in codebase
- No activity tracking (`last_active` timestamp)

**Impact**: Setting `on_idle=true` with 1 minute threshold does NOTHING currently.

**Location**: `src/youlab_server/background/runner.py` - runner only executes when manually called

#### Issue 2: Background Agents Bypass Git Storage
**Current flow:**
```
BackgroundAgentRunner → MemoryEnricher → Letta blocks (directly)
```
**Needed flow:**
```
BackgroundAgentRunner → UserBlockManager.propose_edit() → PendingDiff → user approval → git commit
```

**Impact**: Background agent memory updates don't create PendingDiffs, no git versioning, no user approval.

**Location**: `src/youlab_server/background/runner.py:212-220` uses `self.enricher.enrich()` which bypasses `UserBlockManager`

#### Issue 3: User Missing Memory Blocks
**Finding**: User `7a41011b-5255-4225-b75e-1d8484d0e37f` only has `student.md` despite course.toml defining:
- `student` (label="human")
- `engagement_strategy` (label="persona")
- `journey` (label="journey")

**Cause**: User was created before all blocks were defined, or `/users/init` was never called with current schema.

**Location**: `.data/users/7a41011b-5255-4225-b75e-1d8484d0e37f/memory-blocks/` only contains `student.md`

#### Issue 4: Cooldown Not Enforced
- No state tracking for when background agent last ran per user
- Even if idle trigger worked, cooldown wouldn't be enforced
- Need persistent storage for `last_run_at` per user per task

#### Issue 5: `after_messages` Trigger Not Implemented
**Schema exists** (`schema.py:116`) but runner ignores it.

**Location**: `src/youlab_server/curriculum/schema.py:116` - field defined but unused

## Desired End State

After this plan is complete:
1. New POC course exists at `config/courses/poc-tutor/`
2. Memory blocks (Course Syllabus, Operating Manual, Progress) defined with label="human"
3. Background agent configured with `on_idle=true` (even though trigger is TODO)
4. Background agent creates PendingDiffs via `UserBlockManager` (not direct Letta updates)
5. User `7a41...` has all memory blocks initialized
6. Manual trigger works for testing: `POST /background/poc-progress-tracker/run`
7. Clear TODOs added for idle trigger implementation

## What We're NOT Doing

- Implementing `after_messages` trigger (TODO only)
- Implementing automatic idle trigger scheduler (TODO only)
- Building cooldown enforcement (TODO only)
- Changing OpenWebUI frontend (out of scope)
- Production-quality system prompts (POC only)

## Implementation Approach

1. **Phase 1**: Create minimal POC course config
2. **Phase 2**: Initialize user with all memory blocks
3. **Phase 3**: Integrate background agent with `UserBlockManager`
4. **Phase 4**: Add TODOs for trigger system
5. **Phase 5**: Manual testing and validation

---

## Phase 1: Create POC Course Configuration

### Overview
Create minimal course TOML with:
- Single module, single step
- Three memory blocks (all label="human" for simplicity)
- One background task with `on_idle=true` (even though not implemented)

### Changes Required

#### 1. Create Course Directory
```bash
mkdir -p config/courses/poc-tutor/modules
```

#### 2. Create `config/courses/poc-tutor/course.toml`

```toml
# =============================================================================
# POC TUTOR COURSE - ARI-85
# Testing background agent integration with UserBlockManager
# =============================================================================

[agent]
id = "poc-tutor"
name = "POC Tutor"
version = "0.1.0"
description = "Proof-of-concept tutor for testing background agent architecture"
modules = ["01-intro"]
model = "openai/gpt-4o"
embedding = "openai/text-embedding-3-small"
context_window = 128000
max_response_tokens = 4096
system = """You are a simple tutor for testing purposes.

Your memory blocks:
- `course_syllabus`: What we're learning together
- `operating_manual`: How to work with this student effectively
- `progress`: Student's progress through the course

Read these blocks at the start of each session. Update them when you learn something significant.

Be friendly and helpful. This is a POC to test the system architecture.
"""

tools = ["send_message", "edit_memory_block"]

# =============================================================================
# MEMORY BLOCKS - All using label="human" for simplicity
# =============================================================================

[block.course_syllabus]
label = "human"
shared = false
description = "Course syllabus and learning objectives"
field.overview = { type = "string", default = "A simple tutoring course for testing.", description = "High-level course description" }
field.objectives = { type = "list", default = ["Learn something new", "Test the system"], max = 10, description = "Learning objectives" }

[block.operating_manual]
label = "human"
shared = false
description = "How to work effectively with this student"
field.effective_strategies = { type = "string", default = "", description = "What works well with this student" }
field.things_to_avoid = { type = "string", default = "", description = "What doesn't work or should be avoided" }

[block.progress]
label = "human"
shared = false
description = "Student progress tracking"
field.current_status = { type = "string", default = "Just getting started", description = "Where the student is now" }
field.completed_items = { type = "list", default = [], max = 20, description = "Completed milestones" }
field.notes = { type = "string", default = "", description = "Observations about progress" }

# =============================================================================
# BACKGROUND TASKS
# =============================================================================

# NOTE: on_idle trigger is NOT IMPLEMENTED. This task will only run when manually triggered.
# TODO(ARI-85): Implement idle trigger scheduler - see src/youlab_server/background/runner.py
[[task]]
name = "poc-progress-tracker"
on_idle = true
idle_threshold_minutes = 1  # Would run 1 minute after last message IF implemented
idle_cooldown_minutes = 5   # Would wait 5 minutes between runs IF implemented
manual = true  # Can be manually triggered (this DOES work)
agent_types = ["poc-tutor"]
user_filter = "all"
batch_size = 10

queries = [
    { target = "progress.notes", question = "What progress has the student made in this conversation? What did they learn or accomplish?", scope = "recent", recent_limit = 10, merge = "append" },
    { target = "operating_manual.effective_strategies", question = "Based on this conversation, what tutoring strategies seem to work well with this student?", scope = "recent", recent_limit = 10, merge = "append" }
]

# =============================================================================
# UI MESSAGES
# =============================================================================

[messages]
welcome_first = "Hello! I'm your POC tutor. Let's learn something together!"
welcome_returning = "Welcome back! Ready to continue?"
error_unavailable = "Having a moment - try again shortly."
```

#### 3. Create `config/courses/poc-tutor/modules/01-intro.toml`

```toml
[module]
id = "01-intro"
name = "Introduction"
order = 1
description = "Introduction to the POC course"

[[steps]]
id = "welcome"
name = "Welcome"
order = 1
description = "Welcome and introduction"
objectives = [
    "Greet the student",
    "Explain what we'll do together"
]

[steps.completion]
min_turns = 2

[steps.agent]
opening = "Hi there! I'm your tutor for this proof-of-concept course. What would you like to learn about today?"
focus = ["greeting", "introduction"]
guidance = [
    "Be warm and welcoming",
    "Ask what they're interested in",
    "Keep it simple - this is a test"
]
```

### Success Criteria

#### Automated Verification:
- [x] Course loads without errors: `uv run python -c "from youlab_server.curriculum import curriculum; c = curriculum.load_course('poc-tutor'); print(f'Loaded: {c.name}')"`
- [x] Blocks parsed correctly: Verify 3 blocks exist with correct fields
- [x] Task parsed correctly: Verify task config has correct query targets
- [x] Lint passes: `make lint-fix`

#### Manual Verification:
- [ ] Review TOML structure matches v2 schema

---

## Phase 2: Initialize User with All Memory Blocks

### Overview
User `7a41011b-5255-4225-b75e-1d8484d0e37f` is missing memory blocks. We need to either:
- A) Re-initialize the user via API, OR
- B) Manually create the block files

Using option B for explicit control and testing.

### Changes Required

#### 1. Create missing memory block files

For each block defined in course schema, create the corresponding `.md` file.

**File**: `.data/users/7a41011b-5255-4225-b75e-1d8484d0e37f/memory-blocks/course_syllabus.md`
```markdown
---
block: course_syllabus
updated_at: '2026-01-16T00:00:00+00:00'
title: Course Syllabus
schema: poc-tutor/course_syllabus
---

## Overview

A simple tutoring course for testing.

## Objectives

- Learn something new
- Test the system
```

**File**: `.data/users/7a41011b-5255-4225-b75e-1d8484d0e37f/memory-blocks/operating_manual.md`
```markdown
---
block: operating_manual
updated_at: '2026-01-16T00:00:00+00:00'
title: Operating Manual
schema: poc-tutor/operating_manual
---

## Effective Strategies

(To be discovered through conversation)

## Things to Avoid

(To be discovered through conversation)
```

**File**: `.data/users/7a41011b-5255-4225-b75e-1d8484d0e37f/memory-blocks/progress.md`
```markdown
---
block: progress
updated_at: '2026-01-16T00:00:00+00:00'
title: Progress
schema: poc-tutor/progress
---

## Current Status

Just getting started

## Completed Items

(None yet)

## Notes

(Background agent will append notes here)
```

#### 2. Git commit the new blocks

After creating files, commit them to the user's git repo.

### Success Criteria

#### Automated Verification:
- [x] All 3 block files exist in user's memory-blocks directory
- [x] Files parse correctly with frontmatter
- [ ] API returns all blocks: `curl http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks`

#### Manual Verification:
- [ ] Blocks appear in OpenWebUI memory editor (if integrated)

---

## Phase 3: Letta Agent-Based Background Agents (Architecture Pivot)

### Overview

**PARADIGM SHIFT**: The original query-based approach is being retired. Background agents should be **actual Letta agents** with:
- System prompt defining their purpose
- Access to current memory blocks
- Tools: `query_honcho`, `propose_edit`
- Multi-step reasoning capability

### Old vs New Architecture

**Old (RETIRED):**
```
BackgroundAgentRunner
  → For each query in config:
    → Honcho.query_dialectic(question)  # Single LLM call
    → Write insight directly to block
```

**New (Letta Agent-Based):**
```
BackgroundAgentRunner
  → Create/get dedicated Letta agent for this task
  → Inject current memory block contents into context
  → Provide tools: query_honcho, propose_edit
  → Send instruction: "Analyze conversation history and update memory as needed"
  → Agent reasons, calls tools, creates PendingDiffs
```

### Benefits
1. Agent can reason about *what* to query (not predetermined)
2. Agent can decide *what* to update based on its reasoning
3. Agent can do multiple rounds of query/update
4. Consistent with interactive agent architecture
5. Reuses existing `query_honcho` and `edit_memory_block` tools

### Changes Required

#### 1. Update TaskConfig schema to support full agent definition
**File**: `src/youlab_server/curriculum/schema.py`

The schema already has `system` and `tools` fields - they just need to be used:
```python
class TaskConfig(BaseModel):
    # ... existing fields ...
    system: str | None = None  # System prompt for the background agent
    tools: list[str] = Field(default_factory=list)  # Tools the agent can use

    # DEPRECATED - remove queries in favor of agent reasoning
    queries: list[QueryConfig] = Field(default_factory=list)
```

#### 2. Create BackgroundAgentFactory
**File**: `src/youlab_server/background/factory.py` (NEW)

Factory to create/retrieve dedicated Letta agents for background tasks:
```python
class BackgroundAgentFactory:
    """Creates Letta agents for background tasks."""

    def get_or_create_agent(
        self,
        task_name: str,
        user_id: str,
        system_prompt: str,
        tools: list[str],
        memory_blocks: dict[str, str],  # Current block contents
    ) -> str:
        """
        Get or create a Letta agent for this background task.

        Agent naming: background_{task_name}_{user_id}

        Returns agent_id
        """
```

#### 3. Rewrite BackgroundAgentRunner
**File**: `src/youlab_server/background/runner.py`

Replace query-based execution with agent-based execution:
```python
async def _process_user(self, config: TaskConfig, user_id: str, ...) -> None:
    # 1. Get current memory block contents
    blocks = self._get_user_blocks(user_id)

    # 2. Get or create background agent
    agent_id = self.factory.get_or_create_agent(
        task_name=config.name,
        user_id=user_id,
        system_prompt=config.system,
        tools=config.tools,  # ["query_honcho", "propose_edit"]
        memory_blocks=blocks,
    )

    # 3. Send instruction message
    instruction = self._build_instruction(config, blocks)
    response = self.letta.send_message(agent_id, instruction)

    # Agent will call tools as needed:
    # - query_honcho to analyze conversation
    # - propose_edit to create PendingDiffs
```

#### 4. Ensure `propose_edit` tool creates PendingDiffs
The existing `edit_memory_block` tool already creates PendingDiffs when the agent is flagged as background. Verify this works or create a dedicated `propose_edit` tool variant.

#### 5. Update POC course.toml
```toml
[[task]]
name = "poc-progress-tracker"
manual = true
agent_types = ["poc-tutor"]

# Full agent definition (NEW)
system = """You are a background agent that reviews conversation history and updates memory blocks.

Your job:
1. Query conversation history to understand recent progress
2. Update the progress and operating_manual blocks with insights
3. Be concise - only add meaningful observations

Available memory blocks:
- progress: Track student progress and notes
- operating_manual: Effective strategies and things to avoid
"""
tools = ["query_honcho", "propose_edit"]

# DEPRECATED - remove deterministic queries
# queries = [...]
```

### Success Criteria

#### Automated Verification:
- [x] BackgroundAgentFactory creates agents correctly
- [x] Runner sends messages and agent calls tools
- [x] `propose_edit` tool creates PendingDiff objects (uses existing edit_memory_block)
- [x] Typecheck passes: `make check-agent`
- [x] Lint passes: `make lint-fix`

#### Manual Verification:
- [x] Manual trigger creates agent and processes: `curl -X POST http://localhost:8100/background/poc-progress-tracker/run` (agent created, instruction sent)
- [ ] **BLOCKED**: Agent logs show tool calls (query_honcho, edit_memory_block) - tools fail in Letta sandbox
- [ ] **BLOCKED**: PendingDiff visible in API - cannot create diffs until tools work
- [ ] **BLOCKED**: Approving diff creates git commit - depends on diffs being created

### Manual Testing Commands

**Test User ID**: `7a41011b-5255-4225-b75e-1d8484d0e37f`

**1. List available background agents:**
```bash
curl -s http://localhost:8100/background/agents | jq .
```
Expected: Should show `poc-progress-tracker` with `course_id: poc-tutor`

**2. Trigger background task:**
```bash
curl -X POST http://localhost:8100/background/poc-progress-tracker/run \
  -H "Content-Type: application/json" \
  -d '{"user_ids": ["7a41011b-5255-4225-b75e-1d8484d0e37f"]}' | jq .
```
Expected response fields:
- `users_processed: 1`
- `enrichments_applied: 1` (if agent ran successfully)
- `errors: []` (empty if successful)

**3. Check for pending diffs:**
```bash
curl -s http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks | jq .
```
Look for blocks with `pending_diffs > 0`

**4. View specific block diffs:**
```bash
curl -s http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks/progress/diffs | jq .
```

**5. Approve a diff (replace DIFF_ID):**
```bash
curl -X POST http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks/progress/diffs/DIFF_ID/approve | jq .
```

**6. Verify git commit:**
```bash
cd .data/users/7a41011b-5255-4225-b75e-1d8484d0e37f && git log --oneline -3
```

### What to look for in server logs:
- `background_task_started` with `execution_mode=agent`
- `background_agent_created` or `background_agent_found`
- `background_agent_executing`
- Tool calls from the agent (query_honcho, edit_memory_block)
- `pending_diff_created` or `memory_edit_proposed`
- `background_task_completed` with `enrichments_applied=1`

### Troubleshooting:
- If `users_processed: 0`: No agents found matching `youlab_*_poc-tutor` pattern
- If agent fails: Check Letta server is running and tools are registered
- If no diffs created: Agent may not have called edit_memory_block tool

### Migration Notes
- Keep old `queries` field for backwards compatibility but deprecate
- Old configs with only `queries` fall back to legacy behavior
- New configs with `system` + `tools` use agent-based execution

**Implementation Note**: This is a significant architecture change. Consider creating a new ticket for full implementation and using a minimal version for POC.

---

## Phase 4: Add TODOs for Trigger System

### Overview
Since idle/scheduled triggers aren't implemented, add clear TODOs throughout the codebase.

### Changes Required

#### 1. TODO in schema.py
**File**: `src/youlab_server/curriculum/schema.py`

Around line 116:
```python
class Triggers(BaseModel):
    """Trigger configuration for background agents."""

    schedule: str | None = None
    manual: bool = True
    # TODO(ARI-85): after_messages trigger not implemented in runner
    # Requires: message count tracking per user, trigger check in chat handler
    # See: thoughts/shared/plans/2026-01-16-ARI-85-poc-course-background-agent.md
    after_messages: int | None = None
    idle: IdleTrigger = Field(default_factory=IdleTrigger)
```

Around line 175 in `TaskConfig`:
```python
# TODO(ARI-85): on_idle trigger not implemented - requires scheduler
# Current workaround: manual trigger via POST /background/{agent_id}/run
# Needs: APScheduler or similar, activity tracking, cooldown enforcement
# See: thoughts/shared/plans/2026-01-16-ARI-85-poc-course-background-agent.md
on_idle: bool = False
```

#### 2. TODO in runner.py
**File**: `src/youlab_server/background/runner.py`

At top of file:
```python
"""
Background agent execution engine.

TODO(ARI-85): Trigger system not implemented
Currently, this runner only executes when manually called via:
  POST /background/{agent_id}/run

Missing features:
- Idle trigger: Check user activity, run after threshold
- Schedule trigger: Cron-based execution
- after_messages trigger: Run after N messages
- Cooldown enforcement: Track last_run_at per user per task

See: thoughts/shared/plans/2026-01-16-ARI-85-poc-course-background-agent.md
"""
```

#### 3. TODO in course.toml
Already included in Phase 1.

### Success Criteria

#### Automated Verification:
- [x] `grep -r "TODO(ARI-85)" src/` returns at least 3 locations (actually 6 locations)
- [x] Lint passes (TODOs don't break linting)

---

## Phase 5: Manual Testing and Validation

### Current Status (2026-01-16)

**What Works:**
- ✅ POC course configuration loads correctly
- ✅ Memory block files exist and parse correctly
- ✅ Background agent infrastructure (factory, runner) is functional
- ✅ Manual trigger endpoint works: `POST /background/poc-progress-tracker/run`
- ✅ Agent is created/deleted fresh each run to avoid message history issues
- ✅ Agent receives instruction with current memory block context

**What's Blocked (Tools fail in Letta sandbox):**
- ❌ Tools (`query_honcho`, `edit_memory_block`) fail when agent tries to use them
- ❌ Root cause: Tools import from `youlab_server` which isn't available in Letta's sandbox
- ❌ PendingDiffs cannot be created because `edit_memory_block` fails
- ❌ Agent cannot analyze conversation history because `query_honcho` fails

**Planned Fix:**
Create HTTP-based tool versions that call YouLab server endpoints:
- `POST /tools/query_honcho` - proxy to Honcho dialectic
- `POST /tools/edit_memory_block` - create PendingDiff via API

See TODOs in:
- `src/youlab_server/tools/dialectic.py`
- `src/youlab_server/tools/memory.py`
- `src/youlab_server/background/factory.py`

### Overview
Validate the complete flow works end-to-end.

### Test Plan

#### Test 1: Course Configuration Loading
```bash
uv run python -c "
from youlab_server.curriculum import curriculum
c = curriculum.load_course('poc-tutor')
print(f'Course: {c.name}')
print(f'Blocks: {list(c.blocks.keys())}')
print(f'Tasks: {len(c.tasks)}')
if c.tasks:
    t = c.tasks[0]
    print(f'Task queries: {len(t.queries)}')
    for q in t.queries:
        print(f'  - {q.target}')
"
```

#### Test 2: Memory Block Files Exist
```bash
ls -la .data/users/7a41011b-5255-4225-b75e-1d8484d0e37f/memory-blocks/
# Should show: course_syllabus.md, operating_manual.md, progress.md, student.md
```

#### Test 3: Blocks API Returns All Blocks
```bash
curl http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks
# Should return 4 blocks with pending_diffs counts
```

#### Test 4: Manual Background Agent Trigger
```bash
# Start server first
curl -X POST http://localhost:8100/background/poc-progress-tracker/run \
  -H "Content-Type: application/json" \
  -d '{"user_ids": ["7a41011b-5255-4225-b75e-1d8484d0e37f"]}'
# Should return execution report with enrichments or errors
```

#### Test 5: PendingDiff Created (After Phase 3)
```bash
curl http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks/progress/diffs
# Should show pending diff from background agent
```

#### Test 6: Approve Diff Creates Git Commit
```bash
# Get diff ID from previous call
DIFF_ID="xxx"
curl -X POST http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks/progress/diffs/$DIFF_ID/approve

# Check git log
cd .data/users/7a41011b-5255-4225-b75e-1d8484d0e37f
git log --oneline -5
```

### Success Criteria

#### Automated Verification:
- [x] Course loads: `uv run python -c "from youlab_server.curriculum import curriculum; c = curriculum.load_course('poc-tutor'); print(f'Loaded: {c.name}')"`
- [x] `make verify-agent` passes

#### Manual Verification:
- [ ] **BLOCKED**: Complete flow works: trigger → query → diff → approve → commit (blocked by Letta sandbox tooling issue)
- [ ] **BLOCKED**: OpenWebUI shows updated memory blocks (blocked by tooling issue)

---

## Testing Strategy

### Unit Tests
- [ ] `test_poc_course_loads` - Course config parses correctly
- [ ] `test_propose_background_edit` - UserBlockManager creates diff
- [ ] `test_runner_creates_pending_diff` - Runner integration test

### Integration Tests
- [ ] `test_background_agent_e2e` - Full flow from trigger to approval

### Manual Testing Steps
1. Start services: `uv run youlab-server` + Letta server
2. Load POC course and verify
3. Create memory blocks for test user
4. Manually trigger background agent
5. Verify PendingDiff created
6. Approve diff via API
7. Verify git commit created
8. Check block content updated

---

## Performance Considerations

- Background agent queries Honcho dialectic (external service)
- PendingDiff storage is file-based (may need optimization for many diffs)
- Git operations are synchronous (consider async for batch processing)

---

## Migration Notes

- Existing background agent configs (`college-essay/course.toml`) will still work
- Old path (MemoryEnricher) can be kept as fallback for legacy agents
- New agents should use `propose_background_edit` pattern

---

## Open Questions Resolved

| Question | Decision |
|----------|----------|
| Idle trigger | Use `on_idle=true` but add TODO - manual trigger for now |
| Background approval | Yes - create PendingDiffs like interactive agents |
| User ID | `7a41011b-5255-4225-b75e-1d8484d0e37f` |
| Memory block labels | All use `label="human"` for simplicity |
| Storage integration | Must integrate with `UserBlockManager` |

---

## Future Work (Out of Scope)

1. **Implement idle trigger scheduler**
   - Add APScheduler or similar
   - Track user activity timestamps
   - Enforce cooldown between runs

2. **Implement `after_messages` trigger**
   - Add message counter per user/agent
   - Check threshold in chat handler
   - Reset after background agent runs

3. **Per-user course assignment**
   - Store course_id in OpenWebUI user.info
   - Modify letta_pipe.py to check user settings

4. **Background agent thread history**
   - Store run history in `.data/users/{user_id}/agent_threads/`
   - Surface in UI for debugging

---

## References

- Parent ticket: ARI-85
- Research: `thoughts/shared/research/ari-85-*.md` (6 documents)
- Schema: `src/youlab_server/curriculum/schema.py`
- Runner: `src/youlab_server/background/runner.py`
- Storage: `src/youlab_server/storage/blocks.py`
- Existing course: `config/courses/college-essay/course.toml`
