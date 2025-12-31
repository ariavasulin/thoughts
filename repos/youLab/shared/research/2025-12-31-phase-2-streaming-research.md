---
date: 2025-12-31T15:08:30-08:00
researcher: ariasulin
git_commit: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
branch: main
repository: YouLab
topic: "Phase 2 Full Streaming with Thinking Implementation Research"
tags: [research, streaming, letta, openwebui, pipes, sse, phase-2]
status: in_progress
last_updated: 2025-12-31
last_updated_by: ariasulin
---

# Research: Phase 2 Full Streaming with Thinking Implementation

**Date**: 2025-12-31T15:08:30-08:00
**Researcher**: ariasulin
**Git Commit**: 9edb6a07380fd0c3e5cf1b011ef6199a225d96e4
**Branch**: main
**Repository**: YouLab

## Research Question

How to implement full streaming with thinking from OpenWebUI Pipe → LettaStarter HTTP Service → Letta agent, including native OpenWebUI dev setup?

## Summary

Phase 2 requires implementing streaming responses so students don't see 10-30 second delays. The goal is:
1. OpenWebUI running locally (prefer native dev server over Docker)
2. Pipe forwarding to LettaStarter HTTP service
3. HTTP service streaming from Letta with thinking/reasoning visible
4. Student gets real-time response with "thinking" indicators

**Key finding**: All three layers support streaming, but they need to be wired together correctly.

---

## Detailed Findings

### 1. Letta SDK Streaming API

**Location**: `.venv/lib/python3.12/site-packages/letta_client/resources/agents/messages.py`

The Letta SDK has two approaches for streaming:

```python
# Approach 1: create() with streaming=True
stream = client.agents.messages.create(
    agent_id=agent_id,
    input="Hello",
    streaming=True,           # Enable SSE
    stream_tokens=True,       # Token-by-token streaming
    enable_thinking=True,     # Include reasoning in stream
    include_pings=True,       # Keep-alive for long responses
)

# Approach 2: Explicit stream() method
stream = client.agents.messages.stream(
    agent_id=agent_id,
    input="Hello",
    stream_tokens=True,
    enable_thinking=True,
)

# Iterate over stream
for chunk in stream:
    print(chunk.message_type, chunk)
```

**Stream returns**: `Stream[LettaStreamingResponse]` which is a union type.

#### LettaStreamingResponse Types

**File**: `.venv/lib/python3.12/site-packages/letta_client/types/agents/letta_streaming_response.py`

```python
LettaStreamingResponse = Union[
    SystemMessage,           # message_type: "system_message"
    UserMessage,             # message_type: "user_message"
    ReasoningMessage,        # message_type: "reasoning_message" <- THINKING
    HiddenReasoningMessage,  # message_type: "hidden_reasoning_message"
    ToolCallMessage,         # message_type: "tool_call_message" <- TOOL CALLS
    ToolReturnMessage,       # message_type: "tool_return_message"
    AssistantMessage,        # message_type: "assistant_message" <- FINAL RESPONSE
    ApprovalRequestMessage,
    ApprovalResponseMessage,
    LettaPing,               # message_type: "ping" <- KEEP-ALIVE
    LettaErrorMessage,       # message_type: "error_message"
    LettaStopReason,         # message_type: "stop_reason"
    LettaUsageStatistics,    # message_type: "usage_statistics"
]
```

**Key message types for "thinking"**:

| Type | Purpose | Key Field |
|------|---------|-----------|
| `ReasoningMessage` | Agent's internal reasoning | `reasoning: str` |
| `ToolCallMessage` | Tool being called | `tool_call.name`, `tool_call.arguments` |
| `AssistantMessage` | Final response to user | `content: str` |
| `LettaPing` | Keep-alive | (no content) |

#### ReasoningMessage Structure
```python
class ReasoningMessage(BaseModel):
    id: str
    date: datetime
    reasoning: str           # The actual thinking text
    message_type: "reasoning_message"
    source: "reasoner_model" | "non_reasoner_model"
```

#### AssistantMessage Structure
```python
class AssistantMessage(BaseModel):
    id: str
    date: datetime
    content: Union[str, List[ContentPart]]  # The response text
    message_type: "assistant_message"
```

---

### 2. Current LettaStarter Implementation (Non-Streaming)

#### HTTP Service Chat Endpoint
**File**: `src/letta_starter/server/main.py:149-204`

```python
@app.post("/chat")
async def chat(request: ChatRequest) -> ChatResponse:
    # ... agent lookup ...
    response_text = manager.send_message(request.agent_id, request.message)
    return ChatResponse(response=response_text, agent_id=request.agent_id)
```

