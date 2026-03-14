---
date: 2026-03-14T00:00:00-05:00
researcher: ARI
git_commit: 470afd4214978159e7b45b41cfa6a640707918e0
branch: main
repository: Askademia
topic: "Hiding the Controls/Files/Overview tab header when a lecture video is loaded"
tags: [research, codebase, chatcontrols, filenav, videoplayer, tab-header, ui]
status: complete
last_updated: 2026-03-14
last_updated_by: ARI
---

# Research: Hiding the Controls/Files/Overview Tab Header When Lecture Video is Loaded

**Date**: 2026-03-14
**Researcher**: ARI
**Git Commit**: 470afd4214978159e7b45b41cfa6a640707918e0
**Branch**: main
**Repository**: Askademia

## Research Question
When the controls panel is open (with terminal/file nav), there is a header with tabs "Controls", "Files", "Overview". How can we hide this when the video player is loaded with a lecture to reduce visual noise and reclaim vertical space?

## Summary

The tab header is rendered in `ChatControls.svelte` (two instances: desktop and mobile). The `VideoPlayer` component lives inside `FileNav.svelte`, which is rendered as the content of the "Files" tab. The tab header has no awareness of whether a video is playing. The cleanest approach would be to either: (a) auto-force the "files" tab and hide the tab bar when a lecture video is active, or (b) add a collapsible/auto-hide behavior to the tab bar based on video playback state from the existing `videoCurrentTime` store.

## Detailed Findings

### 1. Where the Tab Header is Rendered

The tab header lives in **`ChatControls.svelte`** at two locations (desktop and mobile):

- **Mobile/Drawer view** — `open-webui/src/lib/components/chat/ChatControls.svelte:289-344`
  - Line 289: `<!-- Controls + Files tabs -->`
  - Lines 292-327: Tab bar with three conditional buttons
  - The tab bar container: `<div class="flex items-center justify-between px-2 pt-2.5 pb-2 shrink-0">`

- **Desktop/Pane view** — `open-webui/src/lib/components/chat/ChatControls.svelte:435-490`
  - Line 435: `<!-- Controls + Files tabs -->`
  - Lines 438-473: Identical tab bar structure
  - Same container styling

Each tab button is conditionally rendered:
```svelte
{#if showControlsTab}  <!-- Controls button -->
{#if showFilesTab}     <!-- Files button -->
{#if showOverviewTab}  <!-- Overview button -->
```

The tab bar also includes a close button (X) on the right side.

### 2. What Controls Tab Visibility

Tab visibility is driven by reactive declarations in `ChatControls.svelte:73-90`:

```svelte
$: showControlsTab = $user?.role === 'admin' || ($user?.permissions?.chat?.controls ?? true);
$: showFilesTab = !!$selectedTerminalId || (codeInterpreterEnabled && $config?.code?.interpreter_engine !== 'jupyter');
$: showOverviewTab = hasMessages;
```

- **Controls tab**: Shown if user is admin OR has `chat.controls` permission
- **Files tab**: Shown if a terminal is selected OR code interpreter is enabled (non-Jupyter)
- **Overview tab**: Shown if the chat has messages

Auto-switching fallback logic (lines 80-85): if the active tab becomes hidden, it falls back to the next available tab.

Auto-close logic (lines 88-90): if no tabs are visible at all, the entire panel closes.

### 3. How the Video Player Relates to the Controls/Files Panel

The component hierarchy is:

```
Chat.svelte
  └── ChatControls.svelte (tab header + tab content container)
        └── [when activeTab === 'files' && $selectedTerminalId]
              └── FileNav.svelte
                    └── VideoPlayer.svelte (hardcoded at line 631-634)
                          └── <mux-player> web component
```

Key details:
- `VideoPlayer` is rendered **inside** `FileNav.svelte` at line 631-634, unconditionally (always present when FileNav mounts):
  ```svelte
  <VideoPlayer
      playbackId="6vmvC02WC1xfKf7wobtERbRst801GOV5GcAHwkX9UaJoo"
      title="Lecture 10"
  />
  ```
- The video player sits **above** the `FileNavToolbar` (line 636), which is the breadcrumb/file toolbar
- `VideoPlayer.svelte` updates the `videoCurrentTime` store on every `timeupdate` event
- The `videoCurrentTime` store is defined in `open-webui/src/lib/stores/index.ts:102`
- `Chat.svelte` reads `$videoCurrentTime` to inject lecture context into API requests (lines 2198-2204)

The tab header and the video player are separated by several component layers — ChatControls renders the tab bar, then conditionally renders FileNav as tab content, and FileNav renders VideoPlayer inside itself.

