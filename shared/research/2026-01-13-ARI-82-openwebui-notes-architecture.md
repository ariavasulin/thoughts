---
date: 2026-01-13T17:05:14+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "OpenWebUI Notes Architecture for Memory Block Reuse"
tags: [research, codebase, notes, memory-blocks, openwebui, svelte, components]
status: complete
last_updated: 2026-01-13
last_updated_by: ariasulin
---

# Research: OpenWebUI Notes Architecture

**Date**: 2026-01-13T17:05:14+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Research OpenWebUI's Notes architecture comprehensively to understand how to reuse Notes components for Memory Blocks. Focus on:
1. Notes components (editor, list view, modal)
2. Notes API and data model
3. Storage and versioning
4. Autosave implementation
5. Diff/version history UI

## Summary

OpenWebUI's Notes system is a full-featured collaborative note-taking application with rich text editing, real-time collaboration, AI enhancement, file attachments, and access control. The architecture is significantly more complex than what Memory Blocks require.

**Key Finding**: The Notes components are tightly coupled to their specific use case (collaborative note-taking with AI). The existing BlockDetailModal for Memory Blocks already has a more appropriate architecture with git-based versioning, unified diff view, and approve/reject workflow. Rather than adapting Notes components, the better path is to enhance the existing Memory Block components with specific features from Notes (autosave debouncing, responsive layout).

## Detailed Findings

### 1. Notes Component Architecture

**File Structure:**
```
OpenWebUI/open-webui/src/lib/components/notes/
├── Notes.svelte                    # Main list view with search, filtering
├── NoteEditor.svelte               # Primary editor (1404 lines)
├── NotePanel.svelte                # Drawer wrapper for side panel
├── AIMenu.svelte                   # AI features menu (edit, chat, generate)
├── RecordMenu.svelte               # Voice/file recording menu
├── utils.ts                        # PDF download utilities
├── NoteEditor/
│   ├── Chat.svelte                 # Chat interface sidebar
│   ├── Controls.svelte             # Settings panel (files, metadata)
│   └── Chat/
│       ├── Message.svelte          # Individual chat message
│       └── Messages.svelte         # Messages container
└── Notes/
    └── NoteMenu.svelte             # Context menu for notes in list
```

**Key Components:**

| Component | Purpose | Reusability for Memory Blocks |
|-----------|---------|-------------------------------|
| `Notes.svelte` | List view with search/filter | Low - tied to notes data model |
| `NoteEditor.svelte` | Rich text editing | Low - includes AI, chat, files |
| `NotePanel.svelte` | Drawer wrapper (uses paneforge) | Medium - layout pattern reusable |
| `RichTextInput.svelte` | TipTap editor wrapper | Medium - if rich text needed |

**NoteEditor.svelte Analysis** (OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte):
- 1404 lines - very large component
- Uses TipTap (ProseMirror-based) rich text editor via `RichTextInput.svelte`
- Includes collaboration via Socket.io and Y.js CRDT
- File upload/attachment handling
- AI enhancement (streaming responses)
- Voice recording integration
- Access control modal
- Two-pane layout with resizable panels (paneforge)

### 2. Notes API and Data Model

**REST Endpoints** (`/api/v1/notes`):

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/` | GET | List notes (paginated, 60/page) |
| `/search` | GET | Search with filters (view, permission, sort) |
| `/create` | POST | Create new note |
| `/{id}` | GET | Get note by ID (includes write_access flag) |
| `/{id}/update` | POST | Update note |
| `/{id}/delete` | DELETE | Delete note |

**Data Model** (backend/open_webui/models/notes.py):

```python
class Note(Base):
    __tablename__ = "note"

    id: str              # UUID
    user_id: str         # Owner
    title: str           # Note title
    data: dict           # JSON - content, versions, files
    meta: dict           # Additional metadata
    access_control: dict # Permission structure
    created_at: int      # Nanoseconds epoch
    updated_at: int      # Nanoseconds epoch
