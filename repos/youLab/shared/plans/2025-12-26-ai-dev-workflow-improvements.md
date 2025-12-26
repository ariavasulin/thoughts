# AI Development Workflow Improvements Implementation Plan

## Overview

Improve the AI-assisted development workflow by:
1. Moving worktrees to `.trees/` (in-repo, gitignored)
2. Adding strict linting enforcement via pre-commit hooks and Makefile
3. Updating all commands to use consistent verification patterns

## Current State Analysis

### Worktrees
- `hack/create_worktree.sh` uses `~/wt/${REPO_NAME}/`
- `hack/cleanup_worktree.sh` uses `~/.humanlayer/worktrees/` (inconsistent!)
- Commands hardcode `~/wt/humanlayer/` paths
- `humanlayer thoughts init --directory humanlayer` parameter may be wrong for YouLab

### Linting
- Ruff and mypy configured in pyproject.toml
- No automated enforcement (no hooks, no Makefile)
- Commands reference `make check test` but no Makefile exists
- commit.md has no verification requirement

## Desired End State

After implementation:
- Worktrees created at `${REPO_ROOT}/.trees/<name>` (gitignored)
- `make verify` runs lint + typecheck + tests
- Pre-commit hook blocks commits with lint/type errors
- All commands use `make verify` consistently
- Commit workflow requires passing verification

### Key Discoveries:
- 17 command files reference non-existent `make` targets
- Create and cleanup scripts use completely different path formats
- Thoughts init uses `--directory humanlayer` but repo is YouLab

## What We're NOT Doing

- Vertical slice architecture restructuring (defer until multi-course)
- GitHub Actions CI/CD (can add later)
- Complex pre-push hooks (just pre-commit for now)

## Implementation Approach

Three phases:
1. Create infrastructure (Makefile, pre-commit config, .gitignore)
2. Update worktree scripts and commands
3. Update all verification references in commands

---

## Phase 1: Create Infrastructure

### Overview
Add Makefile, pre-commit hooks, and update .gitignore.

### Changes Required:

#### 1. Create Makefile
**File**: `Makefile` (new file)

```makefile
# Makefile for YouLab Python Project
# Uses uv for dependency management and task running

.PHONY: help setup check lint lint-fix typecheck test verify clean

# Default target - show help
help:
	@echo "YouLab Development Commands:"
	@echo ""
	@echo "  make setup       - Install dependencies (dev + observability)"
	@echo "  make check       - Run all checks (lint + typecheck)"
	@echo "  make lint        - Run ruff check and format check"
	@echo "  make lint-fix    - Run ruff with --fix and apply formatting"
	@echo "  make typecheck   - Run mypy type checking"
	@echo "  make test        - Run pytest test suite"
	@echo "  make verify      - Run full verification (check + test)"
	@echo "  make clean       - Clean up cache directories"
	@echo ""

setup:
	@echo "Installing dependencies..."
	uv sync --all-extras
	@echo "Setup complete!"

check: lint typecheck
	@echo "All checks passed!"

lint:
	@echo "Running ruff linter..."
	uv run ruff check src/ tests/
	@echo "Checking code formatting..."
	uv run ruff format --check src/ tests/

lint-fix:
	@echo "Running ruff with auto-fix..."
	uv run ruff check --fix src/ tests/
	@echo "Applying code formatting..."
	uv run ruff format src/ tests/

typecheck:
	@echo "Running mypy..."
	uv run mypy src/

test:
	@echo "Running pytest..."
	uv run pytest

verify: check test
	@echo "Full verification complete!"

clean:
	rm -rf .pytest_cache .mypy_cache .ruff_cache __pycache__
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
	find . -type f -name "*.pyc" -delete
```

#### 2. Create Pre-commit Config
**File**: `.pre-commit-config.yaml` (new file)

```yaml
# Pre-commit hooks for YouLab
# Install: pip install pre-commit && pre-commit install

repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: ruff check
        entry: uv run ruff check
        language: system
        types: [python]
        args: [--fix]

      - id: ruff-format
        name: ruff format
        entry: uv run ruff format
        language: system
        types: [python]

      - id: mypy
        name: mypy
        entry: uv run mypy
        language: system
        types: [python]
        args: [src/]
        pass_filenames: false
```

#### 3. Update .gitignore
**File**: `.gitignore`
**Changes**: Add after line 74 (after `thoughts/`)

```gitignore
# Git worktrees (in-repo)
.trees/

# Ruff cache
.ruff_cache/
```

