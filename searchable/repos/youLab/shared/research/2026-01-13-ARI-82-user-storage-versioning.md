---
date: 2026-01-13T10:30:00+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "ARI-82 User Storage and Git Versioning for Undo/Redo"
tags: [research, codebase, git-storage, versioning, undo-redo, memory-blocks, ARI-82]
status: complete
last_updated: 2026-01-13
last_updated_by: ariasulin
---

# Research: ARI-82 User Storage and Git Versioning for Undo/Redo

**Date**: 2026-01-13T10:30:00+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Understand the current git-backed versioning system for memory blocks and analyze how it maps to the new spec's undo/redo requirements (Cursor-like diff view with auto-apply).

## Summary

The codebase has a complete git-backed storage system (`GitUserStorage`, `GitUserStorageManager`) that provides version history, content retrieval at any commit, and restore functionality. The architecture creates one git repository per user under `.data/users/{user_id}/`. Each block edit creates a new commit, providing a complete audit trail.

**Key Finding**: The existing system provides all the primitives needed for undo/redo:
- `get_block_history()` - list versions (newest first)
- `get_block_at_version()` - retrieve content at any SHA
- `restore_block()` - restore old version (creates new commit)
- `diff_versions()` - compare two versions

**Gap for Undo/Redo**: The current `restore_block()` creates a **new commit** when restoring, which is semantically correct for git but may complicate redo navigation. A true undo/redo would need either:
1. A "current position in history" pointer separate from HEAD
2. Or accept that "undo" creates new commits (git's natural model)

## Detailed Findings

### GitUserStorage Class

**Location**: `src/youlab_server/storage/git.py:41-326`

Core class managing a single user's git repository.

**Directory Structure** (documented at git.py:43-57):
```
.data/users/{user_id}/
    .git/
    blocks/
        student.toml
        engagement_strategy.toml
        journey.toml
    pending_diffs/
        {diff_id}.json
    agent_threads/
        {agent_name}/
            {chat_id}.json
```

**Key Methods**:

| Method | Lines | Purpose |
|--------|-------|---------|
| `init()` | 93-105 | Initialize user storage + git repo |
| `read_block()` | 124-138 | Read current block content |
| `write_block()` | 140-184 | Write block + create git commit |
| `get_block_history()` | 186-221 | Get version list for a block |
| `get_block_at_version()` | 223-247 | Read block at specific commit |
| `restore_block()` | 249-275 | Restore old version (new commit) |
| `diff_versions()` | 277-303 | Get old/new content for two SHAs |
| `list_blocks()` | 321-325 | List all block labels |

### Data Classes

**VersionInfo** (git.py:21-28):
```python
@dataclass
class VersionInfo:
    commit_sha: str
    message: str
    author: str
    timestamp: datetime
    is_current: bool = False
```

**FileDiff** (git.py:31-38):
```python
@dataclass
class FileDiff:
    old_content: str
    new_content: str
    old_sha: str
    new_sha: str
```

### GitUserStorageManager

**Location**: `src/youlab_server/storage/git.py:328-359`

Factory for `GitUserStorage` instances with caching.

```python
class GitUserStorageManager:
    def get(self, user_id: str) -> GitUserStorage
    def user_exists(self, user_id: str) -> bool
    def list_users(self) -> list[str]
```

### UserBlockManager (Higher-Level API)

**Location**: `src/youlab_server/storage/blocks.py:21-395`

Wraps `GitUserStorage` with:
- TOML ↔ Markdown conversion
- Letta sync
- Pending diff management

**Key Methods**:

| Method | Lines | Purpose |
|--------|-------|---------|
| `get_history()` | 125-137 | Version history as dicts (for API) |
| `get_version()` | 139-141 | Content at version |
| `restore_version()` | 143-157 | Restore + optionally sync to Letta |

### Git Commit Patterns

Example commit history from `.data/users/ea1ac2d7-9442-400f-b1ba-759891b3f015/`:

```
5942aa4 Add pending diffs and agent threads
01f61b3 Add learning journey block
3a26604 Add engagement strategy block
3482816 Add student profile block
d160875 Initialize user storage
```

**Commit Message Format** (from `write_block()` at git.py:173-175):
```python
commit = self.repo.index.commit(
    f"{message}\n\nAuthor: {author}",
)
```

The author is stored in the commit body (not git author field) and extracted via `_extract_author()`.

### HTTP API Endpoints

**Location**: `src/youlab_server/server/blocks.py`

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /users/{user_id}/blocks/{label}/history` | GET | List versions |
| `GET /users/{user_id}/blocks/{label}/versions/{sha}` | GET | Content at version |
| `POST /users/{user_id}/blocks/{label}/restore` | POST | Restore version |

### Current .data Structure (Sample User)

```
.data/users/ea1ac2d7-9442-400f-b1ba-759891b3f015/
├── .git/                    # Full git repo
├── .gitignore
├── blocks/
│   ├── student.toml         # User profile
│   ├── engagement_strategy.toml
│   └── journey.toml
├── pending_diffs/
│   ├── diff-001-student-strength.json
│   └── diff-002-engagement.json
└── agent_threads/
    ├── essay_coach/
    │   ├── thread_001.json
    │   └── thread_002.json
    └── learning_optimizer/
        └── thread_001.json
```

### New Spec User Repo Schema (Target)

From the spec requirements:
```
.data/{user-id}/
  config.toml               # NEW: user config
  memory-blocks/            # RENAMED from blocks/
  notes/                    # NEW: user notes
  files/                    # NEW: file storage
    private/
    shared/
    archive/
  courses/                  # NEW: course symlinks
```

## Gap Analysis: Undo/Redo Requirements

### What Exists

| Feature | Status | Implementation |
|---------|--------|----------------|
| Version history | ✅ | `get_block_history()` |
| Content at any version | ✅ | `get_block_at_version()` |
| Restore old version | ✅ | `restore_block()` |
| Diff between versions | ✅ | `diff_versions()` |
| Commit messages | ✅ | Stored with author |
| Per-block history | ✅ | `git log --follow` on file path |

### What's Needed for Undo/Redo Buttons

| Feature | Status | Notes |
|---------|--------|-------|
| Navigate back (undo) | 🟡 | `restore_block()` works but creates new commit |
| Navigate forward (redo) | ⚠️ | Not straightforward after restore creates new commit |
| Current position tracking | ❌ | No "position in history" concept |
| Inline diff view | 🟡 | `diff_versions()` returns content, UI must render |
| Auto-apply (Cursor-style) | ✅ | `restore_block()` does this |

### Semantic Challenge: Git's Model vs. Undo/Redo

**Git's Natural Model**:
```
A → B → C → D (HEAD)
              ↓ restore to B
A → B → C → D → E (HEAD, content = B)
```

Restoring B creates commit E with B's content. There's no "redo" to D because D is still in history, but HEAD moved forward.

**Traditional Undo/Redo Model**:
```
A → B → C → D
        ↑
    (current position after undo)

After redo:
A → B → C → D
            ↑
        (back to D)
```

### Implementation Options for Undo/Redo

**Option 1: Git-native (Current System)**
- Undo = `restore_block(older_sha)` (creates new commit)
- Redo = `restore_block(newer_sha)` (creates another commit)
- History grows linearly with each undo/redo
- Pro: Simple, audit trail preserved
- Con: Can't "redo" to the version you were at before undo without knowing its SHA

**Option 2: Position Pointer**
- Store `current_position` SHA per block (outside git)
- Undo = move pointer backward, don't change HEAD
- Redo = move pointer forward
- Apply = `restore_block(position)` only when user saves
- Pro: True undo/redo semantics
- Con: More complexity, orphaned states possible

**Option 3: Branch-based**
- Each "undo" creates a branch
- Redo switches between branches
- Pro: Git-native branching
- Con: Complex for users to understand, many branches

## Architecture Documentation

### Data Flow for Current Restore

```
[User clicks "Restore to version X"]
         ↓
[Frontend calls POST /blocks/{label}/restore]
    • body: { commit_sha: "abc123" }
         ↓
[UserBlockManager.restore_version()]
    • Calls storage.restore_block()
    • Optionally syncs to Letta
         ↓
[GitUserStorage.restore_block()]
    • get_block_at_version(sha)
    • write_block() with restored content
    • Creates NEW commit with message "Restore {label} to version abc123"
         ↓
[Returns new commit SHA to frontend]
```

### Pending Diffs Interaction

Note: The diff approval system (documented in `2026-01-12-ARI-81-phase-4-diff-approval-analysis.md`) operates independently from version history. When a diff is approved:
- It creates a new commit via `write_block()`
- That commit appears in `get_block_history()`
- Users can undo an approved diff by restoring to pre-approval version

## Code References

- `src/youlab_server/storage/git.py:41-326` - GitUserStorage class
- `src/youlab_server/storage/git.py:328-359` - GitUserStorageManager
- `src/youlab_server/storage/blocks.py:125-157` - UserBlockManager version methods
- `src/youlab_server/server/blocks.py:185-226` - HTTP endpoints for history/restore

## Historical Context (from thoughts/)

- `thoughts/shared/research/2026-01-12-ARI-81-phase-4-diff-approval-analysis.md` - Documents issues with diff creation/approval that affect the same storage layer
- `thoughts/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md` - Original design for git-backed memory storage

## Related Research

- `thoughts/shared/research/2026-01-12-ARI-78-toml-markdown-roundtrip.md` - TOML ↔ Markdown conversion (affects what content looks like in versions)
- `thoughts/shared/research/2026-01-12-ARI-78-unified-user-memory-blocks.md` - Block architecture

## Open Questions

1. **Should undo/redo use git-native restore (creates commits) or a position pointer?**
   - Git-native is simpler but history grows with each undo
   - Position pointer is more complex but matches user expectations

2. **How to handle directory structure migration?**
   - Current: `blocks/` → New spec: `memory-blocks/`
   - Need migration strategy for existing user repos

3. **How to show diff inline like Cursor?**
   - `diff_versions()` provides content
   - Frontend needs unified diff rendering component
   - Consider using existing diff libraries (diff-match-patch, jsdiff)

4. **Per-field vs per-file versioning?**
   - Current system versions entire TOML files
   - For "undo field X", would need to extract/merge fields from different versions
