---
date: 2026-01-10T12:45:00+07:00
author: ariasulin
status: draft
ticket: null
title: "Bidirectional File & Knowledge Sync with Agent Write Access"
---

# Implementation Plan: Bidirectional File & Knowledge Sync with Agent Write Access

## Overview

Enable users to upload files via OpenWebUI chat that persist in collections/notes, with bidirectional sync to Letta folders. Agents can read all files and optionally write/edit files based on TOML-configured permissions.

**Key Design Decisions:**
- **MD files → Notes** (editable documents in OpenWebUI)
- **PDF files → Knowledge collections** (converted to MD for Letta)
- **Real-time sync** for agent edits back to OpenWebUI
- **TOML-based permissions** (read-only vs read-write per agent)
- **OpenWebUI as pure UI** - no vector DB usage, Letta handles all RAG
- **Shared + Private folders** with OpenWebUI access control mapping

## Current State Analysis

### Existing Infrastructure

| Component | Status | Location |
|-----------|--------|----------|
| OpenWebUI → Letta Knowledge sync | **Planned** | `thoughts/shared/plans/2026-01-09-openwebui-knowledge-sync.md` |
| OpenWebUI Notes research | **Researched** | `thoughts/shared/research/2026-01-09-openwebui-notes-letta-sync.md` |
| Agent file tools (read-only) | **Available** | Letta built-in: `open_files`, `grep_files`, `semantic_search_files` |
| Agent file write tools | **NOT available** | Must create custom tools |
| Bidirectional sync | **NOT implemented** | New requirement |
| File upload from chat | **NOT implemented** | OpenWebUI supports, not integrated |

### Key Discoveries

1. **OpenWebUI Access Control Model** (`access_control.py`):
   ```python
   access_control = {
       "read": {"group_ids": [...], "user_ids": [...]},
       "write": {"group_ids": [...], "user_ids": [...]}
   }
   ```
   - `None` = public read access
   - `{}` = private (owner only)
   - Custom dict = granular permissions

2. **Letta has NO built-in file write tools** - agents can read files via `open_files`, `grep_files`, `semantic_search_files`, but cannot write. Custom tools required.

