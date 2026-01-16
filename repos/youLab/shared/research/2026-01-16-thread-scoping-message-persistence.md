---
date: 2026-01-16T14:57:18-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Thread Scoping and Message Persistence for POC-Tutor"
tags: [research, codebase, letta, honcho, threading, message-persistence, poc-tutor]
status: complete
last_updated: 2026-01-16
last_updated_by: ariasulin
---

# Research: Thread Scoping and Message Persistence for POC-Tutor

**Date**: 2026-01-16T14:57:18 PST
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

For the poc-tutor agent specifically (but also in general): Is message persistence scoped to thread? If I have two poc-tutor threads, does Letta still see them as one thread?

## Summary

**Yes, Letta sees all threads from the same user as one continuous conversation.** The system uses a **one-agent-per-user-per-course** model. Multiple OpenWebUI chat threads for the same user route to the **same Letta agent**. There is no thread-level separation at the Letta layer.

However, **Honcho message persistence IS per-thread** - each OpenWebUI chat gets its own Honcho session. This means message history is stored per-thread for retrieval purposes, but the Letta agent's context and memory blocks are shared across all threads.

### Key Architecture:

```
OpenWebUI User (user_id: "abc123")
  ├─ Thread 1 (chat_id: "thread-1")  ─┐      Honcho Session: chat_thread-1
  ├─ Thread 2 (chat_id: "thread-2")  ─┼─>    Honcho Session: chat_thread-2
  └─ Thread 3 (chat_id: "thread-3")  ─┘      Honcho Session: chat_thread-3
                                      │
                                      └───> ONE Letta Agent: youlab_abc123_poc-tutor
                                            (shared memory blocks, shared context)
```

## Detailed Findings

### 1. Letta Agent Creation and Routing

**Agent Naming Convention** (`src/youlab_server/server/agents.py:42-44`):
```python
def _agent_name(self, user_id: str, agent_type: str) -> str:
    return f"youlab_{user_id}_{agent_type}"
```

Agents are uniquely identified by `(user_id, agent_type)` tuples. For poc-tutor, the agent name would be `youlab_{user_id}_poc-tutor`.

**Agent Lookup** (`src/youlab_server/server/agents.py:172-188`):
- The system checks a cache first: `(user_id, agent_type) → agent_id`
- If not cached, searches Letta's agent list by name
- **Only ONE agent exists per user per course**

**Pipeline Routing** (`src/youlab_server/pipelines/letta_pipe.py:99-136`):
- `_ensure_agent_exists()` looks for existing agent matching `agent_type`
- Returns existing agent_id or creates new one
- `chat_id` is passed through but **NOT used for routing**, only for persistence

### 2. Honcho Message Persistence (Per-Thread)

**Session ID Generation** (`src/youlab_server/honcho/client.py:117-119`):
```python
def _get_session_id(self, chat_id: str) -> str:
    return f"chat_{chat_id}"
```

Each OpenWebUI chat thread creates a unique Honcho session.

**Persistence Flow** (`src/youlab_server/server/main.py:304-352`):
- User messages persisted with `student_{user_id}` peer
- Agent responses persisted with `"tutor"` peer
- Both use `chat_{chat_id}` session
- Fire-and-forget async pattern (non-blocking)

**Hierarchy**:
```
youlab (workspace)
├── student_{user_id} (peer)
│   ├── chat_{chat_id_1} (session) ← Thread 1 messages
│   ├── chat_{chat_id_2} (session) ← Thread 2 messages
│   └── chat_{chat_id_3} (session) ← Thread 3 messages
└── tutor (peer)
    └── [responses in all sessions]
```

### 3. Dialectic Queries (Currently All Sessions)

**Query Implementation** (`src/youlab_server/honcho/client.py:240-307`):
```python
async def query_dialectic(
    self,
    user_id: str,
    question: str,
    session_scope: SessionScope = SessionScope.ALL,
    ...
):
    peer = self.client.peer(self._get_student_peer_id(user_id))
    response = peer.chat(question)  # Queries ALL sessions
```

**Important**: Only `SessionScope.ALL` is currently implemented. When the agent calls `query_honcho`, it gets insights from **all conversation history** across all threads, not just the current thread.

The `session_scope` parameter exists but session-level filtering is noted as "reserved for future implementation" in comments.

### 4. Memory Blocks (Per-User, Not Per-Thread)

**Storage Structure** (`src/youlab_server/storage/git.py:89-108`):
```
.data/users/{user_id}/
    memory-blocks/           # Shared across all threads
        student.md
        course_syllabus.md
        operating_manual.md
        progress.md
```

Memory blocks are **user-scoped**, not thread-scoped:
- All poc-tutor threads share the same `progress`, `student`, etc. blocks
- When agent calls `edit_memory_block`, it affects ALL threads
- Letta block naming: `youlab_user_{user_id}_{label}`

### 5. What This Means in Practice

**Scenario**: User has two poc-tutor threads - one for homework help, one for exam prep.

| Aspect | Behavior |
|--------|----------|
| **Letta Agent** | Same agent handles both threads |
| **Agent Memory** | Shared - updates in Thread 1 visible in Thread 2 |
| **Conversation Context** | Agent sees combined context from both threads |
| **Honcho Storage** | Separate - messages stored per-thread |
| **query_honcho Results** | Combined - returns insights from ALL threads |
| **"Fresh Start"** | Not possible at Letta level without new user |

## Code References

- `src/youlab_server/server/agents.py:42-44` - Agent naming convention
- `src/youlab_server/server/agents.py:172-188` - Agent lookup logic
- `src/youlab_server/server/agents.py:252-260` - Agent reuse (no duplicates)
- `src/youlab_server/pipelines/letta_pipe.py:99-136` - Pipeline agent lookup
- `src/youlab_server/pipelines/letta_pipe.py:188-199` - Message sending (chat_id passed but not used for routing)
- `src/youlab_server/honcho/client.py:117-119` - Session ID generation
- `src/youlab_server/honcho/client.py:121-220` - Message persistence implementation
- `src/youlab_server/honcho/client.py:240-307` - Dialectic query (ALL sessions only)
- `src/youlab_server/storage/blocks.py:41-43` - Block naming (user-scoped)
- `config/courses/poc-tutor/course.toml:35-63` - Block definitions (all `shared=false`)

## Architecture Documentation

### Design Rationale (Inferred)

The one-agent-per-user-per-course model appears designed for tutoring use cases where:
- The tutor should remember the student across all sessions
- Learning progress should accumulate over time
- The agent builds a "theory of mind" about the student

Thread-level isolation would lose these benefits.

### Current Limitations

1. **No thread isolation** - Cannot have "fresh" conversations
2. **Dialectic queries aggregate** - Cannot ask "what did we discuss in THIS thread?"
3. **Memory blocks global** - Cannot have thread-specific progress tracking

## Open Questions

1. **Is thread-level agent isolation ever needed?** (e.g., for different topics/contexts)
2. **Should session_scope filtering be implemented in query_honcho?**
3. **Would users benefit from a "new conversation" feature that creates a fresh agent?**
