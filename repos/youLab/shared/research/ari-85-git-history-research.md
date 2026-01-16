# Git History Research: ARI-85 Background Agent POC

**Research Date**: 2026-01-16
**Researcher**: Claude Sonnet 4.5
**Ticket**: ARI-85 - Create proof-of-concept single-module course with background agent

## Executive Summary

The past two weeks have seen major infrastructure development across three areas:
1. **TOML-based configuration system** replacing hardcoded agents
2. **Git-backed user memory** with diff approval workflow
3. **Module-based UI** in OpenWebUI with course navigation

The specs have evolved from TOML-only configuration to a hybrid system where modules can include agent-specific system prompt overrides, and the UI now supports step-by-step course navigation with metadata.

## Timeline of Major Changes

### Week of Jan 6-12, 2026

| Date | Commit | Impact |
|------|--------|--------|
| Jan 8 | `cd3216b` | Background agents infrastructure (Phases 1-5) |
| Jan 9 | `47ca115` | Config schema v2 - consolidated [agent] section, [[task]] array |
| Jan 10 | `8424e76` | Curriculum v2 - "lesson" → "step" terminology, advance_lesson tool |
| Jan 12 | `7e5649b` | Memory System MVP - git storage, TOML↔MD conversion, diff approval |
| Jan 12 | `9c5639a` | Module metadata schema documentation |

### Week of Jan 13-16, 2026

| Date | Commit | Impact |
|------|--------|--------|
| Jan 11 | `2611ba2` | Diff approval backend and storage layer refactor |
| Recent | `81c10f9fd` (OpenWebUI) | YouLab branding, theme, modules UI |
| Recent | `b3e763d2d` (OpenWebUI) | Block diff approval UI, memory editing |

## 1. Configuration Evolution: From Monolithic to Modular

### v1 Schema (deprecated)
- Separate `[course]` and `[agent]` sections
- `[background.name]` for each background agent
- Hard to maintain, verbose

### v2 Schema (current)
```toml
[agent]
id = "college-essay"
model = "openai/gpt-4o"
system = """..."""
tools = ["send_message", "edit_memory_block", "advance_lesson"]

[block.student]
label = "human"
field.profile = { type = "string", default = "", ... }

[[task]]
schedule = "0 3 * * *"
queries = [...]
```

**Key improvements**:
- Merged `[course]` and `[agent]` into single `[agent]` section
- `[block.x]` with `field.*` dotted-key syntax for memory blocks
- `[[task]]` array for background agents (replacing `[background.name]`)
- Tool registry with default rules

### Module-Level Configuration (NEW)

Modules can now override agent behavior:

```toml
# config/courses/college-essay/modules/01-first-impression.toml
[module]
id = "01-first-impression"
disabled_tools = ["query_honcho"]  # No dialectic in Module 1

[[steps]]
id = "strengths-upload"
[steps.agent]
opening = """Welcome to YouLab!..."""
focus = ["rapport", "clifton_strengths"]
guidance = ["Be warm...", "If no PDF..."]
disabled_tools = ["advance_lesson"]  # Can't skip this step
```

**Novel pattern**: `steps.agent.guidance` is an array of coaching instructions that supplements the main system prompt. This allows step-specific behavior without duplicating the full system prompt.

## 2. Memory System: Git-Backed with Approval Workflow

### Architecture

```
.data/users/{user_id}/
    .git/                    # Full version history
    blocks/
        student.toml         # User profile
        journey.toml         # Curriculum state
        engagement_strategy.toml
    pending_diffs/
        {diff_id}.json       # Agent proposals awaiting approval
```

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| `GitUserStorage` | `storage/git.py` | Per-user git repos |
| `UserBlockManager` | `storage/blocks.py` | Block CRUD operations |
| `PendingDiff` | `storage/diffs.py` | Diff proposal system |
| `toml_to_markdown()` | `storage/convert.py` | Bidirectional conversion |

### Diff Approval Workflow

1. Agent calls `edit_memory_block` tool
2. `PendingDiff` created (not applied)
3. User reviews in UI → approve/reject
4. If approved → git commit created
5. Older diffs for same block superseded

**Critical insight**: This is a **human-in-the-loop** system, not automatic memory updates. Aligns with YouLab's philosophy of student control.

### TOML ↔ Markdown Conversion

**Storage format (TOML)**:
```toml
name = "Alice"
strengths = ["Creativity", "Communication"]
```

**Editing format (Markdown)**:
```markdown
---
block: student
---

## Name
Alice

## Strengths
- Creativity
- Communication
```

Conversion is **bidirectional** and preserves structure.

## 3. Background Agents Infrastructure

### Current Implementation

Background agents query Honcho (theory-of-mind service) to enrich memory:

```toml
[[task]]
schedule = "0 3 * * *"  # Daily at 3 AM
agent_types = ["college-essay"]
user_filter = "all"
batch_size = 50

queries = [
    {
        target = "human.context_notes",
        question = "What learning style works best for this student?",
        scope = "all",
        merge = "append"
    }
]
```

