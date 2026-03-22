---
date: 2026-02-13T00:35:00-08:00
researcher: ARI
git_commit: 18f229c42 (OpenWebUI fork)
branch: main
repository: YouLab + OpenWebUI/open-webui fork
topic: "Artifact Panel in youlabMode: Root Cause Found, Rendering Fix In Progress"
tags: [artifacts, openwebui, youlabmode, svelte, pane, chatcontrols, socket-io]
status: in_progress
last_updated: 2026-02-13
last_updated_by: ARI
type: implementation_handoff
---

# Handoff: Artifact Panel youlabMode Fix

## Task(s)

**Goal**: Get the live artifact push pipeline working end-to-end so that when Ralph compiles a LaTeX file, the PDF viewer appears in the artifact panel — specifically in youlabMode (the custom OpenWebUI mode used by YouLab).

**Status: Partially Complete** — Root cause found and first fix committed, but the rendering approach needs improvement. The artifact appears but without proper pane resizing.

Working from:
- `thoughts/shared/plans/2026-02-10-live-artifact-push.md` (implementation plan, Phases 1-4)
- `thoughts/shared/research/2026-02-11-ARI-106-artifact-push-race-condition.md` (pushed:true fix)
- `thoughts/shared/handoffs/general/2026-02-12_23-03-29_artifact-push-debugging.md` (previous handoff)

## Critical Discovery: Root Cause

### The `youlabMode` Guard Blocks All Artifact Rendering

**File**: `OpenWebUI/open-webui/src/lib/components/chat/ChatControls.svelte`

`youlabMode` is a Svelte store set to `true` by default (`stores/index.ts:35`). The entire `ChatControls.svelte` template was wrapped in `{#if !$youlabMode}`, which means when youlabMode=true, **zero HTML renders** — including the `<Artifacts>` component.

The artifact push pipeline works perfectly:
1. Ralph `compile_and_push()` → HTTP POST to OpenWebUI `/api/artifact/push` ✅
2. OpenWebUI backend emits Socket.IO `events` to `user:{user_id}` room ✅
3. Browser receives event via polling transport ✅
4. `chatEventHandler` in both `+layout.svelte` and `Chat.svelte` fires ✅
5. `artifactContents` store updated with `{type: 'iframe', content: html, pushed: true}` ✅
6. `showArtifacts.set(true)` and `showControls.set(true)` called ✅
7. **But `<Artifacts>` component is never mounted because of `{#if !$youlabMode}`** ❌

### The Pane Timing Problem

Even after allowing the template to render in youlabMode, there's a second issue:

In `Chat.svelte:595-615`, the `showControls.subscribe` handler calls `controlPaneComponent.openPane()` to resize the pane from 0 to a visible size. But in youlabMode, the `<Pane>` component isn't mounted when `showControls` first becomes true (because the `{#if}` guard needs to re-evaluate first). By the time the Pane mounts, the subscriber has already fired with `controlPane === null`, so `openPane()` is never called. The pane stays at `defaultSize={0}` (collapsed/invisible).

Approaches tried that did NOT work:
- Adding `$: if ($showArtifacts && pane && $youlabMode) { tick().then(() => openPane()) }` — reactive fires but pane still doesn't open, likely because pane.resize() needs the PaneGroup context to be fully initialized
- Using the existing desktop Pane branch with `{#if !$youlabMode || $showArtifacts}` — same pane timing issue

### Current Fix (Committed, Partially Working)

The committed fix (`18f229c42`) adds a new branch at the top of ChatControls:

```svelte
{#if $youlabMode && ($showArtifacts || $showEmbeds)}
    <div class="border-l ... w-[45%] min-w-[350px] max-h-full overflow-y-auto">
        {#if $showArtifacts}
            <Artifacts {history} />
        {:else if $showEmbeds}
            <Embeds />
        {/if}
    </div>
{:else if !$youlabMode}
    ... original code ...
{/if}
```

This renders Artifacts as a plain fixed-width div instead of a Pane. **It works** — the artifact appears! But it lacks:
- Drag-to-resize (no PaneResizer)
- Proper collapse/expand behavior
- Size persistence in localStorage

## Recent Changes

