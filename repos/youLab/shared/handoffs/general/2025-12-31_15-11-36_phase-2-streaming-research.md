---
date: 2025-12-31T15:11:36-08:00
researcher: ariasulin
git_commit: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
branch: main
repository: YouLab
topic: "Phase 2 Full Streaming with Thinking Implementation Research"
tags: [implementation, streaming, letta, openwebui, pipes, phase-2]
status: complete
last_updated: 2025-12-31
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Phase 2 Streaming Research & Planning

## Task(s)

| Task | Status |
|------|--------|
| Research Letta SDK streaming API | Completed |
| Research OpenWebUI Pipe streaming patterns | Completed |
| Research OpenWebUI native dev setup | Completed |
| Document current non-streaming implementation | Completed |
| Create consolidated research document | Completed |
| Create detailed Phase 2 implementation plan | Not Started |

**Goal**: Enable full streaming with thinking from OpenWebUI → LettaStarter HTTP Service → Letta agent, so students see real-time responses with "thinking" indicators instead of 10-30 second delays.

## Critical References

1. **Research Document**: `thoughts/shared/research/2025-12-31-phase-2-streaming-research.md` - Contains all streaming API details, type definitions, and architecture
2. **Phase 2 Consolidated Reference**: `thoughts/shared/research/2025-12-31-phase-2-consolidated-reference.md` - Full Phase 2-4 context
3. **OpenWebUI Pipes Reference**: `thoughts/global/shared/reference/open-web-ui-pipes.md` - Pipe interface and event emitter patterns

## Recent changes

No code changes made. This session was research-only.

## Learnings

### Letta SDK Streaming
- Letta SDK has `client.agents.messages.stream()` that returns `Stream[LettaStreamingResponse]`
- Key params: `streaming=True`, `stream_tokens=True` (token-by-token), `enable_thinking=True` (include reasoning)
- Stream returns discriminated union with `message_type` field:
  - `ReasoningMessage` → Agent thinking (field: `reasoning: str`)
  - `ToolCallMessage` → Tool being called (field: `tool_call.name`)
  - `AssistantMessage` → Final response (field: `content: str`)
  - `LettaPing` → Keep-alive for long responses

### OpenWebUI Pipe Streaming
- Pipes can stream via `__event_emitter__` callback (async required)
- Event format: `{"type": "status"|"message", "data": {...}}`
- Status events show indicators: `{"type": "status", "data": {"description": "Thinking...", "done": false}}`
- Message events stream content: `{"type": "message", "data": {"content": "chunk"}}`

### Current Implementation Gaps
- `src/letta_starter/server/main.py:149-204` - `/chat` endpoint is sync, returns full response
- `src/letta_starter/server/agents.py:158-165` - `send_message()` is sync, uses non-streaming API
- `src/letta_starter/pipelines/letta_pipe.py:121-188` - Pipe is sync, returns string not generator

### OpenWebUI Dev Setup
- **Prefer native over Docker** for faster iteration
- Backend: `uvicorn main:app --reload --port 8080` (Python 3.11+)
- Frontend: `npm run dev` on port 5173 (Node 20+)
- Pipes have NO hot reload - must restart or re-upload
- Unit tests (`tests/test_pipe.py`) are best development approach

## Artifacts

| Type | Path | Description |
|------|------|-------------|
| Research | `thoughts/shared/research/2025-12-31-phase-2-streaming-research.md` | Full streaming research with code examples |
| Reference | `thoughts/shared/research/2025-12-31-phase-2-consolidated-reference.md` | Phase 2-4 consolidated context |
| Reference | `thoughts/global/shared/reference/open-web-ui-pipes.md` | OpenWebUI Pipe interface |

## Action Items & Next Steps

### Immediate (Before Creating Implementation Plan)

1. **Research FastAPI SSE Streaming**
   - How to use `StreamingResponse` with async generators
   - How to proxy SSE from Letta through FastAPI
   - Pattern: `StreamingResponse(event_generator(), media_type="text/event-stream")`

2. **Research httpx Async Streaming**
   - How to stream SSE responses with `httpx.AsyncClient`
   - Pattern for Pipe to stream from HTTP service

3. **Test Letta Streaming with curl**
   ```bash
   curl -N -X POST http://localhost:8283/v1/agents/{agent_id}/messages/stream \
     -H "Content-Type: application/json" \
     -d '{"input": "Hello", "stream_tokens": true}'
   ```

4. **Clone and Run OpenWebUI Locally**
   - Verify native dev server works
   - Test Pipe installation via Admin UI

### Implementation Order (For Plan)

1. Add `POST /chat/stream` endpoint to HTTP service with SSE
2. Test streaming endpoint with curl
3. Convert Pipe to async + `__event_emitter__`
4. Use `httpx.AsyncClient` for streaming in Pipe
5. Test end-to-end with OpenWebUI

### Architecture Target

```
Browser ← SSE ← OpenWebUI ← event_emitter ← Pipe ← SSE ← HTTP Service ← SSE ← Letta
```

## Other Notes

### Letta SDK Location
The Letta client SDK is installed at `.venv/lib/python3.12/site-packages/letta_client/`. Key files:
- `resources/agents/messages.py` - Streaming methods (lines 691-795)
- `types/agents/letta_streaming_response.py` - Response union type
- `types/agents/reasoning_message.py` - Thinking content
- `types/agents/assistant_message.py` - Response content

### Mapping Letta Events to UI
| Letta Type | UI Event | Description |
|------------|----------|-------------|
| `ReasoningMessage` | `status` with "Thinking..." | Show thinking indicator |
| `ToolCallMessage` | `status` with "Using tool..." | Show tool usage |
| `AssistantMessage` | `message` chunks | Stream response text |
| `LettaStopReason` | `status` with `done: true` | Complete |

### User Preferences
- Prefers native OpenWebUI dev setup over Docker
- Wants full streaming with thinking (not just status-only)
- End goal: Student account → Pipe → HTTP Service → Letta agent with real-time streaming
