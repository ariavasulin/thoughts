---
date: 2026-02-12T23:03:29-08:00
researcher: ARI
git_commit: f7c7833
branch: main
repository: YouLab
topic: "Artifact Push Pipeline: WebSocket/Polling Debugging & End-to-End Testing"
tags: [debugging, artifacts, openwebui, socket-io, polling, websocket, docker]
status: complete
last_updated: 2026-02-12
last_updated_by: ARI
type: implementation_strategy
---

# Handoff: Artifact Push Pipeline Debugging

## Task(s)

**Goal**: Get the live artifact push pipeline working end-to-end on localhost:3000 (OpenWebUI in Docker) so that when Ralph compiles a LaTeX file, the PDF viewer appears in the artifact panel.

**Status: In Progress** — We diagnosed the WebSocket transport issue, switched to polling, restarted Ralph, and confirmed both services work individually. Still need to test the full artifact push flow in the browser.

Working from:
- `thoughts/shared/plans/2026-02-10-live-artifact-push.md` (implementation plan, Phases 1-4)
- `thoughts/shared/research/2026-02-11-ARI-106-artifact-push-race-condition.md` (root cause analysis)
- `thoughts/shared/research/2026-02-10-openwebui-artifact-sideloading.md` (artifact system reference)

## Critical References

1. `thoughts/shared/research/2026-02-11-ARI-106-artifact-push-race-condition.md` — The `pushed: true` fix is already committed in the OpenWebUI fork (`e86e6854d`), but has NOT been verified working because events weren't reaching the browser.
2. `thoughts/shared/plans/2026-02-10-live-artifact-push.md` — Full 4-phase implementation plan. Phases 1-3 are code-complete. Phase 4 (cleanup) not started.
3. `src/ralph/pipe.py` — The pipe catches `incomplete chunked read` errors silently (line 121-123), which masked the real issue of Ralph being in a bad state.

## Recent changes

No code changes made this session. All work was diagnostic:
- Restarted OpenWebUI container with `ENABLE_WEBSOCKET_SUPPORT=false` (polling mode)
- Killed stale Ralph process (PID 2190) and started fresh instance
- Added `websockets` to dev deps: `pyproject.toml` / `uv.lock`

## Learnings

### WebSocket through Docker Desktop is broken
- Browser shows constant WebSocket reconnect loop (`Finished` / `0.0 kB` / `Pending` cycling in network tab)
- The raw WebSocket upgrade works from the host (`curl` gets `101 Switching Protocols` and valid SID)
- But browser connections through Docker Desktop port mapping (`3000->8080`) are unstable — connections open and immediately close
- **Fix**: Set `ENABLE_WEBSOCKET_SUPPORT=false` in the Docker container. Polling transport works reliably.
- Server with `ENABLE_WEBSOCKET_SUPPORT=true` sets `transports=["websocket"]` ONLY (no polling fallback), so when WebSocket fails, the client has no fallback and loops.

### Socket.IO event delivery is broken even when connected
- A Python `websockets` client that successfully connects and authenticates (`40{"sid":"..."}`) **never receives pushed artifact events**
- `sio.emit("events", ..., room="user:{user_id}")` returns success and logs `sessions=['...']` but events don't reach the client
- This was tested with `ENABLE_WEBSOCKET_SUPPORT=true` (WebSocket transport). Needs re-testing with polling.
- Root cause hypothesis: the sessions in the room may be stale from the reconnect cycling, OR there's a bug in how `always_connect=True` interacts with WebSocket-only transport.

### Ralph SSE stream fragility
- Ralph can get into a state where SSE streams immediately close after returning 200 (zero events delivered)
- Symptom: `peer closed connection without sending complete message body (incomplete chunked read)`
- The pipe at `src/ralph/pipe.py:121-123` catches this and logs `"Ralph: stream closed (message delivered)"` — which is misleading because no message was actually delivered
- **Fix**: Kill the old Ralph process and restart. Fresh instances stream correctly.
- The user's computer crashed, which left Ralph in this bad state.

### OpenWebUI container startup
- The container was started with `docker run` (not compose): `docker run -d --name youlab-openwebui-dev -p 3000:8080 -v .../tmp/openwebui-data:/app/backend/data -e ENABLE_WEBSOCKET_SUPPORT=false -e WEBUI_SECRET_KEY=dev-secret-key -e OLLAMA_BASE_URL=/ollama -e CORS_ALLOW_ORIGIN="*" youlab-openwebui-fork:latest`
- The pipe function is stored in the SQLite DB at `/app/backend/data/webui.db`, not as a file on disk
- OpenWebUI fork is at `OpenWebUI/open-webui/` with the `pushed: true` fix already committed (`e86e6854d`)

