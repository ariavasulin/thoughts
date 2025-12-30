# Tooling Migration: BasedPyright, Ruff ALL, pytest-cov

## Overview

Replace MyPy with BasedPyright for type checking, enable Ruff's ALL rules for comprehensive linting, and add pytest-cov for test coverage reporting. This creates a strict, modern Python tooling setup.

## Current State Analysis

**MyPy** (`pyproject.toml:51-60`):
- Strict mode with `ignore_missing_imports` for `letta.*` and `langfuse.*`
- Run via `uv run mypy src/`

**Ruff** (`pyproject.toml:40-49`):
- 9 rule sets: `["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM"]`
- Line length 100, Python 3.11 target

**Pytest** (`pyproject.toml:62-64`):
- Minimal config with async mode
- No coverage configuration

**Codebase**: 11 Python source files with 100% type annotation coverage, modern Python 3.10+ syntax.

## Desired End State

After this plan is complete:
1. **BasedPyright** runs type checking with strict mode, ignoring external library stubs
2. **Ruff** runs with ALL rules enabled, selective ignores for inapplicable rules
3. **pytest-cov** reports coverage with branch analysis
4. All tools pass on current codebase
5. Pre-commit hooks use new tools
6. Makefile has updated targets

### Verification Commands:
```bash
make verify        # Should pass (lint + typecheck + tests)
make coverage      # Should show coverage report
git commit --dry-run  # Pre-commit hooks should work
```

## What We're NOT Doing

- Not adding stub packages for letta/langfuse (using ignore instead)
- Not enforcing a minimum coverage threshold yet (can add later)
- Not modifying source code type annotations (they're already complete)
- Not changing the test structure

## Implementation Approach

Update all configuration files in Phase 1, then fix any errors surfaced by the stricter tools in subsequent phases. This allows us to see the full scope of changes needed before making code modifications.

---

## Phase 1: Update Configuration Files

### Overview
Update all config files to use the new tools. This will initially cause failures that we fix in later phases.

### Changes Required:

#### 1. pyproject.toml - Dependencies
**File**: `pyproject.toml`
**Changes**: Replace mypy with basedpyright, add pytest-cov

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.4.0",
    "basedpyright>=1.22.0",
    "pre-commit>=3.7",
]
```

#### 2. pyproject.toml - Ruff Configuration
**File**: `pyproject.toml`
**Changes**: Replace current `[tool.ruff]` section with ALL rules config

```toml
[tool.ruff]
line-length = 100
target-version = "py311"
exclude = ["OpenWebUI"]

[tool.ruff.lint]
select = ["ALL"]
ignore = [
    # Formatting (handled by ruff format)
    "W191",   # tab-indentation
    "E111",   # indentation-with-invalid-multiple
    "E114",   # indentation-not-multiple-of-four-comment
    "E117",   # over-indented
    "E501",   # line-too-long (handled by formatter)
    "D206",   # indent-with-spaces
    "D300",   # triple-single-quotes
    "Q000",   # bad-quotes-inline-string
    "Q001",   # bad-quotes-multiline-string
    "Q002",   # bad-quotes-docstring
    "Q003",   # avoidable-escaped-quote
    "COM812", # missing-trailing-comma
    "COM819", # prohibited-trailing-comma
    "ISC001", # single-line-implicit-string-concatenation
    "ISC002", # multi-line-implicit-string-concatenation

    # Docstrings (too strict for this project)
    "D100",   # undocumented-public-module
    "D101",   # undocumented-public-class
    "D102",   # undocumented-public-method
    "D103",   # undocumented-public-function
    "D104",   # undocumented-public-package
    "D105",   # undocumented-magic-method
    "D107",   # undocumented-public-init
    "D203",   # one-blank-line-before-class (conflicts with D211)
    "D212",   # multi-line-summary-first-line (conflicts with D213)

    # Too opinionated
    "ANN101", # missing-type-self (deprecated)
    "ANN102", # missing-type-cls (deprecated)
    "ANN401", # dynamically-typed-expressions (Any is intentional for letta)
    "TD002",  # missing-todo-author
    "TD003",  # missing-todo-link
    "FIX002", # line-contains-todo
]

