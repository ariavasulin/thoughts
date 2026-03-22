# ARI-82: Memory Blocks as Notes Architecture - Implementation Plan

## Overview

This plan implements a unified architecture where memory blocks are edited using OpenWebUI's NoteEditor component with Markdown + YAML frontmatter storage. The key insight is that NoteEditor already provides everything we need: TipTap rich text editing, autosave, undo/redo, and version history.

**Key Architecture Decision**: Rather than enhance BlockDetailModal with complex editing features, we adapt the existing NoteEditor infrastructure to work with our git-backed storage. This gives us:
- Rich text editing with TipTap (vs Textarea)
- Built-in undo/redo via editor history
- 200ms debounced autosave (already implemented)
- Version snapshots for navigation
- WebSocket collaboration (optional future use)

## Current State Analysis

### What Exists (ARI-80)

| Component | Implementation | Status |
|-----------|----------------|--------|
| Git Storage | Per-user repos with MD + frontmatter | ✅ Updated |
| Block Manager | Direct MD read/write, no TOML conversion | ✅ Updated |
| Diff Approval | PendingDiffStore with approve/reject workflow | ✅ Exists |
| Server API | Full CRUD + history + diff endpoints | ✅ Exists |
| Frontend | BlockCard + BlockDetailModal (simple view) | Needs enhancement |
| NoteEditor | TipTap rich text + autosave + versions | Ready to adapt |

### Storage Format (Implemented)

Memory blocks are stored as Markdown with YAML frontmatter:

```markdown
---
block: student
schema: college-essay/student
updated_at: 2026-01-13T10:00:00Z
---

Freeform markdown content...

## Profile
Background, aspirations, CliftonStrengths...

## Insights
Key observations from conversation...
```

Key files already updated:
- `src/youlab_server/storage/git.py` - `parse_frontmatter()`, `format_frontmatter()`, direct MD storage
- `src/youlab_server/storage/blocks.py` - Removed TOML conversion imports, works with MD directly
- `src/youlab_server/storage/convert.py` - **Deleted** (no longer needed)

### NoteEditor Capabilities (from research)

The NoteEditor at `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte` provides:

1. **TipTap Rich Text**: Full markdown editing with RichTextInput
2. **Autosave**: 200ms debounce via `changeDebounceHandler()`
3. **Undo/Redo**: TipTap's built-in `editor.can().undo()` / `editor.can().redo()`
4. **Versions**: `note.data.versions[]` with version navigation
5. **File Handling**: Image paste, file uploads, drag-and-drop
6. **AI Features**: Title generation, content enhancement

## Desired End State

After implementation:

1. **Memory Block Route**: `/memory/{block_label}` route using adapted NoteEditor
2. **Git-Backed Storage**: All edits go through YouLab API → git commits
3. **Diff Overlays**: Agent-proposed changes shown as inline suggestions
4. **Undo/Redo**: Navigate git history via UI buttons
5. **Autosave**: 200ms debounce with commit on each save
6. **Models & Folders**: Modules as models, auto-created chat folders

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend (Svelte)                         │
├─────────────────────────────────────────────────────────────────┤
│  MemoryBlockEditor.svelte (adapted from NoteEditor)             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  TipTap Editor (RichTextInput)                          │   │
│  │  - Markdown editing                                     │   │
│  │  - Undo/Redo (editor history)                          │   │
│  │  - Autosave (200ms debounce)                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  DiffOverlay (new)                                      │   │
│  │  - Show pending agent diffs inline                      │   │
│  │  - Approve/Reject buttons                               │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    YouLab Server (FastAPI)                       │
├─────────────────────────────────────────────────────────────────┤
│  Memory Blocks API                                               │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  GitNotesAdapter (new)                                    │ │
│  │  - Implements Notes-like API shape                        │ │
│  │  - Maps to UserBlockManager                               │ │
│  │  - Returns { content: { md, html, json }, versions }      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  UserBlockManager                                         │ │
│  │  - Git storage (MD + frontmatter)                         │ │
│  │  - Version history                                        │ │
│  │  - Pending diffs                                          │ │
│  │  - Letta sync                                             │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## What We're NOT Doing

1. **Not using TOML storage** - Migrated to MD + frontmatter
2. **Not enhancing BlockDetailModal** - Adapting NoteEditor instead
3. **Not implementing WebSocket collab** - Single-user editing for now
4. **Not implementing bidirectional Letta sync** - Git remains source of truth
5. **Not migrating existing Notes** - Memory blocks are a separate system

