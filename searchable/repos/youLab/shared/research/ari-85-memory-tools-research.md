# ARI-85 Memory Tools Research: edit_memory_block & Diff Approval System

**Research Date**: 2026-01-16
**Ticket**: ARI-85 - Create proof-of-concept single-module course with background agent
**Focus**: Understanding how agents update memory blocks using the edit_memory_block tool and diff approval system

---

## Executive Summary

The YouLab memory system uses a **propose-approve-apply** pattern for agent memory updates:

1. **Agents propose** changes via `edit_memory_block` tool → creates PendingDiff
2. **Users approve** via API endpoints → applies diff and commits to git
3. **Background agents bypass** approval and apply directly via MemoryEnricher

Key insight: **Two paths exist**:
- **Interactive agents** (tutors) → propose diffs requiring user approval
- **Background agents** → directly apply changes (trusted automation)

---

## Architecture Overview

### Storage Layer (Git-Backed)

**File**: `src/youlab_server/storage/git.py`

Memory blocks are stored as markdown files with YAML frontmatter:

```
.data/users/{user_id}/
  .git/                    # Git repo for version history
  memory-blocks/
    student.md             # Markdown + YAML frontmatter
    progress.md
    operating_manual.md
  pending_diffs/           # Diff approval queue
    {diff_id}.json
  agent_threads/           # Background agent thread history
    {agent_name}/
      {chat_id}.json
```

**Key Operations**:
- `write_block(label, content, message, author, schema, title)` → commits to git
- `read_block(label)` → returns full markdown with frontmatter
- `read_block_body(label)` → returns body only (without frontmatter)
- `read_block_metadata(label)` → returns parsed YAML frontmatter
- `get_block_history(label, limit)` → returns version history from git log

**Frontmatter Structure**:
```yaml
---
block: student
schema: college-essay/student
title: Student Profile
updated_at: 2026-01-16T12:34:56Z
---

Body content here...
```

---

## Agent Memory Update Paths

### Path 1: Interactive Agents (Propose → Approve → Apply)

**Tool**: `src/youlab_server/tools/memory.py::edit_memory_block()`

**Flow**:
```python
# 1. Agent calls tool
edit_memory_block(
    block="student",
    field="learning_style",
    content="Visual learner, responds well to diagrams",
    strategy="append",  # or "replace", "llm_diff", "full_replace"
    reasoning="Student mentioned struggling with text-heavy materials"
)

# 2. Tool creates PendingDiff (does NOT apply)
# Location: src/youlab_server/storage/blocks.py:238-277
manager.propose_edit(
    agent_id=agent_id,
    block_label=block,
    field=field,
    operation=strategy,
    proposed_value=content,
    reasoning=reasoning
)

# 3. Diff saved to: .data/users/{user_id}/pending_diffs/{diff_id}.json

# 4. User approves via API: POST /users/{user_id}/blocks/diffs/{diff_id}/approve

# 5. Approval applies diff and commits
# Location: src/youlab_server/storage/blocks.py:279-335
commit_sha = manager.approve_diff(diff_id)
```

**PendingDiff Schema** (`src/youlab_server/storage/diffs.py`):
```python
@dataclass
class PendingDiff:
    id: str
    user_id: str
    agent_id: str
    block_label: str
    field: str | None
    operation: Literal["append", "replace", "llm_diff", "full_replace"]
    current_value: str        # Snapshot at proposal time
    proposed_value: str       # Agent's proposed content
    reasoning: str            # Why the agent wants this change
    confidence: Literal["low", "medium", "high"]
    source_query: str | None  # Optional context
    status: Literal["pending", "approved", "rejected", "superseded", "expired"]
    created_at: str
    reviewed_at: str | None
    applied_commit: str | None
```

**Operation Types**:
- `append` - Add to end of current content with newline separator
- `replace` - String replace (first occurrence) of current_value → proposed_value
- `full_replace` - Replace entire block body
- `llm_diff` - Reserved for future LLM-powered merge (not yet implemented)

---

### Path 2: Background Agents (Direct Apply, No Approval)

**Service**: `src/youlab_server/memory/enricher.py::MemoryEnricher`

