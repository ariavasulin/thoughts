---
date: 2026-01-20T08:33:02-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Thread-scoped Letta agents and dynamic message-to-archival migration"
tags: [research, codebase, letta, memory, agents, threads]
status: complete
last_updated: 2026-01-20
last_updated_by: ariasulin
---

# Research: Thread-scoped Letta Agents and Dynamic Message-to-Archival Migration

**Date**: 2026-01-20T08:33:02-08:00
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Would it be possible to make Letta agents in the YouLab scope to threads, and move other messages dynamically to archival memory?

## Summary

**Short answer:** Thread-scoped agents are architecturally possible but would require significant changes. Moving messages to archival is NOT directly supported by Letta, but there are workarounds.

### Current State

1. **Agents are user-scoped**: One agent per `(user_id, agent_type)` - shared across ALL OpenWebUI chats
2. **No thread concept in Letta**: Each agent has one perpetual message history
3. **Archival memory exists but for different purposes**: Context rotation, audit trails, RAG knowledge bases

### Feasibility Assessment

| Goal | Feasibility | Approach |
|------|-------------|----------|
| Thread-scoped agents | **Possible** | Change naming: `youlab_{user_id}_{agent_type}_{chat_id}` |
| Move messages to archival | **Not directly** | Use compact/reset + manual archival insert |
| Dynamic archival migration | **Partial** | Letta auto-compacts when context fills; no selective move API |

## Detailed Findings

### Current Agent Scoping Model

**Location**: `src/youlab_server/server/agents.py:42-44`

```python
def _agent_name(self, user_id: str, agent_type: str) -> str:
    """Generate agent name from user_id and type."""
    return f"youlab_{user_id}_{agent_type}"
```

**Implications**:
- Cache key: `(user_id, agent_type)` → maps to single `agent_id`
- Same agent used for all OpenWebUI chats by the same user
- Agent memory (persona, human blocks) persists across all conversations
- Starting a new OpenWebUI chat continues the same Letta conversation history

### How chat_id is Currently Used

**chat_id flows through but doesn't affect agent selection**:

| Component | chat_id Usage |
|-----------|---------------|
| Pipeline (`letta_pipe.py:168`) | Extracted from metadata |
| HTTP Service (`main.py:305-314`) | Passed to Honcho for message persistence |
| Honcho (`honcho/client.py:117`) | Creates session: `chat_{chat_id}` |
| Langfuse (`tracing.py:64`) | Used as `session_id` |
| Agent lookup | **NOT USED** |

### Letta's Message History Model

**Critical insight from Letta docs**: Letta does NOT have threads or sessions.

**Each agent has**:
1. **Recall Memory** - Complete message history, searchable by date/text
2. **Core Memory** - Memory blocks always in context window
3. **Archival Memory** - Semantic vector store (separate from messages)

**Available Letta APIs**:

| API | Purpose |
|-----|---------|
| `client.agents.messages.list()` | Paginated message history retrieval |
| `client.agents.messages.reset()` | Clear ALL messages (no selective delete) |
| `client.agents.passages.insert()` | Insert to archival memory |
| `client.agents.passages.search()` | Semantic search archival |

**Key limitation**: There is NO API to:
- Delete individual messages
- Move messages to archival
- Create threads/sessions within an agent

### Thread-Scoped Agents: Implementation Approach

To scope agents to threads (OpenWebUI chats):

**Option 1: Include chat_id in agent naming**

```python
# Current:
agent_name = f"youlab_{user_id}_{agent_type}"
cache_key = (user_id, agent_type)

# Thread-scoped:
agent_name = f"youlab_{user_id}_{agent_type}_{chat_id}"
cache_key = (user_id, agent_type, chat_id)
```

**Required changes**:
- `AgentManager._agent_name()` - Include chat_id
- `AgentManager._cache` - 3-tuple key
- `AgentManager.get_agent_id()` - Accept chat_id parameter
- Pipeline - Pass chat_id to agent creation
- HTTP endpoints - Thread `create_agent` to take chat_id

**Implications**:
- Each conversation gets fresh agent memory
- No context sharing between chats (good for isolation, bad for continuity)
- More agents to manage (cleanup strategy needed)
- Memory blocks reset per conversation

**Option 2: Use Letta Identities + Tags**

Letta supports identity-agent associations:

```python
# Create identity per conversation
identity = client.identities.create(identifier_key=f"chat-{chat_id}")

# Create agent with identity
agent = client.agents.create(
    name=agent_name,
    identity_ids=[identity.id],
    tags=[user_id, chat_id]
)
```

This provides similar isolation with better querying capabilities.

### Dynamic Message-to-Archival Migration

**What Letta provides**:

1. **Automatic compaction** - When context window fills, Letta summarizes old messages to recall memory. Agent can search with `conversation_search` tool.

