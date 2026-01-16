# ARI-85: Fix Letta Tool Sandbox Compatibility

## Overview

During POC testing (ARI-85), Letta tools (`query_honcho`, `edit_memory_block`) fail in Letta's Docker sandbox because they import `youlab_server` modules and rely on global state that doesn't exist in the isolated environment.

**Root Cause**: Tools were written assuming in-process execution within the YouLab server. Letta executes tools in a Docker sandbox where:
1. Global state (`_honcho_client`, `_letta_client`, `_user_context`) doesn't transfer
2. `youlab_server` package isn't installed
3. Runtime imports fail with `ModuleNotFoundError`

**Solution**: Rewrite tools to be **self-contained** and make **HTTP calls back to YouLab server** at `http://host.docker.internal:8100/...`

## Error Evidence

```
# query_honcho failure:
NameError: name '_honcho_client' is not defined

# edit_memory_block failure:
ModuleNotFoundError: No module named 'youlab_server'
```

## Current Architecture (Broken)

```
Letta Server ─── Docker Sandbox ─── Tool Source Code
                     │
                     └── ❌ No youlab_server package
                     └── ❌ No global state (_honcho_client, etc.)
                     └── ❌ No Letta client reference
```

## Desired Architecture (Fixed)

```
Letta Server ─── Docker Sandbox ─── Self-contained Tool
                     │
                     └── Uses stdlib only (urllib.request)
                     └── Makes HTTP calls to YouLab server
                     └── http://host.docker.internal:8100/...
                            │
                            ├── POST /honcho/query
                            └── POST /users/{user_id}/blocks/{block}/propose
```

## Current Tool Analysis

### `query_honcho` (tools/dialectic.py:31-94)

**Dependencies that break in sandbox:**
- Global `_honcho_client` (line 16) - None in sandbox
- Global `_user_context` dict (line 17) - Empty in sandbox
- Runtime import of `SessionScope` enum (line 69)
- Asyncio event loop management

**Current signature:**
```python
def query_honcho(
    question: str,
    session_scope: str = "all",
    agent_state: dict[str, Any] | None = None,
) -> str:
```

### `edit_memory_block` (tools/memory.py:27-108)

**Dependencies that break in sandbox:**
- Global `_letta_client` (line 18)
- Runtime import of `get_storage_manager()` (line 53)
- Runtime import of `UserBlockManager` (line 54)
- Full storage layer chain

**Current signature:**
```python
def edit_memory_block(
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
    agent_state: dict[str, Any] | None = None,
) -> str:
```

## Implementation Plan

### Phase 1: Add New HTTP Endpoints

Create two new HTTP endpoints that tools will call from within the sandbox.

#### 1.1: `POST /honcho/query` Endpoint

**File**: `src/youlab_server/server/honcho.py` (NEW)

```python
"""HTTP endpoint for Honcho dialectic queries (called by sandboxed tools)."""

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, Field
import structlog

from youlab_server.honcho.client import HonchoClient, SessionScope
from youlab_server.server.main import get_honcho_client

log = structlog.get_logger()
router = APIRouter(prefix="/honcho", tags=["honcho"])


class QueryRequest(BaseModel):
    """Request body for dialectic query."""

    user_id: str = Field(..., description="User ID to query conversations for")
    question: str = Field(..., description="Question to ask about the user")
    session_scope: str = Field(default="all", description="Scope: all, recent, current")


class QueryResponse(BaseModel):
    """Response from dialectic query."""

    insight: str | None = Field(description="Insight from conversation history")
    success: bool = Field(default=True)
    error: str | None = Field(default=None)


@router.post("/query", response_model=QueryResponse)
async def query_dialectic(
    request: QueryRequest,
    honcho_client: HonchoClient | None = Depends(get_honcho_client),
) -> QueryResponse:
    """
    Query Honcho dialectic for insights about a user.

    Called by sandboxed Letta tools that can't access the Honcho client directly.
    """
    if honcho_client is None:
        return QueryResponse(
            insight=None,
            success=False,
            error="Honcho client not configured",
        )

    try:
        scope = SessionScope(request.session_scope)
    except ValueError:
        scope = SessionScope.ALL

    log.info(
        "honcho_query_via_http",
        user_id=request.user_id,
        question_preview=request.question[:50],
    )

    response = await honcho_client.query_dialectic(
        user_id=request.user_id,
        question=request.question,
        session_scope=scope,
    )

    if response is None:
        return QueryResponse(
            insight=None,
            success=False,
            error="No insight available",
        )

    return QueryResponse(insight=response.insight, success=True)
```

