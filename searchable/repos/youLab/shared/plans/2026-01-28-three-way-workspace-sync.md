# Three-Way Workspace Sync System

**Date**: 2026-01-28
**Status**: Draft
**Author**: Claude (Planning Session)

## Overview

Design a lightweight three-way sync system that synchronizes:
- **A) Ralph Server Workspace**: Per-user directories at `RALPH_USER_DATA_DIR/{user_id}/workspace`
- **B) OpenWebUI Knowledge Base**: Per-user knowledge base titled "workspace" for RAG queries
- **C) Local Folder (Optional)**: User's local machine folder via lightweight daemon

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Local Folder   │◄───────►│  Ralph Server   │◄───────►│ OpenWebUI KB    │
│  (User Machine) │   Sync  │   Workspace     │   Sync  │  "workspace"    │
└─────────────────┘   Daemon└─────────────────┘  Service└─────────────────┘
        ▲                          ▲                          ▲
        │                          │                          │
        └──────────────────────────┴──────────────────────────┘
                           File Hashing + Last-Modified
                           for Conflict Detection
```

## Design Principles

1. **Hub-and-Spoke Architecture**: Ralph workspace is the authoritative source
2. **Content-Hash Based Change Detection**: SHA256 for reliable diffing (same as openwebui-content-sync)
3. **Conflict Resolution**: Last-writer-wins with optional conflict markers
4. **Incremental Sync**: Only transfer changed files
5. **Minimal Dependencies**: Local daemon is a single static binary

## Component Details

### A. Ralph Workspace (Hub)

**Current State**:
- Location: `{RALPH_USER_DATA_DIR}/{user_id}/workspace/`
- Created on first chat: `workspace.mkdir(parents=True, exist_ok=True)`
- Used by: FileTools, ShellTools (Agno tools)
- No current sync mechanism

**Required Additions**:
1. File index storage (JSON, same format as openwebui-content-sync)
2. HTTP API endpoints for sync operations
3. Knowledge base ID mapping per user

### B. OpenWebUI Knowledge Base (Spoke 1)

**OpenWebUI API Used**:
```
POST   /api/v1/files/                    # Upload file
GET    /api/v1/files/{id}                # Get file metadata
GET    /api/v1/files/{id}/content        # Download file content
DELETE /api/v1/files/{id}                # Delete file
GET    /api/v1/knowledge/                # List knowledge bases
POST   /api/v1/knowledge/create          # Create knowledge base
POST   /api/v1/knowledge/{id}/file/add   # Add file to KB
POST   /api/v1/knowledge/{id}/file/remove # Remove file from KB
```

**Per-User Knowledge Base**:
- Name convention: `workspace-{user_id}` or user-configurable
- Created on first sync if not exists
- Bidirectional: Ralph ↔ OpenWebUI

### C. Local Daemon (Spoke 2)

**Design Goals**:
- Single static binary (no runtime dependencies)
- Cross-platform: macOS, Windows, Linux
- Minimal footprint: < 10MB binary, < 20MB RAM
- File watching with debouncing
- Automatic reconnection

**Technology Choice**: Go (matches openwebui-content-sync)
- Static compilation → single binary distribution
- Excellent cross-platform support
- Built-in file watching (fsnotify)
- Low memory footprint

## Architecture

### Sync State Model

```json
{
  "version": 1,
  "user_id": "7a41011b-5255-4225-b75e-1d8484d0e37f",
  "knowledge_id": "kb-workspace-7a41011b",
  "last_sync": "2026-01-28T12:00:00Z",
  "files": {
    "notes/todo.md": {
      "path": "notes/todo.md",
      "hash": "sha256:abc123...",
      "size": 1234,
      "modified": "2026-01-28T11:00:00Z",
      "source": "ralph",
      "openwebui_file_id": "file-xyz789",
      "synced_at": "2026-01-28T12:00:00Z"
    }
  }
}
```

### Sync Algorithm

```python
def three_way_sync(ralph_state, openwebui_state, local_state):
    """
    Merge algorithm for three-way sync.
    Ralph workspace is authoritative when conflicts occur.
    """
    merged = {}
    all_paths = set(ralph_state.keys()) | set(openwebui_state.keys()) | set(local_state.keys())

    for path in all_paths:
        ralph = ralph_state.get(path)
        owui = openwebui_state.get(path)
        local = local_state.get(path)

        # Get most recent version
        candidates = [
            (ralph, 'ralph') if ralph else None,
            (owui, 'openwebui') if owui else None,
            (local, 'local') if local else None,
        ]
        candidates = [c for c in candidates if c]

        if not candidates:
            continue  # Deleted everywhere

        # Sort by modified time, prefer ralph on tie
        candidates.sort(key=lambda x: (x[0]['modified'], x[1] == 'ralph'), reverse=True)
        winner = candidates[0]

        # Check for conflicts (different hashes, similar times)
        hashes = set(c[0]['hash'] for c in candidates)
        if len(hashes) > 1:
            # Different content exists
            merged[path] = {
                **winner[0],
                'conflict_from': [c[1] for c in candidates if c[0]['hash'] != winner[0]['hash']],
            }
        else:
            merged[path] = winner[0]

    return merged
