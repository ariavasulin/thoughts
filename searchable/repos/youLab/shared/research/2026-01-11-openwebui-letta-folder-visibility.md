---
date: 2026-01-11T17:38:00+07:00
researcher: claude
git_commit: 166d668bbe5b80daaf2e2e1e00753dcc4f100e4c
branch: module1-local-poc
repository: YouLab
topic: "OpenWebUI → Letta Folder Sync: Visibility Gap Analysis"
tags: [research, codebase, openwebui, letta, folders, sync, knowledge]
status: complete
last_updated: 2026-01-11
last_updated_by: claude
---

# Research: OpenWebUI → Letta Folder Visibility

**Date**: 2026-01-11T17:38:00+07:00
**Researcher**: claude
**Git Commit**: 166d668bbe5b80daaf2e2e1e00753dcc4f100e4c
**Branch**: module1-local-poc
**Repository**: YouLab

## Research Question

How can we get folders to sync with files, folders, and notes in OpenWebUI? The user wants to see notes, files, and folders collections in the OpenWebUI UI.

## Summary

The current implementation provides **one-way sync** from OpenWebUI → Letta. Files uploaded to OpenWebUI (Notes, Knowledge collections) are synced to Letta folders where agents can access them. However, **Letta folders are NOT visible in OpenWebUI's UI** because they are separate storage systems.

**Key Finding**: There is currently no mechanism to display Letta folders back in OpenWebUI. The sync is unidirectional.

## Detailed Findings

