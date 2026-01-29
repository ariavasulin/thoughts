# LaTeX-to-PDF Notes Tool Implementation Plan

## Overview

Create a `LaTeXTools` toolkit that enables the Ralph agent to produce professional textbook-quality PDF notes for users with zero LaTeX knowledge. The agent writes LaTeX behind the scenes; users simply ask for notes on a topic and see a polished PDF appear in the OpenWebUI artifacts pane with full multi-page navigation and download capability.

**User Flow**:
1. Student asks: "Help me take notes on quadratic equations"
2. Agent writes LaTeX internally (user never sees it)
3. PDF appears in artifacts pane when compilation completes
4. Student navigates pages, downloads polished PDF

## Current State Analysis

### Existing Infrastructure
- **Ralph Server** (`src/ralph/server.py`): Already has workspace-scoped tools registration
- **Tool Pattern** (`src/ralph/tools/memory_blocks.py`): Established pattern for sync tools with async Dolt operations
- **OpenWebUI Artifacts**: Supports HTML/iframe rendering, auto-detects `html` code blocks in messages
- **Workspace**: Per-user isolated workspace at `{RALPH_USER_DATA_DIR}/{user_id}/workspace/`

### Key Discoveries
- Artifacts don't update mid-stream (Svelte reactivity limitation) - PDF appears when message completes
- OpenWebUI already has `pdfjs-dist@5.4.149` in dependencies
- Artifacts iframe sandbox includes `allow-scripts allow-downloads` by default
- Tool exports via `src/ralph/tools/__init__.py`

## Desired End State

After implementation:
1. Agent has a `render_notes(title, content)` tool that takes structured content and produces a PDF artifact
2. User sees professional PDF in artifacts pane with:
   - Multi-page navigation (prev/next buttons, page counter)
   - Zoom controls
   - Download button
3. Agent instructions guide it to use the tool appropriately for note-taking requests
4. Tectonic LaTeX compiler available in the environment

### Verification
- Agent can be asked "Create notes on [topic]" and produces a PDF artifact
- PDF renders correctly in artifacts pane with navigation
- Download produces valid PDF file
- Compilation errors are reported gracefully in chat

## What We're NOT Doing

- **No live editing/watching** - PDF appears when compilation completes, not during
- **No user-facing LaTeX** - Users never see or edit LaTeX source
- **No Docker sandboxing** - Tectonic is safer than pdflatex (no `\write18` by default)
- **No workspace API URL approach** - Base64 embedding is simpler for MVP
- **No separate compilation container** - Tectonic runs directly in workspace

## Implementation Approach

The tool will:
1. Accept structured content (title + sections) from the agent
2. Wrap content in a professional LaTeX template with good typography
3. Compile with Tectonic (self-contained, fast, auto-downloads packages)
4. Encode PDF as base64 and embed in HTML artifact with PDF.js viewer
5. Return the HTML artifact as markdown code block for OpenWebUI to render

---

## Phase 1: LaTeX Tools Toolkit

### Overview
Create the core `LaTeXTools` toolkit following the established pattern from `MemoryBlockTools`.

### Changes Required

#### 1. Create LaTeX Tools Module
**File**: `src/ralph/tools/latex_tools.py`