**Changes to main.py:**
- Add `get_honcho_client` dependency
- Include router: `app.include_router(honcho_router)`

#### 1.2: `POST /users/{user_id}/blocks/{block}/propose` Endpoint

**File**: `src/youlab_server/server/blocks.py` (MODIFY)

Add new endpoint to existing blocks router:

```python
class ProposeEditRequest(BaseModel):
    """Request body for proposing a memory block edit (from sandboxed tools)."""

    agent_id: str = Field(..., description="Agent proposing the edit")
    field: str | None = Field(default=None, description="Specific field to edit")
    content: str = Field(..., description="Proposed new content")
    strategy: str = Field(default="append", description="Merge strategy: append, replace, full_replace")
    reasoning: str = Field(default="", description="Why agent wants this change")


class ProposeEditResponse(BaseModel):
    """Response from propose edit."""

    diff_id: str | None = Field(description="ID of created pending diff")
    success: bool = Field(default=True)
    error: str | None = Field(default=None)


@router.post("/{label}/propose", response_model=ProposeEditResponse)
async def propose_block_edit(
    user_id: str,
    label: str,
    request: ProposeEditRequest,
    storage: GitUserStorageManager = Depends(get_storage_manager),
) -> ProposeEditResponse:
    """
    Propose an edit to a memory block.

    Called by sandboxed Letta tools that can't access UserBlockManager directly.
    Creates a PendingDiff for user approval.
    """
    user_storage = storage.get(user_id)
    if user_storage is None:
        return ProposeEditResponse(
            diff_id=None,
            success=False,
            error=f"User {user_id} not found",
        )

    manager = UserBlockManager(user_storage, letta_client=None)  # No Letta sync needed

    # Map strategy to operation
    operation = "append" if request.strategy == "append" else "replace"
    if request.strategy == "full_replace":
        operation = "full_replace"

    try:
        diff = manager.propose_edit(
            agent_id=request.agent_id,
            block_label=label,
            field=request.field,
            operation=operation,
            proposed_value=request.content,
            reasoning=request.reasoning,
            confidence="medium",  # Default for background agents
        )

        log.info(
            "block_edit_proposed_via_http",
            user_id=user_id,
            block=label,
            diff_id=diff.id,
        )

        return ProposeEditResponse(diff_id=diff.id, success=True)
    except Exception as e:
        log.error("block_propose_failed", error=str(e), user_id=user_id, block=label)
        return ProposeEditResponse(
            diff_id=None,
            success=False,
            error=str(e),
        )
```

### Phase 2: Create Self-Contained Tool Module

Create a new module with sandbox-compatible tools that use HTTP.

**File**: `src/youlab_server/tools/sandbox.py` (NEW)

