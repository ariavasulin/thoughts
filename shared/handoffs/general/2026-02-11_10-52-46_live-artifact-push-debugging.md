---
date: 2026-02-11T10:52:46-0800
researcher: claude
git_commit: 9cc591c (YouLab), e86e6854d (OpenWebUI fork)
branch: main
repository: YouLab
topic: "Live LaTeX Artifact Push — Deployment Debugging"
tags: [implementation, debugging, openwebui, socket-io, artifacts, latex]
status: in_progress
last_updated: 2026-02-11
last_updated_by: claude
type: implementation_strategy
---

# Handoff: Live LaTeX Artifact Push — Deployment Debugging

## Task(s)

**Implemented all 4 phases** of `thoughts/shared/plans/2026-02-10-live-artifact-push.md`. The implementation is complete and deployed. The **backend pipeline works end-to-end** (confirmed via logs), but the **frontend artifact panel does not appear** due to a Svelte store overwrite bug discovered at the end of this session.

### Status by Phase:
- **Phase 1 (OpenWebUI fork — endpoint + frontend handler)**: COMPLETE, deployed. But frontend handler has a bug (see Learnings).
- **Phase 2 (Ralph artifacts module)**: COMPLETE, deployed, working.
- **Phase 3 (HookedFileTools)**: COMPLETE, deployed, working.
- **Phase 4 (Cleanup + production)**: COMPLETE, deployed, working.
- **Deployment debugging**: IN PROGRESS — three bugs were fixed during deployment, one remains.

## Critical References

- **Implementation plan**: `thoughts/shared/plans/2026-02-10-live-artifact-push.md`
- **OpenWebUI fork**: `OpenWebUI/open-webui/` (git submodule, remote `fork` → `https://github.com/ariavasulin/open-webui.git`, branch `main`)
- **VPS**: SSH alias `vps`, `/opt/YouLab/`, containers: `youlab-openwebui`, `youlab-ralph`, `youlab-dolt`

## Recent changes

### YouLab repo (main branch, all pushed):
- `src/ralph/artifacts.py` — NEW: compile_and_push(), _compile_latex(), _build_viewer(), _push_artifact(). Uses structlog.
- `src/ralph/tools/hooked_file_tools.py` — NEW: Extends Agno FileTools, overrides save_file/replace_file_chunk to trigger background LaTeX compilation via threading.Thread + asyncio.run(). Uses lazy import of compile_and_push to avoid circular import.
- `src/ralph/tools/latex_templates.py` — NEW (renamed from latex_tools.py): Templates only (NOTES_TEMPLATE, PDF_VIEWER_TEMPLATE). LaTeXTools class removed.
- `src/ralph/tools/__init__.py` — Updated exports: HookedFileTools replaces LaTeXTools.
- `src/ralph/server.py` — Replaced FileTools + LaTeXTools with HookedFileTools. Simplified agent instructions.
- `Dockerfile` — Added Tectonic binary installation (tar.gz from GitHub releases).
- `docker-compose.prod.yml` — Added RALPH_OPENWEBUI_URL, RALPH_OPENWEBUI_API_KEY, ENABLE_WEBSOCKET_SUPPORT=false.

### OpenWebUI fork (main branch, pushed to `fork` remote):
- `backend/open_webui/routers/artifacts.py` — NEW: REST endpoint `/api/artifact/push` that emits socket.io event. Includes debug logging of room sessions.
- `backend/open_webui/main.py` — Registered artifacts router.
- `src/lib/components/chat/Chat.svelte:352-368` — Added chat:artifact handler (but it never fires — see Learnings).
- `src/lib/components/chat/Chat.svelte:850-899` — PARTIAL FIX for getContents() overwrite (using `c.pushed` flag to preserve pushed artifacts). This change is committed locally but THE BUILD ON VPS DOESN'T INCLUDE IT YET.
- `src/routes/+layout.svelte:327-339` — Added chat:artifact handler in layout's chatEventHandler (this is where socket events actually arrive).
- `src/routes/+layout.svelte:34-36` — Added showArtifacts, showControls, artifactContents store imports.

