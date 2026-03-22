---
date: 2026-01-28T18:18:49Z
researcher: Ariav Asulin
git_commit: f08f800a4483a352de6b938a253057d1b6351264
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "Ralph Wiggum Architecture and Agno Pipe Documentation"
tags: [research, codebase, ralph, agno, pipe, openwebui, architecture]
status: complete
last_updated: 2026-01-28
last_updated_by: Ariav Asulin
---

# Research: Ralph Wiggum Architecture and Agno Pipe Documentation

**Date**: 2026-01-28T18:18:49Z
**Researcher**: Ariav Asulin
**Git Commit**: f08f800a4483a352de6b938a253057d1b6351264
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

Review the updated Agno pipe and structure & make sure the CLAUDE.md is entirely accurate and up to date with the new direction of this project. Document the current state of the refactor aka the 'ralph' flow including the new pipe, server, etc.

## Summary

The YouLab project has two parallel implementations:

1. **Legacy Stack** (`src/youlab_server/`) - A complex Letta-based system with curriculum configs, memory blocks, background agents, and git-based storage. Currently 100% Letta with zero Agno code.

2. **Ralph MVP** (`ralph/`) - A new greenfield implementation using Agno for agent creation. This provides a "Claude Code-like" experience with workspace-scoped file and shell tools. MVP phases 1-5 are complete and passing lint/typecheck.

The CLAUDE.md has been updated to accurately reflect this dual architecture and the current state of the Ralph flow.

## Detailed Findings

### Ralph Architecture

Ralph is a lightweight alternative to the legacy YouLab stack, designed to provide a "Claude Code-like" experience without the complexity of curriculum configs, memory blocks, or background agents.

#### Request Flow

```
OpenWebUI Chat
     ↓
Ralph Pipe (HTTP Client)
     ↓ POST /chat/stream with user_id, chat_id, messages[]
Ralph Server (FastAPI on port 8200)
     ↓
Agno Agent Creation
  - OpenRouter model (claude-sonnet-4-20250514)
  - FileTools (scoped to workspace)
  - ShellTools (scoped to workspace)
  - Instructions from CLAUDE.md (if present)
     ↓
Streaming Response
  - SSE events: status, message, done, error
     ↓
Pipe forwards to OpenWebUI
```

#### Component Details

**1. Ralph Pipe (`ralph/ralph/pipe.py:18-166`)**

The pipe is a lightweight HTTP client that forwards requests to the Ralph backend:

- **Valves Configuration**: `RALPH_SERVICE_URL` (default: `http://host.docker.internal:8200`)
- **Request Handling**: Extracts `user_id` from `__user__`, `chat_id` from `__metadata__`, and full message history
- **SSE Streaming**: Uses `httpx-sse` to stream events from backend
- **Event Types**: status (processing updates), message (content), done (completion), error

Key implementation (pipe.py:85-103):
```python
async with aconnect_sse(
    client,
    "POST",
    f"{self.valves.RALPH_SERVICE_URL}/chat/stream",
    json={
        "user_id": user_id,
        "chat_id": chat_id,
        "messages": [
            {"role": m.get("role", "user"), "content": m.get("content", "")}
            for m in messages
        ],
    },
) as event_source:
```

**2. Ralph Server (`ralph/ralph/server.py:1-202`)**

FastAPI backend that creates Agno agents per-request:

- **Endpoint**: `POST /chat/stream` - Streaming chat via SSE
- **Endpoint**: `GET /health` - Health check
- **Workspace Modes**:
  - Shared: `RALPH_AGENT_WORKSPACE` env var points to a shared codebase
  - Per-user: `{user_data_dir}/{user_id}/workspace/` created automatically
- **CLAUDE.md Integration**: Reads project instructions from workspace CLAUDE.md if present

Key Agno agent creation (server.py:138-149):
```python
agent = Agent(
    model=OpenRouter(
        id=settings.openrouter_model,
        api_key=settings.openrouter_api_key,
    ),
    tools=[
        strip_agno_fields(ShellTools(base_dir=workspace)),
        strip_agno_fields(FileTools(base_dir=workspace)),
    ],
    instructions=instructions,
    markdown=True,
)
```

**3. Agno Tool Compatibility (`server.py:56-62`)**

Agno toolkits contain fields that some models (Mistral) don't accept:

```python
def strip_agno_fields(toolkit: Toolkit) -> Toolkit:
    """Strip Agno-specific fields that Mistral doesn't accept."""
    for func in toolkit.functions.values():
        func.requires_confirmation = None
        func.external_execution = None
    return toolkit
```

**4. Message History Context (`server.py:151-174`)**

Multi-turn conversation history is flattened into a single prompt:

- Single message: Use content directly
- Multiple messages: Format as `"User: ... Assistant: ..."` with current message separated

**5. Honcho Client (`ralph/ralph/honcho.py:1-166`)**

Message persistence and dialectic queries for student insights:

- **Architecture**: One workspace, one peer per student (`student_{user_id}`), one tutor peer, one session per chat
- **Lazy Loading**: Client initialized only when first used, graceful degradation if unavailable
- **Methods**:
  - `persist_message()` - Store messages with metadata
  - `query_dialectic()` - Query Honcho for student insights
  - `persist_message_fire_and_forget()` - Non-blocking persistence

**6. Configuration (`ralph/ralph/config.py:1-46`)**

Pydantic settings with `RALPH_` prefix:

