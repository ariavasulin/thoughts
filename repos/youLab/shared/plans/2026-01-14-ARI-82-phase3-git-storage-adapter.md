# ARI-82 Phase 3: Git Storage Adapter Implementation Plan

## Overview

Create a Notes-compatible API adapter that allows NoteEditor to work with our git-backed memory block storage. The adapter translates between OpenWebUI's Notes API format and our markdown + YAML frontmatter storage.

## Current State Analysis

### What Exists (Phases 1-2 Complete)

| Component | Implementation | Location |
|-----------|----------------|----------|
| Git Storage | Markdown with YAML frontmatter | `src/youlab_server/storage/git.py:25-66` |
| Block Manager | Body/metadata accessors, Letta sync | `src/youlab_server/storage/blocks.py:19-394` |
| Blocks API | CRUD + history + diffs | `src/youlab_server/server/blocks.py` |
| Frontmatter Utils | `parse_frontmatter()`, `format_frontmatter()` | `src/youlab_server/storage/git.py:25-66` |

### NoteEditor API Contract

NoteEditor expects these endpoints (from `OpenWebUI/open-webui/src/lib/apis/notes/index.ts`):

```typescript
GET  /notes/           → list notes
GET  /notes/{id}       → get note by id (returns NoteResponse)
POST /notes/create     → create new note
POST /notes/{id}/update → update note
DELETE /notes/{id}/delete → delete note
```

Note data structure:
```typescript
{
  id: string,
  user_id: string,
  title: string,
  data: {
    content: { json: TipTapJSON | null, html: string, md: string },
    versions: Array<{ json, html, md }>,
    files: Array<FileItem> | null
  },
  meta: object | null,
  access_control: object | null,
  created_at: number,  // nanoseconds epoch
  updated_at: number,  // nanoseconds epoch
  write_access: boolean  // response only
}
```

## Desired End State

After implementation:

1. **New API router** at `/api/you/notes/` provides Notes-compatible endpoints for memory blocks
2. **NoteEditor** can load/save memory blocks via these endpoints
3. **Git history** populates `data.versions[]` for in-editor undo/redo
4. **Autosave** works with 200ms debounce (frontend handles debounce, backend commits)
5. **Letta sync** happens on every save

### Verification

```bash
# List memory blocks as notes
curl -s $YOULAB_URL/api/you/notes/ -H "Authorization: Bearer $TOKEN"
# Expected: Array of note objects

# Get single block as note
curl -s $YOULAB_URL/api/you/notes/student -H "Authorization: Bearer $TOKEN"
# Expected: { id: "student", title: "Student Profile", data: { content: {...}, versions: [...] }, ... }

# Update block via notes API
curl -X POST $YOULAB_URL/api/you/notes/student/update \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "Student Profile", "data": {"content": {"md": "Updated content"}}}'
# Expected: Updated note object
```

## What We're NOT Doing

1. **Not implementing WebSocket collaboration** - Memory blocks don't need real-time collab
2. **Not storing TipTap JSON** - We store markdown, return `json: null`, let TipTap parse HTML
3. **Not implementing file uploads** - Memory blocks are text-only for now
4. **Not creating notes from scratch** - Blocks are created via course config, not user action
5. **Not implementing search** - List endpoint is sufficient for memory blocks
6. **Not modifying OpenWebUI's notes backend** - We create a parallel adapter

## Implementation Approach

Create a new FastAPI router that:
1. Maps memory block labels to note IDs (1:1, e.g., `student` → `student`)
2. Converts git-stored markdown to NoteEditor's expected format
3. Extracts versions from git history for `data.versions[]`
4. Uses markdown-it or similar for MD → HTML conversion
5. Syncs to Letta after saves

---

## Phase 3.1: Notes Adapter Router

### Overview

Create the core adapter router with Notes-compatible endpoints.

### Changes Required:

#### 1. Create Notes Adapter Router

