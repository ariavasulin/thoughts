# ARI-111: Stream Tool Calls and Reasoning — Handoff

## Status: Paused (debugging)

## What Was Done

### Code Changes (committed & deployed to prod)

1. **`src/ralph/server.py`** — Stream loop now inspects `chunk.event`:
   - `ToolCallStarted` → emits `{"type": "tool_call", "name": "...", "status": "started"}`
   - `ToolCallCompleted` → emits `{"type": "tool_call", "name": "...", "status": "completed"}`
   - `ReasoningContentDelta` → emits `{"type": "reasoning", "content": "..."}`
   - Skips `RunCompleted`/`RunStarted`/`RunContentCompleted` (contain duplicate full content)
   - **Debug logging live**: `log.info("stream_chunk", event_type=..., has_content=...)` at line 359

2. **`src/ralph/pipe.py`** — Handles new SSE event types:
   - `tool_call` → OpenWebUI status indicators ("Using Save File...")
   - `reasoning` → collapsible `<details>` block, auto-closes on message/done
   - `_in_reasoning` state tracked per-request (reset at line 86)

3. **`src/ralph/memory.py`** — Block labels in context:
   - `### Origin Story (label: `origin_story`)` so agent knows exact identifiers

### Prod Config Changes
- `RALPH_HONCHO_ENVIRONMENT=production` (was `demo`)
- `RALPH_HONCHO_API_KEY=hch-v2-14guq...` (set)
- `RALPH_OPENROUTER_MODEL=openai/gpt-5.1-chat` (was `deepseek/deepseek-v3.2`)

## What's Broken

### Tool calls never fire
Debug logs on prod show **only `RunContent` events** — no `ToolCallStarted` at all. GPT-5.1 via OpenRouter recites block content from the system prompt instead of calling `list_memory_blocks` / `read_memory_block`.

Tested with explicit prompts like "use the read_memory_block tool to read my origin_story block" — model still just echoes content from system prompt.

### Likely causes (investigate in order)
1. **Agno tool registration with OpenRouter/GPT-5.1** — Check if `strip_agno_fields()` is stripping something GPT-5.1 needs, or if Agno's tool serialization doesn't match OpenAI's function calling format for this model
2. **System prompt too rich** — All 4 block bodies are in the system prompt via `build_memory_context()`. Model has no reason to call tools when it already has the data
3. **OpenRouter tool calling** — Verify OpenRouter actually forwards tool definitions for `openai/gpt-5.1-chat`

### Honcho persistence failing
Every message logs `persist_failed: 'Not Found'`. The production Honcho API key is set but the `ralph` workspace may not exist yet in Honcho production. Non-blocking (fire-and-forget) but should be fixed.

### `user_activity` table missing
`activity_tracking_failed: table not found: user_activity` — Dolt doesn't have this table. Needs a migration or the tracking code should be removed/guarded.

## Commits
- `80dc046` feat: Stream tool calls and reasoning from pipe
- `b488871` fix: Include block labels in memory context and skip duplicate stream events
- `96e3e60` debug: Log all stream event types to diagnose tool call streaming
- `47af658` feat: Add chat message injection endpoint and OpenWebUI client

## Next Steps
1. Add Agno tool debug logging — log the actual tool definitions being sent to the model
2. Try switching model to `anthropic/claude-sonnet-4` to see if tools work there
3. If tools work with Claude, the issue is GPT-5.1/OpenRouter-specific
4. Remove debug `stream_chunk` log line once diagnosed
5. Fix Honcho workspace initialization for production