```python
class Settings(BaseSettings):
    openrouter_api_key: str = ""
    openrouter_model: str = "anthropic/claude-sonnet-4-20250514"
    honcho_workspace_id: str = "ralph"
    honcho_environment: str = "demo"
    honcho_api_key: str | None = None
    user_data_dir: str = "/data/ralph/users"
    agent_workspace: str | None = None  # Shared workspace if set
    model_config = {"env_prefix": "RALPH_"}
```

### OpenHands Client (Being Removed)

The file `ralph/ralph/openhands_client.py` exists but is being phased out. It was part of an earlier design that used OpenHands SDK for sandboxed execution. The project direction is now Agno-only with local FileTools/ShellTools.

### Ralph Autonomous Loop Script (`ralph/ralph.sh`)

A bash script that runs Claude Code iteratively to implement the plan:

1. Reads implementation plan from `thoughts/shared/plans/2026-01-26-ralph-wiggum-mvp.md`
2. Executes Claude Code with `CLAUDE.md` instructions
3. Monitors for `<promise>COMPLETE</promise>` signal
4. Archives progress on branch changes
5. Consolidates learnings in `progress.txt`

### Legacy Stack Analysis

The `src/youlab_server/` directory contains 100% Letta code with zero Agno:

- **302 occurrences** of "letta/Letta" across 32 files
- **0 occurrences** of "agno/Agno"
- All agent operations use `letta_client.Letta`

Key legacy components:
- `agents/` - Deprecated BaseAgent wrapper around Letta
- `memory/` - PersonaBlock/HumanBlock classes (deprecated)
- `server/agents.py` - AgentManager for Letta agent lifecycle
- `storage/` - Git-based user storage with PendingDiff approval workflow
- `tools/sandbox.py` - HTTP-based tools for Letta Docker sandbox
- `curriculum/` - TOML-based course configuration system

The CLAUDE.md previously described only this legacy stack. It has been updated to accurately reflect both implementations.

### Project Dependencies

From `pyproject.toml`:

**Main dependencies**: letta, pydantic, fastapi, uvicorn, langfuse, honcho-ai, gitpython

**Ralph optional dependencies** (`[project.optional-dependencies.ralph]`):
- fastapi, uvicorn, sse-starlette, httpx, pydantic-settings, structlog
- Note: `agno` is not listed (imported at runtime from installed package)

**Entry points**:
- `ralph-server = "ralph.server:main"` - Starts Ralph backend on port 8200
- `youlab-server = "youlab_server.server.cli:main"` - Legacy Letta server

### Implementation Status

From `ralph/progress.txt`, all MVP phases are complete:

- **Phase 1**: Project Setup (config, package structure)
- **Phase 2**: Honcho Integration (persistence, dialectic queries)
- **Phase 3 & 4**: OpenHands Client and Query Honcho Tool
- **Phase 5**: OpenWebUI Pipe (streaming, message handling)
- **Phase 6**: Docker Sandbox (optional, not implemented)

## Code References

- Ralph Pipe: `ralph/ralph/pipe.py:18-166`
- Ralph Server: `ralph/ralph/server.py:1-202`
- Agno Agent Creation: `ralph/ralph/server.py:138-149`
- Strip Agno Fields: `ralph/ralph/server.py:56-62`
- Workspace Path: `ralph/ralph/server.py:35-45`
- Honcho Client: `ralph/ralph/honcho.py:39-144`
- Config Settings: `ralph/ralph/config.py:10-35`
- Legacy AgentManager: `src/youlab_server/server/agents.py:19-544`
- Legacy Letta Pipe: `src/youlab_server/pipelines/letta_pipe.py:19-272`

## Architecture Documentation

### Ralph Stack
```
OpenWebUI → Ralph Pipe (HTTP) → Ralph Server (FastAPI:8200) → Agno Agent → OpenRouter
                                        ↓
                                  FileTools + ShellTools
                                        ↓
                                  User Workspace
```

### Legacy Stack
```
OpenWebUI → Letta Pipe (HTTP) → YouLab Server (FastAPI:8100) → Letta Server → Claude API
                                        ↓
                                  Honcho (persistence)
                                        ↓
                                  GitUserStorage
```

### Key Differences

| Aspect | Ralph (New) | Legacy (Letta) |
|--------|-------------|----------------|
| Agent Framework | Agno | Letta |
| Tool Execution | Local FileTools/ShellTools | Docker sandbox via Letta |
| Memory | None (stateless per chat) | Memory blocks, rotation strategies |
| Storage | Workspace files | Git-backed with diff approval |
| Configuration | Env vars only | TOML curriculum configs |
| Background Agents | None | Scheduled/triggered tasks |

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-26-ralph-wiggum-mvp.md` - Original implementation plan for Ralph MVP
- `thoughts/shared/plans/2026-01-27-ralph-http-backend.md` - Evolution plan adding FastAPI backend (implemented)
- `thoughts/shared/research/2026-01-24-agno-reference.md` - Agno framework reference documentation

## Related Research

- `thoughts/shared/research/2026-01-24-openwebui-api-pipes-reference.md` - OpenWebUI Pipe integration patterns
- `thoughts/shared/research/2026-01-24-honcho-integration-patterns.md` - Honcho client patterns

## Open Questions

1. **Agno Dependency**: The `agno` package is not listed in pyproject.toml ralph dependencies - how is it being installed?
2. **Honcho Integration**: The Honcho client exists in Ralph but doesn't appear to be called from the server - is message persistence implemented?

## Resolved Questions

- **OpenHands**: Being phased out, Agno-only direction confirmed
- **Dolt Migration**: Planned but out of scope for current work
