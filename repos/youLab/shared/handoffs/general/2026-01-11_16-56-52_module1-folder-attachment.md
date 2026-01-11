---
date: 2026-01-11T16:56:52+07:00
researcher: ariasulin
git_commit: 166d668bbe5b80daaf2e2e1e00753dcc4f100e4c
branch: module1-local-poc
repository: youLab
topic: "Module 1 Local POC - Folder Attachment Implementation"
tags: [implementation, folder-attachment, agent-creation, file-sync]
status: in_progress
last_updated: 2026-01-11
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Module 1 Local POC - Folder Attachment Bug

## Task(s)
Working on **Phase 2** of `thoughts/shared/plans/2026-01-11-module1-local-poc.md` - wiring up TOML folder config to agent creation.

**Status:**
- [x] Added `get_file_sync_optional()` to sync router
- [x] Added `_attach_folders()` helper method to AgentManager
- [x] Added folder attachment logic to `create_agent_from_curriculum()`
- [x] `make verify-agent` passes (240 tests)
- [ ] **BUG DISCOVERED**: Folders not being attached despite code being in place
- [ ] Manual verification: folders appear in Letta ADE
- [ ] Commit and PR

## Critical References
- Implementation plan: `thoughts/shared/plans/2026-01-11-module1-local-poc.md`
- Course config with folder definition: `config/courses/college-essay/course.toml:78-80`

## Recent changes
- `src/youlab_server/server/agents.py:114-157` - Added `_attach_folders()` method
- `src/youlab_server/server/agents.py:299-318` - Added folder attachment call after agent creation
- `src/youlab_server/server/sync/router.py:32-40` - Added `get_file_sync_optional()` function
- `.env.example:4-9` - Added YOULAB HTTP SERVICE section
- `.env.example:91-111` - Added FILE SYNC section

## Learnings

### Bug: Folders Not Attaching Despite Code Being Present
The folder attachment code runs but doesn't appear to attach folders. Agents are created successfully but return 500 errors on the HTTP response. Key observations:

1. **Agent creation succeeds** - Agents appear in Letta and in `/agents` list endpoint
2. **HTTP returns 500** - Despite successful creation, the endpoint returns "Internal Server Error"
3. **No logs appearing** - Debug logs added to `create_agent_from_curriculum()` never appear in output
4. **Folders empty** - `GET /v1/agents/{id}/folders/` returns `[]` for all newly created agents

### Suspected Root Cause
The `_attach_folders()` method uses `asyncio.get_event_loop().run_until_complete()` inside a synchronous method that's called from an async FastAPI endpoint. This likely causes an event loop conflict since there's already a running event loop.

```python
# Problem code in agents.py:140-141
folder_id = asyncio.get_event_loop().run_until_complete(
    sync_service.ensure_folder(folder_name)
)
```

**Solution needed**: Either:
1. Make `_attach_folders()` async and use `await`
2. Or use `asyncio.run_coroutine_threadsafe()` with the existing loop
3. Or make `ensure_folder()` synchronous

### OpenWebUI Client Parsing Bugs (Separate Issue)
The OpenWebUI client has parsing errors with the current OpenWebUI API responses:
- `KeyError: 'user_id'` in `list_notes()` - API response format changed
- `AttributeError: 'str' object has no attribute 'get'` in `list_knowledge()` - files field format changed

These are in `src/youlab_server/server/sync/openwebui_client.py` and are blocking the sync service but NOT blocking folder attachment.

## Artifacts
- `src/youlab_server/server/agents.py` - Modified with folder attachment code
- `src/youlab_server/server/sync/router.py` - Added `get_file_sync_optional()`
- `.env.example` - Updated with file sync config
- `thoughts/shared/plans/2026-01-11-module1-local-poc.md` - Phase 2 automated criteria checked off

## Action Items & Next Steps

1. **Fix the asyncio event loop issue** in `_attach_folders()`:
   - Option A: Make the method async and propagate async up the call chain
   - Option B: Use thread-safe coroutine execution
   - Reference: `src/youlab_server/server/agents.py:114-157`

2. **Remove debug logging** once fix is verified:
   - `src/youlab_server/server/agents.py:301-306` and `:311`

3. **Test folder attachment manually**:
   - Create new agent via API
   - Verify folders appear in Letta ADE
   - Check for `college-essay-materials` (shared) and `user_{id}_notes` (private)

4. **Fix OpenWebUI client parsing** (separate task):
   - `src/youlab_server/server/sync/openwebui_client.py:87-91` - list_notes
   - `src/youlab_server/server/sync/openwebui_client.py:147` - list_knowledge

5. **Create commit and PR** once folder attachment works

## Other Notes

### Local Dev Environment
- Letta running on Docker: `http://localhost:8283`
- OpenWebUI running on Docker: `http://localhost:8080` (not 3000)
- YouLab service: `http://localhost:8100`
- `.env` file created from `.env.example` with OpenWebUI JWT token configured

### Test Agents Created
Multiple test agents exist in Letta that can be cleaned up:
- `test-folder-user-3`: agent-60bdee67-4297-4c81-81a5-240ccd9e3909
- `test-folder-user-4`: agent-2e25e782-b4ba-43a5-8b6d-494338129a60

### Course Config Note
The course.toml references module `01-first-impression` but the file is named `01-self-discovery.toml`. This causes a warning but doesn't block agent creation:
```
[warning] module_not_found module=01-first-impression
```
