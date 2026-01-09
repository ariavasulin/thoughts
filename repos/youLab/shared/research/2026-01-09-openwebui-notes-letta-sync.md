---
date: 2026-01-09T23:07:15+07:00
researcher: ariasulin
git_commit: 47ca115279c37e41f85e0dd3d7765dc2a1f7156d
branch: main
repository: YouLab
topic: "Syncing OpenWebUI Notes to Letta with Reference Preservation"
tags: [research, codebase, openwebui, letta, notes, knowledge, sync, rag]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: Syncing OpenWebUI Notes to Letta with Reference Preservation

**Date**: 2026-01-09T23:07:15+07:00
**Researcher**: ariasulin
**Git Commit**: 47ca115279c37e41f85e0dd3d7765dc2a1f7156d
**Branch**: main
**Repository**: YouLab

## Research Question

What would it take to sync OpenWebUI Notes to Letta with read-only access, while keeping note references in OpenWebUI working when the chat goes through Letta?

## Summary

**Key Finding**: OpenWebUI's `#` note referencing **already works with Letta** because context injection happens in OpenWebUI's middleware *before* messages reach Letta. No sync is required for basic reference functionality.

However, syncing notes to Letta would give the agent **independent access** to notes via file tools (`search_file`, `open_file`, `grep_file`) without requiring explicit user references. This enables proactive note retrieval and agent-driven knowledge access.

**Notes vs Knowledge**: The existing plan (`2026-01-09-openwebui-knowledge-sync.md`) focuses on **Knowledge** (vector RAG collections). Notes are different - they're simpler markdown documents without vector embeddings. A separate sync approach is needed.

## Critical Distinction: Notes vs Knowledge

| Feature | Notes | Knowledge |
|---------|-------|-----------|
| **Storage** | `note.data.content.md` (markdown) | Vector DB collections |
| **Embedding** | None | Chunked & embedded |
| **Retrieval** | Direct database lookup | Semantic vector search |
| **Files** | No file attachments | Multi-file collections |
| **RAG** | Content injected as-is | Top-k chunk retrieval |
| **API** | `/api/v1/notes/*` | `/api/v1/knowledge/*` |

**Important**: The existing sync plan targets Knowledge, not Notes. To sync Notes, a different approach is needed.

## How Note Referencing Currently Works

### Data Flow (OpenWebUI → Letta)

```
1. User types # in chat
   └── MessageInput.svelte triggers TipTap suggestion

2. Autocomplete shows notes
   └── Commands/Knowledge.svelte searches notes API

3. User selects note
   └── Added to files[] array: {id, name, type: "note"}

4. User sends message
   └── files[] sent in request body

5. OpenWebUI middleware processes
   └── chat_completion_files_handler() at middleware.py:950

6. Note content retrieved (NO vector search)
   └── retrieval/utils.py:982-995:
       note = Notes.get_note_by_id(item.get("id"))
       content = note.data.get("content", {}).get("md", "")

7. Content injected into message
   └── RAG template wraps content in <source> tags

8. Modified message sent to Letta
   └── Letta sees user message WITH note context already embedded
```

**Key Insight**: Letta receives the message *after* OpenWebUI injects note content. The reference resolution happens in OpenWebUI middleware, not in Letta.

### Where Reference Resolution Happens

**File**: `OpenWebUI/open-webui/backend/open_webui/retrieval/utils.py:982-995`

```python
elif item.get("type") == "note":
    note = Notes.get_note_by_id(item.get("id"))
    if note and has_access(user.id, "read", note.access_control):
        query_result = {
            "documents": [[note.data.get("content", {}).get("md", "")]],
            "metadatas": [[{"file_id": note.id, "name": note.title}]],
        }
```

Notes are retrieved directly from the database - no vector search involved. The full markdown content is injected.

## What Syncing Notes to Letta Would Enable

### Without Sync (Current Behavior)
- User must explicitly reference notes with `#` in OpenWebUI
- Letta agent cannot proactively access notes
- Agent has no persistent knowledge of user's notes
- Works today with no changes needed

