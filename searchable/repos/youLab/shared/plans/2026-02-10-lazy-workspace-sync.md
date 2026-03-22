# Lazy Workspace-to-OpenWebUI Sync via Post-Hooks

## Overview

Wire up event-driven sync so that when the Ralph agent writes/deletes a file via FileTools, that specific file is immediately synced to the user's OpenWebUI knowledge base in the background. No polling, no scanning, no periodic jobs. The sync happens fire-and-forget after the agent tool call, adding zero latency to the chat response.

## Current State Analysis

**What exists:**
- `src/ralph/sync/workspace_sync.py` - Full sync module with `sync_to_openwebui()` that scans the entire workspace
- `src/ralph/sync/openwebui_client.py` - HTTP client for OpenWebUI file/KB APIs
- `src/ralph/sync/knowledge.py` - Per-user KB creation with in-memory cache
- `src/ralph/api/workspace.py` - HTTP endpoints including manual `POST /sync`
- `src/ralph/config.py:48-52` - Config: `openwebui_url`, `openwebui_api_key`, `sync_to_openwebui`

**What's missing:**
- No hook on FileTools to detect agent file writes
- No per-file sync (current `sync_to_openwebui` always scans everything)
- A new `OpenWebUIClient` is created per API request (no reuse)
- No way to sync a single file without scanning the whole workspace

### Key Discoveries:
- Agno's `Function` class has `post_hook` field that fires after tool execution (`agno/tools/function.py:106`)
- Post-hooks receive `fc: FunctionCall` with `fc.arguments` (file_name, contents) and `fc.error`
- Post-hooks can be async (`_handle_post_hook_async` at function.py:969)
- FileTools registers these mutating functions: `save_file`, `replace_file_chunk`, `delete_file`
- `strip_agno_fields()` in `server.py:68` already iterates `toolkit.functions` - same pattern for attaching hooks
- `fc.arguments` contains `file_name` for all three mutating tools

## Desired End State

After a Ralph agent calls `save_file`, `replace_file_chunk`, or `delete_file`, the affected file is automatically synced to the user's OpenWebUI knowledge base (`workspace-{user_id}`) as a fire-and-forget background task. No chat latency impact. No full workspace scans.

### Verification:
1. Start Ralph server with `RALPH_OPENWEBUI_URL` and `RALPH_OPENWEBUI_API_KEY` set
2. Send a chat message that causes the agent to write a file (e.g., "create a file called test.txt with hello world")
3. Within seconds, `test.txt` appears in the user's OpenWebUI knowledge base
4. Send another message that modifies the file
5. The KB version updates (old file removed, new file uploaded)
6. Send a message that deletes the file
7. The file is removed from the KB

## What We're NOT Doing

