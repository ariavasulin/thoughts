# Live LaTeX Artifact Push — Implementation Plan

## Overview

Replace the current "agent returns HTML code block" artifact pattern with a **deterministic, push-based system**: whenever the agent writes or edits a `.tex` file via FileTools, Ralph automatically compiles it to PDF, builds an HTML viewer, and pushes it to the OpenWebUI artifact panel via a new REST endpoint. The agent never thinks about compilation — it just edits files and the PDF appears.

## Current State Analysis

### What exists now
- `latex_tools.py:render_notes(title, content)` — takes entire LaTeX content as a tool argument, compiles in a temp dir, returns an HTML code block with embedded PDF.js viewer and base64 PDF
- OpenWebUI's artifact panel detects `html` code blocks in message content via `getContents()` in `Chat.svelte:836-884`
- Agent must echo the entire HTML blob (~100KB+) verbatim in its response for the artifact to render
- No persistent `.tex` files — temp dir is cleaned up after compilation
- Ralph and OpenWebUI are separate containers on the same Docker network (`docker-compose.prod.yml`)
- OpenWebUI socket server at `openwebui:8080/ws/socket.io` emits `events` to `user:{user_id}` rooms
- OpenWebUI REST auth supports API keys (`sk-` prefix) via `utils/auth.py:287-288`

### Key Discoveries
- `routers/notes.py:243` and `routers/channels.py:1104` already import `sio` from `socket.main` and emit events from REST endpoints — this is the established pattern
- `Chat.svelte:352-478` (`chatEventHandler`) handles socket `events` by type — adding a new type is straightforward
- `artifactContents` store (`stores/index.ts:97`) is a writable array of `{type, content}` — can be set directly from any event handler
- Agno's `FileTools` is imported from the SDK (`agno.tools.file`), not defined locally — we need a wrapper, not a fork
- `get_workspace_path()` at `server.py:47-57` resolves per-user workspace paths

## Desired End State

1. Agent edits `.tex` files using standard FileTools (`save_file`, `write_file`, etc.)
2. After any `.tex` write, Ralph automatically compiles → PDF → HTML viewer → pushes to artifact panel
3. Artifact panel opens/updates without any HTML in the chat message
4. Works during chat turns AND from background tasks
5. `.tex` files persist in the user's workspace for future editing
6. The PDF viewer template and compilation logic are reused from the existing `latex_tools.py`

### Verification
- Agent writes a `.tex` file → PDF appears in artifact panel within ~5 seconds
- Agent edits the same `.tex` file → PDF updates in-place
- No HTML code blocks appear in chat messages
- `make verify-agent` passes
- Works in production Docker compose setup

## What We're NOT Doing

- File-system watchers (inotify/watchdog) — the hook triggers on tool calls, not filesystem events
- WebSocket connection from Ralph to OpenWebUI — REST is simpler and stateless
- Modifying Agno's FileTools source — we wrap it
- Supporting non-LaTeX artifact types (HTML, SVG) — those continue using the existing code-block detection
- Collaborative editing or conflict resolution on `.tex` files
- Removing the existing code-block artifact detection — it continues working for non-LaTeX artifacts

## Implementation Approach

**Ralph wraps FileTools with a post-write hook.** When `save_file` (or any write method) targets a `.tex` path, Ralph compiles the file after the write completes, builds the HTML viewer, and POSTs it to a new OpenWebUI endpoint. OpenWebUI's endpoint emits a socket event to the user. The frontend handles the new event type by setting the `artifactContents` store directly.

```
Agent calls save_file("notes.tex", content)
    → FileTools writes file to workspace
    → Post-write hook detects .tex extension
    → compile_latex(tex_path) → PDF bytes
    → build_pdf_viewer(pdf_bytes) → HTML string
    → POST to OpenWebUI /api/artifact/push
        → OpenWebUI sio.emit("events", {type: "chat:artifact", ...})
            → Chat.svelte chatEventHandler
                → artifactContents.set([{type: 'iframe', content: html}])
                → showArtifacts.set(true), showControls.set(true)
```

---

## Phase 1: OpenWebUI Fork — Artifact Push Endpoint + Frontend Handler

### Overview
Add a REST endpoint that accepts HTML content and emits it to a user's artifact panel via socket.io. Add the frontend handler to receive and display it.

### Changes Required:

#### 1. New Router: `backend/open_webui/routers/artifacts.py`

**File**: `OpenWebUI/open-webui/backend/open_webui/routers/artifacts.py` (new)