### `decode_token` doesn't handle API keys
- `backend/open_webui/utils/auth.py:205-210` — `decode_token()` only does JWT decode. API keys (`sk-...`) return `None`.
- The Socket.IO `@sio.on("connect")` handler uses `decode_token(auth["token"])` — so API key auth for Socket.IO connections silently fails (no room join, no events).
- Browser uses JWT from login flow (`localStorage.token`), so this doesn't affect normal usage — only programmatic Socket.IO clients using API keys.

## Artifacts

- No new files created
- `pyproject.toml` — added `websockets` dev dependency (minor, for testing only)

## Action Items & Next Steps

### Immediate (continue testing)
1. **Verify browser has stable polling connection** — After hard refresh on localhost:3000, check network tab for steady `GET/POST /ws/socket.io/?transport=polling` requests (no errors, no reconnect cycling)
2. **Send a chat message** — Verify Ralph streams a response through the pipe to the browser. Ralph is fresh on port 8200.
3. **Push a test artifact via curl** and check if it appears in the browser:
   ```
   curl -s -X POST "http://localhost:3000/api/artifact/push" \
     -H "Authorization: Bearer sk-25d932d048474c1a80d27277dd472d14" \
     -H "Content-Type: application/json" \
     -d '{"user_id":"ed6d1437-7b38-47a4-bd49-670267f0a7ce","content":"<html><body><h1>Test</h1></body></html>","title":"Test"}'
   ```
4. **If curl push works**: Test the full LaTeX flow — ask the agent to create notes, verify HookedFileTools triggers compilation and the PDF viewer appears
5. **If curl push doesn't work**: The Socket.IO room delivery is broken even with polling. Possible next steps:
   - Add server-side logging to the `@sio.on("connect")` handler to verify room joins
   - Check if `get_session_ids_from_room()` returns the current browser session
   - Consider Path B (embeds event) as a WebSocket-free alternative

### If Socket.IO push remains broken (Path B fallback)
- Use the `embeds` event type from the pipe instead of Socket.IO push
- The pipe emits `{"type": "embeds", "data": {"embeds": ["<html>..."]}}` — this goes through the event emitter (HTTP, not WebSocket), gets persisted to DB, and renders inline below the message
- Zero frontend changes needed, but artifacts render inline (not in side panel)
- See `thoughts/shared/research/2026-02-10-openwebui-artifact-sideloading.md` for details on Approach B

### Longer term
- Fix the `"stream closed (message delivered)"` false-positive log in `src/ralph/pipe.py:121-123`
- Investigate why Ralph SSE streams die after the process has been running for a while
- Consider whether WebSocket through Docker Desktop can be fixed (or if polling is acceptable for local dev)

## Other Notes

### Current service state (as of handoff)
- **OpenWebUI**: Docker container `youlab-openwebui-dev` on port 3000, `ENABLE_WEBSOCKET_SUPPORT=false` (polling)
- **Ralph**: Running on port 8200 (host), freshly started, streaming correctly
- **Dolt**: Docker container `youlab-dolt-1` on port 3307, running
- **Browser**: Should be on localhost:3000, needs hard refresh to pick up polling config

### OpenWebUI fork commit history (relevant)
```
e86e6854d fix: Handle chat:artifact in layout event handler  ← pushed:true fix
684867430 fix: Remove chat_id guard from artifact push handler
22b73d73a debug: Add session/room logging to artifact push endpoint
2d7e23073 feat: Add artifact push endpoint for external services
```

### Key file locations
- `src/ralph/artifacts.py` — compile_and_push, _push_artifact
- `src/ralph/tools/hooked_file_tools.py` — HookedFileTools with .tex auto-compile hook
- `src/ralph/pipe.py` — OpenWebUI pipe (SSE client)
- `src/ralph/server.py:182-442` — chat_stream endpoint and SSE generator
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:355-367` — chat:artifact handler
- `OpenWebUI/open-webui/src/routes/+layout.svelte:328-340` — layout chat:artifact handler
- `OpenWebUI/open-webui/backend/open_webui/routers/artifacts.py` — push endpoint
- `OpenWebUI/open-webui/backend/open_webui/socket/main.py:74-93` — Socket.IO server config