**Issue**: `send_message()` is synchronous and waits for complete response.

#### AgentManager.send_message()
**File**: `src/letta_starter/server/agents.py:158-165`

```python
def send_message(self, agent_id: str, message: str) -> str:
    response = self.client.send_message(
        agent_id=agent_id,
        message=message,
        role="user",
    )
    return self._extract_response(response)
```

**Issue**: Uses `send_message()` not `agents.messages.stream()`.

#### Pipe Implementation
**File**: `src/letta_starter/pipelines/letta_pipe.py:121-188`

```python
def pipe(self, user_message, ...) -> str | Generator[str, None, None] | Iterator[str]:
    # Currently returns str only
    response = client.post(f"{url}/chat", json={...})
    return response.json().get("response", "No response from tutor.")
```

**Issue**: Returns full string, not a generator. Uses sync httpx.Client.

---

### 3. OpenWebUI Pipe Streaming Patterns

**Reference**: `thoughts/global/shared/reference/open-web-ui-pipes.md`

#### Generator Return for Streaming
```python
def pipe(self, user_message, ...) -> Generator[str, None, None]:
    for chunk in external_stream:
        yield chunk
```

#### Async with Event Emitter
```python
async def pipe(
    self,
    body: dict,
    __user__: dict,
    __metadata__: dict,
    __event_emitter__: Callable[[dict], Awaitable[None]] = None,
) -> str:
    # Emit status updates
    if __event_emitter__:
        await __event_emitter__({
            "type": "status",
            "data": {"description": "Thinking...", "done": False}
        })

    # Stream message content
    for chunk in stream:
        await __event_emitter__({
            "type": "message",
            "data": {"content": chunk}
        })

    await __event_emitter__({
        "type": "status",
        "data": {"description": "Complete", "done": True}
    })

    return ""  # Empty since we used emitter
```

#### Event Types
| Type | Purpose | Data Structure |
|------|---------|---------------|
| `status` | Show status indicator | `{"description": "...", "done": bool}` |
| `message` | Stream content | `{"content": "..."}` |

---

### 4. OpenWebUI Native Development Setup

#### Requirements
- Python 3.11+
- Node.js 20+ (for frontend)
- Git

#### Backend Setup
```bash
# Clone
git clone https://github.com/open-webui/open-webui.git
cd open-webui

# Backend
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Run with hot reload
uvicorn main:app --reload --host 0.0.0.0 --port 8080
```

#### Frontend Setup (Separate Terminal)
```bash
cd frontend
npm install  # or bun install
npm run dev  # Runs on :5173 with HMR
```

#### Key Environment Variables
```bash
# .env in backend/
WEBUI_AUTH=true
ENABLE_SIGNUP=true
DATABASE_URL=sqlite:///./data/webui.db
# Leave OLLAMA_BASE_URL empty if not using Ollama
```

#### Pipe Installation (Native Dev)
1. Place pipe file in `backend/data/pipelines/`
2. Or upload via Admin Panel > Functions > Pipes
3. **No hot reload** - must restart or re-upload

#### Docker Alternative (If Native Fails)
```bash
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main
```

---

### 5. Architecture for Streaming

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenWebUI Browser                            │
│  (Receives streamed chunks, shows "thinking" indicator)             │
└────────────────────────────────▲────────────────────────────────────┘
                                 │ SSE/WebSocket
                                 │
┌────────────────────────────────┴────────────────────────────────────┐
│                         OpenWebUI Backend                            │
│  (Handles Pipe execution, forwards stream to browser)               │
└────────────────────────────────▲────────────────────────────────────┘
                                 │ Generator/yield or __event_emitter__
                                 │
┌────────────────────────────────┴────────────────────────────────────┐
│                      letta_pipe.py (Pipe)                           │
│  async def pipe(..., __event_emitter__):                            │
│      async for chunk in stream_from_service():                      │
│          await __event_emitter__({"type": "message", ...})          │
└────────────────────────────────▲────────────────────────────────────┘
                                 │ SSE (httpx async streaming)
                                 │
┌────────────────────────────────┴────────────────────────────────────┐
│                 LettaStarter HTTP Service (:8100)                   │
│  POST /chat/stream → StreamingResponse(event_generator())           │
│  Proxies SSE from Letta, maps message types to UI events            │
└────────────────────────────────▲────────────────────────────────────┘
                                 │ SSE from Letta SDK
                                 │
