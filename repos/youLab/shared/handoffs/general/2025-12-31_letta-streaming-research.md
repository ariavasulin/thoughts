# Handoff: Letta Streaming Research

**Created**: 2025-12-31
**Purpose**: Research Letta streaming responses, thinking visibility, and tool calling for OpenWebUI integration
**Priority**: High - blocking POC

---

## Context

We have a working HTTP service (`src/letta_starter/server/`) that proxies to Letta, and an OpenWebUI Pipe (`src/letta_starter/pipelines/letta_pipe.py`) that forwards user messages. The goal is a proof of concept where students can talk to their personal Letta agent through OpenWebUI.

**Current flow**:
```
OpenWebUI → Pipe → HTTP Service (port 8100) → Letta Server (port 8283) → Claude API
```

**What we need to verify**: Does streaming work end-to-end? How do Letta's internal "thinking" and tool calls appear to the user?

---

## Research Questions

### 1. Letta Streaming API

**Primary Questions**:
- Does Letta's `send_message` API support streaming responses?
- What is the response format for streaming vs non-streaming?
- How do we enable streaming in the Python SDK (`letta-client`)?

**Investigate**:
```python
# Current non-streaming call in agents.py:
response = self.client.agents.messages.send(
    agent_id=agent_id,
    messages=[{"role": "user", "content": message}]
)

# Does this support streaming?
# What's the streaming equivalent?
```

**Check these resources**:
- Letta docs: https://docs.letta.com
- `letta-client` SDK source/docs
- `thoughts/global/shared/reference/letta-archival-memory.md` (may have hints)

### 2. Letta Thinking/Reasoning

**Primary Questions**:
- Letta agents have internal "thinking" before responding - is this exposed in the API?
- Should thinking be visible to students, or just the final response?
- How does MemGPT-style "inner monologue" work in the current Letta?

**Context from MemGPT paper**:
- Agents have internal monologue (thinking) separate from user-facing messages
- This is used for memory management decisions, tool planning, etc.

**Investigate**:
- What does a full Letta response object look like?
- Are there different message types (thinking vs assistant_message)?
- Current code in `agents.py:167-176` extracts responses - what's being filtered out?

### 3. Tool Calling Visibility

**Primary Questions**:
- When Letta calls tools (archival_memory_search, archival_memory_insert, etc.), how does this appear?
- Should tool calls be visible in OpenWebUI? As status updates? Hidden?
- How long do tool calls take? Does this affect UX?

**Built-in Letta tools**:
- `archival_memory_insert` - Store to long-term memory
- `archival_memory_search` - Search long-term memory
- `conversation_search` - Search conversation history
- `send_message` - Send response to user (this is the actual output)

**Investigate**:
- Response structure when tools are called
- Timing/latency of tool execution
- Whether tool calls should emit status updates in OpenWebUI

### 4. OpenWebUI Streaming Integration

**Primary Questions**:
- Does our current Pipe support streaming responses?
- How does OpenWebUI expect streaming data? (SSE? Async generator?)
- What's the `__event_emitter__` for in the Pipe signature?

**Current Pipe code** (`letta_pipe.py`):
```python
async def pipe(self, body: dict, __user__: dict, __metadata__: dict, ...) -> str:
    # Currently returns a string
    # Does it need to yield chunks for streaming?
```

**Check**:
- OpenWebUI Pipe streaming examples
- `thoughts/global/shared/reference/open-web-ui-pipes.md` for streaming patterns
- Whether HTTP service needs to stream or if Pipe can buffer

### 5. HTTP Service Streaming

**Primary Questions**:
- Does our FastAPI `/chat` endpoint support streaming?
- Should it use `StreamingResponse`?
- How to propagate Letta streaming through FastAPI?

**Current endpoint** (`server/main.py`):
```python
@app.post("/chat")
async def chat(request: ChatRequest) -> ChatResponse:
    # Currently returns full response
    # Needs to stream?
```

---

## Existing Code to Review

| File | Relevance |
|------|-----------|
| `src/letta_starter/server/main.py` | `/chat` endpoint - needs streaming? |
| `src/letta_starter/server/agents.py` | `send_message()` - how to stream from Letta? |
| `src/letta_starter/pipelines/letta_pipe.py` | Pipe - how to stream to OpenWebUI? |

---

## Expected Deliverables

1. **Streaming architecture recommendation**: How should streaming flow through the stack?
2. **Code snippets**: Working examples of streaming from Letta through to OpenWebUI
3. **Thinking visibility recommendation**: Should students see agent thinking? How?
4. **Tool call UX recommendation**: How to handle tool calls in the UI?

---

## Success Criteria

After this research, we should be able to:
- [ ] Stream Letta responses to OpenWebUI (no waiting for full response)
- [ ] Handle Letta thinking appropriately (visible or filtered)
- [ ] Handle tool calls gracefully (status updates or transparent)
- [ ] Have a clear implementation path for the POC

---

## Notes for Researcher

- The Letta SDK is `letta-client` (Python), installed via `pip install letta-client`
- Letta server runs on port 8283 by default
- We're using Claude as the underlying LLM
- Focus on practical implementation - we need this working for a POC
