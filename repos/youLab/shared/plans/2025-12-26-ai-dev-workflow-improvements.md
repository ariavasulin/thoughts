# AI Development Workflow Improvements - Final Plan

## Problem Statement

The YouLab repository has **broken tooling references** inherited from HumanLayer:

1. **Worktree scripts are inconsistent and broken**:
   - `create_worktree.sh` uses `~/wt/${REPO}/`
   - `cleanup_worktree.sh` uses `~/.humanlayer/worktrees/` (completely different!)
   - Neither location exists by default (requires manual mkdir)
   - `thoughts init --directory humanlayer` is wrong for YouLab

2. **Command files reference non-existent Makefile targets**:
   - No Makefile exists at all
   - Commands reference `make check test`, `make setup`, `make test-component`, etc.
   - `create_worktree.sh` calls `make setup` (fails immediately)

3. **No automated quality enforcement**:
   - Ruff and mypy are configured in pyproject.toml but never run automatically
   - No pre-commit hooks
   - Easy to commit broken code

## Goals

1. **Working worktree workflow**: Scripts that actually work out of the box
2. **Automated quality gates**: Can't commit code that fails lint/typecheck/tests
3. **Consistent commands**: All references point to real, working targets
4. **Clear developer experience**: `CLAUDE.md` documents the actual workflow

---

## Decisions (Finalized)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Worktree location | `.trees/` in repo | Self-contained, no setup required, obvious location |
| Pre-commit auto-fix | Yes | Auto-fix formatting/lint; block on type errors |
| Tests on commit | Yes (full verify) | Catch issues early, don't push broken code |
| `make setup` installs pre-commit | Yes | "Pit of success" - one command gets you ready |
| Additional Makefile targets | No | YAGNI - keep Makefile focused on build/test |
| CLAUDE.md format | `make` primary, keep `uv run letta-starter` for CLI | Don't show two ways to do the same thing |

---

## Current State Summary

| Item | Current State | Problem |
|------|--------------|---------|
| Makefile | Does not exist | Scripts call `make setup` and fail |
| Pre-commit | Not configured | No quality enforcement |
| `.gitignore` | Missing `.trees/`, `.ruff_cache/` | Would track worktrees if added |
| `create_worktree.sh` | Uses `~/wt/`, calls `make setup` | Fails immediately |
| `cleanup_worktree.sh` | Uses `~/.humanlayer/worktrees/` | Different path than create! |
| 5 command files | Hardcode `~/wt/humanlayer/` | Wrong repo, wrong path |
| 17 command files | Reference `make check test` | Target doesn't exist |
| `CLAUDE.md` | Documents `uv run` commands | Fine, but needs Makefile |

---

## Implementation Plan

### Pre-Implementation: Verify Baseline

Before enabling pre-commit hooks, verify the codebase passes checks:

```bash
# Run manually first to catch existing issues
uv sync --all-extras
uv run ruff check src/ tests/
uv run ruff format --check src/ tests/
uv run mypy src/
uv run pytest
```

**If any of these fail**: Fix them BEFORE creating the Makefile and pre-commit hooks. Otherwise, the hooks will block all commits immediately.

---

### Phase 1: Core Infrastructure

Create the foundational tooling that everything else depends on.

#### 1.1 Create Makefile

**File**: `Makefile` (new)

```makefile
# Makefile for YouLab
# All commands use uv for dependency management

.PHONY: help setup lint lint-fix typecheck test check verify clean

help:
	@echo "YouLab Development Commands"
	@echo ""
	@echo "  setup      Install dependencies and configure dev environment"
	@echo "  lint       Run ruff linter (check mode)"
	@echo "  lint-fix   Run ruff with auto-fix"
	@echo "  typecheck  Run mypy type checker"
	@echo "  test       Run pytest"
	@echo "  check      Run lint + typecheck (no tests)"
	@echo "  verify     Run lint + typecheck + tests (full verification)"
	@echo "  clean      Remove cache directories"

setup:
	uv sync --all-extras
	@echo "Installing pre-commit hooks..."
	uv run pre-commit install
	@echo ""
	@echo "Setup complete! Run 'make verify' to check everything works."

lint:
	uv run ruff check src/ tests/
	uv run ruff format --check src/ tests/

lint-fix:
	uv run ruff check --fix src/ tests/
	uv run ruff format src/ tests/

typecheck:
	uv run mypy src/

test:
	uv run pytest

check: lint typecheck

verify: check test
	@echo ""
	@echo "All checks passed!"

clean:
	rm -rf .pytest_cache .mypy_cache .ruff_cache
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
```

**Note**: `pre-commit` needs to be added to dev dependencies in `pyproject.toml`.

#### 1.2 Update pyproject.toml

**File**: `pyproject.toml`

Add `pre-commit` to dev dependencies (line ~17):
```toml
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "ruff>=0.4",
    "mypy>=1.10",
    "pre-commit>=3.7",
]
```

#### 1.3 Create Pre-commit Config

**File**: `.pre-commit-config.yaml` (new)

