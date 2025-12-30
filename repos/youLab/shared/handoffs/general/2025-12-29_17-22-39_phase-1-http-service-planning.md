---
date: 2025-12-29T17:22:39-08:00
researcher: ariasulin
git_commit: 4515afa25953ab891d7dd78383d4a72a23832dce
branch: main
repository: YouLab
topic: "Phase 1 HTTP Service Implementation Planning"
tags: [implementation, planning, phase-1, http-service, fastapi, openwebui]
status: complete
last_updated: 2025-12-29
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Phase 1 HTTP Service Planning Complete

## Task(s)

| Task | Status |
|------|--------|
| Research OpenWebUI codebase for Pipe interface details | Completed |
| Finalize Phase 1 design decisions | Completed |
| Write detailed Phase 1 implementation plan | Completed |
| Review plan for issues (PersonaBlock field mismatch) | Completed |
| Update CLAUDE.md for consistency | Completed |
| Update original 7-phase plan with resolved decisions | Completed |
| Create TDD test plan for Phase 1 | **Next Step** |

We are working from the 7-phase technical foundation plan, currently focusing on Phase 1 (HTTP Service).

## Critical References

1. **Phase 1 Detailed Plan**: `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` - Complete implementation guide with 4 sub-phases
2. **Original 7-Phase Plan**: `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Updated with resolved decisions
3. **Existing Memory Blocks**: `src/letta_starter/memory/blocks.py` - PersonaBlock/HumanBlock schemas (Pydantic, not dataclass)

## Recent changes

- `CLAUDE.md:77` - Added reference to Phase 1 detailed plan
- `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` - Created entire file (new)
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md:69-115` - Replaced Phase 1 with summary + link to detailed plan
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md:119-186` - Updated Phase 2 (now handled mostly by Phase 1)
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md:285-316` - Added Phase 4 decisions
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md:645-717` - Rewrote Open Questions Summary with resolved items

## Learnings

### OpenWebUI Pipe Interface (Researched from cloned codebase at `/OpenWebUI/`)

1. **User ID access**: `__user__["id"]` - UUID format, injected by OpenWebUI
2. **Chat ID access**: `__metadata__["chat_id"]` - UUID format
3. **Chat title**: NOT passed to Pipes directly. Must query: `Chats.get_chat_by_id(chat_id).title`
4. **Temporary chats**: IDs starting with `local:` are client-side only, not in database
5. **Import path**: `from open_webui.models.chats import Chats`

Key files in OpenWebUI:
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:706-719` - `get_chat_by_id()` method
- `OpenWebUI/open-webui/backend/open_webui/functions.py:254-278` - Pipe parameter injection

### PersonaBlock Schema (Critical Fix)

The plan originally used wrong field names. Actual fields in `blocks.py`:
- `name` (not `identity`)
- `tone` (not `style`)
- `constraints` (not `boundaries`)
- Also has: `role`, `capabilities`, `verbosity`, `expertise`

### Design Decisions Made

| Decision | Choice |
|----------|--------|
| Framework | FastAPI |
| Port | 8100 |
| Auth | Trust localhost (API key designed for later) |
| Agent naming | `youlab_{user_id}_{agent_type}` |
| Agent creation | Explicit `POST /agents` endpoint |
| Entry point | `uv run letta-server` (separate from CLI) |
| Agent templates | Pydantic-based `AgentTemplate` class with registry |

## Artifacts

- `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` - Complete Phase 1 implementation plan
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Updated 7-phase plan with resolved decisions
- `CLAUDE.md` - Updated with Phase 1 plan reference

## Action Items & Next Steps

1. **Create TDD Test Plan** - User wants test-driven development. Create a plan/document defining all test cases for Phase 1 before implementation:
   - Unit tests for AgentTemplate/AgentTemplateRegistry
   - Unit tests for AgentManager
   - API endpoint tests (health, agents, chat)
   - Integration tests for Pipe → Service → Letta flow

2. **Implement Phase 1.1** (Agent Template System) after tests are defined:
   - `src/letta_starter/agents/templates.py`
   - Tests first, then implementation

3. **Implement Phase 1.2** (HTTP Service Core):
   - Dependencies in `pyproject.toml`
   - `src/letta_starter/server/` package
   - Tests first for each component

4. **Implement Phase 1.3** (Updated Pipe):
   - Replace `src/letta_starter/pipelines/letta_pipe.py`

5. **Implement Phase 1.4** (Langfuse Integration):
   - `src/letta_starter/server/tracing.py`

## Other Notes

### Phase 1 Sub-phases (from detailed plan)

1. **Phase 1.1**: Agent Template System - `AgentTemplate`, `AgentTemplateRegistry`, `TUTOR_TEMPLATE`
2. **Phase 1.2**: HTTP Service Core - FastAPI app, endpoints, AgentManager
3. **Phase 1.3**: Updated Pipe - Thin HTTP forwarder
4. **Phase 1.4**: Langfuse Integration - Tracing middleware

### Key Endpoints (Phase 1)

```
POST /agents     - Create agent { user_id, agent_type?, user_name? }
GET /agents      - List agents (optional user_id filter)
POST /chat       - Send message { agent_id, message, chat_id?, chat_title? }
GET /health      - Service health { status, letta_connected, version }
```

### Files to Create (from plan)

| File | Purpose |
|------|---------|
| `src/letta_starter/agents/templates.py` | Agent template system |
| `src/letta_starter/server/__init__.py` | Server package |
| `src/letta_starter/server/schemas.py` | Pydantic request/response models |
| `src/letta_starter/server/agents.py` | AgentManager class |
| `src/letta_starter/server/main.py` | FastAPI app |
| `src/letta_starter/server/cli.py` | Server entry point |
| `src/letta_starter/server/tracing.py` | Langfuse integration |
| `tests/test_server.py` | Server tests |