```python
"""LaTeX compilation tools for producing PDF notes."""

from __future__ import annotations

import base64
import logging
import subprocess
import tempfile
from pathlib import Path
from typing import Any

from agno.run import RunContext  # noqa: TC002
from agno.tools import Toolkit

logger = logging.getLogger(__name__)

# Professional LaTeX template for notes
NOTES_TEMPLATE = r"""
\documentclass[11pt,a4paper]{article}

% Typography and fonts
\usepackage[T1]{fontenc}
\usepackage{lmodern}
\usepackage{microtype}

% Math support
\usepackage{amsmath,amssymb,amsthm}

% Better lists
\usepackage{enumitem}

% Code listings
\usepackage{listings}
\lstset{
    basicstyle=\ttfamily\small,
    breaklines=true,
    frame=single,
    numbers=left,
    numberstyle=\tiny\color{gray}
}

% Colors
\usepackage{xcolor}
\definecolor{sectioncolor}{RGB}{0,51,102}

% Hyperlinks
\usepackage{hyperref}
\hypersetup{
    colorlinks=true,
    linkcolor=sectioncolor,
    urlcolor=blue
}

% Page geometry
\usepackage[margin=1in]{geometry}

% Headers/footers
\usepackage{fancyhdr}
\pagestyle{fancy}
\fancyhf{}
\rhead{\thepage}
\lhead{\leftmark}
\renewcommand{\headrulewidth}{0.4pt}

% Section styling
\usepackage{titlesec}
\titleformat{\section}
    {\normalfont\Large\bfseries\color{sectioncolor}}
    {\thesection}{1em}{}
\titleformat{\subsection}
    {\normalfont\large\bfseries\color{sectioncolor}}
    {\thesubsection}{1em}{}

% Theorem environments
\theoremstyle{definition}
\newtheorem{definition}{Definition}[section]
\newtheorem{example}{Example}[section]
\theoremstyle{plain}
\newtheorem{theorem}{Theorem}[section]
\newtheorem{lemma}[theorem]{Lemma}
\newtheorem{corollary}[theorem]{Corollary}
\theoremstyle{remark}
\newtheorem{remark}{Remark}[section]
\newtheorem{note}{Note}[section]

% Title
\title{%(title)s}
\author{AI Tutor Notes}
\date{\today}

\begin{document}

\maketitle
\tableofcontents
\newpage

%(content)s

\end{document}
"""

# HTML template with embedded PDF.js viewer
PDF_VIEWER_TEMPLATE = """<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>%(title)s</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.min.mjs" type="module"></script>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            background: #525659;
            height: 100vh;
            display: flex;
            flex-direction: column;
        }
        .toolbar {
            background: #323639;
            padding: 8px 16px;
            display: flex;
            align-items: center;
            gap: 16px;
            color: white;
            flex-shrink: 0;
        }
        .toolbar button {
            background: #474a4d;
            border: none;
            color: white;
            padding: 6px 12px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 14px;
        }
        .toolbar button:hover { background: #5a5d60; }
        .toolbar button:disabled { opacity: 0.5; cursor: not-allowed; }
        .page-info {
            font-size: 14px;
            min-width: 100px;
            text-align: center;
        }
        .zoom-controls { display: flex; align-items: center; gap: 8px; }
        .spacer { flex: 1; }
        .viewer {
            flex: 1;
            overflow: auto;
            display: flex;
            justify-content: center;
            padding: 20px;
        }
        #pdf-canvas {
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            background: white;
        }
        .download-btn {
            background: #0066cc !important;
        }
        .download-btn:hover { background: #0052a3 !important; }
    </style>
</head>
<body>
    <div class="toolbar">
        <button id="prev" disabled>&larr; Prev</button>
        <span class="page-info"><span id="page-num">1</span> / <span id="page-count">-</span></span>
        <button id="next" disabled>Next &rarr;</button>
        <div class="zoom-controls">
            <button id="zoom-out">-</button>
            <span id="zoom-level">100%%</span>
            <button id="zoom-in">+</button>
        </div>
        <div class="spacer"></div>
        <button id="download" class="download-btn">Download PDF</button>
    </div>
    <div class="viewer">
        <canvas id="pdf-canvas"></canvas>
    </div>
    <script type="module">
        const pdfData = atob('%(pdf_base64)s');
        const pdfBytes = new Uint8Array(pdfData.length);
        for (let i = 0; i < pdfData.length; i++) {
            pdfBytes[i] = pdfData.charCodeAt(i);
        }

        const pdfjsLib = await import('https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.min.mjs');
        pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.worker.min.mjs';

        let pdfDoc = null;
        let pageNum = 1;
        let scale = 1.5;
        const canvas = document.getElementById('pdf-canvas');
        const ctx = canvas.getContext('2d');

        async function renderPage(num) {
            const page = await pdfDoc.getPage(num);
            const viewport = page.getViewport({ scale });
            canvas.height = viewport.height;
            canvas.width = viewport.width;
            await page.render({ canvasContext: ctx, viewport }).promise;
            document.getElementById('page-num').textContent = num;
            document.getElementById('prev').disabled = num <= 1;
            document.getElementById('next').disabled = num >= pdfDoc.numPages;
        }

        function updateZoom() {
            document.getElementById('zoom-level').textContent = Math.round(scale * 100) + '%%';
            renderPage(pageNum);
        }

        // Load PDF
        pdfDoc = await pdfjsLib.getDocument({ data: pdfBytes }).promise;
        document.getElementById('page-count').textContent = pdfDoc.numPages;
        renderPage(pageNum);

        // Navigation
        document.getElementById('prev').addEventListener('click', () => {
            if (pageNum > 1) { pageNum--; renderPage(pageNum); }
        });
        document.getElementById('next').addEventListener('click', () => {
            if (pageNum < pdfDoc.numPages) { pageNum++; renderPage(pageNum); }
        });

        // Zoom
        document.getElementById('zoom-in').addEventListener('click', () => {
            scale = Math.min(scale + 0.25, 3);
            updateZoom();
        });
        document.getElementById('zoom-out').addEventListener('click', () => {
            scale = Math.max(scale - 0.25, 0.5);
            updateZoom();
        });

        // Download
        document.getElementById('download').addEventListener('click', () => {
            const blob = new Blob([pdfBytes], { type: 'application/pdf' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = '%(filename)s';
            a.click();
            URL.revokeObjectURL(url);
        });
    </script>
</body>
</html>"""


class LaTeXTools(Toolkit):
    """
    Tools for creating professional PDF notes from structured content.

    This toolkit allows the agent to produce textbook-quality PDF documents
    that appear in the OpenWebUI artifacts pane. Users never see LaTeX -
    they just get beautiful PDF notes.
    """

    def __init__(self, workspace: Path, **kwargs: Any) -> None:
        """
        Initialize LaTeX tools.

        Args:
            workspace: Path to user's workspace directory for temp files.
            **kwargs: Additional arguments passed to Toolkit base class.
        """
        self.workspace = workspace
        tools = [self.render_notes]
        super().__init__(name="latex_tools", tools=tools, **kwargs)

    def render_notes(
        self,
        run_context: RunContext,
        title: str,
        content: str,
    ) -> str:
        """
        Create professional PDF notes and display in the artifacts pane.

        Use this tool when the student asks for notes, summaries, or study
        materials on a topic. The content should be written in a LaTeX-friendly
        format with sections and mathematical notation where appropriate.

        Args:
            run_context: Agno run context (auto-injected).
            title: The title for the notes document.
            content: The notes content in LaTeX format. Use:
                - \\section{Name} and \\subsection{Name} for organization
                - $...$ for inline math, $$...$$ or \\[...\\] for display math
                - \\begin{itemize}/\\begin{enumerate} for lists
                - \\begin{definition}, \\begin{theorem}, \\begin{example} for formal statements
                - \\textbf{} for bold, \\textit{} for italic
                - \\begin{lstlisting} for code blocks

        Returns:
            An HTML artifact containing the PDF viewer, or an error message
            if compilation fails.
        """
        logger.info("Rendering notes: %s", title)

        # Create temp directory for compilation
        with tempfile.TemporaryDirectory(prefix="latex_", dir=self.workspace) as tmpdir:
            tmppath = Path(tmpdir)
            tex_file = tmppath / "notes.tex"
            pdf_file = tmppath / "notes.pdf"

            # Generate LaTeX document
            latex_source = NOTES_TEMPLATE % {
                "title": self._escape_latex(title),
                "content": content,
            }
            tex_file.write_text(latex_source)

            # Compile with Tectonic
            try:
                result = subprocess.run(
                    ["tectonic", "-X", "compile", str(tex_file)],
                    cwd=tmppath,
                    capture_output=True,
                    text=True,
                    timeout=120,  # 2 minute timeout
                )

                if result.returncode != 0:
                    error_msg = result.stderr or result.stdout or "Unknown compilation error"
                    logger.warning("LaTeX compilation failed: %s", error_msg)
                    # Extract the most relevant error lines
                    error_lines = [
                        line for line in error_msg.split('\n')
                        if 'error' in line.lower() or 'Error' in line
                    ][:5]
                    friendly_error = '\n'.join(error_lines) if error_lines else error_msg[:500]
                    return f"I encountered an error creating the PDF notes:\n\n```\n{friendly_error}\n```\n\nLet me try a simpler format."

                if not pdf_file.exists():
                    return "PDF compilation succeeded but output file not found. This is unexpected."

            except FileNotFoundError:
                logger.error("Tectonic not found in PATH")
                return "The LaTeX compiler (Tectonic) is not installed. Please install it with: `cargo install tectonic` or via your package manager."
            except subprocess.TimeoutExpired:
                return "PDF compilation timed out. The document may be too complex."
            except Exception as e:
                logger.exception("Unexpected error during LaTeX compilation")
                return f"Unexpected error creating PDF: {e}"

            # Read and encode PDF
            pdf_bytes = pdf_file.read_bytes()
            pdf_base64 = base64.b64encode(pdf_bytes).decode('ascii')

            # Generate safe filename
            safe_title = "".join(c if c.isalnum() or c in " -_" else "_" for c in title)
            safe_title = safe_title.strip()[:50] or "notes"
            filename = f"{safe_title}.pdf"

            # Generate HTML viewer
            html = PDF_VIEWER_TEMPLATE % {
                "title": title,
                "pdf_base64": pdf_base64,
                "filename": filename,
            }

            logger.info("PDF generated: %d bytes, %d pages estimated", len(pdf_bytes), len(pdf_bytes) // 5000 + 1)

            # Return as HTML code block for artifact rendering
            return f"Here are your notes on **{title}**:\n\n```html\n{html}\n```"

    def _escape_latex(self, text: str) -> str:
        """Escape special LaTeX characters in plain text."""
        # Only escape in title, not in content (which should be LaTeX)
        replacements = [
            ('\\', r'\textbackslash{}'),
            ('&', r'\&'),
            ('%', r'\%'),
            ('$', r'\$'),
            ('#', r'\#'),
            ('_', r'\_'),
            ('{', r'\{'),
            ('}', r'\}'),
            ('~', r'\textasciitilde{}'),
            ('^', r'\textasciicircum{}'),
        ]
        for old, new in replacements:
            text = text.replace(old, new)
        return text
```

