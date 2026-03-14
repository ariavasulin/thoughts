# Video Timestamp → Lecture Context Pipeline

## Overview

Wire up the Askademia video player end-to-end so that when a student sends a chat message, the lecture transcript up to their current video position is automatically injected into the LLM system prompt.

**Flow:** `<mux-player>` timeupdate → Svelte store → Chat.svelte request body → Open WebUI filter → askademia-api `/context` → transcript injected into system message

## Current State Analysis

- **VideoPlayer.svelte** (`open-webui/src/lib/components/chat/FileNav/VideoPlayer.svelte`): Thin wrapper around `<mux-player>` web component. Accepts `playbackId`, `src`, `title`, `aspectRatio` props. Does NOT expose `currentTime` or bind to any events.
- **FileNav.svelte** (`open-webui/src/lib/components/chat/FileNav.svelte:631-634`): Mounts VideoPlayer with a hardcoded demo playback ID (`EcHgOK9coz5K4rjSwOkoE7Y7O01201YMIC200RI6lNxnhs`).
- **Chat.svelte** (`open-webui/src/lib/components/chat/Chat.svelte:2179-2243`): Builds request body for `generateOpenAIChatCompletion`. No video state included.
- **Backend metadata** (`open-webui/backend/open_webui/main.py:1775-1802`): Constructs metadata dict from an explicit allowlist. Custom fields sent from the frontend survive at the top level of `form_data` but are NOT copied into `form_data["metadata"]`.
- **askademia-api** (`askademia-api/askademia/main.py:43-52`): `POST /context` accepts `{lecture_id, current_time}`, returns `{transcript}`. Full lecture 10 transcript is ~18k tokens (534 segments, 80 min) — well within context limits.
- **Filters**: Python modules stored in DB, created via admin UI or API. `inlet(body, __user__)` runs before the LLM call. Receives full `form_data` as `body`.

### Key Discoveries:
- `<mux-player>` supports standard HTMLMediaElement API: `.currentTime` property and `timeupdate` event work identically to `<video>` — confirmed via Mux docs
- Filter `inlet()` receives `body` where custom fields from frontend are at top level: `body.get("video_current_time")`, NOT `body["metadata"]["video_current_time"]`
- Existing utility `add_or_update_system_message(content, messages, append=True)` at `open-webui/backend/open_webui/utils/misc.py:356` handles system message injection
- Filter runs backend-to-backend (`requests.post`), so CORS on askademia-api is irrelevant

## Desired End State

1. VideoPlayer shows lecture 10 video via Mux
2. A Svelte store tracks the current playback timestamp
3. Every chat completion request includes `video_current_time` and `lecture_id`
4. An Open WebUI filter intercepts the request, fetches the transcript from askademia-api, and injects it into the system prompt
5. The LLM receives the lecture transcript as context and can answer questions about the lecture

### How to verify:
- Play the lecture 10 video to ~5 minutes
- Send a chat message asking about the lecture content
- Confirm the LLM response references lecture material from the first 5 minutes
- Check browser network tab: the chat completion request body includes `video_current_time` and `lecture_id`

## What We're NOT Doing

- Dynamic lecture selection (lecture ID stays hardcoded as `"lec10"`)
- Transcript sliding window / token budget management (full transcript up to current time is fine at ~18k tokens max)
- Video OCR transcript integration (only audio transcript)
- Persisting video playback position across sessions
- Any backend schema changes to Open WebUI's metadata allowlist

## Implementation Approach

Use a Svelte writable store as the bridge between the video player and chat submission. The store avoids prop-drilling through FileNav → ChatControls → Chat. The filter lives as a `.py` file in the repo for version control, pasted into the Open WebUI admin UI for deployment.

---

## Phase 1: VideoPlayer Store + Mux Lecture 10

### Overview
Create a `videoCurrentTime` store, wire up the `<mux-player>` element to update it on `timeupdate`, and swap in the real lecture 10 playback ID.

### Changes Required:

#### 1. Add store
**File**: `open-webui/src/lib/stores/index.ts`
**Changes**: Add a writable store for video current time after the existing `selectedTerminalId` store (line 101).

```typescript
export const videoCurrentTime: Writable<number> = writable(0);
```

#### 2. Update VideoPlayer to write to store
**File**: `open-webui/src/lib/components/chat/FileNav/VideoPlayer.svelte`
**Changes**: Bind the `<mux-player>` element, listen for `timeupdate` via `addEventListener`, write to store.

**Why `addEventListener` instead of Svelte's `on:timeupdate`**: This project uses Svelte 5, where `on:` is deprecated. The replacement `ontimeupdate={handler}` property syntax uses event delegation (listener at document root), which breaks for non-bubbling events. Mux-player re-dispatches `timeupdate` from shadow DOM as a `CustomEvent` without `bubbles: true`, so delegated listeners won't receive it. Direct `addEventListener` is the correct approach.

