---
date: 2026-01-28T12:00:00-08:00
researcher: ariav
git_commit: f08f800a4483a352de6b938a253057d1b6351264
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "Memory Block System: Current State for Dolt Refactor"
tags: [research, codebase, memory-blocks, dolt, letta, openwebui, git-storage]
status: complete
last_updated: 2026-01-28
last_updated_by: ariav
---

# Research: Memory Block System Current State

**Date**: 2026-01-28T12:00:00-08:00
**Researcher**: Ariav Asulin
**Git Commit**: f08f800a4483a352de6b938a253057d1b6351264
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question
What is the current state of the memory block system? How is it integrated into the frontend (OpenWebUI)? This is preliminary research for the Dolt memory block system refactor.

## Summary

The memory block system is a **git-backed, user-scoped storage system** that enables persistent memory for AI agents with human-in-the-loop approval workflows. Key components:

1. **Backend Storage**: GitPython-based storage in `.data/users/{user_id}/memory-blocks/` with markdown files and YAML frontmatter
2. **Letta Integration**: Blocks sync to Letta agent framework (being deprecated for Agno)
3. **Diff Approval Workflow**: Agents propose edits → stored as pending diffs → users approve/reject
4. **Frontend UI**: OpenWebUI Svelte components with rich text editing and git-based version history
5. **Notes API Adapter**: Bridge between OpenWebUI's Notes format and git-backed blocks

The system is functional but transitioning away from Letta and pygit toward Dolt for versioning.

## Detailed Findings

### 1. Backend Storage Architecture

#### Git Storage Layer (`src/youlab_server/storage/git.py`)

**Directory Structure:**
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

**File Format**: Markdown with YAML frontmatter
```markdown
---
block: student
schema: college-essay/student
updated_at: 2026-01-28T12:00:00+00:00
title: Student Profile
---

Student name: John Doe
Interests: Computer science, music
...
```

**Key Classes:**
- `GitUserStorage` (`git.py:89`): Per-user storage with GitPython repo management
- `GitUserStorageManager` (`git.py:443`): Factory for user storage instances with caching

**Core Operations:**
- `read_block(label)` / `read_block_body(label)` / `read_block_metadata(label)`
- `write_block(label, content, message, author, schema, title)` → creates git commit
- `get_block_history(label, limit)` → returns `VersionInfo[]`
- `get_block_at_version(label, commit_sha)` / `restore_block(label, commit_sha)`

#### User Block Manager (`src/youlab_server/storage/blocks.py:19`)

`UserBlockManager` provides high-level block operations:
- CRUD operations with git commits
- Letta sync (optional)
- Pending diff management for approval workflow

**Letta Sync Pattern** (`blocks.py:161-204`):
```python
def _sync_block_to_letta(self, label: str) -> None:
    block_name = f"youlab_user_{self.user_id}_{label}"
    body = self.storage.read_block_body(label)
    # Find or create block in Letta
    existing = next((b for b in self.letta.blocks.list() if b.name == block_name), None)
    if existing:
        self.letta.blocks.update(block_id=existing.id, value=body)
    else:
        self.letta.blocks.create(label=label, name=block_name, value=body)
```

### 2. Pending Diff System (Approval Workflow)

#### Diff Storage (`src/youlab_server/storage/diffs.py`)

**`PendingDiff` dataclass** (`diffs.py:20-38`):
```python
@dataclass
class PendingDiff:
    id: str
    user_id: str
    agent_id: str
    block_label: str
    field: str | None
    operation: Literal["append", "replace", "llm_diff", "full_replace"]
    current_value: str
    proposed_value: str
    reasoning: str
    confidence: Literal["low", "medium", "high"]
    status: Literal["pending", "approved", "rejected", "superseded", "expired"]
```

**Storage**: JSON files in `pending_diffs/{diff_id}.json`

**Workflow:**
1. Agent calls `edit_memory_block()` tool → creates `PendingDiff` via `propose_edit()`
2. User sees pending badge in UI → reviews in `DiffApprovalOverlay`
3. User approves → `approve_diff()` applies edit, commits to git, syncs to Letta
4. User rejects → `reject_diff()` marks as rejected

#### Block API Endpoints (`src/youlab_server/server/blocks.py`)

**Router**: `APIRouter(prefix="/users/{user_id}/blocks", tags=["blocks"])`

**Endpoints:**
- `GET /users/{user_id}/blocks` → list blocks with pending diff counts
- `GET /users/{user_id}/blocks/{label}` → block detail
- `PUT /users/{user_id}/blocks/{label}` → update block (user edit)
- `GET /users/{user_id}/blocks/{label}/history` → version history
- `GET /users/{user_id}/blocks/{label}/versions/{commit_sha}` → specific version
- `POST /users/{user_id}/blocks/{label}/restore` → restore version
- `GET /users/{user_id}/blocks/{label}/diffs` → pending diffs
- `POST /users/{user_id}/blocks/{label}/diffs/{diff_id}/approve` → approve
- `POST /users/{user_id}/blocks/{label}/diffs/{diff_id}/reject` → reject
- `POST /users/{user_id}/blocks/{label}/propose` → HTTP tool endpoint for sandboxed agents