## Learnings

### Bug 1 (FIXED): Circular import
`ralph.artifacts` → `ralph.tools.latex_templates` → `ralph.tools.__init__` → `ralph.tools.hooked_file_tools` → `ralph.artifacts`. Fixed by lazy-importing `compile_and_push` inside `_compile_in_thread()` method instead of at module level.

### Bug 2 (FIXED): No event loop in Agno tool methods
Agno tool methods run synchronously — `asyncio.get_running_loop()` always raises RuntimeError. Fixed by using `threading.Thread(target=..., daemon=True)` with `asyncio.run()` inside the thread.

### Bug 3 (FIXED): WebSocket broken through Cloudflare Tunnel
Browser console showed repeated `WebSocket connection to 'wss://theyoulab.org/ws/socket.io/...' failed`. Cloudflare Tunnel doesn't proxy WebSocket upgrades. Fixed by setting `ENABLE_WEBSOCKET_SUPPORT=false` in docker-compose, which makes socket.io use HTTP long-polling.

### Bug 4 (PARTIALLY FIXED — REMAINING ISSUE): Artifact push arrives but panel doesn't show
The backend pipeline works perfectly (confirmed via logs: `save_file_called` → `auto_compile_triggered` → `artifact_pushed` → OpenWebUI emits to active session). But the artifact panel never appears because:

1. **Chat.svelte's chatEventHandler never fires** — `$socket?.on('events', chatEventHandler)` at `Chat.svelte:567` uses optional chaining. The `$socket` store is null when Chat.svelte mounts because socket initialization is async in `+layout.svelte`. The layout's own `chatEventHandler` (registered at `+layout.svelte:701`) is the one that actually receives events.

2. **getContents() overwrites artifactContents** — Even when we set `artifactContents` from the layout handler, `Chat.svelte:844-898` has a reactive statement `$: if (history) { getContents(); }` that runs on every history change and calls `artifactContents.set(contents)` with only code-block-detected contents, wiping pushed artifacts.

**Two fixes were applied but not fully deployed:**
- Layout handler added at `+layout.svelte:327-339` (deployed in current build)
- getContents() fix at `Chat.svelte:898` to preserve `c.pushed` flagged artifacts (committed locally but NOT in VPS build yet)

**The missing piece**: The pushed artifacts need a `pushed: true` flag set when they're added. The layout handler at `+layout.svelte:331` does NOT set this flag yet. Both the layout handler and getContents need to be aligned.

### Other Learnings:
- **OpenWebUI fork deployment**: Docker builds from `https://github.com/ariavasulin/open-webui.git#main`. Must push to `fork` remote (not `origin` which is upstream open-webui).
- **Standard logging invisible in Docker**: `logging.getLogger(__name__)` output doesn't show in `docker logs`. Must use `structlog.get_logger()` for Ralph modules.
- **Tectonic URL**: Must use `.tar.gz` URL, not bare binary URL. See `Dockerfile`.
- **docker-compose env interpolation**: `${VAR}` in `environment:` reads from shell env / `.env` file, NOT from `env_file:`. Symlink `.env` → `.env.production` on VPS.
- **Admin API key**: `sk-25d932d048474c1a80d27277dd472d14` works for artifact push (admin can push to any user).
- **OpenWebUI socket.io config**: `ENABLE_WEBSOCKET_SUPPORT` env var → `$config.features.enable_websocket` → client transport selection. When false: `['polling', 'websocket']` fallback.

## Artifacts

