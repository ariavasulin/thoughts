# Ralph Quick Cleanup Implementation Plan

## Overview

Quick cleanup to remove unused OpenHands code from Ralph, fix documentation drift, and clean up dependencies. This is hygiene work to support ongoing refactoring.

## Current State Analysis

### Files to Remove
- `src/ralph/openhands_client.py` (219 lines) - OpenHands phased out, Agno-only
- `src/ralph/tools/query_honcho.py` (88 lines) - Depends on OpenHands BaseTool
- `src/ralph/tools/__init__.py` (131 bytes) - Only exports unused tool

### Documentation Drift
- CLAUDE.md references `ralph/ralph/` but code is at `src/ralph/`
- CLAUDE.md mentions files that don't exist (`ralph.sh`, `progress.txt`, `ralph/CLAUDE.md`)

### Dependency Issues
- `agno` is imported but not in pyproject.toml
- Duplicate dependencies between core and ralph optional group

## Desired End State

After this plan:
1. No OpenHands code remains in ralph
2. CLAUDE.md accurately reflects `src/ralph/` structure
3. pyproject.toml has correct, deduplicated dependencies
4. `make check-agent` passes

## What We're NOT Doing

- Removing legacy `src/youlab_server/` (still needed)
- Adding tests for Ralph
- Dolt migration
- Any feature work

## Implementation Approach

Single phase - all changes are small and independent.

## Phase 1: Quick Cleanup

### Overview
Delete unused files, fix docs, clean deps.

### Changes Required:

#### 1. Delete Unused OpenHands Files

**Delete**: `src/ralph/openhands_client.py`
**Delete**: `src/ralph/tools/query_honcho.py`
**Delete**: `src/ralph/tools/__init__.py`
**Delete**: `src/ralph/tools/` (directory, now empty)

#### 2. Fix CLAUDE.md Project Structure

**File**: `CLAUDE.md`
**Changes**: Update ralph section to reflect actual `src/ralph/` location

```markdown
## Project Structure

```
src/ralph/                   # NEW: Agno-based MVP (active development)
  __init__.py                # Exports Pipe
  pipe.py                    # OpenWebUI Pipe (HTTP client to Ralph server)
  server.py                  # FastAPI backend with Agno agent
  honcho.py                  # Honcho client for message persistence
  config.py                  # Pydantic settings (RALPH_* env vars)

src/youlab_server/           # LEGACY: Letta-based backend (being phased out)
  ...
```
```

Also update codebase patterns section:
```markdown
## Codebase Patterns

- Use `model_config = {"env_prefix": "..."}` instead of nested `class Config` for Pydantic Settings
- OpenWebUI pipe docstrings need proper format: summary line ending with period, then metadata
- The ralph package lives at `src/ralph/`
- Run checks from the YouLab root: `make check-agent` - this runs on src/ and tests/
- For ralph-specific checks: `uv run ruff check src/ralph/` and `uv run basedpyright src/ralph/`
- Agno tools need `strip_agno_fields()` helper to remove fields Mistral doesn't accept
```

#### 3. Fix pyproject.toml

**File**: `pyproject.toml`
**Changes**:
1. Add `agno` to ralph dependencies
2. Remove duplicates from ralph group (keep only ralph-specific deps)
3. Fix package path from `ralph/ralph` to `src/ralph`

Current ralph section (lines 54-63):
```toml
ralph = [
    "fastapi>=0.115.11",
    "uvicorn[standard]>=0.34.2",
    "sse-starlette>=2.2.1",
    "httpx>=0.28.1",
    "pydantic-settings>=2.8.1",
    "structlog>=25.1.0",
    "httpx-sse>=0.4.0",
]
```

Replace with:
```toml
ralph = [
    "agno>=1.4.5",
    "sse-starlette>=2.2.1",
    "httpx-sse>=0.4.0",
]
```

Note: fastapi, uvicorn, httpx, pydantic-settings, structlog are already in core deps.

Also fix line 60:
```toml
# FROM:
packages = ["src/youlab_server", "ralph/ralph"]
# TO:
packages = ["src/youlab_server", "src/ralph"]
```

### Success Criteria:

#### Automated Verification:
- [ ] Files deleted: `ls src/ralph/openhands_client.py` returns "No such file"
- [ ] Files deleted: `ls src/ralph/tools/` returns "No such file or directory"
- [ ] Lint passes: `uv run ruff check src/ralph/`
- [ ] Type check passes: `uv run basedpyright src/ralph/`
- [ ] Full check passes: `make check-agent`

#### Manual Verification:
- [ ] CLAUDE.md project structure matches `ls src/ralph/`
- [ ] `uv sync` succeeds with updated deps

---

## Testing Strategy

### Automated:
- `make check-agent` covers lint + typecheck

### Manual:
- Verify `uv run ralph-server` still starts (if deps are correct)

## References

- Research: `thoughts/shared/research/2026-01-28-ralph-agno-architecture.md`
