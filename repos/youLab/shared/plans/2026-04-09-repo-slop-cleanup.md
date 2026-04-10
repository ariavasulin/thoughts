# Repo Slop Cleanup — Employer-Ready Presentation

## Overview

Clean the YouLab repo of dead code, legacy artifacts, AI-generated comment slop, and
broken config so it's presentable to employers. Focus on things that ARE there that
SHOULDN'T be. No new features, no refactoring logic — pure deletion and config fixes.

## What We're NOT Doing

- Refactoring working code (no extracting `get_workspace_path` to shared module, etc.)
- Adding missing features, tests, or docs
- Rewriting git history (already done for CLAUDE.md)
- Touching `openwebui-content-sync/` or `tmp/` (untracked, invisible on GitHub)

## Phase 1: First Impressions (what employers see in 5 seconds)

### 1a. Replace root README stub with real README
**File**: `README.md` (root)
**Change**: Replace 3-line stub with contents of `docs/README.md`, then delete `docs/README.md` and update `docs/_sidebar.md` to not link the old location.

### 1b. Fix `.env.example`
**File**: `.env.example`
**Change**: Replace entire file with a Ralph-stack `.env.example` modeled on `.env.production.example`. Remove Letta, Langfuse, and `CORE_MEMORY_MAX_CHARS` sections.

### 1c. Fix manifesto typos
**File**: `docs/README-MANIFESTO.md`
**Changes**:
- Line 7: `probally` → `probably`
- Line 16: `presepece` → `precipice`
- Line 16: `[https://hdsr.mitpress.mit.edu/pub/ujvharkk/release/1 | Intention]` → `[Intention](https://hdsr.mitpress.mit.edu/pub/ujvharkk/release/1)`

### Success Criteria:
- [ ] Root README.md is the full architecture doc
- [ ] `.env.example` only references Ralph-stack config
- [ ] No typos in manifesto

---

## Phase 2: Delete the Dead (12k+ lines of legacy code)

### 2a. Delete `src/youlab_server/`
**58 files, 12,418 lines.** Nothing in `src/ralph/` imports from it.

### 2b. Delete legacy tests
Delete all test files that only import from `youlab_server`:
- `tests/conftest.py`
- `tests/test_background_runner.py`
- `tests/test_background_schema.py`
- `tests/test_curriculum_loader.py`
- `tests/test_honcho.py`
- `tests/test_memory.py`
- `tests/test_memory_enricher.py`
- `tests/test_pipe.py`
- `tests/test_templates.py`
- `tests/test_storage/` (entire directory)
- `tests/test_server/` (entire directory)
- `tests/test_server_honcho.py`

**Keep**: `tests/ralph/` (active tests) and `tests/__init__.py`

### 2c. Delete legacy config
- `config/courses/` (entire directory — only loaded by `youlab_server`)
- `config/course-new.toml` (scratch file, nothing references it)

### 2d. Delete dead Ralph files
- `src/ralph/tools/query_honcho.py` (OpenHands-era, entire file unused)
- Remove `QueryHonchoTool` from `src/ralph/tools/__init__.py` exports

### Success Criteria:
- [ ] `src/youlab_server/` doesn't exist
- [ ] `tests/` only contains `tests/__init__.py` and `tests/ralph/`
- [ ] `config/courses/` doesn't exist
- [ ] `src/ralph/tools/query_honcho.py` doesn't exist
- [ ] `uv run pytest tests/ralph/` passes

---

## Phase 3: Fix Build Config

### 3a. `pyproject.toml`
**Changes**:
1. Remove `youlab` and `youlab-server` entry points (keep `ralph-server`)
2. Remove `[dependency-groups].legacy` section entirely
3. Remove `[project.optional-dependencies].dev` section (keep only `[dependency-groups].dev`)
4. Change `[tool.pytest.ini_options].addopts` from `--cov=src/youlab_server` to `--cov=src/ralph`
5. Change `[tool.coverage.run].source` from `["src/youlab_server"]` to `["src/ralph"]`
6. Remove `src/youlab_server` from `[tool.hatch.build.targets.wheel].packages`

### 3b. `Makefile`
**Changes**:
1. `coverage` target: `--cov=src/youlab_server` → `--cov=src/ralph`
2. `coverage-html` target: `--cov=src/youlab_server` → `--cov=src/ralph`

### Success Criteria:
- [x] `uv run pytest tests/ralph/` measures coverage of `src/ralph`
- [x] `make coverage` points at `src/ralph`
- [x] No references to `youlab_server` remain in pyproject.toml or Makefile
- [x] `uv sync` succeeds