```

### API Endpoints (Ralph Server)

Add to `src/ralph/server.py`:

```python
# New endpoints for workspace sync

@router.get("/users/{user_id}/workspace/files")
async def list_workspace_files(user_id: str) -> WorkspaceIndex:
    """List all files in workspace with hashes."""
    pass

@router.get("/users/{user_id}/workspace/files/{path:path}")
async def get_workspace_file(user_id: str, path: str) -> FileContent:
    """Download file content."""
    pass

@router.put("/users/{user_id}/workspace/files/{path:path}")
async def put_workspace_file(user_id: str, path: str, content: bytes) -> FileMetadata:
    """Upload/update file."""
    pass

@router.delete("/users/{user_id}/workspace/files/{path:path}")
async def delete_workspace_file(user_id: str, path: str):
    """Delete file."""
    pass

@router.post("/users/{user_id}/workspace/sync")
async def trigger_sync(user_id: str, direction: Literal["to_openwebui", "from_openwebui", "bidirectional"]):
    """Manually trigger sync with OpenWebUI."""
    pass
```

## Implementation Plan

### Phase 1: Ralph ↔ OpenWebUI Sync (Server-Side)

**Goal**: Files in Ralph workspace sync to per-user OpenWebUI knowledge base

**Tasks**:

1. **Create workspace sync service** (`src/ralph/sync/workspace_sync.py`)
   - [x] Port openwebui-content-sync's local adapter logic to Python
   - [x] Implement file indexing (hash calculation, modification tracking)
   - [x] Create OpenWebUI client (reuse patterns from existing Go code)

2. **Add HTTP API for sync** (`src/ralph/api/workspace.py`)
   - [x] `GET /users/{user_id}/workspace/files` - List files with metadata
   - [x] `GET /users/{user_id}/workspace/files/{path}` - Download file
   - [x] `PUT /users/{user_id}/workspace/files/{path}` - Upload file
   - [x] `DELETE /users/{user_id}/workspace/files/{path}` - Delete file
   - [x] `POST /users/{user_id}/workspace/sync` - Trigger sync

3. **Per-user knowledge base management** (`src/ralph/sync/knowledge.py`)
   - [x] Create KB on first user chat (name: `workspace-{user_id}`)
   - [ ] Store KB mapping in Dolt (new table: `user_knowledge_bases`) - Deferred, using sync state file instead
   - [x] Access control: user can only access their own KB

4. **Background sync scheduler**
   - [ ] Use existing background task infrastructure
   - [ ] Sync on interval (configurable, default 5 minutes)
   - [ ] Sync on file change (via FileTools hook)

**Estimated Effort**: 3-4 days

### Phase 2: Local Daemon (Client-Side)

**Goal**: Lightweight daemon syncs local folder to Ralph workspace

**Tasks**:

1. **Create Go daemon** (`tools/youlab-sync/`)
   - [x] Fork/adapt openwebui-content-sync structure
   - [x] Replace adapters with single "ralph" adapter
   - [x] Implement file watching (fsnotify)
   - [x] Add bidirectional sync support

2. **Configuration**
   ```yaml
   # ~/.youlab-sync/config.yaml
   server:
     url: "https://theyoulab.org"
     api_key: "user-api-key"
   sync:
     local_folder: "/Users/student/YouLab"
     interval: "30s"
     ignore_patterns:
       - ".git"
       - "node_modules"
       - "*.tmp"
   ```

3. **Distribution**
   - [x] Cross-compile for macOS (arm64, amd64), Windows, Linux
   - [ ] Create installer scripts
   - [x] Document setup process

4. **Features**
   - [ ] Tray icon (optional, macOS/Windows)
   - [x] Automatic reconnection
   - [x] Conflict notification

**Estimated Effort**: 5-7 days

### Phase 3: Polish & Integration

1. **OpenWebUI UI Integration**
   - [ ] Add "Sync Status" indicator to workspace KB
   - [ ] Show sync conflicts in UI

2. **Ralph Agent Awareness**
   - [ ] Inform agent when files sync from local
   - [ ] Agent can trigger sync explicitly

3. **Monitoring**
   - [ ] Sync metrics (files synced, errors, latency)
   - [ ] Langfuse integration

**Estimated Effort**: 2-3 days

## File Structure

```
src/ralph/
  sync/
    __init__.py
    workspace_sync.py      # Core sync logic
    knowledge.py           # OpenWebUI KB management
    openwebui_client.py    # OpenWebUI API client
    models.py              # Pydantic models for sync state
  api/
    workspace.py           # HTTP endpoints for sync

