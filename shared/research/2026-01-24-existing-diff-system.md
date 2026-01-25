---
date: 2026-01-24T10:45:00-08:00
researcher: ARI
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Existing Diff/Approval Workflow Analysis for Dolt Migration"
tags: [research, codebase, diff-system, pending-diffs, memory-blocks, dolt-migration]
status: complete
last_updated: 2026-01-24
last_updated_by: ARI
---

# Research: Existing Diff/Approval Workflow in YouLab

**Date**: 2026-01-24T10:45:00-08:00
**Researcher**: ARI
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Analyze the current diff/approval workflow in YouLab. Examine: PendingDiff data model, approval workflow implementation, frontend components for diff review, backend endpoints, integration with memory blocks. Key questions: keep or simplify for Dolt? UX improvements needed? How does it map to Dolt's native diffing?

## Summary

YouLab implements a custom diff/approval workflow where AI agents propose memory block edits, stored as `PendingDiff` objects in JSON files. Users review these proposals through a dedicated overlay UI and can approve (applying changes to git-backed storage) or reject them. The system supports three operation types: append, replace, and full_replace.

The current implementation is built on Git for versioning and JSON files for pending diff storage. This research documents the existing system to inform decisions about whether to keep, simplify, or replace it when migrating to Dolt.

---

## Detailed Findings

### 1. PendingDiff Data Model

#### Location: `src/youlab_server/storage/diffs.py:20-68`

The `PendingDiff` dataclass represents a proposed change awaiting user approval:

```python
@dataclass
class PendingDiff:
    id: str                    # UUID generated on creation
    user_id: str               # Who owns this memory
    agent_id: str              # Which agent proposed it
    block_label: str           # Target block (e.g., "student", "journey")
    field: str | None          # Specific field (optional)
    operation: Literal["append", "replace", "llm_diff", "full_replace"]
    current_value: str         # Full block content at proposal time
    proposed_value: str        # New content to apply
    reasoning: str             # Agent's explanation for the change
    confidence: Literal["low", "medium", "high"]
    source_query: str | None   # Query that triggered this (optional)
    status: Literal["pending", "approved", "rejected", "superseded", "expired"]
    created_at: str            # ISO timestamp
    reviewed_at: str | None = None
    applied_commit: str | None = None  # Git SHA when approved
```

#### Storage Format

Pending diffs are stored as JSON files at:
```
.data/users/{user_id}/pending_diffs/{diff_id}.json
```

#### PendingDiffStore Class: `src/youlab_server/storage/diffs.py:71-143`

Provides persistence operations:
- `save(diff)` - Write diff to JSON file
- `get(diff_id)` - Retrieve by ID
- `list_pending(block_label?)` - List pending diffs, optionally filtered
- `count_pending()` - Count pending diffs per block
- `update_status(diff_id, status, commit?)` - Update diff status
- `supersede_older(block_label, keep_id)` - Mark older diffs as superseded

---

### 2. Approval Workflow Implementation

#### Creating Diffs: `src/youlab_server/storage/blocks.py:238-277`

The `UserBlockManager.propose_edit()` method creates pending diffs:

1. Reads current block content from git storage
2. Creates `PendingDiff` via factory method
3. Saves to JSON file via `PendingDiffStore`
4. Logs "diff_proposed" event
5. Returns the `PendingDiff` object

#### Approving Diffs: `src/youlab_server/storage/blocks.py:279-335`

The `UserBlockManager.approve_diff()` method:

1. Retrieves diff by ID, validates status is "pending"
2. Gets current block body content
3. Applies edit based on operation type:
   - **append**: Adds content to end with double newline
   - **replace**: Finds and replaces `current_value` in content
   - **full_replace**: Replaces entire content
4. Updates block via `update_block()` (commits to git)
5. Updates diff status to "approved" with commit SHA
6. Supersedes older pending diffs for same block

#### Rejecting Diffs: `src/youlab_server/storage/blocks.py:337-340`

Simply updates status to "rejected" via `update_status()`.

---

### 3. Backend API Endpoints

#### Location: `src/youlab_server/server/blocks.py`

All endpoints are under `/users/{user_id}/blocks`:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/{label}/diffs` | List pending diffs for a block (line 260) |
| `POST` | `/{label}/diffs/{diff_id}/approve` | Approve and apply a diff (line 272) |
| `POST` | `/{label}/diffs/{diff_id}/reject` | Reject a diff (line 288) |
| `POST` | `/{label}/propose` | Create new pending diff (line 307) |
| `GET` | `/diffs/counts` | Get diff counts across all blocks (line 143) |

#### Request/Response Models

**DiffSummary** (line 83-95):
```python
class DiffSummary(BaseModel):
    id: str
    block: str
    field: str | None
    operation: str
    reasoning: str
    confidence: str
    created_at: str
    agent_id: str
    old_value: str | None = None
    new_value: str | None = None