A minimal router following the same pattern as `notes.py` — imports `sio`, emits events from a REST handler. Auth via API key or JWT (uses existing `get_verified_user` dependency).

```python
"""Artifact push endpoint for external services (e.g., Ralph)."""

import logging
from typing import Optional

from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel

from open_webui.socket.main import sio
from open_webui.utils.auth import get_verified_user

log = logging.getLogger(__name__)
router = APIRouter()


class ArtifactPushRequest(BaseModel):
    user_id: str
    chat_id: Optional[str] = None
    content: str  # Full HTML string
    title: Optional[str] = None


@router.post("/artifact/push")
async def push_artifact(
    form_data: ArtifactPushRequest,
    user=Depends(get_verified_user),
):
    """Push HTML content to a user's artifact panel via socket.io."""
    # Only admins or the user themselves can push artifacts
    if user.role != "admin" and user.id != form_data.user_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Cannot push artifacts to other users",
        )

    await sio.emit(
        "events",
        {
            "chat_id": form_data.chat_id,
            "message_id": None,
            "data": {
                "type": "chat:artifact",
                "data": {
                    "content": form_data.content,
                    "title": form_data.title,
                },
            },
        },
        room=f"user:{form_data.user_id}",
    )

    return {"status": "ok"}
```

#### 2. Register Router in Main App

**File**: `OpenWebUI/open-webui/backend/open_webui/main.py`

Add the router import and registration alongside existing routers (notes, channels, etc.):

```python
from open_webui.routers.artifacts import router as artifacts_router

# Near other router registrations:
app.include_router(artifacts_router, prefix="/api", tags=["artifacts"])
```

#### 3. Frontend Event Handler

**File**: `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte`

Add handler in `chatEventHandler` (after the existing `embeds` handler at line ~385):

```typescript
} else if (type === 'chat:artifact') {
    // Pushed artifact from external service (e.g., Ralph)
    const artifactContent = data.content;
    if (artifactContent) {
        artifactContents.update((current) => {
            const updated = [...(current || []), { type: 'iframe', content: artifactContent }];
            return updated;
        });
        showArtifacts.set(true);
        showControls.set(true);
    }
```

Note: This handler does NOT require `message_id` to be valid. The `chatEventHandler` checks `if (message)` at line 359 — but the event still arrives via socket. We need to handle `chat:artifact` **before** the message check, or handle it even when `message` is null. The cleanest approach: add an early return handler at the top of `chatEventHandler`, before the `if (message)` guard:

```typescript
const chatEventHandler = async (event, cb) => {
    if (event.chat_id === $chatId || !event.chat_id) {
        const type = event?.data?.type ?? null;
        const data = event?.data?.data ?? null;

        // Artifact push — doesn't require a message context
        if (type === 'chat:artifact') {
            if (data?.content) {
                artifactContents.update((current) => {
                    return [...(current || []), { type: 'iframe', content: data.content }];
                });
                showArtifacts.set(true);
                showControls.set(true);
            }
            return;
        }

        // ... existing message-dependent handlers below
        await tick();
        let message = history.messages[event.message_id];
        if (message) {
            // ... rest of existing handler
```

### Success Criteria:

#### Automated Verification:
- [ ] OpenWebUI builds without errors: `cd OpenWebUI/open-webui && npm run build`
- [ ] OpenWebUI backend starts without import errors
- [ ] `curl -X POST http://localhost:8080/api/artifact/push -H "Authorization: Bearer sk-..." -H "Content-Type: application/json" -d '{"user_id": "...", "content": "<html><body><h1>Test</h1></body></html>"}'` returns `{"status": "ok"}`

#### Manual Verification:
- [ ] Open a chat in OpenWebUI, run the curl command above, and confirm the artifact panel opens with the HTML content
- [ ] Confirm the artifact panel updates (not replaces) when a second push is sent
- [ ] Confirm existing code-block artifacts still work normally

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Ralph — LaTeX Compiler Service + Artifact Push Client

### Overview
Extract the compilation logic from `latex_tools.py` into a standalone module that compiles `.tex` files and pushes the result to OpenWebUI. Add configuration for OpenWebUI API key.

### Changes Required:

#### 1. New Config Values

**File**: `src/ralph/config.py`

```python
# Add to RalphSettings:
openwebui_api_key: str = ""     # env: RALPH_OPENWEBUI_API_KEY (admin API key)
openwebui_url: str = "http://openwebui:8080"  # env: RALPH_OPENWEBUI_URL (internal Docker URL)
```