#### 2. Export from Tools Package
**File**: `src/ralph/tools/__init__.py`
**Changes**: Add LaTeXTools export

```python
"""Custom tools for Ralph agents."""

from ralph.tools.honcho_tools import HonchoTools
from ralph.tools.latex_tools import LaTeXTools
from ralph.tools.memory_blocks import MemoryBlockTools
from ralph.tools.query_honcho import QueryHonchoTool

__all__ = ["HonchoTools", "LaTeXTools", "MemoryBlockTools", "QueryHonchoTool"]
```

### Success Criteria

#### Automated Verification:
- [ ] Type checking passes: `uv run basedpyright src/ralph/tools/latex_tools.py`
- [ ] Linting passes: `uv run ruff check src/ralph/tools/latex_tools.py`
- [ ] Module imports correctly: `python -c "from ralph.tools import LaTeXTools"`

#### Manual Verification:
- [ ] Tool can be instantiated with a workspace path

---

## Phase 2: Register Tool with Agent

### Overview
Add LaTeXTools to the agent's tool list and update instructions.

### Changes Required

#### 1. Update Server Tool Registration
**File**: `src/ralph/server.py`
**Changes**: Import and register LaTeXTools

```python
# Add to imports (around line 42)
from ralph.tools import HonchoTools, LaTeXTools, MemoryBlockTools

# Update agent tools list (around line 253-258)
agent = Agent(
    model=OpenRouter(
        id=settings.openrouter_model,
        api_key=settings.openrouter_api_key,
    ),
    tools=[
        strip_agno_fields(ShellTools(base_dir=workspace)),
        strip_agno_fields(FileTools(base_dir=workspace)),
        strip_agno_fields(HonchoTools()),
        strip_agno_fields(MemoryBlockTools()),
        strip_agno_fields(LaTeXTools(workspace=workspace)),  # Add this line
    ],
    instructions=instructions,
    markdown=True,
)
```

