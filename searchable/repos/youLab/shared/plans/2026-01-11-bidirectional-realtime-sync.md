---
date: 2026-01-11T17:45:00+07:00
author: claude
status: ready
ticket: null
title: "Bidirectional Real-Time Sync: Letta Folders ↔ OpenWebUI Knowledge"
priority: high
---

# Implementation Plan: Bidirectional Real-Time Sync

## Overview

Make Letta folders appear as OpenWebUI Knowledge collections with immediate, seamless sync in both directions.

**Requirements:**
- Letta folders visible as Knowledge collections in OpenWebUI UI
- Immediate sync on upload/update (no polling delay)
- Bidirectional: changes in either system sync to the other

## Current State

```
OpenWebUI Knowledge → FileSyncService (30s poll) → Letta Folders
        ↑                                              ↓
   User sees                                    Agent accesses
                    ← NO REVERSE SYNC ←
```

## Desired End State

```
OpenWebUI Knowledge ←─────→ Letta Folders
        ↑          IMMEDIATE          ↓
   User sees                    Agent accesses
        ↑                             ↓
   Both systems stay in sync (< 1 second)
```

## API Endpoints Available

### OpenWebUI (discovered via research)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/knowledge/create` | POST | Create knowledge collection |
| `/api/v1/knowledge/` | GET | List all collections |
| `/api/v1/knowledge/{id}` | GET | Get single collection |
| `/api/v1/knowledge/{id}/file/add` | POST | Add file to collection |
| `/api/v1/files/` | POST | Upload file |
| `/api/v1/files/{id}/content` | GET | Download file content |
| Webhooks | - | 28+ event types including `knowledge` category |

### Letta

| Endpoint | Purpose |
|----------|---------|
| `letta.folders.create()` | Create folder |
| `letta.folders.list()` | List folders |
| `letta.folders.files.upload()` | Upload file to folder |
| `letta.folders.files.list()` | List files in folder |
| `letta.folders.files.delete()` | Delete file |

## Architecture

### Phase 1: Reverse Sync (Letta → OpenWebUI) - PRIORITY

Enable Letta folders to appear as Knowledge collections.

```
Agent creates/modifies file in Letta folder
              │
              ▼
      Sync Service detects change
              │
              ▼
      Create/Update OpenWebUI Knowledge collection
              │
              ▼
      User sees changes in OpenWebUI UI
```

### Phase 2: Webhook-Based Forward Sync

Replace polling with webhooks for immediate sync.

```
User uploads file in OpenWebUI
              │
              ▼
      OpenWebUI fires webhook event
              │
              ▼
      YouLab webhook endpoint receives event
              │
              ▼
      Sync file to Letta folder immediately
```

### Phase 3: Conflict Resolution

Handle simultaneous edits in both systems.

---

## Phase 1: Reverse Sync Implementation

### 1.1 Extend OpenWebUI Client

**File**: `src/youlab_server/server/sync/openwebui_client.py`

Add write methods:

```python
async def create_knowledge(
    self,
    name: str,
    description: str = "",
    access_control: dict | None = None,
) -> str:
    """
    Create a knowledge collection in OpenWebUI.

    Args:
        name: Collection name (will match Letta folder name)
        description: Optional description
        access_control: Access control settings

    Returns:
        Knowledge collection ID
    """
    resp = await self.client.post(
        "/api/v1/knowledge/create",
        json={
            "name": name,
            "description": description,
            "data": {},
            "access_control": access_control or {},
        },
    )
    resp.raise_for_status()
    return resp.json()["id"]

async def upload_file(self, filename: str, content: bytes, content_type: str = "text/markdown") -> str:
    """
    Upload a file to OpenWebUI.

    Returns:
        File ID
    """
    files = {"file": (filename, content, content_type)}
    resp = await self.client.post("/api/v1/files/", files=files)
    resp.raise_for_status()
    return resp.json()["id"]

async def add_file_to_knowledge(self, knowledge_id: str, file_id: str) -> None:
    """Add an uploaded file to a knowledge collection."""
    resp = await self.client.post(
        f"/api/v1/knowledge/{knowledge_id}/file/add",
        json={"file_id": file_id},
    )
    resp.raise_for_status()

async def delete_file(self, file_id: str) -> None:
    """Delete a file from OpenWebUI."""
    resp = await self.client.delete(f"/api/v1/files/{file_id}")
    resp.raise_for_status()

async def delete_knowledge(self, knowledge_id: str) -> None:
    """Delete a knowledge collection."""
    resp = await self.client.delete(f"/api/v1/knowledge/{knowledge_id}")
    resp.raise_for_status()
```