tools/
  youlab-sync/             # Local daemon (Go)
    main.go
    cmd/
      root.go
      sync.go
      watch.go
    internal/
      config/
      ralph/               # Ralph API client
      watcher/             # File watching
      sync/                # Sync logic
    go.mod
    go.sum
    Makefile               # Cross-compilation
```

## Configuration

### Environment Variables (Ralph Server)

```bash
# OpenWebUI integration
RALPH_OPENWEBUI_URL=https://openwebui.theyoulab.org
RALPH_OPENWEBUI_API_KEY=sk-...

# Sync settings
RALPH_SYNC_INTERVAL=300              # seconds (default: 5 min)
RALPH_SYNC_TO_OPENWEBUI=true         # Enable KB sync
RALPH_SYNC_KNOWLEDGE_PREFIX=workspace  # KB naming prefix
```

### Local Daemon Config

```yaml
# ~/.youlab-sync/config.yaml
server:
  url: "https://theyoulab.org"
  api_key: "${YOULAB_API_KEY}"  # or direct string

sync:
  local_folder: "/Users/student/YouLab"
  interval: "30s"
  bidirectional: true

watch:
  enabled: true
  debounce: "500ms"

ignore:
  - ".git"
  - ".DS_Store"
  - "*.tmp"
  - "node_modules"
  - "__pycache__"
```

## Security Considerations

1. **Authentication**
   - Local daemon uses API key (stored securely in OS keychain)
   - Ralph validates user_id from API key claims

2. **Path Traversal**
   - Validate all paths stay within workspace
   - Reject `..` components

3. **File Size Limits**
   - Max file size: 10MB per file (configurable)
   - Max workspace size: 100MB (configurable)

4. **Rate Limiting**
   - Sync API: 60 requests/minute per user
   - Debounce file watching to prevent storms

## Open Questions

1. **Conflict UI**: Should we show conflicts in OpenWebUI chat or create a separate UI?
2. **Selective Sync**: Should users be able to exclude certain files/folders?
3. **Version History**: Keep previous versions? (Dolt could help here)
4. **Offline Mode**: Local daemon queue changes when server unreachable?

## Success Criteria

- [ ] Files created by Ralph agent appear in user's local folder within 60s
- [ ] Files edited locally appear in Ralph workspace within 60s
- [ ] OpenWebUI knowledge base stays in sync for RAG queries
- [ ] Local daemon binary < 15MB, memory usage < 30MB
- [ ] Zero data loss during normal operation
- [ ] Graceful conflict handling

## Dependencies

### Phase 1 (Python)
- `httpx` - HTTP client for OpenWebUI API
- `aiofiles` - Async file operations
- `watchfiles` - File watching (optional, for future real-time)

### Phase 2 (Go)
- `fsnotify/fsnotify` - File system notifications
- `sirupsen/logrus` - Logging
- `spf13/cobra` - CLI framework
- `spf13/viper` - Configuration

## References

- [openwebui-content-sync](../../../openwebui-content-sync/) - Existing Go sync tool
- [OpenWebUI Knowledge API](../research/2026-01-28-openwebui-content-sync-integration-feasibility.md)
- [Ralph Workspace Code](../../../src/ralph/server.py:46-56)
