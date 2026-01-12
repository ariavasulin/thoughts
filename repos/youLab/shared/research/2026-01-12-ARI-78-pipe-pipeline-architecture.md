---
date: 2026-01-12T03:12:31Z
researcher: Claude
git_commit: 5921af254188eca4151966cebff1e202da4db885
branch: main
repository: YouLab
topic: "YouLab Pipe/Pipeline Architecture for ARI-78"
tags: [research, codebase, pipe, pipeline, openwebui, letta, honcho, background-agents]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude
---

# Research: YouLab Pipe/Pipeline Architecture

**Date**: 2026-01-12T03:12:31Z
**Researcher**: Claude
**Git Commit**: 5921af254188eca4151966cebff1e202da4db885
**Branch**: main
**Repository**: YouLab
**Linear Ticket**: ARI-78

## Research Question

Understand how the current YouLab pipe intercepts messages and where to hook in for new features:
1. How does the Letta pipe work?
2. Where are messages intercepted before/after Letta?
3. How to trigger background agents from the pipe?
4. How does Honcho session management integrate?
5. Where to hook in for memory block updates?

## Summary

The YouLab platform uses a **multi-layer architecture** for message processing:

```
OpenWebUI → Letta Pipe → HTTP Service → AgentManager → Letta Server
                              ↘ Honcho (fire-and-forget persistence)
                              ↘ Background Agents (manual trigger only)
```

**Key findings:**
- The **Letta Pipe** (`letta_pipe.py`) is the OpenWebUI integration point that forwards messages to the YouLab HTTP service
- **Message interception** occurs at both the pipe level (before HTTP) and server level (before/after Letta)
- **Background agents** are configured but only support manual triggering - idle/scheduled triggers are defined but not implemented
- **Honcho sessions** map 1:1 to OpenWebUI chats via `chat_{chat_id}` naming
- **Memory updates** happen via agent-callable tools (`edit_memory_block`, `advance_lesson`) or background agent enrichment

## Detailed Findings

### 1. Letta Pipe Architecture

**Location**: `src/youlab_server/pipelines/letta_pipe.py`

The pipe is an OpenWebUI "Pipe" class that:
1. Receives messages from the OpenWebUI frontend
2. Ensures an agent exists for the user
3. Forwards messages to the YouLab HTTP service
4. Streams SSE responses back to the frontend

#### Pipe Entry Point (lines 138-231)

```python
async def pipe(
    self,
    body: dict[str, Any],
    __user__: dict[str, Any] | None = None,
    __metadata__: dict[str, Any] | None = None,
    __event_emitter__: Callable[[dict[str, Any]], Awaitable[None]] | None = None,
) -> str:
```

**Parameters injected by OpenWebUI:**
- `body`: Contains `messages` array with conversation history
- `__user__`: User info with `id` and `name`
- `__metadata__`: Contains `chat_id` for session tracking
- `__event_emitter__`: Callback for streaming events to frontend

#### Message Flow in Pipe

1. **Extract user message** (line 147-148): Gets last message content from `body["messages"]`
2. **Get chat context** (lines 168-169): Extracts `chat_id` from metadata, retrieves `chat_title` from OpenWebUI DB
3. **Ensure agent exists** (line 176): Calls `GET /agents?user_id=X` then `POST /agents` if needed
4. **Stream to HTTP service** (lines 188-202): POSTs to `/chat/stream` with SSE connection
5. **Forward SSE events** (lines 200-202): Converts server events to OpenWebUI events

#### Lifecycle Hooks (lines 45-55)

```python
async def on_startup(self) -> None      # Called when pipe loads
async def on_shutdown(self) -> None     # Called when pipe unloads
async def on_valves_updated(self) -> None  # Called when config changes
```

**Integration Point for New Features**: These hooks could initialize idle timers or session trackers.

### 2. HTTP Service Message Handling

**Location**: `src/youlab_server/server/main.py`

The HTTP service provides two chat endpoints:

#### POST /chat (lines 239-323) - Synchronous

```python
@app.post("/chat")
async def chat(request: ChatRequest) -> ChatResponse:
```

**Pre-Letta hooks** (lines 246-269):
1. Verify agent exists via `manager.get_agent_info()`
2. Set user context: `set_user_context(agent_id, user_id)` - enables tools to resolve user
3. Persist user message to Honcho (fire-and-forget)

**Letta call** (line 287):
```python
response_text = manager.send_message(request.agent_id, request.message)
```

**Post-Letta hooks** (lines 289-307):
1. Record generation in Langfuse tracing
2. Persist agent response to Honcho (fire-and-forget)

