---
date: 2026-01-29T06:19:35+0000
researcher: ariasulin
git_commit: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "Live Interactive LaTeX-to-PDF Sandboxing with OpenWebUI Artifacts"
tags: [research, latex, pdf, artifacts, openwebui, agno, sandbox, live-editing]
status: complete
last_updated: 2026-01-28
last_updated_by: ariasulin
---

# Research: Live Interactive LaTeX-to-PDF Sandboxing with OpenWebUI Artifacts

**Date**: 2026-01-29T06:19:35+0000
**Researcher**: ariasulin
**Git Commit**: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

How to add live interactive sandboxing of LaTeX to PDF, where the Agno agent edits a LaTeX file in the workspace and the compiled PDF is rendered live via OpenWebUI's native interactive artifacts system?

## Summary

The implementation path involves three main integration points:

1. **OpenWebUI Artifacts System**: Supports HTML/SVG/iframe rendering but **not native PDF rendering in artifacts**. PDFs are rendered via iframe with `pdfjs-dist` in the FileItemModal (file attachments), not the artifacts pane.

2. **Ralph Agent Workspace**: Agno agent already has workspace-scoped `FileTools` and `ShellTools`. The agent can write `.tex` files and execute shell commands (e.g., `pdflatex`, `tectonic`) to compile them.

3. **Live Update Mechanism**: Currently no file watching exists. Updates require either (a) polling the workspace API, (b) SSE events from Ralph server, or (c) a new WebSocket channel for file change notifications.

**Recommended Architecture**: Agent edits LaTeX via FileTools → triggers compilation via ShellTools → emits artifact event containing either (a) HTML wrapper with embedded PDF.js viewer, or (b) base64-encoded PDF in iframe `srcdoc`. OpenWebUI's artifact system handles the display.

## Detailed Findings

### 1. OpenWebUI Artifacts System

**Location**: `OpenWebUI/open-webui/src/lib/components/chat/Artifacts.svelte`

The artifacts system provides a side panel for rendering interactive HTML/SVG content from LLM responses.

#### How It Works

1. **Detection** (`ContentRenderer.svelte:183-191`): Automatically detects `html`, `svg`, or `xml` code blocks in messages
2. **Extraction** (`utils/index.ts:1622-1680`): `getCodeBlockContents()` parses HTML/CSS/JS from fenced code blocks
3. **Assembly** (`Chat.svelte:836-885`): Combines extracted code into full HTML document with DOCTYPE, head, body
4. **Rendering** (`Artifacts.svelte:233-251`): Displays in sandboxed iframe with configurable permissions

#### Supported Content Types

| Type | Detection | Rendering |
|------|-----------|-----------|
| HTML | `lang="html"` code block | iframe with `srcdoc` |
| CSS | `lang="css"` code block | Injected into HTML `<style>` |
| JavaScript | `lang="javascript"` or `lang="js"` | Injected into HTML `<script>` |
| SVG | `lang="svg"` or `lang="xml"` with `<svg` | `SvgPanZoom` component |
| **PDF** | **Not supported in artifacts** | N/A |

#### Iframe Sandbox Permissions

```html
<iframe
  sandbox="allow-scripts allow-downloads {allowForms} {allowSameOrigin}"
  srcdoc={content}
/>
```

- `allow-scripts`: Always enabled (JavaScript execution)
- `allow-downloads`: Always enabled
- `allow-forms`: User setting `$settings?.iframeSandboxAllowForms`
- `allow-same-origin`: User setting `$settings?.iframeSandboxAllowSameOrigin`

#### Version Navigation

Artifacts maintain version history as conversation progresses. Each message with HTML/SVG creates a new version. Users navigate with prev/next buttons.

#### API for Pipes

**There is no special API for emitting artifacts.** Pipes emit standard markdown with code blocks:

```python
await __event_emitter__({
    "type": "message",
    "data": {"content": "```html\n<h1>Hello</h1>\n```"}
})
```

The frontend automatically extracts and renders artifacts from message content.

### 2. PDF Rendering in OpenWebUI

**Location**: `OpenWebUI/open-webui/src/lib/components/common/FileItemModal.svelte`

PDF rendering exists but is separate from the artifacts system:

- **Library**: `pdfjs-dist` version 5.4.149 (`package.json:120`)
- **Detection**: `content_type === 'application/pdf'` or `.pdf` extension
- **Rendering**: Standard iframe with `src={WEBUI_API_BASE_URL}/files/${item.id}/content`
- **Context**: Only in FileItemModal for file attachments, not artifacts pane