**File**: `src/youlab_server/server/notes_adapter.py` (new file)

```python
"""Notes API adapter for git-backed memory blocks.

Provides Notes-compatible endpoints that NoteEditor can use to
edit memory blocks stored in git with markdown + YAML frontmatter.
"""

from __future__ import annotations

import time
from typing import Annotated, Any

import markdown
import structlog
from fastapi import APIRouter, Depends, HTTPException, Request
from pydantic import BaseModel

from youlab_server.server.users import get_current_user, get_storage_manager
from youlab_server.storage.blocks import UserBlockManager
from youlab_server.storage.git import GitUserStorageManager, parse_frontmatter

log = structlog.get_logger()
router = APIRouter(prefix="/you/notes", tags=["notes-adapter"])

# Type aliases
StorageDep = Annotated[GitUserStorageManager, Depends(get_storage_manager)]


# =============================================================================
# Pydantic Models (Notes API compatible)
# =============================================================================


class NoteContent(BaseModel):
    """TipTap content structure."""

    json: dict | None = None  # Always null - we don't store TipTap JSON
    html: str = ""
    md: str = ""


class NoteVersion(BaseModel):
    """A version snapshot for undo/redo."""

    json: dict | None = None
    html: str = ""
    md: str = ""
    sha: str = ""  # Git commit SHA (extension)
    message: str = ""  # Commit message (extension)
    timestamp: str = ""  # ISO timestamp (extension)


class NoteData(BaseModel):
    """Note data payload."""

    content: NoteContent
    versions: list[NoteVersion] = []
    files: list[dict] | None = None


class NoteModel(BaseModel):
    """Full note model matching OpenWebUI's NoteModel."""

    id: str
    user_id: str
    title: str
    data: NoteData
    meta: dict | None = None
    access_control: dict | None = None
    created_at: int  # nanoseconds epoch
    updated_at: int  # nanoseconds epoch


class NoteResponse(NoteModel):
    """Note response with write_access flag."""

    write_access: bool = True


class NoteItemResponse(BaseModel):
    """Summary for list endpoint."""

    id: str
    title: str
    data: dict | None = None
    updated_at: int
    created_at: int


class NoteForm(BaseModel):
    """Form for create/update."""

    title: str
    data: dict | None = None
    meta: dict | None = None
    access_control: dict | None = None


# =============================================================================
# Helper Functions
# =============================================================================


def _md_to_html(md_content: str) -> str:
    """Convert markdown to HTML."""
    return markdown.markdown(
        md_content,
        extensions=["fenced_code", "tables", "nl2br"],
    )


def _block_to_note(
    label: str,
    user_id: str,
    content: str,
    metadata: dict[str, Any],
    versions: list[dict[str, Any]],
) -> NoteResponse:
    """Convert a memory block to NoteResponse format."""
    _, body = parse_frontmatter(content)
    html = _md_to_html(body)

    # Convert git versions to NoteVersion format
    note_versions = []
    for v in versions:
        v_content = v.get("content", "")
        _, v_body = parse_frontmatter(v_content) if v_content else ("", "")
        note_versions.append(
            NoteVersion(
                json=None,
                html=_md_to_html(v_body) if v_body else "",
                md=v_body,
                sha=v.get("sha", ""),
                message=v.get("message", ""),
                timestamp=v.get("timestamp", ""),
            )
        )

    # Extract timestamps from metadata
    updated_at_str = metadata.get("updated_at", "")
    # Convert ISO to nanoseconds epoch (NoteEditor expects nanoseconds)
    try:
        from datetime import datetime

        if updated_at_str:
            dt = datetime.fromisoformat(updated_at_str.replace("Z", "+00:00"))
            updated_at = int(dt.timestamp() * 1_000_000_000)
        else:
            updated_at = int(time.time() * 1_000_000_000)
    except Exception:
        updated_at = int(time.time() * 1_000_000_000)

    # Use title from metadata or generate from label
    title = metadata.get("title", label.replace("_", " ").title())

    return NoteResponse(
        id=label,
        user_id=user_id,
        title=title,
        data=NoteData(
            content=NoteContent(json=None, html=html, md=body),
            versions=note_versions,
            files=None,
        ),
        meta=metadata.get("meta"),
        access_control=metadata.get("access_control"),
        created_at=updated_at,  # We don't track created_at separately
        updated_at=updated_at,
        write_access=True,
    )


# =============================================================================
# Endpoints
# =============================================================================


@router.get("/", response_model=list[NoteItemResponse])
async def list_notes(
    request: Request,
    storage: StorageDep,
) -> list[NoteItemResponse]:
    """List all memory blocks as notes."""
    user = await get_current_user(request)
    user_storage = storage.get(user.id)

    if not user_storage.exists:
        return []

    manager = UserBlockManager(user.id, user_storage)
    labels = manager.list_blocks()

    notes = []
    for label in labels:
        metadata = manager.get_block_metadata(label) or {}
        updated_at_str = metadata.get("updated_at", "")

        try:
            from datetime import datetime

            if updated_at_str:
                dt = datetime.fromisoformat(updated_at_str.replace("Z", "+00:00"))
                updated_at = int(dt.timestamp() * 1_000_000_000)
            else:
                updated_at = int(time.time() * 1_000_000_000)
        except Exception:
            updated_at = int(time.time() * 1_000_000_000)

        title = metadata.get("title", label.replace("_", " ").title())

        notes.append(
            NoteItemResponse(
                id=label,
                title=title,
                data=None,
                updated_at=updated_at,
                created_at=updated_at,
            )
        )

    return notes


@router.get("/{note_id}", response_model=NoteResponse)
async def get_note_by_id(
    request: Request,
    note_id: str,
    storage: StorageDep,
) -> NoteResponse:
    """Get a memory block as a note."""
    user = await get_current_user(request)
    user_storage = storage.get(user.id)

    if not user_storage.exists:
        raise HTTPException(status_code=404, detail="User storage not found")

    manager = UserBlockManager(user.id, user_storage)

    content = manager.get_block_markdown(note_id)
    if content is None:
        raise HTTPException(status_code=404, detail=f"Block {note_id} not found")

    metadata = manager.get_block_metadata(note_id) or {}

    # Get version history with content
    history = manager.get_history(note_id, limit=20)
    versions_with_content = []
    for v in history:
        v_content = manager.get_version(note_id, v["sha"])
        versions_with_content.append({**v, "content": v_content or ""})

    return _block_to_note(
        label=note_id,
        user_id=user.id,
        content=content,
        metadata=metadata,
        versions=versions_with_content,
    )


@router.post("/{note_id}/update", response_model=NoteResponse)
async def update_note_by_id(
    request: Request,
    note_id: str,
    form_data: NoteForm,
    storage: StorageDep,
) -> NoteResponse:
    """Update a memory block via notes API."""
    user = await get_current_user(request)
    user_storage = storage.get(user.id)

    if not user_storage.exists:
        raise HTTPException(status_code=404, detail="User storage not found")

    manager = UserBlockManager(user.id, user_storage)

    # Check block exists
    if manager.get_block_markdown(note_id) is None:
        raise HTTPException(status_code=404, detail=f"Block {note_id} not found")

    # Extract markdown content from form_data
    md_content = ""
    if form_data.data and "content" in form_data.data:
        content_data = form_data.data["content"]
        if isinstance(content_data, dict):
            md_content = content_data.get("md", "")

    if not md_content:
        raise HTTPException(status_code=400, detail="No markdown content provided")

    # Update the block (this handles frontmatter and Letta sync)
    manager.update_block(
        label=note_id,
        content=md_content,
        message=f"Update {note_id}",
        sync_to_letta=True,
    )

    # Return updated note
    content = manager.get_block_markdown(note_id) or ""
    metadata = manager.get_block_metadata(note_id) or {}

    history = manager.get_history(note_id, limit=20)
    versions_with_content = []
    for v in history:
        v_content = manager.get_version(note_id, v["sha"])
        versions_with_content.append({**v, "content": v_content or ""})

    return _block_to_note(
        label=note_id,
        user_id=user.id,
        content=content,
        metadata=metadata,
        versions=versions_with_content,
    )
```