These likely already exist (used by workspace sync at `api/workspace.py:259-263`). Verify and reuse.

#### 2. New Module: `src/ralph/artifacts.py`

**File**: `src/ralph/artifacts.py` (new)

This module owns compilation and push. It reuses the templates from `latex_tools.py`.

```python
"""Artifact compilation and push to OpenWebUI."""

from __future__ import annotations

import base64
import logging
import subprocess
import tempfile
from pathlib import Path

import httpx

from ralph.config import get_settings

logger = logging.getLogger(__name__)

# Import templates from latex_tools (keep them there as single source of truth)
from ralph.tools.latex_tools import PDF_VIEWER_TEMPLATE


async def compile_and_push(
    tex_path: Path,
    user_id: str,
    chat_id: str | None = None,
) -> str:
    """Compile a .tex file to PDF and push the viewer to OpenWebUI's artifact panel.

    Args:
        tex_path: Absolute path to the .tex file in the user's workspace.
        user_id: OpenWebUI user ID for socket targeting.
        chat_id: Optional chat ID for scoping the artifact.

    Returns:
        Short status string suitable for tool output.
    """
    if not tex_path.exists():
        return f"Error: {tex_path.name} not found."

    if not tex_path.suffix == ".tex":
        return f"Error: {tex_path.name} is not a .tex file."

    # Compile LaTeX → PDF
    pdf_bytes = _compile_latex(tex_path)
    if isinstance(pdf_bytes, str):
        return pdf_bytes  # Error message

    # Build HTML viewer
    title = tex_path.stem.replace("_", " ").replace("-", " ").title()
    html = _build_viewer(pdf_bytes, title)

    # Push to OpenWebUI
    await _push_artifact(user_id, html, chat_id=chat_id, title=title)

    page_estimate = len(pdf_bytes) // 5000 + 1
    return f"PDF compiled ({page_estimate} pages). Displayed in artifact panel."


def _compile_latex(tex_path: Path) -> bytes | str:
    """Compile .tex → PDF bytes. Returns error string on failure."""
    # Use the .tex file's directory as the working directory
    work_dir = tex_path.parent

    try:
        result = subprocess.run(
            ["tectonic", "-X", "compile", str(tex_path)],  # noqa: S603, S607
            cwd=work_dir,
            capture_output=True,
            text=True,
            timeout=120,
            check=False,
        )

        if result.returncode != 0:
            error_msg = result.stderr or result.stdout or "Unknown compilation error"
            error_lines = [
                line for line in error_msg.split("\n")
                if "error" in line.lower()
            ][:5]
            friendly = "\n".join(error_lines) if error_lines else error_msg[:500]
            return f"LaTeX compilation failed:\n```\n{friendly}\n```"

        pdf_path = tex_path.with_suffix(".pdf")
        if not pdf_path.exists():
            return "Compilation succeeded but PDF not found."

        return pdf_path.read_bytes()

    except FileNotFoundError:
        return "Tectonic compiler not installed."
    except subprocess.TimeoutExpired:
        return "Compilation timed out (>120s)."


def _build_viewer(pdf_bytes: bytes, title: str) -> str:
    """Build self-contained HTML PDF viewer."""
    pdf_base64 = base64.b64encode(pdf_bytes).decode("ascii")
    safe_title = "".join(c if c.isalnum() or c in " -_" else "_" for c in title)
    filename = f"{safe_title.strip()[:50] or 'notes'}.pdf"

    return PDF_VIEWER_TEMPLATE % {
        "title": title,
        "pdf_base64": pdf_base64,
        "filename": filename,
    }


async def _push_artifact(
    user_id: str,
    html: str,
    chat_id: str | None = None,
    title: str | None = None,
) -> None:
    """Push HTML content to OpenWebUI's artifact panel."""
    settings = get_settings()

    if not settings.openwebui_url or not settings.openwebui_api_key:
        logger.warning("artifact_push_skipped: RALPH_OPENWEBUI_URL or RALPH_OPENWEBUI_API_KEY not set")
        return

    async with httpx.AsyncClient(timeout=30.0) as client:
        resp = await client.post(
            f"{settings.openwebui_url}/api/artifact/push",
            json={
                "user_id": user_id,
                "chat_id": chat_id,
                "content": html,
                "title": title,
            },
            headers={"Authorization": f"Bearer {settings.openwebui_api_key}"},
        )
        if resp.status_code != 200:
            logger.error("artifact_push_failed", status=resp.status_code, body=resp.text[:200])
        else:
            logger.info("artifact_pushed", user_id=user_id, title=title)
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv run ruff check src/ralph/artifacts.py` passes
- [ ] `uv run basedpyright src/ralph/artifacts.py` passes
- [ ] Unit test: `_compile_latex` on a valid .tex file returns bytes
- [ ] Unit test: `_build_viewer` returns HTML string containing the base64 data
- [ ] `make check-agent` passes