**Key Insight**: To render PDFs in artifacts, we must either:
1. Serve PDF via URL and use iframe with that URL
2. Embed PDF.js viewer in HTML artifact with base64-encoded PDF
3. Use an HTML artifact that loads PDF.js and fetches PDF from workspace API

### 3. Ralph Agent Workspace & Tools

**Location**: `src/ralph/server.py:248-261`

#### Workspace Configuration

```python
def get_workspace_path(user_id: str) -> Path:
    settings = get_settings()
    if settings.agent_workspace:
        # Shared workspace (e.g., a codebase)
        workspace = Path(settings.agent_workspace)
    else:
        # Per-user isolated workspace
        workspace = Path(settings.user_data_dir) / user_id / "workspace"
        workspace.mkdir(parents=True, exist_ok=True)
    return workspace
```

**Environment Variables**:
- `RALPH_AGENT_WORKSPACE`: Shared workspace path (optional)
- `RALPH_USER_DATA_DIR`: Per-user workspace root (default: `/data/ralph/users`)

#### Tool Registration

```python
agent = Agent(
    model=OpenRouter(...),
    tools=[
        strip_agno_fields(ShellTools(base_dir=workspace)),
        strip_agno_fields(FileTools(base_dir=workspace)),
        # ... other tools
    ],
    instructions=instructions,
)
```

**FileTools Capabilities** (Agno built-in):
- Read files
- Write files
- List directory contents
- Create directories
- Delete files

**ShellTools Capabilities** (Agno built-in):
- Execute shell commands with `cwd` set to workspace
- Capture stdout/stderr
- Access system binaries (including LaTeX compilers)

#### Current Limitation: No Live File Updates

The codebase has workspace sync capabilities but **no active file watching**:

- `src/ralph/sync/workspace_sync.py`: Polling-based sync with OpenWebUI knowledge base
- `src/ralph/api/workspace.py`: HTTP API for file CRUD operations
- No inotify/FSEvents/watchdog implementation

Updates require manual triggers via:
- `GET /users/{user_id}/workspace/files?refresh=true`
- `POST /users/{user_id}/workspace/sync`

### 4. Ralph SSE Streaming

**Location**: `src/ralph/pipe.py`, `src/ralph/server.py`

#### Event Flow

1. OpenWebUI sends message to Ralph Pipe
2. Pipe POSTs to `/chat/stream` endpoint
3. Server creates Agno agent and streams response
4. SSE events emitted during execution:

```python
# Status event (during processing)
yield {"event": "message", "data": json.dumps({"type": "status", "content": "Thinking..."})}

# Message content (streamed chunks)
yield {"event": "message", "data": json.dumps({"type": "message", "content": "chunk"})}

# Completion
yield {"event": "message", "data": json.dumps({"type": "done"})}
```

5. Pipe translates to OpenWebUI event emitter format

#### Extension Point

The server can emit custom events during agent execution. After compiling LaTeX, we could emit:

```python
yield {"event": "message", "data": json.dumps({
    "type": "message",
    "content": "```html\n<iframe src='...'></iframe>\n```"
})}
```

### 5. LaTeX Compilation Options

#### Compilers

| Compiler | Speed | Unicode | Fonts | Notes |
|----------|-------|---------|-------|-------|
| pdfLaTeX | Fastest | Limited | Basic | Default choice |
| XeLaTeX | Slower | Native | System | Deprecated |
| LuaLaTeX | Slowest | Native | Best | Preferred for Unicode |
| **Tectonic** | Fast | Native | TrueType | Self-contained, Rust-based |

**Recommendation**: **Tectonic** for modern deployments:
- Self-contained binary (no TeX distribution needed)
- Automatic package downloads on demand
- Based on XeTeX engine (good font support)
- No intermediate files by default

#### Incremental Compilation (Speed)

For live editing, compilation speed is critical:

1. **Preamble Precompilation** (Traditional):
   ```bash
   pdflatex -ini -jobname="myformat" "&pdflatex preamble.tex\dump"
   ```
   Document uses `%&myformat` directive.

2. **latexmk -pvc** (Continuous Preview):
   ```bash
   latexmk -pvc -pdf document.tex
   ```
   File watching + auto-recompile.

3. **Tectonic** (Built-in efficiency):
   - Automatic looping (runs as many times as needed)
   - Typically 500ms-2s compilation

#### Docker Sandboxing

For security with untrusted input:

```yaml
services:
  latex-compiler:
    image: texlive/texlive:latest
    volumes:
      - ./workspace:/workspace:ro
      - ./output:/output
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
```

**Overleaf's Approach**: Sibling containers for each compilation, no network access.

### 6. Browser-Based PDF Rendering

#### PDF.js

