---
date: 2026-01-25T02:31:46Z
researcher: ARI
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Letta Tool Patterns for Agno Migration"
tags: [research, letta, tools, agno, migration, sandbox]
status: complete
last_updated: 2026-01-24
last_updated_by: ARI
---

# Research: Letta Tool Patterns for Agno Migration

**Date**: 2026-01-25T02:31:46Z
**Researcher**: ARI
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Document Letta tool patterns for Agno migration. Analyze current implementations in `src/youlab_server/tools/`, sandbox execution patterns, tool definition patterns, result handling, error handling, and agent state integration. Focus on what patterns to preserve vs simplify in Agno.

## Summary

The YouLab codebase implements a dual-versioning tool architecture for Letta: **legacy in-process tools** and **sandbox-compatible HTTP-based tools**. Tools are registered globally at service startup via `client.tools.upsert_from_function()` and attached to agents during creation. Letta automatically injects `agent_state` into tools at runtime for context access.

Key patterns that exist:
1. **Tool Versioning** - Two implementations of each tool (legacy + sandbox)
2. **HTTP-Based Sandbox Tools** - stdlib-only tools that call back to the YouLab server
3. **Tool Rule System** - Registry-based defaults with TOML override capability
4. **agent_state Injection** - Letta's automatic context injection mechanism
5. **Two-Tier Fallback** - Extracting user_id from direct field or nested metadata

## Detailed Findings

### 1. Tool Directory Structure

```
src/youlab_server/tools/
├── __init__.py        # Tool registry, exports, rule system
├── sandbox.py         # HTTP-based tools for Docker sandbox
├── dialectic.py       # Legacy query_honcho (in-process)
├── memory.py          # Legacy edit_memory_block (in-process)
└── curriculum.py      # advance_lesson tool
```

### 2. Tool Definition Pattern

All tools follow a consistent signature pattern:

```python
def tool_function(
    # Tool-specific parameters first
    param1: str,
    param2: str = "default",
    # agent_state always last, optional with None default
    agent_state: dict[str, Any] | None = None,
) -> str:
    """
    Tool docstring with Google-style formatting.

    Args:
        param1: Description of param1
        param2: Description of param2
        agent_state: Injected by Letta - contains agent_id for context lookup

    Returns:
        Result string or error message

    Example:
        tool_function("value", "other")
    """
    # Implementation
    return result_string
```

Key characteristics:
- **Return type is always `str`** - Tools return human-readable messages
- **agent_state is the last parameter** - Always optional with `None` default
- **Docstrings follow Google style** - Letta parses these for tool schemas
- **Examples in docstring** - Help the agent understand usage

**Reference**: `src/youlab_server/tools/sandbox.py:27-53` (query_honcho signature)

### 3. Sandbox Execution Pattern

Sandbox tools use only Python stdlib and make HTTP calls to YouLab server:

```python
# src/youlab_server/tools/sandbox.py:4-12
"""
IMPORTANT: These tools must:
1. Use only Python stdlib (no external dependencies)
2. Make HTTP calls to YouLab server at YOULAB_SERVER_URL
3. Not import any youlab_server modules
4. Be completely self-contained
"""

YOULAB_SERVER_URL = "http://host.docker.internal:8100"
```

HTTP call pattern (`sandbox.py:84-105`):

```python
import json
import urllib.error
import urllib.request

url = f"{YOULAB_SERVER_URL}/honcho/query"
payload = json.dumps({...}).encode("utf-8")
headers = {"Content-Type": "application/json"}

req = urllib.request.Request(url, data=payload, headers=headers, method="POST")
with urllib.request.urlopen(req, timeout=30) as response:
    result = json.loads(response.read().decode("utf-8"))
```

### 4. Tool Registration

Tools are registered globally during FastAPI lifespan (`main.py:92-115`):

```python
# Get Letta client
client = app.state.agent_manager.client

# Unregister old tools to prevent conflicts
for tool_name in ["query_honcho", "edit_memory_block"]:
    with contextlib.suppress(Exception):
        client.tools.delete(name=tool_name)

# Register sandbox-compatible versions
client.tools.upsert_from_function(func=sandbox_query_honcho)
client.tools.upsert_from_function(func=sandbox_edit_memory_block)
client.tools.upsert_from_function(func=advance_lesson)
```

Tools are attached to agents during creation (`agents.py:324-338`):

```python
agent = self.client.agents.create(
    name=agent_name,
    tools=tool_names if tool_names else None,
    tool_rules=tool_rules if tool_rules else None,
    # ... other params
)
```

### 5. agent_state Structure

The `agent_state` dict contains:

**Top-level fields:**
- `agent_id` (str): Letta agent identifier
- `user_id` (str | None): User ID (may be None)
- `metadata` (dict): Agent metadata from creation

**Metadata fields:**
- `youlab_user_id`: User identifier (primary fallback)
- `youlab_agent_type`: Agent type (e.g., "tutor", "background")
- `course_id`: Course identifier
- `course_version`: Course version string
- `youlab_background_task`: Task name (for background agents)

### 6. Context Extraction Pattern

Sandbox tools use a two-tier fallback for user_id (`sandbox.py:59-70`):

```python
if agent_state is None:
    return "Error: No agent state available"

# Try direct field first
user_id = agent_state.get("user_id")
if not user_id:
    # Fall back to metadata
    metadata = agent_state.get("metadata", {})
    user_id = metadata.get("youlab_user_id")

if not user_id:
    return "Error: Could not determine user_id from agent state"
```

### 7. Result Handling Pattern

Tools return descriptive strings on success:

