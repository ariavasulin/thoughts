---
date: 2026-01-10T12:45:00+07:00
author: ariasulin
status: draft
ticket: null
title: "OpenWebUI → Letta File Sync (One-Way)"
---

# Implementation Plan: OpenWebUI → Letta File Sync

## Overview

Enable users to upload files via OpenWebUI that sync to Letta folders. Agents can read synced files via built-in file tools.

**Key Design Decisions:**
- **MD files → Notes** (editable documents in OpenWebUI)
- **PDF files → Knowledge collections** (Letta handles PDF→MD via Mistral OCR)
- **One-way sync**: OpenWebUI → Letta (agent write tools deferred)
- **OpenWebUI as pure UI** - no vector DB usage, Letta handles all RAG
- **Shared + Private folders** with OpenWebUI access control mapping

## Current State Analysis

### Existing Infrastructure

| Component | Status | Location |
|-----------|--------|----------|
| OpenWebUI → Letta Knowledge sync | **Planned** | `thoughts/shared/plans/2026-01-09-openwebui-knowledge-sync.md` |
| OpenWebUI Notes research | **Researched** | `thoughts/shared/research/2026-01-09-openwebui-notes-letta-sync.md` |
| Agent file tools (read-only) | **Available** | Letta built-in: `open_files`, `grep_files`, `semantic_search_files` |

### Key Discoveries from Research

1. **Letta file content IS available** via SDK:
   ```python
   # List files with content
   files = client.folders.files.list(folder_id=folder_id, include_content=True)

   # Get single file with content
   file = client.file_manager.get_file_by_id(file_id, actor, include_content=True)
   file.content  # Full text content as string
   ```

2. **Letta ALREADY uses Mistral OCR** for PDF→MD conversion:
   - When `MISTRAL_API_KEY` is set, Letta uses `MistralFileParser`
   - Falls back to `MarkitdownFileParser` otherwise
   - Converted text stored in `FileMetadata.content` field
   - **We don't need custom PDF conversion**

3. **OpenWebUI Access Control Model**:
   ```python
   access_control = {
       "read": {"group_ids": [...], "user_ids": [...]},
       "write": {"group_ids": [...], "user_ids": [...]}
   }
   ```
   - `None` = public read access
   - `{}` = private (owner only)

## Desired End State

### User Experience

1. User uploads file in OpenWebUI chat
2. MD files appear as editable Notes, PDFs go to Knowledge collection
3. Files sync to Letta folders within 30 seconds
4. Agent can read and search uploaded files via built-in tools
5. Access controls from OpenWebUI are respected (private vs shared)

### Verification

- [ ] Upload MD file → appears as Note in OpenWebUI → syncs to Letta folder
- [ ] Upload PDF → appears in Knowledge collection → syncs to Letta (as MD)
- [ ] Agent can search and read synced files
- [ ] Private files only accessible to owner's agent
- [ ] `/sync/status` shows running state
- [ ] `/sync/mappings` shows correct mappings

## What We're NOT Doing (Deferred)

- **Agent file write tools** - requires more research on diff/edit tooling
- **Bidirectional sync** - depends on write tools
- **Real-time sync for edits** - depends on write tools
- **Per-file edit blacklists** - future iteration
- **DOCX/TXT/image support** - only MD and PDF for now
- **Webhook-based sync** - OpenWebUI webhooks not stable

## Implementation Approach

### Architecture (Simplified)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        OpenWebUI (Pure UI)                                  │
│  • File upload in chat                                                      │
│  • Notes management                                                         │
│  • Knowledge collection management                                          │
│  • Access control UI                                                        │
└────────────────────────────────────────────────────────────────────────────┘
           │ upload
           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                      Sync Service (One-Way)                                 │
│  • Poll OpenWebUI for Notes + Knowledge changes                            │
│  • Upload files to Letta folders                                            │
│  • Letta handles PDF→MD via Mistral OCR                                    │
│  • Maintain mapping tables                                                  │
│  • Respect access_control for folder isolation                             │
└────────────────────────────────────────────────────────────────────────────┘
           │ sync
           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                      Letta (Agent Runtime)                                  │
