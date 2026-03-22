---
date: 2026-01-09T18:30:00+07:00
author: ariasulin
status: draft
ticket: null
title: "OpenWebUI Knowledge Sync to Letta Folders"
---

# Implementation Plan: OpenWebUI Knowledge Sync to Letta Folders

## Overview

Sync OpenWebUI's Knowledge feature (files, collections) to Letta's Folders API so that:
1. Users upload/manage files via OpenWebUI's Knowledge UI
2. Files are automatically synced to Letta folders
3. Letta agents can access files via built-in tools (`open_file`, `grep_file`, `search_file`)

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        OpenWebUI Knowledge UI                               │
│  • User uploads files, creates collections                                  │
│  • Standard UI, no modifications needed                                     │
└────────────────────────────────────────────────────────────────────────────┘
           │                                              ▲
           ▼ (writes)                                     │ (reads for display)
┌────────────────────────────────────────────────────────────────────────────┐
│                      OpenWebUI Database                                     │
│  • Files table                                                              │
│  • Knowledge collections                                                    │
│  • ChromaDB vectors (ignored - Letta handles RAG)                          │
└────────────────────────────────────────────────────────────────────────────┘
           │
           ▼ (poll for changes)
┌────────────────────────────────────────────────────────────────────────────┐
│                      Sync Service (new)                                     │
│  • Polls OpenWebUI /api/v1/knowledge/* every N seconds                     │
│  • Detects new/updated/deleted files                                        │
│  • Syncs to Letta Folders API                                               │
│  • Maintains mapping: openwebui_file_id ↔ letta_file_id                    │
└────────────────────────────────────────────────────────────────────────────┘
           │
           ▼ (sync)
┌────────────────────────────────────────────────────────────────────────────┐
│                      Letta Server                                           │
│  • Folders with user files                                                  │
│  • Agent file tools (open_file, search_file, grep_file)                    │
│  • THIS is where RAG actually happens                                       │
└────────────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. Configuration (`src/letta_starter/config/settings.py`)

Add to `ServiceSettings`:

```python
# OpenWebUI Knowledge Sync configuration
openwebui_sync_enabled: bool = Field(
    default=False,
    description="Enable OpenWebUI Knowledge sync to Letta",
)
openwebui_url: str = Field(
    default="http://localhost:3000",
    description="OpenWebUI base URL",
)
openwebui_api_key: str | None = Field(
    default=None,
    description="OpenWebUI API key for Knowledge sync",
)
knowledge_sync_interval: int = Field(
    default=30,
    description="Seconds between Knowledge sync cycles",
)
knowledge_embedding_model: str = Field(
    default="openai/text-embedding-3-small",
    description="Embedding model for Letta folders (must match agent embedding)",
)
```

### 2. Sync Service (`src/letta_starter/server/sync/service.py`)

**Class: `KnowledgeSyncService`**

**Responsibilities:**
- Poll OpenWebUI Knowledge API for collections and files
- Detect new/updated/deleted files
- Upload files to Letta folders
- Maintain mapping between OpenWebUI and Letta resources
- Attach folders to agents

**Key Methods:**

| Method | Description |
|--------|-------------|
| `start()` | Start background sync loop |
| `stop()` | Stop sync loop |
| `sync_all()` | Perform full sync cycle |
| `attach_knowledge_to_agent(agent_id, knowledge_id)` | Attach folders to agent |
| `get_sync_status()` | Return current status |
| `get_mappings()` | Return all file mappings |

**Data Structures:**

```python
@dataclass
class SyncMapping:
    openwebui_file_id: str
    openwebui_knowledge_id: str
    letta_folder_id: str
    letta_file_id: str | None
    filename: str
    last_synced: datetime
    status: str  # synced, pending, error

@dataclass
class SyncStats:
    collections_scanned: int
    files_synced: int
    files_skipped: int
    files_failed: int
    duration_ms: int
```

**Naming Conventions:**
- Letta folder: `openwebui_knowledge_{knowledge_id}`
- Files retain original names

### 3. Sync Router (`src/letta_starter/server/sync/router.py`)

**Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/sync/status` | Get sync service status |
| `POST` | `/sync/trigger` | Manually trigger sync cycle |
| `POST` | `/sync/attach` | Attach knowledge to agent |
| `GET` | `/sync/mappings` | List all file mappings |
| `GET` | `/sync/folders` | List synced Letta folders |

### 4. Main App Integration (`src/letta_starter/server/main.py`)

**Lifespan Changes:**
- Initialize `KnowledgeSyncService` if enabled
- Start sync loop as background task
- Stop sync on shutdown

**Router Registration:**
- Add `sync_router` to app

**Helper Function:**
- `get_knowledge_sync()` - Get sync service from app state

### 5. Agent Creation Hook (Optional)

When agent is created, optionally attach all synced knowledge folders:
- Modify `create_agent()` endpoint or `AgentManager.create_agent()`
- Call `sync_service.attach_knowledge_to_agent(agent_id)`

## Sync Logic

### Sync Cycle Flow

```
1. List all Knowledge collections from OpenWebUI
   GET /api/v1/knowledge

2. For each collection:
   a. Get collection details with files
      GET /api/v1/knowledge/{id}

   b. Get or create Letta folder
      - Check folder cache
      - Search Letta folders by name
      - Create if not exists with embedding model

   c. For each file in collection:
      - Check if already synced (in mappings)
      - Check if exists in Letta folder by name
      - If new:
        - Download content: GET /api/v1/files/{id}/content
        - Upload to Letta: client.folders.files.upload()
        - Record mapping

3. Return stats
```

### File Processing

Letta handles file processing automatically:
- Chunking into passages
- Embedding generation
- Vector storage

Processing is async - `upload()` returns a job that can be polled.

### Folder-Agent Attachment

When a folder is attached to an agent:
- Agent automatically gets file tools
- `open_file` - Read file contents
- `grep_file` - Regex search
- `search_file` - Semantic search

## File Structure

```
src/letta_starter/server/sync/
├── __init__.py           # Exports
├── service.py            # KnowledgeSyncService class
└── router.py             # HTTP endpoints + init function
```

## Environment Variables

```bash
# .env additions
YOULAB_SERVICE_OPENWEBUI_SYNC_ENABLED=true
YOULAB_SERVICE_OPENWEBUI_URL=http://localhost:3000
YOULAB_SERVICE_OPENWEBUI_API_KEY=sk-xxxx
YOULAB_SERVICE_KNOWLEDGE_SYNC_INTERVAL=30
YOULAB_SERVICE_KNOWLEDGE_EMBEDDING_MODEL=openai/text-embedding-3-small
```

## OpenWebUI API Reference

**Authentication:** `Authorization: Bearer {api_key}`

**List Knowledge:**
```
GET /api/v1/knowledge
→ [{ id, name, description, files: [...] }, ...]
```

**Get Knowledge Details:**
```
GET /api/v1/knowledge/{id}
→ { id, name, description, files: [{ id, filename, ... }] }
```

**Download File Content:**
```
GET /api/v1/files/{id}/content
→ binary file content
```

## Letta API Reference

**Create Folder:**
```python
folder = client.folders.create(
    name="openwebui_knowledge_{id}",
    embedding="openai/text-embedding-3-small"
)
```

**Upload File:**
```python
job = client.folders.files.upload(
    folder_id=folder.id,
    file=file_obj,
    name=filename,
    duplicate_handling="replace"
)
```

**Attach to Agent:**
```python
client.agents.folders.attach(
    agent_id=agent.id,
    folder_id=folder.id
)
```

## Success Criteria

1. **Sync works:** Files uploaded to OpenWebUI Knowledge appear in Letta folders
2. **Agents can access:** Agents with attached folders can use file tools
3. **Idempotent:** Re-running sync doesn't duplicate files
4. **Status visible:** `/sync/status` shows running state and stats
5. **Manual trigger:** `/sync/trigger` forces immediate sync
6. **Attach works:** `/sync/attach` attaches folders to agents

## Testing Plan

### Unit Tests

- `test_sync_service.py`
  - Test mapping management
  - Test folder cache
  - Test stats accumulation
  - Mock OpenWebUI and Letta APIs

### Integration Tests

- `test_sync_integration.py`
  - Full sync cycle with mocked APIs
  - Verify file upload calls
  - Verify folder creation

### Manual Testing

1. Enable sync in `.env`
2. Start service
3. Upload file to OpenWebUI Knowledge
4. Wait for sync cycle
5. Check `/sync/status` and `/sync/mappings`
6. Create agent
7. Attach knowledge: `POST /sync/attach`
8. Chat with agent, ask about uploaded file

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Large files timeout | Set appropriate httpx timeout (60s) |
| Letta processing fails | Track status in mapping, retry on next cycle |
| OpenWebUI API changes | Version-specific handling, graceful fallbacks |
| Embedding mismatch | Config requires explicit model setting |
| Rate limiting | Configurable sync interval, batch processing |

## Future Enhancements

1. **Bidirectional sync:** Push Letta files to OpenWebUI
2. **Webhook support:** Use OpenWebUI webhooks when available (PR #16426)
3. **Delete sync:** Detect and propagate deletions
4. **Update sync:** Re-sync when file updated_at changes
5. **User-specific folders:** Separate folders per user
6. **Access control:** Respect OpenWebUI collection permissions

## Dependencies

- `httpx` - Async HTTP client (already in project)
- `letta-client` - Letta SDK (already in project)

No new dependencies required.

## Estimated Effort

| Task | Estimate |
|------|----------|
| Settings additions | 15 min |
| Sync service core | 2 hours |
| Sync router | 30 min |
| Main app integration | 30 min |
| Unit tests | 1 hour |
| Integration tests | 1 hour |
| Manual testing & fixes | 1 hour |
| **Total** | **~6-7 hours** |
