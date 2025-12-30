# Context-Efficient Test Outputs Implementation Plan

## Overview

Implement HumanLayer's "context-efficient backpressure" pattern for pytest outputs, optimizing test feedback for AI agent consumption while preserving essential diagnostic information on failures.

## Current State Analysis

### What Exists:
- `pyproject.toml:145-152` - pytest config with coverage enabled by default
- `hack/run_silent.sh:69-137` - Already has HumanLayer-style `run_silent_with_test_count()` for pytest (unused)
- `Makefile:38-39` - `make test` runs `uv run pytest` directly (verbose output)
- `CLAUDE.md:46` - Documents `make test` but no agent-specific guidance

### Problems:
1. **Coverage in default addopts** - Every test run outputs coverage tables (wastes ~500+ tokens)
2. **No fail-fast** - Agent must parse multiple failures instead of fixing one at a time
3. **Verbose success output** - "PASSED" for every test wastes context
4. **run_silent.sh exists but unused** - The solution is already written, just not wired up

## Desired End State

After implementation:

1. **`make test`** - Default for humans, includes coverage report
2. **`make test-agent`** - Optimized for AI agents:
   - Fail-fast (`-x`)
   - Minimal output on success: `  ✓ Tests (N tests, in X.Xs)`
   - Full failure output on failure (only relevant lines)
   - No coverage, no timing breakdown, no headers

### Verification:
```bash
# Success case should output only:
$ make test-agent
  ✓ Tests (42 tests, in 1.2s)

# Failure case should output:
$ make test-agent
  ✗ Tests
FAILED tests/test_memory.py::TestPersonaBlock::test_creation
    AssertionError: expected X got Y
```

## What We're NOT Doing

- NOT adding pytest-json-report (overkill for this project size)
- NOT changing pre-commit hooks (they should stay verbose for debugging)
- NOT removing coverage from `make test` (humans need it)
- NOT implementing MCP pytest server (future consideration)

## Implementation Approach

Use the existing `hack/run_silent.sh` infrastructure and wire it into the Makefile with agent-specific flags.

---

## Phase 1: Create Agent Test Wrapper

### Overview
Create a dedicated wrapper script that invokes pytest with agent-optimized flags and uses run_silent.sh for output formatting.

### Changes Required:

#### 1. Create `hack/test-agent.sh`
**File**: `hack/test-agent.sh` (new)
**Purpose**: Wrapper script for agent-optimized test execution

```bash
#!/bin/bash
# Agent-optimized test runner
# Implements HumanLayer's "swallow success, show failure" pattern
#
# Usage:
#   ./hack/test-agent.sh           # Run all tests
#   ./hack/test-agent.sh tests/    # Run specific path
#   VERBOSE=1 ./hack/test-agent.sh # Show full output

set -e

# Source the run_silent helpers
source "$(dirname "$0")/run_silent.sh"

# Agent-optimized pytest flags:
#   -x           Fail fast (stop on first failure)
#   --tb=short   Short traceback format
#   -q           Quiet mode (minimal output)
#   --no-header  No pytest header
#   --no-cov     Disable coverage (reduce noise)
PYTEST_AGENT_FLAGS="-x --tb=short -q --no-header --no-cov"

# Allow passing additional args (e.g., specific test path)
EXTRA_ARGS="${*:-}"

run_silent_with_test_count "Tests" "uv run pytest $PYTEST_AGENT_FLAGS $EXTRA_ARGS" "pytest"
```

### Success Criteria:

#### Automated Verification:
- [x] Script exists and is executable: `test -x hack/test-agent.sh`
- [x] Script runs without error: `./hack/test-agent.sh`
- [x] Success output is minimal (single line with checkmark)
- [ ] Lint passes: `make lint` (pre-existing lint issues in tests/)

#### Manual Verification:
- [ ] Introduce a failing test, verify failure output shows only relevant info
- [ ] Remove failing test, verify success output is single line

---

## Phase 2: Update Makefile

### Overview
Add `test-agent` target that uses the new wrapper script.

### Changes Required:

#### 1. Update Makefile
**File**: `Makefile`
**Changes**: Add test-agent target and update help

```makefile
# Add to .PHONY line:
.PHONY: help setup lint lint-fix typecheck test test-agent check verify coverage coverage-html clean

# Add after test target (around line 40):
test-agent:
	@./hack/test-agent.sh

# Update help section to include:
#   @echo "  test-agent   Run tests with agent-optimized output"
```

### Success Criteria:

#### Automated Verification:
- [x] `make test-agent` runs successfully
- [x] `make help` shows test-agent option
- [ ] Lint passes: `make lint` (pre-existing lint issues in tests/)

#### Manual Verification:
- [ ] Confirm `make test-agent` output is minimal on success
- [ ] Confirm `make test` still shows full coverage report

---

## Phase 3: Update pyproject.toml (Optional Optimization)

### Overview
Create separate pytest marker/config for agent runs. This is optional but allows `pytest --agent` syntax.

### Changes Required:

#### 1. Update pyproject.toml
**File**: `pyproject.toml`
**Changes**: Keep coverage in addopts but document the agent pattern

No changes needed to pyproject.toml - the wrapper script handles flags. However, if desired, you could add:

```toml
# Add to [tool.pytest.ini_options] section:
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
]
```

This is optional and only needed if you want to skip slow tests during agent iteration.

### Success Criteria:

#### Automated Verification:
- [ ] `make test` still works with coverage
- [ ] `make test-agent` works without coverage

---

## Phase 4: Update CLAUDE.md

### Overview
Document the agent-optimized testing workflow so Claude (and other agents) know to use it.

### Changes Required:

#### 1. Update CLAUDE.md Commands Section
**File**: `CLAUDE.md`
**Changes**: Add agent testing guidance

Replace the Commands section testing lines with:

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
make typecheck                   # BasedPyright
make test                        # Pytest (with coverage)
make test-agent                  # Pytest (agent-optimized, minimal output)
```

**Claude**: Run `make lint-fix` frequently during development and after every file edit to catch issues early.

**Claude**: Use `make test-agent` instead of `make test` for faster feedback with minimal context usage. It:
- Stops on first failure (`-x`)
- Shows only failure details, not passing tests
- Omits coverage report
- Uses single-line success output: `✓ Tests (N tests)`
```

### Success Criteria:

#### Automated Verification:
- [x] CLAUDE.md updated with new command
- [ ] Lint passes: `make lint` (pre-existing lint issues in tests/)

#### Manual Verification:
- [ ] Review CLAUDE.md reads clearly for agents

---

## Phase 5: Update Pre-commit (Optional)

### Overview
Consider whether pre-commit should use agent-optimized output. Current recommendation: **keep verbose** for pre-commit since developers need to see failures clearly.

### Decision:
**No changes to pre-commit.** The verbose output is appropriate for the commit workflow where humans are reviewing. Agent-optimized output is for interactive development loops.

---

## Testing Strategy

### Unit Tests:
- No new tests needed (this is infrastructure)

### Integration Tests:
- Run `make test-agent` with passing tests - verify minimal output
- Run `make test-agent` with a deliberately failing test - verify failure is shown

### Manual Testing Steps:
1. Run `make test-agent` - should show `✓ Tests (N tests, in X.Xs)`
2. Break a test intentionally
3. Run `make test-agent` - should show failure details only
4. Fix the test
5. Run `make test` - should show full coverage report
6. Run `make verify` - should work as before

## Performance Considerations

- Agent test runs will be ~20% faster (no coverage collection)
- Context usage reduced by ~500-1000 tokens per test run
- Failure debugging improved by fail-fast (one issue at a time)

## References

- HumanLayer Context-Efficient Backpressure: https://www.hlyr.dev/blog/context-efficient-backpressure
- HumanLayer run_silent.sh: https://github.com/humanlayer/humanlayer/blob/main/hack/run_silent.sh
- 12 Factor Agents (Factor 3 & 9): https://www.humanlayer.dev/blog/12-factor-agents
- Existing implementation: `hack/run_silent.sh:69-137`