```python
# query_honcho success (sandbox.py:94-96)
insight = result.get("insight")
if insight:
    return insight
return "No insight available from conversation history."

# edit_memory_block success (sandbox.py:189-192)
diff_id = result.get("diff_id", "unknown")
short_id = diff_id[:8] if len(diff_id) > 8 else diff_id
return f"Proposed change to {block}.{field} (ID: {short_id}). User will review."
```

### 8. Error Handling Pattern

Tools catch all exceptions and return error messages (`sandbox.py:98-105`):

```python
except urllib.error.HTTPError as e:
    return f"Server error: {e.code} {e.reason}"
except urllib.error.URLError as e:
    return f"Error connecting to server: {e.reason}"
except json.JSONDecodeError:
    return "Error: Invalid response from server"
except Exception as e:
    return f"Error: {e!s}"
```

Error messages are always strings - tools never raise exceptions to Letta.

### 9. Tool Rule System

The codebase has a registry of default tool rules (`tools/__init__.py:32-70`):

```python
class ToolRule(str, Enum):
    EXIT_LOOP = "exit"
    CONTINUE_LOOP = "continue"
    RUN_FIRST = "first"

TOOL_REGISTRY: dict[str, ToolMetadata] = {
    "send_message": ToolMetadata("send_message", ToolRule.EXIT_LOOP, ...),
    "query_honcho": ToolMetadata("query_honcho", ToolRule.CONTINUE_LOOP, ...),
    "edit_memory_block": ToolMetadata("edit_memory_block", ToolRule.CONTINUE_LOOP, ...),
    "advance_lesson": ToolMetadata("advance_lesson", ToolRule.CONTINUE_LOOP, ...),
}
```

TOML configuration can override defaults:

```toml
# V2 format with inline overrides
[agent]
tools = ["query_honcho", "edit_memory_block", "send_message:exit"]
```

### 10. Server Endpoints for Sandbox Tools

**POST /honcho/query** (`server/honcho.py:38-79`)
- Request: `{user_id, question, session_scope}`
- Response: `{insight, success, error}`
- Queries conversation history via Honcho dialectic

**POST /users/{user_id}/blocks/{label}/propose** (`server/blocks.py:307-360`)
- Request: `{agent_id, field, content, strategy, reasoning}`
- Response: `{diff_id, success, error}`
- Creates pending memory block edit for user approval

## Code References

### Core Tool Files
- `src/youlab_server/tools/__init__.py` - Tool registry and exports
- `src/youlab_server/tools/sandbox.py:27-105` - query_honcho sandbox tool
- `src/youlab_server/tools/sandbox.py:108-201` - edit_memory_block sandbox tool
- `src/youlab_server/tools/curriculum.py:27-84` - advance_lesson tool
- `src/youlab_server/tools/dialectic.py:39-102` - Legacy query_honcho
- `src/youlab_server/tools/memory.py:33-114` - Legacy edit_memory_block

### Tool Registration
- `src/youlab_server/server/main.py:92-115` - Global tool registration at startup
- `src/youlab_server/server/agents.py:324-338` - Agent creation with tools

### Server Endpoints
- `src/youlab_server/server/honcho.py:38-79` - /honcho/query handler
- `src/youlab_server/server/blocks.py:307-360` - /users/{user_id}/blocks/{label}/propose handler

### Configuration
- `src/youlab_server/curriculum/loader.py:192-269` - TOML tool parsing
- `src/youlab_server/curriculum/schema.py:37-73` - ToolConfig schema

## Letta SDK Documentation

From web research, key Letta patterns:

**Tool Definition Methods:**
1. Function with Google-style docstring (used in YouLab)
2. Pydantic model with BaseTool
3. From source file

**agent_state Injection:**
- Letta automatically injects `agent_state` parameter
- Provides access to `agent_state.memory` for block operations
- Alternative: Client injection with `LETTA_AGENT_ID`, `LETTA_API_KEY` env vars

**Tool Registration:**
- `client.tools.create_from_function(func=my_function)`
- `client.tools.upsert_from_function(func=my_function)`
- `client.agents.tools.attach(agent_id, tool_id)`

**Sources:**
- [Define and customize tools | Letta Docs](https://docs.letta.com/guides/agents/custom-tools/)
- [Context engineering | Letta Docs](https://docs.letta.com/guides/agents/context-engineering/)
- [Memory overview | Letta Docs](https://docs.letta.com/guides/agents/memory/)

## Patterns to Consider for Agno Migration

### Patterns to Preserve

1. **Google-style docstrings** - Clear documentation for tool schemas
2. **String return types** - Human-readable results for agent processing
3. **Error messages as strings** - Never raise exceptions to the framework
4. **Two-tier context extraction** - Robust user_id lookup with fallback
5. **Tool rule system** - Control over loop exit behavior

### Patterns That May Simplify

1. **Dual tool versioning** - Agno may not need sandbox isolation
2. **HTTP-based indirection** - Direct function calls if no sandbox
3. **Global client injection** - May use dependency injection instead
4. **YOULAB_SERVER_URL constant** - May not need Docker host workaround
5. **urllib stdlib usage** - Can use httpx/requests in non-sandboxed tools

### Questions for Agno Architecture

1. Does Agno have sandbox execution like Letta?
2. How does Agno inject agent context into tools?
3. What is Agno's equivalent of `agent_state`?
4. Does Agno support tool rules (exit_loop, continue_loop)?
5. How are tools registered and attached to agents in Agno?

## Open Questions

1. **Sandbox requirement** - Does Agno require stdlib-only tools?
2. **Context injection** - What is Agno's mechanism for passing agent context?
3. **Memory integration** - How do Agno tools interact with agent memory?
4. **Tool lifecycle** - How are tools registered, attached, and versioned in Agno?