### 3. Memory Tool (`src/youlab_server/tools/memory.py`)

**`edit_memory_block()` function** (`memory.py:33-115`):
- Called by Letta agents to propose memory edits
- Creates `PendingDiff` for user approval
- **Known Issue**: TODO comment at line 8 notes this fails in Letta sandbox due to import access

```python
def edit_memory_block(block, field, content, strategy="append", reasoning="", agent_state=None):
    """Propose an update to a memory block. Change is NOT applied immediately."""
    manager = UserBlockManager(user_id, user_storage, letta_client=_letta_client)
    diff = manager.propose_edit(
        agent_id=agent_id, block_label=block, field=field,
        operation=strategy, proposed_value=content, reasoning=reasoning
    )
    return f"Proposed change to {block}.{field} (ID: {diff.id[:8]}). User will review."
```

### 4. Notes API Adapter (`src/youlab_server/server/notes_adapter.py`)

**Purpose**: Bridge OpenWebUI's Notes API format to git-backed blocks

**Router**: `APIRouter(prefix="/you/notes", tags=["notes-adapter"])`
**Full path**: `/api/you/notes/*` (mounted at `/api` in main.py:177)

**Endpoints:**
- `GET /api/you/notes/` → list all blocks as notes
- `GET /api/you/notes/{note_id}` → block detail with version history
- `POST /api/you/notes/{note_id}/update` → update with git commit

**Key Transformations:**
- Block markdown → `NoteContent` with `html`, `md`, `json` (json always null)
- Git history → `NoteVersion[]` with `sha`, `message`, `timestamp`
- Timestamps converted to nanoseconds epoch for OpenWebUI compatibility

### 5. Letta Integration (Being Deprecated)

#### Current Usage Patterns:

**1. UserBlockManager Sync** (`storage/blocks.py:161`):
- Syncs git blocks to Letta as `youlab_user_{user_id}_{label}` blocks
- Body content only (frontmatter stripped)
- Triggered after every `update_block()` call

**2. AgentManager** (`server/agents.py:19`):
- Creates Letta agents with memory blocks from curriculum config
- **Per-agent blocks**: Created fresh for each agent
- **Shared blocks**: Named `youlab_shared_{course_id}_{label}`, reused across agents

**3. BackgroundAgentFactory** (`background/factory.py:22`):
- Creates ephemeral agents for background tasks
- `fresh=True` deletes old agent to prevent message history contamination
- Updates blocks on reused agents via `letta.blocks.modify()`

**4. Curriculum Tools** (`tools/curriculum.py`):
- `_get_journey_block()` / `_update_journey_block()` read/write journey progress
- Uses `letta.agents.blocks.list()` and `letta.agents.blocks.update()`

#### Letta SDK APIs Used:
```python
# Global blocks
letta.blocks.list()
letta.blocks.create(label=..., value=..., name=..., description=...)
letta.blocks.update(block_id=..., value=...)
letta.blocks.modify(block_id=..., value=...)

# Agent blocks
letta.agents.blocks.list(agent_id=...)
letta.agents.blocks.attach(agent_id=..., block_id=...)
letta.agents.blocks.update(agent_id=..., block_id=..., value=...)

# Agent lifecycle
letta.agents.create(name=..., memory_blocks=[...], block_ids=[...], ...)
letta.agents.delete(agent_id=...)
```

### 6. Frontend Integration (OpenWebUI)

#### API Clients (`OpenWebUI/open-webui/src/lib/apis/memory/`)

**Two parallel APIs:**
1. `index.ts`: Legacy memory API → `/users/{userId}/blocks/*`
2. `notes.ts`: Notes adapter → `/api/you/notes/*`

**Configuration** (`src/lib/constants.ts:16-19`):
```typescript
export const YOULAB_API_BASE_URL = browser
    ? import.meta.env.VITE_YOULAB_API_URL || 'http://localhost:8100'
    : 'http://localhost:8100';
```

#### Svelte Stores (`src/lib/stores/memory.ts`)

```typescript
export const memoryBlocks = writable<MemoryBlock[]>([]);
export const selectedBlock = writable<BlockDetail | null>(null);
export const blockHistory = writable<VersionInfo[]>([]);
export const pendingDiffsCount = derived(memoryBlocks,
    $blocks => $blocks.reduce((sum, b) => sum + b.pendingDiffs, 0));
```

#### UI Components

**Profile Page** (`/workspace/profile` via redirect from `/you`):
- Lists all memory blocks with pending diff badges
- `BlockCard` components link to `/you/blocks/{label}`

