# Ruff Configuration Tightening Implementation Plan

## Overview

Fix code issues identified by disabled Ruff rules and re-enable those rules to prevent future regressions. This tightens the linting configuration by addressing 10 violations (2 mutable class defaults, 8 magic values) and removing unnecessary rule exceptions.

## Current State Analysis

The codebase uses `select = ["ALL"]` with 40+ ignored rules. Most ignores are justified (formatting handled by ruff format, docstrings too strict, CLI patterns). However, two rules were catching real issues:

- **RUF012** (mutable-class-default): 2 violations - can cause shared state bugs
- **PLR2004** (magic-value-comparison): 8 violations - reduces code readability

### Key Discoveries:
- `src/letta_starter/memory/strategies.py:158` - Mutable list default (mitigated by `__init__` but bad form)
- `src/letta_starter/observability/metrics.py:119` - Class constant missing `ClassVar`
- Magic numbers scattered in `strategies.py` (rotation thresholds) and `blocks.py` (parsing indices)

## Desired End State

After this plan is complete:
1. `uv run ruff check src/` passes with RUF012 and PLR2004 enabled
2. `make verify` passes completely
3. Two fewer rules in the ignore list
4. Code is more readable with named constants

### Verification:
```bash
make verify  # Full lint + typecheck + tests
```

## What We're NOT Doing

- Refactoring complex functions (C901/PLR0912/PLR0915) - acceptable for REPL/parsers
- Changing exception handling patterns (BLE001) - intentional for CLI error recovery
- Removing singleton patterns (PLW0603) - deliberate architectural choice
- Adding timezone info to datetime (DTZ005/006) - local timestamps are fine
- Reducing function arguments (PLR0913) - factory functions need params

---

## Phase 1: Fix Mutable Class Defaults (RUF012)

### Overview
Fix 2 instances where mutable defaults could cause shared state bugs.

### Changes Required:

#### 1. Fix `AdaptiveStrategy` class
**File**: `src/letta_starter/memory/strategies.py`
**Line**: 158
**Issue**: `recent_rotations: list[float] = []` at class level is dangerous

```python
# BEFORE (line 156-162):
class AdaptiveStrategy(RotationStrategy):
    """..."""

    base_threshold: float = 0.8
    recent_rotations: list[float] = []

    def __init__(self) -> None:
        self.recent_rotations = []

# AFTER:
class AdaptiveStrategy(RotationStrategy):
    """..."""

    base_threshold: float = 0.8

    def __init__(self) -> None:
        self.recent_rotations: list[float] = []
```

#### 2. Fix `MetricsCollector` class constant
**File**: `src/letta_starter/observability/metrics.py`
**Line**: 119
**Issue**: `MODEL_COSTS` is a class constant but missing `ClassVar` annotation

```python
# BEFORE (line 117-129):
class MetricsCollector:
    """..."""

    # Approximate cost per 1K tokens by model (as of late 2024)
    MODEL_COSTS: dict[str, dict[str, float]] = {
        "gpt-4": {"input": 0.03, "output": 0.06},
        ...
    }

# AFTER:
from typing import ClassVar

class MetricsCollector:
    """..."""

    # Approximate cost per 1K tokens by model (as of late 2024)
    MODEL_COSTS: ClassVar[dict[str, dict[str, float]]] = {
        "gpt-4": {"input": 0.03, "output": 0.06},
        ...
    }
```

Also add `ClassVar` import at the top of the file (line ~8):
```python
from typing import ClassVar
```

### Success Criteria:

#### Automated Verification:
- [ ] No RUF012 violations: `uv run ruff check src/ --select=RUF012`
- [ ] Type checking passes: `make typecheck`
- [ ] Tests pass: `make test`

---

## Phase 2: Extract Magic Values to Constants (PLR2004)

### Overview
Replace 8 magic numbers with named constants for better readability.

### Changes Required:

#### 1. Add constants to `strategies.py`
**File**: `src/letta_starter/memory/strategies.py`
**Location**: After imports, before class definitions

```python
# Adaptive rotation strategy constants
ROTATION_HISTORY_MIN_SIZE = 5  # Minimum rotations before adapting threshold
ROTATION_HISTORY_MAX_SIZE = 20  # Maximum rotation history to keep
ROTATION_INTERVAL_TOO_FAST = 0.5  # Rotating too often if avg < this
ROTATION_INTERVAL_TOO_SLOW = 0.9  # Rarely rotating if avg > this
```

**Update usages** (lines 174-188):