```svelte
<script lang="ts">
	import { onMount } from 'svelte';
	import { videoCurrentTime } from '$lib/stores';

	/** Mux playback ID. Takes precedence over src. */
	export let playbackId: string = '';
	/** Direct video URL (HLS or MP4) as fallback when no playbackId. */
	export let src: string = '';
	/** Video title for accessibility and Mux metadata. */
	export let title: string = '';
	/** CSS aspect ratio string. Default 16/9. */
	export let aspectRatio: string = '16 / 9';

	let playerEl: HTMLElement;

	function handleTimeUpdate() {
		if (playerEl) {
			videoCurrentTime.set((playerEl as any).currentTime ?? 0);
		}
	}

	onMount(async () => {
		await import('@mux/mux-player');
		if (playerEl) {
			playerEl.addEventListener('timeupdate', handleTimeUpdate);
		}
		return () => {
			if (playerEl) {
				playerEl.removeEventListener('timeupdate', handleTimeUpdate);
			}
			videoCurrentTime.set(0);
		};
	});
</script>

{#if playbackId || src}
	<div class="shrink-0 w-full bg-black">
		<mux-player
			bind:this={playerEl}
			playback-id={playbackId || undefined}
			src={!playbackId && src ? src : undefined}
			stream-type="on-demand"
			metadata-video-title={title}
			disable-tracking
			style="aspect-ratio: {aspectRatio}; width: 100%; display: block;"
		/>
	</div>
{/if}

<style>
	mux-player {
		--media-object-fit: contain;
	}
</style>
```

#### 3. Swap playback ID in FileNav
**File**: `open-webui/src/lib/components/chat/FileNav.svelte`
**Changes**: Update lines 631-634 to use the lecture 10 playback ID.

```svelte
<VideoPlayer
    playbackId="6vmvC02WC1xfKf7wobtERbRst801GOV5GcAHwkX9UaJoo"
    title="Lecture 10"
/>
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `cd open-webui && npm run build`
- [x] Store exports correctly: grep confirms `videoCurrentTime` in `stores/index.ts`

#### Manual Verification:
- [ ] Video plays lecture 10 content in the FileNav panel
- [ ] Open browser console, run: `window.__videoDebug = true` — or add a temporary `console.log` in `handleTimeUpdate` — confirm `videoCurrentTime` updates as video plays
- [ ] Pausing the video stops the store updates
- [ ] Navigating away resets the store to 0

**Implementation Note**: After completing this phase, pause for manual verification before proceeding.

---

## Phase 2: Thread Timestamp into Chat Request

### Overview
Import the `videoCurrentTime` store in Chat.svelte and include it in the chat completion request body as top-level fields.

### Changes Required:

#### 1. Add store import to Chat.svelte
**File**: `open-webui/src/lib/components/chat/Chat.svelte`
**Changes**: Add `videoCurrentTime` to the existing store import (line 17-49).

```typescript
import {
    // ... existing imports ...
    selectedTerminalId,
    showFileNavPath,
    showFileNavDir,
    videoCurrentTime  // add this
} from '$lib/stores';
```

#### 2. Add fields to request body
**File**: `open-webui/src/lib/components/chat/Chat.svelte`
**Changes**: Add `video_current_time` and `lecture_id` to the `generateOpenAIChatCompletion` call body, after the `terminal_id` field (line 2196). Only include when the video has been played (currentTime > 0).

At approximately line 2196, after `terminal_id: activeTerminalId ?? undefined,`:

```typescript
// Lecture video context
...($videoCurrentTime > 0
    ? {
        video_current_time: $videoCurrentTime,
        lecture_id: 'lec10'
    }
    : {}),
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `cd open-webui && npm run build`

#### Manual Verification:
- [ ] Play lecture video for ~30 seconds, send a chat message
- [ ] In browser DevTools Network tab, inspect the POST to `/api/chat/completions`
- [ ] Confirm request body contains `video_current_time` (number > 0) and `lecture_id: "lec10"` at the top level
- [ ] When video has NOT been played (currentTime = 0), confirm these fields are absent from the request

**Implementation Note**: After completing this phase, pause for manual verification before proceeding.

---

## Phase 3: Create the Lecture Context Filter

### Overview
Write a Python filter that reads `video_current_time` from the request body, calls the askademia-api `/context` endpoint, and injects the returned transcript into the system message.

### Changes Required:

#### 1. Create filter file
**File**: `askademia-api/filters/lecture_context_filter.py`
**Changes**: New file — the canonical version of the filter for version control. This gets pasted into the Open WebUI admin UI (Workspace > Functions > Create > Filter).