┌────────────────────────────────┴────────────────────────────────────┐
│                      Letta Server (:8283)                           │
│  client.agents.messages.stream(streaming=True, stream_tokens=True)  │
│  Returns: ReasoningMessage → ToolCallMessage → AssistantMessage     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 6. Mapping Letta Events to UI

| Letta Message Type | UI Representation | Event Format |
|--------------------|-------------------|--------------|
| `ReasoningMessage` | "Thinking..." status | `{"type": "status", "data": {"description": "Thinking...", "done": false}}` |
| `ToolCallMessage` | "Using tool X..." | `{"type": "status", "data": {"description": "Using memory_search...", "done": false}}` |
| `AssistantMessage` | Streamed text | `{"type": "message", "data": {"content": "Hello!"}}` |
| `LettaPing` | (ignored) | - |
| `LettaStopReason` | Complete | `{"type": "status", "data": {"done": true}}` |

---

## Open Questions / Further Research Needed

### Critical Path
1. **FastAPI SSE Streaming**: How to properly proxy SSE from Letta through FastAPI?
   - Need: `StreamingResponse` with async generator
   - Need: Proper content-type headers for SSE

2. **Pipe Async Streaming**: Does OpenWebUI properly handle async pipes with `__event_emitter__`?
   - Current pipe is sync, needs to become async
   - Need to verify event format is correct

3. **httpx Async Streaming**: How to stream SSE with httpx in the pipe?
   - Current: sync `httpx.Client`
   - Need: `httpx.AsyncClient` with streaming

### Nice to Have
4. **Token-level streaming**: Does `stream_tokens=True` give word-by-word streaming?
5. **Error handling**: What happens if Letta stream errors mid-response?
6. **Timeout handling**: Long-running responses with `include_pings=True`

---

## Code References

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| HTTP Chat Endpoint | `src/letta_starter/server/main.py` | 149-204 | Current non-streaming endpoint |
| AgentManager.send_message | `src/letta_starter/server/agents.py` | 158-165 | Sync message sending |
| Pipe | `src/letta_starter/pipelines/letta_pipe.py` | 121-188 | Sync forwarding to HTTP |
| Letta messages.stream() | `.venv/.../letta_client/resources/agents/messages.py` | 691-795 | SDK streaming method |
| LettaStreamingResponse | `.venv/.../letta_client/types/agents/letta_streaming_response.py` | 108-125 | Response union type |
| ReasoningMessage | `.venv/.../letta_client/types/agents/reasoning_message.py` | 12-51 | Thinking content |
| AssistantMessage | `.venv/.../letta_client/types/agents/assistant_message.py` | 13-49 | Response content |

---

## Historical Context

| Document | Path | Relevance |
|----------|------|-----------|
| Phase 2 Consolidated Reference | `thoughts/shared/research/2025-12-31-phase-2-consolidated-reference.md` | Full Phase 2-4 context |
| OpenWebUI Pipes Reference | `thoughts/global/shared/reference/open-web-ui-pipes.md` | Pipe interface details |
| Phase 1 Plan | `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` | Current implementation |

---

## Handoff Notes

### To Continue This Research

1. **FastAPI SSE Streaming**: Research `StreamingResponse` with async generators
   ```python
   from fastapi.responses import StreamingResponse

   async def event_generator():
       async for chunk in letta_stream:
           yield f"data: {json.dumps(chunk)}\n\n"

   return StreamingResponse(event_generator(), media_type="text/event-stream")
   ```

2. **Pipe Async Conversion**: The current pipe needs to become async:
   - Change `def pipe` → `async def pipe`
   - Add `__event_emitter__` parameter
   - Use `httpx.AsyncClient` for streaming

3. **Test Streaming Locally**:
   - Start Letta server: `letta server`
   - Test streaming with curl:
     ```bash
     curl -N -X POST http://localhost:8283/v1/agents/{agent_id}/messages/stream \
       -H "Content-Type: application/json" \
       -d '{"input": "Hello", "stream_tokens": true}'
     ```

4. **OpenWebUI Native Setup**: Clone and run to test pipe integration
   - Backend: `uvicorn main:app --reload --port 8080`
   - Frontend: `npm run dev` on port 5173

### Implementation Order Recommendation

1. Add streaming endpoint to HTTP service (`POST /chat/stream`)
2. Test with curl to verify SSE works
3. Update Pipe to async + httpx streaming
4. Test end-to-end with OpenWebUI

### Key Dependencies

- `letta_client` (already installed) - has streaming support
- `httpx` (already installed) - needs async usage
- `fastapi` (already installed) - needs StreamingResponse
- OpenWebUI (needs to clone/run locally)