### OpenWebUI Fork (`OpenWebUI/open-webui`, commit `18f229c42`)
- `src/lib/components/chat/ChatControls.svelte`: Added youlabMode artifact rendering branch
- `src/lib/components/chat/Chat.svelte`: Added `pushed: true` to artifact objects in chatEventHandler
- `src/routes/+layout.svelte`: Same `pushed: true` fix (was already committed in `e86e6854d` but verified)

### Container Debug Patches (NOT committed, lost on container restart)
- `/app/backend/open_webui/socket/main.py`: Added SOCKET_CONNECT, SOCKET_USER_JOIN, SOCKET_USER_JOIN_ROOM logging
- `/app/backend/open_webui/routers/artifacts.py`: Added `/artifact/push-broadcast`, `/artifact/test-notification`, `/artifact/debug-rooms` debug endpoints

## Learnings

### Socket.IO Polling Transport Works
- After hard refresh, the browser establishes a stable polling connection
- JWT auth in the `connect` handler decodes correctly and joins the user room
- Events are delivered successfully (confirmed with `chat:completion` toast notification AND `chat:artifact` console.log)
- The `ERR_CONNECTION_RESET` errors from the previous handoff were caused by stale SIDs after Docker container restarts

### Event Delivery is NOT the Problem
We confirmed with instrumented JS (console.log patches in the built bundles) that:
- `LAYOUT_EVENT` fires for every Socket.IO event
- `LAYOUT_ARTIFACT_HIT` fires when type === 'chat:artifact'
- `CHAT_EVENT` and `CHAT_ARTIFACT_HIT` also fire
- `CHAT_ARTIFACT_STORE_SET {storeLen: 2}` confirms the store is updated
- `getContents()` does NOT wipe pushed artifacts (the `pushed: true` flag works)

The stores get set correctly. The problem is purely rendering — nothing subscribes to the stores in youlabMode.

### The Pane System in ChatControls
- `<PaneGroup>` is in Chat.svelte (parent)
- `<PaneResizer>` + `<Pane>` are in ChatControls (child)
- The Pane starts with `defaultSize={0}` and `collapsible={true}`
- `openPane()` in ChatControls calls `pane.resize(size)` where size comes from localStorage or minSize
- The `showControls.subscribe` in Chat.svelte triggers `openPane()` — but this fires BEFORE the Pane mounts in youlabMode

### youlabMode Purpose
`youlabMode` was added to customize the OpenWebUI UI for YouLab — hiding the model selection controls, parameter tuning, and other power-user features. It was never intended to block artifacts/embeds, but the `{#if !$youlabMode}` guard was too broad.

## Artifacts

- OpenWebUI fork commit: `18f229c42` — ChatControls youlabMode artifact fix + pushed:true
- Previous commits in fork: `e86e6854d` (pushed:true in layout), `684867430`, `22b73d73a`, `2d7e23073` (artifact push endpoint)

## Action Items & Next Steps

### Immediate: Fix Artifact Pane Rendering (Priority 1)

The current fix renders Artifacts in a plain div. To get proper pane behavior, investigate these approaches:

1. **Approach A: Standalone resizable div** — Use CSS `resize: horizontal` or a lightweight resize handle instead of paneforge. The Artifacts component doesn't need the full PaneGroup system.

2. **Approach B: Fix the Pane timing** — Make the Pane start visible (non-zero defaultSize) when $showArtifacts is true in youlabMode. The key files:
   - `ChatControls.svelte:236-258` — The `<Pane>` config with `defaultSize={0}`
   - `ChatControls.svelte:46-56` — `openPane()` function
   - `Chat.svelte:595-615` — `showControls.subscribe` that calls `openPane()`

   A possible fix: instead of `defaultSize={0}`, use a reactive defaultSize: `defaultSize={$showArtifacts ? minSize : 0}`. Or call `pane.resize()` from a reactive block inside ChatControls after tick().

3. **Approach C: Render Artifacts outside ChatControls entirely** — Add an `{#if $showArtifacts && $youlabMode}` block directly in Chat.svelte, next to the PaneGroup, so it renders independently of the controls pane system. This avoids the entire Pane timing issue.

### Test the Full LaTeX Pipeline (Priority 2)