#### 2. Register Router in Main App

**File**: `src/youlab_server/server/main.py`

Add import and include router:

```python
# After existing imports
from youlab_server.server.notes_adapter import router as notes_adapter_router

# In create_app() or where routers are registered
app.include_router(notes_adapter_router, prefix="/api")
```

#### 3. Add markdown dependency

**File**: `pyproject.toml`

Add to dependencies:

```toml
dependencies = [
    # ... existing deps
    "markdown>=3.5",
]
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] List endpoint returns blocks:
  ```bash
  curl -s $YOULAB_URL/api/you/notes/ -H "Authorization: Bearer $TOKEN" | jq length
  # Expected: > 0
  ```
- [ ] Get endpoint returns note format:
  ```bash
  curl -s $YOULAB_URL/api/you/notes/student -H "Authorization: Bearer $TOKEN" | jq '.data.content.md'
  # Expected: non-empty string
  ```

#### Manual Verification:
- [ ] Response matches NoteEditor expected format
- [ ] Versions array populated from git history

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 3.2: Title Management

### Overview

Add title support to memory blocks via frontmatter, so blocks have meaningful titles in NoteEditor.

### Changes Required:

#### 1. Update write_block to preserve/set title

**File**: `src/youlab_server/storage/git.py`

Update `write_block` method to handle title in frontmatter:

```python
def write_block(
    self,
    label: str,
    content: str,
    message: str | None = None,
    author: str = "user",
    schema: str | None = None,
    title: str | None = None,  # Add title parameter
) -> str:
    """Write a memory block and commit."""
    # ... existing code ...

    # Build metadata
    metadata: dict[str, Any] = {
        "block": label,
        **existing_meta,
        "updated_at": datetime.now(UTC).isoformat(),
    }
    if schema:
        metadata["schema"] = schema
    if title:
        metadata["title"] = title
    elif "title" not in metadata:
        # Generate default title from label
        metadata["title"] = label.replace("_", " ").title()

    # ... rest of method ...