```

**Data Field Structure:**
```json
{
  "content": {
    "json": {...},      // TipTap JSON document
    "html": "...",      // Rendered HTML
    "md": "..."         // Markdown representation
  },
  "versions": [...],    // In-memory version snapshots
  "files": [...]        // Attached files
}
```

**Access Control Structure:**
```json
{
  "read": { "group_ids": ["..."] },
  "write": { "group_ids": ["..."] }
}
```

### 3. Storage and Versioning

**Notes Versioning** (NoteEditor.svelte:206-382):

Notes use an in-memory version system stored in `data.versions` array:

```typescript
function insertNoteVersion(note) {
    const current = {
        json: note.data.content.json,
        html: note.data.content.html,
        md: note.data.content.md
    };
    const lastVersion = note.data.versions?.at(-1);

    if (!lastVersion || !areContentsEqual(lastVersion, current)) {
        note.data.versions = (note.data.versions ?? []).concat(current);
        return true;
    }
    return false;
}
```

- Versions are snapshots stored in the note's `data.versions` array
- Inserted when AI content is applied or before major changes
- Navigation functions exist (`versionNavigateHandler`) but are **commented out in UI** (lines 969, 980)
- No git-based versioning, no persistent history

**Memory Blocks Versioning** (BlockDetailModal.svelte):

Memory Blocks use git-based versioning via YouLab server:

```typescript
// API functions
getBlockHistory()     // Git log for block
restoreVersion()      // Git checkout
getBlockDiffs()       // Pending changes
approveDiff()         // Apply and commit
rejectDiff()          // Discard change
```

- Full git history with SHA, author, timestamp, message
- Restore to any previous version
- Pending diffs with approve/reject workflow
- Unified diff visualization

### 4. Autosave Implementation

**Notes Autosave** (NoteEditor.svelte:178-196):

```typescript
let debounceTimeout: NodeJS.Timeout | null = null;

