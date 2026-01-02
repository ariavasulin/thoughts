---
date: 2025-12-29T16:46:51-08:00
researcher: ariasulin
git_commit: e3efda49df8d2ed7da9e91b474a50dd9cd9d4cab
branch: main
repository: YouLab
topic: "Current MyPy, Ruff, and Pytest Configuration for Migration to BasedPyright"
tags: [research, codebase, mypy, basedpyright, ruff, pytest, coverage, tooling]
status: complete
last_updated: 2025-12-29
last_updated_by: ariasulin
---

# Research: Current MyPy, Ruff, and Pytest Configuration for Migration to BasedPyright

**Date**: 2025-12-29T16:46:51-08:00
**Researcher**: ariasulin
**Git Commit**: e3efda49df8d2ed7da9e91b474a50dd9cd9d4cab
**Branch**: main
**Repository**: YouLab

## Research Question

Document the current MyPy, Ruff, and Pytest configurations to understand what changes are needed to:
1. Replace MyPy with BasedPyright
2. Add strict Ruff configuration
3. Add pytest-cov for test coverage

## Summary

The codebase has a complete tooling setup in `pyproject.toml` with MyPy in strict mode, Ruff with a moderate rule set, and basic Pytest configuration. There is **no coverage configuration currently**. The Makefile and pre-commit hooks run all three tools. The source code uses modern Python 3.10+ type syntax throughout with complete type annotations on all functions, using `Any` as an escape hatch for untyped Letta library objects.

## Detailed Findings

### MyPy Configuration

**Location**: `pyproject.toml:51-60`

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

**Key aspects**:
- **Strict mode enabled** (`strict = true`) - enables all strict checking options
- Additional strict settings:
  - `warn_return_any = true` - warns when returning `Any` types
  - `warn_unused_configs = true` - warns about unused mypy configuration
  - `disallow_untyped_defs = true` - requires all functions to be typed
- **Import overrides** for external libraries without stubs:
  - `letta.*` - Letta agent framework
  - `langfuse.*` - Observability library

**Run via**: `uv run mypy src/` (Makefile target: `make typecheck`)

### Ruff Configuration

**Location**: `pyproject.toml:40-49`

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM"]
ignore = ["E501"]

[tool.ruff.lint.isort]
known-first-party = ["letta_starter"]
```

**Enabled rule sets** (9 categories):
| Code | Category | Description |
|------|----------|-------------|
| E | pycodestyle errors | Basic Python style errors |
| F | Pyflakes | Logic errors, undefined names |
| I | isort | Import sorting |
| N | pep8-naming | Naming conventions |
| W | pycodestyle warnings | Style warnings |
| UP | pyupgrade | Python version upgrade suggestions |
| B | flake8-bugbear | Bug-prone patterns |
| C4 | flake8-comprehensions | List/dict comprehension improvements |
| SIM | flake8-simplify | Code simplification |

**Ignored rules**:
- `E501` - Line too long (handled by `line-length = 100` but not enforced)

**NOT enabled** (potential additions for strict config):
- `S` - flake8-bandit (security)
- `A` - flake8-builtins (shadowing builtins)
- `PTH` - flake8-use-pathlib
- `RET` - flake8-return
- `ARG` - flake8-unused-arguments
- `PL` - Pylint rules
- `RUF` - Ruff-specific rules
- `D` - pydocstyle (documentation)
- `ANN` - flake8-annotations
- `T` - flake8-print

**Run via**:
- `uv run ruff check src/ tests/` (linting)
- `uv run ruff format --check src/ tests/` (format check)
- Makefile targets: `make lint`, `make lint-fix`

### Pytest Configuration

**Location**: `pyproject.toml:62-64`

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

**Current settings**:
- `asyncio_mode = "auto"` - Automatically handle async test functions
- `testpaths = ["tests"]` - Look for tests in `tests/` directory

**NOT configured**:
- No coverage settings (`pytest-cov` not in dependencies)
- No minimum coverage threshold
- No coverage report format
- No exclude patterns

**Run via**: `uv run pytest` (Makefile target: `make test`)

### Dependencies

**Location**: `pyproject.toml:16-23`

```toml
[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "ruff>=0.4.0",
    "mypy>=1.10.0",
    "pre-commit>=3.7",
]
```

**Current dev dependencies**:
- `pytest>=8.0.0`
- `pytest-asyncio>=0.23.0`
- `ruff>=0.4.0`
- `mypy>=1.10.0`
- `pre-commit>=3.7`

**Not present** (needed for migration):
- `basedpyright` or `pyright` - Will replace mypy
- `pytest-cov` - For coverage reporting

### Pre-commit Hooks

**Location**: `.pre-commit-config.yaml`

```yaml
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

**Hook order**: ruff-check → ruff-format → mypy → pytest

**Changes needed**:
- Replace `mypy` hook with `basedpyright` equivalent
- Entry will change from `uv run mypy src/` to `uv run basedpyright src/`

### Makefile Targets

**Location**: `Makefile`