2. **Manual archival insert** - You can extract messages and store summaries:
   ```python
   # List messages
   messages = client.agents.messages.list(agent_id, limit=100)

   # Summarize and archive
   summary = summarize(messages)
   client.agents.passages.insert(agent_id, content=summary)

   # Reset message history
   client.agents.messages.reset(agent_id, add_default_initial_messages=True)
   ```

3. **Context rotation** (existing in YouLab) - When human block fills (>80%), content rotates to archival via `insert_archival_memory()`. This is for memory blocks, not messages.

**What's NOT possible**:
- Selectively delete/move individual messages
- Move messages directly to archival (must summarize + insert + reset)
- Maintain searchable raw message archive in archival (only summaries practical)

### Honcho as Alternative for Thread-Scoped History

YouLab already uses Honcho for message persistence with chat_id scoping:

**Architecture** (`honcho/client.py:44-51`):
- Session per chat: `chat_{chat_id}`
- Messages tagged with metadata
- Dialectic queries available via `query_honcho` tool

**Current capability**:
- Honcho already provides per-conversation message history
- Agent can query past conversations via `query_honcho` tool
- Session scope parameters defined (ALL, RECENT, CURRENT) but only ALL implemented

**Potential approach**: Use Letta for short-term context + Honcho for long-term history search:
1. Keep current agent-per-user model
2. Use `reset()` to clear Letta messages periodically
3. Honcho retains full history (already implemented)
4. Agent queries Honcho for historical context via `query_honcho`

## Code References

- `src/youlab_server/server/agents.py:42-44` - Agent naming convention
- `src/youlab_server/server/agents.py:172-188` - Agent lookup/caching
- `src/youlab_server/honcho/client.py:117-119` - Session ID from chat_id
- `src/youlab_server/memory/manager.py:238-264` - Context rotation to archival
- `src/youlab_server/pipelines/letta_pipe.py:168` - chat_id extraction

## Architecture Documentation

### Current Data Flow

```
User Message → OpenWebUI → Pipeline → HTTP Service → Letta Agent
                                                        ↓
                              ┌────────────────────────────────────┐
                              │       Letta Agent State            │
                              │  ┌────────────────────────────────┐│
                              │  │ Core Memory (blocks)           ││
                              │  │ - persona, human, curriculum   ││
                              │  └────────────────────────────────┘│
                              │  ┌────────────────────────────────┐│
                              │  │ Recall Memory (messages)       ││
                              │  │ - All conversation history     ││
                              │  │ - Single perpetual thread      ││
                              │  └────────────────────────────────┘│
                              │  ┌────────────────────────────────┐│
                              │  │ Archival Memory (passages)     ││
                              │  │ - Vector-indexed context       ││
                              │  │ - Rotated human block content  ││
                              │  └────────────────────────────────┘│
                              └────────────────────────────────────┘
                                                        ↓
                              ┌────────────────────────────────────┐
                              │       Honcho (Parallel)            │
                              │  - Session per chat_id             │
                              │  - Full message persistence        │
                              │  - Dialectic query capability      │
                              └────────────────────────────────────┘
```

### Thread-Scoped Approach (If Implemented)

```
User Message → OpenWebUI → Pipeline → HTTP Service → AgentManager
                                                        ↓
                              ┌─────────────────────────────────────┐
                              │    Agent Selection with chat_id    │
                              │                                     │
                              │  cache_key = (user_id, type, chat_id)│
                              │  agent_name = youlab_{u}_{t}_{c}   │
                              │                                     │
                              │  ┌─────────┐  ┌─────────┐          │
                              │  │ Agent 1 │  │ Agent 2 │  ...     │
                              │  │ chat_a  │  │ chat_b  │          │
                              │  │ fresh   │  │ fresh   │          │
                              │  │ memory  │  │ memory  │          │
                              │  └─────────┘  └─────────┘          │
                              └─────────────────────────────────────┘
```

## Historical Context (from thoughts/)

No existing research documents found on this specific topic.

## Related Research

- Letta documentation: https://docs.letta.com/guides/agents/memory/
- Letta multi-user patterns: https://docs.letta.com/guides/agents/multi-user

## Open Questions

1. **Memory sharing between threads**: If agents become thread-scoped, how should user facts/preferences persist across conversations? Options:
   - Shared blocks via Letta's block sharing feature
   - External storage (current git-versioned `.data/users/` approach)
   - Cross-agent archival queries

2. **Agent lifecycle management**: With potentially many agents per user (one per chat), what's the cleanup strategy?
   - TTL-based deletion?
   - Archive-on-chat-close?
   - Lazy cleanup on storage pressure?

3. **Honcho session scope implementation**: The `session_scope` parameter in `query_honcho` is defined but only ALL is implemented. Should this be completed to enable per-conversation queries?

4. **Hybrid approach viability**: Could a hybrid model work?
   - Single agent per user (continuity)
   - Periodic message reset (context management)
   - Honcho for full history (retrieval)
   - Archival for summaries (semantic search)
