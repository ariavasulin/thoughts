---
date: 2025-12-23T19:56:17Z
researcher: ariasulin
git_commit: 5137ec6af499fe9bea3e843194fe4a744d7ca1d3
branch: main
repository: Dealtrail
topic: "Comprehensive codebase documentation for CLAUDE.md creation"
tags: [research, codebase, tracewriter, documentation]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: Codebase Documentation for CLAUDE.md

**Date**: 2025-12-23T11:55:37 PST
**Researcher**: ariasulin
**Git Commit**: 5137ec6af499fe9bea3e843194fe4a744d7ca1d3
**Branch**: main
**Repository**: Dealtrail

## Research Question

Document the Dealtrail/TraceWriter codebase comprehensively to create an effective CLAUDE.md file following best practices (concise, universally applicable, progressive disclosure).

## Summary

**Dealtrail** is a repository containing **TraceWriter**, a React/Vite application for annotating email threads from real estate transactions. The tool captures "off-screen actions" - activities that happened between emails but aren't visible in the email chain itself. This creates training data for an AI agent to understand transaction coordinator workflows.

The project is currently **WIP** with the core annotation functionality complete but export and some UX features still in progress.

## Detailed Findings

### What is TraceWriter?

An email annotation tool built to:
- Process MBOX email exports from real estate transaction coordinators
- Group emails by property address (not traditional threading)
- Allow annotators to document what happened between emails
- Generate training data for AI agents

**Current Status**: 150+ real estate transaction coordinator email threads being annotated.

### Core Components

| Component | Location | Purpose |
|-----------|----------|---------|
| React App | `tracewriter/src/App.jsx` | Main UI with keyboard navigation |
| Email Parser | `tracewriter/src/utils/emailParser.js` | JSON import/export handling |
| MBOX Converter | `tracewriter/scripts/mbox_to_json.py` | Preprocesses email data |
| Analyzer | `tracewriter/scripts/analyze_unmatched_emails.py` | Recovers unmatched threads |
| Linear CLI | `hack/linear/linear-cli.ts` | Project management integration |

### Tech Stack

**Frontend:**
- React 19.2.0
- Vite 7.2.4
- Pure CSS with custom theme variables

**Backend/Data Processing:**
- Python scripts for MBOX preprocessing
- No traditional backend - client-only application

**Developer Tools:**
- ESLint for linting
- TypeScript (for Linear CLI tool only)
- Claude AI workflow automation (26 commands)

### Key Commands

```bash
# TraceWriter development
cd tracewriter && npm run dev    # Start dev server
npm run build                     # Production build
npm run lint                      # Run linter

# Data processing
python tracewriter/scripts/mbox_to_json.py <input.mbox> <output.json>

# Linear CLI (if installed)
linear list                       # List assigned issues
linear show <issue-id>           # Show issue details
```

### Data Flow

1. **Input**: Gmail MBOX export
2. **Processing**: Python script groups emails by property address
3. **Import**: Browser app loads JSON
4. **Annotation**: User fills gaps between emails via keyboard navigation
5. **Export**: JSON with `_annotation_after` fields (WIP)

### Keyboard Navigation (Core UX)

- `Tab / Shift+Tab` - Navigate emails
- `Shift+Arrow` - Switch threads
- `Enter` - Edit annotation
- `Esc` - Exit edit mode
- `Alt+u` - Jump to next unannotated gap (planned)

### Directory Structure

```
Dealtrail/
├── tracewriter/           # Main React application
│   ├── src/               # React components and utils
│   └── scripts/           # Python data processing
├── Mail/                  # Email data (gitignored)
├── plans/                 # Implementation specs
├── hack/                  # Developer utilities
│   └── linear/           # Linear CLI tool
└── .claude/              # Claude AI configuration
    ├── commands/         # 26 workflow commands
    └── agents/           # 8 specialized agents
```

### WIP Status

**Completed:**
- Core annotation UI with keyboard navigation
- JSON import functionality
- MBOX preprocessing pipeline
- Thread sidebar and email navigation

**In Progress/Not Yet Implemented:**
- Export functionality (placeholder exists)
- localStorage persistence
- "Jump to next unannotated" shortcut
- Thread filtering by annotation status
- Clear All annotations button

## Code References

- Main app: `tracewriter/src/App.jsx:36-440`
- Email parser: `tracewriter/src/utils/emailParser.js:1-97`
- MBOX converter: `tracewriter/scripts/mbox_to_json.py:1-484`
- Implementation plan: `plans/2024-12-16-tracewriter-email-annotation-tool.md`

## Architecture Documentation

**Client-Only Architecture**: No backend server. All data processing happens either:
- Offline via Python scripts (heavy preprocessing)
- In-browser via React (annotation and display)

**Keyboard-First UX**: Designed for rapid annotation workflow with minimal mouse usage.

**Property-Based Threading**: Emails grouped by extracted property address rather than traditional Message-ID threading, which is more meaningful for real estate transactions.

## Related Files

- `plans/2024-12-16-tracewriter-email-annotation-tool.md` - Full implementation spec
- `.claude/` directory - Claude AI workflow configuration
- `hack/linear/README.md` - Linear CLI documentation