| Target | Command | Description |
|--------|---------|-------------|
| `lint` | `ruff check` + `ruff format --check` | Check linting and formatting |
| `lint-fix` | `ruff check --fix` + `ruff format` | Auto-fix lint issues |
| `typecheck` | `mypy src/` | Type checking |
| `test` | `pytest` | Run tests |
| `check` | `lint` + `typecheck` | Quick verification |
| `verify` | `check` + `test` | Full verification |
| `clean` | Remove cache dirs | Clean `.pytest_cache`, `.mypy_cache`, `.ruff_cache` |

**Changes needed**:
- Update `typecheck` target to use `basedpyright`
- Update `clean` target (replace `.mypy_cache` cleanup)
- Potentially add `coverage` target

### Test Structure

**Location**: `tests/`

```
tests/
  __init__.py
  conftest.py      # Fixtures (sample_persona_data, sample_human_data)
  test_memory.py   # 21 test cases across 4 test classes
```

**Test classes in `test_memory.py`**:
- `TestPersonaBlock` - 4 tests
- `TestHumanBlock` - 9 tests
- `TestContextMetrics` - 1 test
- `TestContextStrategies` - 5 tests

**Total**: 19 test functions (counted from file)

### Type Annotation Patterns in Source

The codebase uses modern Python 3.10+ type syntax consistently:

**Union types**: `X | None` instead of `Optional[X]`
```python
tracer: Tracer | None = None
```

**Built-in generics**: `list[str]` instead of `List[str]`
```python
capabilities: list[str]
```

**External library handling**: `Any` for untyped libraries
```python
client: Any,  # Letta client instance
def _send_message_internal(self, message: str) -> Any:
```

**Protocol for structural typing**:
```python
class ContextStrategy(Protocol):
    def should_rotate(self, metrics: ContextMetrics) -> bool: ...
    def compress(self, content: str, target_chars: int) -> str: ...
```

**No type suppression comments**: Zero `# type: ignore` or `# noqa` comments in source code.

**Complete annotation coverage**: All 11 Python source files have complete type annotations on all functions, methods, class attributes, and module-level variables.

## Code References

- `pyproject.toml:40-64` - Ruff, MyPy, Pytest configuration
- `pyproject.toml:16-23` - Dev dependencies
- `.pre-commit-config.yaml:1-32` - Pre-commit hook definitions
- `Makefile:1-47` - Build targets
- `tests/conftest.py:1-27` - Test fixtures
- `tests/test_memory.py:1-210` - Test cases

## Architecture Documentation

### Current Tool Integration Flow

```
Developer writes code
        ↓
make lint-fix (auto-fix) OR git commit
        ↓
Pre-commit hooks trigger:
  1. ruff check --fix (lint + auto-fix)
  2. ruff format (format code)
  3. mypy src/ (type check)
  4. pytest (run tests)
        ↓
If all pass → commit succeeds
If any fail → commit blocked
```

### File Organization for Tooling

```
pyproject.toml          # All tool config (ruff, mypy, pytest)
.pre-commit-config.yaml # Hook definitions
Makefile                # Developer-facing commands
.env.example            # Environment variables
```

## Migration Impact Analysis

### Files Requiring Changes

| File | Change Type | Reason |
|------|-------------|--------|
| `pyproject.toml` | Modify | Replace `[tool.mypy]` with `[tool.basedpyright]`, add `pytest-cov`, enhance Ruff |
| `.pre-commit-config.yaml` | Modify | Replace mypy hook with basedpyright |
| `Makefile` | Modify | Update typecheck target, add coverage target |

### Configuration Translation: MyPy → BasedPyright

| MyPy Setting | BasedPyright Equivalent |
|--------------|------------------------|
| `strict = true` | `typeCheckingMode = "strict"` |
| `python_version = "3.11"` | `pythonVersion = "3.11"` |
| `warn_return_any = true` | Included in strict mode |
| `warn_unused_configs = true` | N/A (pyproject.toml based) |
| `disallow_untyped_defs = true` | Included in strict mode |
| `ignore_missing_imports` for modules | `reportMissingImports = false` or stub packages |

### Potential Type Errors with BasedPyright

BasedPyright may be stricter than MyPy in some areas:
1. **`Any` type usage** - May require more explicit casts or type narrowing
2. **Protocol implementation** - Stricter structural type checking
3. **Letta/Langfuse imports** - Will need `reportMissingImports` configuration

## Historical Context (from thoughts/)

No existing research documents found on this topic.

## Related Research

None found.

## Open Questions

1. **BasedPyright vs Pyright**: Should we use `basedpyright` (community fork with more features) or standard `pyright`?
2. **Stub packages**: Should we generate stub packages for `letta` and `langfuse` or continue ignoring missing imports?
3. **Coverage threshold**: What minimum coverage percentage should be enforced?
4. **Coverage scope**: Should coverage include `tests/` directory or just `src/`?
5. **Ruff strictness level**: Which additional rule sets should be enabled? Full list or selective?