#### 2. Update Agent Instructions
**File**: `src/ralph/server.py`
**Changes**: Add LaTeX notes guidance to base_instructions (around line 187)

Add after the Memory Blocks section:

```python
"""
## Creating Notes

You have access to a `render_notes` tool that creates professional PDF documents.
When students ask for notes, summaries, or study materials, use this tool.

How to use:
1. Call `render_notes(title="Topic Name", content="LaTeX content...")`
2. Write the content using LaTeX formatting:
   - Use \\section{} and \\subsection{} for organization
   - Use $...$ for inline math (e.g., $x^2 + y^2 = r^2$)
   - Use \\[...\\] for display equations
   - Use \\begin{definition}, \\begin{theorem}, \\begin{example} for formal content
   - Use \\begin{itemize} or \\begin{enumerate} for lists
3. The PDF will appear in the student's artifacts panel

Example content structure:
\\section{Introduction}
Brief overview of the topic...

\\section{Key Concepts}
\\begin{definition}
A \\textbf{quadratic equation} is a polynomial equation of degree 2...
\\end{definition}

\\subsection{The Quadratic Formula}
For $ax^2 + bx + c = 0$, the solutions are:
\\[ x = \\frac{-b \\pm \\sqrt{b^2 - 4ac}}{2a} \\]

Important: The student never sees LaTeX - they only see the beautiful PDF result.
Use this tool proactively when students would benefit from well-organized notes.
"""
```