---

## Phase 1: Storage Format Migration

**Status**: ✅ Completed

Storage has been migrated from TOML to Markdown with YAML frontmatter.

### What Was Done

1. **GitUserStorage** (`src/youlab_server/storage/git.py`)
   - Added `parse_frontmatter()` and `format_frontmatter()` helpers
   - `write_block()` now creates frontmatter automatically
   - `read_block_body()` strips frontmatter for Letta sync

2. **UserBlockManager** (`src/youlab_server/storage/blocks.py`)
   - Removed TOML conversion imports
   - `update_block()` accepts raw markdown
   - `get_block_markdown()` returns full content with frontmatter
   - `get_block_body()` returns body only

3. **Deleted Files**
   - `src/youlab_server/storage/convert.py` - No longer needed
   - `tests/test_storage/test_convert.py` - Test file removed

### Verification

```bash
# Tests pass
make test-agent

# Storage format
cat .data/users/test-user/memory-blocks/student.md
# Expected:
# ---
# block: student
# schema: college-essay/student
# updated_at: 2026-01-14T...
# ---
#
# Content here...
```

---

## Phase 2: Memory Block Editor Route

**Status**: 🔲 Not Started

Create a dedicated route `/memory/{label}` that uses an adapted NoteEditor for memory block editing.

### Sub-Plan

See: `thoughts/shared/plans/2026-01-14-ARI-82-phase2-memory-editor-route.md`

### Overview

1. **Create MemoryBlockEditor.svelte** - Adapted from NoteEditor
   - Remove WebSocket collaboration (not needed)
   - Remove Notes-specific features (folders, sharing)
   - Add block label display and schema info
   - Wire to YouLab API instead of Notes API

2. **Create Memory API Functions** - Client-side API
   - `getMemoryBlock(userId, label)` → GET /users/{user_id}/blocks/{label}
   - `updateMemoryBlock(userId, label, content)` → PUT /users/{user_id}/blocks/{label}
   - `getMemoryBlockHistory(userId, label)` → GET /users/{user_id}/blocks/{label}/history

3. **Create Route** - `/memory/[label]/+page.svelte`
   - Load block data on mount
   - Initialize TipTap with markdown content
   - Wire autosave to API

### Success Criteria

- [ ] Route `/memory/student` loads and displays block content
- [ ] TipTap editor renders markdown correctly
- [ ] Edits trigger autosave after 200ms
- [ ] Undo/Redo buttons work
- [ ] Block label shown in header

---

## Phase 3: Git Storage Adapter

**Status**: 🔲 Not Started

Create an adapter that makes UserBlockManager compatible with the Notes component's expected data shape.

### Sub-Plan

See: `thoughts/shared/plans/2026-01-14-ARI-82-phase3-git-storage-adapter.md`

### Overview

The NoteEditor expects data in this shape:
```typescript
interface Note {
  id: string;
  title: string;
  data: {
    content: {
      json: object | null;  // TipTap JSON
      html: string;
      md: string;
    };
    versions: ContentVersion[];
    files: File[] | null;
  };
  write_access: boolean;
}
```

Our blocks have:
```python
# From UserBlockManager
block_markdown: str  # Full MD with frontmatter
block_body: str      # Body only
metadata: dict       # Frontmatter fields
history: list[VersionInfo]
```

### Changes Required

1. **GitNotesAdapter** (`src/youlab_server/server/notes_adapter.py`)
   - `get_block_as_note(user_id, label)` → Note-shaped response
   - `update_block_from_note(user_id, label, note_data)` → Commit
   - Maps git history to `versions[]` format

2. **API Endpoint** (`src/youlab_server/server/blocks.py`)
   - Add `GET /users/{user_id}/blocks/{label}/note` - Note-shaped response
   - Add `PUT /users/{user_id}/blocks/{label}/note` - Note-shaped update

### Success Criteria

- [ ] `/blocks/{label}/note` returns Note-compatible shape
- [ ] MemoryBlockEditor can consume the response directly
- [ ] Updates preserve frontmatter metadata
- [ ] Git commits created on save

---

## Phase 4: Diff Approval Integration

**Status**: 🔲 Not Started