#### Manual Verification:
- [ ] `await compile_and_push(Path("test.tex"), user_id)` from a Python shell compiles and pushes successfully
- [ ] Artifact appears in OpenWebUI panel

**Implementation Note**: After completing this phase, pause for manual testing before proceeding to Phase 3.

---

## Phase 3: Hooked FileTools — Auto-Compile on `.tex` Write

### Overview
Wrap Agno's `FileTools` so that any write to a `.tex` file triggers automatic compilation and artifact push. Remove `LaTeXTools` from the agent's tool list.

### Changes Required:

#### 1. New Module: `src/ralph/tools/hooked_file_tools.py`

**File**: `src/ralph/tools/hooked_file_tools.py` (new)

Wraps Agno's FileTools. After any successful write operation, checks if the target was a `.tex` file and triggers compilation.

```python
"""FileTools wrapper with post-write hooks for automatic LaTeX compilation."""

from __future__ import annotations

import asyncio
import logging
from concurrent.futures import ThreadPoolExecutor
from pathlib import Path
from typing import Any

from agno.tools.file import FileTools
from agno.tools import Toolkit

from ralph.artifacts import compile_and_push

logger = logging.getLogger(__name__)

# Thread pool for running async compile_and_push from sync tool context
_executor = ThreadPoolExecutor(max_workers=2)


class HookedFileTools(Toolkit):
    """FileTools with automatic LaTeX compilation on .tex file writes.

    Wraps Agno's FileTools and intercepts write operations. When a .tex file
    is written, automatically compiles it to PDF and pushes the viewer to
    the OpenWebUI artifact panel.
    """

    def __init__(
        self,
        base_dir: Path,
        user_id: str,
        chat_id: str | None = None,
        **kwargs: Any,
    ) -> None:
        self.base_dir = base_dir
        self.user_id = user_id
        self.chat_id = chat_id

        # Create the underlying Agno FileTools
        self._inner = FileTools(base_dir=base_dir)

        # Build our tool list by wrapping inner tools
        tools = []
        for func in self._inner.functions.values():
            tools.append(func.entrypoint)

        super().__init__(name="file_tools", tools=tools, **kwargs)

        # Patch write methods to add post-write hooks
        self._patch_write_methods()

    def _patch_write_methods(self) -> None:
        """Wrap write-capable functions with post-write .tex detection."""
        write_method_names = {"save_file", "write_file", "create_file"}

        for name in write_method_names:
            if name in self._inner.functions:
                original = self._inner.functions[name].entrypoint

                def make_wrapper(orig_fn, method_name):
                    def wrapper(*args, **kwargs):
                        result = orig_fn(*args, **kwargs)
                        # Extract file path from first positional arg or 'file_path'/'path' kwarg
                        file_path = (
                            kwargs.get("file_path")
                            or kwargs.get("path")
                            or (args[1] if len(args) > 1 else None)  # args[0] is RunContext
                        )
                        if file_path and str(file_path).endswith(".tex"):
                            self._trigger_compile(str(file_path))
                        return result
                    wrapper.__name__ = method_name
                    wrapper.__doc__ = orig_fn.__doc__
                    return wrapper

                self.functions[name].entrypoint = make_wrapper(original, name)

    def _trigger_compile(self, file_path: str) -> None:
        """Trigger async compile_and_push from sync tool context."""
        tex_path = self.base_dir / file_path if not Path(file_path).is_absolute() else Path(file_path)

        if not tex_path.exists():
            logger.warning("tex_file_not_found_for_compile", path=str(tex_path))
            return

        logger.info("auto_compile_triggered", path=str(tex_path), user_id=self.user_id)

        # Run the async compile_and_push in the existing event loop
        try:
            loop = asyncio.get_running_loop()
            # Schedule on the event loop — don't block the tool return
            loop.create_task(self._compile_async(tex_path))
        except RuntimeError:
            # No running loop — run in thread pool
            _executor.submit(self._compile_sync, tex_path)

    async def _compile_async(self, tex_path: Path) -> None:
        """Async compilation wrapper with error handling."""
        try:
            result = await compile_and_push(tex_path, self.user_id, self.chat_id)
            logger.info("auto_compile_result", path=str(tex_path), result=result)
        except Exception:
            logger.exception("auto_compile_failed", path=str(tex_path))

    def _compile_sync(self, tex_path: Path) -> None:
        """Sync compilation wrapper for thread pool fallback."""
        try:
            asyncio.run(compile_and_push(tex_path, self.user_id, self.chat_id))
        except Exception:
            logger.exception("auto_compile_failed_sync", path=str(tex_path))
```