- Reverse sync (OpenWebUI KB -> workspace) - deferred until OpenWebUI adds webhook support
- Full workspace scanning or periodic polling
- Shell command interception (agents writing via `run_shell_command` won't trigger sync)
- Sync state persistence to `.sync_state.json` (we track KB file IDs in memory per-process; state file is only for the manual API endpoint which still works independently)
- Changes to the workspace HTTP API endpoints (they continue working as-is)

## Implementation Approach

The core idea: attach async `post_hook` callbacks to FileTools' mutating functions. When the hook fires, it reads the file from disk, uploads it to OpenWebUI, and adds it to the user's KB. All of this happens in the background via `asyncio.create_task` so it doesn't block the agent's response.

We need a process-level singleton `OpenWebUIClient` (httpx connection pool) and `KnowledgeService` (KB ID cache) to avoid creating new HTTP clients per request.

## Phase 1: Singleton OpenWebUI Client

### Overview
Create a process-level `OpenWebUIClient` and `KnowledgeService` that persist across requests, reusing HTTP connections and caching KB IDs.

### Changes Required:

#### 1. New module: sync service singleton
**File**: `src/ralph/sync/service.py`

```python
"""Process-level sync service singleton."""

from __future__ import annotations

import structlog
from ralph.config import get_settings
from ralph.sync.openwebui_client import OpenWebUIClient
from ralph.sync.knowledge import KnowledgeService

log = structlog.get_logger()

_client: OpenWebUIClient | None = None
_knowledge: KnowledgeService | None = None


def get_sync_client() -> OpenWebUIClient | None:
    """Get or create process-level OpenWebUI client."""
    global _client
    settings = get_settings()
    if not settings.openwebui_url or not settings.openwebui_api_key:
        return None
    if not settings.sync_to_openwebui:
        return None
    if _client is None:
        _client = OpenWebUIClient(
            base_url=settings.openwebui_url,
            api_key=settings.openwebui_api_key,
        )
        log.info("sync_client_created", base_url=settings.openwebui_url)
    return _client


def get_knowledge_service() -> KnowledgeService | None:
    """Get or create process-level knowledge service."""
    global _knowledge
    client = get_sync_client()
    if client is None:
        return None
    if _knowledge is None:
        settings = get_settings()
        _knowledge = KnowledgeService(
            openwebui_client=client,
            name_prefix=settings.sync_knowledge_prefix,
        )
    return _knowledge


async def close_sync_client() -> None:
    """Close the singleton client. Call on shutdown."""
    global _client, _knowledge
    if _client:
        await _client.close()
        _client = None
    _knowledge = None
```

#### 2. Wire shutdown into lifespan
**File**: `src/ralph/server.py`

Add `close_sync_client()` call to the lifespan shutdown sequence.

```python
# Add import
from ralph.sync.service import close_sync_client

# In lifespan(), after yield, before close_dolt_client:
    await close_sync_client()
    log.info("sync_client_closed")
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] `make test-agent` passes
- [ ] Server starts without errors: `uv run ralph-server`

---

## Phase 2: Per-File Sync Function

### Overview
Add a `sync_file_to_openwebui()` function that uploads/updates/deletes a single file in the user's KB without scanning the workspace.

### Changes Required:

#### 1. Add per-file sync function
**File**: `src/ralph/sync/workspace_sync.py`

Add a module-level async function (not a method on WorkspaceSync, since we don't need a full WorkspaceSync instance for single-file operations):

```python
async def sync_file_to_kb(
    file_path: Path,
    user_id: str,
    openwebui_client: OpenWebUIClient,
    knowledge_service: KnowledgeService,
) -> None:
    """
    Sync a single file to the user's OpenWebUI knowledge base.

    If the file exists on disk, uploads it (replacing any previous version).
    If the file doesn't exist, removes it from the KB.
    """
    kb_id = await knowledge_service.get_or_create_knowledge(user_id)

    if file_path.exists() and file_path.is_file():
        content = file_path.read_bytes()
        file_hash = compute_hash(content)

        # Check if file already exists in KB (by filename)
        kb_files = await openwebui_client.get_knowledge_files(kb_id)
        for existing in kb_files:
            existing_name = existing.get("meta", {}).get("name", existing.get("filename", ""))
            if existing_name == file_path.name:
                # Same content? Skip.
                # Different? Delete old, upload new.
                await openwebui_client.remove_file_from_knowledge(kb_id, existing["id"])
                await openwebui_client.delete_file(existing["id"])
                break

        # Upload new version
        file_info = await openwebui_client.upload_file(
            filename=file_path.name,
            content=content,
        )
        await openwebui_client.add_file_to_knowledge(kb_id, file_info["id"])

        log.info(
            "file_synced_to_kb",
            user_id=user_id,
            file=file_path.name,
            hash=file_hash[:20],
            kb_id=kb_id,
        )
    else:
        # File was deleted - remove from KB
        kb_files = await openwebui_client.get_knowledge_files(kb_id)
        for existing in kb_files:
            existing_name = existing.get("meta", {}).get("name", existing.get("filename", ""))
            if existing_name == file_path.name:
                await openwebui_client.remove_file_from_knowledge(kb_id, existing["id"])
                await openwebui_client.delete_file(existing["id"])
                log.info(
                    "file_removed_from_kb",
                    user_id=user_id,
                    file=file_path.name,
                    kb_id=kb_id,
                )
                break
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] `make test-agent` passes

---

## Phase 3: Post-Hook Wiring

### Overview
Create a function that attaches `post_hook` callbacks to FileTools' mutating functions (`save_file`, `replace_file_chunk`, `delete_file`). When fired, the hook resolves the full file path and dispatches `sync_file_to_kb` as a fire-and-forget `asyncio.Task`.

### Changes Required:

#### 1. New module: hook attachment
**File**: `src/ralph/sync/hooks.py`

```python
"""Post-hooks for FileTools to trigger lazy sync."""

from __future__ import annotations

import asyncio
from pathlib import Path

import structlog
from agno.tools.file import FileTools
from agno.tools.function import FunctionCall

from ralph.sync.service import get_knowledge_service, get_sync_client
from ralph.sync.workspace_sync import sync_file_to_kb

log = structlog.get_logger()

# Track background tasks to prevent GC
_background_tasks: set[asyncio.Task[None]] = set()


def _fire_and_forget(coro) -> None:  # noqa: ANN001
    """Schedule a coroutine as a background task."""
    try:
        loop = asyncio.get_running_loop()
    except RuntimeError:
        log.warning("sync_hook_no_event_loop")
        return

    task = loop.create_task(coro)
    _background_tasks.add(task)
    task.add_done_callback(_background_tasks.discard)


def _make_sync_hook(workspace: Path, user_id: str):  # noqa: ANN202
    """Create a post_hook closure for file mutation tools."""

    def _on_file_mutated(fc: FunctionCall) -> None:
        # Skip if the tool call failed
        if fc.error:
            return

        # Extract file_name from arguments
        args = fc.arguments or {}
        file_name = args.get("file_name")
        if not file_name:
            return

        # Resolve full path
        file_path = (workspace / file_name).resolve()

        # Safety: ensure it's within workspace
        try:
            file_path.relative_to(workspace.resolve())
        except ValueError:
            log.warning("sync_hook_path_escape", file_name=file_name)
            return

        # Check sync is configured
        client = get_sync_client()
        knowledge = get_knowledge_service()
        if not client or not knowledge:
            return

        log.debug("sync_hook_fired", tool=fc.function.name, file=file_name, user_id=user_id)
        _fire_and_forget(sync_file_to_kb(file_path, user_id, client, knowledge))

    return _on_file_mutated


def attach_sync_hooks(file_tools: FileTools, workspace: Path, user_id: str) -> FileTools:
    """
    Attach post_hooks to FileTools' mutating functions.

    Mutating functions: save_file, replace_file_chunk, delete_file.
    Returns the same FileTools instance (mutated in place).
    """
    hook = _make_sync_hook(workspace, user_id)

    for func_name in ("save_file", "replace_file_chunk", "delete_file"):
        func = file_tools.functions.get(func_name)
        if func is not None:
            func.post_hook = hook

    return file_tools
```