### With Sync (New Capability)
- Agent can search notes proactively via `search_file` tool
- Agent can browse notes via `open_file` tool
- Agent maintains awareness of note contents between sessions
- Enables agent-driven knowledge retrieval without explicit references

## Implementation Options

### Option A: Notes Sync Service (Parallel to Knowledge Sync)

Add a separate sync service for Notes, similar to the Knowledge sync plan but simpler:

```
src/letta_starter/server/sync/
├── knowledge.py    # Existing Knowledge sync (from plan)
└── notes.py        # New Notes sync
```

**Notes Sync Flow:**

```
1. Poll OpenWebUI /api/v1/notes/ (paginated)
2. For each note with changes since last sync:
   a. Get note content: GET /api/v1/notes/{id}
   b. Export markdown: note.data.content.md
   c. Create/update file in Letta folder:
      - Folder: openwebui_notes_{user_id}
      - File: {note_title}.md
3. Track mappings: openwebui_note_id ↔ letta_file_id
4. Attach folder to user's agent
```

**Advantages:**
- Agent has independent access to notes
- Enables proactive note retrieval
- Notes become part of agent's persistent context

**Challenges:**
- Must track note updates (updated_at field)
- Must handle note deletions
- Naming conflicts (notes can have same title)
- Real-time sync difficult (notes use collaborative editing)

### Option B: On-Demand Note Loading (No Persistent Sync)

Instead of syncing all notes, load them into Letta's archival memory when referenced:

```python
# In LettaStarter's message handler
if note_references := extract_note_refs(message):
    for note_id in note_references:
        note = fetch_note_from_openwebui(note_id)
        client.insert_archival_memory(
            agent_id=agent_id,
            memory=f"[NOTE: {note.title}]\n{note.content}"
        )
```

**Advantages:**
- No background sync needed
- Only referenced notes loaded
- Simpler implementation

**Disadvantages:**
- Agent can't proactively search notes
- Duplicates OpenWebUI's reference resolution
- Archival memory not as searchable as folders

### Option C: Hybrid Approach (Recommended)

Keep OpenWebUI's reference system for UI-driven access, add Letta folder sync for agent-driven access:

```
┌─────────────────────────────────────────────────────────────────┐
│                     User References Note (#)                      │
│                              ↓                                    │
│              OpenWebUI middleware injects content                 │
│                              ↓                                    │
│              Message with context reaches Letta                   │
└─────────────────────────────────────────────────────────────────┘
                              AND
┌─────────────────────────────────────────────────────────────────┐
│                     Notes Sync Service                           │
│                              ↓                                    │
│              Notes exported to Letta folder                       │
│                              ↓                                    │
│              Agent can search_file/open_file proactively          │
└─────────────────────────────────────────────────────────────────┘
```

**Both paths work independently:**
1. `#` references work via OpenWebUI's middleware (existing)
2. Agent can search notes via file tools (new)

## Notes Sync Service Design

If implementing Option A or C, here's the Notes-specific design:

### API Endpoints Needed

**From OpenWebUI:**
- `GET /api/v1/notes/` - List notes (paginated)
- `GET /api/v1/notes/{id}` - Get note with content

**To Letta:**
- `client.folders.create(name, embedding)` - Create notes folder
- `client.folders.files.upload(folder_id, file)` - Upload note as file
- `client.agents.folders.attach(agent_id, folder_id)` - Attach to agent

### Mapping Table

```python
@dataclass
class NoteSyncMapping:
    openwebui_note_id: str
    letta_folder_id: str
    letta_file_id: str | None
    note_title: str
    content_hash: str  # For change detection
    last_synced: datetime
    status: str  # synced, pending, error
```

### Change Detection

Notes don't have vector embeddings, so we need different change detection:

```python
def note_changed(note: Note, mapping: NoteSyncMapping) -> bool:
    current_hash = hashlib.md5(
        note.data.get("content", {}).get("md", "").encode()
    ).hexdigest()
    return current_hash != mapping.content_hash
```

### File Naming

```python
def note_filename(note: Note) -> str:
    # Sanitize title for filesystem
    safe_title = re.sub(r'[^\w\s-]', '', note.title)[:50]
    return f"{safe_title}_{note.id[:8]}.md"
```

