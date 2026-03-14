# Hide Tab Header During Lecture Video — Implementation Plan

## Overview

Hide the "Controls / Files / Overview" tab bar in `ChatControls.svelte` when the Files tab is active with a terminal selected (which currently always renders the lecture video). This reclaims vertical space and reduces UI noise during lecture playback.

## Current State Analysis

The tab bar is rendered twice in `ChatControls.svelte` — once for mobile (line 292) and once for desktop (line 438). Each contains the same three conditional tab buttons plus a close (X) button. The `VideoPlayer` component is always rendered inside `FileNav.svelte:631-634` with a hardcoded `playbackId`, so whenever the Files tab is active with `$selectedTerminalId`, the video is present.

## Desired End State

When `activeTab === 'files'` and a terminal is selected, the tab buttons ("Controls", "Files", "Overview") are hidden. The close (X) button remains visible so the user can still dismiss the panel. When the user switches away from the Files tab (via code or future UI), the tab bar reappears normally.

**How to verify**: Open a chat with a terminal connected, open the controls panel → the video should appear with no tab buttons above it, only the close button. If you switch to a state without a terminal, the tab bar should return.

## What We're NOT Doing

- No new Svelte stores
- No changes to `VideoPlayer.svelte` or `FileNav.svelte`
- No changes to the store layer
- Not hiding the close (X) button

## Implementation Approach

Add one reactive declaration and wrap the tab button containers in both mobile and desktop views with `{#if !hideTabBar}`.

## Phase 1: Hide Tab Bar in ChatControls.svelte

### Overview
Single-file change to `ChatControls.svelte`.

### Changes Required:

#### 1. Add reactive declaration
**File**: `open-webui/src/lib/components/chat/ChatControls.svelte`
**Location**: After line 77 (after the existing `showOverviewTab` declaration)

Add:
```svelte
$: hideTabBar = activeTab === 'files' && !!$selectedTerminalId;
```

#### 2. Wrap mobile tab buttons (line 293)
**File**: `open-webui/src/lib/components/chat/ChatControls.svelte`
**Location**: Line 293 — the `<div class="flex gap-1 ...">` containing the tab buttons

Wrap the existing `<div class="flex gap-1 ...">...</div>` (lines 293-327) with:
```svelte
{#if !hideTabBar}
    <div class="flex gap-1 min-w-0 overflow-x-auto scrollbar-hidden">
        ... existing tab buttons ...
    </div>
{/if}
```

#### 3. Wrap desktop tab buttons (line 439)
**File**: `open-webui/src/lib/components/chat/ChatControls.svelte`
**Location**: Line 439 — the identical `<div class="flex gap-1 ...">` in the desktop pane

Same wrap:
```svelte
{#if !hideTabBar}
    <div class="flex gap-1 min-w-0 overflow-x-auto scrollbar-hidden">
        ... existing tab buttons ...
    </div>
{/if}
```

The close (X) button sits outside this div (at lines 328 and 474), so it remains visible.

### Success Criteria:

#### Automated Verification:
- [ ] Frontend builds without errors: `make dev-frontend` starts cleanly
- [ ] No TypeScript errors in `ChatControls.svelte`

#### Manual Verification:
- [ ] Open a chat with terminal connected → open controls panel → tab buttons are hidden, only close (X) button visible
- [ ] Video player takes full vertical space without tab bar above it
- [ ] Close (X) button still works to dismiss the panel
- [ ] Without a terminal selected, tab bar appears normally
- [ ] On mobile (narrow viewport), same behavior in the drawer view

## References

- Research: `thoughts/shared/research/2026-03-14-chatcontrols-tab-header-hiding.md`