[tool.ruff.lint.isort]
known-first-party = ["letta_starter"]

[tool.ruff.lint.per-file-ignores]
"tests/*" = [
    "S101",   # assert (allowed in tests)
    "PLR2004", # magic-value-comparison (allowed in tests)
    "D",      # docstrings (not required in tests)
]
```

#### 3. pyproject.toml - BasedPyright Configuration
**File**: `pyproject.toml`
**Changes**: Remove `[tool.mypy]` section, add `[tool.basedpyright]`

Remove this section:
```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[[tool.mypy.overrides]]
module = ["letta.*", "langfuse.*"]
ignore_missing_imports = true
```

Add this section:
```toml
[tool.basedpyright]
pythonVersion = "3.11"
typeCheckingMode = "strict"
include = ["src/letta_starter", "tests"]
exclude = [".venv", "__pycache__", "OpenWebUI"]
reportMissingImports = false
reportMissingTypeStubs = false
```

#### 4. pyproject.toml - Pytest Coverage Configuration
**File**: `pyproject.toml`
**Changes**: Update `[tool.pytest.ini_options]` and add coverage sections

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = [
    "--cov=src/letta_starter",
    "--cov-branch",
    "--cov-report=term-missing",
]

[tool.coverage.run]
source = ["src/letta_starter"]
branch = true
omit = [
    "*/tests/*",
    "*/__pycache__/*",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
    "class .*\\bProtocol\\):",
    "@abstractmethod",
]
precision = 2
skip_empty = true
```

#### 5. Makefile
**File**: `Makefile`
**Changes**: Update typecheck target, add coverage targets, update clean

```makefile
# Makefile for YouLab
# All commands use uv for dependency management

.PHONY: help setup lint lint-fix typecheck test check verify coverage coverage-html clean

help:
	@echo "YouLab Development Commands"
	@echo ""
	@echo "  setup        Install dependencies and configure dev environment"
	@echo "  lint         Run ruff linter (check mode)"
	@echo "  lint-fix     Run ruff with auto-fix"
	@echo "  typecheck    Run basedpyright type checker"
	@echo "  test         Run pytest (with coverage)"
	@echo "  check        Run lint + typecheck (no tests)"
	@echo "  verify       Run lint + typecheck + tests (full verification)"
	@echo "  coverage     Run tests with coverage report"
	@echo "  coverage-html Generate HTML coverage report"
	@echo "  clean        Remove cache directories"

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
	uv run basedpyright src/

test:
	uv run pytest

check: lint typecheck

verify: check test
	@echo ""
	@echo "All checks passed!"

coverage:
	uv run pytest --cov=src/letta_starter --cov-report=term-missing

coverage-html:
	uv run pytest --cov=src/letta_starter --cov-report=html
	@echo "Coverage report generated in htmlcov/index.html"

clean:
	rm -rf .pytest_cache .ruff_cache htmlcov .coverage coverage.xml
	find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null || true
```

#### 6. .pre-commit-config.yaml
**File**: `.pre-commit-config.yaml`
**Changes**: Replace mypy hook with basedpyright

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

      - id: basedpyright
        name: basedpyright typecheck
        entry: uv run basedpyright src/
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

#### 7. .gitignore
**File**: `.gitignore`
**Changes**: Add coverage.xml, update type checking section

Add after line 45 (Testing section):
```gitignore
coverage.xml
```

Update Type checking section (lines 52-55) to:
```gitignore
# Type checking
.mypy_cache/
.dmypy.json
dmypy.json
.basedpyright/
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv sync --all-extras` installs new dependencies without errors
- [ ] Config files have correct syntax (no TOML parse errors)

