---
date: 2026-02-12T11:00:00-08:00
researcher: ariasulin
git_commit: 96e3e603864fccec1742db570463d05491cbf4ec
branch: main
repository: YouLab
topic: "Tool Call Streaming Status in Ralph Pipe"
tags: [research, tool-calls, streaming, pipe, sse, agno, openwebui]
status: complete
last_updated: 2026-02-12
last_updated_by: ariasulin
---

# Research: Tool Call Streaming Status in Ralph Pipe

**Date**: 2026-02-12T11:00:00-08:00
**Researcher**: ariasulin
**Git Commit**: 96e3e60
**Branch**: main
**Repository**: YouLab

## Research Question

What is the current status of tool call display/streaming in the YouLab pipe? Is it working end-to-end?

## Summary

Tool call streaming was implemented on **Feb 11, 2026** across 3 commits in a single session. The implementation spans both the Ralph server (SSE event emission) and the pipe (OpenWebUI event consumption). As of the last commit (`96e3e60`), a **debug log line was added** to diagnose why tool call events may not be showing up — indicating the feature was implemented but **not yet confirmed working end-to-end**. The session ended at the debugging stage.

### Current Status: **Implemented but unverified / debugging in progress**

The code is in place on both sides (server emits tool call SSE events, pipe receives and renders them as status indicators), but the last commit specifically added diagnostic logging to figure out what event types Agno is actually emitting during streaming — suggesting the events may not be arriving as expected.

## Timeline (Feb 11, 2026)

| Time | Commit | Description |
|------|--------|-------------|
| 22:42 | `80dc046` | **feat**: Initial implementation — server emits `ToolCallStarted`/`ToolCallCompleted`/`ReasoningContentDelta` from Agno stream; pipe handles `tool_call` events as status indicators and `reasoning` events as collapsible `<details>` blocks |
| 22:55 | `b488871` | **fix**: Skip `RunCompleted`/`RunStarted`/`RunContentCompleted` events that were causing duplicate content output |
| 22:59 | `96e3e60` | **debug**: Added `log.info("stream_chunk", event_type=event_type, has_content=...)` to diagnose what event types are actually being emitted |

## Detailed Findings

### Server Side (`src/ralph/server.py:347-403`)

The server's SSE streaming loop processes Agno agent events:

1. **Event extraction** (line 358): `event_type = getattr(chunk, "event", None)` — gets the string event type from each Agno `RunOutputEvent`
2. **Debug logging** (line 359): `log.info("stream_chunk", event_type=event_type, has_content=bool(chunk.content))` — added in the last commit to diagnose what's actually flowing
3. **Event filtering** (lines 362-367): Skips `RunCompleted`, `RunStarted`, `RunContentCompleted` to avoid duplicate content
4. **Tool call handling** (lines 369-388): Checks for `ToolCallStarted` and `ToolCallCompleted`, extracts `tool_name` from `chunk.tool.tool_name`, emits SSE events with `{"type": "tool_call", "name": ..., "status": "started"|"completed"}`
5. **Reasoning handling** (lines 389-395): Checks for `ReasoningContentDelta`, extracts `chunk.reasoning_content`, emits as `{"type": "reasoning", "content": ...}`
6. **Content passthrough** (lines 397-403): All events with `.content` get forwarded as `{"type": "message", "content": ...}`

### Pipe Side (`src/ralph/pipe.py:135-210`)

The pipe's `_handle_sse_event()` method processes received SSE events:

1. **Tool call handling** (lines 156-174): On `event_type == "tool_call"`:
   - `started` → emits OpenWebUI `status` event: `"Using {Friendly Tool Name}..."`
   - `completed` → emits OpenWebUI `status` event: `"Tool complete"` with `done: True`
2. **Reasoning handling** (lines 175-189): On `event_type == "reasoning"`:
   - First reasoning chunk opens a `<details><summary>Thinking...</summary>` block
   - Subsequent chunks append reasoning text
   - Tracks state via `self._in_reasoning` flag
3. **Message handling** (lines 190-195): On `event_type == "message"`:
   - Closes any open reasoning block first
   - Forwards content to OpenWebUI
4. **Done handling** (lines 196-200): Closes reasoning block if open, sends completion status

### Agno Event Types Available

Per analysis of the Agno library (`agno.run.agent`), the relevant `RunEvent` enum values are:

- `"ToolCallStarted"` — `ToolCallStartedEvent` with `.tool: ToolExecution`
- `"ToolCallCompleted"` — `ToolCallCompletedEvent` with `.tool: ToolExecution` and `.content`
- `"ToolCallError"` — `ToolCallErrorEvent` (not handled in current code)
- `"ReasoningContentDelta"` — `ReasoningContentDeltaEvent` with `.reasoning_content: str`
- `"RunContent"` — main content deltas with `.content`
- `"RunStarted"`, `"RunCompleted"`, `"RunContentCompleted"` — terminal events (skipped)

The attribute access patterns in the server code (`getattr(chunk, "tool", None)`, `getattr(tool, "tool_name", None)`) match the Agno event dataclass definitions.

### What's Likely Not Working

The debug commit (`96e3e60`) suggests the developer was trying to determine **whether Agno actually emits `ToolCallStarted`/`ToolCallCompleted` events** during streaming. Possible issues:

1. **Model/provider may not trigger tool call events**: The default model is `anthropic/claude-sonnet-4` via OpenRouter. Tool call events depend on the model actually calling tools during the conversation.
2. **Agno's streaming may not emit these events by default**: Some event types may require configuration (e.g., `store_events=True` or specific agent settings).
3. **Events may be emitted but with unexpected structure**: The `getattr` chain (`chunk.tool.tool_name`) may silently fail if `tool` is `None`.
4. **No test with actual tool invocation**: The debug log was added but no evidence of a test run with results logged.

### No Beads Issue Tracking This

There is no open bead specifically tracking tool call streaming. The three open beads are:
- `YouLab-599`: Lazy workspace→OpenWebUI sync via FileTools post-hooks
- `YouLab-16f`: OpenWebUI artifacts not rendering PDF.js viewer
- `YouLab-0hp`: Review Phase 1 & 2 workspace sync integration

## Code References

- `src/ralph/server.py:347-403` — Server-side event streaming loop
- `src/ralph/server.py:359` — Debug log line (most recent change)
- `src/ralph/pipe.py:135-210` — Pipe-side SSE event handler
- `src/ralph/pipe.py:156-174` — Tool call → OpenWebUI status emission
- `src/ralph/pipe.py:175-189` — Reasoning → collapsible details block

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-27-ralph-http-backend.md` — Original Ralph backend plan included tool call streaming as a goal
- `thoughts/shared/research/2026-01-24-agno-reference.md` — Agno framework reference documenting streaming event types
- `thoughts/shared/research/2026-01-24-openwebui-api-pipes-reference.md` — OpenWebUI pipe/SSE event patterns

## Open Questions

1. What event types does Agno actually emit when tools are called? (The debug log at line 359 should reveal this)
2. Does the OpenRouter provider pass through tool call events, or are they only available with direct Anthropic/OpenAI providers?
3. Is `reasoning=True` enabled on the agent? Currently it is not set in `server.py:285-304`, which means `ReasoningContentDelta` events likely never fire.
4. Has the debug log output been reviewed? Server logs from a tool-using conversation would definitively show what events flow through.