3. **Notes vs Knowledge** in OpenWebUI:
   - Notes: Markdown in DB, no embeddings, direct content access
   - Knowledge: File collections, vector embeddings (which we'll ignore)

## Desired End State

### User Experience

1. User uploads file in OpenWebUI chat
2. MD files appear as editable Notes, PDFs appear in Knowledge collection (converted to MD)
3. Files are automatically synced to Letta folders
4. Agent can read all synced files via file tools
5. Agents with write permission can edit/create files
6. Agent edits appear immediately in OpenWebUI
7. Access controls from OpenWebUI are respected

### Verification

- [ ] Upload MD file in chat → appears as Note in OpenWebUI
- [ ] Upload PDF file in chat → appears in Knowledge collection
- [ ] Files appear in Letta folders within sync interval
- [ ] Agent can search and read uploaded files
- [ ] Agent with write permission can edit files
- [ ] Agent edits appear in OpenWebUI within 5 seconds
- [ ] Private files only accessible to owner
- [ ] Shared files accessible per access_control

## What We're NOT Doing

- **Not using OpenWebUI vector DB** - Letta handles all RAG
- **Not implementing collaborative editing** - no Yjs/real-time doc editing
- **Not implementing file versioning** - agent edits overwrite, no history
- **Not implementing per-file blacklists** - deferred to future iteration
- **Not implementing DOCX/TXT/image support** - only MD and PDF for now
- **Not implementing webhook-based sync** - polling only (OpenWebUI webhooks not stable)

## Implementation Approach

### Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        OpenWebUI (Pure UI)                                  │
│  • File upload in chat                                                      │
│  • Notes management                                                         │
│  • Knowledge collection management                                          │
│  • Access control UI                                                        │
└────────────────────────────────────────────────────────────────────────────┘
           │ upload                              ▲ sync back
           ▼                                     │
┌────────────────────────────────────────────────────────────────────────────┐
│                      Bidirectional Sync Service                             │
│  • Poll OpenWebUI for Notes + Knowledge changes                            │
│  • Poll Letta for agent file changes                                        │
│  • Convert PDF → MD on sync                                                 │
│  • Maintain mapping tables                                                  │
│  • Respect access_control for folder isolation                             │
└────────────────────────────────────────────────────────────────────────────┘
           │ sync down                           ▲ sync up
           ▼                                     │
┌────────────────────────────────────────────────────────────────────────────┐
│                      Letta (Agent Runtime)                                  │
│  • Folders with synced files                                                │
│  • Built-in file tools (read): open_files, grep_files, semantic_search     │
│  • Custom file tools (write): edit_file, create_file                       │
│  • RAG via semantic_search_files                                           │
└────────────────────────────────────────────────────────────────────────────┘
```

### File Type Routing

```
User uploads file in chat
         │
         ▼
    ┌─────────────┐
    │  File type? │
    └─────────────┘
         │
    ┌────┴────┐
    ▼         ▼
  .md        .pdf
    │         │
    ▼         ▼
 Create    Upload to
  Note     Knowledge
    │      collection
    │         │
    ▼         ▼
 Sync to   Convert to
 Letta     MD, sync to
 folder    Letta folder
```

### Permission Model

```toml
# config/courses/college-essay/course.toml

[agent]
name = "Essay Coach"
# ... other config

[agent.tools]
# Read-only file access (default)
file_read = true

# Write access (explicit opt-in)
file_write = false  # or true for agents that can edit

[agent.folders]
# Folder attachment (shared across agents)
shared = ["course-materials", "student-submissions"]
# User-private folder (per-user)
private = true  # Creates openwebui_user_{user_id} folder
```

**Tool Registration Based on Permissions:**

| `file_write` | Tools Registered |
|--------------|------------------|
| `false` | `open_files`, `grep_files`, `semantic_search_files` |
| `true` | Above + `edit_file`, `create_file` |

---

## Phase 1: Core Sync Service (OpenWebUI → Letta)

### Overview

Implement the foundational sync service that pulls Notes and Knowledge from OpenWebUI into Letta folders.

### Changes Required

#### 1. Configuration Updates

**File**: `src/letta_starter/config/settings.py`

Add to `ServiceSettings`:

```python
# Bidirectional File Sync configuration
file_sync_enabled: bool = Field(
    default=False,
    description="Enable bidirectional file sync with OpenWebUI",
)
openwebui_url: str = Field(
    default="http://localhost:3000",
    description="OpenWebUI base URL",
)
openwebui_api_key: str | None = Field(
    default=None,
    description="OpenWebUI API key for sync",
)
file_sync_interval: int = Field(
    default=30,
    description="Seconds between sync cycles",
)
file_sync_embedding_model: str = Field(
    default="openai/text-embedding-3-small",
    description="Embedding model for Letta folders",
)
pdf_conversion_enabled: bool = Field(
    default=True,
    description="Convert PDF to MD when syncing",
)
```

#### 2. OpenWebUI Client

**File**: `src/letta_starter/server/sync/openwebui_client.py`

```python
"""OpenWebUI API client for file sync."""

from dataclasses import dataclass
from datetime import datetime
import httpx

@dataclass
class OpenWebUINote:
    id: str
    user_id: str
    title: str
    content: str  # markdown content
    access_control: dict | None
    created_at: datetime
    updated_at: datetime

@dataclass
class OpenWebUIFile:
    id: str
    user_id: str
    filename: str
    content_type: str
    size: int
    created_at: datetime
    updated_at: datetime

@dataclass
class OpenWebUIKnowledge:
    id: str
    user_id: str
    name: str
    description: str
    files: list[OpenWebUIFile]
    access_control: dict | None
    created_at: datetime
    updated_at: datetime

class OpenWebUIClient:
    """Client for OpenWebUI API."""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=60.0,
        )

    async def list_notes(self, page: int = 1) -> list[OpenWebUINote]:
        """List all notes (paginated, 60/page)."""
        ...

    async def get_note(self, note_id: str) -> OpenWebUINote:
        """Get note with content."""
        ...

    async def update_note(self, note_id: str, title: str, content: str) -> None:
        """Update note content (for sync-back)."""
        ...

    async def create_note(self, title: str, content: str, access_control: dict | None = None) -> OpenWebUINote:
        """Create new note (for agent-created files)."""
        ...

    async def list_knowledge(self, page: int = 1) -> list[OpenWebUIKnowledge]:
        """List all knowledge collections."""
        ...

    async def get_knowledge(self, knowledge_id: str) -> OpenWebUIKnowledge:
        """Get knowledge collection with files."""
        ...

    async def get_file_content(self, file_id: str) -> bytes:
        """Download file content."""
        ...

    async def close(self) -> None:
        """Close HTTP client."""
        await self.client.aclose()
```

#### 3. PDF Converter

**File**: `src/letta_starter/server/sync/pdf_converter.py`

```python
"""PDF to Markdown conversion."""

import subprocess
import tempfile
from pathlib import Path

def convert_pdf_to_md(pdf_bytes: bytes, filename: str) -> str:
    """
    Convert PDF to Markdown using marker-pdf.

    Falls back to basic text extraction if marker unavailable.
    """
    with tempfile.TemporaryDirectory() as tmpdir:
        pdf_path = Path(tmpdir) / filename
        pdf_path.write_bytes(pdf_bytes)

        try:
            # Use marker-pdf for high-quality conversion
            result = subprocess.run(
                ["marker_single", str(pdf_path), tmpdir, "--output_format", "markdown"],
                capture_output=True,
                text=True,
                timeout=120,
            )

            md_path = Path(tmpdir) / f"{pdf_path.stem}.md"
            if md_path.exists():
                return md_path.read_text()
        except (subprocess.TimeoutExpired, FileNotFoundError):
            pass

        # Fallback: basic text extraction with pdfplumber
        import pdfplumber

        text_parts = []
        with pdfplumber.open(pdf_path) as pdf:
            for page in pdf.pages:
                text_parts.append(page.extract_text() or "")

        return "\n\n".join(text_parts)