Once artifacts render properly:
1. Start Ralph (should be running on port 8200)
2. Send a chat message asking to create LaTeX notes
3. Verify: agent writes .tex → HookedFileTools triggers compile_and_push → PDF appears in artifact panel
4. Verify: editing the .tex file updates the PDF

### Clean Up Debug Code (Priority 3)

- Remove debug endpoints from `artifacts.py` (push-broadcast, test-notification, debug-rooms)
- Remove debug logging from `socket/main.py` (SOCKET_CONNECT, SOCKET_USER_JOIN)
- These are only in the container, not committed

### Rebuild Docker Image (Priority 4)

The current container has patched Python files. For a clean deployment:
1. Push OpenWebUI fork changes to GitHub
2. Rebuild the `youlab-openwebui-fork:latest` Docker image
3. Recreate the container

## Current Service State

- **OpenWebUI**: Docker container `youlab-openwebui-dev` on port 3000
  - `ENABLE_WEBSOCKET_SUPPORT=false` (polling transport)
  - Has debug patches in Python backend (not committed)
  - Has the new frontend build (committed changes)
- **Ralph**: Running on port 8200 (host)
- **Dolt**: Docker container on port 3307
- **OpenWebUI Fork**: `OpenWebUI/open-webui/` with commit `18f229c42`

## Key File Locations

### OpenWebUI Fork
- `src/lib/components/chat/ChatControls.svelte:155-163` — **THE FIX** (youlabMode artifact branch)
- `src/lib/components/chat/Chat.svelte:358-367` — chatEventHandler artifact push (pushed:true)
- `src/lib/components/chat/Chat.svelte:595-615` — showControls.subscribe (pane open logic)
- `src/lib/components/chat/Chat.svelte:844-870` — getContents() reactive (store rebuilder)
- `src/lib/components/chat/Artifacts.svelte` — The Artifacts component itself
- `src/lib/stores/index.ts:35` — `youlabMode = writable(true)`
- `src/routes/+layout.svelte:328-340` — layout chatEventHandler (pushed:true)
- `backend/open_webui/routers/artifacts.py` — push endpoint
- `backend/open_webui/socket/main.py:301-315` — connect handler + room join

### Ralph (YouLab)
- `src/ralph/artifacts.py` — compile_and_push, _push_artifact
- `src/ralph/tools/hooked_file_tools.py` — HookedFileTools with .tex auto-compile
- `src/ralph/pipe.py` — OpenWebUI pipe (SSE client)
- `src/ralph/server.py` — chat_stream endpoint

## Debugging Tips

### Quick Artifact Push Test
```bash
cat > /tmp/artifact-test.json << 'EOF'
{
  "user_id": "ed6d1437-7b38-47a4-bd49-670267f0a7ce",
  "content": "<html><body style='font-family:sans-serif;padding:40px;'><h1>Test</h1></body></html>",
  "title": "Test"
}
EOF
curl -s -X POST "http://localhost:3000/api/artifact/push" \
  -H "Authorization: Bearer sk-25d932d048474c1a80d27277dd472d14" \
  -H "Content-Type: application/json" \
  -d @/tmp/artifact-test.json
```

### Check Socket.IO Room State
```bash
curl -s "http://localhost:3000/api/artifact/debug-rooms" \
  -H "Authorization: Bearer sk-25d932d048474c1a80d27277dd472d14" | python3 -m json.tool
```
(Only works if debug endpoints are still patched in the container)

### Verify Browser Socket Connection
After hard refresh, check container logs:
```bash
docker logs youlab-openwebui-dev 2>&1 | grep "SOCKET_USER_JOIN_ROOM"
```

### Patch JS Bundles for Console Logging
To add console.log to minified JS without rebuilding:
```bash
docker cp youlab-openwebui-dev:/app/build/_app/immutable/chunks/XXXXX.js /tmp/chunk.js
perl -pi -e 's/PATTERN/console.log("DEBUG");PATTERN/' /tmp/chunk.js
docker cp /tmp/chunk.js youlab-openwebui-dev:/app/build/_app/immutable/chunks/XXXXX.js
# No restart needed for static JS files, just hard refresh browser
```