```python
"""
Self-contained Letta tools for Docker sandbox execution.

IMPORTANT: These tools must:
1. Use only Python stdlib (no external dependencies)
2. Make HTTP calls to YouLab server at YOULAB_SERVER_URL
3. Not import any youlab_server modules
4. Be completely self-contained

The server URL is configurable via the YOULAB_SERVER_URL constant.
For Docker sandbox: http://host.docker.internal:8100
"""

# =============================================================================
# SANDBOX-COMPATIBLE TOOLS - No external imports allowed!
# =============================================================================

# Server URL - Docker's special DNS for host machine
YOULAB_SERVER_URL = "http://host.docker.internal:8100"


def query_honcho(
    question: str,
    session_scope: str = "all",
    agent_state: dict | None = None,
) -> str:
    """
    Query conversation history to gain insights about the student.

    Use this tool to understand:
    - What the student has discussed previously
    - Their learning patterns and preferences
    - Progress they've made in past conversations
    - Any concerns or challenges they've mentioned

    Args:
        question: Natural language question about the student (e.g., "What progress has this student made?")
        session_scope: Which conversations to query - "all" (default), "recent", or "current"
        agent_state: Injected by Letta - contains agent_id for context lookup

    Returns:
        Insight about the student based on conversation history, or an error message.

    Example:
        query_honcho("What are this student's main interests?")
        query_honcho("Has the student struggled with any topics?", session_scope="recent")
    """
    import json
    import urllib.request
    import urllib.error

    # Extract user_id from agent metadata
    if agent_state is None:
        return "Error: No agent state available"

    # User ID should be in agent metadata or we need to look it up
    # The server endpoint now handles user lookup via agent context
    user_id = agent_state.get("user_id")
    if not user_id:
        # Try to get from agent metadata
        metadata = agent_state.get("metadata", {})
        user_id = metadata.get("youlab_user_id")

    if not user_id:
        return "Error: Could not determine user_id from agent state"

    # Build request
    url = f"{YOULAB_SERVER_URL}/honcho/query"
    payload = json.dumps({
        "user_id": user_id,
        "question": question,
        "session_scope": session_scope,
    }).encode("utf-8")

    headers = {"Content-Type": "application/json"}

    try:
        req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
        with urllib.request.urlopen(req, timeout=30) as response:
            result = json.loads(response.read().decode("utf-8"))

            if not result.get("success"):
                error = result.get("error", "Unknown error")
                return f"Query failed: {error}"

            insight = result.get("insight")
            if insight:
                return insight
            else:
                return "No insight available from conversation history."

    except urllib.error.URLError as e:
        return f"Error connecting to server: {e.reason}"
    except urllib.error.HTTPError as e:
        return f"Server error: {e.code} {e.reason}"
    except json.JSONDecodeError:
        return "Error: Invalid response from server"
    except Exception as e:
        return f"Error: {str(e)}"


def edit_memory_block(
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
    agent_state: dict | None = None,
) -> str:
    """
    Propose an update to a memory block field.

    The change will be queued for user approval before being applied.
    Use this to record important observations about the student.

    Args:
        block: Name of the memory block to update (e.g., "progress", "operating_manual")
        field: Specific field within the block (e.g., "notes", "effective_strategies")
        content: The content to add or replace
        strategy: How to apply the change:
            - "append": Add to existing content (default, best for notes)
            - "replace": Replace specific content
            - "full_replace": Replace the entire field
        reasoning: Brief explanation of why you're making this change
        agent_state: Injected by Letta - contains agent_id and user context

    Returns:
        Confirmation message with diff ID, or an error message.

    Example:
        edit_memory_block(
            block="progress",
            field="notes",
            content="Student showed strong interest in essay structure today.",
            reasoning="Recording observation from today's session"
        )
    """
    import json
    import urllib.request
    import urllib.error

    # Extract identifiers from agent state
    if agent_state is None:
        return "Error: No agent state available"

    agent_id = agent_state.get("agent_id")
    if not agent_id:
        return "Error: No agent_id in agent state"

    # Get user_id from metadata
    user_id = agent_state.get("user_id")
    if not user_id:
        metadata = agent_state.get("metadata", {})
        user_id = metadata.get("youlab_user_id")

    if not user_id:
        return "Error: Could not determine user_id from agent state"

    # Build request
    url = f"{YOULAB_SERVER_URL}/users/{user_id}/blocks/{block}/propose"
    payload = json.dumps({
        "agent_id": agent_id,
        "field": field,
        "content": content,
        "strategy": strategy,
        "reasoning": reasoning,
    }).encode("utf-8")

    headers = {"Content-Type": "application/json"}

    try:
        req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
        with urllib.request.urlopen(req, timeout=30) as response:
            result = json.loads(response.read().decode("utf-8"))

            if not result.get("success"):
                error = result.get("error", "Unknown error")
                return f"Failed to propose edit: {error}"

            diff_id = result.get("diff_id", "unknown")
            # Truncate diff_id for readability
            short_id = diff_id[:8] if len(diff_id) > 8 else diff_id
            return f"Proposed change to {block}.{field} (ID: {short_id}). User will review."

    except urllib.error.URLError as e:
        return f"Error connecting to server: {e.reason}"
    except urllib.error.HTTPError as e:
        return f"Server error: {e.code} {e.reason}"
    except json.JSONDecodeError:
        return "Error: Invalid response from server"
    except Exception as e:
        return f"Error: {str(e)}"
```