```

#### 4. Sync Mapping Storage

**File**: `src/letta_starter/server/sync/mappings.py`

```python
"""Mapping storage for sync state."""

from dataclasses import dataclass, field
from datetime import datetime
import hashlib
import json
from pathlib import Path

@dataclass
class NoteMapping:
    openwebui_note_id: str
    letta_folder_id: str
    letta_file_id: str | None
    title: str
    content_hash: str
    last_synced: datetime
    status: str  # synced, pending, error

@dataclass
class FileMapping:
    openwebui_file_id: str
    openwebui_knowledge_id: str
    letta_folder_id: str
    letta_file_id: str | None
    filename: str
    content_hash: str
    last_synced: datetime
    status: str

class SyncMappingStore:
    """Persistent storage for sync mappings."""

    def __init__(self, storage_path: Path):
        self.storage_path = storage_path
        self.note_mappings: dict[str, NoteMapping] = {}
        self.file_mappings: dict[str, FileMapping] = {}
        self._load()

    def _load(self) -> None:
        """Load mappings from disk."""
        ...

    def _save(self) -> None:
        """Persist mappings to disk."""
        ...

    def get_note_mapping(self, openwebui_note_id: str) -> NoteMapping | None:
        ...

    def set_note_mapping(self, mapping: NoteMapping) -> None:
        ...

    def get_file_mapping(self, openwebui_file_id: str) -> FileMapping | None:
        ...

    def set_file_mapping(self, mapping: FileMapping) -> None:
        ...

    @staticmethod
    def compute_hash(content: str | bytes) -> str:
        """Compute content hash for change detection."""
        if isinstance(content, str):
            content = content.encode()
        return hashlib.sha256(content).hexdigest()[:16]
```

#### 5. Sync Service Core

**File**: `src/letta_starter/server/sync/service.py`

```python
"""Bidirectional sync service for OpenWebUI ↔ Letta."""

import asyncio
import logging
from dataclasses import dataclass
from datetime import datetime
from typing import TYPE_CHECKING

from letta_client import Letta

from .openwebui_client import OpenWebUIClient, OpenWebUINote, OpenWebUIKnowledge
from .mappings import SyncMappingStore, NoteMapping, FileMapping
from .pdf_converter import convert_pdf_to_md

if TYPE_CHECKING:
    from letta_starter.config import ServiceSettings

logger = logging.getLogger(__name__)

@dataclass
class SyncStats:
    notes_synced: int = 0
    notes_skipped: int = 0
    notes_failed: int = 0
    files_synced: int = 0
    files_skipped: int = 0
    files_failed: int = 0
    duration_ms: int = 0