### Components Added (cd3216b)

| Component | File | Purpose |
|-----------|------|---------|
| `BackgroundAgentRunner` | `background/runner.py` | Scheduled execution |
| `CourseConfig` schema | `background/schema.py` | Pydantic validation |
| `query_dialectic()` | `honcho/client.py` | Theory-of-mind queries |
| `MemoryEnricher` | `memory/enricher.py` | Audit trail to archival |

### HTTP Endpoints

- `GET /background/agents` - list configured agents
- `POST /background/{agent_id}/run` - manual trigger
- `POST /background/config/reload` - hot reload TOML

**Status**: Infrastructure exists, but **not yet integrated with new git-backed memory system**. Background agents currently update old `MemoryManager`, not `UserBlockManager`.

## 4. OpenWebUI Frontend Changes

### Branding & Theme (81c10f9fd)

- **App name**: "Open WebUI" → "YouLab"
- **Font**: iA Writer Duo (monospace feel)
- **Themes**: YouLab Dark/Light with custom colors
- **Hidden sections**: Workspace, Controls (via `youlabMode` store)

### Module Navigation (81c10f9fd)

New sidebar components:
- `ModuleItem.svelte` - individual module with status badge
- `ModuleList.svelte` - course-based grouping

Metadata schema:
```typescript
interface YouLabModuleMeta {
  course_id: string;
  module_index: number;
  status?: 'locked' | 'available' | 'in_progress' | 'completed';
  welcome_message?: string;
}
```

### Memory Editing UI (b3e763d2d)

New components:
- `DiffApprovalOverlay.svelte` - inline diff view
- `BlockEditor.svelte` - TipTap-based Markdown editor
- `MemoryBlockEditor.svelte` - WYSIWYG block editing
- `/workspace/profile` page - user memory dashboard

**Flow**:
1. User clicks "Edit" on memory block
2. Markdown editor opens (with syntax highlighting)
3. User saves → `PUT /users/{id}/blocks/{label}`
4. Git commit created automatically

**Diff approval flow**:
1. Agent proposes change
2. Badge appears on block card
3. User clicks → diff overlay shows
4. Approve → `POST /users/{id}/blocks/{label}/diffs/{id}/approve`
5. Reject → `POST /users/{id}/blocks/{label}/diffs/{id}/reject`

## 5. Current Course Structure

### College Essay Course v2.0

```
config/courses/college-essay/
    course.toml              # Main agent config
    modules/
        01-first-impression.toml
        01-self-discovery.toml (duplicate - needs cleanup)
        02-topic-development.toml
        03-drafting.toml
```

**Note**: Two versions of Module 1 exist - `01-first-impression.toml` (newer, CliftonStrengths-first) and `01-self-discovery.toml` (older, generic). The course.toml references `"01-first-impression"` in the modules array.

### Module 1: First Impression (NEW)

3 steps:
1. **strengths-upload** - Welcome, get PDF, analyze thoroughly
2. **strengths-deepdive** - Explore top strength through stories
3. **strengths-bridge** - Connect to potential essay themes