#### 2. Wire hooks into agent creation
**File**: `src/ralph/server.py`

In the `generate()` function inside `chat_stream`, after creating FileTools but before passing to Agent:

```python
# Add import
from ralph.sync.hooks import attach_sync_hooks

# Replace:
#   strip_agno_fields(FileTools(base_dir=workspace)),
# With:
    file_tools = FileTools(base_dir=workspace)
    attach_sync_hooks(file_tools, workspace, request.user_id)
    strip_agno_fields(file_tools)
```

The `tools` list becomes:
```python
tools=[
    strip_agno_fields(ShellTools(base_dir=workspace)),
    file_tools,  # already stripped above
    strip_agno_fields(HonchoTools()),
    strip_agno_fields(MemoryBlockTools()),
    strip_agno_fields(LaTeXTools(workspace=workspace)),
],
```

#### 3. Wire hooks into background tasks
**File**: `src/ralph/background/tools.py`

Same pattern for background agent FileTools:

```python
# Add import
from ralph.sync.hooks import attach_sync_hooks

# In create_tools_for_task, replace the file_tools branch:
    if name == "file_tools":
        ft = FileTools(base_dir=workspace)
        attach_sync_hooks(ft, workspace, user_id)
        tools.append(strip_agno_fields(ft))
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] `make test-agent` passes
- [ ] Server starts without errors: `uv run ralph-server`

#### Manual Verification:
- [ ] Send a chat message asking the agent to create a file. Verify it appears in the user's OpenWebUI KB within a few seconds.
- [ ] Send a message asking the agent to modify the file. Verify the KB version updates.
- [ ] Send a message asking the agent to delete the file. Verify it's removed from the KB.
- [ ] Verify chat response latency is not impacted (sync happens in background).
- [ ] Check Ralph server logs show `sync_hook_fired` and `file_synced_to_kb` entries.
- [ ] Verify sync doesn't happen when `RALPH_SYNC_TO_OPENWEBUI=false` or OpenWebUI creds are missing.

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding.

---

## Testing Strategy

### Unit Tests:
- Test `sync_file_to_kb` with mocked OpenWebUI client (upload, update, delete cases)
- Test `attach_sync_hooks` sets post_hook on correct functions
- Test `_make_sync_hook` skips on error, missing file_name, path escape
- Test `get_sync_client` returns None when config is missing

### Integration Tests:
- Test that writing a file via FileTools triggers the post_hook (mock the sync call)

### Manual Testing Steps:
1. Start Ralph with OpenWebUI credentials configured
2. Chat with agent, ask it to create `notes.md` with some content
3. Check OpenWebUI KB `workspace-{user_id}` - file should appear
4. Ask agent to update `notes.md`
5. Check KB - file content should be updated
6. Ask agent to delete `notes.md`
7. Check KB - file should be gone
8. Verify no errors in Ralph logs
9. Verify chat responses feel equally fast

## Performance Considerations

- **Zero chat latency impact**: Sync is fire-and-forget via `asyncio.create_task`. The chat SSE stream is already complete before sync runs.
- **No scanning**: Only the specific file that was written/deleted gets synced. No `rglob`, no full workspace hash.
- **Connection reuse**: Singleton `OpenWebUIClient` reuses httpx connection pool across all syncs.
- **KB ID caching**: `KnowledgeService` caches `user_id -> kb_id` mapping in memory, avoiding repeated lookups.
- **One KB file list per sync**: Each `sync_file_to_kb` call does one `GET /knowledge/{id}` to find existing files. This is unavoidable since OpenWebUI doesn't support "get file by name".
- **OpenWebUI not slowed down**: We only make API calls (file upload, KB add/remove) when the agent actually writes a file. No background polling, no scheduled jobs hitting OpenWebUI.

## References

- Research: `thoughts/shared/research/2026-02-10-workspace-openwebui-sync-status.md`
- Three-way sync plan (original): `thoughts/shared/plans/2026-01-28-three-way-workspace-sync.md`
- Agno post_hook implementation: `.venv/lib/python3.12/site-packages/agno/tools/function.py:106,718-737,969-989`
- Agno FileTools: `.venv/lib/python3.12/site-packages/agno/tools/file.py:64,112,167`