### Current Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OpenWebUI UI                                  │
│  • Notes (/workspace/notes) → Editable markdown documents           │
│  • Knowledge (/workspace/knowledge) → File collections              │
│  • Files → Uploaded documents (PDFs, etc.)                          │
│  ✗ Folders → NOT a native OpenWebUI concept                        │
└──────────────────────────────────────────────────────────────────────┘
           │ REST API (one-way)
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      FileSyncService                                  │
│  • Polls OpenWebUI every 30 seconds                                  │
│  • Downloads Notes → uploads to Letta as .md files                  │
│  • Downloads Knowledge files → uploads to Letta (PDF→MD via OCR)    │
│  • Creates Letta folders based on access_control                    │
│  • Tracks mappings in .data/sync_mappings.json                      │
└──────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Letta Server                                     │
│  • Folders → Agent-accessible file storage                          │
│  • Built-in tools: open_files, grep_files, semantic_search_files    │
│  • PDF→MD conversion via Mistral OCR                                │
│  • RAG via archival memory                                          │
│  ✗ NOT visible in OpenWebUI                                         │
└──────────────────────────────────────────────────────────────────────┘
```

### OpenWebUI API Endpoints Used

| Endpoint | Method | Purpose | Implementation |
|----------|--------|---------|----------------|
| `/api/v1/notes/` | GET | List all notes | `openwebui_client.py:86` |
| `/api/v1/notes/{id}` | GET | Get single note | `openwebui_client.py:112` |
| `/api/v1/knowledge/` | GET | List knowledge collections | `openwebui_client.py:133` |
| `/api/v1/files/{id}/content` | GET | Download file bytes | `openwebui_client.py:174` |

### OpenWebUI API Endpoints NOT Used

Based on [OpenWebUI API documentation](https://docs.openwebui.com/getting-started/api-endpoints/), the following endpoints exist but are **not utilized**:

| Endpoint | Purpose | Notes |
|----------|---------|-------|
| `/api/v1/files/` | List all files | Not used - files accessed via knowledge collections |
| `/api/v1/knowledge/{id}/file/add` | Add file to knowledge | Not used - one-way sync only |

**Note**: OpenWebUI does NOT have a native "folders" API endpoint. The `/workspace/knowledge` UI shows Knowledge collections, not folders.

### Key Implementation Files

| File | Purpose |
|------|---------|
| `src/youlab_server/server/sync/openwebui_client.py` | HTTP client for OpenWebUI API |
| `src/youlab_server/server/sync/service.py` | FileSyncService with background loop |
| `src/youlab_server/server/sync/mappings.py` | Persistent sync state tracking |
| `src/youlab_server/server/sync/router.py` | REST endpoints for sync management |
| `src/youlab_server/server/agents.py:118-155` | Agent folder attachment during creation |

### How Data Flows Today

1. **User uploads in OpenWebUI**:
   - MD files → Create a Note (`/workspace/notes`)
   - PDF files → Add to Knowledge collection (`/workspace/knowledge`)

2. **FileSyncService polls** (every 30 seconds):
   - Fetches all Notes via `GET /api/v1/notes/`
   - Fetches all Knowledge via `GET /api/v1/knowledge/`
   - Downloads file content via `GET /api/v1/files/{id}/content`

3. **Files uploaded to Letta**:
   - Notes → `shared_notes` or `user_{id}_notes` folder
   - Knowledge files → `shared_knowledge_{id}` or `user_{id}_knowledge` folder
   - PDFs converted to MD via Letta's Mistral OCR

4. **Agent accesses files**:
   - Folders attached during agent creation (`agents.py:349-354`)
   - Agent uses built-in tools: `open_files`, `grep_files`, `semantic_search_files`

### Why Letta Folders Are NOT Visible in OpenWebUI

**OpenWebUI and Letta have separate storage systems**:

| Concept | OpenWebUI | Letta |
|---------|-----------|-------|
| Notes | `/api/v1/notes/` → Database | Not used |
| Knowledge | `/api/v1/knowledge/` → Database + vector store | Not used |
| Files | `/api/v1/files/` → Blob storage | Folders → FileMetadata table + embeddings |
| Folders | **Does not exist** | `letta.folders.*` API |

The FileSyncService creates Letta folders, but there is no reverse sync to create corresponding OpenWebUI resources.

### OpenWebUI Concepts Explained

1. **Notes** (`/workspace/notes`):
   - Editable markdown documents
   - Stored in `note` table with `data.content.md` for markdown
   - Access control via `access_control` field

2. **Knowledge** (`/workspace/knowledge`):
   - Collections of files (PDFs, etc.)
   - Used for RAG within OpenWebUI's native vector store
   - Each collection has `name`, `description`, `files[]`

3. **Files** (standalone):
   - Uploaded files not in a knowledge collection
   - Can be attached to chats
   - Stored in blob storage

### Options to Make Folders Visible

#### Option A: Reverse Sync (Letta → OpenWebUI Knowledge)

Create OpenWebUI Knowledge collections that mirror Letta folders.

**Implementation**:
- When Letta folder is created/updated, call `POST /api/v1/knowledge/`
- When files are added, call `POST /api/v1/knowledge/{id}/file/add`

**Challenges**:
- Requires OpenWebUI API write access (not currently implemented)
- Could create duplicate storage (files in both systems)
- Sync conflicts when both systems modify

#### Option B: OpenWebUI Extension (Pipe)

Create a custom Pipe that exposes Letta folders in the OpenWebUI UI.

**Implementation**:
- Pipe queries Letta API for folders and files
- Returns formatted response for OpenWebUI to display

**Challenges**:
- Pipes are for chat completions, not UI extensions
- Would need custom frontend components

#### Option C: Direct OpenWebUI API Integration

Use OpenWebUI's Knowledge collections as the source of truth.

**Implementation**:
- Agents query OpenWebUI Knowledge API directly (instead of Letta folders)
- No sync needed - single source of truth

**Challenges**:
- Requires custom tools to query OpenWebUI API
- Loses Letta's built-in RAG capabilities
- Would need to re-implement search functionality

#### Option D: Accept Current Behavior

Keep the current one-way sync and accept that:
- Users upload in OpenWebUI
- Files sync to Letta
- Agents access via Letta tools
- Letta folders are "invisible" to users (behind the scenes)

**Advantages**:
- Simplest to maintain
- Letta handles all RAG complexity
- Clear separation of concerns

## Code References

- `src/youlab_server/server/sync/openwebui_client.py:86` - `list_notes()` API call
- `src/youlab_server/server/sync/openwebui_client.py:133` - `list_knowledge()` API call
- `src/youlab_server/server/sync/service.py:294-326` - `ensure_folder()` creates Letta folders
- `src/youlab_server/server/sync/service.py:259-275` - `_get_folder_name_for_note()` determines folder naming
- `src/youlab_server/server/agents.py:118-155` - `_attach_folders()` attaches folders to agents
- `src/youlab_server/curriculum/loader.py:207-224` - Parses folder config from TOML

## Architecture Documentation

### Current Folder Naming Convention

| OpenWebUI Resource | Letta Folder Name |
|-------------------|-------------------|
| Public Note (`access_control=None`) | `shared_notes` |
| Private Note (`access_control={}`) | `user_{user_id}_notes` |
| Public Knowledge | `shared_knowledge_{collection_id[:8]}` |
| Private Knowledge | `user_{user_id}_knowledge` |

### Course TOML Configuration

```toml
[agent.folders]
shared = ["college-essay-materials"]  # Shared folders for all users
private = true                         # User gets user_{id}_notes folder
```

### Sync Mapping Storage

Location: `.data/sync_mappings.json`

```json
{
  "notes": {
    "note-abc123": {
      "openwebui_note_id": "note-abc123",
      "letta_folder_id": "folder-xyz",
      "letta_file_id": "file-123",
      "title": "My Essay Draft",
      "content_hash": "a1b2c3d4e5f6",
      "last_synced": "2026-01-11T10:00:00",
      "status": "synced"
    }
  },
  "files": { ... }
}
```

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-10-file-knowledge-sync.md` - Original implementation plan
- `thoughts/shared/plans/2026-01-09-openwebui-knowledge-sync.md` - Knowledge sync design
- `thoughts/shared/research/2026-01-09-openwebui-notes-letta-sync.md` - Notes sync research
- `thoughts/shared/handoffs/general/2026-01-11_17-30-00_folder-attachment-fix.md` - Recent folder attachment bug fix

## Related Research

- [OpenWebUI API Documentation](https://docs.openwebui.com/getting-started/api-endpoints/)
- [OpenWebUI GitHub Discussion #16402](https://github.com/open-webui/open-webui/discussions/16402) - API Reference

## Open Questions

1. **Does OpenWebUI have a folders API?** - Based on documentation search, there is no dedicated folders endpoint. Knowledge collections are the closest concept.

2. **Can OpenWebUI display external resources?** - Would require custom UI extension or Pipe, neither of which currently supports folder display.

3. **What is the desired UX?** - Clarification needed:
   - Should users see a "Folders" section in OpenWebUI?
   - Should Letta folders appear as Knowledge collections?
   - Should the sync be bidirectional?

4. **Is there an OpenWebUI extension system?** - Beyond Pipes (for chat), OpenWebUI may support custom UI extensions. Requires further research.