```yaml
# Pre-commit hooks for YouLab
# Runs full verification (lint + typecheck + tests) before each commit

repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: ruff check (with auto-fix)
        entry: uv run ruff check --fix
        language: system
        types: [python]

      - id: ruff-format
        name: ruff format
        entry: uv run ruff format
        language: system
        types: [python]

      - id: mypy
        name: mypy typecheck
        entry: uv run mypy src/
        language: system
        types: [python]
        pass_filenames: false

      - id: pytest
        name: pytest
        entry: uv run pytest
        language: system
        types: [python]
        pass_filenames: false
```

#### 1.4 Update .gitignore

**File**: `.gitignore`

Add at end of file:
```gitignore

# Git worktrees (in-repo)
.trees/

# Ruff cache
.ruff_cache/
```

#### 1.5 Update CLAUDE.md

**File**: `CLAUDE.md`

Replace lines 31-48 (Commands section) with:

```markdown
## Commands

```bash
# Setup (run once, installs deps + pre-commit hooks)
make setup

# Run the CLI
uv run letta-starter             # Interactive mode (requires letta server)
uv run letta-starter --agent X   # Use specific agent

# Verification (run before committing)
make verify                      # Full: lint + typecheck + tests
make check                       # Quick: lint + typecheck only

# Individual tools
make lint                        # Ruff check + format check
make lint-fix                    # Auto-fix lint issues
make typecheck                   # Mypy
make test                        # Pytest
```

Pre-commit hooks run `make verify` automatically - commits are blocked if checks fail.

Requires Letta server: `pip install letta && letta server`
```

---

### Phase 2: Fix Worktree Scripts

#### 2.1 Update create_worktree.sh

**File**: `hack/create_worktree.sh`

**Change 1 - Lines 43-54** (path logic):
```bash
# Get repository root
REPO_ROOT=$(git rev-parse --show-toplevel)

if [ ! -z "$HUMANLAYER_WORKTREE_OVERRIDE_BASE" ]; then
    WORKTREES_BASE="${HUMANLAYER_WORKTREE_OVERRIDE_BASE}"
else
    WORKTREES_BASE="${REPO_ROOT}/.trees"
fi

WORKTREE_PATH="${WORKTREES_BASE}/${WORKTREE_NAME}"
```

**Change 2 - Lines 59-64** (auto-create directory):
```bash
# Create worktrees directory if it doesn't exist
if [ ! -d "$WORKTREES_BASE" ]; then
    echo "Creating worktrees directory: $WORKTREES_BASE"
    mkdir -p "$WORKTREES_BASE"
fi
```

**Change 3 - Line 123** (thoughts directory):
```bash
if humanlayer thoughts init --directory YouLab > /dev/null 2>&1; then
```

**Change 4 - Lines 103-117** (uncomment and update verification):
```bash
echo "Verifying worktree with full checks..."
temp_output=$(mktemp)
if make verify > "$temp_output" 2>&1; then
    rm "$temp_output"
    echo "All checks pass!"
else
    cat "$temp_output"
    rm "$temp_output"
    echo "Verification failed. Cleaning up worktree..."
    cd - > /dev/null
    git worktree remove --force "$WORKTREE_PATH"
    git branch -D "$WORKTREE_NAME" 2>/dev/null || true
    echo "Cannot create worktree from a branch that fails verification."
    exit 1
fi
```

#### 2.2 Update cleanup_worktree.sh

**File**: `hack/cleanup_worktree.sh`

**Change 1 - Lines 11-12** (path logic - use absolute path for grep compatibility):
```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_BASE_DIR="${REPO_ROOT}/.trees"
```

**Change 2 - Line 23** (list function grep - already uses $WORKTREE_BASE_DIR which is now absolute):
```bash
# No change needed - the absolute path from Change 1 makes this work
git worktree list | grep -E "^${WORKTREE_BASE_DIR}"
```

**Change 3 - Line 11** (remove unused REPO_BASE_NAME):
```bash
# Delete this line - it's only used for the prefix we're removing
# REPO_BASE_NAME=$(basename "$(git rev-parse --show-toplevel)")
```