#### Manual Verification:
- [ ] Review pyproject.toml changes look correct

**Implementation Note**: After completing this phase, run `uv sync --all-extras` to install new dependencies before proceeding to Phase 2.

---

## Phase 2: Fix BasedPyright Type Errors

### Overview
Run BasedPyright and fix any type errors. BasedPyright may catch issues MyPy missed.

### Steps:

1. Run `uv run basedpyright src/` and capture errors
2. Fix any type errors found
3. Common issues to expect:
   - Stricter `Any` handling
   - More precise Protocol checking
   - Potential issues with Pydantic models

### Success Criteria:

#### Automated Verification:
- [ ] `make typecheck` passes (exit code 0)
- [ ] No type errors reported

#### Manual Verification:
- [ ] Review any code changes made to fix type errors

**Implementation Note**: If no errors are found (MyPy was already strict), proceed to Phase 3.

---

## Phase 3: Fix Ruff ALL Lint Errors

### Overview
Run Ruff with ALL rules and fix or ignore any errors. This will likely surface the most changes.

### Steps:

1. Run `uv run ruff check src/ tests/` and capture errors
2. For each error category, decide:
   - **Fix**: Update code to comply
   - **Ignore globally**: Add to `ignore` list in pyproject.toml
   - **Ignore per-file**: Add to `per-file-ignores`
3. Run `uv run ruff check --fix src/ tests/` for auto-fixable issues

### Expected Issues (and recommended handling):

| Rule | Description | Recommendation |
|------|-------------|----------------|
| `ERA001` | Commented-out code | Fix (remove or uncomment) |
| `T201` | Print statements | Fix (use logging) or ignore if intentional |
| `PLR0913` | Too many arguments | Ignore (config pattern) |
| `S101` | Assert usage | Already ignored in tests |
| `TRY003` | Long exception messages | Consider ignoring |
| `EM101/EM102` | Exception string formatting | Consider ignoring |

### Success Criteria:

#### Automated Verification:
- [ ] `make lint` passes (exit code 0)
- [ ] `make lint-fix` makes no further changes

#### Manual Verification:
- [ ] Review any code changes made
- [ ] Verify ignore list is reasonable

**Implementation Note**: Document any rules added to the ignore list with comments explaining why.

---

## Phase 4: Final Verification

### Overview
Verify all tools work together and pre-commit hooks function correctly.

### Steps:

1. Run full verification suite
2. Test pre-commit hooks
3. Verify coverage reporting

### Success Criteria:

#### Automated Verification:
- [ ] `make verify` passes completely
- [ ] `make coverage` shows coverage report
- [ ] `make coverage-html` generates htmlcov/index.html
- [ ] Pre-commit dry run: `uv run pre-commit run --all-files`

#### Manual Verification:
- [ ] Open htmlcov/index.html and verify report looks correct
- [ ] Make a test commit to verify hooks work

---

## Testing Strategy

### Automated Tests:
- Existing test suite (`tests/test_memory.py`) should pass unchanged
- Coverage report should show current coverage percentage

### Manual Testing:
1. Run `make verify` - should pass
2. Run `make coverage-html` - should generate report
3. Make a small change and commit - pre-commit hooks should run

---

## Migration Notes

### Rollback Plan:
If issues arise, revert changes to:
- `pyproject.toml`
- `Makefile`
- `.pre-commit-config.yaml`
- `.gitignore`

Then run `uv sync --all-extras` to restore mypy.

### Breaking Changes:
- None expected for CI/CD (same `make verify` interface)
- Developers need to run `uv sync --all-extras` to get new dependencies

---

## References

- Research document: `thoughts/shared/research/2025-12-29-tooling-mypy-ruff-pytest-config.md`
- BasedPyright docs: https://docs.basedpyright.com/
- Ruff ALL rules: https://docs.astral.sh/ruff/rules/
- pytest-cov docs: https://pytest-cov.readthedocs.io/
