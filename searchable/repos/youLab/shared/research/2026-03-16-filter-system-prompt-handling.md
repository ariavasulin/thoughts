---
date: 2026-03-16T12:00:00-05:00
researcher: ARI
git_commit: 850ef44dcc0952f91e36bb38eb4c39f11fb231bf
branch: main
repository: Askademia
topic: "How does the lecture context filter handle system prompts — append or create new?"
tags: [research, codebase, openwebui, filters, system-prompt, lecture-context]
status: complete
last_updated: 2026-03-16
last_updated_by: ARI
---

# Research: How the Lecture Context Filter Handles System Prompts

**Date**: 2026-03-16
**Researcher**: ARI
**Git Commit**: 850ef44dcc0952f91e36bb38eb4c39f11fb231bf
**Branch**: main
**Repository**: Askademia

## Research Question

Does the lecture context filter append the lecture transcript to an existing system prompt, or create a new system prompt? How does Open WebUI's system prompt pipeline interact with the filter?

## Summary

**The filter does both — it conditionally appends OR creates, depending on whether a system message already exists.** The branching logic at `lecture_context_filter.py:68-77`:

- **If `messages[0]` has `role == "system"`**: Appends the transcript context block to the existing system message's content (line 69)
- **If no system message exists**: Inserts a new system message at index 0 with a default preamble + context block (lines 71-77)

**In practice, a system message will almost always already exist** by the time the filter runs, because Open WebUI injects system prompts from model configuration, user settings, and folder settings BEFORE filters execute. So the filter will almost always be appending to an existing system prompt.

## Detailed Findings

### 1. The Filter's Branching Logic

At `askademia-api/filters/lecture_context_filter.py:68-77`:

```python
if messages and messages[0].get("role") == "system":
    messages[0]["content"] += context_block
else:
    messages.insert(0, {
        "role": "system",
        "content": (
            "You are a helpful teaching assistant for a university course."
            + context_block
        ),
    })
```

**Branch A (line 69)**: Appends `context_block` to `messages[0]["content"]` via string concatenation (`+=`). The existing system message content is preserved; the transcript is added at the end.

**Branch B (lines 71-77)**: Creates a brand-new system message with the hardcoded preamble `"You are a helpful teaching assistant for a university course."` followed by the context block. Inserts it at position 0.

### 2. When Each Branch Executes

**The pipeline order in `middleware.py:2144-2331` determines what exists when the filter runs:**

1. Model system prompt applied (`openai.py:977` or `ollama.py:1336`) via `apply_system_prompt_to_body()`
2. User/chat settings system prompt applied (`middleware.py:2196`) with `replace=True`
3. Folder system prompt appended (`middleware.py:2248`)
4. Pipeline inlet filters run (`middleware.py:2311`)
5. **Function inlet filters run** (`middleware.py:2323`) — this is where the lecture context filter executes

**Branch A fires when** any of the above steps (1-4) has already placed a system message at `messages[0]`. This happens when:
- The model has a system prompt configured in Open WebUI admin
- The user has a system prompt in their chat settings (`$settings.system` from the frontend)
- The chat is in a folder that has a system prompt
- A pipeline filter added a system message

**Branch B fires when** none of the above applied — no model system prompt, no user settings system prompt, no folder prompt, and the frontend didn't send one. This would be unusual in a typical Askademia setup where models likely have system prompts configured.

### 3. Frontend System Prompt Construction

At `Chat.svelte:2081-2086`, the frontend conditionally prepends a system message:

```javascript
let messages = [
    params?.system || $settings.system
        ? { role: 'system', content: `${params?.system ?? $settings?.system ?? ''}` }
        : undefined,
    ..._messages.map(...)
].filter((message) => message);
```

This only adds a system message if the user has configured one in settings. The video context fields (`video_current_time`, `lecture_id`) are sent as **top-level body fields** at `Chat.svelte:2199-2204`, not inside the messages array.

### 4. The Context Block Structure

The transcript is wrapped in XML tags at `lecture_context_filter.py:58-66`:

```python
context_block = (
    "\n\n<lecture_transcript"
    f' lecture_id="{lecture_id}"'
    f' timestamp="{current_time:.1f}">\n'
    f"{transcript}\n"
    "</lecture_transcript>\n\n"
    "Use the lecture transcript above to answer questions about the lecture. "
    "Reference specific parts of the lecture when relevant."
)
```

This block is what gets either appended to the existing system prompt or used as part of the new one.

### 5. Comparison with Open WebUI's Built-in Pattern

Open WebUI's own system prompt utilities use `add_or_update_system_message()` at `utils/misc.py:356-374`, which has similar branching:

```python
def add_or_update_system_message(content, messages, append=False):
    if messages and messages[0].get("role") == "system":
        messages[0] = update_message_content(messages[0], content, append)
    else:
        messages.insert(0, {"role": "system", "content": content})
    return messages
```

The filter's logic mirrors this pattern but uses direct string concatenation (`+=`) instead of the utility function.

## Code References

- `askademia-api/filters/lecture_context_filter.py:68-77` — Branching logic for system message injection
- `askademia-api/filters/lecture_context_filter.py:58-66` — Context block construction
- `askademia-api/filters/lecture_context_filter.py:34-35` — Extracting video_current_time and lecture_id from body
- `open-webui/src/lib/components/chat/Chat.svelte:2081-2086` — Frontend system message construction
- `open-webui/src/lib/components/chat/Chat.svelte:2199-2204` — Video context fields added to request body
- `open-webui/backend/open_webui/utils/middleware.py:2193-2331` — Pipeline order showing filters run after system prompts
- `open-webui/backend/open_webui/utils/payload.py:14-42` — `apply_system_prompt_to_body()`
- `open-webui/backend/open_webui/utils/misc.py:356-374` — `add_or_update_system_message()`

## Architecture Documentation

### System Prompt Assembly Timeline

```
Frontend (Chat.svelte)
  └─ Adds system message IF $settings.system exists
  └─ Adds video_current_time + lecture_id as top-level body fields

Backend Router (openai.py / ollama.py)
  └─ Injects model system prompt from DB config

Middleware (middleware.py)
  └─ Replaces system message with chat settings version (template vars processed)
  └─ Appends folder system prompt (if in folder)
  └─ Runs pipeline inlet filters
  └─ Runs function inlet filters ← LECTURE CONTEXT FILTER RUNS HERE
      └─ If system message exists at messages[0]: APPENDS transcript
      └─ If no system message: CREATES new one with default preamble + transcript
  └─ Adds feature-specific prompts (voice, skills, etc.)
```

## Historical Context

- `thoughts/shared/research/2026-03-14-openwebui-filter-context-accumulation.md` — Related research confirming filter modifications are transient (not persisted to DB). The transcript is re-injected on every turn from the filter, using clean messages loaded from the database.

## Open Questions

- In the typical Askademia setup, does the model always have a system prompt configured? If so, Branch B (creating a new system message) may never execute in practice.
- The hardcoded preamble in Branch B ("You are a helpful teaching assistant...") would be overridden if a model system prompt is later added to the admin config — should this preamble be removed since it's redundant with model config?