**Change 4 - Line 32** (remove REPO_BASE_NAME prefix - create doesn't use it):
```bash
# Old: local worktree_path="$WORKTREE_BASE_DIR/${REPO_BASE_NAME}_${worktree_name}"
# New: matches create script's format
local worktree_path="$WORKTREE_BASE_DIR/${worktree_name}"
```

---

### Phase 3: Fix Command File References

#### 3.1 Fix Hardcoded Paths

| File | Line(s) | Current | New |
|------|---------|---------|-----|
| `create_worktree.md` | 27 | `~/wt/humanlayer/ENG-XXXX` | `.trees/ENG-XXXX` |
| `create_worktree.md` | 36, 41 | `-w ~/wt/humanlayer/ENG-XXXX` | `-w .trees/ENG-XXXX` |
| `local_review.md` | 26 | `~/wt/humanlayer/SHORT_NAME` | `.trees/SHORT_NAME` |
| `local_review.md` | 47 | `~/wt/humanlayer/eng-1696` | `.trees/eng-1696` |
| `ralph_impl.md` | 31 | `-w ~/wt/humanlayer/ENG-XXXX` | `-w .trees/ENG-XXXX` |

#### 3.2 Fix Make Target References

**Replace `make check test` with `make verify`:**

| File | Line |
|------|------|
| `validate_plan.md` | 27 |
| `validate_plan.md` | 96 |
| `implement_plan.md` | 46 |
| `cl/implement_plan.md` | 42 |
| `describe_pr.md` | 44 |
| `describe_pr_nt.md` | 58 |
| `ci_describe_pr.md` | 43 |
| `cl/describe_pr.md` | 58 |

**Replace `make test-component` / `make test-integration` with `make test`:**

| File | Lines |
|------|-------|
| `create_plan.md` | 229, 232 |
| `create_plan_nt.md` | 224, 227 |
| `create_plan_generic.md` | 228, 231 |
| `cl/create_plan.md` | 225, 228 |

**Note**: References to generic `make test` are fine (target exists).

---

## Success Criteria

### Pre-Implementation Complete When:
- [ ] `uv run ruff check src/ tests/` passes (or issues fixed)
- [ ] `uv run ruff format --check src/ tests/` passes (or issues fixed)
- [ ] `uv run mypy src/` passes (or issues fixed)
- [ ] `uv run pytest` passes (or issues fixed)

### Phase 1 Complete When:
- [ ] `make setup` installs dependencies and pre-commit hooks
- [ ] `make verify` runs lint + typecheck + tests
- [ ] `make lint` catches a deliberate lint violation
- [ ] Pre-commit hook blocks a commit with a type error
- [ ] Pre-commit hook auto-fixes a formatting issue and includes it
- [ ] `.trees/` directory is gitignored
- [ ] `pre-commit` is in pyproject.toml dev dependencies

### Phase 2 Complete When:
- [ ] `./hack/create_worktree.sh test-wt` creates `.trees/test-wt/`
- [ ] Directory is auto-created (no manual mkdir needed)
- [ ] Worktree has thoughts initialized with `--directory YouLab`
- [ ] `make verify` passes in worktree
- [ ] `./hack/cleanup_worktree.sh test-wt` removes worktree cleanly

### Phase 3 Complete When:
- [ ] `grep -r "~/wt/humanlayer" .claude/commands/` returns nothing
- [ ] `grep -r "make check test" .claude/commands/` returns nothing
- [ ] `grep -r "test-component\|test-integration" .claude/commands/` returns nothing

---

## What We're NOT Doing

- **GitHub Actions CI** - Pre-commit is sufficient for now; can add CI later
- **Pre-push hooks** - Just pre-commit; keeps workflow simple
- **Complex test categorization** - Single `make test` target is enough
- **`make run` target** - `uv run letta-starter` is already simple and accepts flags
- **Backwards compatibility with `~/wt/`** - Clean break to `.trees/`

---

## File Change Summary

| File | Action | Description |
|------|--------|-------------|
| `Makefile` | Create | Build/test automation |
| `.pre-commit-config.yaml` | Create | Git hooks configuration |
| `pyproject.toml` | Edit | Add `pre-commit` to dev deps |
| `.gitignore` | Edit | Add `.trees/`, `.ruff_cache/` |
| `CLAUDE.md` | Edit | Update Commands section |
| `hack/create_worktree.sh` | Edit | Fix paths, auto-create dir, fix thoughts |
| `hack/cleanup_worktree.sh` | Edit | Fix paths to match create script |
| `.claude/commands/create_worktree.md` | Edit | Fix hardcoded paths |
| `.claude/commands/local_review.md` | Edit | Fix hardcoded paths |
| `.claude/commands/ralph_impl.md` | Edit | Fix hardcoded paths |
| `.claude/commands/validate_plan.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/implement_plan.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/describe_pr.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/describe_pr_nt.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/ci_describe_pr.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/create_plan.md` | Edit | Fix test target references |
| `.claude/commands/create_plan_nt.md` | Edit | Fix test target references |
| `.claude/commands/create_plan_generic.md` | Edit | Fix test target references |
| `.claude/commands/cl/implement_plan.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/cl/describe_pr.md` | Edit | `make check test` → `make verify` |
| `.claude/commands/cl/create_plan.md` | Edit | Fix test target references |
| `.claude/commands/cl/iterate_plan.md` | Edit | Fix generic make references |
| `.claude/commands/iterate_plan.md` | Edit | Fix generic make references |
| `.claude/commands/iterate_plan_nt.md` | Edit | Fix generic make references |

**Total**: 2 new files, 22 edited files

---

## Implementation Order

1. **Phase 1** first - Creates the Makefile that Phase 2 depends on
2. **Phase 2** second - Fixes worktree scripts (now that `make setup` exists)
3. **Phase 3** third - Updates all command file references

Each phase can be tested independently before moving to the next.
