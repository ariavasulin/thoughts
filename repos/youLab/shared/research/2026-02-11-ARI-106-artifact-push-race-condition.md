---
date: 2026-02-11T21:18:00-08:00
researcher: ARI
git_commit: 9cc591c95bc9d87928836fdcf5d989c66b14bb98
branch: main
repository: YouLab
topic: "Artifact Push Race Condition: Why pushed artifacts don't display"
tags: [research, codebase, artifacts, openwebui, socket-io, race-condition, svelte]
status: complete
last_updated: 2026-02-11
last_updated_by: ARI
linear_ticket: ARI-106
---

# Research: Artifact Push Race Condition

**Date**: 2026-02-11T21:18:00-08:00
**Researcher**: ARI
**Git Commit**: 9cc591c
**Branch**: main
**Repository**: YouLab
**Linear**: [ARI-106](https://linear.app/ariav/issue/ARI-106)

## Research Question

Why do artifacts pushed via the `chat:artifact` Socket.IO event not appear in the OpenWebUI artifact panel, despite the full push pipeline (Ralph compilation -> HTTP POST -> OpenWebUI backend -> Socket.IO emit) executing successfully?

## Summary

The root cause is a **missing `pushed: true` flag** on artifacts created by the `chat:artifact` event handler. OpenWebUI's `getContents()` reactive function runs on every `history` change (i.e., every streamed message chunk) and rebuilds the `artifactContents` store from scratch by scanning message code blocks. It attempts to preserve pushed artifacts by filtering for `c.pushed === true`, but the event handler never sets this flag. As a result, pushed artifacts are immediately wiped on the next history update.

This is a **one-line fix** in two files in the OpenWebUI fork.

## Detailed Findings

### The Two Competing Systems

#### System 1: Push Handler (`Chat.svelte:355-367`)

When a `chat:artifact` Socket.IO event arrives, the handler appends the content to the `artifactContents` Svelte store:

```javascript
// Chat.svelte:355-367
if (event?.data?.type === 'chat:artifact') {
    const artifactData = event?.data?.data ?? null;
    if (artifactData?.content) {
        artifactContents.update((current) => {
            return [...(current || []), { type: 'iframe', content: artifactData.content }];
            //                           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            //                           Note: NO `pushed: true` property
        });
        showArtifacts.set(true);
        showControls.set(true);
    }
    return;
}
```

The same handler exists in `+layout.svelte:328-340` with identical code.

#### System 2: Reactive Content Scanner (`Chat.svelte:844-902`)

A Svelte reactive statement triggers `getContents()` whenever `history` changes:

```javascript
// Chat.svelte:844-848
$: if (history) {
    getContents();
} else {
    artifactContents.set([]);
}
```

`getContents()` at line 850-902:
1. Scans ALL messages in history for HTML/CSS/JS/SVG code blocks
2. Builds a new `contents` array from those code blocks
3. Attempts to preserve pushed artifacts:

```javascript
// Chat.svelte:898-901
const currentContents = get(artifactContents) || [];
const pushedArtifacts = currentContents.filter((c) => c.pushed);  // <-- THE BUG
artifactContents.set([...contents, ...pushedArtifacts]);
```

The filter `c.pushed` finds nothing because the push handler never sets `pushed: true`.

### The Race Timeline

1. **T+0s**: User sends message, agent starts streaming response
2. **T+1s**: Agent calls `save_file("notes.tex", content)` via HookedFileTools
3. **T+1s**: HookedFileTools spawns background daemon thread for compilation
4. **T+1-3s**: Agent continues streaming text response; each SSE chunk updates `history`, triggering `getContents()`
5. **T+3-8s**: Background thread compiles LaTeX via Tectonic, builds HTML viewer, POSTs to `/api/artifact/push`
6. **T+3-8s**: OpenWebUI backend receives push, emits `chat:artifact` to `user:{user_id}` room
7. **T+3-9s**: Frontend receives Socket.IO event (extra latency from polling transport), appends to `artifactContents`
8. **T+3-9s**: On the very next message chunk (or history reactivity tick), `getContents()` runs and **wipes the pushed artifact**

Even if WebSocket were enabled (lower latency), the race would still occur because `history` updates continuously during streaming.

### Full Data Flow (What Works)

The entire backend pipeline is correctly implemented:

#### Ralph Side
- `HookedFileTools.save_file()` (`src/ralph/tools/hooked_file_tools.py:38-50`): Detects `.tex` writes, triggers `_trigger_compile()`
- `_trigger_compile()` (`hooked_file_tools.py:66-86`): Spawns daemon thread
- `_compile_in_thread()` (`hooked_file_tools.py:88-96`): Runs `asyncio.run(compile_and_push(...))`
- `compile_and_push()` (`src/ralph/artifacts.py:21-66`): Compiles via Tectonic, builds HTML viewer, pushes
- `_push_artifact()` (`artifacts.py:114-141`): HTTP POST to `{openwebui_url}/api/artifact/push`

#### OpenWebUI Backend
- `push_artifact()` (`routers/artifacts.py:23-62`): Validates auth, emits Socket.IO event to user room
- Logs show: room, session IDs, content length (line 38-42), emit completed (line 60)

#### OpenWebUI Frontend
- `chatEventHandler` in `Chat.svelte:352-367`: Receives event, appends to store
- `chatEventHandler` in `+layout.svelte:328-340`: Backup handler, same logic
- Console logs: `'chat:artifact received'` (line 357) confirms event arrives

**Everything works correctly up to the point where `getContents()` wipes the store.**

### Secondary Issue: Polling Transport Latency

Production config (`docker-compose.prod.yml:33`):
```yaml
ENABLE_WEBSOCKET_SUPPORT=false  # Use polling; Cloudflare Tunnel doesn't proxy WebSocket
```

Frontend transport config (`+layout.svelte:107`):
```javascript
transports: enableWebsocket ? ['websocket'] : ['polling', 'websocket'],
```

When `enable_websocket` is false, the client uses `['polling', 'websocket']` — polling first, with WebSocket upgrade as fallback. But since the server only allows polling (`socket/main.py:77`), the connection stays on polling.

HTTP long-polling adds ~1-5 seconds latency to event delivery compared to WebSocket (~50ms). This means:
- The pushed artifact arrives later
- More `history` updates (and thus more `getContents()` calls) happen before the push arrives
- The window where the artifact is visible (between push receipt and next `getContents()` run) is shorter

However, polling is NOT the root cause — even with WebSocket, the missing `pushed: true` flag would still cause wipes.

### What the Previous Agent Tried

Based on the plan document and git history, the previous session implemented the full pipeline correctly:
- `92cf300` feat: Live artifact push (auto-compile LaTeX to PDF in artifact panel)
- `520d854` fix: Run LaTeX compilation in background thread instead of event loop
- `5f22483` fix: Break circular import in hooked_file_tools by lazy-importing compile_and_push
- `96126c8` fix: Switch to structlog in hooked_file_tools and artifacts for visible logging
- `9cc591c` fix: Disable WebSocket in OpenWebUI, use polling for Cloudflare Tunnel

The previous agent was debugging by:
1. Adding extensive logging (structlog migration)
2. Fixing the threading model (background thread instead of event loop)
3. Fixing circular imports (lazy import)
4. Trying WebSocket vs polling

None of these addressed the actual bug because the bug is in the **frontend store management**, not the backend push pipeline.

## The Fix

### Primary Fix (2 lines)

**`Chat.svelte` line 361**: Add `pushed: true` to the artifact object:
```javascript
return [...(current || []), { type: 'iframe', content: artifactData.content, pushed: true }];
```

**`+layout.svelte` line 334**: Same change:
```javascript
return [...(current || []), { type: 'iframe', content: artifactData.content, pushed: true }];
```

This ensures `getContents()` at line 900 preserves pushed artifacts when it rebuilds the store.

### Deployment Note

The OpenWebUI fork is not checked out locally — it's built from `https://github.com/ariavasulin/open-webui.git#main` in docker-compose. The fix must be pushed to that repo, then the OpenWebUI container rebuilt.

## Code References

### OpenWebUI Fork (YouLab local copy)
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:355-367` - Push event handler (missing `pushed: true`)
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:844-902` - `getContents()` reactive scanner
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:898-901` - Preservation filter (`c.pushed`)
- `OpenWebUI/open-webui/src/routes/+layout.svelte:328-340` - Backup push handler (also missing `pushed: true`)
- `OpenWebUI/open-webui/src/routes/+layout.svelte:107` - Transport config (polling vs websocket)
- `OpenWebUI/open-webui/backend/open_webui/routers/artifacts.py:23-62` - Backend push endpoint
- `OpenWebUI/open-webui/backend/open_webui/socket/main.py:77` - Server transport config
- `OpenWebUI/open-webui/backend/open_webui/env.py:610-612` - `ENABLE_WEBSOCKET_SUPPORT` env var

### Ralph (YouLab)
- `src/ralph/artifacts.py:21-66` - `compile_and_push()` main entry point
- `src/ralph/artifacts.py:114-141` - `_push_artifact()` HTTP POST to OpenWebUI
- `src/ralph/tools/hooked_file_tools.py:38-50` - `save_file()` override with .tex detection
- `src/ralph/tools/hooked_file_tools.py:66-96` - Background thread compilation
- `src/ralph/server.py:275-279` - HookedFileTools agent registration
- `src/ralph/config.py:47-49` - `openwebui_url` and `openwebui_api_key` settings

### Production Config
- `docker-compose.prod.yml:33` - `ENABLE_WEBSOCKET_SUPPORT=false`
- `docker-compose.prod.yml:56-57` - `RALPH_OPENWEBUI_URL=http://openwebui:8080`

## Architecture Documentation

### Current Artifact Push Architecture
```
Agent calls save_file("notes.tex", content)
    -> HookedFileTools.save_file() writes file to workspace
    -> _trigger_compile() spawns daemon thread
        -> _compile_in_thread() creates new event loop
            -> compile_and_push(tex_path, user_id, chat_id)
                -> _compile_latex() runs Tectonic subprocess
                -> _build_viewer() builds HTML with base64 PDF
                -> _push_artifact() POSTs to OpenWebUI
                    -> OpenWebUI /api/artifact/push validates auth
                    -> sio.emit("events", {...}, room="user:{user_id}")
                        -> Frontend chatEventHandler receives event
                        -> artifactContents.update([..., {type: 'iframe', content}])
                        -> showArtifacts.set(true)
                        -> [BUG] getContents() immediately wipes it
```

### Svelte Store Lifecycle
```
history changes (every streamed chunk)
    -> $: if (history) getContents()
        -> scan all messages for code blocks
        -> build new contents array
        -> filter current store for c.pushed === true
        -> set store to [...codeBlockContents, ...pushedArtifacts]
        -> pushedArtifacts is ALWAYS [] because pushed flag never set
        -> pushed artifact vanishes
```

## Historical Context

- `thoughts/shared/plans/2026-02-10-live-artifact-push.md` - Implementation plan (Phases 1-4)
- `thoughts/shared/research/2026-02-10-openwebui-artifact-sideloading.md` - Research on artifact approaches (A-D)
- `thoughts/shared/research/2026-02-05-latex-artifacts-persistent-html-rendering.md` - Earlier LaTeX artifact research

The plan at Phase 1 (line 170) correctly identified that the handler needs to go before the `if (message)` guard and includes the preservation code pattern. However, the `pushed: true` flag was added to `getContents()` filter (line 900) but was never added to the push handler that creates the artifacts (line 361).

## Open Questions

1. Should pushed artifacts persist across page refresh? Currently they only live in the Svelte store (in-memory). After refresh, only code-block artifacts survive (from message history).
2. Should the OpenWebUI fork be checked out as a submodule in YouLab for easier local development?
3. Is Cloudflare Tunnel truly incapable of WebSocket, or could the tunnel config be updated? (Would reduce polling latency)