class FileSyncService:
    """
    Bidirectional sync between OpenWebUI and Letta.

    Sync directions:
    - DOWN: OpenWebUI Notes/Knowledge → Letta Folders
    - UP: Letta agent edits → OpenWebUI Notes/Knowledge
    """

    def __init__(
        self,
        settings: "ServiceSettings",
        letta_client: Letta,
    ):
        self.settings = settings
        self.letta = letta_client
        self.openwebui = OpenWebUIClient(
            base_url=settings.openwebui_url,
            api_key=settings.openwebui_api_key,
        )
        self.mappings = SyncMappingStore(
            storage_path=Path(settings.data_dir) / "sync_mappings.json"
        )
        self._running = False
        self._task: asyncio.Task | None = None

        # Cache for folder lookups
        self._folder_cache: dict[str, str] = {}  # name → folder_id

    async def start(self) -> None:
        """Start background sync loop."""
        if self._running:
            return
        self._running = True
        self._task = asyncio.create_task(self._sync_loop())
        logger.info("File sync service started")

    async def stop(self) -> None:
        """Stop background sync loop."""
        self._running = False
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        await self.openwebui.close()
        logger.info("File sync service stopped")

    async def _sync_loop(self) -> None:
        """Background sync loop."""
        while self._running:
            try:
                await self.sync_all()
            except Exception as e:
                logger.error(f"Sync cycle failed: {e}")
            await asyncio.sleep(self.settings.file_sync_interval)

    async def sync_all(self) -> SyncStats:
        """Perform full bidirectional sync cycle."""
        start = datetime.now()
        stats = SyncStats()

        # Sync DOWN: OpenWebUI → Letta
        await self._sync_notes_down(stats)
        await self._sync_knowledge_down(stats)

        # Sync UP: Letta → OpenWebUI (agent edits)
        await self._sync_notes_up(stats)

        stats.duration_ms = int((datetime.now() - start).total_seconds() * 1000)
        logger.info(f"Sync complete: {stats}")
        return stats

    async def _sync_notes_down(self, stats: SyncStats) -> None:
        """Sync OpenWebUI Notes → Letta folders."""
        notes = await self.openwebui.list_notes()

        for note in notes:
            try:
                mapping = self.mappings.get_note_mapping(note.id)
                content_hash = SyncMappingStore.compute_hash(note.content)

                # Skip if unchanged
                if mapping and mapping.content_hash == content_hash:
                    stats.notes_skipped += 1
                    continue

                # Determine folder based on access_control
                folder_name = self._get_folder_name_for_note(note)
                folder_id = await self._ensure_folder(folder_name)

                # Upload/update file in Letta
                file_id = await self._upload_to_letta(
                    folder_id=folder_id,
                    filename=f"{note.title}.md",
                    content=note.content.encode(),
                    existing_file_id=mapping.letta_file_id if mapping else None,
                )

                # Update mapping
                self.mappings.set_note_mapping(NoteMapping(
                    openwebui_note_id=note.id,
                    letta_folder_id=folder_id,
                    letta_file_id=file_id,
                    title=note.title,
                    content_hash=content_hash,
                    last_synced=datetime.now(),
                    status="synced",
                ))
                stats.notes_synced += 1

            except Exception as e:
                logger.error(f"Failed to sync note {note.id}: {e}")
                stats.notes_failed += 1

    async def _sync_knowledge_down(self, stats: SyncStats) -> None:
        """Sync OpenWebUI Knowledge → Letta folders with PDF conversion."""
        collections = await self.openwebui.list_knowledge()

        for collection in collections:
            folder_name = f"knowledge_{collection.id}"
            folder_id = await self._ensure_folder(folder_name)

            for file in collection.files:
                try:
                    mapping = self.mappings.get_file_mapping(file.id)

                    # Get file content
                    content = await self.openwebui.get_file_content(file.id)
                    content_hash = SyncMappingStore.compute_hash(content)

                    # Skip if unchanged
                    if mapping and mapping.content_hash == content_hash:
                        stats.files_skipped += 1
                        continue

                    # Convert PDF to MD if needed
                    if file.filename.lower().endswith(".pdf"):
                        md_content = convert_pdf_to_md(content, file.filename)
                        upload_content = md_content.encode()
                        upload_filename = file.filename.rsplit(".", 1)[0] + ".md"
                    else:
                        upload_content = content
                        upload_filename = file.filename

                    # Upload to Letta
                    file_id = await self._upload_to_letta(
                        folder_id=folder_id,
                        filename=upload_filename,
                        content=upload_content,
                        existing_file_id=mapping.letta_file_id if mapping else None,
                    )

                    # Update mapping
                    self.mappings.set_file_mapping(FileMapping(
                        openwebui_file_id=file.id,
                        openwebui_knowledge_id=collection.id,
                        letta_folder_id=folder_id,
                        letta_file_id=file_id,
                        filename=file.filename,
                        content_hash=content_hash,
                        last_synced=datetime.now(),
                        status="synced",
                    ))
                    stats.files_synced += 1

                except Exception as e:
                    logger.error(f"Failed to sync file {file.id}: {e}")
                    stats.files_failed += 1

    async def _sync_notes_up(self, stats: SyncStats) -> None:
        """Sync Letta agent edits → OpenWebUI Notes."""
        # For each note mapping, check if Letta file has changed
        for note_id, mapping in list(self.mappings.note_mappings.items()):
            if not mapping.letta_file_id:
                continue

            try:
                # Get current content from Letta
                letta_content = await self._get_letta_file_content(
                    mapping.letta_folder_id,
                    mapping.letta_file_id,
                )
                letta_hash = SyncMappingStore.compute_hash(letta_content)

                # Skip if unchanged
                if letta_hash == mapping.content_hash:
                    continue

                # Update OpenWebUI note
                await self.openwebui.update_note(
                    note_id=note_id,
                    title=mapping.title,
                    content=letta_content.decode(),
                )

                # Update mapping hash
                mapping.content_hash = letta_hash
                mapping.last_synced = datetime.now()
                self.mappings.set_note_mapping(mapping)

                logger.info(f"Synced agent edit to note {note_id}")

            except Exception as e:
                logger.error(f"Failed to sync note {note_id} up: {e}")

    def _get_folder_name_for_note(self, note: OpenWebUINote) -> str:
        """Determine folder name based on note access control."""
        if note.access_control is None:
            # Public note → shared folder
            return "shared_notes"
        elif note.access_control == {}:
            # Private note → user folder
            return f"user_{note.user_id}_notes"
        else:
            # Granular access → user folder (simplified)
            return f"user_{note.user_id}_notes"

    async def _ensure_folder(self, name: str) -> str:
        """Get or create Letta folder by name."""
        if name in self._folder_cache:
            return self._folder_cache[name]

        # Search for existing folder
        folders = self.letta.folders.list()
        for folder in folders:
            if folder.name == name:
                self._folder_cache[name] = folder.id
                return folder.id

        # Create new folder
        folder = self.letta.folders.create(
            name=name,
            embedding=self.settings.file_sync_embedding_model,
        )
        self._folder_cache[name] = folder.id
        return folder.id

    async def _upload_to_letta(
        self,
        folder_id: str,
        filename: str,
        content: bytes,
        existing_file_id: str | None = None,
    ) -> str:
        """Upload file to Letta folder, replacing if exists."""
        import io

        # Delete existing file if updating
        if existing_file_id:
            try:
                self.letta.folders.files.delete(
                    folder_id=folder_id,
                    file_id=existing_file_id,
                )
            except Exception:
                pass  # File may not exist

        # Upload new file
        file_obj = io.BytesIO(content)
        file_obj.name = filename

        job = self.letta.folders.files.upload(
            folder_id=folder_id,
            file=file_obj,
        )

        # Wait for processing
        while True:
            job = self.letta.jobs.retrieve(job.id)
            if job.status == "completed":
                break
            elif job.status == "failed":
                raise RuntimeError(f"File upload failed: {job.error}")
            await asyncio.sleep(0.5)

        # Get file ID from job result
        files = self.letta.folders.files.list(folder_id=folder_id)
        for f in files:
            if f.name == filename:
                return f.id

        raise RuntimeError(f"File {filename} not found after upload")

    async def _get_letta_file_content(self, folder_id: str, file_id: str) -> bytes:
        """Get file content from Letta."""
        # Note: Letta SDK may need a specific method for this
        # This is a placeholder - need to verify actual API
        file = self.letta.folders.files.retrieve(
            folder_id=folder_id,
            file_id=file_id,
        )
        return file.content  # Verify this is the correct accessor
