# Plan: Full-Screen PDF Viewer in Artifact Panel

## Goal
Make the PDF viewer fill the entire artifact pane — no horizontal or vertical margins. The PDF should use the full width of the pane with scrollable pages stacked vertically.

## Current Problem
The PDF viewer in `latex_templates.py` renders a single `<canvas>` element centered in a `.viewer` div with `padding: 20px` and `justify-content: center`. The canvas has a fixed scale (`1.5`) and renders one page at a time with prev/next navigation. This wastes space — the PDF page is small and centered with large margins.

## Root Cause Analysis

Three issues combine to produce the small, padded PDF:

1. **`.viewer` has `padding: 20px` and `justify-content: center`** — adds whitespace around the canvas
2. **Single-page rendering** — only one `<canvas>` exists, showing one page at a time (prev/next buttons)
3. **Fixed scale (`1.5`)** — doesn't adapt to container width; the PDF page width is `pageWidth * 1.5` regardless of how wide the pane is

## Design Decision: Fit-to-Width vs Multi-Page Scroll

**Option A: Keep single-page, fit to width** — Remove padding, auto-scale to container width
**Option B: Continuous scroll, fit to width** — Render all pages stacked vertically, each fitting container width

**Chosen: Option B** — Continuous scroll is the standard for PDF viewers (like browser native, Google Docs, etc.) and provides a better reading experience. The toolbar simplifies too — no prev/next buttons needed, just zoom and download.

## Implementation

### Single file change: `src/ralph/tools/latex_templates.py`

Replace the `PDF_VIEWER_TEMPLATE` with a new version that:

#### CSS Changes
1. **Remove `.viewer` padding** — `padding: 0` instead of `padding: 20px`
2. **Remove `justify-content: center`** — left-align content or use `align-items: center` for horizontal centering only
3. **Change `.viewer` to vertical scroll container** — `flex-direction: column; align-items: center; overflow-y: auto; gap: 0;`
4. **Canvas/page elements fill width** — each page canvas should be `width: 100%` of the viewer
5. **Remove box-shadow from canvas** — it adds visual "margin" effect
6. **Remove prev/next buttons** from toolbar — not needed with continuous scroll
7. **Keep zoom controls and download button** in toolbar
8. **Add page separator** — thin subtle line or small gap between pages

#### JavaScript Changes
1. **Render ALL pages on load** — iterate through `pdfDoc.numPages`, create a `<canvas>` for each page
2. **Fit-to-width scaling** — calculate scale as `containerWidth / pageWidth` so each page fills the viewer width exactly
3. **Recalculate on resize** — listen for `ResizeObserver` on `.viewer` to re-render when pane resizes
4. **Zoom adjusts the fit-to-width base** — zoom in/out multiplies the fit-to-width scale
5. **Update page counter on scroll** — use `IntersectionObserver` to track which page is visible and update the toolbar page indicator

#### Toolbar Simplification
- Remove: Prev/Next buttons
- Keep: Page indicator (updates on scroll), Zoom +/-, Zoom level display, Download button
- Add: "Fit width" reset button (resets zoom to 1x fit-to-width)

### Detailed Template Structure

```html
<!-- Toolbar -->
<div class="toolbar">
    <span class="page-info">Page <span id="page-num">1</span> / <span id="page-count">-</span></span>
    <div class="zoom-controls">
        <button id="zoom-out">−</button>
        <button id="zoom-fit">Fit</button>
        <span id="zoom-level">100%</span>
        <button id="zoom-in">+</button>
    </div>
    <div class="spacer"></div>
    <button id="download" class="download-btn">Download PDF</button>
</div>

<!-- Viewer: scrollable container with all pages -->
<div class="viewer" id="viewer">
    <!-- Canvases inserted dynamically by JS -->
</div>
```

```css
.viewer {
    flex: 1;
    overflow-y: auto;
    overflow-x: hidden;
    display: flex;
    flex-direction: column;
    align-items: center;  /* center pages if zoomed smaller than container */
    background: #525659;
}
.page-canvas {
    display: block;
    width: 100%;  /* at 1x zoom, fill container width */
    background: white;
}
```

```javascript
// Key logic:
var baseScale = 1;  // Will be computed as containerWidth / pageWidth
var zoomFactor = 1;  // User zoom multiplier

function computeBaseScale(page) {
    var viewport = page.getViewport({ scale: 1 });
    var containerWidth = document.getElementById('viewer').clientWidth;
    return containerWidth / viewport.width;
}

async function renderAllPages() {
    var viewer = document.getElementById('viewer');
    viewer.innerHTML = '';

    var firstPage = await pdfDoc.getPage(1);
    baseScale = computeBaseScale(firstPage);
    var scale = baseScale * zoomFactor;

    for (var i = 1; i <= pdfDoc.numPages; i++) {
        var page = await pdfDoc.getPage(i);
        var viewport = page.getViewport({ scale: scale });
        var canvas = document.createElement('canvas');
        canvas.className = 'page-canvas';
        canvas.width = viewport.width;
        canvas.height = viewport.height;
        viewer.appendChild(canvas);
        var ctx = canvas.getContext('2d');
        await page.render({ canvasContext: ctx, viewport: viewport }).promise;
    }
}

// ResizeObserver for pane resize
new ResizeObserver(() => renderAllPages()).observe(viewer);

// IntersectionObserver for page tracking
// ... observe each canvas, update page-num on intersection
```

## Edge Cases

1. **Large PDFs** — Many pages with continuous render could be slow. For MVP, render all pages. If performance is an issue later, add virtual scrolling (only render visible pages + buffer).
2. **Pane resize during scroll** — ResizeObserver triggers re-render; need to preserve scroll position (save scroll ratio before re-render, restore after).
3. **Very wide panes** — PDF might look too large. The zoom controls let users zoom out. No max-width constraint needed since the pane itself is bounded.
4. **ResizeObserver loop** — Debounce the resize handler to avoid render loops.

## Testing

1. Make the template change in `src/ralph/tools/latex_templates.py`
2. No build needed (Python-side change only — no Svelte changes)
3. Test with existing test command:
   ```bash
   uv run python3 -c "import asyncio; from pathlib import Path; from ralph.artifacts import compile_and_push; asyncio.run(compile_and_push(Path('/tmp/latex-test/test.tex'), 'ed6d1437-7b38-47a4-bd49-670267f0a7ce'))"
   ```
4. Hard refresh browser (Cmd+Shift+R) and check artifact panel
5. Verify: PDF pages stack vertically, filling pane width, scrollable, no margins
6. Verify: Zoom in/out works, download works, page indicator updates on scroll
7. Verify: Resizing the pane re-renders pages to fit new width

## Success Criteria

- [ ] PDF fills 100% of artifact pane width (no horizontal margins)
- [ ] All pages visible via continuous scroll (no prev/next pagination)
- [ ] Pages auto-scale to pane width on resize
- [ ] Zoom in/out works relative to fit-to-width baseline
- [ ] Page indicator in toolbar updates as user scrolls
- [ ] Download button still works
- [ ] No visual regressions (toolbar looks clean, background matches)

## Files Changed

| File | Change |
|------|--------|
| `src/ralph/tools/latex_templates.py` | Replace `PDF_VIEWER_TEMPLATE` with continuous-scroll, fit-to-width version |

No Svelte changes needed. No build/deploy of OpenWebUI required.

## Risk Assessment

**Low risk** — Single file change, no API changes, no database changes, no Svelte changes. The template is self-contained HTML that runs inside a sandboxed iframe. If something breaks, the old template can be restored instantly.