### 1.2 Extend Mapping Storage

**File**: `src/youlab_server/server/sync/mappings.py`

Add folder-to-knowledge mapping:

```python
@dataclass
class FolderMapping:
    """Tracks Letta folder ↔ OpenWebUI Knowledge mapping."""
    letta_folder_id: str
    letta_folder_name: str
    openwebui_knowledge_id: str
    last_synced: str
    status: str  # synced, pending, error

class SyncMappingStore:
    def __init__(self, storage_path: Path):
        # ... existing code ...
        self.folder_mappings: dict[str, FolderMapping] = {}  # keyed by letta_folder_id

    def get_folder_mapping(self, letta_folder_id: str) -> FolderMapping | None:
        return self.folder_mappings.get(letta_folder_id)

    def get_folder_mapping_by_name(self, folder_name: str) -> FolderMapping | None:
        for m in self.folder_mappings.values():
            if m.letta_folder_name == folder_name:
                return m
        return None

    def set_folder_mapping(self, mapping: FolderMapping) -> None:
        self.folder_mappings[mapping.letta_folder_id] = mapping
        self._save()
```

### 1.3 Add Reverse Sync to Service

**File**: `src/youlab_server/server/sync/service.py`

Add method to sync Letta folder → OpenWebUI Knowledge:

```python
async def ensure_knowledge_for_folder(self, folder_name: str, folder_id: str) -> str:
    """
    Ensure OpenWebUI Knowledge collection exists for Letta folder.
    Creates if not exists, returns Knowledge ID.
    """
    # Check mapping first
    mapping = self.mappings.get_folder_mapping(folder_id)
    if mapping and mapping.status == "synced":
        return mapping.openwebui_knowledge_id

    # Check if Knowledge exists by name
    collections = await self.openwebui.list_knowledge()
    for coll in collections:
        if coll.name == folder_name:
            # Found existing - create mapping
            self.mappings.set_folder_mapping(FolderMapping(
                letta_folder_id=folder_id,
                letta_folder_name=folder_name,
                openwebui_knowledge_id=coll.id,
                last_synced=datetime.now().isoformat(),
                status="synced",
            ))
            return coll.id

    # Create new Knowledge collection
    knowledge_id = await self.openwebui.create_knowledge(
        name=folder_name,
        description=f"Synced from Letta folder: {folder_name}",
    )

    self.mappings.set_folder_mapping(FolderMapping(
        letta_folder_id=folder_id,
        letta_folder_name=folder_name,
        openwebui_knowledge_id=knowledge_id,
        last_synced=datetime.now().isoformat(),
        status="synced",
    ))

    logger.info(f"Created OpenWebUI Knowledge for folder: {folder_name}")
    return knowledge_id

async def sync_file_to_openwebui(
    self,
    folder_id: str,
    folder_name: str,
    letta_file_id: str,
    filename: str,
    content: bytes,
) -> str:
    """
    Sync a Letta file to OpenWebUI Knowledge.

    Returns:
        OpenWebUI file ID
    """
    # Ensure Knowledge collection exists
    knowledge_id = await self.ensure_knowledge_for_folder(folder_name, folder_id)

    # Upload file to OpenWebUI
    openwebui_file_id = await self.openwebui.upload_file(filename, content)

    # Add to Knowledge collection
    await self.openwebui.add_file_to_knowledge(knowledge_id, openwebui_file_id)

    logger.info(f"Synced file to OpenWebUI: {filename} → {folder_name}")
    return openwebui_file_id
```

### 1.4 Modify ensure_folder for Immediate Sync

**File**: `src/youlab_server/server/sync/service.py`

Update `ensure_folder()` to create OpenWebUI Knowledge immediately:

```python
async def ensure_folder(self, name: str) -> str:
    """
    Get or create Letta folder by name.
    ALSO creates corresponding OpenWebUI Knowledge collection.
    """
    if name in self._folder_cache:
        folder_id = self._folder_cache[name]
        # Ensure OpenWebUI Knowledge exists (idempotent)
        await self.ensure_knowledge_for_folder(name, folder_id)
        return folder_id

    # Search for existing folder
    folders = self.letta.folders.list(name=name)
    for folder in folders:
        if folder.name == name:
            if folder.id is None:
                raise RuntimeError(f"Letta returned folder without ID: {name}")
            self._folder_cache[name] = folder.id
            # Ensure OpenWebUI Knowledge exists
            await self.ensure_knowledge_for_folder(name, folder.id)
            return folder.id

    # Create new folder in Letta
    folder = self.letta.folders.create(
        name=name,
        embedding=self.settings.file_sync_embedding_model,
    )
    if folder.id is None:
        raise RuntimeError(f"Letta returned new folder without ID: {name}")
    self._folder_cache[name] = folder.id

    # Create corresponding OpenWebUI Knowledge
    await self.ensure_knowledge_for_folder(name, folder.id)

    logger.info(f"Created folder with OpenWebUI mirror: {name}")
    return folder.id
```