```

#### 6. Sync Router

**File**: `src/letta_starter/server/sync/router.py`

```python
"""HTTP endpoints for file sync management."""

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from .service import FileSyncService, SyncStats

router = APIRouter(prefix="/sync", tags=["sync"])

class SyncStatusResponse(BaseModel):
    enabled: bool
    running: bool
    last_sync: str | None
    last_stats: SyncStats | None

class AttachFolderRequest(BaseModel):
    agent_id: str
    folder_name: str

@router.get("/status")
async def get_sync_status(
    sync: FileSyncService = Depends(get_file_sync),
) -> SyncStatusResponse:
    """Get sync service status."""
    return SyncStatusResponse(
        enabled=sync.settings.file_sync_enabled,
        running=sync._running,
        last_sync=None,  # TODO: track last sync time
        last_stats=None,
    )

@router.post("/trigger")
async def trigger_sync(
    sync: FileSyncService = Depends(get_file_sync),
) -> SyncStats:
    """Manually trigger a sync cycle."""
    return await sync.sync_all()

@router.post("/attach")
async def attach_folder_to_agent(
    request: AttachFolderRequest,
    sync: FileSyncService = Depends(get_file_sync),
) -> dict:
    """Attach a synced folder to an agent."""
    folder_id = await sync._ensure_folder(request.folder_name)
    sync.letta.agents.folders.attach(
        agent_id=request.agent_id,
        folder_id=folder_id,
    )
    return {"status": "attached", "folder_id": folder_id}

@router.get("/mappings")
async def list_mappings(
    sync: FileSyncService = Depends(get_file_sync),
) -> dict:
    """List all sync mappings."""
    return {
        "notes": list(sync.mappings.note_mappings.values()),
        "files": list(sync.mappings.file_mappings.values()),
    }
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`
- [ ] Sync service starts without errors

#### Manual Verification:
- [ ] Create a Note in OpenWebUI, verify it appears in Letta folder within sync interval
- [ ] Upload PDF to Knowledge collection, verify converted MD appears in Letta
- [ ] Check `/sync/status` returns running state
- [ ] Check `/sync/mappings` shows correct mappings

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Custom File Write Tools

### Overview

Create custom Letta tools that allow agents to edit and create files, with edits synced back to OpenWebUI.

### Changes Required

#### 1. File Write Tools

**File**: `src/letta_starter/tools/files.py`

```python
"""Custom file tools for Letta agents."""

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from letta.agent import AgentState

