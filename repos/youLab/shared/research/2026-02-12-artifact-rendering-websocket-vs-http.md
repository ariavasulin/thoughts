---
date: 2026-02-12T22:49:00-08:00
researcher: ARI
git_commit: f7c7833
branch: main
repository: YouLab
topic: "Artifact Rendering: WebSocket vs HTTP — Why artifacts don't render and what the alternatives are"
tags: [research, codebase, artifacts, openwebui, websocket, http, sse, socket-io, svelte]
status: complete
last_updated: 2026-02-12
last_updated_by: ARI
---

# Research: Artifact Rendering — WebSocket vs HTTP

**Date**: 2026-02-12T22:49:00-08:00
**Researcher**: ARI
**Git Commit**: f7c7833
**Branch**: main
**Repository**: YouLab

## Research Question

Artifacts can't render via WebSocket. Would it make sense to switch to HTTP?

## Summary

There are **two completely separate artifact delivery mechanisms** in the system, and the WebSocket/HTTP question applies differently to each:

### Mechanism 1: Code-Block Artifacts (message content)
Artifacts triggered by ` ```html ` code blocks in the agent's streamed message. These flow through **HTTP/SSE** (Ralph → Pipe) then **Socket.IO/WebSocket** (Pipe → Frontend). They work correctly today — the issue isn't the transport, it's that the agent currently doesn't emit HTML code blocks (the system switched to push-based delivery).

### Mechanism 2: Push Artifacts (out-of-band HTTP POST)
Artifacts pushed via `POST /api/artifact/push` from Ralph's background thread. These flow through **HTTP** (Ralph → OpenWebUI backend) then **Socket.IO** (OpenWebUI → Frontend). The backend pipeline works correctly, but the frontend has a **bug**: pushed artifacts are wiped by `getContents()` because the push handler doesn't set `pushed: true` on the artifact object (documented in ARI-106 research).

**The transport protocol (WebSocket vs HTTP) is not the root cause of the rendering failure.** The actual issue is a Svelte store management bug in the OpenWebUI fork's frontend code.

### Would HTTP Help?

Switching the artifact delivery from Socket.IO push to a pure HTTP polling approach would **not solve the problem** — the bug is in how `getContents()` rebuilds the `artifactContents` store on every history change, not in how the artifact arrives. Even if you delivered the artifact via HTTP polling, `getContents()` would still wipe it on the next message chunk.

There is, however, a **secondary benefit** to considering HTTP: the production environment uses Socket.IO in polling mode (`ENABLE_WEBSOCKET_SUPPORT=false` due to Cloudflare Tunnel limitations), which adds 1-5 seconds of latency. This doesn't cause the bug but shrinks the window where artifacts are visible before being wiped.

## Detailed Findings

### Current Architecture: How Artifacts Flow

```
┌──────────────────────────────────────────────────────────────────┐
│ Code-Block Artifacts (in-message)                                │
│                                                                  │
│  Agent response includes ```html ... ```                         │
│    → SSE stream: Ralph server → Ralph pipe                       │
│    → Event emitter: Pipe → OpenWebUI backend (in-process)        │
│    → Socket.IO: OpenWebUI → Browser                              │
│    → getContents() scans message → artifactContents store        │
│    → Artifacts.svelte renders iframe                             │
│                                                                  │
│  STATUS: Works. Not currently used (switched to push model).     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ Push Artifacts (out-of-band)                                     │
│                                                                  │
│  Agent writes .tex → HookedFileTools fires background thread     │
│    → Tectonic compiles → PDF bytes → HTML viewer                 │
│    → HTTP POST to /api/artifact/push (Ralph → OpenWebUI)         │
│    → sio.emit("events", {type: "chat:artifact"}, room=user)      │
│    → Chat.svelte handler appends to artifactContents             │
│    → [BUG] getContents() wipes it (missing pushed:true flag)     │
│                                                                  │
│  STATUS: Backend works. Frontend bug prevents display.           │
└──────────────────────────────────────────────────────────────────┘
```

### The Frontend Bug (Root Cause)

In the OpenWebUI fork's `Chat.svelte`, there are two competing systems:

**Push Handler** (`Chat.svelte:355-367`):
```javascript
artifactContents.update((current) => {
    return [...(current || []), { type: 'iframe', content: artifactData.content }];
    //                           ^^^ missing: pushed: true
});
```

**Reactive Scanner** (`Chat.svelte:844-902`):
```javascript
$: if (history) { getContents(); }  // Fires on EVERY streamed chunk