const changeDebounceHandler = () => {
    if (debounceTimeout) {
        clearTimeout(debounceTimeout);
    }

    debounceTimeout = setTimeout(async () => {
        const res = await updateNoteById(localStorage.token, id, {
            title: note?.title === '' ? $i18n.t('Untitled') : note.title,
            data: { files: files },
            access_control: note?.access_control
        }).catch((e) => { toast.error(`${e}`); });
    }, 200);  // 200ms debounce
};
```

**Autosave Triggers:**
- Title field blur (line 928)
- File uploads (lines 464, 547)
- Access control changes (line 852)
- Title generation completion (line 298)
- Controls panel updates (line 1398)

**Key Characteristics:**
- 200ms debounce timeout
- Server-side persistence only (no localStorage drafts)
- WebSocket sync for multi-user collaboration
- Rich text content saved via Y.js CRDT (collaborative editing)

**Memory Blocks Current Behavior:**
- No autosave - manual Save button only
- Full content sent on each save
- Git commit created on each save

### 5. Diff/Version History UI

**Notes Version UI:**
- Undo/redo buttons use TipTap's built-in history (lines 968-985)
- Version navigation code exists but is **commented out**
- No visible version history sidebar
- No diff visualization

**Memory Blocks Version UI** (BlockDetailModal.svelte):

**Version History Sidebar:**
```svelte
{#if showHistory}
    <div class="w-64 border-l pl-4">
        <h3>Version History</h3>
        {#each $blockHistory as version (version.sha)}
            <div>
                <div>{version.message}</div>
                <div>{version.author} · {formatDate(version.timestamp)}</div>
                {#if !version.isCurrent}
                    <button on:click={() => restore(version.sha)}>Restore</button>
                {:else}
                    <span>Current</span>
                {/if}
            </div>
        {/each}
    </div>
{/if}
```

**Unified Diff View:**
```svelte
{#each diffLines as line}
    {#if line.type === 'deletion'}
        <div class="bg-red-100">− {line.content}</div>
    {:else if line.type === 'addition'}
        <div class="bg-green-100">+ {line.content}</div>
        {#if line.isLastInDiff}
            <div>
                <span class="italic">{line.diff.reasoning}</span>
                <button on:click={() => handleApprove(line.diff.id)}>Approve</button>
                <button on:click={() => handleReject(line.diff.id)}>Reject</button>
            </div>
        {/if}
    {:else}
        <div>{line.lineNum} {line.content}</div>
    {/if}
{/each}
```

### 6. Real-time Collaboration

**Notes WebSocket Events:**

| Event | Direction | Purpose |
|-------|-----------|---------|
| `join-note` | Client→Server | Join room for note updates |
| `note-events` | Server→Client | Broadcast note changes |
| `ydoc:document:state` | Bidirectional | Y.js CRDT sync |

**Implementation** (socket/main.py + NoteEditor.svelte):
- Socket.io rooms per note (`note:{id}`)
- Y.js for CRDT-based collaborative editing
- Automatic merge of concurrent edits

**Memory Blocks:**
- No real-time collaboration (single-user focused)
- Agent-driven edits create pending diffs
- Human reviews and approves/rejects

## Code References

**Notes Components:**
- `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte:178-196` - Autosave debounce
- `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte:206-382` - Version management
- `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte:968-985` - Undo/redo (version nav commented)
- `OpenWebUI/open-webui/src/lib/components/notes/Notes.svelte` - List view

**Notes API:**
- `OpenWebUI/open-webui/backend/open_webui/routers/notes.py` - REST endpoints
- `OpenWebUI/open-webui/backend/open_webui/models/notes.py` - Data model
- `OpenWebUI/open-webui/backend/open_webui/socket/main.py:380` - WebSocket handlers
- `OpenWebUI/open-webui/src/lib/apis/notes/index.ts` - Frontend API client

**Memory Blocks:**
- `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte` - Version/diff UI
- `OpenWebUI/open-webui/src/lib/components/you/BlockCard.svelte` - Card component
- `OpenWebUI/open-webui/src/lib/apis/memory/index.ts` - API client
- `src/youlab_server/server/blocks.py` - Backend API

**Shared Components:**
- `OpenWebUI/open-webui/src/lib/components/common/RichTextInput.svelte` - TipTap wrapper
- `OpenWebUI/open-webui/src/lib/components/common/Modal.svelte` - Modal component
- `OpenWebUI/open-webui/src/lib/components/common/Textarea.svelte` - Text input

## Gap Analysis: Notes vs Memory Blocks

| Feature | Notes | Memory Blocks | Gap |
|---------|-------|---------------|-----|
| **Editor** | TipTap rich text | Plain textarea | Different needs - MB may not need rich text |
| **Versioning** | In-memory array | Git-based | MB has better versioning already |
| **Diff UI** | None visible | Unified diff view | MB has better diff already |
| **Autosave** | 200ms debounce | Manual save | **Autosave needed for MB** |
| **Collaboration** | Socket.io + Y.js | N/A | Not needed for MB |
| **AI Features** | Enhance, chat, generate | Agent-driven edits | Different paradigm |
| **File Attachments** | Full support | N/A | Not needed for MB |
| **History Sidebar** | Commented out | Working | MB has this already |
| **Approve/Reject** | N/A | Full workflow | MB has this already |

## Recommendations for ARI-82

Based on this research, here are the components and patterns worth adopting:

### 1. Autosave Pattern (High Value)
Copy the debounce handler pattern from NoteEditor.svelte:
```typescript
let debounceTimeout: NodeJS.Timeout | null = null;
const changeDebounceHandler = () => {
    if (debounceTimeout) clearTimeout(debounceTimeout);
    debounceTimeout = setTimeout(async () => {
        await saveBlock();
    }, 200);
};
```

### 2. Panel Layout (Medium Value)
Notes uses paneforge for resizable two-pane layout. Could be useful for side-by-side diff view.

### 3. Shared Components (Available)
These existing components can be reused:
- `Modal.svelte` - Already in use
- `Textarea.svelte` - Already in use
- `Spinner.svelte` - Already in use
- `Tooltip.svelte` - For UI hints

### 4. What NOT to Adopt
- TipTap/RichTextInput - Overkill for structured TOML/Markdown
- Socket.io collaboration - Not needed for single-user blocks
- AI enhancement features - Agents handle this differently
- File attachments - Not relevant for memory blocks

### 5. Enhance Existing BlockDetailModal
The current BlockDetailModal already has:
- Version history sidebar
- Unified diff view
- Approve/reject workflow
- Edit mode toggle

What to add:
- Autosave debounce (from Notes)
- Better responsive layout
- Loading states during operations

## Open Questions

1. Should Memory Blocks support rich text (Markdown rendered), or stay with plain text/TOML?
2. Should autosave be opt-in or default behavior?
3. Should there be a "draft" state before commits, or immediate commits on save?

## Related Research

- `thoughts/shared/plans/` - Check for existing Memory Block enhancement plans