def edit_file(
    agent_state: "AgentState",
    filename: str,
    new_content: str,
    folder_name: str | None = None,
) -> str:
    """
    Edit an existing file's content.

    This tool modifies a file that exists in one of your attached folders.
    The edit will be synced back to OpenWebUI so the user can see your changes.

    Args:
        agent_state: Current agent state (auto-injected)
        filename: Name of the file to edit (e.g., "essay_draft.md")
        new_content: The complete new content for the file
        folder_name: Optional folder name. If not specified, searches all attached folders.

    Returns:
        Status message indicating success or failure
    """
    from letta_client import Letta
    import io

    # Get Letta client
    # Note: Need to access client from agent state or global config
    client = Letta(base_url="http://localhost:8283")  # TODO: get from config

    # Find the file in attached folders
    agent_folders = client.agents.folders.list(agent_id=agent_state.agent_state.id)

    target_file = None
    target_folder_id = None

    for folder in agent_folders:
        if folder_name and folder.name != folder_name:
            continue

        files = client.folders.files.list(folder_id=folder.id)
        for f in files:
            if f.name == filename:
                target_file = f
                target_folder_id = folder.id
                break

        if target_file:
            break

    if not target_file:
        return f"Error: File '{filename}' not found in attached folders"

    # Delete old file and upload new content
    client.folders.files.delete(folder_id=target_folder_id, file_id=target_file.id)

    file_obj = io.BytesIO(new_content.encode())
    file_obj.name = filename

    job = client.folders.files.upload(folder_id=target_folder_id, file=file_obj)

    # Wait for processing
    import time
    while True:
        job = client.jobs.retrieve(job.id)
        if job.status == "completed":
            break
        elif job.status == "failed":
            return f"Error: File upload failed - {job.error}"
        time.sleep(0.5)

    return f"Successfully updated '{filename}'. Changes will sync to OpenWebUI shortly."


def create_file(
    agent_state: "AgentState",
    filename: str,
    content: str,
    folder_name: str,
) -> str:
    """
    Create a new file in a folder.

    This tool creates a new file that will be synced to OpenWebUI as a Note
    (for .md files) or added to a Knowledge collection (for other files).

    Args:
        agent_state: Current agent state (auto-injected)
        filename: Name for the new file (e.g., "feedback.md")
        content: Content to write to the file
        folder_name: Name of the folder to create the file in

    Returns:
        Status message indicating success or failure
    """
    from letta_client import Letta
    import io

    client = Letta(base_url="http://localhost:8283")

    # Find folder
    agent_folders = client.agents.folders.list(agent_id=agent_state.agent_state.id)

    target_folder = None
    for folder in agent_folders:
        if folder.name == folder_name:
            target_folder = folder
            break

    if not target_folder:
        return f"Error: Folder '{folder_name}' not found in attached folders"

    # Check if file already exists
    files = client.folders.files.list(folder_id=target_folder.id)
    for f in files:
        if f.name == filename:
            return f"Error: File '{filename}' already exists. Use edit_file to modify it."

    # Create file
    file_obj = io.BytesIO(content.encode())
    file_obj.name = filename

    job = client.folders.files.upload(folder_id=target_folder.id, file=file_obj)

    import time
    while True:
        job = client.jobs.retrieve(job.id)
        if job.status == "completed":
            break
        elif job.status == "failed":
            return f"Error: File creation failed - {job.error}"
        time.sleep(0.5)

    return f"Successfully created '{filename}' in folder '{folder_name}'. It will sync to OpenWebUI shortly."
```

#### 2. Tool Registration in TOML Schema

**File**: `src/letta_starter/curriculum/schema.py`

Add to agent config schema:

```python
class AgentToolsConfig(BaseModel):
    """Configuration for agent tool access."""

    file_read: bool = Field(
        default=True,
        description="Enable file reading tools (open_files, grep_files, semantic_search_files)",
    )
    file_write: bool = Field(
        default=False,
        description="Enable file writing tools (edit_file, create_file)",
    )
```

#### 3. Agent Creation with Tool Selection

**File**: `src/letta_starter/server/agents.py`

Modify agent creation to register tools based on config:

```python
def _get_tools_for_agent(config: AgentToolsConfig) -> list[str]:
    """Get tool list based on agent configuration."""
    tools = []

    # File read tools are automatic when folders attached
    # File write tools need explicit registration
    if config.file_write:
        tools.extend(["edit_file", "create_file"])

    return tools

async def create_agent(...):
    # ... existing code ...

    # Register custom tools if needed
    if curriculum.agent.tools.file_write:
        # Register edit_file and create_file tools
        from letta_starter.tools.files import edit_file, create_file

        client.tools.upsert_from_function(edit_file)
        client.tools.upsert_from_function(create_file)

    # Create agent with tools
    tools = _get_tools_for_agent(curriculum.agent.tools)
    # ... pass tools to agent creation ...
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests for file tools pass
- [ ] Type checking passes
- [ ] Integration test: agent can call edit_file

#### Manual Verification:
- [ ] Create agent with `file_write = true`
- [ ] Agent successfully edits a file via tool call
- [ ] Edited file syncs back to OpenWebUI within sync interval
- [ ] Agent can create new file
- [ ] New file appears as Note in OpenWebUI

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Access Control & Folder Attachment

