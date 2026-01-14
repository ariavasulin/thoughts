# ARI-83: Remove Rounded Corners from Diff View

## Overview

Remove rounded corners from diff view containers to create a cleaner, more code-editor-like appearance.

## Size: XS

Two single-line CSS class changes.

## Affected Files

| File | Line | Change |
|------|------|--------|
| `OpenWebUI/open-webui/src/lib/components/you/DiffApprovalOverlay.svelte` | 225 | Remove `rounded-lg` |
| `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte` | 269 | Remove `rounded-lg` |

## Implementation

### Phase 1: DiffApprovalOverlay.svelte (Line 225)

**Before:**
```html
<div class="absolute inset-0 flex flex-col bg-white/95 dark:bg-gray-900/95 backdrop-blur-sm rounded-lg border border-gray-200 dark:border-gray-700 shadow-lg z-20 overflow-hidden">
```

**After:**
```html
<div class="absolute inset-0 flex flex-col bg-white/95 dark:bg-gray-900/95 backdrop-blur-sm border border-gray-200 dark:border-gray-700 shadow-lg z-20 overflow-hidden">
```

### Phase 2: BlockDetailModal.svelte (Line 269)

**Before:**
```html
<div class="rounded-lg border border-gray-200 dark:border-gray-700 overflow-hidden w-full">
```

**After:**
```html
<div class="border border-gray-200 dark:border-gray-700 overflow-hidden w-full">
```

## Verification

1. Open a memory block with pending diffs
2. Verify the diff container has sharp corners
3. Open the full-screen diff approval overlay
4. Verify the overlay container has sharp corners

## Success Criteria

- [ ] DiffApprovalOverlay container has no border-radius
- [ ] BlockDetailModal diff container has no border-radius
- [ ] Visual appearance matches code editor aesthetic
