---
date: 2026-01-11T17:30:00+07:00
researcher: claude
git_commit: 166d668bbe5b80daaf2e2e1e00753dcc4f100e4c
branch: module1-local-poc
repository: YouLab
topic: "Folder Attachment Bug Fix - Async + Loader"
tags: [bugfix, folder-attachment, async, curriculum-loader]
status: completed
last_updated: 2026-01-11
last_updated_by: claude
type: implementation
---

# Handoff: Folder Attachment Bug Fix

## Task(s)

Resumed from `2026-01-11_16-56-52_module1-folder-attachment.md` to fix the folder attachment bug identified in Phase 2 of Module 1 Local POC.

**Status:**
- [x] Fixed asyncio event loop conflict in `_attach_folders()`
- [x] Fixed folder config not being parsed from TOML
- [x] All 240 tests passing
- [x] Manual verification: folders attach correctly to new agents
- [ ] **DISCOVERY**: Letta folders are NOT visible as OpenWebUI Knowledge collections

## Critical Finding

**Letta folders ≠ OpenWebUI Knowledge collections**

The folder attachment fix works correctly - Letta agents now have folders attached. However, these Letta folders are **not** the same as OpenWebUI's Knowledge system. They are separate storage systems:

- **OpenWebUI Knowledge**: Collections visible at `/workspace/knowledge` in OpenWebUI UI
- **Letta Folders**: Agent-attached storage accessible via Letta's archival memory tools

The FileSyncService syncs files FROM OpenWebUI TO Letta, but the folders created in Letta don't appear back in OpenWebUI's Knowledge UI.

## Recent Changes

### Bug 1: Asyncio Event Loop Conflict (FIXED)

**Problem**: `asyncio.get_event_loop().run_until_complete()` called inside sync method from async FastAPI endpoint.

**Files Changed**:
- `src/youlab_server/server/agents.py:118-155` - Made `_attach_folders()` async
- `src/youlab_server/server/agents.py:196-224` - Made `create_agent()` async
- `src/youlab_server/server/agents.py:226-360` - Made `create_agent_from_curriculum()` async
- `src/youlab_server/server/main.py:173` - Added `await` to endpoint call
- `tests/test_server/test_agents.py` - Updated 6 tests to async
- `tests/test_server/test_endpoints.py` - Updated 3 mocks to async functions

### Bug 2: Folder Config Not Parsed (FIXED)

**Problem**: `_parse_agent_config()` in curriculum loader didn't pass `folders` to `AgentConfig`.

**Files Changed**:
- `src/youlab_server/curriculum/loader.py:17` - Added `AgentFoldersConfig` import
- `src/youlab_server/curriculum/loader.py:206-222` - Added folder parsing and passing to `AgentConfig`

## Verification

```bash
# Tests pass
make verify-agent  # 240 tests, all pass

# Manual verification - agent created with both folders
curl -X POST http://localhost:8100/agents \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test-user", "agent_type": "college-essay"}'

# Folders attached correctly
curl http://localhost:8283/v1/agents/{agent_id}/folders/
# Returns: college-essay-materials, user_{id}_notes
```

## Artifacts

- `src/youlab_server/server/agents.py` - Async folder attachment
- `src/youlab_server/curriculum/loader.py` - Folder config parsing
- `tests/test_server/test_agents.py` - Updated async tests
- `tests/test_server/test_endpoints.py` - Updated async mocks

## Action Items & Next Steps

1. **Commit the changes** - All code changes are complete and tested

2. **Clarify folder visibility requirement** - Need to determine if:
   - Option A: Letta folders should sync BACK to OpenWebUI (reverse sync)
   - Option B: Agent should use OpenWebUI Knowledge collections directly (different approach)
   - Option C: Current behavior is acceptable (folders in Letta only)

3. **Fix OpenWebUI client parsing** (separate task from original handoff):
   - `src/youlab_server/server/sync/openwebui_client.py:91` - `list_notes()` KeyError on `user_id`
   - `src/youlab_server/server/sync/openwebui_client.py:147` - `list_knowledge()` AttributeError

## Architecture Note

Current data flow:
```
OpenWebUI Knowledge → FileSyncService → Letta Folders → Agent Access
         ↑                                    ↓
    (User uploads)                    (Agent can search)

    NOT synced back ←─────────────────────────┘
```

To make folders visible in OpenWebUI, would need either:
1. Reverse sync from Letta → OpenWebUI
2. Direct OpenWebUI Knowledge API integration in agent tools
3. Different architecture altogether

## Other Notes

### Test Agents Created
- `agent-e3ca866c-4e06-4cb4-88bb-4877251a70a4` - Admin user (7a41011b-5255-4225-b75e-1d8484d0e37f)
- `agent-2927d554-1d28-4666-98d5-dcf10d10470b` - test-both-folders-1768126922

### Local Dev Environment
- Letta: `http://localhost:8283`
- OpenWebUI: `http://localhost:8080`
- YouLab Service: `http://localhost:8100`