### Overview

Implement proper access control mapping and automatic folder attachment based on TOML config.

### Changes Required

#### 1. Access Control Mapping

**File**: `src/letta_starter/server/sync/access_control.py`

```python
"""Map OpenWebUI access control to Letta folder structure."""

from dataclasses import dataclass

@dataclass
class FolderAccess:
    folder_name: str
    user_ids: list[str]
    is_public: bool

def map_access_control(
    user_id: str,
    access_control: dict | None,
    resource_type: str,  # "note" or "knowledge"
    resource_id: str,
) -> FolderAccess:
    """
    Map OpenWebUI access_control to Letta folder naming.

    Rules:
    - access_control=None → shared folder (public)
    - access_control={} → user-private folder
    - access_control with group_ids/user_ids → shared with specific users
    """
    if access_control is None:
        # Public → shared folder
        return FolderAccess(
            folder_name=f"shared_{resource_type}s",
            user_ids=[],
            is_public=True,
        )

    if access_control == {}:
        # Private → user folder
        return FolderAccess(
            folder_name=f"user_{user_id}_{resource_type}s",
            user_ids=[user_id],
            is_public=False,
        )

    # Granular access
    read_users = access_control.get("read", {}).get("user_ids", [])
    write_users = access_control.get("write", {}).get("user_ids", [])
    all_users = list(set([user_id] + read_users + write_users))

    return FolderAccess(
        folder_name=f"shared_{resource_type}_{resource_id[:8]}",
        user_ids=all_users,
        is_public=False,
    )
```

#### 2. TOML Folder Configuration

**File**: `config/courses/college-essay/course.toml`

```toml
[agent]
name = "Essay Coach"
model = "claude-sonnet-4-20250514"

[agent.tools]
file_read = true
file_write = false

[agent.folders]
# Shared course materials (all users)
shared = ["college-essay-materials"]

# User gets a private folder for their work
private = true

# Background agents that should have write access
[background_agents.feedback_agent]
name = "Feedback Agent"

[background_agents.feedback_agent.tools]
file_read = true
file_write = true  # This agent can edit user files

[background_agents.feedback_agent.folders]
private = true  # Access user's private folder
```

#### 3. Automatic Folder Attachment

**File**: `src/letta_starter/server/agents.py`

```python
async def attach_folders_for_agent(
    client: Letta,
    agent_id: str,
    user_id: str,
    folder_config: FolderConfig,
    sync_service: FileSyncService,
) -> list[str]:
    """Attach configured folders to agent."""
    attached = []

    # Attach shared folders
    for folder_name in folder_config.shared:
        folder_id = await sync_service._ensure_folder(folder_name)
        client.agents.folders.attach(agent_id=agent_id, folder_id=folder_id)
        attached.append(folder_name)

    # Attach user's private folder if configured
    if folder_config.private:
        private_folder_name = f"user_{user_id}_notes"
        folder_id = await sync_service._ensure_folder(private_folder_name)
        client.agents.folders.attach(agent_id=agent_id, folder_id=folder_id)
        attached.append(private_folder_name)

    return attached
```

### Success Criteria

#### Automated Verification:
- [ ] Access control mapping unit tests pass
- [ ] TOML parsing includes folder config
- [ ] Agent creation attaches correct folders

#### Manual Verification:
- [ ] Create Note in OpenWebUI (private) → syncs to user folder only
- [ ] Create Note in OpenWebUI (public) → syncs to shared folder
- [ ] Agent only sees folders it should have access to
- [ ] Background agent with write=true can edit, main agent cannot

---

## Phase 4: Real-Time Sync for Agent Edits

### Overview

Make agent edits sync to OpenWebUI immediately rather than waiting for the next sync cycle.

### Changes Required

#### 1. Edit Notification System

**File**: `src/letta_starter/tools/files.py`

Modify edit_file and create_file to trigger immediate sync:

```python
def edit_file(...) -> str:
    # ... existing code ...

    # After successful file update, trigger immediate sync-up
    _notify_file_edit(filename, folder_name)

    return f"Successfully updated '{filename}'. Changes synced to OpenWebUI."

def _notify_file_edit(filename: str, folder_name: str) -> None:
    """Notify sync service of file edit for immediate sync."""
    import httpx

    # Call sync service endpoint
    try:
        httpx.post(
            "http://localhost:8000/sync/notify-edit",
            json={"filename": filename, "folder_name": folder_name},
            timeout=5.0,
        )
    except Exception:
        pass  # Non-critical, regular sync will catch it
```

#### 2. Immediate Sync Endpoint

**File**: `src/letta_starter/server/sync/router.py`

```python
class EditNotification(BaseModel):
    filename: str
    folder_name: str

@router.post("/notify-edit")
async def notify_file_edit(
    notification: EditNotification,
    sync: FileSyncService = Depends(get_file_sync),
) -> dict:
    """Handle immediate sync of agent file edit."""
    await sync.sync_single_file_up(
        filename=notification.filename,
        folder_name=notification.folder_name,
    )
    return {"status": "synced"}
```