---

## Phase 4: Clean Docs

### 4a. Delete Letta docs
- `docs/Letta-Reference.md`
- `docs/Letta-Integration.md`
- `docs/config-schema.md` (TOML course config for legacy stack)
- `docs/module-metadata-schema.md` (same)

### 4b. Update `docs/_sidebar.md`
Remove the Letta section and links to deleted files.

### 4c. Update `docs/Roadmap.md`
Remove/rewrite references to Letta architecture if they describe it as current.
(This may need a judgment call on how much to rewrite vs. delete.)

### Success Criteria:
- [x] No Letta-*.md files in docs/
- [x] `_sidebar.md` has no dead links
- [x] Docsify site doesn't 404 on any sidebar link

---

## Phase 5: Delete Dead Code Within `src/ralph/`

Delete these unused items:

| Item | File | Lines |
|------|------|-------|
| `get_block_for_agent()` | `memory.py` | 139-159 |
| `TaskRegistry.list_enabled()` | `background/registry.py` | 85-87 |
| `BackgroundScheduler.run_task_now()` | `background/scheduler.py` | 147-160 |
| `KnowledgeService.get_knowledge_id()` | `sync/knowledge.py` | 80-107 |
| `KnowledgeService.clear_cache()` | `sync/knowledge.py` | 109-120 |
| `OpenWebUIClient.get_file()` | `sync/openwebui_client.py` | 215-226 |
| `BlockListResponse` | `api/blocks.py` | 29-31 |
| `ProposalResponse` | `api/blocks.py` | 60-68 |
| `get_workspace_sync()` + `WorkspaceSyncDep` | `api/workspace.py` | 66-80 |
| `DoltClient.get_user_activity()` | `dolt.py` | 863-880 |
| `NOTES_TEMPLATE` | `tools/latex_templates.py` | 5-87 |
| 4 sandbox config fields | `config.py` | 35-38 |
| `conversations_dir` config field | `config.py` | 29 |
| Unused `__init__.py` re-exports | `api/__init__.py`, `sync/__init__.py` | various |

### Success Criteria:
- [ ] `uv run pytest tests/ralph/` still passes
- [ ] `uv run basedpyright src/ralph/` still passes (or no new errors)
- [ ] `uv run ruff check src/ralph/` passes

---

## Phase 6: Strip AI-Slop Comments (most tedious, biggest visual impact)

This is the most pervasive problem. Every file has it. The patterns to remove:

### 6a. Kill the `Args:/Returns:` docstrings
Files with the worst concentration:
- `sync/openwebui_client.py` (10 methods)
- `sync/knowledge.py` (5 methods)
- `sync/workspace_sync.py` (~8 methods)
- `api/workspace.py` (5 endpoints)
- `tools/memory_blocks.py` (3 methods)
- `tools/honcho_tools.py` (1 method)
- `memory.py` (2 functions)

**Rule**: If the docstring just restates the function signature, delete it entirely. Keep
docstrings that explain *why* or describe non-obvious behavior.

### 6b. Kill `# VERB the NOUN` comments
`dolt.py` alone has ~12. `server.py` has ~8. `sync/workspace_sync.py` has ~20.

**Rule**: If the comment restates what the next line of code does, delete it. Keep comments
that explain *why* something is done or describe non-obvious behavior.

### 6c. Kill section banners
```python
# =========================================================================
# Helper Functions
# =========================================================================
```
Found in `notes_adapter.py`, `api/blocks.py`, `api/background.py`, `dolt.py`.

### 6d. Kill `# Module-level singleton` (4 copies)
Same comment on the same pattern in `dolt.py`, `honcho.py`, `registry.py`, `scheduler.py`.

### 6e. Kill attribute docstrings on Pydantic models
`sync/models.py` has `"""Field description."""` below every field.

### Success Criteria:
- [ ] No docstring contains `Args:` followed by parameter descriptions that restate type hints
- [ ] No `# ====` section banners remain
- [ ] No `# VERB the NOUN` comments that restate the next line
- [ ] Code still passes ruff/pyright/tests

---

## Execution Order

Phases 1-5 are pure deletion/config fixes — zero risk of breaking logic. Do them first
in a single commit or two. Phase 6 is tedious manual editing across ~15 files — do it as
a separate pass.

Estimated effort: Phases 1-5 ~20 min, Phase 6 ~30 min.