// Inside getContents():
const pushedArtifacts = currentContents.filter((c) => c.pushed);  // Always []
artifactContents.set([...contents, ...pushedArtifacts]);           // Wipes pushed
```

The scanner runs on every `history` change (i.e., every streamed message chunk) and rebuilds the store from scratch. It tries to preserve pushed artifacts by filtering for `c.pushed === true`, but the push handler never sets that flag. Result: pushed artifacts are wiped within milliseconds.

**Fix**: Add `pushed: true` to the artifact object in both `Chat.svelte:361` and `+layout.svelte:334`.

### Protocol Analysis

| Transport | Used For | Latency | Status |
|-----------|----------|---------|--------|
| SSE (HTTP) | Ralph server → Ralph pipe | ~0ms (same network) | Working |
| Socket.IO (polling) | OpenWebUI → Browser | 1-5s | Working but slow |
| Socket.IO (WebSocket) | OpenWebUI → Browser | ~50ms | Disabled (Cloudflare) |
| HTTP POST | Ralph → OpenWebUI `/api/artifact/push` | ~100ms | Working |

**Key insight**: The Ralph pipe runs **inside** OpenWebUI as an internal function. It has direct access to `__event_emitter__`, which is an in-process callback routed through Socket.IO. There is no HTTP→WebSocket translation issue — the pipe is already on the "inside" of the WebSocket boundary.

### Three Viable Delivery Approaches

#### Approach A: Fix the Push Bug (2-line frontend fix)
- Add `pushed: true` to `Chat.svelte:361` and `+layout.svelte:334`
- Artifacts arrive via Socket.IO push as designed
- Requires OpenWebUI fork rebuild and redeploy
- **Best approach**: minimal change, fixes the intended architecture

#### Approach B: Embeds Event (HTTP-adjacent, no fork change)
- Use `__event_emitter__({ "type": "embeds", "data": { "embeds": ["<html>..."] } })`
- Renders HTML iframe **below the message** (not in side panel)
- Persists in database, survives page refresh
- No OpenWebUI fork changes needed
- **Trade-off**: different UX (inline vs side panel)

#### Approach C: Code-Block in Message (original pattern)
- Agent includes ` ```html\n<viewer HTML>\n``` ` in response text
- `getContents()` detects it and populates artifact panel
- **Trade-off**: 100KB+ HTML blob in chat message, visible to user as code

### Production Environment Constraints

From `docker-compose.prod.yml:33`:
```yaml
ENABLE_WEBSOCKET_SUPPORT=false  # Cloudflare Tunnel doesn't proxy WebSocket
```

Socket.IO falls back to HTTP long-polling, adding 1-5s latency to all real-time events. This affects artifact push delivery time but is NOT the cause of the rendering bug.

If Cloudflare Tunnel could be configured for WebSocket (or if the tunnel was bypassed for Socket.IO traffic), latency would drop to ~50ms. This would make the "flash before wipe" window even shorter but wouldn't fix the underlying store bug.

## Code References

### Ralph (YouLab)
- `src/ralph/artifacts.py:21-66` — `compile_and_push()` main entry point
- `src/ralph/artifacts.py:114-143` — `_push_artifact()` HTTP POST to OpenWebUI
- `src/ralph/tools/hooked_file_tools.py:38-50` — `save_file()` with .tex hook
- `src/ralph/tools/hooked_file_tools.py:68-100` — Background thread compilation
- `src/ralph/pipe.py:46-131` — Pipe SSE consumption and event translation
- `src/ralph/pipe.py:160-222` — Event handler (SSE → event emitter)
- `src/ralph/pipe.py:134-158` — `_format_tool_html()` details tag rendering
- `src/ralph/server.py:182-442` — SSE streaming endpoint
- `src/ralph/server.py:291-310` — Agent construction with HookedFileTools
- `src/ralph/tools/latex_templates.py:91-249` — PDF viewer HTML template

### OpenWebUI Fork
- `Chat.svelte:355-367` — Push event handler (missing `pushed: true`)
- `Chat.svelte:844-902` — `getContents()` reactive scanner
- `Chat.svelte:898-901` — Preservation filter (`c.pushed`)
- `+layout.svelte:328-340` — Backup push handler (also missing flag)
- `+layout.svelte:107` — Transport config (polling vs websocket)
- `routers/artifacts.py:23-62` — Backend push endpoint
- `socket/main.py:77` — Server transport config
- `socket/main.py:693-834` — Event emitter implementation

### Production Config
- `docker-compose.prod.yml:33` — `ENABLE_WEBSOCKET_SUPPORT=false`
- `docker-compose.prod.yml:56-57` — `RALPH_OPENWEBUI_URL=http://openwebui:8080`

