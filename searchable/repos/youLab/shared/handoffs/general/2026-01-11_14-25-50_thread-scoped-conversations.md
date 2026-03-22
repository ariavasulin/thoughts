---
date: 2026-01-11T14:25:50+07:00
researcher: ariasulin
git_commit: b30f58c47d3e40dab378d14038bab8be9deb5cf4
branch: main
repository: YouLab
topic: "Thread-Scoped Conversations for Letta Agents"
tags: [implementation, strategy, letta, honcho, threads, openwebui]
status: planning
last_updated: 2026-01-11
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Thread-Scoped Conversations for Letta Agents

## Task(s)

| Task | Status |
|------|--------|
| Research current thread handling architecture | Completed |
| Research Letta SDK thread/session capabilities | Completed |
| Design thread-scoping approach | In Progress |
| Create implementation plan | Planned |

**Goal**: For curriculum and agents defined in TOML, Letta agents should auto-scope conversations to individual OpenWebUI threads. Even if agents ultimately have one context thread, new threads should be scoped as such and persisted to Honcho.

## Critical References

- `src/letta_starter/server/thread_context.py` - Existing ThreadContextManager (context injection approach)
- `src/letta_starter/honcho/client.py` - Honcho integration (already thread-scoped sessions)
- `docs/Architecture.md` - System architecture overview

## Recent changes

No code changes made - this is a planning/design session.

## Learnings

### Current Architecture

1. **OpenWebUI → Backend Flow**: `chat_id` is passed from OpenWebUI through the pipeline to the HTTP service
2. **Honcho Already Thread-Scoped**: Messages persist to separate Honcho sessions per thread (`session_id = chat_{chat_id}`)
3. **Letta NOT Thread-Aware**: One Letta agent per `(user_id, agent_type)` with continuous message history across all threads

### Letta SDK Capabilities

1. **No native thread/session support** - Letta explicitly doesn't have threads; each agent has a single perpetual message history
2. **`client.agents.messages.reset(agent_id)`** - Clears message history **while preserving memory blocks**
3. **No `session_id` parameter** in the messages API
4. **`message_buffer_autoclear`** - Agent setting to auto-clear (too aggressive for our needs)

### Existing ThreadContextManager (`thread_context.py`)

Already implements context injection approach:
- Tracks current thread per agent
- Detects thread switches (new thread, returning thread, continuation)
- Loads thread history from Honcho
- Prepends context to messages (e.g., `[NEW CONVERSATION: {title}]`)

### User Requirements Clarified

- **Memory blocks persist** across threads (student profile, journey, etc.)
- **Conversation history scoped** to current thread
- **Other threads = archival memory** - agent shouldn't reference them directly
- **Returning to thread** = restore that thread's context from Honcho
- **New thread** = user initiates, no automatic opening

## Artifacts

- This handoff document: `thoughts/shared/handoffs/general/2026-01-11_14-25-50_thread-scoped-conversations.md`

## Action Items & Next Steps

### Recommended Approach: Reset + Restore

The recommended design is a hybrid approach using Letta's message reset combined with Honcho context restoration:

```
New Thread → Reset Letta messages → User initiates fresh
Return to Thread → Reset Letta messages → Load history from Honcho → Inject as context
Continue in Thread → No reset, normal flow
```

### Implementation Phases (Proposed)

1. **Phase 1: Add Reset on Thread Switch**
   - Modify `ThreadContextManager` or HTTP endpoints to detect thread switch
   - Call `client.agents.messages.reset(agent_id)` on thread change
   - Preserve `add_default_initial_messages=True` to restore system prompt

2. **Phase 2: Enhance Honcho Context Restoration**
   - When returning to a thread, load full conversation history from Honcho
   - Inject as structured context (not just summary)
   - Consider replaying messages or injecting as a "conversation so far" block

3. **Phase 3: TOML Configuration**
   - Add thread-scoping options to course TOML config
   - Allow per-course customization of thread behavior

4. **Phase 4: Testing & Refinement**
   - Test thread switching scenarios
   - Verify memory block persistence
   - Tune context injection format

### Immediate Next Steps

1. Create detailed implementation plan in `thoughts/shared/plans/`
2. Prototype Phase 1 (reset on thread switch)
3. Verify memory blocks persist after reset
4. Test with OpenWebUI thread switching

## Other Notes

### Key Files to Understand

| File | Purpose |
|------|---------|
| `src/letta_starter/server/thread_context.py:80-142` | `update_thread()` - detects thread changes |
| `src/letta_starter/server/thread_context.py:144-179` | `load_thread_history()` - loads from Honcho |
| `src/letta_starter/server/main.py:277-306` | Thread context detection in `/chat` endpoint |
| `src/letta_starter/server/agents.py:344-350` | `send_message()` - no thread params to Letta |
| `src/letta_starter/honcho/client.py:240-283` | `get_session_messages()` - retrieves thread history |

### Alternative Approaches Considered

1. **Agent Per Thread** - Create new Letta agent per OpenWebUI thread, share memory via Letta's shared blocks
   - Rejected: Agent proliferation, complex lifecycle management

2. **Enhanced Context Injection Only** - Keep current approach, improve context quality
   - Rejected: Context pollution, agent may reference wrong thread's content

3. **Message Buffer Autoclear** - Set `message_buffer_autoclear=True`
   - Rejected: Too aggressive, loses even within-session context

### Research Sources

- [Letta Reset Messages API](https://docs.letta.com/api-reference/agents/messages/reset)
- [Letta Core Concepts](https://docs.letta.com/core-concepts/) - confirms no native threads
- [Letta Agent Settings](https://docs.letta.com/guides/ade/settings/)
