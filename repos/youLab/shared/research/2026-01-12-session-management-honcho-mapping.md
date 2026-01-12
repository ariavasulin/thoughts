---
date: 2026-01-12T04:06:00Z
researcher: ariasulin
git_commit: 47f59dd7178a188ca91b17e0b5865ea76ede3ca3
branch: chore/coderabbit-setup
repository: YouLab
topic: "Honcho Sessions ↔ OpenWebUI Threads Mapping Implementation"
tags: [research, honcho, openwebui, sessions, threads, archival-memory, letta]
status: complete
last_updated: 2026-01-12
last_updated_by: ariasulin
---

# Research: Honcho Sessions ↔ OpenWebUI Threads Mapping

**Date**: 2026-01-12T04:06:00Z
**Researcher**: ariasulin
**Git Commit**: 47f59dd7178a188ca91b17e0b5865ea76ede3ca3
**Branch**: chore/coderabbit-setup
**Repository**: YouLab

## Research Question

How to implement 1:1 Honcho sessions ↔ OpenWebUI threads mapping per product spec section 6.4:
- Each OpenWebUI thread/chat should have exactly one Honcho session
- New chat with same module = new Honcho session, same Letta agent
- Previous thread messages should move to Letta archival memory
- Agent keeps full history in archival memory across threads

## Summary

**Good news**: The 1:1 Honcho session ↔ OpenWebUI thread mapping is **already implemented**. Each OpenWebUI `chat_id` maps to a Honcho session ID `chat_{chat_id}`. The missing piece is moving previous thread messages to Letta archival memory on thread switch.

**Key Findings**:

1. **Honcho sessions already 1:1 with threads** - `HonchoClient` creates sessions as `chat_{chat_id}` (`client.py:119`)
2. **No "new chat" hook in OpenWebUI** - Chat creation happens via REST API only, no event callbacks
3. **Thread switch detection exists** - `ThreadContextManager` already detects thread transitions (`thread_context.py:80-142`)
4. **Archival memory patterns available** - `client.agents.passages.insert()` for programmatic insertion

**Implementation path**: Extend `ThreadContextManager` to archive previous thread's messages to Letta when switching threads.

---

## Detailed Findings

### 1. OpenWebUI Thread/Chat Management

#### Database Schema