#### POST /chat/stream (lines 326-408) - Streaming

Same pre/post hooks, but wraps the stream to capture full response for persistence:

```python
def stream_with_persistence() -> Iterator[str]:
    full_response: list[str] = []
    for chunk in manager.stream_message(...):
        # Capture message content
        yield chunk
    # Persist complete response after stream ends
    create_persist_task(honcho_client, user_id, chat_id, "".join(full_response), ...)
```

**Integration Points:**
- Line 257: `set_user_context()` - could add session state tracking here
- Lines 260-269: User message persistence - could trigger idle reset here
- Lines 298-307: Agent response persistence - could check for conversation end signals

### 3. Background Agent System

**Locations**:
- `src/youlab_server/server/background.py` - Router and initialization
- `src/youlab_server/background/runner.py` - Execution engine

#### Current State: Manual Triggering Only

The background system is initialized at startup (main.py:92-96):

```python
initialize_background(
    letta_client=app.state.agent_manager.client,
    honcho_client=app.state.honcho_client,
    config_dir=Path("config/courses"),
)
```

**Manual trigger endpoint** (background.py:96-138):
```
POST /background/{agent_id}/run
Body: {"user_ids": ["user123"]}  # Optional, None = all users
```

#### Trigger Configuration (Not Yet Implemented)

The schema defines three trigger types that are **configured but not active**:

```python
class Triggers(BaseModel):
    schedule: str | None = None       # Cron expression - NOT IMPLEMENTED
    manual: bool = True               # API endpoint - IMPLEMENTED
    after_messages: int | None = None # Message count - NOT IMPLEMENTED
    idle: IdleTrigger = ...          # Idle detection - NOT IMPLEMENTED

class IdleTrigger(BaseModel):
    enabled: bool = False
    threshold_minutes: int = 30
    cooldown_minutes: int = 60
```

#### Background Agent Execution Flow

1. **Find config**: Iterates courses to find `BackgroundAgentConfig` matching agent_id
2. **Get users**: Either from request or discovers from Letta agents
3. **Batch process**: For each user in `batch_size` chunks:
   - Execute each configured Honcho dialectic query
   - Apply enrichment to agent memory blocks

**Integration Point for Idle Trigger**: Would need to:
1. Track last message timestamp per user/chat
2. Run periodic check against `threshold_minutes`
3. Call `_runner.run_agent()` for idle users

### 4. Honcho Session Management

**Location**: `src/youlab_server/honcho/client.py`

#### Session Architecture (lines 43-51)

```
Workspace: "youlab"
├── Peer: "student_{user_id}"     # One per student
├── Peer: "tutor"                 # Shared for all agents
└── Session: "chat_{chat_id}"     # One per OpenWebUI chat
```

#### Key Methods

**Session ID generation** (line 117-119):
```python
def _get_session_id(self, chat_id: str) -> str:
    return f"chat_{chat_id}"
```

**Message persistence** (lines 121-220):
- `persist_user_message()`: Student peer → Session
- `persist_agent_message()`: Tutor peer → Session
- Both include metadata: `chat_id`, `agent_type`, `chat_title`

**Dialectic queries** (lines 240-307):
```python
async def query_dialectic(
    self,
    user_id: str,
    question: str,
    session_scope: SessionScope = SessionScope.ALL,  # all, recent, current, specific
) -> DialecticResponse | None:
    peer = self.client.peer(self._get_student_peer_id(user_id))
    response = peer.chat(question)  # Queries all conversation history
```

#### Fire-and-Forget Pattern (lines 310-363)

```python
def create_persist_task(honcho_client, user_id, chat_id, message, is_user, ...):
    task = asyncio.create_task(_persist())
    _background_tasks.add(task)  # Prevent GC
    task.add_done_callback(_background_tasks.discard)
```

**Integration Point for Session Switching**:
- Current: Sessions are created implicitly on first message
- New chat = new `chat_{chat_id}` session automatically
- **Gap**: No explicit "archive previous session" or "close session" mechanism

### 5. Memory Block Updates

Memory can be updated through two paths:

#### Path 1: Agent-Callable Tools

**edit_memory_block** (`src/youlab_server/tools/memory.py:36-123`):
```python
def edit_memory_block(
    block: str,      # "human" or "persona"
    field: str,      # e.g., "context_notes", "facts"
    content: str,    # New content
    strategy: str,   # "append", "replace", "llm_diff"
    agent_state: dict,  # Injected by Letta
) -> str:
```

**advance_lesson** (`src/youlab_server/tools/curriculum.py:27-85`):
- Updates `journey` block with new lesson position
- Adds completed lesson to milestones
- Returns lesson opening message

