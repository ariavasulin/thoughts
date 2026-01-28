# Dolt Memory Blocks - Phase 5 Handoff

## Current State

**Plan:** `thoughts/shared/plans/2026-01-28-dolt-memory-blocks.md`

### Completed Phases

- **Phase 1: Docker Infrastructure** ✅
  - `docker-compose.yml` with Dolt service on port 3307
  - `config/dolt/config.yaml` and `config/dolt/init.sql`
  - Dependencies: sqlalchemy, aiomysql, greenlet in pyproject.toml
  - Dolt settings in `src/ralph/config.py`

- **Phase 2: DoltClient** ✅
  - Full implementation at `src/ralph/dolt.py`
  - CRUD operations, version history, proposal workflow
  - All type checks and lint checks pass

- **Phase 3: Unit Tests** ⏸️ (Deferred)

- **Phase 4: HTTP API Layer** ✅
  - Created `src/ralph/api/__init__.py`
  - Created `src/ralph/api/blocks.py` with all endpoints
  - Integrated router into `src/ralph/server.py`
  - Added Dolt lifecycle hooks (startup/shutdown)
  - **Key change:** Removed title from proposal workflow - agents can only propose body changes

### Files Created/Modified in Phase 4

```
src/ralph/api/__init__.py      # NEW - exports blocks_router
src/ralph/api/blocks.py        # NEW - all block endpoints
src/ralph/server.py            # MODIFIED - added router + lifecycle hooks
src/ralph/dolt.py              # MODIFIED - removed title from create_proposal
pyproject.toml                 # MODIFIED - added [dependency-groups] dev
```

## Phase 5: Notes API Adapter

### Goal
Create a Notes API adapter that bridges OpenWebUI's Notes format to the Dolt-backed blocks. This allows the existing OpenWebUI Notes UI to work with the new Dolt storage.

### Key Files to Create
- `src/ralph/api/notes_adapter.py` - The adapter router

### Key Considerations

1. **OpenWebUI Notes Format**: The adapter needs to translate between:
   - OpenWebUI expects: `id`, `title`, `content.html`, `content.md`, `versions[]`, timestamps in nanoseconds
   - Dolt provides: `user_id`, `label`, `title`, `body`, `updated_at`, version history via DoltClient

2. **User ID handling**: The plan shows `user_id` as a query param, but you should research how OpenWebUI actually passes user context (likely via headers or auth context).

3. **Endpoint paths**: Mount at `/api/you/notes/` for OpenWebUI compatibility

4. **Research needed**:
   - Check `OpenWebUI/open-webui/` for how Notes API is consumed
   - Look at existing `src/youlab_server/server/notes_adapter.py` (legacy) for reference
   - Understand how OpenWebUI authenticates and passes user_id

### API Endpoints Needed

```
GET  /api/you/notes/           - List notes for user
GET  /api/you/notes/{note_id}  - Get single note with versions
POST /api/you/notes/{id}/update - Update a note
```

### Data Model Translation

```python
# OpenWebUI Note format
{
    "id": str,                    # maps to block.label
    "title": str,                 # maps to block.title
    "content": {
        "html": str,              # convert from block.body (markdown)
        "md": str,                # block.body
        "json": None              # always null
    },
    "versions": [                 # from dolt.get_block_history()
        {"sha": str, "message": str, "timestamp": int}  # nanoseconds!
    ],
    "user_id": str,
    "created_at": int,            # nanoseconds epoch
    "updated_at": int             # nanoseconds epoch
}
```

## Verification Commands

```bash
# Type check
uv run basedpyright src/ralph/api/notes_adapter.py

# Lint
uv run ruff check src/ralph/api/notes_adapter.py

# Start server (requires Dolt running)
docker compose up dolt -d
uv run ralph-server
```

## Important Context

- The schema changed: agents can only propose `body` changes, not `title`
- DoltClient dependency injection: use `Annotated[DoltClient, Depends(get_dolt_client)]`
- Timestamps need conversion to nanoseconds for OpenWebUI compatibility