**Memory Block Editor** (`/you/blocks/[label]`):
- `MemoryBlockEditor.svelte`: TipTap rich text editor
- Git-based undo/redo via `versionIdx` navigation
- 200ms debounced autosave to `/api/you/notes/{label}/update`
- Shows `DiffApprovalOverlay` when pending diffs exist

**Diff Approval Overlay** (`DiffApprovalOverlay.svelte`):
- Full-screen overlay with syntax-highlighted diff view
- Keyboard shortcuts: 'a' approve, 'r' reject, arrows navigate, 'e' dismiss
- Context lines shown for `replace` operations
- Inline approve/reject buttons with reasoning text

**Version History**:
- Undo button → `versionIdx++` (older)
- Redo button → `versionIdx--` (newer) or null (current)
- Restore button → creates new commit from historical version

### 7. Data Flow Diagrams

#### User Edit Flow:
```
User types in MemoryBlockEditor
    ↓
scheduleAutosave() (200ms debounce)
    ↓
updateMemoryNoteById() → POST /api/you/notes/{label}/update
    ↓
notes_adapter.py → UserBlockManager.update_block()
    ↓
GitUserStorage.write_block() → git commit
    ↓
_sync_block_to_letta() → letta.blocks.update()
    ↓
Response with new versions → UI updates
```

#### Agent Edit Flow:
```
Letta agent calls edit_memory_block() tool
    ↓
UserBlockManager.propose_edit()
    ↓
PendingDiff saved to pending_diffs/{id}.json
    ↓
User opens block editor → fetches /blocks/{label}/diffs
    ↓
DiffApprovalOverlay shows pending change
    ↓
User clicks Approve → POST /blocks/{label}/diffs/{id}/approve
    ↓
UserBlockManager.approve_diff()
    ↓
Applies edit, git commit, syncs to Letta
    ↓
Supersedes older diffs for same block
```

## Code References

### Backend
- `src/youlab_server/storage/git.py:89` - GitUserStorage class
- `src/youlab_server/storage/blocks.py:19` - UserBlockManager class
- `src/youlab_server/storage/diffs.py:20` - PendingDiff dataclass
- `src/youlab_server/server/blocks.py:130` - Block API router
- `src/youlab_server/server/notes_adapter.py:24` - Notes API adapter
- `src/youlab_server/tools/memory.py:33` - edit_memory_block tool

### Frontend
- `OpenWebUI/open-webui/src/lib/apis/memory/index.ts` - Memory API client
- `OpenWebUI/open-webui/src/lib/apis/memory/notes.ts` - Notes API client
- `OpenWebUI/open-webui/src/lib/stores/memory.ts` - Svelte stores
- `OpenWebUI/open-webui/src/lib/components/you/MemoryBlockEditor.svelte` - Editor
- `OpenWebUI/open-webui/src/lib/components/you/DiffApprovalOverlay.svelte` - Diff UI
- `OpenWebUI/open-webui/src/lib/components/workspace/Profile.svelte` - Block list

## Architecture Documentation

### Current Patterns

1. **Git as Source of Truth**: Blocks stored as markdown with YAML frontmatter, GitPython for version control
2. **Letta as Runtime**: Blocks synced to Letta for agent access, separate from storage
3. **Dual API Strategy**: Legacy `/blocks/*` API + Notes adapter `/api/you/notes/*`
4. **Approval Workflow**: Agents propose → JSON diffs → user approves → git commit
5. **Component Separation**: Storage layer (git.py) → Manager layer (blocks.py) → API layer (blocks.py/notes_adapter.py)

### Key Dependencies

- **GitPython** (`git`): Git repository operations
- **PyYAML** (`yaml`): Frontmatter parsing
- **Letta SDK** (`letta_client`): Agent framework (being deprecated)
- **FastAPI**: HTTP API framework
- **Svelte**: Frontend UI framework
- **TipTap**: Rich text editor (via RichTextInput)

## Historical Context (from thoughts/)

The system evolved from a Letta-first approach to a git-backed storage model:
1. Initially, memory blocks lived only in Letta
2. Git storage added for version history and human-in-the-loop editing
3. Notes adapter added for OpenWebUI compatibility
4. Current refactor: Replace Letta+GitPython with Dolt+Agno

## Open Questions

1. **Dolt Migration Path**: How to migrate existing git history to Dolt tables?
2. **Schema Evolution**: Current frontmatter is untyped - will Dolt enforce schemas?
3. **Multi-User Support**: Current per-user repos → Dolt single database with user_id column?
4. **Diff Format**: Keep JSON files or move diffs into Dolt?
5. **Letta Removal**: Which sync patterns need replacement in Agno?
6. **Frontend API**: Keep dual API or consolidate to one?

## Related Research

- `thoughts/shared/research/2026-01-28-ralph-agno-architecture.md` - Agno architecture research
- `thoughts/shared/plans/2026-01-27-ralph-http-backend.md` - HTTP backend planning
