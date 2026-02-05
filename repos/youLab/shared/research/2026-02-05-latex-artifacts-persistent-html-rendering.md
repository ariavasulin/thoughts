---
date: 2026-02-05T17:03:59+0000
researcher: ariasulin
git_commit: 9dae3bb
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "LaTeX/PDF Artifact Status and Persistent HTML Rendering in OpenWebUI"
tags: [research, latex, pdf, artifacts, openwebui, html, iframe, persistent-rendering]
status: complete
last_updated: 2026-02-05
last_updated_by: ariasulin
---

# Research: LaTeX/PDF Artifact Status and Persistent HTML Rendering in OpenWebUI

**Date**: 2026-02-05T17:03:59+0000
**Researcher**: ariasulin
**Git Commit**: 9dae3bb
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

1. How far did we get with LaTeX → PDF artifact rendering in OpenWebUI?
2. Is it possible to render a persistent HTML file as an artifact in chat, without the LLM generating the HTML itself?
3. How could we hack OpenWebUI artifacts to display persistent HTML/JS content?

## Summary

### LaTeX/PDF Status: Fully implemented but untested end-to-end

The `LaTeXTools` toolkit (`src/ralph/tools/latex_tools.py`) is **fully implemented** with:
- Professional LaTeX template with theorem environments, code listings, math support
- Tectonic compilation with error handling and timeouts
- PDF.js viewer with page navigation, zoom, and download
- Base64-encoded PDF embedded in self-contained HTML artifact
- Agent instructions in `server.py:210-244` teaching the agent how to use it
- Test suite in `tests/ralph/tools/test_latex_tools.py`

**What was NOT completed / verified:**
- End-to-end testing with actual OpenWebUI artifact rendering
- Whether the base64 HTML code block is too large for OpenWebUI's artifact detection
- Whether the streaming chunks reassemble correctly into a renderable artifact
- Whether OpenWebUI's `allow-scripts` sandbox permits PDF.js CDN loading

### Persistent HTML Rendering: Possible via three approaches

OpenWebUI's artifact system renders **any complete HTML page** found in a fenced `html` code block. There is no requirement that the LLM "generates" the HTML — it just needs to appear in the message content. This opens three paths for persistent HTML:

1. **Tool HTMLResponse** (cleanest): An OpenWebUI Tool returns `HTMLResponse` with `Content-Disposition: inline` — renders as iframe directly in chat
2. **Agent emits static HTML**: The agent tool reads a persistent HTML file and returns it in a code block — triggers artifact rendering
3. **Artifact with fetch**: HTML artifact contains JS that fetches content from a URL (e.g., Ralph workspace API) — renders dynamic content inside the artifact iframe

## Detailed Findings

### 1. Current LaTeX Implementation State

#### What exists (`src/ralph/tools/latex_tools.py`)

- **LaTeX template** (lines 19-101): Full document class with `amsmath`, `amsthm`, `listings`, `hyperref`, `fancyhdr`, `titlesec`, theorem environments
- **PDF.js viewer** (lines 105-263): Complete HTML page with toolbar (prev/next, zoom, download), canvas rendering, base64 PDF decoding
- **`render_notes` tool** (lines 288-397): Takes `title` + `content` (LaTeX), compiles with Tectonic, returns HTML artifact in markdown code block
- **Error handling**: Compilation errors, missing Tectonic, timeouts all return friendly messages
- **Agent instructions** (`server.py:210-244`): Detailed LaTeX syntax guide + CRITICAL instruction to include HTML verbatim

#### The gap: artifact rendering in practice

The tool returns content like:
```
I've created your notes on **Title**...

\`\`\`html
<!DOCTYPE html>...base64 PDF embedded...
\`\`\`
```

For this to render as an artifact in OpenWebUI:
1. The streamed chunks must reassemble into a complete message with the code block
2. OpenWebUI's `ContentRenderer.svelte` must detect the `html` code block
3. `getCodeBlockContents()` must extract the HTML
4. The sandboxed iframe must allow PDF.js CDN script loading

**Potential issues**: Base64-encoded PDFs can be very large (a 1MB PDF = ~1.3MB base64 string). This may cause problems with message size limits or artifact detection parsing.

### 2. OpenWebUI Artifact System (How It Works)

Detection pipeline:
1. `ContentRenderer.svelte:183-191` — detects `html`/`svg`/`xml` code blocks
2. `utils/index.ts:1622-1680` — `getCodeBlockContents()` extracts HTML/CSS/JS
3. `Chat.svelte:836-885` — assembles into full HTML document
4. `Artifacts.svelte:233-251` — renders in sandboxed iframe with `srcdoc`

**Key constraint**: Artifacts are triggered by code block detection in markdown, NOT by a special event type. There is no `artifact` event in the emitter API.