```javascript
import * as pdfjsLib from 'pdfjs-dist';

const pdf = await pdfjsLib.getDocument(pdfUrl).promise;
const page = await pdf.getPage(1);
const canvas = document.getElementById('pdf-canvas');
await page.render({ canvasContext: canvas.getContext('2d'), viewport: page.getViewport({ scale: 1.5 }) }).promise;
```

**Features**:
- Open source (Apache 2.0)
- Web worker support
- Canvas rendering
- OpenWebUI already has `pdfjs-dist` in dependencies

#### Embedding in Artifact

Option A: URL-based loading
```html
<iframe src="/api/v1/workspace/{user_id}/files/output.pdf"></iframe>
```

Option B: Base64 embedded
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<canvas id="pdf-canvas"></canvas>
<script>
const pdfData = atob('JVBERi0xLjQ...'); // base64 PDF
pdfjsLib.getDocument({data: pdfData}).promise.then(pdf => {
    pdf.getPage(1).then(page => {
        const canvas = document.getElementById('pdf-canvas');
        page.render({canvasContext: canvas.getContext('2d'), viewport: page.getViewport({scale: 1.5})});
    });
});
</script>
```

## Architecture Options

### Option A: Agent-Driven Artifact Emission (Simplest)

```
User Message → Ralph Server → Agno Agent
                                 ↓
                         FileTools.write("doc.tex")
                                 ↓
                         ShellTools.run("tectonic doc.tex")
                                 ↓
                         FileTools.read("doc.pdf") as base64
                                 ↓
                         Emit HTML artifact with embedded PDF.js
                                 ↓
                         OpenWebUI Artifacts Pane
```

**Pros**: Uses existing infrastructure, no new endpoints
**Cons**: Large base64 PDFs in message content, no live updates during edit

**Implementation**:
1. Agent writes `.tex` file
2. Agent runs `tectonic doc.tex`
3. Agent reads `doc.pdf` as base64
4. Agent emits markdown with HTML artifact containing PDF.js viewer

### Option B: Workspace API + Artifact URL Reference

```
User Message → Ralph Server → Agno Agent
                                 ↓
                         FileTools.write("doc.tex")
                                 ↓
                         ShellTools.run("tectonic doc.tex")
                                 ↓
                         Emit HTML artifact referencing workspace URL
                                 ↓
                         OpenWebUI Artifacts Pane
                                 ↓
                         Iframe loads PDF from workspace API
```

**Pros**: Smaller messages, PDF served via proper endpoint
**Cons**: Requires CORS configuration, `allow-same-origin` sandbox permission

**Implementation**:
1. Add PDF content-type support to workspace file API
2. Agent writes/compiles as above
3. Agent emits HTML with iframe pointing to `/users/{user_id}/workspace/files/doc.pdf`
4. Artifact iframe loads PDF directly

### Option C: Real-Time File Watching (Most Feature-Rich)

```
OpenWebUI ←── WebSocket ←── Ralph Server ←── File Watcher
                                 ↑
                         Agno Agent edits .tex file
                                 ↓
                         File Watcher detects change
                                 ↓
                         Triggers compilation
                                 ↓
                         WebSocket notifies OpenWebUI
                                 ↓
                         Artifact panel refreshes PDF
```

**Pros**: True live preview, updates on every save
**Cons**: Significant new infrastructure (WebSocket, file watcher)

**Implementation**:
1. Add `watchdog` or similar file watcher to Ralph server
2. Implement WebSocket endpoint for file change notifications
3. Modify artifacts component to subscribe to file changes
4. Auto-refresh artifact on notification

### Option D: Polling-Based Live Preview (Pragmatic Middle Ground)

```
OpenWebUI ←── Polls workspace API ←── Ralph Server
                                           ↓
                         Agno Agent edits .tex continuously
                                           ↓
                         Each edit triggers compilation
                                           ↓
                         UI polls for updated PDF at interval
```

**Pros**: No WebSocket needed, simpler implementation
**Cons**: Latency from polling interval, wasted requests

**Implementation**:
1. Agent tool that writes `.tex` AND compiles in one operation
2. Artifact HTML includes JavaScript that polls workspace API
3. PDF.js reloads when hash changes

## Recommended Implementation Path

### Phase 1: Basic Agent-Driven Preview (Option A)

**Goal**: Proof of concept with minimal infrastructure changes

1. **Create LaTeX compilation tool** for Agno agent:
   ```python
   class LaTeXTools(Toolkit):
       def compile_latex(self, source_file: str) -> str:
           """Compile LaTeX and return PDF as base64."""
           subprocess.run(["tectonic", source_file], cwd=self.base_dir)
           pdf_path = source_file.replace('.tex', '.pdf')
           with open(pdf_path, 'rb') as f:
               return base64.b64encode(f.read()).decode()
   ```

2. **Create PDF artifact template**:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
       <style>canvas { width: 100%; }</style>
   </head>
   <body>
       <canvas id="pdf"></canvas>
       <script>
           const data = atob('{{PDF_BASE64}}');
           pdfjsLib.getDocument({data}).promise.then(pdf => {
               pdf.getPage(1).then(page => {
                   const canvas = document.getElementById('pdf');
                   const ctx = canvas.getContext('2d');
                   const viewport = page.getViewport({scale: 1.5});
                   canvas.height = viewport.height;
                   canvas.width = viewport.width;
                   page.render({canvasContext: ctx, viewport});
               });
           });
       </script>
   </body>
   </html>
   ```