#### 4. Update CLAUDE.md Commands Section
**File**: `CLAUDE.md`
**Changes**: Replace lines 31-48 with:

```markdown
## Commands

```bash
# Setup
uv sync --all-extras             # Install deps with dev tools
make setup                       # Alternative: install via Makefile

# Run
uv run letta-starter             # Interactive CLI (requires letta server)
uv run letta-starter --agent X   # Use specific agent name

# Verification (run before every commit!)
make verify                      # Full verification: lint + typecheck + test
make check                       # Just lint + typecheck (no tests)
make lint-fix                    # Auto-fix linting issues

# Individual tools
make lint                        # Ruff check + format check
make typecheck                   # Mypy type checking
make test                        # Pytest test suite
```

Requires Letta server running: `pip install letta && letta server`
```

### Success Criteria:

#### Automated Verification:
- [ ] `make setup` completes successfully
- [ ] `make verify` runs lint + typecheck + test
- [ ] `make lint` catches violations (test with intentional error)
- [ ] Pre-commit hook installs: `pip install pre-commit && pre-commit install`
- [ ] Pre-commit blocks bad commits (test with lint violation)

#### Manual Verification:
- [ ] `make help` shows all available commands
- [ ] CLAUDE.md commands section is clear and accurate

---

## Phase 2: Update Worktree Scripts

### Overview
Move worktrees from `~/wt/${REPO}/` to `.trees/` and fix inconsistencies.

### Changes Required:

#### 1. Update create_worktree.sh
**File**: `hack/create_worktree.sh`

**Change lines 44-54** (worktree base directory):
```bash
# Get repository root
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_BASE_NAME=$(basename "$REPO_ROOT")

if [ ! -z "$HUMANLAYER_WORKTREE_OVERRIDE_BASE" ]; then
    WORKTREE_DIR_NAME="${WORKTREE_NAME}"
    WORKTREES_BASE="${HUMANLAYER_WORKTREE_OVERRIDE_BASE}/${REPO_BASE_NAME}"
else
    WORKTREE_DIR_NAME="${WORKTREE_NAME}"
    WORKTREES_BASE="${REPO_ROOT}/.trees"
fi

WORKTREE_PATH="${WORKTREES_BASE}/${WORKTREE_DIR_NAME}"
```

**Change lines 60-64** (auto-create directory):
```bash
# Create worktrees directory if it doesn't exist
if [ ! -d "$WORKTREES_BASE" ]; then
    echo "Creating worktrees directory: $WORKTREES_BASE"
    mkdir -p "$WORKTREES_BASE"
fi
```

**Change line 123** (thoughts init directory - verify this is correct):
```bash
# Change from "humanlayer" to match repo name or shared directory
if humanlayer thoughts init --directory YouLab > /dev/null 2>&1; then
```

#### 2. Update cleanup_worktree.sh
**File**: `hack/cleanup_worktree.sh`

**Change lines 11-12** (worktree base directory):
```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
REPO_BASE_NAME=$(basename "$REPO_ROOT")
WORKTREE_BASE_DIR="${REPO_ROOT}/.trees"
```

**Change line 32** (path format to match create script):
```bash
local worktree_path="$WORKTREE_BASE_DIR/${worktree_name}"
```

#### 3. Update create_worktree.md Command
**File**: `.claude/commands/create_worktree.md`

**Change line 27** (example path):
```markdown
worktree path: $(git rev-parse --show-toplevel)/.trees/ENG-XXXX
```

**Change lines 36 and 41** (launch commands):
```markdown
humanlayer launch --model opus -w $(git rev-parse --show-toplevel)/.trees/ENG-XXXX "/implement_plan..."
```

#### 4. Update local_review.md Command
**File**: `.claude/commands/local_review.md`

**Change line 26**:
```markdown
- Create worktree: `git worktree add -b BRANCHNAME $(git rev-parse --show-toplevel)/.trees/SHORT_NAME USERNAME/BRANCHNAME`
```

**Change line 47**:
```markdown
- Create worktree at `$(git rev-parse --show-toplevel)/.trees/eng-1696`
```

#### 5. Update ralph_impl.md Command
**File**: `.claude/commands/ralph_impl.md`

**Change line 31** (launch command):
```markdown
-w $(git rev-parse --show-toplevel)/.trees/ENG-XXXX
```

### Success Criteria:

#### Automated Verification:
- [ ] `./hack/create_worktree.sh test-wt` creates worktree at `.trees/test-wt`
- [ ] Worktree has thoughts initialized
- [ ] `./hack/cleanup_worktree.sh test-wt` removes the worktree
- [ ] `.trees/` directory is gitignored (not tracked)

#### Manual Verification:
- [ ] Worktree can run `make verify` successfully
- [ ] Claude Code can launch in worktree and access thoughts

---

## Phase 3: Update Verification References

### Overview
Update all command files to use `make verify` instead of non-existent make targets.

### Changes Required:

#### 1. Update commit.md
**File**: `.claude/commands/commit.md`

**Add after line 15** (in "Plan your commit(s)" section):
```markdown
3. **Run verification before committing:**
   - Run `make verify` to ensure lint + typecheck + tests pass
   - If verification fails, fix issues before proceeding
   - Do NOT commit code that fails verification
```

#### 2. Update ci_commit.md
**File**: `.claude/commands/ci_commit.md`

**Add after line 15**:
```markdown
3. **Run verification:**
   - Run `make verify` to ensure all checks pass
   - Fix any issues before committing
```

#### 3. Update implement_plan.md
**File**: `.claude/commands/implement_plan.md`

**Change line 46** (verification approach):
```markdown
- Run the success criteria checks (`make verify` covers lint + typecheck + test)
```

#### 4. Update validate_plan.md
**File**: `.claude/commands/validate_plan.md`

**Change lines 27-28**:
```bash
# Run comprehensive checks
make verify
```

**Change lines 94-97**:
```markdown
### Automated Verification Results
✓ Checks pass: `make check`
✓ Tests pass: `make test`
✓ Full verification: `make verify`
```

#### 5. Update describe_pr.md
**File**: `.claude/commands/describe_pr.md`

**Change line 44**:
```markdown
- If it's a command you can run (like `make verify`, `make test`, etc.), run it
```

#### 6. Update create_plan.md (and variants)
**File**: `.claude/commands/create_plan.md`

**Change lines 228-232** (success criteria template):
```markdown
#### Automated Verification:
- [ ] Lint passes: `make lint`
- [ ] Type checking passes: `make typecheck`
- [ ] Tests pass: `make test`
- [ ] Full verification: `make verify`
```

**Apply same changes to**:
- `.claude/commands/create_plan_nt.md` (lines 224-227)
- `.claude/commands/create_plan_generic.md` (lines 228-231)
- `.claude/commands/cl/create_plan.md` (lines 225-228)

#### 7. Update iterate_plan.md (and variants)
**File**: `.claude/commands/iterate_plan.md`

**Update any `make check test` references to `make verify`**

**Apply same changes to**:
- `.claude/commands/iterate_plan_nt.md`
- `.claude/commands/cl/iterate_plan.md`

#### 8. Update ci_describe_pr.md and describe_pr_nt.md
**Files**:
- `.claude/commands/ci_describe_pr.md`
- `.claude/commands/describe_pr_nt.md`
- `.claude/commands/cl/describe_pr.md`

**Update verification references to use `make verify`**

### Success Criteria:

#### Automated Verification:
- [ ] `grep -r "make check test" .claude/commands/` returns no results
- [ ] `grep -r "make verify" .claude/commands/` returns expected results
- [ ] All command files reference existing Makefile targets

#### Manual Verification:
- [ ] Running `/commit` command mentions verification requirement
- [ ] Running `/implement_plan` uses `make verify` for verification

---

## Testing Strategy

### Unit Tests:
- Existing tests in `tests/` continue to pass
- `make test` runs all tests

### Integration Tests:
- Create test worktree, verify thoughts work
- Run full `make verify` pipeline
- Test pre-commit hook blocks bad commits

### Manual Testing Steps:
1. Create a worktree: `./hack/create_worktree.sh test-workflow`
2. cd into worktree, run `make verify`
3. Make intentional lint error, verify `git commit` is blocked
4. Clean up: `./hack/cleanup_worktree.sh test-workflow`

## Migration Notes

### Pre-commit Installation
After Phase 1, all developers need to run:
```bash
pip install pre-commit
pre-commit install
```

Add this to README.md development section.

### Existing Worktrees
Existing worktrees at `~/wt/` will still work but new ones will be created at `.trees/`. Consider cleaning up old worktrees after migration.

## References

- Research: AI-friendly codebase best practices (this session)
- WriteClaudeMD.md: `thoughts/global/shared/reference/WriteClaudeMD.md`
- Existing worktree scripts: `hack/create_worktree.sh`, `hack/cleanup_worktree.sh`