**Iframe sandbox**: `allow-scripts allow-downloads` always enabled. `allow-same-origin` is a user setting — needed for cross-origin fetch from within artifact.

**Persistent storage API**: OpenWebUI advertises built-in key-value storage for artifacts (journals, trackers) using `window.parent.postMessage`. Underdocumented but exists.

### 3. Approaches for Persistent HTML Rendering

#### Approach A: OpenWebUI Tool with HTMLResponse

Create a native OpenWebUI Tool (not a pipe) that returns `HTMLResponse`:

```python
from fastapi.responses import HTMLResponse

class Tools:
    def show_dashboard(self) -> HTMLResponse:
        html = open("/path/to/persistent/dashboard.html").read()
        headers = {"Content-Disposition": "inline"}
        return HTMLResponse(content=html, headers=headers)
```

- Renders directly as iframe in chat message
- Auto-height support built in
- **Limitation**: Must be a native OpenWebUI Tool, not a pipe function
- **Limitation**: Tools are invoked by the LLM, so the LLM still decides when to show it

#### Approach B: Ralph Tool Emits Pre-Built HTML

Add a tool to Ralph that reads a persistent HTML file and returns it as an artifact:

```python
class DashboardTools(Toolkit):
    def show_dashboard(self, run_context: RunContext, name: str) -> str:
        html = (self.workspace / "dashboards" / f"{name}.html").read_text()
        return f"```html\n{html}\n```"
```

- Uses existing artifact pipeline (code block → artifact pane)
- HTML file is persistent on disk, not generated per-request
- Agent just needs to call the tool — doesn't write the HTML
- File can be updated independently, next invocation shows latest version

#### Approach C: Artifact with Live Fetch (Most Powerful)

Emit an HTML artifact that fetches content from an API:

```html
<!DOCTYPE html>
<html>
<body>
  <div id="app"></div>
  <script>
    // Fetch latest content from Ralph workspace API
    const API = 'http://localhost:8200/users/USER_ID/workspace/files/dashboard.html';
    fetch(API)
      .then(r => r.text())
      .then(html => document.getElementById('app').innerHTML = html);

    // Or poll for updates
    setInterval(() => {
      fetch(API).then(r => r.text()).then(html => {
        document.getElementById('app').innerHTML = html;
      });
    }, 5000);
  </script>
</body>
</html>
```

- **Requires**: `allow-same-origin` sandbox setting enabled in OpenWebUI
- **Requires**: CORS headers on Ralph server for iframe origin
- Content is truly persistent — changes to the file reflect on next poll
- Can include full JS frameworks (React, D3, etc.) loaded from CDN

#### Approach D: Hack OpenWebUI Frontend Directly

Since YouLab runs a fork of OpenWebUI, you could:
1. Add a custom message type or annotation that triggers a persistent iframe
2. Modify `Artifacts.svelte` to accept a URL parameter instead of just `srcdoc`
3. Add a "pinned artifact" concept that persists across messages
4. Use the existing `__event_emitter__` `execute` event to inject DOM elements

This is the most flexible but requires maintaining fork divergence.

## Existing Related Plans

- `thoughts/shared/plans/2026-01-28-latex-pdf-notes-tool.md` — Implementation plan for LaTeXTools (Phase 1 = what's implemented)
- `thoughts/shared/research/2026-01-28-latex-pdf-live-sandbox-artifacts.md` — Deep research on artifact system, PDF rendering options, and live update architectures (Options A-D)
- `thoughts/shared/research/2026-01-24-openwebui-youlab-patches.md` — YouLab's OpenWebUI fork patches (theme, rendering modifications)

## Code References

- `src/ralph/tools/latex_tools.py:266-417` — LaTeXTools class and render_notes
- `src/ralph/tools/latex_tools.py:105-263` — PDF_VIEWER_TEMPLATE (HTML + PDF.js)
- `src/ralph/tools/latex_tools.py:19-101` — NOTES_TEMPLATE (LaTeX template)
- `src/ralph/server.py:210-244` — Agent instructions for LaTeX usage
- `src/ralph/server.py:293` — LaTeXTools registration with agent
- `src/ralph/server.py:343-359` — SSE streaming of agent responses
- `src/ralph/pipe.py:132-167` — SSE event translation to OpenWebUI emitter
- `src/ralph/api/workspace.py:120-168` — File serving endpoint (supports .html)
- `tests/ralph/tools/test_latex_tools.py` — Test suite

## Open Questions

1. Has the LaTeX tool been tested end-to-end with a running OpenWebUI instance?
2. What's OpenWebUI's practical message size limit for artifact detection?
3. Would Approach C (fetch-based artifact) work with the current CORS config?
4. Is the `allow-same-origin` sandbox setting enabled in the YouLab OpenWebUI deployment?
5. For truly persistent display (not LLM-triggered), should this be a custom OpenWebUI component rather than an artifact hack?