**Novel features**:
- `disabled_tools = ["query_honcho"]` at module level (no dialectic in first module)
- `disabled_tools = ["advance_lesson"]` at step level (can't skip PDF upload)
- `steps.agent.guidance` - array of coaching instructions
- `steps.completion.min_turns` - minimum conversation turns required

## 6. Migration from TOML to MD (NOT happening)

**Finding**: Despite my initial search directive, there is **no evidence** of a migration from TOML to Markdown for configuration.

What's actually happening:
- **Configuration**: Still TOML (`course.toml`, `modules/*.toml`)
- **User memory**: TOML storage, Markdown editing (bidirectional conversion)
- **No plans** to migrate config to Markdown

The Markdown component is purely for **user-facing memory editing**, not system configuration.

## 7. Gaps and Next Steps for ARI-85

### What's Ready
✅ Module TOML schema with step-by-step structure
✅ Agent-specific prompts via `steps.agent.opening` and `guidance`
✅ Tool disabling per module/step
✅ Git-backed user memory with version history
✅ Diff approval UI in OpenWebUI
✅ Module navigation UI with status badges

### What's Missing
❌ Background agent integration with `UserBlockManager` (still uses deprecated `MemoryManager`)
❌ Automatic module status updates based on `advance_lesson` calls
❌ Connection between OpenWebUI module metadata and backend curriculum state
❌ Progression-grader background task (mentioned in 8424e76 but not implemented)

### What Needs Cleanup
⚠️ Duplicate Module 1 files (`01-first-impression.toml` vs `01-self-discovery.toml`)
⚠️ Deprecated code still in codebase (emits `DeprecationWarning`)
⚠️ Background agent TOML format changed but old configs may exist

## 8. Recommendations for ARI-85

### Approach 1: Single-Module POC (Recommended)
Create a minimal course with one module to test the full stack:

```
config/courses/background-test/
    course.toml              # Minimal agent with 1 block
    modules/
        01-test.toml         # 1 step with simple objective
```

**Test flow**:
1. User completes step
2. Agent calls `advance_lesson`
3. Background agent runs (manual trigger)
4. Proposes memory update via `edit_memory_block`
5. User approves diff in UI
6. Verify git commit created

**Benefits**:
- Tests entire pipeline without complexity
- Validates background agent → `UserBlockManager` integration
- Proves diff approval workflow end-to-end

### Approach 2: Add Background Agent to Existing Module
Add a background task to `01-first-impression`:

```toml
[[task]]
schedule = "0 3 * * *"
manual = true
queries = [
    {
        target = "student.insights",
        question = "Based on CliftonStrengths discussion, what essay themes are emerging?",
        scope = "recent",
        merge = "append"
    }
]
```

**Benefits**:
- Realistic use case
- Tests background enrichment of actual student data

**Risks**:
- More complex debugging if something breaks
- Hard to isolate issues

## 9. Key Architectural Decisions

### Decision 1: Human-in-the-Loop Memory
**Choice**: Agent proposes changes via diffs, user approves
**Rationale**: Students control their own narrative
**Impact**: Requires UI for diff approval (now implemented)

### Decision 2: Git-Backed Storage
**Choice**: Each user gets a git repo under `.data/users/{id}`
**Rationale**: Full version history, easy rollback, audit trail
**Impact**: All block changes create commits, browsable history

### Decision 3: TOML Storage, Markdown Editing
**Choice**: Store as TOML, convert to Markdown for editing
**Rationale**: TOML is structured (machine-readable), Markdown is familiar (human-readable)
**Impact**: Bidirectional conversion required, potential for round-trip bugs

### Decision 4: Module-Level Agent Overrides
**Choice**: Allow `steps.agent.guidance` to supplement system prompt
**Rationale**: Avoid duplicating full prompt per step
**Impact**: Prompt composition logic in curriculum loader

## 10. Code Locations Reference

| Feature | Backend | Frontend | Tests |
|---------|---------|----------|-------|
| Git storage | `storage/git.py` | - | `test_storage/` |
| User blocks | `storage/blocks.py` | - | `test_storage/test_blocks.py` |
| Pending diffs | `storage/diffs.py` | - | `test_storage/test_diffs.py` |
| TOML↔MD | `storage/convert.py` | - | `test_storage/test_convert.py` |
| Block API | `server/blocks.py` | `apis/memory/notes.ts` | `test_server/test_blocks_api.py` |
| User init | `server/users.py` | - | `test_server/test_users.py` |
| Background agents | `background/runner.py` | - | `test_background_runner.py` |
| Module nav | - | `layout/Sidebar/ModuleList.svelte` | - |
| Diff approval | - | `you/DiffApprovalOverlay.svelte` | - |
| Block editor | - | `you/BlockEditor.svelte` | - |

## 11. Breaking Changes Log

### 2026-01-09: Config Schema v2 (47ca115)
- Merged `[course]` and `[agent]` sections
- Changed `[background.name]` to `[[task]]` array
- Added `[block.x]` with `field.*` syntax
- **Migration**: Old TOML configs won't load

### 2026-01-10: "Lesson" → "Step" (8424e76)
- Renamed terminology throughout
- Changed tool name `advance_lesson` (kept old name for now)
- **Migration**: Docs updated, code compatible

### 2026-01-12: Memory System MVP (7e5649b)
- Deprecated `MemoryManager`, `PersonaBlock`, `HumanBlock`
- New storage layer with `UserBlockManager`
- **Migration**: Old agent path still works (with warnings)

## 12. Open Questions

1. **How should background agents integrate with `UserBlockManager`?**
   - Current: Uses deprecated `MemoryEnricher` → `MemoryManager`
   - Needed: Background agent → `UserBlockManager.create_pending_diff()`

2. **Should background agent changes require approval?**
   - Pro: Consistent with agent-driven edits
   - Con: User might not understand why they're approving
   - Middle ground: Auto-approve background enrichment, manual approve agent edits?

3. **How does OpenWebUI module status sync with backend?**
   - Frontend: `youlab_module.status` in model metadata
   - Backend: `journey.current_module` and `journey.current_step` in TOML
   - Missing: API endpoint to update module status on `advance_lesson`

4. **What triggers progression-grader task?**
   - Mentioned in curriculum v2 commit
   - Not found in codebase
   - Should it run after each `advance_lesson` call?

## Conclusion

The infrastructure for a background agent POC is **mostly ready**:
- ✅ Module/step configuration system
- ✅ Git-backed user memory
- ✅ Diff approval UI
- ✅ Background agent scheduling

The main gap is **connecting background agents to the new memory system**. They still use deprecated `MemoryManager` instead of `UserBlockManager.create_pending_diff()`.

**Recommendation**: Start with Approach 1 (minimal single-module course) to validate the integration, then apply to real college-essay course.

---

**Research complete.** Report saved to: `thoughts/shared/research/ari-85-git-history-research.md`