Add inline diff visualization and approval UI to the memory editor.

### Sub-Plan

See: `thoughts/shared/plans/2026-01-14-ARI-82-phase4-diff-approval.md`

### Overview

When an agent proposes changes via `edit_memory_block` tool:
1. Diff is stored in `PendingDiffStore`
2. Editor shows inline overlay with proposed changes
3. User can approve (applies change) or reject (dismisses)

### Changes Required

1. **DiffOverlay.svelte** (new component)
   - Render proposed changes as inline additions/deletions
   - Approve/Reject buttons per diff
   - Batch approve all button

2. **API Integration**
   - Poll for pending diffs on editor mount
   - `POST /blocks/{label}/diffs/{diff_id}/approve`
   - `POST /blocks/{label}/diffs/{diff_id}/reject`

3. **Visual Design**
   - Green background for additions
   - Red strikethrough for deletions
   - Floating action buttons

### Success Criteria

- [ ] Pending diffs shown as inline overlays
- [ ] Approve applies the change and commits
- [ ] Reject dismisses the diff
- [ ] Editor content updates after approval

---

## Phase 5: Models and Folders

**Status**: 🔲 Not Started

Create course modules as OpenWebUI models and auto-organize chats into folders.

### Sub-Plan

See: `thoughts/shared/plans/2026-01-14-ARI-82-phase5-models-folders.md`

### Overview

1. **Module → Model**: Each course module becomes a visible model in OpenWebUI
   - `base_model_id = "youlab-pipe"`
   - Custom system prompt from module config
   - `meta.type = "module"`

2. **Background Agent → Hidden Model**: Hidden from model selector
   - `meta.hidden = true`
   - `meta.type = "background_agent"`

3. **Auto-Folders**: Create folder per module, auto-assign new chats

### Changes Required

1. **OpenWebUIClient** (`src/youlab_server/server/sync/openwebui_client.py`)
   - `create_module_model(module_config)`
   - `create_background_agent_model(agent_config)`
   - `ensure_model_folder(model_id, user_id)`

2. **Course Enrollment Flow**
   - On enrollment, create models for all modules
   - Create folders for each module
   - Background agents created as hidden

### Success Criteria

- [ ] Module models appear in OpenWebUI model selector
- [ ] Background agents don't appear in selector
- [ ] Folders auto-created per module
- [ ] New chats in module auto-assigned to folder

---

## Testing Strategy

### Unit Tests

| Test File | Coverage |
|-----------|----------|
| `tests/test_storage/test_blocks.py` | MD frontmatter parsing, body extraction |
| `tests/test_storage/test_git.py` | Version history, restore, diff |
| `tests/test_server/test_blocks_api.py` | Note-shaped API responses |

### Integration Tests

| Test | Description |
|------|-------------|
| `test_editor_round_trip` | Edit in UI → API → Git → Reload |
| `test_diff_approval_flow` | Agent proposes → User approves → Committed |
| `test_model_sync` | Enrollment → Models created in OpenWebUI |

### Manual Testing Checklist

- [ ] Edit block at `/memory/student`, verify autosave
- [ ] Click undo, verify previous content loads
- [ ] Agent proposes change, verify diff overlay appears
- [ ] Approve diff, verify content updates
- [ ] Enroll in course, verify module appears as model

---

## Performance Considerations

1. **Autosave Debounce**: 200ms prevents excessive git commits
2. **Lazy Loading**: Don't load TipTap until editor focused
3. **Version Pagination**: Limit history to 20 versions initially
4. **Model Sync**: Only on enrollment, not on every request

---

## Migration Notes

### Existing Blocks

Blocks created by ARI-80 are already in MD format. No migration needed.

### convert.py Removal

The TOML↔MD conversion module has been deleted. Any code importing from `youlab_server.storage.convert` will fail. Known impact:
- Removed from `blocks.py` imports
- Test file `test_convert.py` deleted

---

## References

- Research: `thoughts/searchable/shared/research/2026-01-13-ARI-82-*.md`
- Linear: [ARI-82](https://linear.app/ariav/issue/ARI-82)
- Related: ARI-79 (UI+Modules), ARI-80 (Memory MVP), ARI-81 (Demo-Ready)
- NoteEditor: `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte`
- Storage: `src/youlab_server/storage/git.py`, `src/youlab_server/storage/blocks.py`