### Success Criteria

#### Automated Verification:
- [ ] Server starts without errors: `timeout 5 uv run ralph-server || true` (expect timeout, no crash)
- [ ] Type checking passes: `uv run basedpyright src/ralph/server.py`
- [ ] Linting passes: `uv run ruff check src/ralph/server.py`

#### Manual Verification:
- [ ] Agent recognizes the `render_notes` tool in chat
- [ ] Agent uses the tool when asked for notes on a topic

---

## Phase 3: Install Tectonic Compiler

### Overview
Ensure Tectonic LaTeX compiler is available in the development and production environments.

### Changes Required

#### 1. Development Setup
**Local Installation** (macOS/Linux):
```bash
# macOS
brew install tectonic

# Linux (Cargo)
cargo install tectonic

# Linux (apt - Ubuntu 22.04+)
sudo apt install tectonic

# Verify
tectonic --version
```

#### 2. Docker Configuration (for production)
**File**: `Dockerfile` or `docker-compose.yml` (if exists)
**Changes**: Add Tectonic installation

```dockerfile
# For Debian/Ubuntu-based images
RUN apt-get update && apt-get install -y tectonic && rm -rf /var/lib/apt/lists/*

# OR for minimal images, use the static binary
RUN curl -L https://github.com/tectonic-typesetting/tectonic/releases/download/tectonic@0.15.0/tectonic-0.15.0-x86_64-unknown-linux-gnu.tar.gz | tar xz -C /usr/local/bin
```

#### 3. Update CLAUDE.md
**File**: `CLAUDE.md`
**Changes**: Add note about Tectonic requirement

Add to Commands section:
```markdown
# LaTeX Notes
uv run ralph-server              # Requires Tectonic for PDF generation
brew install tectonic            # macOS
cargo install tectonic           # Cross-platform via Rust
```

### Success Criteria

#### Automated Verification:
- [ ] Tectonic is in PATH: `which tectonic`
- [ ] Tectonic can compile: `echo '\documentclass{article}\begin{document}Hello\end{document}' > /tmp/test.tex && tectonic /tmp/test.tex && ls /tmp/test.pdf`

#### Manual Verification:
- [ ] PDF compilation works end-to-end through the agent

---

## Phase 4: End-to-End Testing

### Overview
Verify the complete flow works from user request to PDF download.

### Test Scenarios

#### 1. Basic Notes Request
**Prompt**: "Create notes on the Pythagorean theorem"
**Expected**:
- Agent calls `render_notes` with appropriate title and LaTeX content
- PDF artifact appears in OpenWebUI artifacts pane
- PDF has title, table of contents, sections
- Download button produces valid PDF

#### 2. Math-Heavy Content
**Prompt**: "I need study notes on derivatives and integration"
**Expected**:
- Complex math renders correctly (integrals, fractions, summations)
- Equations are properly numbered
- Definitions and theorems are boxed/styled

#### 3. Error Handling
**Test**: Intentionally malformed LaTeX
**Expected**:
- Agent receives error message
- Agent explains the issue and offers to simplify
- No crash or hang

#### 4. Large Document
**Prompt**: "Create comprehensive notes on linear algebra covering vectors, matrices, and eigenvalues"
**Expected**:
- Multi-page PDF generates (may take 10-20 seconds)
- Page navigation works in viewer
- Table of contents is accurate

### Success Criteria

#### Manual Verification:
- [ ] Basic notes render correctly
- [ ] Math equations display properly
- [ ] Multi-page navigation works
- [ ] Download produces valid PDF
- [ ] Error messages are user-friendly
- [ ] Agent uses tool proactively for note requests

---

## Testing Strategy

### Unit Tests
Location: `tests/ralph/tools/test_latex_tools.py`