### Phase 3: Update Tool Registration

Modify the server to register the sandbox-compatible tools instead of the current ones.

**File**: `src/youlab_server/server/main.py`

**Changes at startup (around line 85-95):**

```python
# Register custom tools with Letta
try:
    client = app.state.agent_manager.client

    # Import sandbox-compatible tools (these use HTTP, not direct imports)
    from youlab_server.tools.sandbox import (
        query_honcho as sandbox_query_honcho,
        edit_memory_block as sandbox_edit_memory_block,
    )

    # Unregister old tools if they exist (prevent conflicts)
    try:
        client.tools.delete(name="query_honcho")
        client.tools.delete(name="edit_memory_block")
        log.info("old_tools_unregistered")
    except Exception:
        pass  # Tools may not exist yet

    # Register sandbox-compatible versions
    client.tools.upsert_from_function(func=sandbox_query_honcho)
    client.tools.upsert_from_function(func=sandbox_edit_memory_block)

    # advance_lesson stays as-is for now (Phase 4 future work)
    from youlab_server.tools.curriculum import advance_lesson
    client.tools.upsert_from_function(func=advance_lesson)

    log.info(
        "sandbox_tools_registered",
        tools=["query_honcho", "edit_memory_block", "advance_lesson"]
    )
except Exception as e:
    log.warning("custom_tools_registration_failed", error=str(e))
```

### Phase 4: Update Background Agent Factory

Ensure `BackgroundAgentFactory` provides `user_id` in agent metadata so sandbox tools can access it.

**File**: `src/youlab_server/background/factory.py`

**In `_create_agent()` around line 146-150:**

```python
# Ensure user_id is in metadata for sandbox tool access
metadata = {
    "youlab_background_task": task_name,
    "youlab_user_id": user_id,  # Critical for sandbox tools
    "youlab_agent_type": "background",
}

agent = self._client.agents.create(
    name=agent_name,
    model=self._default_model,
    system=system_prompt,
    tools=[self._client.tools.get(name=t) for t in tools],
    memory_blocks=letta_blocks,
    metadata=metadata,
)
```

**In `_update_agent_context()` around line 186-202:**

```python
# Also ensure metadata includes user_id (for existing agents)
try:
    self._client.agents.update(
        agent_id=agent_id,
        metadata={
            "youlab_background_task": task_name,
            "youlab_user_id": user_id,
            "youlab_agent_type": "background",
        },
    )
except Exception as e:
    log.debug("metadata_update_skipped", error=str(e))
```

### Phase 5: Keep Legacy Tools for Local Development

The original tools still work when running in-process (not in sandbox). Keep them for:
- Local development without Docker
- Direct API testing
- Fallback if sandbox tools fail

**File**: `src/youlab_server/tools/__init__.py`

Update to expose both versions:

```python
# Sandbox tools (HTTP-based, for Letta registration)
from youlab_server.tools.sandbox import (
    query_honcho as sandbox_query_honcho,
    edit_memory_block as sandbox_edit_memory_block,
)

# Legacy tools (in-process, for direct use)
from youlab_server.tools.dialectic import query_honcho as local_query_honcho
from youlab_server.tools.memory import edit_memory_block as local_edit_memory_block

# Default exports are sandbox versions (what gets registered with Letta)
query_honcho = sandbox_query_honcho
edit_memory_block = sandbox_edit_memory_block
```