**Design note**: The compilation runs as a fire-and-forget async task (`loop.create_task`). The tool call returns immediately to the agent — the PDF push happens in the background. This keeps the agent responsive. If compilation fails, it's logged but doesn't interrupt the agent.

#### 2. Update Server Agent Construction

**File**: `src/ralph/server.py`

Replace `FileTools` and `LaTeXTools` with `HookedFileTools`:

```python
# Remove:
from agno.tools.file import FileTools
from ralph.tools import LaTeXTools

# Add:
from ralph.tools.hooked_file_tools import HookedFileTools

# In generate() function, replace the tools list:
tools=[
    strip_agno_fields(ShellTools(base_dir=workspace)),
    strip_agno_fields(HookedFileTools(
        base_dir=workspace,
        user_id=request.user_id,
        chat_id=request.chat_id,
    )),
    strip_agno_fields(HonchoTools()),
    strip_agno_fields(MemoryBlockTools()),
]
```

#### 3. Update Agent Instructions

**File**: `src/ralph/server.py`

Remove the "Creating Notes" section from `base_instructions` (lines 215-249) that tells the agent about `render_notes` and HTML code blocks. Replace with:

```python
## Creating Notes

You can create professional PDF notes by writing LaTeX files in the workspace.

How to use:
1. Create or edit a .tex file using save_file (e.g., "lecture-notes.tex")
2. The PDF will automatically compile and appear in the student's artifact panel
3. Each time you save the .tex file, the PDF updates automatically

Write complete LaTeX documents with \\documentclass, \\begin{document}, etc.
Use sections, math environments, theorems, and all standard LaTeX features.
The student never sees LaTeX — they only see the beautiful PDF result.

Use this capability proactively when students would benefit from organized notes.
```

#### 4. Update `__init__.py` Exports

**File**: `src/ralph/tools/__init__.py`

Remove `LaTeXTools` export (if present), add `HookedFileTools`.

### Success Criteria:

#### Automated Verification:
- [ ] `uv run ruff check src/ralph/` passes
- [ ] `uv run basedpyright src/ralph/` passes
- [ ] `make verify-agent` passes
- [ ] Existing tests still pass: `make test-agent`

#### Manual Verification:
- [ ] Start Ralph and OpenWebUI locally
- [ ] In chat, ask agent to "create notes on quadratic equations"
- [ ] Agent uses `save_file` to write a `.tex` file (not `render_notes`)
- [ ] PDF appears in artifact panel without HTML in chat
- [ ] Ask agent to "add a section on the discriminant"
- [ ] Agent edits the `.tex` file, PDF updates in artifact panel
- [ ] The `.tex` file persists in workspace (check via workspace API)

**Implementation Note**: After completing this phase, pause for manual testing. This is the critical integration point.

---

## Phase 4: Cleanup and Production Hardening

### Overview
Remove the old `render_notes` tool, add error handling for production edge cases, update Docker config.

### Changes Required:

#### 1. Remove Old LaTeX Tool

**File**: `src/ralph/tools/latex_tools.py`

Keep `PDF_VIEWER_TEMPLATE` and `NOTES_TEMPLATE` (imported by `artifacts.py`). Remove the `LaTeXTools` class and `render_notes` method entirely. The file becomes a templates-only module.

Rename to `src/ralph/tools/latex_templates.py` for clarity, update imports in `artifacts.py`.

#### 2. Add OpenWebUI API Key to Production Config

**File**: `.env.production` / `docker-compose.prod.yml`

```yaml
# In ralph service environment:
- RALPH_OPENWEBUI_URL=http://openwebui:8080
- RALPH_OPENWEBUI_API_KEY=${RALPH_OPENWEBUI_API_KEY}
```

The API key must be created in OpenWebUI admin panel (Settings → Account → API Keys) for an admin user.

