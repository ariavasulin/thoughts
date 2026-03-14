# Hide Tab Header During Lecture Video — Implementation Plan

## Overview

Hide the entire "Controls / Files / Overview" tab bar (including close button) in `ChatControls.svelte` when the Files tab is active with a terminal selected (which currently always renders the lecture video). This reclaims vertical space and reduces UI noise during lecture playback.

## Current State Analysis

The tab bar is rendered twice in `ChatControls.svelte` — once for mobile (line 292) and once for desktop (line 438). Each contains the same three conditional tab buttons plus a close (X) button inside a single container div. The `VideoPlayer` component is always rendered inside `FileNav.svelte:631-634` with a hardcoded `playbackId`, so whenever the Files tab is active with `$selectedTerminalId`, the video is present.

## Desired End State

When `activeTab === 'files'` and a terminal is selected, the entire tab bar container (buttons + close button + padding) is hidden. The video player sits flush at the top of the panel. The panel can still be closed via the pane resizer (desktop) or clicking outside the drawer (mobile).

**How to verify**: Open a chat with a terminal connected, open the controls panel → the video should appear at the top with no tab bar above it. Without a terminal, the tab bar appears normally.

## What We're NOT Doing

- No new Svelte stores
- No changes to `VideoPlayer.svelte` or `FileNav.svelte`
- No changes to the store layer

## Implementation Approach

Add one reactive declaration and wrap the entire tab bar container div in both mobile and desktop views with `{#if !hideTabBar}`.

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

#### 2. Wrap mobile tab bar container (line 292)
**File**: `open-webui/src/lib/components/chat/ChatControls.svelte`
**Location**: Line 292 — the outer `<div class="flex items-center justify-between px-2 pt-2.5 pb-2 shrink-0">` containing both the tab buttons and the close button

Wrap the entire container (lines 292-344) with `{#if !hideTabBar}`:
```svelte
{#if !hideTabBar}
    <div class="flex items-center justify-between px-2 pt-2.5 pb-2 shrink-0">
        ... tab buttons + close button ...
    </div>
{/if}
```

#### 3. Wrap desktop tab bar container (line 438)
**File**: `open-webui/src/lib/components/chat/ChatControls.svelte`
**Location**: Line 438 — the identical container in the desktop pane

Same wrap (lines 438-490):
```svelte
{#if !hideTabBar}
    <div class="flex items-center justify-between px-2 pt-2.5 pb-2 shrink-0">
        ... tab buttons + close button ...
    </div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] Frontend builds without errors: `make dev-frontend` starts cleanly
- [ ] No TypeScript errors in `ChatControls.svelte`

#### Manual Verification:
- [ ] Open a chat with terminal connected → open controls panel → entire tab bar is gone, video sits at top
- [ ] Video player takes full vertical space without any bar above it
- [ ] Panel can still be closed via pane resizer drag (desktop) or tapping outside drawer (mobile)
- [ ] Without a terminal selected, tab bar appears normally
- [ ] On mobile (narrow viewport), same behavior in the drawer view

## References

- Research: `thoughts/shared/research/2026-03-14-chatcontrols-tab-header-hiding.md`