#### 3. Single File Sync Method

**File**: `src/letta_starter/server/sync/service.py`

```python
async def sync_single_file_up(self, filename: str, folder_name: str) -> None:
    """Immediately sync a single file from Letta to OpenWebUI."""
    # Find mapping by filename and folder
    for note_id, mapping in self.mappings.note_mappings.items():
        folder = self.letta.folders.retrieve(folder_id=mapping.letta_folder_id)
        if folder.name != folder_name:
            continue
        if mapping.title + ".md" != filename:
            continue

        # Get content from Letta
        content = await self._get_letta_file_content(
            mapping.letta_folder_id,
            mapping.letta_file_id,
        )

        # Update OpenWebUI
        await self.openwebui.update_note(
            note_id=note_id,
            title=mapping.title,
            content=content.decode(),
        )

        # Update hash
        mapping.content_hash = SyncMappingStore.compute_hash(content)
        mapping.last_synced = datetime.now()
        self.mappings.set_note_mapping(mapping)

        logger.info(f"Immediate sync: {filename} → OpenWebUI")
        return

    logger.warning(f"No mapping found for {filename} in {folder_name}")
```

### Success Criteria

#### Automated Verification:
- [ ] Integration test: edit triggers immediate sync
- [ ] Notify endpoint returns quickly (<100ms)

#### Manual Verification:
- [ ] Agent edits file → appears in OpenWebUI within 5 seconds
- [ ] No errors in logs during immediate sync
- [ ] Regular sync cycle still works for non-immediate changes

---

## Testing Strategy

### Unit Tests

- `tests/test_sync_service.py` - Sync logic, mapping management
- `tests/test_openwebui_client.py` - API client (mocked)
- `tests/test_pdf_converter.py` - PDF conversion
- `tests/test_file_tools.py` - Custom tool functions
- `tests/test_access_control.py` - Access control mapping

### Integration Tests

- `tests/integration/test_sync_integration.py` - Full sync cycle
- `tests/integration/test_agent_file_edit.py` - Agent edits file, syncs back

### Manual Testing Steps

1. **Basic Sync**
   - Enable sync in `.env`
   - Create Note in OpenWebUI
   - Wait 30 seconds
   - Verify Note appears in Letta folder via `/sync/mappings`

2. **PDF Conversion**
   - Upload PDF to Knowledge collection
   - Verify MD file appears in Letta folder
   - Check MD content is readable

3. **Agent Read**
   - Create agent with folder attached
   - Ask agent about file contents
   - Verify agent can search and open files

4. **Agent Write**
   - Create agent with `file_write = true`
   - Ask agent to edit a file
   - Verify edit appears in OpenWebUI immediately

5. **Access Control**
   - Create private Note (user A)
   - Create agent for user B
   - Verify user B's agent cannot see user A's Note

## Dependencies

**New:**
- `pdfplumber` - PDF text extraction fallback
- `marker-pdf` (optional) - High-quality PDF→MD conversion

**Existing:**
- `httpx` - Async HTTP client
- `letta-client` - Letta SDK

## Environment Variables

```bash
# Add to .env
YOULAB_SERVICE_FILE_SYNC_ENABLED=true
YOULAB_SERVICE_OPENWEBUI_URL=http://localhost:3000
YOULAB_SERVICE_OPENWEBUI_API_KEY=sk-xxxx
YOULAB_SERVICE_FILE_SYNC_INTERVAL=30
YOULAB_SERVICE_FILE_SYNC_EMBEDDING_MODEL=openai/text-embedding-3-small
YOULAB_SERVICE_PDF_CONVERSION_ENABLED=true
```

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Large PDF conversion timeouts | 120s timeout, async processing |
| Sync conflicts (simultaneous edits) | Last-write-wins, warn on hash mismatch |
| Letta file API changes | Pin letta-client version, add tests |
| OpenWebUI API changes | Version check on startup |
| Performance with many files | Batch processing, incremental sync |

## Future Enhancements

1. **Per-file edit blacklist** in TOML
2. **Version history** for file edits
3. **Conflict detection** with user notification
4. **Webhook-based sync** when OpenWebUI supports it
5. **DOCX/TXT support** with conversion
6. **Image OCR** for image-to-text extraction
7. **Collaborative editing** awareness (Yjs integration)

## References

- Existing plan: `thoughts/shared/plans/2026-01-09-openwebui-knowledge-sync.md`
- Notes research: `thoughts/shared/research/2026-01-09-openwebui-notes-letta-sync.md`
- Letta filesystem: https://docs.letta.com/guides/agents/filesystem
- OpenWebUI API: https://docs.openwebui.com/getting-started/api-endpoints/