```python
"""
title: Askademia Lecture Context
author: askademia
version: 0.1
description: Injects lecture transcript context based on the current video timestamp.
"""

from pydantic import BaseModel, Field
from typing import Optional

import requests


class Filter:
    class Valves(BaseModel):
        priority: int = Field(
            default=0, description="Priority level for the filter operations."
        )
        askademia_api_url: str = Field(
            default="http://localhost:8000",
            description="Askademia API base URL.",
        )
        enabled: bool = Field(
            default=True, description="Enable lecture context injection."
        )

    def __init__(self):
        self.valves = self.Valves()

    def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        if not self.valves.enabled:
            return body

        current_time = body.get("video_current_time")
        lecture_id = body.get("lecture_id", "lec10")

        if current_time is None or current_time <= 0:
            return body

        # Fetch transcript from askademia-api
        try:
            resp = requests.post(
                f"{self.valves.askademia_api_url}/context",
                json={"lecture_id": lecture_id, "current_time": current_time},
                timeout=5,
            )
            resp.raise_for_status()
            transcript = resp.json()["transcript"]
        except Exception as e:
            print(f"[lecture-context-filter] Failed to fetch context: {e}")
            return body

        if not transcript:
            return body

        # Inject transcript into system message
        messages = body.get("messages", [])
        context_block = (
            "\n\n<lecture_transcript"
            f' lecture_id="{lecture_id}"'
            f' timestamp="{current_time:.1f}">\n'
            f"{transcript}\n"
            "</lecture_transcript>\n\n"
            "Use the lecture transcript above to answer questions about the lecture. "
            "Reference specific parts of the lecture when relevant."
        )

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

        body["messages"] = messages
        return body

    def outlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        return body
```

#### 2. Register the filter in Open WebUI
**Manual step**: In the Open WebUI admin UI:
1. Go to Workspace > Functions
2. Click "Create" > select "Filter"
3. Paste the contents of `askademia-api/filters/lecture_context_filter.py`
4. Save, then toggle it to active and global
5. Configure Valves: set `askademia_api_url` to `http://localhost:8000` (dev) or `http://host.docker.internal:8000` (Docker)

### Success Criteria:

#### Automated Verification:
- [x] Filter file is valid Python: `python3 -c "exec(open('askademia-api/filters/lecture_context_filter.py').read())"`
- [ ] askademia-api is running and healthy: `curl http://localhost:8000/health`
- [ ] askademia-api returns transcript: `curl -X POST http://localhost:8000/context -H 'Content-Type: application/json' -d '{"lecture_id":"lec10","current_time":300}'`

#### Manual Verification:
- [ ] Filter appears in Open WebUI Workspace > Functions list
- [ ] Filter is toggled on and set to global
- [ ] Play lecture video to ~5 minutes, ask "What has the professor been talking about?"
- [ ] LLM response references specific lecture content from the first ~5 minutes
- [ ] With video at 0:00 (not played), ask a question — LLM responds without lecture context (normal behavior)
- [ ] Check Open WebUI logs for any `[lecture-context-filter]` error messages

**Implementation Note**: After completing this phase, pause for full end-to-end manual verification.

---

## Testing Strategy

### End-to-End Test:
1. Start askademia-api (`make dev-api` or equivalent)
2. Start Open WebUI (`make dev-backend` + `make dev-frontend` or equivalent)
3. Ensure the filter is registered and active in the admin UI
4. Open a chat, play the lecture video
5. At ~2 minutes, ask: "What is the professor discussing?"
6. At ~10 minutes, ask the same question — response should include more content
7. Pause the video, send another message — should use the paused timestamp

### Edge Cases:
- Video not played (currentTime = 0) → no transcript injection
- askademia-api is down → filter logs error, chat works normally without transcript
- Very beginning of video (currentTime < 1s) → minimal/empty transcript, no crash
- Filter disabled via Valves → no transcript injection, normal chat behavior

## Files Changed (Summary)

| File | Change |
|------|--------|
| `open-webui/src/lib/stores/index.ts` | Add `videoCurrentTime` writable store |
| `open-webui/src/lib/components/chat/FileNav/VideoPlayer.svelte` | Bind mux-player, write to store on timeupdate |
| `open-webui/src/lib/components/chat/FileNav.svelte` | Swap playback ID to lecture 10 |
| `open-webui/src/lib/components/chat/Chat.svelte` | Import store, add `video_current_time` + `lecture_id` to request body |
| `askademia-api/filters/lecture_context_filter.py` | **New file** — filter Python code for version control |

## References

- Mux Player API: `.currentTime` property + `timeupdate` event (standard HTMLMediaElement)
- Lecture 10 Mux playback ID: `6vmvC02WC1xfKf7wobtERbRst801GOV5GcAHwkX9UaJoo`
- Lecture 10 Mux asset ID: `zp7ZCXIqvgfFHLJ5dErsOMZqiR5djFmhwnyPnUe6mbs`
- askademia-api POST /context: `askademia-api/askademia/main.py:43-52`
- Transcript loader: `askademia-api/askademia/transcripts.py`
- Filter execution: `open-webui/backend/open_webui/utils/filter.py:62-139`
- System message utility: `open-webui/backend/open_webui/utils/misc.py:356`
