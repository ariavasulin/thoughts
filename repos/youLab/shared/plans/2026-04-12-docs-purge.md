# Docs Purge — Delete Legacy Letta Docs

## Overview

The legacy Letta/youlab_server architecture was replaced by Ralph/Agno, but 14 docs files still describe Letta as current. Most of these are fully superseded by `Architecture.md` (574 lines, comprehensive, accurate) and the root `README.md`. Rather than rewrite 14 stale docs, delete them and keep the 5 that are already accurate.

This follows up on the Phase 4 work from `thoughts/shared/plans/2026-04-09-repo-slop-cleanup.md`, which already deleted `Letta-Reference.md`, `Letta-Integration.md`, `config-schema.md`, and `module-metadata-schema.md`.

## Current State Analysis

### Docs Inventory (19 files in docs/)

**Already accurate (KEEP):**
| File | Lines | Why |
|---|---|---|
| `Architecture.md` | 574 | Comprehensive Ralph/Agno/Dolt architecture doc |
| `dev-testing.md` | 90 | Current dev workflow SOP |
| `Roadmap.md` | 92 | Current status and next milestone |
| `OpenWebUI-Development.md` | 80 | Frontend dev guide, backend-independent |
| `openwebuiapi.md` | 590 | External OpenWebUI API reference |

**Legacy / stale (DELETE):**
| File | Lines | Key Issue |
|---|---|---|
| `Quickstart.md` | 307 | Letta Docker setup, port 8283, `youlab_server` |
| `Development.md` | 280 | `src/youlab_server/` structure, Letta server setup |
| `Configuration.md` | 210 | All `LETTA_*` env vars, `YOULAB_SERVICE_*` prefix |
| `HTTP-Service.md` | 672 | Legacy `youlab_server` API endpoints |
| `API.md` | 444 | Legacy endpoint reference with `letta_connected` |
| `Agent-System.md` | 534 | Letta SDK agent templates |
| `Agent-Tools.md` | 212 | Letta tool registration |
| `Memory-System.md` | 471 | `MemoryManager`/`PersonaBlock`/Letta blocks |
| `Background-Agents.md` | 281 | Letta-backed `BackgroundAgentRunner` |
| `Pipeline.md` | 267 | `letta_pipe.py`, `LETTA_SERVICE_URL` |
| `Schemas.md` | 235 | Legacy `youlab_server` Pydantic schemas |
| `Honcho.md` | 300 | Mostly current but references `youlab_server` paths, easier to delete (Honcho usage is documented in Architecture.md) |
| `README-MANIFESTO.md` | 235 | Manifesto with stale Letta link — vision content is already in root README |

**Untracked (DELETE):**
| File | Why |
|---|---|
| `youlab-lecture-notes-plan.md` | Draft planning doc, not docs |
| `youlab-system-design.md` | References Ralph but is a design scratchpad, not reference docs |

## Desired End State

`docs/` contains exactly 6 files:
```
docs/
├── _sidebar.md              # Updated navigation
├── Architecture.md           # Comprehensive architecture reference
├── dev-testing.md            # Developer workflow SOP
├── Roadmap.md                # Current status + next milestone
├── OpenWebUI-Development.md  # Frontend dev guide
└── openwebuiapi.md           # OpenWebUI API reference
```

No file in `docs/` references Letta as current architecture. All docsify sidebar links resolve.

### Verification:
```bash
# No deleted files remain
ls docs/ | wc -l  # Should be 6

# No dead sidebar links (all linked files exist)
grep -oP '\(([^)]+\.md)\)' docs/_sidebar.md | tr -d '()' | while read f; do
  [ -f "docs/$f" ] || echo "DEAD LINK: $f"
done

# No Letta references describing it as current (historical mentions in Architecture.md are fine)
grep -ri "letta" docs/ --include="*.md" -l
# Should only return Architecture.md (which has a historical "Evolution from Legacy" section)
```

## What We're NOT Doing

- Rewriting any docs (the remaining 5 are already accurate)
- Moving content from deleted docs elsewhere (Architecture.md + README already cover everything)
- Touching root `README.md` (already current)
- Deleting `src/youlab_server/` or `src/letta_starter/` residual dirs (separate cleanup task)

## Phase 1: Delete Legacy Docs

### Changes Required:

#### 1. Delete 15 files
```bash
git rm docs/Quickstart.md
git rm docs/Development.md
git rm docs/Configuration.md
git rm docs/HTTP-Service.md
git rm docs/API.md
git rm docs/Agent-System.md
git rm docs/Agent-Tools.md
git rm docs/Memory-System.md
git rm docs/Background-Agents.md
git rm docs/Pipeline.md
git rm docs/Schemas.md
git rm docs/Honcho.md
git rm docs/README-MANIFESTO.md

# Untracked files (just rm, not git rm)
rm docs/youlab-lecture-notes-plan.md
rm docs/youlab-system-design.md
```

#### 2. Update `docs/_sidebar.md`
Replace contents with:

```markdown
- [Architecture](Architecture.md)
- [Developer Testing](dev-testing.md)
- [Roadmap](Roadmap.md)
- [OpenWebUI Development](OpenWebUI-Development.md)
- [OpenWebUI API Reference](openwebuiapi.md)
```

### Success Criteria:

#### Automated Verification:
- [ ] `ls docs/ | wc -l` returns 6
- [ ] All sidebar links resolve to existing files
- [ ] `grep -ri "letta" docs/ -l` returns only `Architecture.md`
- [ ] `git status` shows only the expected deletions + `_sidebar.md` modification

#### Manual Verification:
- [ ] Skim `_sidebar.md` — clean, no dead links
- [ ] Confirm no valuable content was lost that isn't covered by Architecture.md or README.md

## References

- Prior cleanup: `thoughts/shared/plans/2026-04-09-repo-slop-cleanup.md` (Phases 1-4)
- Source of truth for current architecture: `docs/Architecture.md`
- Source of truth for getting started: root `README.md`
