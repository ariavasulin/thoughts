# PDF Viewer Auto-Resize on Container Change

## Overview

Add a `ResizeObserver` to `PDFViewer.svelte` so the PDF automatically re-renders to fit the available space when the container is resized (e.g., dragging the PaneResizer, window resize). On resize end, panzoom resets to origin and all canvases re-render at the new container width.

## Current State Analysis

`PDFViewer.svelte` renders PDF pages as canvases scaled to `outerContainer.clientWidth` at load time, but has no mechanism to detect or respond to container size changes. The `rerenderPages()` function re-reads `clientWidth` when called but is only triggered by panzoom zoom events. Additionally, `rerenderPages()` updates canvas bitmap dimensions but not CSS dimensions (`canvas.style.width`/`canvas.style.height`), which would cause layout issues if called after a container resize.

### Key Discoveries:
- `rerenderPages()` does not update `canvas.style.width`/`canvas.style.height` (`PDFViewer.svelte:99-101`)
- XTerminal uses `ResizeObserver` + `requestAnimationFrame` + `FitAddon.fit()` as a working pattern (`XTerminal.svelte:200-242`)
- Paneforge's `onDraggingChange` could detect drag-end, but a self-contained `ResizeObserver` approach is simpler and handles all resize sources

## Desired End State

When the PDFViewer's container changes size (pane drag, window resize, etc.), the PDF re-renders to fit the new width after a 300ms debounce. Panzoom resets to origin. Text remains crisp.

**Verify by:**
1. Open a PDF in FileNav
2. Drag the PaneResizer — on release, PDF re-fits within ~300ms
3. Resize browser window — PDF re-fits
4. No blurry text, no stale pan/zoom offsets

## What We're NOT Doing

- No intermediate visual updates during drag (CSS-only resize) — just a single re-render on debounce
- No changes to FilePreview, ChatControls, or any other component
- No paneforge `onDraggingChange` wiring

## Implementation Approach

Self-contained change in `PDFViewer.svelte` only. Add a `ResizeObserver` that debounces and triggers a full re-render at zoom=1 with panzoom reset.

## Phase 1: Add ResizeObserver and Fix rerenderPages

### Overview
Add container resize detection with debounced re-render, fix missing CSS dimension updates in `rerenderPages()`.

### Changes Required:

#### 1. Fix `rerenderPages()` to update CSS dimensions

**File**: `open-webui/src/lib/components/common/PDFViewer.svelte`
**Location**: Inside the `for` loop in `rerenderPages()`, after line 101

Add CSS dimension updates so the canvas layout matches the new container width:

```typescript
// After canvas.height = scaledViewport.height; (line 101)
canvas.style.width = `${Math.round(cssScale * viewport.width)}px`;
canvas.style.height = `${Math.round(cssScale * viewport.height)}px`;
```

This mirrors what `renderAllPages()` already does at lines 133-134.

#### 2. Add `resizeToFit()` function

**File**: `open-webui/src/lib/components/common/PDFViewer.svelte`
**Location**: After the `resetView()` function (after line 82)

```typescript
// Re-fit PDF to container after a resize (resets panzoom + re-renders at 1x)
const resizeToFit = () => {
    if (pzInstance) {
        pzInstance.moveTo(0, 0);
        pzInstance.zoomAbs(0, 0, 1);
        zoomLevel = 1;
    }
    rerenderPages(1);
};
```

#### 3. Add ResizeObserver with debounce

**File**: `open-webui/src/lib/components/common/PDFViewer.svelte`
**Location**: Add `resizeObserver` and `resizeTimer` variables near existing state (around line 18-19), and set up observer in `onMount`

New variables:
```typescript
let resizeObserver: ResizeObserver | null = null;
let resizeTimer: ReturnType<typeof setTimeout> | null = null;
```

Update `onMount` to create the observer after initial load:
```typescript
onMount(() => {
    loadPdf().then(() => {
        if (!outerContainer) return;
        resizeObserver = new ResizeObserver(() => {
            if (resizeTimer) clearTimeout(resizeTimer);
            resizeTimer = setTimeout(() => {
                resizeToFit();
            }, 300);
        });
        resizeObserver.observe(outerContainer);
    });
});
```

Note: The observer is created *after* `loadPdf()` completes to avoid triggering a resize during the initial render.

#### 4. Clean up in `onDestroy`

**File**: `open-webui/src/lib/components/common/PDFViewer.svelte`
**Location**: Existing `onDestroy` block (line 187-194)

Add cleanup for the new resources:
```typescript
onDestroy(() => {
    if (resizeTimer) clearTimeout(resizeTimer);
    if (rerenderTimer) clearTimeout(rerenderTimer);
    resizeObserver?.disconnect();
    pzInstance?.dispose();
    if (pdfDoc) {
        pdfDoc.destroy();
        pdfDoc = null;
    }
});
```

### Success Criteria:

#### Automated Verification:
- [x] Frontend dev server starts without errors: `make dev-frontend`

#### Manual Verification:
- [ ] Open a PDF in the FileNav panel (controls pane)
- [ ] Drag the PaneResizer to make the pane wider → on release, PDF re-renders to fit new width within ~300ms
- [ ] Drag the PaneResizer to make the pane narrower → same behavior
- [ ] Resize the browser window → PDF re-fits
- [ ] After resize, panzoom is reset (no leftover pan offset or zoom)
- [ ] Text is crisp after re-render (not blurry/stretched)
- [ ] Zoom (ctrl+scroll) still works correctly after a resize
- [ ] No console errors during resize operations

## References

- Research: `thoughts/shared/research/2026-03-15-pdf-viewer-resize-behavior.md`
- XTerminal resize pattern: `open-webui/src/lib/components/chat/XTerminal.svelte:200-242`
- PDFViewer component: `open-webui/src/lib/components/common/PDFViewer.svelte`