---

## Success Criteria

### Automated Verification

- [ ] `make check-agent` passes (lint + typecheck)
- [ ] New endpoints return correct schemas
- [ ] Sandbox tools use only stdlib imports
- [ ] No `from youlab_server import ...` in sandbox.py

### Manual Verification

**1. Test Honcho endpoint directly:**
```bash
curl -X POST http://localhost:8100/honcho/query \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "7a41011b-5255-4225-b75e-1d8484d0e37f",
    "question": "What has this student been working on?",
    "session_scope": "all"
  }' | jq .
```

**2. Test propose endpoint directly:**
```bash
curl -X POST http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks/progress/propose \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "test-agent",
    "field": "notes",
    "content": "Test note from HTTP endpoint",
    "strategy": "append",
    "reasoning": "Testing sandbox tool endpoint"
  }' | jq .
```

**3. Trigger background agent and verify it works:**
```bash
curl -X POST http://localhost:8100/background/poc-progress-tracker/run \
  -H "Content-Type: application/json" \
  -d '{"user_ids": ["7a41011b-5255-4225-b75e-1d8484d0e37f"]}' | jq .
```

**4. Check for pending diffs:**
```bash
curl http://localhost:8100/users/7a41011b-5255-4225-b75e-1d8484d0e37f/blocks/progress/diffs | jq .
```

---

## Files Changed

| File | Change |
|------|--------|
| `src/youlab_server/server/honcho.py` | NEW - HTTP endpoint for Honcho queries |
| `src/youlab_server/server/blocks.py` | ADD - `/propose` endpoint |
| `src/youlab_server/server/main.py` | MODIFY - Register sandbox tools, add router |
| `src/youlab_server/tools/sandbox.py` | NEW - Self-contained tools |
| `src/youlab_server/tools/__init__.py` | MODIFY - Export sandbox tools |
| `src/youlab_server/background/factory.py` | MODIFY - Include user_id in metadata |

---

## Key Design Decisions

### Why HTTP instead of direct imports?

1. **Sandbox isolation**: Letta's Docker sandbox doesn't have `youlab_server` installed
2. **Clean separation**: Tools are truly independent, can be tested in isolation
3. **Future-proof**: Supports potential multi-container deployments

### Why `host.docker.internal`?

- Standard Docker DNS for reaching the host machine from inside a container
- Works on Docker Desktop (Mac, Windows) and Linux with proper configuration
- Configurable via constant if deployment changes

### Why keep legacy tools?

- Local development without Docker works faster
- Direct testing of tool logic without HTTP overhead
- Graceful fallback if network issues occur

### Why not WebSocket or gRPC?

- HTTP is simpler and sufficient for request/response pattern
- Tools make synchronous calls - no streaming needed
- Easier to debug and test

---

## Open Questions

1. **Authentication**: Should HTTP endpoints require auth token? Currently open.
2. **Rate limiting**: Should we rate-limit tool HTTP calls? Consider if abuse is possible.
3. **Timeouts**: Current 30s timeout may be too long/short for dialectic queries.

---

## Future Work

1. **advance_lesson tool**: Also needs sandbox conversion (Phase 4 in parent plan)
2. **Caching**: Consider caching Honcho responses to reduce latency
3. **Observability**: Add tracing for HTTP tool calls
4. **Error recovery**: Retry logic for transient failures

---

## References

- Parent plan: `thoughts/shared/plans/2026-01-16-ARI-85-poc-course-background-agent.md`
- Current tools: `src/youlab_server/tools/dialectic.py`, `src/youlab_server/tools/memory.py`
- Background runner: `src/youlab_server/background/runner.py`
- Factory: `src/youlab_server/background/factory.py`