**Flow**:
```python
# Background agent execution (src/youlab_server/background/runner.py)
enricher = MemoryEnricher(letta_client)

# 1. Query Honcho for insights
response = await honcho_client.query_dialectic(
    user_id=user_id,
    question="What progress has the student made this week?",
    session_scope=SessionScope.LAST_WEEK,
    recent_limit=20
)

# 2. Apply enrichment DIRECTLY (bypasses approval)
result = enricher.enrich(
    agent_id=target_agent_id,
    block="progress",
    field="weekly_summary",
    content=response.insight,
    strategy=MergeStrategy.APPEND,
    source="background:progress_tracker",
    source_query="What progress has the student made this week?"
)

# 3. Writes audit entry to Letta archival memory
# Format: [MEMORY_EDIT 2026-01-16T12:34:56Z]
#         Source: background:progress_tracker
#         Block: progress
#         Field: weekly_summary
#         Strategy: append
#         Content: {insight}
```

**Key Difference**: MemoryEnricher bypasses diff approval and writes directly to memory blocks, then logs to Letta's archival memory for transparency.

**Why Two Paths?**:
- **Interactive agents** need user oversight (tutoring decisions)
- **Background agents** are trusted automation (metrics, summaries)

---

## UserBlockManager: Unified API

**File**: `src/youlab_server/storage/blocks.py`

This class provides the unified interface for both paths:

### CRUD Operations
```python
manager = UserBlockManager(user_id, storage, letta_client)

# Read
blocks = manager.list_blocks()
content = manager.get_block_markdown(label)      # Full markdown with frontmatter
body = manager.get_block_body(label)             # Body only
metadata = manager.get_block_metadata(label)     # Parsed YAML

# Write (user or background agent)
commit_sha = manager.update_block(
    label="student",
    content="New content...",
    message="Update student profile",
    author="user",  # or "agent:xyz" or "background:tracker"
    schema="college-essay/student",
    title="Student Profile",
    sync_to_letta=True  # Auto-sync to Letta blocks
)
```

### Pending Diff Management
```python
# Propose (interactive agents)
diff = manager.propose_edit(
    agent_id=agent_id,
    block_label="student",
    field="learning_style",
    operation="append",
    proposed_value="Visual learner",
    reasoning="Mentioned preference for diagrams",
    confidence="medium"
)

# Approve
commit_sha = manager.approve_diff(diff.id)
# - Applies diff operation to current block body
# - Commits to git with message: "Apply agent suggestion: {reasoning}"
# - Marks older diffs for same block as "superseded"
# - Syncs to Letta blocks

# Reject
manager.reject_diff(diff.id, reason="Not accurate")

# List pending
diffs = manager.list_pending_diffs(block_label="student")
counts = manager.count_pending_diffs()  # {"student": 2, "progress": 1}
```

### Version History
```python
# Get history
versions = manager.get_history(label="student", limit=20)
# Returns: [{"sha": "abc123", "message": "Update student",
#           "author": "user", "timestamp": "...", "is_current": True}, ...]

# Get specific version
old_content = manager.get_version(label="student", commit_sha="abc123")

# Restore
new_sha = manager.restore_version(label="student", commit_sha="abc123")
```

### Letta Sync
```python
# Blocks are auto-synced to Letta when sync_to_letta=True
# Letta block naming: youlab_user_{user_id}_{label}

block_id = manager.get_or_create_letta_block_id(label="student")
```

---

## Diff Approval Backend (Commit 2611ba2)

### New Components Added

**1. Blocks API Enhancements** (`src/youlab_server/server/blocks.py`)
- GET `/users/{user_id}/blocks` - List all blocks with pending diff counts
- GET `/users/{user_id}/blocks/{label}` - Get block with metadata + body + pending diffs
- PUT `/users/{user_id}/blocks/{label}` - Update block (user edit)
- GET `/users/{user_id}/blocks/{label}/history` - Version history
- GET `/users/{user_id}/blocks/{label}/versions/{sha}` - Get specific version
- POST `/users/{user_id}/blocks/{label}/restore` - Restore version
- GET `/users/{user_id}/blocks/diffs` - List all pending diffs
- POST `/users/{user_id}/blocks/diffs/{diff_id}/approve` - Approve diff
- POST `/users/{user_id}/blocks/diffs/{diff_id}/reject` - Reject diff

**2. Background Agent Threads** (`src/youlab_server/server/agents_threads.py`)
- GET `/users/{user_id}/agents` - List background agents with thread history
- POST `/users/{user_id}/agents/{agent_name}/threads` - Register new thread run
- Helper: `create_agent_thread()` - Creates OpenWebUI chat for background agent run

**3. Notes Adapter** (`src/youlab_server/server/notes_adapter.py`)
- Provides OpenWebUI Notes API compatibility for memory blocks
- GET `/api/you/notes/` - List blocks as notes
- GET `/api/you/notes/{note_id}` - Get block as note with versions
- POST `/api/you/notes/{note_id}/update` - Update block via notes API
- Converts markdown ↔ HTML for TipTap editor
- Exposes version history as undo/redo snapshots