```python
# BEFORE:
if len(self.recent_rotations) >= 5:
    avg_interval = sum(self.recent_rotations[-5:]) / 5
    if avg_interval < 0.5:  # Rotating too often
        threshold = min(0.95, threshold + 0.05)
    elif avg_interval > 0.9:  # Rarely rotating
        threshold = max(0.6, threshold - 0.05)
...
if len(self.recent_rotations) > 20:
    self.recent_rotations = self.recent_rotations[-20:]

# AFTER:
if len(self.recent_rotations) >= ROTATION_HISTORY_MIN_SIZE:
    avg_interval = (
        sum(self.recent_rotations[-ROTATION_HISTORY_MIN_SIZE:])
        / ROTATION_HISTORY_MIN_SIZE
    )
    if avg_interval < ROTATION_INTERVAL_TOO_FAST:
        threshold = min(0.95, threshold + 0.05)
    elif avg_interval > ROTATION_INTERVAL_TOO_SLOW:
        threshold = max(0.6, threshold - 0.05)
...
if len(self.recent_rotations) > ROTATION_HISTORY_MAX_SIZE:
    self.recent_rotations = self.recent_rotations[-ROTATION_HISTORY_MAX_SIZE:]
```

#### 2. Add constants to `blocks.py`
**File**: `src/letta_starter/memory/blocks.py`
**Location**: After imports, before class definitions

```python
# Memory string parsing constants
PERSONA_PARTS_WITH_ROLE = 2  # Expected parts when role is included (name, role)
HUMAN_PARTS_WITH_ROLE = 2  # Expected parts when role is included (name, role)
```

**Update usages** in `PersonaBlock.from_memory_string` (lines 108-123):

```python
# BEFORE:
if len(parts) >= 2:
    data["role"] = parts[1].strip()
...
if len(style) >= 2:
    data["verbosity"] = style[1].strip()

# AFTER:
if len(parts) >= PERSONA_PARTS_WITH_ROLE:
    data["role"] = parts[1].strip()
...
if len(style) >= PERSONA_PARTS_WITH_ROLE:
    data["verbosity"] = style[1].strip()
```

**Update usages** in `HumanBlock.from_memory_string` (line 253):

```python
# BEFORE:
if len(parts) >= 2:
    data["role"] = parts[1].strip()

# AFTER:
if len(parts) >= HUMAN_PARTS_WITH_ROLE:
    data["role"] = parts[1].strip()
```

### Success Criteria:

#### Automated Verification:
- [ ] No PLR2004 violations: `uv run ruff check src/ --select=PLR2004`
- [ ] Lint passes: `make lint`
- [ ] Tests pass: `make test`

---

## Phase 3: Update Ruff Configuration

### Overview
Remove RUF012 and PLR2004 from the ignore list to prevent future regressions.

### Changes Required:

#### 1. Update `pyproject.toml`
**File**: `pyproject.toml`
**Section**: `[tool.ruff.lint]`

Remove these two lines from the `ignore` list:
```toml
# REMOVE these lines:
    "RUF012",  # mutable-class-default (ClassVar annotation too strict)
    "PLR2004", # magic-value-comparison (clear in context)
```

Also update/remove their comments since they're no longer ignored.

### Success Criteria:

#### Automated Verification:
- [ ] Full lint passes: `make lint`
- [ ] Full verification passes: `make verify`

---

## Phase 4: Final Verification

### Overview
Run complete verification suite to ensure no regressions.

### Commands to Run:

```bash
# Individual checks
uv run ruff check src/                    # Linting
uv run ruff format --check src/           # Formatting
make typecheck                            # BasedPyright
make test                                 # Pytest

# Full verification
make verify
```

### Success Criteria:

#### Automated Verification:
- [ ] `make verify` passes completely
- [ ] No new warnings or errors introduced

#### Manual Verification:
- [ ] Review the pyproject.toml changes are clean
- [ ] Confirm the ignore list is reduced by 2 entries

---

## Testing Strategy

### Unit Tests:
- Existing tests cover the affected code paths
- No new tests needed - this is a refactoring with no behavior change

### Regression Check:
- `make verify` covers lint + typecheck + tests
- All existing tests must continue to pass

---

## Summary of Changes

| File | Change Type | Description |
|------|-------------|-------------|
| `src/letta_starter/memory/strategies.py` | Fix + Constants | Remove mutable default, add rotation constants |
| `src/letta_starter/observability/metrics.py` | Fix | Add ClassVar annotation, import typing.ClassVar |
| `src/letta_starter/memory/blocks.py` | Constants | Add parsing constants |
| `pyproject.toml` | Config | Remove RUF012 and PLR2004 from ignore list |

## References

- Original analysis: This conversation's Ruff configuration research
- Ruff documentation: https://docs.astral.sh/ruff/rules/