```

**ProposeEditRequest** (line 98-108):
```python
class ProposeEditRequest(BaseModel):
    agent_id: str
    field: str | None = None
    content: str
    strategy: str = "append"  # append, replace, full_replace
    reasoning: str = ""
```

---

### 4. Tools That Create Diffs

#### Two Implementations Exist

**Legacy (in-process)**: `src/youlab_server/tools/memory.py`
- Direct `UserBlockManager` access
- Has Letta client for sync

**Sandbox (HTTP-based)**: `src/youlab_server/tools/sandbox.py`
- Uses stdlib only (works in Docker sandbox)
- Makes HTTP POST to `/users/{user_id}/blocks/{label}/propose`
- Default export in `tools/__init__.py:25`

#### edit_memory_block Parameters

```python
def edit_memory_block(
    block: str,          # Block label (e.g., "student")
    field: str,          # Field to update
    content: str,        # Content to add/replace
    strategy: str = "append",  # append, replace, full_replace
    reasoning: str = "",       # Explanation
    agent_state: dict = None,  # Injected by Letta
) -> str
```

---

### 5. Frontend Components

#### DiffApprovalOverlay: `OpenWebUI/open-webui/src/lib/components/you/DiffApprovalOverlay.svelte`

Main component for reviewing pending diffs:

**Features**:
- Navigation between multiple diffs (prev/next buttons)
- Confidence badge with color coding (high=green, medium=blue, low=amber)
- Displays agent reasoning in italics
- Diff visualization with line-by-line view:
  - Context lines (gray)
  - Deletions (red background with `-` prefix)
  - Additions (green background with `+` prefix)
- Keyboard shortcuts: `A` approve, `R` reject, `←/→` navigate, `E` dismiss

**Props**:
- `label: string` - Block label
- `diffs: PendingDiff[]` - Array of pending diffs
- `currentContent: string` - Current block content for context

**Events**:
- `approved` - Emitted with `{ diffId }` after successful approval
- `rejected` - Emitted with `{ diffId }` after successful rejection
- `dismiss` - Emitted when user wants to edit directly
- `error` - Emitted with `{ message }` on failures

#### DiffBadge: `OpenWebUI/open-webui/src/lib/components/you/DiffBadge.svelte`

Simple badge component showing pending diff count:
- Links to `/you` page
- Uses warning-style Badge component
- Shows "{count} pending" text

#### API Client: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`

Frontend API functions:

```typescript
interface PendingDiff {
    id: string;
    block: string;
    field: string | null;
    operation: string;
    reasoning: string;
    confidence: string;
    createdAt: string;
    agentId: string;
    oldValue?: string;
    newValue?: string;
}

getBlockDiffs(userId, label, token): Promise<PendingDiff[]>
approveDiff(userId, label, diffId, token): Promise<{diffId, commitSha}>
rejectDiff(userId, label, diffId, token, reason?): Promise<{status, diffId}>
```

#### Integration Points

Components that use diff functionality:
- `MemoryBlockEditor.svelte` - Block editor with overlay integration
- `BlockEditor.svelte` - Generic block editor
- `BlockDetailModal.svelte` - Modal view with diff info
- `BlockCard.svelte` - Card with pending diff indicators
- `Sidebar.svelte` - Shows pendingDiffsCount badge

---

### 6. Git Integration

#### Location: `src/youlab_server/storage/git.py`

Memory blocks are stored as markdown files with YAML frontmatter:
```
.data/users/{user_id}/
    .git/
    memory-blocks/
        student.md
        engagement_strategy.md
        journey.md
    pending_diffs/
        {diff_id}.json
```

#### Version History

`GitUserStorage` provides:
- `get_block_history(label, limit)` - Get commit history for block
- `get_block_at_version(label, sha)` - Get content at specific commit
- `restore_block(label, sha)` - Restore to previous version
- `diff_versions(label, old_sha, new_sha)` - Get diff between versions

Each approved diff creates a new git commit with:
- Commit message: "Apply agent suggestion: {reasoning[:50]}"
- Author footer: "Author: agent:{agent_id}"

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Svelte)                        │
├─────────────────────────────────────────────────────────────────┤
│  DiffBadge      │  DiffApprovalOverlay   │  MemoryBlockEditor   │
│  (count badge)  │  (review & approve)    │  (integrates both)   │
└────────┬────────┴──────────┬─────────────┴──────────┬───────────┘
         │                   │                        │
         │          ┌────────▼────────┐               │
         │          │ memory/index.ts │               │
         │          │ (API client)    │               │
         │          └────────┬────────┘               │
         │                   │                        │
─────────┴───────────────────┼────────────────────────┴───────────
                             │ HTTP
         ┌───────────────────▼───────────────────┐
         │          FastAPI Endpoints             │
         │       /users/{id}/blocks/...           │
         │  (GET diffs, POST approve, reject)     │
         └───────────────────┬───────────────────┘
                             │
         ┌───────────────────▼───────────────────┐
         │          UserBlockManager              │
         │  propose_edit(), approve_diff()        │
         └───────┬───────────────────┬───────────┘
                 │                   │
    ┌────────────▼───────┐  ┌───────▼────────┐
    │  PendingDiffStore  │  │ GitUserStorage │
    │  (JSON files)      │  │ (markdown+git) │
    └────────────────────┘  └────────────────┘