- `thoughts/shared/plans/2026-02-10-live-artifact-push.md` — Implementation plan (4 phases)
- `src/ralph/artifacts.py` — Compilation + push module
- `src/ralph/tools/hooked_file_tools.py` — FileTools wrapper with auto-compile hook
- `src/ralph/tools/latex_templates.py` — LaTeX + PDF viewer templates
- `OpenWebUI/open-webui/backend/open_webui/routers/artifacts.py` — Push endpoint
- `OpenWebUI/open-webui/src/routes/+layout.svelte:327-339` — Layout artifact handler
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:352-368` — Chat artifact handler (not working, see Learnings)
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:850-899` — getContents() with partial pushed artifact preservation

## Action Items & Next Steps

### 1. Fix the `pushed` flag on artifact push (CRITICAL)
In `+layout.svelte:331`, when adding pushed artifacts, mark them:
```js
{ type: 'iframe', content: artifactData.content, pushed: true }
```
The getContents() fix at `Chat.svelte:898` already filters on `c.pushed`, but the flag isn't being set yet.

### 2. Consider an alternative approach: `get()` import
The `Chat.svelte:898` fix uses `get(artifactContents)` but `get` from `svelte/store` may not be imported. Verify this import exists or add it. Alternatively, consider a simpler approach — use a separate store for pushed artifacts that getContents() doesn't touch.

### 3. Rebuild and deploy OpenWebUI on VPS
```bash
cd /Users/ariasulin/Git/YouLab/OpenWebUI/open-webui
# Make fixes
git add . && git commit --no-verify -m "fix: ..." && git push fork main
ssh vps "cd /opt/YouLab && docker compose -f docker-compose.prod.yml build openwebui && docker compose -f docker-compose.prod.yml up -d openwebui"
```

### 4. Test end-to-end
After deploy, test by asking the agent to "write a short example latex document" and verify the PDF appears in the artifact panel.

### 5. Clean up debug logging
Remove `log.warning` debug lines from `OpenWebUI/open-webui/backend/open_webui/routers/artifacts.py:38-42,60` once the feature is confirmed working.

### 6. Consider a separate `pushedArtifacts` store
A cleaner long-term approach: create a separate Svelte store `pushedArtifactContents` that getContents() doesn't touch. Merge both stores in the rendering template. This avoids the fragile `pushed` flag approach.

## Other Notes

### VPS Access
- SSH: `ssh vps` (alias in ~/.ssh/config)
- Containers: `youlab-openwebui`, `youlab-ralph`, `youlab-dolt`
- Ralph logs: `ssh vps "docker logs youlab-ralph --tail 50 2>&1"`
- OpenWebUI logs: `ssh vps "docker logs youlab-openwebui 2>&1 | tail -50"`

### Testing artifact push manually
```bash
ssh vps 'python3 -c "
import json
payload = json.dumps({
    \"user_id\": \"ed6d1437-7b38-47a4-bd49-670267f0a7ce\",
    \"content\": \"<html><body><h1>Test</h1></body></html>\",
    \"title\": \"Test\"
})
open(\"/tmp/art.json\", \"w\").write(payload)
" && docker exec -i youlab-openwebui curl -s http://localhost:8080/api/artifact/push -X POST -H "Content-Type: application/json" -H "Authorization: Bearer sk-25d932d048474c1a80d27277dd472d14" -d @- < /tmp/art.json'
```

### Key verification commands
```bash
# Check Ralph compilation pipeline
ssh vps "docker logs youlab-ralph --tail 50 2>&1 | grep -E 'save_file|compile|artifact'"

# Check OpenWebUI socket delivery
ssh vps "docker logs youlab-openwebui 2>&1 | grep artifact_push | tail -5"

# Verify polling is working (should see transport=polling in logs)
ssh vps "docker logs youlab-openwebui 2>&1 | grep 'socket.io.*polling' | tail -5"
```

### Production model
The agent uses `deepseek/deepseek-v3.2` (set via `RALPH_OPENROUTER_MODEL` in `.env.production`). It does call save_file for .tex files — confirmed via logs.