3. **Agent workflow**:
   - User asks agent to help with LaTeX document
   - Agent creates/edits `.tex` file via FileTools
   - Agent calls `compile_latex()` tool
   - Tool returns HTML artifact with embedded PDF
   - OpenWebUI displays in artifacts pane

### Phase 2: Workspace URL Reference (Option B)

**Goal**: Cleaner URLs, proper file serving

1. **Enhance workspace API** to serve PDFs:
   ```python
   @router.get("/users/{user_id}/workspace/files/{path:path}")
   async def get_file(user_id: str, path: str):
       # ... existing implementation ...
       if path.endswith('.pdf'):
           return FileResponse(file_path, media_type='application/pdf')
   ```

2. **Update artifact template** to use URL:
   ```html
   <iframe src="/api/workspace/users/{{USER_ID}}/files/{{PDF_PATH}}"
           style="width:100%;height:100%;border:none;"></iframe>
   ```

3. **Configure CORS** if needed for cross-origin iframe access

### Phase 3: Real-Time Updates (Option C/D)

**Goal**: Live preview as agent edits

**Option 3a (Polling)**:
- Add cache-busting timestamp to PDF URL
- Artifact polls workspace API every 2-3 seconds
- Reloads PDF when file hash changes

**Option 3b (WebSocket)**:
- Add WebSocket endpoint to Ralph server
- File watcher triggers compilation on `.tex` changes
- Broadcasts update event to connected clients
- Artifact component subscribes and refreshes

## Code References

### OpenWebUI Artifacts
- `OpenWebUI/open-webui/src/lib/components/chat/Artifacts.svelte` - Main viewer
- `OpenWebUI/open-webui/src/lib/components/chat/Messages/ContentRenderer.svelte:183-191` - Detection
- `OpenWebUI/open-webui/src/lib/utils/index.ts:1622-1680` - Content extraction
- `OpenWebUI/open-webui/src/lib/stores/index.ts:96-97` - Svelte stores

### Ralph Server
- `src/ralph/server.py:47-57` - Workspace path resolution
- `src/ralph/server.py:248-261` - Agent tool registration
- `src/ralph/server.py:298-325` - SSE event emission

### Ralph Pipe
- `src/ralph/pipe.py:132-167` - SSE event translation
- `src/ralph/pipe.py:153-154` - Message event forwarding

### Workspace API
- `src/ralph/api/workspace.py:86-117` - File listing
- `src/ralph/api/workspace.py:120-168` - File download
- `src/ralph/sync/workspace_sync.py` - Sync service

## Historical Context (from thoughts/)

No existing documents specifically address LaTeX/PDF generation. Related documents:

- `thoughts/shared/plans/2026-01-28-memory-edit-tool.md` - MemoryBlockTools pattern applicable to LaTeXTools
- `thoughts/shared/research/2026-01-12-ARI-78-openwebui-realtime-updates.md` - WebSocket/SSE patterns for live updates
- `thoughts/shared/research/2026-01-16-letta-to-agno-migration.md` - Mentions `PDFUrlKnowledgeBase` for PDF ingestion (RAG, not rendering)
- `thoughts/shared/research/2026-01-28-ralph-agno-architecture.md` - Workspace-scoped tools architecture

## Open Questions

1. **PDF Size Limits**: Base64 encoding increases size by ~33%. Large PDFs may hit message size limits. What's the practical limit?

2. **Multi-Page Navigation**: PDF.js can render single pages. Full document viewing needs page navigation UI. Should this be in artifact or separate viewer?

3. **LaTeX Distribution**: Should Tectonic be bundled in Ralph Docker image, or use a separate compilation container?

4. **Security**: LaTeX can execute shell commands (`\write18`). How to sandbox compilation for untrusted input?

5. **Live Edit Granularity**: Should compilation trigger on every character (debounced) or explicit save? What's acceptable latency?

6. **Error Display**: How to show LaTeX compilation errors in the UI? Inline in artifact, separate panel, or chat message?