```

---

## Data Flow

### Agent Proposes Edit

1. Agent calls `edit_memory_block()` tool
2. Tool extracts `agent_id`, `user_id` from `agent_state`
3. **Sandbox version**: POST to `/users/{user_id}/blocks/{label}/propose`
4. **Legacy version**: Direct `UserBlockManager.propose_edit()` call
5. Current block content captured as `current_value`
6. `PendingDiff` created with UUID, saved as JSON
7. Tool returns confirmation message

### User Reviews Diff

1. User opens block in MemoryBlockEditor
2. Component fetches diffs via `getBlockDiffs()`
3. If diffs exist, DiffApprovalOverlay is shown
4. User sees: reasoning, confidence, operation, diff visualization
5. Keyboard shortcuts available for quick action

### User Approves/Rejects

**Approve flow**:
1. POST to `/blocks/{label}/diffs/{id}/approve`
2. `approve_diff()` validates status is "pending"
3. Applies edit based on operation type
4. Commits new content to git
5. Updates diff status to "approved" with commit SHA
6. Supersedes older pending diffs for same block
7. Frontend refreshes block content

**Reject flow**:
1. POST to `/blocks/{label}/diffs/{id}/reject`
2. Updates status to "rejected"
3. Frontend removes diff from list

---

## Code References

### Backend
- `src/youlab_server/storage/diffs.py:20-68` - PendingDiff dataclass
- `src/youlab_server/storage/diffs.py:71-143` - PendingDiffStore class
- `src/youlab_server/storage/blocks.py:238-277` - propose_edit() method
- `src/youlab_server/storage/blocks.py:279-335` - approve_diff() method
- `src/youlab_server/server/blocks.py:260-299` - HTTP endpoints for diffs
- `src/youlab_server/tools/sandbox.py:108-201` - Sandbox edit_memory_block

### Frontend
- `OpenWebUI/open-webui/src/lib/components/you/DiffApprovalOverlay.svelte` - Review overlay
- `OpenWebUI/open-webui/src/lib/components/you/DiffBadge.svelte` - Count badge
- `OpenWebUI/open-webui/src/lib/apis/memory/index.ts:151-336` - API client

---

## Key Observations

### Current System Characteristics

1. **Dual Storage**:
   - Pending diffs: JSON files in `pending_diffs/`
   - Applied changes: Git commits in `memory-blocks/`

2. **Three Operation Types**:
   - `append` - Add content to end
   - `replace` - Find and replace substring
   - `full_replace` - Replace entire content

3. **Superseding Logic**: When a diff is approved, older pending diffs for the same block are marked as "superseded"

4. **Confidence Levels**: low/medium/high with visual indicators (amber/blue/green)

5. **Git-backed History**: Every approved change creates a commit, enabling version restoration

6. **Sandbox Compatibility**: HTTP-based tool works in Docker sandbox environment

### Mapping to Dolt

| Current Concept | Dolt Equivalent |
|-----------------|-----------------|
| Pending diff JSON | Working set changes (uncommitted) |
| Approved diff → git commit | Dolt commit |
| Block history (git log) | `dolt log` / `dolt_log()` |
| Version restore | `dolt checkout` / branch from commit |
| Diff visualization | `dolt diff` output |

### Potential Simplifications with Dolt

1. **Remove PendingDiffStore**: Use Dolt's working set as the "pending" state
2. **Native diffing**: Use `dolt diff` instead of storing old_value/new_value
3. **Branch per agent**: Each agent could work on a branch, user merges to main
4. **SQL-based queries**: Replace JSON file traversal with SQL queries

---

## Related Research

- `thoughts/shared/plans/2026-01-14-ARI-82-phase4-diff-approval.md` - UI implementation plan
- `thoughts/shared/plans/2026-01-12-ARI-81-diff-approval-fix.md` - Bug fix plan
- `thoughts/shared/research/2026-01-12-ARI-81-phase-4-diff-approval-analysis.md` - Technical analysis
- `docs/Memory-System.md` - Official memory system documentation
- `docs/Agent-Tools.md` - Tool documentation including edit_memory_block

---

## Open Questions

1. **Keep reasoning/confidence metadata?** These fields provide valuable context but aren't native to Dolt. Could store in a separate table or commit message.

2. **How to handle superseding with Dolt branches?** Current logic marks older diffs as superseded when one is approved. With branches, this might be handled via merge conflicts or automatic rebasing.

3. **Frontend changes needed?** The DiffApprovalOverlay currently expects specific JSON structure. Would need adaptation for Dolt's diff format.

4. **Agent-per-branch model?** Consider whether each agent should work on its own branch, with user as gatekeeper for main.

5. **Real-time diff display?** Dolt diffing is synchronous SQL. Could enable real-time diff preview as agent types.