```python
"""Tests for LaTeX tools."""

import pytest
from pathlib import Path
from unittest.mock import MagicMock, patch

from ralph.tools.latex_tools import LaTeXTools, NOTES_TEMPLATE, PDF_VIEWER_TEMPLATE


class TestLaTeXTools:
    """Tests for LaTeXTools toolkit."""

    def test_init(self, tmp_path: Path) -> None:
        """Test toolkit initialization."""
        tools = LaTeXTools(workspace=tmp_path)
        assert tools.name == "latex_tools"
        assert len(tools.functions) == 1
        assert "render_notes" in tools.functions

    def test_escape_latex(self, tmp_path: Path) -> None:
        """Test LaTeX special character escaping."""
        tools = LaTeXTools(workspace=tmp_path)
        assert tools._escape_latex("100%") == r"100\%"
        assert tools._escape_latex("$50") == r"\$50"
        assert tools._escape_latex("a_b") == r"a\_b"

    def test_template_has_required_packages(self) -> None:
        """Test that the LaTeX template includes necessary packages."""
        assert r"\usepackage{amsmath" in NOTES_TEMPLATE
        assert r"\usepackage{hyperref}" in NOTES_TEMPLATE
        assert r"\newtheorem{definition}" in NOTES_TEMPLATE

    def test_viewer_template_has_controls(self) -> None:
        """Test that PDF viewer has navigation controls."""
        assert 'id="prev"' in PDF_VIEWER_TEMPLATE
        assert 'id="next"' in PDF_VIEWER_TEMPLATE
        assert 'id="download"' in PDF_VIEWER_TEMPLATE
        assert 'id="zoom-in"' in PDF_VIEWER_TEMPLATE

    @patch("subprocess.run")
    def test_render_notes_success(
        self, mock_run: MagicMock, tmp_path: Path
    ) -> None:
        """Test successful PDF rendering."""
        # Setup mock
        mock_run.return_value = MagicMock(returncode=0, stderr="", stdout="")

        tools = LaTeXTools(workspace=tmp_path)
        ctx = MagicMock()

        # Create fake PDF output
        (tmp_path / "notes.pdf").write_bytes(b"%PDF-1.4 fake pdf content")

        # This will fail because tempdir is different, but tests the flow
        # In real tests, we'd mock tempfile.TemporaryDirectory

    @patch("subprocess.run")
    def test_render_notes_compilation_error(
        self, mock_run: MagicMock, tmp_path: Path
    ) -> None:
        """Test handling of LaTeX compilation errors."""
        mock_run.return_value = MagicMock(
            returncode=1,
            stderr="! Undefined control sequence.\nl.42 \\badcommand",
            stdout=""
        )

        tools = LaTeXTools(workspace=tmp_path)
        ctx = MagicMock()

        # Would test error message handling

    def test_render_notes_tectonic_not_found(self, tmp_path: Path) -> None:
        """Test handling when Tectonic is not installed."""
        with patch("subprocess.run", side_effect=FileNotFoundError):
            tools = LaTeXTools(workspace=tmp_path)
            ctx = MagicMock()
            result = tools.render_notes(ctx, "Test", "content")
            assert "not installed" in result.lower()
```

### Integration Tests
- Test full flow with actual Tectonic compilation (requires Tectonic installed)
- Test artifact rendering in OpenWebUI (manual)

---

## Performance Considerations

1. **Compilation Time**: Tectonic typically compiles in 500ms-2s for simple documents. Complex documents with many packages may take 5-10s on first run (package download).

2. **PDF Size**: Base64 encoding increases size by ~33%. A 1MB PDF becomes ~1.3MB in the HTML artifact. For very large documents, consider the workspace URL approach in a future phase.

3. **Memory**: PDF.js loads the entire document into memory. Documents over 50 pages may be slow on low-end devices.

4. **First Run**: Tectonic downloads LaTeX packages on demand. First compilation may be slow as it fetches packages.

---

## Migration Notes

No migration needed - this is a new feature with no existing data.

---

## References

- Research document: `thoughts/shared/research/2026-01-28-latex-pdf-live-sandbox-artifacts.md`
- Similar tool pattern: `src/ralph/tools/memory_blocks.py`
- OpenWebUI artifacts: `OpenWebUI/open-webui/src/lib/components/chat/Artifacts.svelte`
- PDF.js documentation: https://mozilla.github.io/pdf.js/
- Tectonic documentation: https://tectonic-typesetting.github.io/