```

#### 2. Update notes adapter to save title

**File**: `src/youlab_server/server/notes_adapter.py`

Update `update_note_by_id` to pass title:

```python
# In update_note_by_id, after extracting md_content:

# Update the block with title
manager.storage.write_block(
    label=note_id,
    content=md_content,
    message=f"Update {note_id}",
    author="user",
    title=form_data.title,
)

# Sync to Letta
if manager.letta:
    manager._sync_block_to_letta(note_id)
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Title persists in frontmatter:
  ```bash
  curl -X POST $YOULAB_URL/api/you/notes/student/update \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"title": "My Custom Title", "data": {"content": {"md": "content"}}}'

  cat .data/users/$USER_ID/memory-blocks/student.md | head -5
  # Expected: title: My Custom Title in frontmatter
  ```

#### Manual Verification:
- [ ] Title survives round-trip (save → reload)

---

## Phase 3.3: Frontend API Client

### Overview

Create a TypeScript API client for the notes adapter, so NoteEditor can call our endpoints.

### Changes Required:

#### 1. Create Memory Notes API Client

**File**: `OpenWebUI/open-webui/src/lib/apis/memory/notes.ts` (new file)

```typescript
/**
 * Notes-compatible API client for YouLab memory blocks.
 *
 * Uses the same interface as OpenWebUI's notes API but routes
 * to YouLab's git-backed storage adapter.
 */

import { YOULAB_API_BASE_URL } from '$lib/constants';

export type NoteContent = {
    json: object | null;
    html: string;
    md: string;
};

export type NoteVersion = {
    json: object | null;
    html: string;
    md: string;
    sha: string;
    message: string;
    timestamp: string;
};

export type NoteData = {
    content: NoteContent;
    versions: NoteVersion[];
    files: object[] | null;
};

export type NoteModel = {
    id: string;
    user_id: string;
    title: string;
    data: NoteData;
    meta: object | null;
    access_control: object | null;
    created_at: number;
    updated_at: number;
    write_access?: boolean;
};

export type NoteItem = {
    id: string;
    title: string;
    data: object | null;
    updated_at: number;
    created_at: number;
};

export type NoteForm = {
    title: string;
    data?: {
        content?: {
            md?: string;
        };
        files?: object[];
    };
    meta?: object | null;
    access_control?: object | null;
};

/**
 * Get all memory blocks as notes.
 */
export async function getMemoryNotes(token: string): Promise<NoteItem[]> {
    const res = await fetch(`${YOULAB_API_BASE_URL}/you/notes/`, {
        method: 'GET',
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
        },
    });

    if (!res.ok) {
        const error = await res.json().catch(() => ({ detail: res.statusText }));
        throw new Error(error.detail || 'Failed to fetch memory notes');
    }

    return res.json();
}

/**
 * Get a single memory block as a note.
 */
export async function getMemoryNoteById(
    token: string,
    noteId: string
): Promise<NoteModel> {
    const res = await fetch(`${YOULAB_API_BASE_URL}/you/notes/${noteId}`, {
        method: 'GET',
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
        },
    });

    if (!res.ok) {
        const error = await res.json().catch(() => ({ detail: res.statusText }));
        throw new Error(error.detail || 'Note not found');
    }

    return res.json();
}

/**
 * Update a memory block via notes API.
 */
export async function updateMemoryNoteById(
    token: string,
    noteId: string,
    note: NoteForm
): Promise<NoteModel> {
    const res = await fetch(`${YOULAB_API_BASE_URL}/you/notes/${noteId}/update`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
        },
        body: JSON.stringify(note),
    });

    if (!res.ok) {
        const error = await res.json().catch(() => ({ detail: res.statusText }));
        throw new Error(error.detail || 'Failed to update note');
    }

    return res.json();
}
```