### Folder Structure

```
Letta Folders:
├── openwebui_notes_{user_id}/
│   ├── Meeting_Notes_abc12345.md
│   ├── Project_Ideas_def67890.md
│   └── ...
└── openwebui_knowledge_{knowledge_id}/  # From Knowledge sync
    └── ...
```

## Letta Folders API Reference

### Creating a Notes Folder

```python
folder = client.folders.create(
    name=f"openwebui_notes_{user_id}",
    embedding="openai/text-embedding-3-small"  # Must match agent
)
```

### Uploading a Note as File

```python
import io

note_content = note.data.get("content", {}).get("md", "")
file_obj = io.BytesIO(note_content.encode("utf-8"))
file_obj.name = note_filename(note)

job = client.folders.files.upload(
    folder_id=folder.id,
    file=file_obj
)

# Wait for processing
while True:
    job = client.jobs.retrieve(job.id)
    if job.status == "completed":
        break
    time.sleep(1)
```

### Attaching to Agent

```python
client.agents.folders.attach(
    agent_id=agent.id,
    folder_id=folder.id
)
# Agent now has: open_file, grep_file, search_file tools
```

## Code References

### OpenWebUI Notes

| File | Line | Description |
|------|------|-------------|
| `backend/open_webui/models/notes.py` | 26-40 | Note database schema |
| `backend/open_webui/routers/notes.py` | 50-77 | GET /api/v1/notes/ endpoint |
| `backend/open_webui/routers/notes.py` | 163-195 | GET /api/v1/notes/{id} endpoint |
| `backend/open_webui/retrieval/utils.py` | 982-995 | Note content extraction for RAG |
| `src/lib/apis/notes/index.ts` | 1-286 | Frontend API client |

### OpenWebUI Note Referencing

| File | Line | Description |
|------|------|-------------|
| `src/lib/components/chat/MessageInput.svelte` | 902 | `#` trigger configuration |
| `src/lib/components/chat/MessageInput/Commands/Knowledge.svelte` | 72-123 | Note search in autocomplete |
| `backend/open_webui/utils/middleware.py` | 950-1066 | RAG retrieval orchestration |
| `backend/open_webui/utils/middleware.py` | 1572-1611 | Context injection into message |

### YouLab Letta Integration

| File | Line | Description |
|------|------|-------------|
| `src/letta_starter/server/agents.py` | 177-309 | Agent creation with curriculum |
| `src/letta_starter/server/main.py` | 38-96 | Service lifespan and initialization |
| `src/letta_starter/config/settings.py` | 106-172 | Service configuration |

## Recommendations

1. **For MVP**: Do nothing. OpenWebUI's `#` references already work with Letta because context injection happens before messages reach Letta.

2. **For Enhanced Agent Autonomy**: Implement Notes sync service (Option C) to give agents independent access to notes via file tools.

3. **Separate from Knowledge Sync**: Notes sync should be a separate service from Knowledge sync due to different data models and retrieval patterns.

4. **Consider Real-Time**: Notes support collaborative editing via Yjs. Real-time sync is complex - consider periodic polling (30-60s) instead.

5. **Access Control**: Must respect OpenWebUI's note access control when syncing. Only sync notes the user owns or has explicit read access to.

## Open Questions

1. Should notes be synced per-user (private folders) or shared (single folder with ACL)?
2. How to handle note deletions - delete from Letta or mark as archived?
3. What happens when a note title changes - new file or rename?
4. Should the agent be told about available notes in its system prompt?
5. How to prevent duplicate context when user references a note that's also in the synced folder?

## Related Research

- `thoughts/shared/plans/2026-01-09-openwebui-knowledge-sync.md` - Knowledge sync plan (different from Notes)
- `thoughts/shared/research/2026-01-09-openwebui-course-menu-item.md` - OpenWebUI sidebar research

## Conclusion

OpenWebUI note references **already work** with Letta through middleware context injection. Syncing notes to Letta folders would add **agent-initiated** note access via file tools, enabling the agent to proactively search and read user notes without explicit `#` references. This is an enhancement, not a requirement for basic functionality.