### 4. Approaches to Conditionally Hide the Tab Header

**Approach A: Use `videoCurrentTime` store to detect active lecture playback**

The `videoCurrentTime` store already exists and is non-zero whenever a lecture video is playing. `ChatControls.svelte` could import it and conditionally hide the tab bar:

```svelte
// In ChatControls.svelte
import { videoCurrentTime } from '$lib/stores';

// New reactive: is a lecture video actively loaded?
$: lectureVideoActive = $videoCurrentTime > 0 || hasLectureVideo;
```

Then wrap the tab bar div with `{#if !lectureVideoActive}` or apply a CSS class to collapse it.

**Pros**: Minimal change, uses existing store, no new state needed.
**Cons**: `videoCurrentTime` is 0 when video is paused at 0:00 or before first play — may need a separate boolean like a `videoLoaded` store.

**Approach B: Add a new `lectureVideoLoaded` store**

Create a new store (e.g., `lectureVideoLoaded: Writable<boolean>`) that `VideoPlayer.svelte` sets to `true` on mount and `false` on destroy. `ChatControls.svelte` reads this to hide the tab bar.

**Pros**: Clear semantic meaning, doesn't depend on playback position.
**Cons**: Adds a new store.

**Approach C: Auto-force "files" tab and collapse header when video is present**

When a lecture is loaded, automatically set `activeTab = 'files'` and hide the tab bar entirely since the user doesn't need to switch tabs during a lecture. Could still keep the close (X) button visible.

**Pros**: Cleanest UX — removes noise without losing the ability to close the panel.
**Cons**: User loses ability to switch to Controls/Overview while video is showing.

**Approach D: CSS-only collapse with transition**

Instead of removing the tab bar, reduce its height to 0 with a CSS transition when the video is active. The close button could remain accessible via an alternative mechanism (e.g., clicking outside).

**Pros**: Smooth visual transition, no layout jump.
**Cons**: Still needs a reactive signal to trigger the CSS change.

### Recommended Approach

**Approach B + C combined**: Add a `lectureVideoLoaded` boolean store. In `ChatControls.svelte`, when `lectureVideoLoaded` is true, auto-set `activeTab = 'files'` and hide the tab bar (keeping only the close button). This is the cleanest because:
1. It uses clear, semantic state (not a proxy like `videoCurrentTime > 0`)
2. It reduces noise by removing the unused tab headers
3. It preserves the close button so the user can still dismiss the panel
4. The implementation touches only 3 files: `stores/index.ts` (new store), `VideoPlayer.svelte` (set on mount/destroy), `ChatControls.svelte` (read and conditionally hide)

## Code References

- `open-webui/src/lib/components/chat/ChatControls.svelte:2` — `savedTab` module-level state
- `open-webui/src/lib/components/chat/ChatControls.svelte:64-96` — Tab state management and reactive logic
- `open-webui/src/lib/components/chat/ChatControls.svelte:289-344` — Mobile tab bar rendering
- `open-webui/src/lib/components/chat/ChatControls.svelte:435-490` — Desktop tab bar rendering
- `open-webui/src/lib/components/chat/ChatControls.svelte:499-519` — Tab content rendering (desktop)
- `open-webui/src/lib/components/chat/FileNav.svelte:631-634` — VideoPlayer instantiation (hardcoded playbackId)
- `open-webui/src/lib/components/chat/FileNav/VideoPlayer.svelte:1-55` — Complete VideoPlayer component
- `open-webui/src/lib/stores/index.ts:102` — `videoCurrentTime` store definition
- `open-webui/src/lib/components/chat/Chat.svelte:2198-2204` — Video context sent to API

## Architecture Documentation

The controls panel uses a **tab-based architecture** inside `ChatControls.svelte`:
- State is managed at module level (`savedTab`) so it persists across mount/unmount cycles
- Tab visibility is reactive and derived from user permissions, terminal selection, and chat state
- The panel itself uses `paneforge` for resizable desktop layout and `Drawer` for mobile
- Content switching uses Svelte `{#if}` blocks based on `activeTab`

The video player is **deeply nested** inside the Files tab content:
- `ChatControls` → `FileNav` → `VideoPlayer`
- The player communicates upward only via the `videoCurrentTime` store
- There is currently no mechanism for the player to influence the tab bar's visibility

## Open Questions

1. Should the tab header hiding be tied to a specific lecture being loaded, or any video being present?
2. Should the close (X) button remain visible when the tab bar is hidden, or should there be an alternative way to close the panel?
3. Should the tab bar auto-show again when the video is paused/ended, or only when the video component unmounts?