### 1.5 Add Reverse Sync to Upload Flow

**File**: `src/youlab_server/server/sync/service.py`

Extend `_upload_file()` to also sync to OpenWebUI:

```python
async def _upload_file(
    self,
    folder_id: str,
    filename: str,
    content: bytes,
) -> str:
    """
    Upload file to Letta folder AND sync to OpenWebUI Knowledge.
    """
    # Upload to Letta (existing logic)
    file_obj = io.BytesIO(content)
    file_obj.name = filename

    job = self.letta.folders.files.upload(folder_id=folder_id, file=file_obj)
    if job.id is None:
        raise RuntimeError("Letta returned upload job without ID")

    # Wait for Letta processing
    job_id = job.id
    while True:
        job = self.letta.jobs.retrieve(job_id)
        if job.status == "completed":
            break
        if job.status == "failed":
            raise RuntimeError(f"File upload failed: {job.error}")
        await asyncio.sleep(0.5)

    # Find the uploaded file
    files = self.letta.folders.files.list(folder_id=folder_id)
    letta_file_id = None
    for f in files:
        if filename in (f.file_name, f.original_file_name):
            if f.id is None:
                raise RuntimeError(f"Letta returned file without ID: {filename}")
            letta_file_id = f.id
            break

    if not letta_file_id:
        raise RuntimeError(f"File {filename} not found after upload")

    # REVERSE SYNC: Also upload to OpenWebUI
    folder_name = self._get_folder_name_by_id(folder_id)
    if folder_name:
        await self.sync_file_to_openwebui(
            folder_id=folder_id,
            folder_name=folder_name,
            letta_file_id=letta_file_id,
            filename=filename,
            content=content,
        )

    return letta_file_id

def _get_folder_name_by_id(self, folder_id: str) -> str | None:
    """Lookup folder name from ID using cache."""
    for name, fid in self._folder_cache.items():
        if fid == folder_id:
            return name
    return None
```

---

## Phase 2: Webhook-Based Forward Sync

### 2.1 Add Webhook Endpoint

**File**: `src/youlab_server/server/sync/router.py`

```python
@router.post("/webhook")
async def handle_openwebui_webhook(
    request: Request,
    sync: SyncDep,
) -> dict[str, str]:
    """
    Receive OpenWebUI webhook events for immediate sync.

    Configure in OpenWebUI Admin → Settings → Webhooks:
    URL: http://youlab-server:8100/sync/webhook
    Events: knowledge.*, file.*
    """
    payload = await request.json()
    event_type = payload.get("event")

    logger.info(f"Received webhook: {event_type}")

    if event_type in ("knowledge.created", "knowledge.updated"):
        # Sync knowledge collection to Letta
        knowledge_id = payload.get("data", {}).get("id")
        if knowledge_id:
            await sync.sync_knowledge_by_id(knowledge_id)

    elif event_type == "knowledge.file.added":
        # Sync new file to Letta
        file_id = payload.get("data", {}).get("file_id")
        knowledge_id = payload.get("data", {}).get("knowledge_id")
        if file_id and knowledge_id:
            await sync.sync_file_by_id(file_id, knowledge_id)

    elif event_type == "file.uploaded":
        # Check if file is in a knowledge collection we're tracking
        file_id = payload.get("data", {}).get("id")
        if file_id:
            await sync.check_and_sync_file(file_id)

    return {"status": "ok"}
```

### 2.2 Add Targeted Sync Methods

**File**: `src/youlab_server/server/sync/service.py`

```python
async def sync_knowledge_by_id(self, knowledge_id: str) -> None:
    """Sync a specific knowledge collection immediately."""
    collections = await self.openwebui.list_knowledge()
    for coll in collections:
        if coll.id == knowledge_id:
            folder_name = self._get_folder_name_for_knowledge(coll)
            folder_id = await self.ensure_folder(folder_name)

            for file in coll.files:
                await self._sync_single_file(file, coll, folder_id, SyncStats())
            return

async def sync_file_by_id(self, file_id: str, knowledge_id: str) -> None:
    """Sync a specific file immediately."""
    collections = await self.openwebui.list_knowledge()
    for coll in collections:
        if coll.id == knowledge_id:
            folder_name = self._get_folder_name_for_knowledge(coll)
            folder_id = await self.ensure_folder(folder_name)

            for file in coll.files:
                if file.id == file_id:
                    await self._sync_single_file(file, coll, folder_id, SyncStats())
                    return
```

