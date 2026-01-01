---
date: 2025-12-31T16:20:23-08:00
researcher: ariasulin
git_commit: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
branch: main
repository: YouLab
topic: "Phase 2 Streaming Implementation"
tags: [implementation, streaming, letta, openwebui, sse]
status: complete
last_updated: 2025-12-31
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Phase 2 Streaming Implementation

## Task(s)

**Completed:**
- Phase 1: Added `httpx-sse>=0.4.0,<0.5.0` dependency to `pyproject.toml`
- Phase 2: Added `stream_message()` method to `AgentManager` in `server/agents.py`
- Phase 3: Added `/chat/stream` endpoint to `server/main.py` with `StreamChatRequest` schema
- Phase 4: Rewrote `pipelines/letta_pipe.py` with correct async signature and streaming support
- Fixed Letta SDK API compatibility (resource-based API)
- Updated README with proper `make setup` and `uv sync --all-extras` instructions

**Status:** All implementation phases complete. All 164 tests pass. Ready for manual integration testing.

## Critical References

- Implementation plan: `thoughts/shared/plans/2025-12-31-phase-2-streaming-implementation.md`
- Letta SDK streaming API: `.venv/lib/python3.12/site-packages/letta_client/resources/agents/messages.py:691-795`

## Recent changes

- `pyproject.toml:17` - Added httpx-sse dependency
- `src/letta_starter/server/agents.py:49-72` - Updated to use `client.agents.list()` instead of `client.list_agents()`
- `src/letta_starter/server/agents.py:108-115` - Updated to use `client.agents.create()` with `memory_blocks` parameter
- `src/letta_starter/server/agents.py:130` - Updated to use `client.agents.retrieve()`
- `src/letta_starter/server/agents.py:162-166` - Updated to use `client.agents.messages.create()`
- `src/letta_starter/server/agents.py:168-246` - Added `stream_message()` and `_chunk_to_sse_event()` methods
- `src/letta_starter/server/schemas.py:45-52` - Added `StreamChatRequest` model
- `src/letta_starter/server/main.py:207-232` - Added `/chat/stream` endpoint
- `src/letta_starter/pipelines/letta_pipe.py` - Complete rewrite with async streaming support
- `README.md:37-47,262-280` - Updated installation and development instructions

## Learnings

1. **Letta SDK uses resource-based API**: The `letta_client` SDK does NOT have `client.list_agents()`, `client.create_agent()`, etc. Instead it uses:
   - `client.agents.list()`
   - `client.agents.create(name=..., memory_blocks=[...], metadata=...)`
   - `client.agents.retrieve(agent_id)`
   - `client.agents.messages.create(agent_id, input=message)`
   - `client.agents.messages.stream(agent_id, input=message, ...)`

2. **Memory blocks format changed**: Old API used `memory={"persona": ..., "human": ...}`. New API uses `memory_blocks=[{"label": "persona", "value": ...}, {"label": "human", "value": ...}]`

3. **OpenWebUI Pipe signature**: Correct signature is `async def pipe(self, body, __user__, __metadata__, __event_emitter__)`. The old code had wrong params.

4. **User exposed their OpenAI API key** in the terminal output during testing - they need to rotate it.

## Artifacts

- `src/letta_starter/server/agents.py` - AgentManager with streaming support
- `src/letta_starter/server/schemas.py:45-52` - StreamChatRequest model
- `src/letta_starter/server/main.py:207-232` - /chat/stream endpoint
- `src/letta_starter/pipelines/letta_pipe.py` - Complete async streaming Pipe
- `tests/test_server/conftest.py` - Updated mock fixtures for new SDK API
- `tests/test_server/test_agents.py` - Updated tests for new SDK API
- `tests/test_server/test_endpoints.py:139-166` - Updated mock for list_all_agents test
- `tests/test_pipe.py` - Updated tests for async streaming Pipe
- `README.md` - Updated setup instructions

## Action Items & Next Steps

1. **User must rotate their OpenAI API key** - it was exposed in the conversation

2. **Manual integration testing required** (Phase 5 from plan):
   ```bash
   # Terminal 1: Start Letta
   pip install letta
   export OPENAI_API_KEY=<new-key>
   letta server

   # Terminal 2: Start LettaStarter
   uv run letta-server

   # Terminal 3: Test streaming
   curl -X POST http://localhost:8100/agents \
     -H "Content-Type: application/json" \
     -d '{"user_id": "test-user", "agent_type": "tutor"}'

   # Then test streaming with the agent_id from response
   curl -N -X POST http://localhost:8100/chat/stream \
     -H "Content-Type: application/json" \
     -d '{"agent_id": "<agent_id>", "message": "Hello!"}'
   ```

3. **Upload Pipe to OpenWebUI** after curl testing works:
   - Admin > Functions > Pipes
   - Upload `src/letta_starter/pipelines/letta_pipe.py`
   - Restart OpenWebUI (Pipes don't hot-reload)

4. **Test error scenarios**:
   - Invalid agent_id
   - Letta server down
   - Long response streaming

## Other Notes

- Port 8100 may already be in use from previous runs - kill the old process before starting
- The `enable_thinking` parameter is passed as string `"true"`/`"false"` to the SDK (typed as `str | Omit`)
- OpenWebUI requires Pipes to be async because `__event_emitter__` is an async function
- All verification passes: `make verify-agent` shows 164 tests passing
