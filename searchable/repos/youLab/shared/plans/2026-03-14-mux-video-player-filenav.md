# Mux Video Player in FileNav Sidebar - Implementation Plan

## Overview

Add a Mux video player component to the FileNav sidebar in Open WebUI for Askademia's live lecture note-taking feature. The player sits above the FileNav toolbar, scales by width with aspect-ratio-based height, and remains visible regardless of file selection state. For this phase, we hardcode a demo Mux playback ID. The architecture is designed for future state management (store-driven video sources, module/lecture selection).

## Current State Analysis

- **FileNav.svelte** (`open-webui/src/lib/components/chat/FileNav.svelte`) is the main sidebar component with a vertical flex layout: `[DragOverlay] > FileNavToolbar > ContentArea(flex-1) > PortList > Terminal`
- **FilePreview.svelte** renders video files from the terminal filesystem using a plain HTML5 `<video>` tag — this is separate from lecture video playback
- **No Mux dependencies exist** in the project — `@mux/mux-player` needs to be installed
- **PortList hides** when `selectedFile` is set (`{#if selectedTerminal && !selectedFile}`) — the video player must NOT follow this pattern
- **The project uses Svelte 4** (`export let` props, not runes)

### Key Discoveries:
- FileNav layout is `flex flex-col h-full min-h-0 min-w-0` on the container (`FileNav.svelte:601`)
- FileNavToolbar is the first child after the drag overlay (`FileNav.svelte:630-880`)
- Content area is `flex-1 overflow-y-auto min-h-0 min-w-0` (`FileNav.svelte:883`)
- The `shrink-0` class is used by PortList and Terminal panels to claim fixed space
- Mux Player supports `style="aspect-ratio: 16/9; width: 100%;"` for width-driven sizing
- Dynamic `import('@mux/mux-player')` in `onMount` is needed for SSR safety in SvelteKit

## Desired End State

A `VideoPlayer.svelte` component renders above the FileNav toolbar when a video source is provided. It:
- Takes 100% of the sidebar width
- Maintains 16:9 aspect ratio (height adjusts with width)
- Stays visible during file preview mode (unlike PortList)
- Uses Mux Player web component (`<mux-player>`)
- Is hardcoded with a Mux demo playback ID for now
- Has props ready for future store-driven video sources

### Verification:
- The video player renders at the top of the FileNav sidebar
- Resizing the sidebar width causes the video height to adjust proportionally
- Opening a file for preview does NOT hide the video player
- The player has working playback controls (play, pause, seek, volume)

## What We're NOT Doing

- No backend changes (no FastAPI lecture context injection)
- No state management / stores for video sources or modules
- No lecture/module switching UI
- No integration with chat context injection
- No Mux Data analytics setup
- No changes to FilePreview.svelte's existing video handling (that's for terminal filesystem videos)

## Implementation Approach

Minimal, focused approach:
1. Install `@mux/mux-player` npm package
2. Create a small `VideoPlayer.svelte` component in the FileNav directory
3. Insert it into FileNav.svelte above the toolbar
4. Hardcode a Mux demo playback ID

The component is intentionally simple — it's a thin wrapper around `<mux-player>` with props for future extensibility.

## Phase 1: Install Mux Player Dependency

### Overview
Add the `@mux/mux-player` web component package to the project.

### Changes Required:

#### 1. Install npm package
```bash
cd open-webui && npm install @mux/mux-player
```

### Success Criteria:

#### Automated Verification:
- [x] Package installs without errors
- [x] `node_modules/@mux/mux-player` exists
- [x] `package.json` contains `@mux/mux-player` in dependencies

---

## Phase 2: Create VideoPlayer Component

### Overview
Create a self-contained `VideoPlayer.svelte` component that wraps `<mux-player>`.

### Changes Required:

#### 1. New component
**File**: `open-webui/src/lib/components/chat/FileNav/VideoPlayer.svelte`