│  • Folders with synced files                                                │
│  • Built-in file tools: open_files, grep_files, semantic_search_files      │
│  • PDF→MD conversion via Mistral OCR (built-in)                            │
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
    └────┬────┘
         ▼
   Sync to Letta folder
   (Letta handles PDF→MD)
```

---

## Phase 1: Core Sync Service (OpenWebUI → Letta)

### Overview

Implement sync service that pulls Notes and Knowledge from OpenWebUI into Letta folders.

### Changes Required

#### 1. Configuration Updates

**File**: `src/letta_starter/config/settings.py`

Add to `ServiceSettings`:

```python
# File Sync configuration
file_sync_enabled: bool = Field(
    default=False,
    description="Enable file sync from OpenWebUI to Letta",
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
    description="Embedding model for Letta folders (must match agent)",
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
    content: str  # markdown from data.content.md
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
        resp = await self.client.get("/api/v1/notes/", params={"page": page})
        resp.raise_for_status()
        notes = []
        for item in resp.json():
            notes.append(OpenWebUINote(
                id=item["id"],
                user_id=item["user_id"],
                title=item["title"],
                content=item.get("data", {}).get("content", {}).get("md", ""),
                access_control=item.get("access_control"),
                created_at=datetime.fromisoformat(item["created_at"]) if item.get("created_at") else datetime.now(),
                updated_at=datetime.fromisoformat(item["updated_at"]) if item.get("updated_at") else datetime.now(),
            ))
        return notes

    async def get_note(self, note_id: str) -> OpenWebUINote:
        """Get note with content."""
        resp = await self.client.get(f"/api/v1/notes/{note_id}")
        resp.raise_for_status()
        item = resp.json()
        return OpenWebUINote(
            id=item["id"],
            user_id=item["user_id"],
            title=item["title"],
            content=item.get("data", {}).get("content", {}).get("md", ""),
            access_control=item.get("access_control"),
            created_at=datetime.fromisoformat(item["created_at"]) if item.get("created_at") else datetime.now(),
            updated_at=datetime.fromisoformat(item["updated_at"]) if item.get("updated_at") else datetime.now(),
        )

    async def list_knowledge(self, page: int = 1) -> list[OpenWebUIKnowledge]:
        """List all knowledge collections."""
        resp = await self.client.get("/api/v1/knowledge/")
        resp.raise_for_status()
        collections = []
        for item in resp.json():
            files = [
                OpenWebUIFile(
                    id=f["id"],
                    user_id=item["user_id"],
                    filename=f["filename"],
                    content_type=f.get("meta", {}).get("content_type", ""),
                    size=f.get("meta", {}).get("size", 0),
                    created_at=datetime.now(),
                    updated_at=datetime.now(),
                )
                for f in item.get("files", [])
            ]
            collections.append(OpenWebUIKnowledge(
                id=item["id"],
                user_id=item["user_id"],
                name=item["name"],
                description=item.get("description", ""),
                files=files,
                access_control=item.get("access_control"),
                created_at=datetime.fromisoformat(item["created_at"]) if item.get("created_at") else datetime.now(),
                updated_at=datetime.fromisoformat(item["updated_at"]) if item.get("updated_at") else datetime.now(),
            ))
        return collections

    async def get_file_content(self, file_id: str) -> bytes:
        """Download file content."""
        resp = await self.client.get(f"/api/v1/files/{file_id}/content")
        resp.raise_for_status()
        return resp.content

    async def close(self) -> None:
        """Close HTTP client."""
        await self.client.aclose()
```

#### 3. Sync Mapping Storage

**File**: `src/letta_starter/server/sync/mappings.py`

```python
"""Mapping storage for sync state."""

from dataclasses import dataclass, field, asdict
from datetime import datetime
import hashlib
import json
from pathlib import Path
from typing import Any

@dataclass
class NoteMapping:
    openwebui_note_id: str
    letta_folder_id: str
    letta_file_id: str | None
    title: str
    content_hash: str
    last_synced: str  # ISO format for JSON serialization
    status: str  # synced, pending, error

@dataclass
class FileMapping:
    openwebui_file_id: str
    openwebui_knowledge_id: str
    letta_folder_id: str
    letta_file_id: str | None
    filename: str
    content_hash: str
    last_synced: str
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
        if not self.storage_path.exists():
            return
        try:
            data = json.loads(self.storage_path.read_text())
            for note_id, m in data.get("notes", {}).items():
                self.note_mappings[note_id] = NoteMapping(**m)
            for file_id, m in data.get("files", {}).items():
                self.file_mappings[file_id] = FileMapping(**m)
        except Exception:
            pass

    def _save(self) -> None:
        """Persist mappings to disk."""
        self.storage_path.parent.mkdir(parents=True, exist_ok=True)
        data = {
            "notes": {k: asdict(v) for k, v in self.note_mappings.items()},
            "files": {k: asdict(v) for k, v in self.file_mappings.items()},
        }
        self.storage_path.write_text(json.dumps(data, indent=2))

    def get_note_mapping(self, openwebui_note_id: str) -> NoteMapping | None:
        return self.note_mappings.get(openwebui_note_id)

    def set_note_mapping(self, mapping: NoteMapping) -> None:
        self.note_mappings[mapping.openwebui_note_id] = mapping
        self._save()

    def get_file_mapping(self, openwebui_file_id: str) -> FileMapping | None:
        return self.file_mappings.get(openwebui_file_id)

    def set_file_mapping(self, mapping: FileMapping) -> None:
        self.file_mappings[mapping.openwebui_file_id] = mapping
        self._save()

    @staticmethod
    def compute_hash(content: str | bytes) -> str:
        """Compute content hash for change detection."""
        if isinstance(content, str):
            content = content.encode()
        return hashlib.sha256(content).hexdigest()[:16]
```

#### 4. Sync Service Core

**File**: `src/letta_starter/server/sync/service.py`

```python
"""One-way sync service: OpenWebUI → Letta."""

import asyncio
import logging
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
from typing import TYPE_CHECKING
import io

from letta_client import Letta

from .openwebui_client import OpenWebUIClient, OpenWebUINote, OpenWebUIKnowledge
from .mappings import SyncMappingStore, NoteMapping, FileMapping

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
    One-way sync: OpenWebUI → Letta.

    Notes and Knowledge files from OpenWebUI are uploaded to Letta folders.
    Letta handles PDF→MD conversion via built-in Mistral OCR.
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
        self._last_sync: datetime | None = None
        self._last_stats: SyncStats | None = None

        # Cache for folder lookups: name → folder_id
        self._folder_cache: dict[str, str] = {}

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
                self._last_stats = await self.sync_all()
                self._last_sync = datetime.now()
            except Exception as e:
                logger.error(f"Sync cycle failed: {e}", exc_info=True)
            await asyncio.sleep(self.settings.file_sync_interval)

    async def sync_all(self) -> SyncStats:
        """Perform full sync cycle: OpenWebUI → Letta."""
        start = datetime.now()
        stats = SyncStats()

        await self._sync_notes(stats)
        await self._sync_knowledge(stats)

        stats.duration_ms = int((datetime.now() - start).total_seconds() * 1000)
        logger.info(f"Sync complete: {stats}")
        return stats

    async def _sync_notes(self, stats: SyncStats) -> None:
        """Sync OpenWebUI Notes → Letta folders."""
        try:
            notes = await self.openwebui.list_notes()
        except Exception as e:
            logger.error(f"Failed to list notes: {e}")
            return

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

                # Upload to Letta (delete old file if updating)
                if mapping and mapping.letta_file_id:
                    try:
                        self.letta.folders.files.delete(
                            folder_id=mapping.letta_folder_id,
                            file_id=mapping.letta_file_id,
                        )
                    except Exception:
                        pass

                file_id = await self._upload_file(
                    folder_id=folder_id,
                    filename=f"{note.title}.md",
                    content=note.content.encode(),
                )

                # Update mapping
                self.mappings.set_note_mapping(NoteMapping(
                    openwebui_note_id=note.id,
                    letta_folder_id=folder_id,
                    letta_file_id=file_id,
                    title=note.title,
                    content_hash=content_hash,
                    last_synced=datetime.now().isoformat(),
                    status="synced",
                ))
                stats.notes_synced += 1
                logger.info(f"Synced note: {note.title}")

            except Exception as e:
                logger.error(f"Failed to sync note {note.id}: {e}")
                stats.notes_failed += 1

    async def _sync_knowledge(self, stats: SyncStats) -> None:
        """Sync OpenWebUI Knowledge → Letta folders.

        Letta handles PDF→MD conversion via Mistral OCR automatically.
        """
        try:
            collections = await self.openwebui.list_knowledge()
        except Exception as e:
            logger.error(f"Failed to list knowledge: {e}")
            return

        for collection in collections:
            # Determine folder name based on access_control
            folder_name = self._get_folder_name_for_knowledge(collection)
            folder_id = await self._ensure_folder(folder_name)

            for file in collection.files:
                try:
                    mapping = self.mappings.get_file_mapping(file.id)

                    # Get file content from OpenWebUI
                    content = await self.openwebui.get_file_content(file.id)
                    content_hash = SyncMappingStore.compute_hash(content)

                    # Skip if unchanged
                    if mapping and mapping.content_hash == content_hash:
                        stats.files_skipped += 1
                        continue

                    # Delete old file if updating
                    if mapping and mapping.letta_file_id:
                        try:
                            self.letta.folders.files.delete(
                                folder_id=mapping.letta_folder_id,
                                file_id=mapping.letta_file_id,
                            )
                        except Exception:
                            pass

                    # Upload to Letta - Letta handles PDF→MD via Mistral OCR
                    file_id = await self._upload_file(
                        folder_id=folder_id,
                        filename=file.filename,
                        content=content,
                        content_type=file.content_type,
                    )

                    # Update mapping
                    self.mappings.set_file_mapping(FileMapping(
                        openwebui_file_id=file.id,
                        openwebui_knowledge_id=collection.id,
                        letta_folder_id=folder_id,
                        letta_file_id=file_id,
                        filename=file.filename,
                        content_hash=content_hash,
                        last_synced=datetime.now().isoformat(),
                        status="synced",
                    ))
                    stats.files_synced += 1
                    logger.info(f"Synced file: {file.filename}")

                except Exception as e:
                    logger.error(f"Failed to sync file {file.id}: {e}")
                    stats.files_failed += 1

    def _get_folder_name_for_note(self, note: OpenWebUINote) -> str:
        """Determine folder name based on note access control."""
        if note.access_control is None:
            return "shared_notes"
        elif note.access_control == {}:
            return f"user_{note.user_id}_notes"
        else:
            # Granular access → user folder (simplified)
            return f"user_{note.user_id}_notes"

    def _get_folder_name_for_knowledge(self, collection: OpenWebUIKnowledge) -> str:
        """Determine folder name based on knowledge access control."""
        if collection.access_control is None:
            return f"shared_knowledge_{collection.id[:8]}"
        elif collection.access_control == {}:
            return f"user_{collection.user_id}_knowledge"
        else:
            return f"user_{collection.user_id}_knowledge"

    async def _ensure_folder(self, name: str) -> str:
        """Get or create Letta folder by name."""
        if name in self._folder_cache:
            return self._folder_cache[name]

        # Search for existing folder
        folders = self.letta.folders.list(name=name)
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
        logger.info(f"Created folder: {name}")
        return folder.id

    async def _upload_file(
        self,
        folder_id: str,
        filename: str,
        content: bytes,
        content_type: str | None = None,
    ) -> str:
        """Upload file to Letta folder and wait for processing."""
        file_obj = io.BytesIO(content)
        file_obj.name = filename

        # Upload returns a job for async processing
        job = self.letta.folders.files.upload(
            folder_id=folder_id,
            file=file_obj,
        )

        # Wait for processing to complete
        while True:
            job = self.letta.jobs.retrieve(job.id)
            if job.status == "completed":
                break
            elif job.status == "failed":
                raise RuntimeError(f"File upload failed: {job.error}")
            await asyncio.sleep(0.5)

        # Find the uploaded file to get its ID
        files = self.letta.folders.files.list(folder_id=folder_id)
        for f in files:
            if f.file_name == filename or f.original_file_name == filename:
                return f.id

        raise RuntimeError(f"File {filename} not found after upload")

    def get_status(self) -> dict:
        """Get sync service status."""
        return {
            "enabled": self.settings.file_sync_enabled,
            "running": self._running,
            "last_sync": self._last_sync.isoformat() if self._last_sync else None,
            "last_stats": self._last_stats,
        }
```

#### 5. Sync Router

**File**: `src/letta_starter/server/sync/router.py`

```python
"""HTTP endpoints for file sync management."""

from fastapi import APIRouter, Depends

from .service import FileSyncService, SyncStats

router = APIRouter(prefix="/sync", tags=["sync"])

# Dependency will be set up in main.py
_sync_service: FileSyncService | None = None

def get_file_sync() -> FileSyncService:
    if _sync_service is None:
        raise RuntimeError("Sync service not initialized")
    return _sync_service

def set_file_sync(service: FileSyncService) -> None:
    global _sync_service
    _sync_service = service

@router.get("/status")
async def get_sync_status(
    sync: FileSyncService = Depends(get_file_sync),
) -> dict:
    """Get sync service status."""
    return sync.get_status()

@router.post("/trigger")
async def trigger_sync(
    sync: FileSyncService = Depends(get_file_sync),
) -> SyncStats:
    """Manually trigger a sync cycle."""
    return await sync.sync_all()

@router.get("/mappings")
async def list_mappings(
    sync: FileSyncService = Depends(get_file_sync),
) -> dict:
    """List all sync mappings."""
    return {
        "notes": [
            {
                "openwebui_note_id": m.openwebui_note_id,
                "letta_folder_id": m.letta_folder_id,
                "letta_file_id": m.letta_file_id,
                "title": m.title,
                "last_synced": m.last_synced,
                "status": m.status,
            }
            for m in sync.mappings.note_mappings.values()
        ],
        "files": [
            {
                "openwebui_file_id": m.openwebui_file_id,
                "openwebui_knowledge_id": m.openwebui_knowledge_id,
                "letta_folder_id": m.letta_folder_id,
                "letta_file_id": m.letta_file_id,
                "filename": m.filename,
                "last_synced": m.last_synced,
                "status": m.status,
            }
            for m in sync.mappings.file_mappings.values()
        ],
    }

@router.post("/attach/{agent_id}/{folder_name}")
async def attach_folder_to_agent(
    agent_id: str,
    folder_name: str,
    sync: FileSyncService = Depends(get_file_sync),
) -> dict:
    """Attach a synced folder to an agent."""
    folder_id = await sync._ensure_folder(folder_name)
    sync.letta.agents.folders.attach(
        agent_id=agent_id,
        folder_id=folder_id,
    )
    return {"status": "attached", "folder_id": folder_id}
```

#### 6. Main App Integration

**File**: `src/letta_starter/server/main.py`

Add to lifespan:

```python
from letta_starter.server.sync.service import FileSyncService
from letta_starter.server.sync.router import router as sync_router, set_file_sync

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... existing startup code ...

    # Initialize file sync service if enabled
    if settings.file_sync_enabled:
        sync_service = FileSyncService(
            settings=settings,
            letta_client=letta_client,
        )
        set_file_sync(sync_service)
        await sync_service.start()
        logger.info("File sync service started")

    yield

    # ... existing shutdown code ...

    # Stop sync service
    if settings.file_sync_enabled:
        await sync_service.stop()

# Register router
app.include_router(sync_router)
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`
- [ ] Sync service starts without errors

#### Manual Verification:
- [ ] Create a Note in OpenWebUI → appears in Letta folder within 30s
- [ ] Upload PDF to Knowledge collection → appears in Letta (processed by Mistral)
- [ ] `GET /sync/status` returns running state
- [ ] `GET /sync/mappings` shows correct mappings
- [ ] Agent with attached folder can search and read files

---

## Phase 2: Access Control & Folder Attachment

### Overview

Implement TOML-based folder configuration and automatic attachment during agent creation.

### Changes Required

#### 1. TOML Schema Updates

**File**: `src/letta_starter/curriculum/schema.py`

```python
class AgentFoldersConfig(BaseModel):
    """Configuration for agent folder access."""

    shared: list[str] = Field(
        default_factory=list,
        description="List of shared folder names to attach",
    )
    private: bool = Field(
        default=True,
        description="Whether to attach user's private folder",
    )
```

#### 2. Course TOML Configuration

**File**: `config/courses/college-essay/course.toml`

```toml
[agent]
name = "Essay Coach"
model = "claude-sonnet-4-20250514"

[agent.folders]
# Shared course materials (all users)
shared = ["college-essay-materials"]
# User gets a private folder for their work
private = true
```

#### 3. Folder Attachment on Agent Creation

**File**: `src/letta_starter/server/agents.py`

```python
async def attach_folders_for_agent(
    letta_client: Letta,
    agent_id: str,
    user_id: str,
    folder_config: AgentFoldersConfig,
    sync_service: FileSyncService | None,
) -> list[str]:
    """Attach configured folders to agent."""
    if not sync_service:
        return []

    attached = []

    # Attach shared folders
    for folder_name in folder_config.shared:
        folder_id = await sync_service._ensure_folder(folder_name)
        letta_client.agents.folders.attach(agent_id=agent_id, folder_id=folder_id)
        attached.append(folder_name)

    # Attach user's private folder
    if folder_config.private:
        private_folder = f"user_{user_id}_notes"
        folder_id = await sync_service._ensure_folder(private_folder)
        letta_client.agents.folders.attach(agent_id=agent_id, folder_id=folder_id)
        attached.append(private_folder)

    return attached
```

### Success Criteria

#### Automated Verification:
- [ ] TOML parsing includes folder config
- [ ] Agent creation attaches correct folders
- [ ] Type checking passes

#### Manual Verification:
- [ ] Create agent → shared and private folders attached
- [ ] Agent can search files in attached folders
- [ ] Private notes only visible to owner's agent

---

## Testing Strategy

### Unit Tests

- `tests/test_sync_service.py` - Sync logic, mapping management
- `tests/test_openwebui_client.py` - API client (mocked)
- `tests/test_sync_mappings.py` - Mapping persistence

### Integration Tests

- `tests/integration/test_sync_integration.py` - Full sync cycle with mocked APIs

### Manual Testing Steps

1. **Basic Sync**
   - Enable sync in `.env`
   - Create Note in OpenWebUI
   - Wait 30 seconds
   - Verify via `GET /sync/mappings`
   - Verify file in Letta via ADE

2. **PDF Conversion**
   - Upload PDF to Knowledge collection
   - Wait for sync + Letta processing
   - Verify MD content via agent query

3. **Agent Access**
   - Create agent with folder attached
   - Ask agent about file contents
   - Verify agent can search and open files

4. **Access Control**
   - Create private Note (user A)
   - Create agent for user B
   - Verify user B's agent cannot see user A's Note

---

## Environment Variables

```bash
# Add to .env
YOULAB_SERVICE_FILE_SYNC_ENABLED=true
YOULAB_SERVICE_OPENWEBUI_URL=http://localhost:3000
YOULAB_SERVICE_OPENWEBUI_API_KEY=sk-xxxx
YOULAB_SERVICE_FILE_SYNC_INTERVAL=30
YOULAB_SERVICE_FILE_SYNC_EMBEDDING_MODEL=openai/text-embedding-3-small

# Required for Letta PDF→MD conversion
MISTRAL_API_KEY=your-mistral-api-key
```

---

## Dependencies

**No new dependencies required** - uses existing:
- `httpx` - Async HTTP client
- `letta-client` - Letta SDK

Letta's built-in Mistral OCR handles PDF conversion.

---

## Future Enhancements (TODOs)

These items are deferred and should be implemented in future iterations:

### Agent File Write Tools
```python
# TODO(file-write): Implement edit_file tool for agents
# Requires research on diff/edit tooling for good UX
# See: thoughts/shared/plans/2026-01-10-file-knowledge-sync.md

# TODO(file-write): Implement create_file tool for agents

# TODO(file-write): Implement bidirectional sync (Letta → OpenWebUI)

# TODO(file-write): Implement real-time sync for agent edits
```

### Other Enhancements
- Per-file edit blacklist in TOML
- Version history for file edits
- Conflict detection with user notification
- Webhook-based sync (when OpenWebUI supports it)
- DOCX/TXT/image support with conversion

---

## References

- Existing plan: `thoughts/shared/plans/2026-01-09-openwebui-knowledge-sync.md`
- Notes research: `thoughts/shared/research/2026-01-09-openwebui-notes-letta-sync.md`
- Letta filesystem: https://docs.letta.com/guides/agents/filesystem
- OpenWebUI API: https://docs.openwebui.com/getting-started/api-endpoints/
- Letta repo (cloned): `letta-repo/` - see `letta/services/file_processor/parser/mistral_parser.py`
