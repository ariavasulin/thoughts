---
date: 2026-02-11T21:41:41-08:00
researcher: claude
git_commit: cb19c4c71dacbb8bda75457bb204519ba0e3f8aa
branch: main
repository: YouLab
topic: "Welcome Course Demo - Per-Chat System Prompts & Block Init"
tags: [implementation, welcome-course, system-prompts, memory-blocks, ralph]
status: complete
last_updated: 2026-02-11
last_updated_by: claude
type: implementation_strategy
---

# Handoff: Welcome Course Demo & System Prompt Review

## Task(s)

### Completed
1. **Per-chat system prompt support** — Ralph server now extracts `role: "system"` messages from OpenWebUI and uses them as agent instructions instead of the hardcoded generic prompt. Falls back to default when no system message is present.
2. **Welcome block auto-initialization** — When a new user sends their first message, Ralph creates 4 memory blocks from welcome.toml templates: `origin_story`, `tech_relationship`, `ai_partnership`, `onboarding_progress`.
3. **Tool instructions separation** — Tool usage instructions (memory blocks, LaTeX notes, workspace) are now appended separately from the base prompt, so they're always included regardless of whether a per-chat system prompt is set.
4. **Deployed to VPS** — Changes pushed and Ralph rebuilt on production (`/opt/YouLab` on `vps`).
5. **Chat injection endpoint plan** — Written at `thoughts/shared/plans/2026-02-11-chat-message-injection.md`, Linear ticket ARI-105 created. Not implemented yet — for later.
6. **Chat injection endpoint implemented** — `src/ralph/api/chats.py` and `src/ralph/openwebui_client.py` were created (visible in server.py imports). These wrap OpenWebUI's chat API for programmatic message injection.

### Next: Review and refine system prompts
The user wants to review and work on each system prompt — specifically the welcome.toml system prompt content and how it integrates with the per-chat system prompt mechanism.

## Critical References
- `~/Desktop/welcome.toml` — The full welcome course definition (system prompt, block templates, first message, background tasks)
- `thoughts/shared/plans/2026-02-11-chat-message-injection.md` — Plan for the chat injection endpoint (ARI-105)

## Recent changes
- `src/ralph/server.py:191-260` — System message extraction + tool instructions separation + welcome block init
- `src/ralph/memory.py:14-100` — `WELCOME_BLOCK_TEMPLATES` constant + `ensure_welcome_blocks()` function
- `src/ralph/api/chats.py` — New chat injection API router (POST /chats/send)
- `src/ralph/openwebui_client.py` — OpenWebUI API client for chat creation/message append

## Learnings

1. **OpenWebUI per-chat system prompts arrive as the first message** with `role: "system"` in the messages array sent to pipes. The pipe already forwards this — Ralph just wasn't extracting it.
2. **OpenWebUI Chat Controls** (top-right icon in chat) has a "System Prompt" field where you paste per-chat prompts. This is how you configure individual chats with different agent personalities.
3. **OpenWebUI channels** are a collaborative Discord-like feature, NOT what we want. Regular chats with per-chat system prompts are the right mechanism for single-thread module chats.
4. **OpenWebUI's `POST /api/chats/new` cannot create chats for other users** — only for the authenticated user. The fork needs a patch to add admin `user_id` override (detailed in the chat injection plan).
5. **Ralph has 4 tool kits**: ShellTools, HookedFileTools, HonchoTools, MemoryBlockTools. The welcome.toml only expects 3 (memory_blocks, honcho_tools, file_tools — no shell). Tool selection is still hardcoded, not TOML-driven.
6. **Pre-existing test failures**: Legacy `youlab_server` tests fail due to missing `letta_client` module. Ralph-specific tests (`tests/ralph/`) all pass (18/18).
7. **VPS YouLab directory**: `/opt/YouLab` (not `/root/YouLab`).

## Artifacts
- `thoughts/shared/plans/2026-02-11-chat-message-injection.md` — Chat injection endpoint plan
- `src/ralph/memory.py:14-100` — Welcome block templates and init function
- `src/ralph/server.py:191-260` — Per-chat system prompt extraction logic
- `src/ralph/api/chats.py` — Chat injection endpoint
- `src/ralph/openwebui_client.py` — OpenWebUI API client
- Linear ticket: ARI-105 (Chat message injection endpoint)

## Action Items & Next Steps

1. **Review welcome.toml system prompt** (`~/Desktop/welcome.toml:11-131`) — The user wants to iterate on the system prompt content, refine tone/instructions, and potentially adjust what the agent is told to do in each phase.
2. **Test the demo flow end-to-end** — Create a chat in OpenWebUI, paste welcome system prompt into Chat Controls, send a message, verify blocks get created and agent behaves correctly.
3. **Set up professor's demo chat** — Paste welcome system prompt, optionally pre-type the first_message content as the opening assistant message.
4. **Consider moving welcome.toml to `config/courses/welcome/agents.toml`** — Currently sitting on Desktop, should be in the repo for persistence.

## Other Notes

- The `docs/Agent-Tools.md` file was also modified (visible in git status) but those changes are unrelated to this work.
- `CLAUDE.md` was also modified — check if those changes need committing.
- The `openwebui-content-sync/` and `tmp/` directories are untracked — likely scratch work.
- The welcome.toml has a mismatch: `blocks = [..."progress"]` but the actual block definition uses `label = "onboarding_progress"`. The code uses `onboarding_progress` correctly.