#### 2. Add YOULAB_API_BASE_URL constant (if not exists)

**File**: `OpenWebUI/open-webui/src/lib/constants.ts`

```typescript
// Add if not present
export const YOULAB_API_BASE_URL =
    import.meta.env.VITE_YOULAB_API_URL || 'http://localhost:8100';
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors
- [ ] API client functions are exported correctly

#### Manual Verification:
- [ ] Import works in Svelte component

---

## Phase 3.4: NoteEditor Integration

### Overview

Create a memory block editor route that uses NoteEditor with our notes adapter.

### Changes Required:

#### 1. Create Memory Block Editor Route

**File**: `OpenWebUI/open-webui/src/routes/(app)/you/blocks/[label]/+page.svelte` (new file)

```svelte
<script lang="ts">
    import { page } from '$app/stores';
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import { toast } from 'svelte-sonner';

    import NoteEditor from '$lib/components/notes/NoteEditor.svelte';
    import { getMemoryNoteById, updateMemoryNoteById } from '$lib/apis/memory/notes';
    import type { NoteModel, NoteForm } from '$lib/apis/memory/notes';

    // Get block label from URL
    $: label = $page.params.label;

    let note: NoteModel | null = null;
    let loading = true;
    let error: string | null = null;

    // Custom save handler that uses our adapter
    async function handleSave(updatedNote: NoteForm) {
        if (!label) return;

        try {
            note = await updateMemoryNoteById(
                localStorage.token,
                label,
                updatedNote
            );
        } catch (e) {
            toast.error(`Failed to save: ${e}`);
            throw e;
        }
    }

    onMount(async () => {
        if (!label) {
            goto('/you');
            return;
        }

        try {
            note = await getMemoryNoteById(localStorage.token, label);
        } catch (e) {
            error = `${e}`;
            toast.error(error);
        } finally {
            loading = false;
        }
    });
