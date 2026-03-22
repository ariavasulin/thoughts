---
date: 2026-03-14T12:00:00-05:00
researcher: ARI
git_commit: 89ec636eea2cc61aa667d255b4393deefb0a4c49
branch: main
repository: Askademia
topic: "Do Open WebUI filter-injected contexts accumulate in conversation history?"
tags: [research, codebase, openwebui, filters, pipelines, context-injection]
status: complete
last_updated: 2026-03-14
last_updated_by: ARI
---

# Research: Do Open WebUI Filter-Injected Contexts Accumulate in Conversation History?

**Date**: 2026-03-14
**Researcher**: ARI
**Git Commit**: 89ec636eea2cc61aa667d255b4393deefb0a4c49
**Branch**: main
**Repository**: Askademia

## Research Question

When injecting a video transcript into a message via an Open WebUI filter's `inlet` function at timestep N, does the injected context accumulate in conversation history? On subsequent messages, is the previously-injected transcript re-sent as part of the full conversation history to the LLM?

## Summary

**No, filter-injected content does NOT accumulate in conversation history.** Filter modifications are **transient** — they only affect the in-memory payload sent to the LLM for that specific request. The database stores only the original, unmodified messages. On each new turn, messages are freshly loaded from the database and re-processed through filters.

However, there is a critical nuance: **the filter's `inlet` function receives the entire conversation history on every turn**, so if your filter modifies ALL messages (not just the latest one), it will re-inject context into older messages on every request. This isn't accumulation in the stored history, but it IS redundant context being sent to the LLM if you're not careful.

## Detailed Findings

### 1. Message Storage is Separate from Filter Processing

Messages are saved to the database **before** filters run:

- **User message saved first** at `main.py:1859-1866` — the original user message is persisted via `Chats.upsert_message_to_chat_by_id_and_message_id()` before `process_chat_payload()` is called
- **Database stores original content** in two places:
  - `chat` table JSON column (`chat.chat.history.messages`) — tree structure with parentId links
  - `chat_message` structured table — flat records with role, content, files

### 2. Messages are Loaded Fresh from DB on Each Turn

At `middleware.py:2157-2163`, on every new chat request:

```python
if chat_id and parent_message_id and not chat_id.startswith("local:"):
    db_messages = load_messages_from_db(chat_id, parent_message_id)
    if db_messages:
        system_message = get_system_message(form_data.get("messages", []))
        form_data["messages"] = (
            [system_message, *db_messages] if system_message else db_messages
        )
```

`load_messages_from_db()` at `middleware.py:2101-2117` queries the database and returns only raw stored fields (`role`, `content`, `output`, `files`). No filter artifacts are present in these stored messages.

### 3. Filters Modify an In-Memory Copy

At `middleware.py:2310-2331`, filters are applied **after** the DB load:

1. Pipeline inlet filters run (`process_pipeline_inlet_filter`)
2. Function inlet filters run (`process_filter_functions` with `filter_type="inlet"`)
3. The modified `form_data` is sent to the LLM
4. **The modified form_data is never written back to the database**

The filter execution at `filter.py:62-138` iterates through each filter, calling its `inlet()` handler. The returned `form_data` replaces the in-memory copy but is never persisted.

### 4. The Practical Implication for Your Transcript Filter

Given this architecture:

- **Turn 1**: User sends "explain the video". Your filter injects the transcript into the messages. LLM receives `[{user: "explain the video" + transcript}]`. DB stores `[{user: "explain the video"}]`.
- **Turn 2**: User sends "what about minute 5?". Messages loaded from DB: `[{user: "explain the video"}, {assistant: "..."}, {user: "what about minute 5?"}]`. Your filter runs again on ALL messages. If your filter appends the transcript to every user message, the LLM receives it three times. If your filter only appends to the latest message, it's sent once.

### 5. Important Design Consideration

Since filters receive the **entire conversation history** on each turn, a naive filter that appends a transcript to "the last user message" will work correctly — only the newest message gets the transcript, and previous messages are loaded clean from the DB.

But if your filter logic iterates through all messages looking for something to inject into, you could inadvertently inject into messages that previously had the injection (even though the DB version is clean, your filter would re-inject on every turn).

**Recommended approach**: Only modify the last message in the `body["messages"]` array, or use metadata/flags to determine which message to inject into.

## Code References

- `backend/open_webui/main.py:1859-1866` — User message saved to DB before filter processing
- `backend/open_webui/utils/middleware.py:2101-2117` — `load_messages_from_db()` loads original messages
- `backend/open_webui/utils/middleware.py:2157-2163` — Messages replaced with DB-loaded originals
- `backend/open_webui/utils/middleware.py:2310-2331` — Filter pipeline execution (inlet)
- `backend/open_webui/utils/filter.py:62-138` — `process_filter_functions()` core filter engine
- `backend/open_webui/models/chats.py:481-519` — `upsert_message_to_chat_by_id_and_message_id()` persists original messages
- `backend/open_webui/models/chat_messages.py:49-91` — `ChatMessage` table schema for structured storage

## Architecture Documentation

### Filter Execution Pipeline (per request)

```
1. User submits message → POST /api/chat/completions
2. Original message saved to DB (chat + chat_message tables)
3. process_chat_payload() called:
   a. Load ALL messages from DB (original, unmodified)
   b. Process output items
   c. Pipeline inlet filters (external HTTP services)
   d. Function inlet filters (Python classes from DB)
   e. Feature injection (memory, web search, skills, tools)
   f. System prompt application
4. Modified payload sent to LLM API
5. LLM response received
6. Outlet filters run on response
7. Assistant response saved to DB (original, unmodified)
```

### Filter Interface

```python
class Filter:
    def inlet(self, body: dict, __user__: dict = None) -> dict:
        # body["messages"] contains full conversation history
        # Modify and return body
        # Modifications are TRANSIENT — not saved to DB
        return body

    def outlet(self, body: dict, __user__: dict = None) -> dict:
        # body contains response data
        return body
```

## Open Questions

- If a filter needs the transcript available across all turns (not just the one where it's injected), should the transcript be re-injected every turn via the filter, or stored separately and referenced?
- For very large transcripts, what's the token budget impact of re-injecting on every turn vs. using RAG/retrieval?