```svelte
<script lang="ts">
	import { onMount } from 'svelte';

	/** Mux playback ID. Takes precedence over src. */
	export let playbackId: string = '';
	/** Direct video URL (HLS or MP4) as fallback when no playbackId. */
	export let src: string = '';
	/** Video title for accessibility and Mux metadata. */
	export let title: string = '';
	/** CSS aspect ratio string. Default 16/9. */
	export let aspectRatio: string = '16 / 9';

	onMount(async () => {
		await import('@mux/mux-player');
	});
</script>

{#if playbackId || src}
	<div class="shrink-0 w-full bg-black">
		<mux-player
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

**Key design decisions:**
- `shrink-0` prevents the flex container from squishing the player
- `w-full` makes it fill sidebar width
- `bg-black` provides background while loading
- `aspect-ratio` is set via inline style so it responds to width changes
- `disable-tracking` skips Mux analytics (not needed for demo)
- Props are ready for future: `playbackId`, `src`, `title`, `aspectRatio`
- The `{#if}` guard means the component renders nothing when no source is provided, making it safe to always include in FileNav

### Success Criteria:

#### Automated Verification:
- [x] File exists at `open-webui/src/lib/components/chat/FileNav/VideoPlayer.svelte`
- [x] No TypeScript errors: `cd open-webui && npx svelte-check --threshold error`

---

## Phase 3: Integrate VideoPlayer into FileNav

### Overview
Import and render VideoPlayer in FileNav.svelte, positioned above the FileNavToolbar. Hardcode a Mux demo playback ID.

### Changes Required:

#### 1. Import VideoPlayer
**File**: `open-webui/src/lib/components/chat/FileNav.svelte`
**Changes**: Add import alongside other FileNav sub-component imports

After `FileNav.svelte:41` (the PortList import), add:
```svelte
import VideoPlayer from './FileNav/VideoPlayer.svelte';
```

#### 2. Render VideoPlayer above FileNavToolbar
**File**: `open-webui/src/lib/components/chat/FileNav.svelte`
**Changes**: Insert VideoPlayer between the drag overlay (`{/if}` at line 628) and the FileNavToolbar (line 630)

Insert after the `{/if}` closing the drag overlay block (line 628), before `<FileNavToolbar`:
```svelte
		<VideoPlayer
			playbackId="EcHgOK9coz5K4rjSwOkoE7Y7O01201YMIC200RI6lNxnhs"
			title="Demo Lecture"
		/>
```

This uses Mux's public demo playback ID. The component's `{#if playbackId || src}` guard means it will render the player. When we later make this dynamic, setting the playbackId to empty string will hide the player.

**Note**: The VideoPlayer is rendered unconditionally (not gated by `!selectedFile` like PortList), so it remains visible when previewing files.

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript/Svelte errors: `cd open-webui && npx svelte-check --threshold error`
- [x] Build succeeds: `cd open-webui && npm run build`

#### Manual Verification:
- [ ] Video player appears at the top of the FileNav sidebar (above breadcrumbs/toolbar)
- [ ] Video plays with working controls (play, pause, seek, volume)
- [ ] Resizing the sidebar width causes the video height to adjust proportionally (16:9 ratio maintained)
- [ ] Clicking on a file to preview it does NOT hide the video player
- [ ] The file list and other FileNav content remain scrollable below the video
- [ ] Terminal panel at the bottom still works correctly
- [ ] PortList still appears/disappears based on its own logic (unchanged)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding.

---

## Testing Strategy

### Manual Testing Steps:
1. Open a chat with a terminal connection active
2. Open the Files tab in the sidebar
3. Verify the Mux video player appears at the top, above the breadcrumbs
4. Play the demo video — verify controls work
5. Resize the sidebar — verify video maintains aspect ratio
6. Click on a file to preview it — verify video stays visible above
7. Navigate back to directory listing — verify video still there
8. Open the terminal panel — verify both video and terminal coexist
9. On mobile/small screen (drawer mode) — verify video renders correctly

## Performance Considerations

- Mux Player loads HLS.js internally (~50KB gzipped) — this is loaded lazily via dynamic import in `onMount`, so it doesn't affect initial page load
- The demo playback ID uses Mux's CDN for adaptive bitrate streaming, so bandwidth is managed automatically
- No preload by default — video won't start buffering until user interaction

## Future Extensibility Notes

When ready to make this dynamic:
1. Add a store (e.g., `lectureVideoSource` in `stores/index.ts`) with `{ playbackId?, src?, title? }`
2. Subscribe to it in FileNav.svelte and pass values to VideoPlayer
3. Module/lecture selection UI sets the store
4. Empty store = no video player rendered (the `{#if}` guard handles this)
5. The FastAPI backend can provide playback IDs per module/lecture

## References

- Mux Player web component: `@mux/mux-player` (npm)
- [Mux Player docs](https://docs.mux.com/guides/player)
- [Mux SvelteKit integration](https://www.mux.com/video-for/sveltekit)
- Demo playback ID: `EcHgOK9coz5K4rjSwOkoE7Y7O01201YMIC200RI6lNxnhs` (Mux public example)
- Research: OpenTerminal integration architecture (provided in task context)
- Research: Mux Player in FileNav sidebar (provided in task context)
