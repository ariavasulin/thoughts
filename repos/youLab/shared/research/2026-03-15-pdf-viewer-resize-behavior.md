---
date: 2026-03-15T12:00:00-05:00
researcher: ARI
git_commit: b6f1a40a6b4e98458cbe20f8b706b74aa672cbba
branch: main
repository: Askademia
topic: "How does the PDF viewer in FileNav resize and fit to available space?"
tags: [research, codebase, pdf-viewer, filenav, resize, pdfjs, panzoom]
status: complete
last_updated: 2026-03-15
last_updated_by: ARI
---

# Research: PDF Viewer Resize and Fit Behavior in FileNav

**Date**: 2026-03-15
**Researcher**: ARI
**Git Commit**: b6f1a40a6b4e98458cbe20f8b706b74aa672cbba
**Branch**: main
**Repository**: Askademia

## Research Question

How does the current PDF viewer in the nav files/terminal view on OWUI refresh and resize the PDF to the right size given the space? This will be used as context to plan how to make an autoresize and fit-to-space when the component is resized.

## Summary

The PDF viewer renders each page as an HTML canvas scaled to fit the container width at load time, but **does not currently auto-resize when the container changes size** (e.g., when the pane is resized by dragging the PaneResizer). The initial render captures `outerContainer.clientWidth` once and uses that to compute a CSS scale. Subsequent container resizes are not observed — there is no `ResizeObserver` on the PDF viewer's container. The panzoom library handles interactive zoom/pan after initial render, with debounced re-rendering at higher resolution when the user zooms. This contrasts with the XTerminal component, which uses a `ResizeObserver` + `FitAddon` to continuously fit to its container.

## Detailed Findings

### 1. PDFViewer Component (`open-webui/src/lib/components/common/PDFViewer.svelte`)

**Initialization flow:**

1. `onMount` calls `loadPdf()` (line 183-185)
2. `loadPdf()` dynamically imports `pdfjs-dist`, fetches/reads the PDF data, then calls `renderAllPages()` (line 154-181)
3. `renderAllPages()` renders every page into canvases appended to `sceneElement` (line 111-152)

**How initial sizing works (line 111-152):**

For each page:
- Gets the PDF page's native viewport at scale=1: `page.getViewport({ scale: 1 })`
- Reads container width: `const containerWidth = outerContainer?.clientWidth || 800`
- Computes CSS scale to fit: `const cssScale = containerWidth / viewport.width`
- Computes render scale for crisp pixels: `const renderScale = cssScale * dpr` (where `dpr = window.devicePixelRatio`)
- Creates canvas at high-res (`scaledViewport.width × scaledViewport.height`)
- Sets CSS dimensions to CSS-pixel size: `canvas.style.width = cssScale * viewport.width + 'px'` / `canvas.style.height = cssScale * viewport.height + 'px'`
- This means the canvas bitmap is `dpr`× larger than the CSS size for retina sharpness

**Key observation: `containerWidth` is read once at render time.** There is no `ResizeObserver`, no `bind:clientWidth`, and no `resize` event listener to trigger re-rendering when the container changes size.

**Panzoom integration (line 21-57):**

After all pages render, `initPanzoom()` is called. It wraps `sceneElement` (the div containing all canvases) with the `panzoom` library:
- Zoom only on pinch (ctrl/meta + wheel)
- Pan only when zoomed in beyond 1x
- On zoom, debounces a 300ms `rerenderPages()` call to re-render canvases at the new zoom level for crisp text

**Re-render on zoom (line 85-109):**

`rerenderPages(forZoom)` re-reads `outerContainer.clientWidth` and re-renders all canvases at `cssScale * forZoom * dpr`. This means if the user zooms after a resize, the new container width is picked up. But without zoom interaction, stale sizing persists.

**Exported API:**
- `resetView()` — resets panzoom to origin and re-renders at zoom=1 (line 75-82)

**DOM structure (line 197-259):**
```html
<div class="relative {className}">  <!-- className default: 'w-full h-[70vh]' -->
  <div class="overflow-y-auto h-full" bind:this={outerContainer}>
    <div bind:this={sceneElement} class="w-full">
      <!-- canvases appended here by JS -->
    </div>
  </div>
  <!-- zoom controls overlay at bottom-center -->
</div>
```

The `className` prop controls the outer container's size. When used in FilePreview, it's overridden to `"w-full h-full"`.

### 2. FilePreview Container (`open-webui/src/lib/components/chat/FileNav/FilePreview.svelte`)

**How PDFViewer is mounted (line 311-312):**
```svelte
{:else if filePdfData !== null}
    <PDFViewer bind:this={pdfViewerRef} data={filePdfData} className="w-full h-full" />
```

FilePreview passes `className="w-full h-full"` so the PDF viewer fills the available flex space.

**FilePreview's own sizing (line 281-284):**
```svelte
<div class="flex-1 {overflow_class} min-h-0 min-w-0 relative h-full">
```
This is a flex child that grows to fill remaining space, with `min-h-0` / `min-w-0` to allow shrinking.

**Exported methods (line 272-274):**
```javascript
export const resetPdfView = () => {
    pdfViewerRef?.resetView();
};
```

### 3. Container Hierarchy (Chat → ChatControls → FileNav → FilePreview)

**Layout chain:**