**4. Full Replace Operation**
- Added `full_replace` operation type to diffs
- Replaces entire block body (vs append/replace specific content)

**5. Storage Layer Consolidation**
- Removed `convert.py` (functionality merged into `blocks.py` and `git.py`)
- `parse_frontmatter()` and `format_frontmatter()` now in `git.py`

---

## How Background Agents Would Update Progress/Operating Manual

Based on the codebase, here's how a background agent would update these blocks:

### Scenario 1: Progress Tracker (Weekly Summary)

**Config** (in `config/courses/*/course.toml`):
```toml
[[background_agents]]
id = "progress_tracker"
enabled = true
schedule = "0 0 * * 0"  # Weekly on Sunday
agent_types = ["college_essay"]
batch_size = 10

[[background_agents.queries]]
id = "weekly_progress"
question = "What progress has the student made this week? What milestones did they reach?"
session_scope = "last_week"
recent_limit = 20
target_block = "progress"
target_field = "weekly_summary"
merge_strategy = "append"
```

**Execution** (`src/youlab_server/background/runner.py`):
```python
# Runner executes query via Honcho
response = await honcho_client.query_dialectic(
    user_id=user_id,
    question="What progress has the student made this week?",
    session_scope=SessionScope.LAST_WEEK,
    recent_limit=20
)

# Apply to Progress block
enricher.enrich(
    agent_id=target_agent_id,
    block="progress",
    field="weekly_summary",
    content=response.insight,
    strategy=MergeStrategy.APPEND,
    source="background:progress_tracker",
    source_query="What progress..."
)

# Result:
# 1. Appends to progress.md body
# 2. Commits to git: "Update progress block"
# 3. Syncs to Letta block: youlab_user_{user_id}_progress
# 4. Writes audit entry to agent's archival memory
```

### Scenario 2: Operating Manual (Strategy Updates)

**Config**:
```toml
[[background_agents.queries]]
id = "engagement_insights"
question = "What engagement patterns have emerged? What strategies are working or not working?"
session_scope = "last_month"
recent_limit = 50
target_block = "operating_manual"
target_field = "effective_strategies"
merge_strategy = "append"
```

**Execution**:
```python
enricher.enrich(
    agent_id=target_agent_id,
    block="operating_manual",
    field="effective_strategies",
    content="Student responds well to morning sessions and short 25-min focused bursts",
    strategy=MergeStrategy.APPEND,
    source="background:engagement_analyzer",
    source_query="What engagement patterns..."
)
```

### Alternative: Interactive Agent Proposes, User Approves

If you want **user oversight** instead of auto-apply:

**Config Change**: Use `edit_memory_block` tool in main agent instead of background agent

```toml
# In main agent tools
[[agent.tools]]
id = "edit_memory_block"
rules = { type = "continue_loop" }
```

**Agent Behavior**:
```python
# Agent analyzes recent conversations and proposes update
edit_memory_block(
    block="operating_manual",
    field="effective_strategies",
    content="Morning sessions seem most productive",
    strategy="append",
    reasoning="Student has completed all assignments in morning sessions (3/3), but struggled in evening sessions (1/3 completed)"
)

# Creates PendingDiff - user sees notification in UI
# User reviews and approves → applied to operating_manual.md
```

---

## Key Implementation Patterns

### 1. Git-as-Database
- Every block edit = git commit
- Full version history with author attribution
- Enables rollback, diff viewing, audit trails

### 2. Frontmatter Metadata
- Blocks store schema reference, title, updated_at in YAML frontmatter
- Body contains actual memory content
- Letta only sees body (frontmatter is storage metadata)

### 3. Diff Approval Queue
- Agent proposals don't auto-apply (for interactive agents)
- Stored as JSON files in `pending_diffs/`
- Frontend can show pending count badges
- Approval triggers git commit with agent attribution

### 4. Dual-Client Pattern
- `UserBlockManager` handles git storage + diff queue
- Optionally syncs to Letta blocks for agent context
- Background agents bypass queue via `MemoryEnricher`

### 5. Audit Transparency
- Background agent edits logged to Letta archival memory
- Format: `[MEMORY_EDIT timestamp] Source: X, Block: Y, Field: Z, Content: ...`
- Agent can query its own edit history

---

## Files Analyzed