#### 3. Compilation Error Feedback

When compilation fails, the agent gets the error string back from `compile_and_push`. But the artifact panel should also show feedback. Push an error artifact:

In `artifacts.py`, after compilation failure, push a simple error HTML:

```python
if isinstance(pdf_bytes, str):
    # Push error to artifact panel so user sees it
    error_html = f"""<html><body style="font-family: sans-serif; padding: 20px;">
    <h2 style="color: #dc3545;">Compilation Error</h2>
    <pre style="background: #f8f9fa; padding: 16px; border-radius: 8px; overflow-x: auto;">{pdf_bytes}</pre>
    </body></html>"""
    await _push_artifact(user_id, error_html, chat_id=chat_id, title="Compilation Error")
    return pdf_bytes
```

#### 4. Tectonic in Docker Image

**File**: `Dockerfile`

Ensure Tectonic is installed in the Ralph container. Add to the Dockerfile if not already present:

```dockerfile
# Install tectonic for LaTeX compilation
RUN cargo install tectonic || apt-get install -y tectonic
```

Or use a pre-built binary. Check what the current Dockerfile does and ensure tectonic is available.

### Success Criteria:

#### Automated Verification:
- [ ] `make verify-agent` passes
- [ ] `docker compose -f docker-compose.prod.yml build ralph` succeeds
- [ ] No references to `render_notes` or `LaTeXTools` in `server.py`

#### Manual Verification:
- [ ] Full end-to-end test in production Docker compose
- [ ] LaTeX compilation error shows error artifact (not just a text message)
- [ ] Agent can iteratively edit and recompile without issues
- [ ] Artifact panel persists across messages in same chat

---

## Testing Strategy

### Unit Tests
- `artifacts.py:_compile_latex` — valid .tex → bytes, invalid .tex → error string
- `artifacts.py:_build_viewer` — returns HTML containing base64 data
- `hooked_file_tools.py` — `.tex` write triggers compile, `.py` write does not

### Integration Tests
- Ralph → OpenWebUI artifact push (requires both services running)
- Full agent flow: prompt → file write → compile → artifact push

### Manual Testing Steps
1. Open OpenWebUI chat with Ralph pipe
2. Ask: "Create notes on the Pythagorean theorem"
3. Verify: Agent writes `.tex` file, PDF appears in artifact panel
4. Ask: "Add a proof section"
5. Verify: PDF updates with new content
6. Ask: "Add some intentionally broken LaTeX like `\badcommand`"
7. Verify: Compilation error shown in artifact panel + agent informed
8. Refresh page → verify artifact panel shows latest PDF (from message history or re-push)

## Performance Considerations

- **Compilation latency**: Tectonic first-run downloads packages (~30s). Subsequent runs ~2-5s. Cache persists in workspace volume.
- **Base64 PDF size**: A 10-page PDF is ~500KB base64. Socket.io handles this fine but we should monitor for very large documents.
- **Fire-and-forget compilation**: The `loop.create_task()` pattern means the agent doesn't wait for compilation. If the user sends another message before compilation finishes, the artifact will still appear when ready.
- **No debouncing**: If the agent writes the `.tex` file 3 times in rapid succession (e.g., creating sections), all 3 will compile. The artifact panel shows the latest. This is acceptable — Tectonic is fast and the push is idempotent (last one wins visually).

## Migration Notes

- Existing chats with HTML code block artifacts will continue working (code-block detection is unchanged)
- No database migration needed
- The `render_notes` tool disappears from the agent's tool list — existing agent instructions in memory blocks that reference it should be updated
- API key for Ralph → OpenWebUI communication must be created manually in OpenWebUI admin

## References

- Research: `thoughts/shared/research/2026-02-10-openwebui-artifact-sideloading.md`
- Prior research: `thoughts/shared/research/2026-01-28-latex-pdf-live-sandbox-artifacts.md`
- Prior research: `thoughts/shared/research/2026-02-05-latex-artifacts-persistent-html-rendering.md`
- OpenWebUI socket: `OpenWebUI/open-webui/backend/open_webui/socket/main.py:693-810`
- OpenWebUI event handler: `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:352-478`
- Artifact stores: `OpenWebUI/open-webui/src/lib/stores/index.ts:93-97`
- Current latex tools: `src/ralph/tools/latex_tools.py`
- Ralph server: `src/ralph/server.py:288-302`
- Docker compose: `docker-compose.prod.yml`