```
Chat.svelte
  └─ PaneGroup (horizontal, paneforge)
       ├─ Pane (main chat)
       ├─ PaneResizer (draggable)
       └─ Pane (ChatControls)
            └─ ChatControls.svelte
                 └─ Tab: "files"
                      └─ FileNav.svelte
                           ├─ VideoPlayer (shrink-0, top)
                           ├─ FileNavToolbar
                           └─ FilePreview (flex-1, fills remaining)
                                └─ PDFViewer (w-full h-full)
```

**Pane resizing (ChatControls.svelte):**

ChatControls uses a `ResizeObserver` on `#chat-container` to enforce minimum pane width (350px). The `onResize` handler saves the pixel size to `localStorage.chatControlsSize`. However, this ResizeObserver **does not propagate resize events to child components** like PDFViewer.

When the user drags the `PaneResizer`:
1. Paneforge resizes the Pane element
2. ChatControls' ResizeObserver fires to enforce min-size
3. CSS flex layout flows the new width down to FileNav → FilePreview → PDFViewer
4. The PDFViewer's `outerContainer` gets a new `clientWidth`
5. But **nothing triggers PDFViewer to re-render** — the canvases retain their original pixel dimensions

### 4. Comparison with XTerminal Resize Pattern

The XTerminal component (`open-webui/src/lib/components/chat/XTerminal.svelte:200-242`) demonstrates the auto-resize pattern that PDFViewer currently lacks:

```javascript
// XTerminal approach:
fitAddon = new FitAddon();
term.loadAddon(fitAddon);

requestAnimationFrame(() => { fitAddon?.fit(); });

resizeObserver = new ResizeObserver(() => {
    requestAnimationFrame(() => { fitAddon?.fit(); });
});
resizeObserver.observe(terminalEl);
```

Key differences:
- XTerminal has its own `ResizeObserver` on its container element
- On every resize, it calls `fitAddon.fit()` wrapped in `requestAnimationFrame`
- The fit call recalculates terminal columns/rows based on new container dimensions

### 5. Other Resize Patterns in the Codebase

| Component | Resize Mechanism | Target |
|-----------|-----------------|--------|
| ChatControls | `ResizeObserver` on `#chat-container` | Enforce min pane width |
| XTerminal | `ResizeObserver` on `terminalEl` | Refit terminal to container |
| KnowledgeBase | `ResizeObserver` on container | Adjust pane min-size % |
| NotePanel | `ResizeObserver` on container | Adjust pane min-size % |
| VoiceRecording | `ResizeObserver` on `document.body` + `bind:clientWidth` | Adjust visualizer buffer |
| VideoPlayer | CSS `width: 100%; aspect-ratio` | Fluid resize via CSS |
| PDFViewer | **None** | Renders once at initial width |

## Code References

- `open-webui/src/lib/components/common/PDFViewer.svelte:111-152` — `renderAllPages()` initial render with container width
- `open-webui/src/lib/components/common/PDFViewer.svelte:85-109` — `rerenderPages()` zoom-level re-render
- `open-webui/src/lib/components/common/PDFViewer.svelte:21-57` — panzoom initialization
- `open-webui/src/lib/components/common/PDFViewer.svelte:183-185` — `onMount` → `loadPdf()`
- `open-webui/src/lib/components/common/PDFViewer.svelte:197-210` — DOM structure
- `open-webui/src/lib/components/chat/FileNav/FilePreview.svelte:311-312` — PDFViewer usage with `className="w-full h-full"`
- `open-webui/src/lib/components/chat/FileNav/FilePreview.svelte:281-284` — flex container sizing
- `open-webui/src/lib/components/chat/ChatControls.svelte:191-230` — ResizeObserver for pane min-size
- `open-webui/src/lib/components/chat/XTerminal.svelte:200-242` — ResizeObserver + FitAddon pattern

## Architecture Documentation

**PDF rendering pipeline:** pdfjs-dist loads the PDF → iterates pages → for each page, creates a canvas element sized to `(containerWidth / nativeWidth) * dpr` pixels, with CSS size set to `containerWidth / nativeWidth * nativeWidth` (i.e., container width) × proportional height. All canvases are appended to a scrollable div. Panzoom wraps the canvas container for interactive zoom/pan.

**Key sizing variables:**
- `containerWidth` = `outerContainer.clientWidth` (read once during render)
- `cssScale` = `containerWidth / viewport.width` (fits page width to container)
- `renderScale` = `cssScale * dpr` (high-res for retina)
- `canvas.style.width` = `cssScale * viewport.width` px (always equals containerWidth)
- `canvas.style.height` = `cssScale * viewport.height` px (proportional)

**Current gap:** No mechanism exists to detect when `outerContainer.clientWidth` changes after initial render and trigger `rerenderPages()` or `renderAllPages()`. The existing `rerenderPages()` function already reads `outerContainer.clientWidth` fresh each call, so it would pick up new dimensions if invoked.

## Open Questions

- Should auto-resize debounce or throttle to avoid excessive re-rendering during drag?
- Should the panzoom transform be preserved or reset on container resize?
- Should the canvas CSS dimensions update immediately (fast visual resize) with a debounced full re-render (for crisp text)?
- Does `rerenderPages(1)` suffice or does a full `renderAllPages()` need to be called on resize (the former reuses existing canvases, the latter recreates them)?