**Chat Table** (`OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-65`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String (PK) | Chat UUID (generated at creation) |
| `user_id` | String | Owner |
| `title` | Text | Chat title |
| `chat` | JSON | Complete chat data including messages |
| `meta` | JSON | Metadata (tags, etc.) |
| `folder_id` | Text | Folder reference |
| `created_at` | BigInteger | Unix timestamp |
| `updated_at` | BigInteger | Unix timestamp |

**Message Structure** (inside `chat.chat.history`):
```json
{
  "messages": {
    "msg-id-1": {
      "id": "msg-id-1",
      "parentId": null,
      "content": "message text",
      "role": "user",
      "timestamp": 1234567890
    }
  },
  "currentId": "msg-id-N"
}
```

#### Chat Lifecycle APIs

| Endpoint | Method | Purpose | File:Line |
|----------|--------|---------|-----------|
| `/chats/new` | POST | Create new chat | `routers/chats.py:259-268` |
| `/chats/{id}` | POST | Update chat | `routers/chats.py:571-584` |
| `/chats/{id}/archive` | POST | Archive chat | `routers/chats.py:859-882` |
| `/chats/{id}` | DELETE | Delete chat | `routers/chats.py:693-719` |

**Chat Creation Flow**:
1. Frontend calls `createNewChat()` API
2. Backend generates UUID v4 at `chats.py:243`
3. Creates database record with timestamps
4. Returns `ChatResponse` with `id` (chat_id)

**Critical**: No event hooks or callbacks fired during creation. Detection must happen at message time.

### 2. How Chat ID Flows to Pipes

**No dedicated "new chat" hook** - Detection happens when messages are sent.

#### Metadata Flow

```
Frontend → metadata.chat_id in request body
    ↓
Backend → __metadata__ parameter to pipe (functions.py:264)
    ↓
Pipe → __metadata__.get("chat_id") (letta_pipe.py:168)
    ↓
HTTP Service → chat_id in request payload (main.py:239-269)
```

**Key Code Paths**:
- `OpenWebUI/open-webui/backend/open_webui/functions.py:254-267` - Builds `extra_params` with `__chat_id__`
- `src/youlab_server/pipelines/letta_pipe.py:167-169` - Extracts `chat_id` from metadata
- `src/youlab_server/server/main.py:261-269` - Persists to Honcho

### 3. Current Honcho Integration (Already 1:1)

**File**: `src/youlab_server/honcho/client.py`

#### Session Naming Convention

```python
# client.py:119
def _get_session_id(self, chat_id: str) -> str:
    return f"chat_{chat_id}"
```

**Current Peer/Session Model**:
- Student peer: `student_{user_id}` (one per user)
- Tutor peer: `tutor` (shared singleton)
- Session: `chat_{chat_id}` (one per OpenWebUI thread)

This **already implements 1:1 mapping** between Honcho sessions and OpenWebUI threads.

#### Message Persistence Flow

```python
# client.py:121-169 (user message)
def persist_user_message(self, user_id, chat_id, message, ...):
    peer = self.honcho.peer(f"student_{user_id}")
    session = self.honcho.session(f"chat_{chat_id}")  # <-- 1:1 with thread
    session.add_messages([peer.message(content, metadata=metadata)])

# client.py:171-220 (agent message)
def persist_agent_message(self, user_id, chat_id, message, ...):
    session = self.honcho.session(f"chat_{chat_id}")  # <-- Same session
    session.add_messages([self.tutor.message(content, metadata=metadata)])
```

#### Fire-and-Forget Pattern

```python
# client.py:310-363
def create_persist_task(honcho_client, user_id, chat_id, message, is_user, ...):
    asyncio.create_task(_persist())  # Background task, never blocks
```

### 4. Thread Context Detection (Exists But Incomplete)

**File**: `src/letta_starter/server/thread_context.py`

The `ThreadContextManager` already detects thread transitions:

```python
# thread_context.py:80-142
def update_thread(self, agent_id, chat_id, chat_title) -> ThreadContext:
    current = self._current_threads.get(agent_id)

    if not current or current.chat_id != chat_id:
        # Thread changed!
        is_new = not self._thread_history.get(agent_id, {}).get(chat_id)
        is_returning = not is_new

        # Update tracking
        self._current_threads[agent_id] = ThreadState(chat_id, chat_title)
        self._thread_history[agent_id][chat_id] = ThreadState(...)

        return ThreadContext(
            is_new_thread=is_new,
            is_returning=is_returning,
            previous_title=current.chat_title if current else None,
            ...
        )
```

**What's Missing**: When thread switches, previous messages need to move to Letta archival memory.

### 5. Letta Archival Memory Patterns

#### SDK Methods for Programmatic Insertion

**Modern API** (from reference docs):
```python
# Insert passage programmatically
client.agents.passages.insert(
    agent_id=agent_id,
    content="Previous conversation from Thread X",
    tags=["thread_archive", "chat_abc123"]
)

# Search archival memory
results = client.agents.passages.search(
    agent_id=agent_id,
    query="What did we discuss about essays?",
    tags=["thread_archive"]
)
```

**Legacy API** (currently used in codebase):
```python
# src/youlab_server/memory/manager.py:247-250
self.client.insert_archival_memory(
    agent_id=agent_id,
    memory=archival_entry,
)
```

#### Existing Archival Patterns

| Usage | File:Line | Format |
|-------|-----------|--------|
| Context rotation | `manager.py:238-263` | `[ARCHIVED {timestamp}]\n{content}` |
| Task completion | `manager.py:265-282` | `[TASK COMPLETED {timestamp}]\n...` |
| Memory edit audit | `enricher.py:210-254` | `[MEMORY_EDIT {timestamp}]\n...` |
| Strategy docs | `strategy/manager.py:107-135` | `[TAGS: ...]\n{content}` |

### 6. Implementation Strategy for Thread-to-Archival

#### When to Archive

Archive previous thread messages when:
1. `ThreadContextManager.update_thread()` detects thread change (`is_new_thread=True` or `is_returning=True`)
2. Previous thread exists (`current.chat_id` is not None)

#### What to Archive

Retrieve previous thread's messages from Honcho:

```python
# Using Honcho session to get messages
prev_session = honcho.session(f"chat_{previous_chat_id}")
messages = prev_session.get_messages()  # Get all messages from previous thread
```

#### Archive Format

Recommended format for archival entries:

```
[THREAD_ARCHIVE {timestamp}]
Thread: {previous_chat_title or previous_chat_id}
Messages: {message_count}
Date Range: {first_timestamp} - {last_timestamp}

---
User: {message_1}
Assistant: {response_1}
User: {message_2}
...
```

#### Where to Implement

Extend `ThreadContextManager.update_thread()` to call archival:

```python
# Proposed addition to thread_context.py:update_thread()
if thread_changed and current:  # Previous thread exists
    await self._archive_thread_to_letta(
        agent_id=agent_id,
        chat_id=current.chat_id,
        chat_title=current.chat_title,
    )
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OpenWebUI Frontend                           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ POST /chat with metadata.chat_id
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        OpenWebUI Backend                            │
│  - Chat table: id (UUID), chat (JSON with messages)                │
│  - No "new chat" hooks, REST API only                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ __metadata__ with chat_id
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     YouLab Pipe (letta_pipe.py)                     │
│  - Extracts chat_id, chat_title                                     │
│  - Forwards to HTTP service                                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     YouLab HTTP Service                             │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  ThreadContextManager (thread_context.py)                      │  │
│  │  - Tracks current thread per agent                             │  │
│  │  - Detects: new thread, returning, continuation                │  │
│  │  - [TODO] Archive previous thread on switch                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                               │                                     │
│         ┌─────────────────────┴─────────────────────┐               │
│         ▼                                           ▼               │
│  ┌─────────────────────┐                   ┌─────────────────────┐  │
│  │   Honcho Client     │                   │   Letta Client      │  │
│  │                     │                   │                     │  │
│  │  Session per chat:  │                   │  Agent per user:    │  │
│  │  chat_{chat_id}     │                   │  youlab_{user_id}   │  │
│  │                     │                   │                     │  │
│  │  Already 1:1 ✓      │                   │  Archival memory:   │  │
│  │                     │                   │  [TODO] Thread      │  │
│  │                     │                   │  archives go here   │  │
│  └─────────────────────┘                   └─────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Checklist

### Already Done ✓

- [x] Honcho sessions 1:1 with OpenWebUI threads (`client.py:119`)
- [x] Thread detection in `ThreadContextManager` (`thread_context.py:80-142`)
- [x] Message persistence to Honcho fire-and-forget (`client.py:310-363`)
- [x] Archival memory insertion patterns (`memory/manager.py`, `memory/enricher.py`)

### TODO

- [ ] Add method to retrieve previous thread messages from Honcho
- [ ] Add archival insertion on thread switch in `ThreadContextManager`
- [ ] Define thread archive format (timestamp, title, messages)
- [ ] Add tags for thread archives (e.g., `thread_archive`, `chat_{id}`)
- [ ] Test thread switch → archive → new thread flow
- [ ] Consider chunking for long threads (archival may have size limits)

---

## Code References

### OpenWebUI Chat Management
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-65` - Chat table schema
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:241-264` - Chat creation
- `OpenWebUI/open-webui/backend/open_webui/routers/chats.py:259-268` - POST /chats/new endpoint
- `OpenWebUI/open-webui/backend/open_webui/functions.py:254-267` - Metadata passing to pipes

### YouLab Honcho Integration
- `src/youlab_server/honcho/client.py:119` - Session ID generation (`chat_{chat_id}`)
- `src/youlab_server/honcho/client.py:121-169` - User message persistence
- `src/youlab_server/honcho/client.py:171-220` - Agent message persistence
- `src/youlab_server/honcho/client.py:310-363` - Fire-and-forget async pattern

### Thread Context Management
- `src/letta_starter/server/thread_context.py:80-142` - Thread change detection
- `src/letta_starter/server/thread_context.py:144-179` - Thread history loading from Honcho

### Archival Memory Patterns
- `src/youlab_server/memory/manager.py:238-263` - Context rotation to archival
- `src/youlab_server/memory/enricher.py:210-254` - Audit trail to archival
- `thoughts/global/shared/reference/letta-archival-memory.md` - Comprehensive API guide

---

## Historical Context (from thoughts/)

- `thoughts/global/shared/reference/honcho-reference.md` - Honcho SDK comprehensive research
- `thoughts/shared/research/2026-01-06-phase-3-honcho-integration-research.md` - Phase 3 integration plan
- `thoughts/shared/handoffs/general/2026-01-11_14-25-50_thread-scoped-conversations.md` - Thread scoping strategy
- `thoughts/shared/research/2025-01-12-ARI-78-openwebui-database-schema.md` - Database schema deep dive
- `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Product spec section 6.4 (source requirement)

---

## Related Research

- `thoughts/shared/research/2026-01-06-phase-3-honcho-integration-research.md` - Initial Honcho integration
- `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Memory architecture decisions

---

## Open Questions

1. **Message retrieval from Honcho**: Does the SDK support `session.get_messages()` to retrieve all messages from a session? Need to verify.

2. **Archival size limits**: What's the maximum content size for a single archival entry? Long threads may need chunking.

3. **Archive on every switch or only on "new chat"?**: Should returning to an already-archived thread re-archive it? Probably not.

4. **Timing**: Should archival happen synchronously (blocking) or async (fire-and-forget like message persistence)?

5. **Honcho session cleanup**: After archiving to Letta, should the Honcho session be deleted or kept for dialectic queries?