#### Path 2: Background Agent Enrichment

**MemoryEnricher** (`src/youlab_server/memory/enricher.py`):
```python
result = self.enricher.enrich(
    agent_id=target_agent_id,
    block=query.target_block,
    field=query.target_field,
    content=response.insight,
    strategy=strategy,
    source=f"background:{agent_id}",
)
```

Called by `BackgroundAgentRunner._execute_query()` after Honcho dialectic query.

## Integration Points Summary

| Feature | Hook Location | Current State |
|---------|---------------|---------------|
| **Trigger background on idle** | `main.py` chat endpoints | NOT IMPLEMENTED - need idle tracker |
| **Switch Honcho sessions** | `honcho/client.py` | IMPLICIT - new chat_id = new session |
| **Archive to Letta archival** | `main.py` or background agent | NOT IMPLEMENTED |
| **Module progression** | `tools/curriculum.py` | IMPLEMENTED - agent calls `advance_lesson` |

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenWebUI Frontend                          │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Letta Pipe (letta_pipe.py)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ on_startup() │  │    pipe()    │  │on_shutdown() │              │
│  │  HOOK POINT  │  │  Main entry  │  │  HOOK POINT  │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│         Extract: user_id, chat_id, message                          │
│         Get/Create agent via HTTP                                   │
│         Stream SSE to/from HTTP service                             │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                          HTTP POST /chat/stream
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    HTTP Service (main.py)                           │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ PRE-LETTA HOOKS:                                               ││
│  │   1. set_user_context(agent_id, user_id)  ← Tool context       ││
│  │   2. create_persist_task(user_msg)        ← Honcho             ││
│  │   *** HOOK POINT: Idle timer reset ***                         ││
│  └────────────────────────────────────────────────────────────────┘│
│                              │                                      │
│                              ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ AgentManager.stream_message(agent_id, message)                 ││
│  │   → Letta SDK: client.agents.messages.stream()                 ││
│  │   → Agent may call tools: edit_memory_block, advance_lesson    ││
│  └────────────────────────────────────────────────────────────────┘│
│                              │                                      │
│                              ▼                                      │
│  ┌────────────────────────────────────────────────────────────────┐│
│  │ POST-LETTA HOOKS:                                              ││
│  │   1. create_persist_task(agent_msg)       ← Honcho             ││
│  │   2. trace_generation()                   ← Langfuse           ││
│  │   *** HOOK POINT: Check conversation signals ***               ││
│  └────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
         │                                              │
         ▼                                              ▼
┌─────────────────────┐                    ┌─────────────────────────┐
│   Honcho Client     │                    │   Background System     │
│                     │                    │                         │
│ Session: chat_{id}  │                    │ POST /background/{id}/run│
│ Peer: student_{uid} │◄───────────────────│                         │
│ Peer: tutor         │  dialectic query   │ BackgroundAgentRunner   │
│                     │                    │   → Honcho queries      │
│ persist_user_msg()  │                    │   → MemoryEnricher      │
│ persist_agent_msg() │                    │                         │
│ query_dialectic()   │                    │ *** NOT IMPLEMENTED:    │
└─────────────────────┘                    │   - Idle trigger        │
                                           │   - Scheduled trigger   │
                                           └─────────────────────────┘
```

## Code References

- `src/youlab_server/pipelines/letta_pipe.py:138-231` - Pipe main entry point
- `src/youlab_server/server/main.py:239-323` - Chat endpoint with hooks
- `src/youlab_server/server/main.py:326-408` - Streaming chat endpoint
- `src/youlab_server/server/main.py:92-96` - Background system initialization
- `src/youlab_server/server/background.py:96-138` - Manual trigger endpoint
- `src/youlab_server/background/runner.py:67-133` - Background agent execution
- `src/youlab_server/honcho/client.py:121-220` - Message persistence
- `src/youlab_server/honcho/client.py:240-307` - Dialectic queries
- `src/youlab_server/tools/memory.py:36-123` - edit_memory_block tool
- `src/youlab_server/tools/curriculum.py:27-85` - advance_lesson tool
- `src/youlab_server/curriculum/schema.py:103-118` - Idle trigger config

## Open Questions

1. **Idle detection implementation**: Should idle tracking be at pipe level (OpenWebUI) or service level (HTTP)?
2. **Session archival**: What triggers archiving previous thread messages to Letta archival memory?
3. **Background agent scheduling**: Will we use APScheduler, Celery, or external cron for scheduled triggers?
4. **Chat lifecycle events**: Does OpenWebUI emit events when a chat is created/closed that we could hook into?