## Architecture Documentation

### OpenWebUI Event Emitter Types

| Event Type | Alias | Purpose |
|-----------|-------|--------|
| `status` | — | Progress/status above message |
| `message` | `chat:message:delta` | Append to message content |
| `replace` | `chat:message` | Replace entire message |
| `embeds` | `chat:message:embeds` | Add HTML iframes below message |
| `files` | `chat:message:files` | Attach files to message |
| `source`/`citation` | — | Add citations |
| `notification` | — | Toast notification |
| `chat:artifact` | — | Push artifact to side panel (custom) |

### OpenWebUI Dual-Channel Architecture

```
Browser ←→ Socket.IO (WebSocket or polling) ←→ OpenWebUI Server
                                                    ↑
                                              Pipe functions
                                              (in-process)
                                                    ↑
                                              __event_emitter__
                                              (Python callback)
                                                    ↑
                                              Ralph Pipe
                                              (HTTP/SSE client)
                                                    ↑
                                              Ralph Server
                                              (FastAPI + SSE)
```

The pipe runs in-process within OpenWebUI. It consumes SSE from Ralph's server and emits events via `__event_emitter__`, which routes through Socket.IO to the browser. There is no HTTP→WebSocket translation boundary for chat messages — only artifact pushes use a separate HTTP path.

## Historical Context

- `thoughts/shared/research/2026-02-11-ARI-106-artifact-push-race-condition.md` — Root cause analysis of the `pushed: true` bug
- `thoughts/shared/research/2026-02-10-openwebui-artifact-sideloading.md` — Four approaches (A-D) for artifact delivery
- `thoughts/shared/plans/2026-02-10-live-artifact-push.md` — Implementation plan for push-based artifacts (Phases 1-4)
- `thoughts/shared/research/2026-02-05-latex-artifacts-persistent-html-rendering.md` — Earlier LaTeX artifact research
- `thoughts/shared/research/2026-01-28-latex-pdf-live-sandbox-artifacts.md` — Original artifact system research

## Open Questions

1. Has the `pushed: true` fix been applied to the OpenWebUI fork yet? The research from 2026-02-11 identifies the fix but deployment status is unknown.
2. Could Cloudflare Tunnel be configured to support WebSocket? This would reduce Socket.IO latency from 1-5s to ~50ms for all real-time events (not just artifacts).
3. Should pushed artifacts persist across page refresh? Currently they only live in the Svelte store (in-memory). After refresh, only code-block artifacts survive from message history.
4. Is the `embeds` event type a viable fallback that avoids the `getContents()` race entirely? It renders inline (not side panel) but persists in the database.