</script>

<svelte:head>
    <title>{note?.title || 'Memory Block'}</title>
</svelte:head>

{#if loading}
    <div class="flex items-center justify-center h-full">
        <span class="loading loading-spinner loading-lg"></span>
    </div>
{:else if error}
    <div class="flex flex-col items-center justify-center h-full gap-4">
        <p class="text-error">{error}</p>
        <button class="btn btn-primary" on:click={() => goto('/you')}>
            Back to Dashboard
        </button>
    </div>
{:else if note}
    <!--
        Note: This uses NoteEditor but we need to:
        1. Override the save behavior to use our adapter
        2. Disable WebSocket collaboration
        3. Handle version navigation from git history

        This may require creating a MemoryBlockEditor wrapper component
        that adapts NoteEditor for our use case.
    -->
    <div class="h-full">
        <!-- TODO: Integrate NoteEditor with custom save handler -->
        <p>Memory Block Editor for: {label}</p>
        <pre>{JSON.stringify(note.data.content, null, 2)}</pre>
    </div>
{/if}
```

**Note**: Full NoteEditor integration requires additional work to:
1. Override save behavior (disable WebSocket, use HTTP)
2. Handle our `versions[]` format (includes git SHA)
3. Potentially create a `MemoryBlockEditor.svelte` wrapper

### Success Criteria:

#### Automated Verification:
- [ ] Route loads without errors
- [ ] Note data fetched from adapter

#### Manual Verification:
- [ ] Navigate to `/you/blocks/student`
- [ ] See block content displayed

---

## Phase 3.5: Full NoteEditor Adaptation

### Overview

Create a wrapper component that adapts NoteEditor for memory blocks, handling:
- HTTP-based saving (no WebSocket)
- Git-based version navigation
- 200ms autosave debounce

### Changes Required:

#### 1. Create MemoryBlockEditor Component

**File**: `OpenWebUI/open-webui/src/lib/components/you/MemoryBlockEditor.svelte` (new file)

This component wraps NoteEditor and:
1. Overrides `changeDebounceHandler` to call our adapter
2. Implements git-based undo/redo using `versions[]` SHA
3. Disables WebSocket collaboration features

```svelte
<script lang="ts">
    import { getContext, onDestroy, onMount, tick } from 'svelte';
    import { toast } from 'svelte-sonner';
    import { marked } from 'marked';

    import { goto } from '$app/navigation';
    import { user } from '$lib/stores';

    import RichTextInput from '$lib/components/common/RichTextInput.svelte';
    import Spinner from '$lib/components/common/Spinner.svelte';
    import ArrowUturnLeft from '$lib/components/icons/ArrowUturnLeft.svelte';
    import ArrowUturnRight from '$lib/components/icons/ArrowUturnRight.svelte';

    import { getMemoryNoteById, updateMemoryNoteById } from '$lib/apis/memory/notes';
    import type { NoteModel, NoteVersion } from '$lib/apis/memory/notes';

    const i18n = getContext('i18n');

    export let label: string;

    let editor: any = null;
    let note: NoteModel | null = null;
    let loading = true;

    // Version navigation
    let versionIdx: number | null = null;  // null = current, 0 = most recent commit, etc.
    let versions: NoteVersion[] = [];

    // Autosave
    let debounceTimeout: ReturnType<typeof setTimeout> | null = null;
    let saveStatus: 'idle' | 'saving' | 'saved' | 'error' = 'idle';

    // Content tracking
    let currentContent = {
        html: '',
        md: ''
    };

    async function init() {
        loading = true;
        try {
            note = await getMemoryNoteById(localStorage.token, label);
            if (note) {
                currentContent = {
                    html: note.data.content.html,
                    md: note.data.content.md
                };
                versions = note.data.versions || [];
            }
        } catch (e) {
            toast.error(`Failed to load: ${e}`);
            goto('/you');
        } finally {
            loading = false;
        }
    }

    // Autosave with 200ms debounce
    function scheduleAutosave() {
        if (debounceTimeout) {
            clearTimeout(debounceTimeout);
        }

        debounceTimeout = setTimeout(async () => {
            if (!note || versionIdx !== null) return;  // Don't save when viewing history

            saveStatus = 'saving';
            try {
                const updated = await updateMemoryNoteById(
                    localStorage.token,
                    label,
                    {
                        title: note.title,
                        data: {
                            content: {
                                md: currentContent.md
                            }
                        }
                    }
                );

                note = updated;
                versions = updated.data.versions || [];
                saveStatus = 'saved';

                setTimeout(() => {
                    if (saveStatus === 'saved') saveStatus = 'idle';
                }, 2000);
            } catch (e) {
                saveStatus = 'error';
                toast.error(`Autosave failed: ${e}`);
            }
        }, 200);
    }

    // Git-based undo (go to previous version)
    async function undo() {
        if (versions.length === 0) return;

        if (versionIdx === null) {
            versionIdx = 0;  // Most recent commit
        } else if (versionIdx < versions.length - 1) {
            versionIdx++;
        }

        const version = versions[versionIdx];
        if (version && editor) {
            currentContent = { html: version.html, md: version.md };
            editor.commands.setContent(version.html);
        }
    }

    // Git-based redo (go to newer version)
    async function redo() {
        if (versionIdx === null || versionIdx === 0) {
            // Back to current (unsaved) state
            versionIdx = null;
            if (note && editor) {
                currentContent = {
                    html: note.data.content.html,
                    md: note.data.content.md
                };
                editor.commands.setContent(note.data.content.html);
            }
        } else {
            versionIdx--;
            const version = versions[versionIdx];
            if (version && editor) {
                currentContent = { html: version.html, md: version.md };
                editor.commands.setContent(version.html);
            }
        }
    }

    // Restore a specific version (creates new commit)
    async function restoreVersion(sha: string) {
        try {
            // Call restore endpoint (if implemented) or just save the version content
            const version = versions.find(v => v.sha === sha);
            if (!version || !note) return;

            await updateMemoryNoteById(
                localStorage.token,
                label,
                {
                    title: note.title,
                    data: { content: { md: version.md } }
                }
            );

            versionIdx = null;
            await init();  // Reload
            toast.success('Version restored');
        } catch (e) {
            toast.error(`Failed to restore: ${e}`);
        }
    }

    $: canUndo = versions.length > 0 && (versionIdx === null || versionIdx < versions.length - 1);
    $: canRedo = versionIdx !== null && versionIdx >= 0;

    onMount(() => {
        init();
    });

    onDestroy(() => {
        if (debounceTimeout) {
            clearTimeout(debounceTimeout);
        }
    });
</script>

<div class="h-full flex flex-col">
    <!-- Header -->
    <div class="flex items-center justify-between p-4 border-b">
        <div class="flex items-center gap-4">
            <button class="btn btn-ghost btn-sm" on:click={() => goto('/you')}>
                ← Back
            </button>

            {#if note}
                <input
                    type="text"
                    class="text-xl font-semibold bg-transparent border-none focus:outline-none"
                    bind:value={note.title}
                    on:blur={scheduleAutosave}
                />
            {/if}
        </div>

        <div class="flex items-center gap-2">
            <!-- Save status -->
            <span class="text-sm text-gray-500">
                {#if saveStatus === 'saving'}
                    Saving...
                {:else if saveStatus === 'saved'}
                    <span class="text-green-500">Saved</span>
                {:else if saveStatus === 'error'}
                    <span class="text-red-500">Error</span>
                {/if}
            </span>

            <!-- Undo/Redo -->
            <div class="flex items-center gap-1">
                <button
                    class="btn btn-ghost btn-sm"
                    disabled={!canUndo}
                    on:click={undo}
                    title="Undo (view previous version)"
                >
                    <ArrowUturnLeft className="size-4" />
                </button>
                <button
                    class="btn btn-ghost btn-sm"
                    disabled={!canRedo}
                    on:click={redo}
                    title="Redo (view newer version)"
                >
                    <ArrowUturnRight className="size-4" />
                </button>
            </div>

            <!-- Version indicator -->
            {#if versionIdx !== null}
                <span class="text-sm text-warning">
                    Viewing version {versionIdx + 1} of {versions.length}
                </span>
                <button
                    class="btn btn-sm btn-warning"
                    on:click={() => restoreVersion(versions[versionIdx].sha)}
                >
                    Restore this version
                </button>
            {/if}
        </div>
    </div>

    <!-- Editor -->
    <div class="flex-1 overflow-auto p-4">
        {#if loading}
            <div class="flex items-center justify-center h-full">
                <Spinner className="size-8" />
            </div>
        {:else if note}
            <RichTextInput
                bind:editor
                className="prose max-w-none"
                html={currentContent.html}
                editable={versionIdx === null}
                placeholder="Write something..."
                onChange={(content) => {
                    if (versionIdx === null) {
                        currentContent = {
                            html: content.html,
                            md: content.md
                        };
                        scheduleAutosave();
                    }
                }}
            />
        {/if}
    </div>
</div>
```

#### 2. Update Route to Use Wrapper

**File**: `OpenWebUI/open-webui/src/routes/(app)/you/blocks/[label]/+page.svelte`

```svelte
<script lang="ts">
    import { page } from '$app/stores';
    import MemoryBlockEditor from '$lib/components/you/MemoryBlockEditor.svelte';

    $: label = $page.params.label;
</script>

<svelte:head>
    <title>Edit Memory Block</title>
</svelte:head>

{#if label}
    <MemoryBlockEditor {label} />
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] Component compiles without TypeScript errors
- [ ] Route renders the editor

#### Manual Verification:
- [ ] Navigate to `/you/blocks/student`
- [ ] Edit content, see autosave indicator
- [ ] Click undo, see previous git version
- [ ] Click redo, return to current
- [ ] "Restore this version" creates new commit

---

## Testing Strategy

### Unit Tests

Add to `tests/test_server/`:

1. **test_notes_adapter.py** - Test Notes adapter endpoints
   - List returns all blocks as notes
   - Get returns proper NoteResponse format
   - Update saves content and syncs to Letta
   - Versions populated from git history

### Integration Tests

1. **test_notes_adapter_integration.py**
   - Full round-trip: create block → get via adapter → update → verify git

### Manual Testing Steps

1. Start YouLab server: `uv run youlab-server`
2. Navigate to `/you/blocks/student`
3. Edit content, wait 200ms, verify autosave
4. Check git log for new commit
5. Click undo, verify previous content shown
6. Click "Restore", verify new commit created
7. Verify Letta block updated

## Performance Considerations

1. **Version loading**: Only fetch version content on-demand (when user clicks undo)
2. **Autosave debounce**: 200ms prevents excessive commits
3. **Git history limit**: Default to 20 versions, paginate if needed

## Migration Notes

- No data migration needed - storage format unchanged from Phase 1-2
- Existing blocks accessible via new adapter immediately
- BlockDetailModal still works (parallel path during transition)

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Linear ticket: [ARI-82](https://linear.app/ariav/issue/ARI-82)
- OpenWebUI Notes: `OpenWebUI/open-webui/src/lib/components/notes/`
- NoteEditor: `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte`
- Notes API: `OpenWebUI/open-webui/src/lib/apis/notes/index.ts`
