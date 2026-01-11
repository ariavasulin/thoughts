---
date: 2026-01-10T13:35:19+07:00
researcher: Ariav Asulin
git_commit: b30f58c47d3e40dab378d14038bab8be9deb5cf4
branch: main
repository: YouLab
topic: "Documentation Verification - File Sync Service"
tags: [research, documentation, file-sync, verification]
status: complete
last_updated: 2026-01-10
last_updated_by: Ariav Asulin
---

# Research: Documentation Verification - File Sync Service

**Date**: 2026-01-10T13:35:19+07:00
**Researcher**: Ariav Asulin
**Git Commit**: b30f58c47d3e40dab378d14038bab8be9deb5cf4
**Branch**: main
**Repository**: YouLab

## Research Question

Verify that all documentation in docs/ accurately reflects the current codebase implementation, particularly focusing on the new file sync service. Check for any outdated information, missing features, or inaccuracies.

## Summary

The file sync service (commit b30f58c) is **completely undocumented** in the docs/ directory. This is the most significant gap found. Additionally, several related configuration options and schema fields are missing from documentation.

**Total Discrepancies Found: 7**

| Severity | Count | Description |
|----------|-------|-------------|
| Critical | 1 | HTTP-Service.md missing entire /sync/* section |
| High | 3 | Settings, Configuration, Architecture missing sync details |
| Medium | 2 | config-schema.md missing folders and task.name fields |
| Low | 1 | Roadmap.md not updated to reflect file sync feature |

## Detailed Findings

### 1. HTTP-Service.md - Missing File Sync Section (Critical)

**File**: `docs/HTTP-Service.md`

The entire file sync endpoint group is missing. The implemented endpoints are:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/sync/status` | GET | Get sync service status |
| `/sync/trigger` | POST | Manually trigger sync cycle |
| `/sync/mappings` | GET | List all sync mappings |
| `/sync/attach/{agent_id}/{folder_name}` | POST | Attach folder to agent |

**Implementation**: `src/letta_starter/server/sync/router.py:10-133`

**Router registration**: `src/letta_starter/server/main.py:143`

---

### 2. Settings.md - Missing File Sync Settings (High)

**File**: `docs/Settings.md`

The `ServiceSettings` class documentation is missing these fields:

```python
# File sync configuration (OpenWebUI → Letta)
file_sync_enabled: bool = False
openwebui_url: str = "http://localhost:3000"
openwebui_api_key: str | None = None
file_sync_interval: int = 30
file_sync_embedding_model: str = "openai/text-embedding-3-small"
data_dir: str = ".data"
```

**Implementation**: `src/letta_starter/config/settings.py:173-197`

---

### 3. Configuration.md - Missing File Sync Environment Variables (High)

**File**: `docs/Configuration.md`

The environment variables table is missing a "File Sync" section:

| Variable | Default | Description |
|----------|---------|-------------|
| `YOULAB_SERVICE_FILE_SYNC_ENABLED` | `false` | Enable file sync from OpenWebUI to Letta |
| `YOULAB_SERVICE_OPENWEBUI_URL` | `http://localhost:3000` | OpenWebUI base URL |
| `YOULAB_SERVICE_OPENWEBUI_API_KEY` | `null` | OpenWebUI API key for sync |
| `YOULAB_SERVICE_FILE_SYNC_INTERVAL` | `30` | Seconds between sync cycles |
| `YOULAB_SERVICE_FILE_SYNC_EMBEDDING_MODEL` | `openai/text-embedding-3-small` | Embedding model for Letta folders |
| `YOULAB_SERVICE_DATA_DIR` | `.data` | Directory for persistent data |

---

### 4. Architecture.md - Missing sync/ Directory (High)

**File**: `docs/Architecture.md`

The project structure diagram is missing the `sync/` subdirectory:

```
├── server/              # HTTP service
│   ├── main.py          # FastAPI app
│   ├── agents.py        # AgentManager
│   ├── sync/            # <-- MISSING
│   │   ├── __init__.py
│   │   ├── service.py   # FileSyncService
│   │   ├── router.py    # FastAPI router
│   │   ├── mappings.py  # Sync mapping storage
│   │   └── openwebui_client.py  # OpenWebUI API client
```

The HTTP Service table also needs a Sync row:

| Domain | Endpoints | Manager |
|--------|-----------|---------|
| Sync | `/sync/*` | FileSyncService |

---

### 5. config-schema.md - Missing Folder Configuration (Medium)

**File**: `docs/config-schema.md`

The `[agent]` table is missing the `folders` field documentation:

```toml
[agent.folders]
shared = ["college-essay-materials"]  # Shared folder names to attach
private = true  # Whether to attach user's private folder
```

**Schema**: `src/letta_starter/curriculum/schema.py:81-96` (`AgentFoldersConfig`)

**Example usage**: `config/courses/college-essay/course.toml:77-80`

The `[agent]` field reference table needs:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| folders | AgentFoldersConfig | no | {} | Folder configuration for file access |

And a new section:

```markdown
### [agent.folders] - Folder Configuration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| shared | list[string] | [] | Shared folder names to attach to agent |
| private | bool | true | Whether to attach user's private folder |
```

---

### 6. config-schema.md - Missing task.name Field (Medium)

**File**: `docs/config-schema.md`

The `[[task]]` table is missing the `name` field:

```toml
[[task]]
name = "progression-grader"  # <-- Not documented
```

**Schema**: The `TaskConfig` in `schema.py` doesn't have `name` field, but it's used in the TOML config - this is a schema/implementation mismatch.

**Action needed**: Either add `name` to `TaskConfig` schema, or remove it from the example TOML.

---

### 7. Roadmap.md - File Sync Not Mentioned (Low)

**File**: `docs/Roadmap.md`

The file sync service is a completed feature but not mentioned in the roadmap. It could be:
- Added as an incremental feature under "What's NOT Included" that was later added
- Mentioned in the Phase 1 or Phase 3 deliverables as an enhancement
- Listed in a new section for post-Phase 6 features

---

## Code References

- `src/letta_starter/server/sync/__init__.py` - Module exports
- `src/letta_starter/server/sync/service.py:1-394` - FileSyncService implementation
- `src/letta_starter/server/sync/router.py:1-133` - HTTP endpoints
- `src/letta_starter/server/sync/mappings.py:1-133` - Sync mapping storage
- `src/letta_starter/server/sync/openwebui_client.py:1-190` - OpenWebUI API client
- `src/letta_starter/config/settings.py:173-197` - ServiceSettings sync fields
- `src/letta_starter/curriculum/schema.py:81-96` - AgentFoldersConfig
- `config/courses/college-essay/course.toml:77-80` - Folder config example

## Architecture Documentation

The file sync service implements one-way sync from OpenWebUI to Letta:

```
OpenWebUI Notes & Knowledge
         │
         ▼
   FileSyncService
   (periodic sync)
         │
         ├──► Letta Folders (shared_notes, user_X_notes)
         │
         └──► Mapping Storage (.data/sync_mappings.json)
```

**Key features**:
- Content-hash based change detection
- Automatic PDF→MD conversion via Letta's Mistral OCR
- Access control mapping (shared vs user-specific folders)
- Background sync loop with configurable interval
- Manual trigger endpoint

## Related Research

- `thoughts/shared/research/2026-01-09-openwebui-notes-letta-sync.md` - Original file sync research
- `thoughts/shared/research/2026-01-09-toml-openwebui-two-way-sync.md` - Two-way sync considerations
- `thoughts/shared/research/2026-01-09-documentation-verification.md` - Previous doc verification

## Open Questions

1. Should file sync be documented as a standalone page (e.g., `docs/File-Sync.md`) or integrated into HTTP-Service.md?
2. Is the `name` field in `[[task]]` intentional? It's not in the Pydantic schema.
3. Should the Roadmap be updated to reflect file sync as a completed feature?