### Core Implementation
- `src/youlab_server/tools/memory.py` - edit_memory_block tool (238 lines)
- `src/youlab_server/storage/blocks.py` - UserBlockManager (364 lines)
- `src/youlab_server/storage/git.py` - Git storage layer (475 lines)
- `src/youlab_server/storage/diffs.py` - PendingDiff data model (144 lines)
- `src/youlab_server/memory/enricher.py` - Background agent enricher (255 lines)

### API Layer
- `src/youlab_server/server/blocks.py` - Blocks API endpoints
- `src/youlab_server/server/agents_threads.py` - Background agent threads (221 lines)
- `src/youlab_server/server/notes_adapter.py` - Notes API compatibility (313 lines)

### Background Execution
- `src/youlab_server/background/runner.py` - Background agent runner (237 lines)

### Recent Changes (Commit 2611ba2)
- Added `full_replace` operation type
- Consolidated `convert.py` → `blocks.py` + `git.py`
- Added Notes API adapter for frontend editors
- Added background agent thread tracking

---

## Recommendations for ARI-85 POC

For a **single-module course with background agent**, use this approach:

### 1. Memory Block Schema
```toml
# config/courses/poc-course/course.toml

[blocks.progress]
label = "progress"
description = "Student progress tracking"

[blocks.progress.fields]
module_completion = { type = "string", default = "" }
weekly_summary = { type = "string", default = "" }
current_challenges = { type = "list", max = 5 }

[blocks.operating_manual]
label = "operating_manual"
description = "Effective tutoring strategies"

[blocks.operating_manual.fields]
effective_strategies = { type = "list", max = 10 }
avoid_patterns = { type = "list", max = 5 }
```

### 2. Background Agent Config
```toml
[[background_agents]]
id = "weekly_progress_tracker"
enabled = true
schedule = "0 0 * * 0"  # Sunday midnight
agent_types = ["poc_tutor"]
batch_size = 10

[[background_agents.queries]]
id = "weekly_progress"
question = "Summarize the student's progress this week. What did they accomplish? What challenges did they face?"
session_scope = "last_week"
recent_limit = 20
target_block = "progress"
target_field = "weekly_summary"
merge_strategy = "append"

[[background_agents.queries]]
id = "effective_strategies"
question = "What tutoring strategies were most effective this week? What should we avoid?"
session_scope = "last_week"
recent_limit = 20
target_block = "operating_manual"
target_field = "effective_strategies"
merge_strategy = "append"
```

### 3. Main Agent Tools
```toml
# Give main agent ability to propose memory edits (with user approval)
[[agent.tools]]
id = "edit_memory_block"
rules = { type = "continue_loop" }
```

### 4. Implementation Flow

**Weekly Background Agent Run**:
1. Queries Honcho dialectic for weekly insights
2. Appends to Progress block (auto-applied, no approval needed)
3. Appends to Operating Manual (strategies learned)
4. Logs edits to agent archival memory
5. Syncs to Letta blocks

**Interactive Agent During Session**:
1. Reads current Progress/Operating Manual from Letta context
2. Can propose updates via `edit_memory_block` tool
3. Creates PendingDiff for user review
4. User approves → applied and synced

**User via Frontend**:
1. Views blocks via GET `/users/{user_id}/blocks/{label}`
2. Sees pending diffs count
3. Reviews/approves diffs via API
4. Can manually edit blocks (creates git commit)
5. Can view version history and restore previous versions

---

## Open Questions / Future Work

1. **LLM_DIFF Operation**: Currently reserved but not implemented. Would use LLM to intelligently merge agent proposals with current content.

2. **Diff Expiration**: PendingDiff has "expired" status but no automatic expiration logic yet.

3. **Diff Conflicts**: What happens if block changes between diff proposal and approval? Currently throws error on replace operation.

4. **Background Agent Approval**: Should some background agent queries require approval instead of auto-apply? Config flag?

5. **Audit Query**: No API endpoint yet to query agent edit history from archival memory.

6. **Diff Batching**: Can users approve multiple diffs at once?

---

## Conclusion

The memory tools system provides a robust, git-backed, dual-path approach to agent memory updates:

- **Interactive agents** use `edit_memory_block` → propose diffs → user approval
- **Background agents** use `MemoryEnricher` → auto-apply with audit logging
- **Git storage** provides version history, rollback, and audit trails
- **Letta sync** keeps agent context up-to-date
- **Frontmatter metadata** enables rich block schemas while keeping Letta context clean

For ARI-85, background agents can safely update Progress and Operating Manual blocks with weekly summaries, and the main tutor agent can propose memory updates that require user approval.