### 2.3 Configure Webhook in OpenWebUI

Add to documentation/setup:

```bash
# In OpenWebUI Admin Panel:
# Settings → Webhooks → Add Webhook
#
# URL: http://localhost:8100/sync/webhook
# Events: Select all `knowledge.*` and `file.*` events
#
# Or via API:
curl -X POST http://localhost:3000/api/v1/webhooks/ \
  -H "Authorization: Bearer $OPENWEBUI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://localhost:8100/sync/webhook",
    "events": ["knowledge.created", "knowledge.updated", "knowledge.file.added", "file.uploaded"]
  }'
```

---

## Phase 3: Hybrid Approach (Recommended)

For maximum reliability, use **both** mechanisms:

1. **Webhooks** for immediate sync when available
2. **Background polling** as fallback for missed events

```python
class FileSyncService:
    def __init__(self, ...):
        # ... existing init ...
        self._webhook_active = False

    def mark_webhook_active(self) -> None:
        """Called when webhook endpoint receives first event."""
        self._webhook_active = True

    async def _sync_loop(self) -> None:
        """Background loop - runs less frequently when webhooks active."""
        while self._running:
            try:
                # If webhooks are working, just do periodic full sync for consistency
                interval = self.settings.file_sync_interval * 10 if self._webhook_active else self.settings.file_sync_interval

                self._last_stats = await self.sync_all()
                self._last_sync = datetime.now()
            except Exception as e:
                logger.error(f"Sync cycle failed: {e}", exc_info=True)
            await asyncio.sleep(interval)
```

---

## Testing Strategy

### Unit Tests

1. `test_reverse_sync_creates_knowledge` - Creating folder creates Knowledge
2. `test_reverse_sync_uploads_file` - Uploading to Letta syncs to OpenWebUI
3. `test_webhook_triggers_sync` - Webhook event triggers immediate sync
4. `test_mapping_persistence` - Folder mappings are persisted

### Integration Tests

1. Create Letta folder → verify Knowledge appears in OpenWebUI
2. Upload file via OpenWebUI → verify appears in Letta folder immediately
3. Upload file via Letta → verify appears in OpenWebUI Knowledge

### Manual Testing

```bash
# 1. Create agent with folder
curl -X POST http://localhost:8100/agents \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "agent_type": "college-essay"}'

# 2. Check OpenWebUI for Knowledge collection
# → Should see "college-essay-materials" and "user_test_notes"

# 3. Upload file to Knowledge in OpenWebUI
# → Should appear in Letta folder immediately (< 1 second)

# 4. Add file to Letta folder via agent
# → Should appear in OpenWebUI Knowledge immediately
```

---

## Environment Variables

No new variables needed. Uses existing:

```bash
YOULAB_SERVICE_FILE_SYNC_ENABLED=true
YOULAB_SERVICE_OPENWEBUI_URL=http://localhost:3000
YOULAB_SERVICE_OPENWEBUI_API_KEY=sk-xxxx  # Required for write operations
```

---

## Success Criteria

### Phase 1 (Reverse Sync)
- [x] Creating Letta folder creates OpenWebUI Knowledge
- [x] Files uploaded to Letta appear in OpenWebUI Knowledge
- [x] Folder-to-Knowledge mappings are persisted
- [x] All tests pass

### Phase 2 (Webhook Sync)
- [x] Webhook endpoint receives events
- [ ] Knowledge changes sync to Letta immediately
- [ ] File uploads sync within 1 second

### Phase 3 (Full Bidirectional)
- [ ] Changes in either system sync to the other
- [ ] No duplicate files created
- [ ] Conflict handling works correctly

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| OpenWebUI webhook events not stable | Keep polling as fallback |
| Duplicate files on race conditions | Use content hash for deduplication |
| API rate limits | Batch operations, implement backoff |
| Large files timeout | Chunk uploads, async processing |

---

## References

- [OpenWebUI API Endpoints](https://docs.openwebui.com/getting-started/api-endpoints/)
- [OpenWebUI Webhook Enhancements](https://github.com/open-webui/open-webui/discussions/16428)
- [Knowledge Collection API Discussion](https://github.com/open-webui/open-webui/discussions/11761)
- Previous research: `thoughts/shared/research/2026-01-11-openwebui-letta-folder-visibility.md`
- Previous plan: `thoughts/shared/plans/2026-01-10-file-knowledge-sync.md`
