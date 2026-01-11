# Rename letta-starter to youlab-server Implementation Plan

## Overview

Rename the project from `letta-starter` to `youlab-server` across the entire codebase, including the source directory, package name, CLI commands, imports, documentation, and all references.

**Scope**: 81 files total
- 46 Python source files
- 15+ test files
- 20 documentation files
- Config files (pyproject.toml, Makefile, .env.example)

## Current State Analysis

**Current naming:**
- Package name: `letta-starter`
- Python module: `letta_starter`
- Source directory: `src/letta_starter/`
- CLI commands: `letta-starter`, `letta-server`
- Default service name: `letta-starter`

**New naming:**
- Package name: `youlab-server`
- Python module: `youlab_server`
- Source directory: `src/youlab_server/`
- CLI commands: `youlab`, `youlab-server`
- Default service name: `youlab`

## Desired End State

After implementation:
- `src/youlab_server/` directory exists with all modules
- All imports use `youlab_server`
- CLI commands `youlab` and `youlab-server` work
- All documentation references updated
- All tests pass with new imports
- No references to `letta-starter`, `letta_starter`, or `LettaStarter` remain (except historical notes in Changelog)

### Verification:
```bash
# No old references should exist (except Changelog historical notes)
grep -r "letta.starter" src/ tests/ --include="*.py"
grep -r "letta_starter" src/ tests/ --include="*.py"
grep -r "LettaStarter" src/ tests/ --include="*.py"

# New CLI commands work
uv run youlab --help
uv run youlab-server --help

# All tests pass
make verify-agent
```

## What We're NOT Doing

- Not changing the repository name (YouLab stays)
- Not changing any functionality
- Not restructuring modules
- Not updating external deployment configs (if any exist outside repo)

## Implementation Approach

This is a mechanical rename operation. We'll use `sed` and `mv` for bulk changes, then manually verify and fix any edge cases. The order is important to avoid broken imports during the process.

---

## Phase 1: Rename Source Directory

### Overview
Rename the main source directory from `src/letta_starter/` to `src/youlab_server/`.

### Changes Required:

```bash
mv src/letta_starter src/youlab_server
```

### Success Criteria:

#### Automated Verification:
- [ ] Directory exists: `ls src/youlab_server/`
- [ ] Old directory gone: `! ls src/letta_starter 2>/dev/null`

---

## Phase 2: Update pyproject.toml

### Overview
Update package configuration with new names, paths, and CLI entry points.

### Changes Required:

**File**: `pyproject.toml`

| Line | Old | New |
|------|-----|-----|
| 2 | `name = "letta-starter"` | `name = "youlab-server"` |
| 37 | `letta-starter = "letta_starter.main:main"` | `youlab = "youlab_server.main:main"` |
| 38 | `letta-server = "letta_starter.server.cli:main"` | `youlab-server = "youlab_server.server.cli:main"` |
| 45 | `packages = ["src/letta_starter"]` | `packages = ["src/youlab_server"]` |
| 122 | `known-first-party = ["letta_starter"]` | `known-first-party = ["youlab_server"]` |
| 136 | `include = ["src/letta_starter", "tests"]` | `include = ["src/youlab_server", "tests"]` |
| 152 | `"--cov=src/letta_starter",` | `"--cov=src/youlab_server",` |
| 158 | `source = ["src/letta_starter"]` | `source = ["src/youlab_server"]` |

### Success Criteria:

#### Automated Verification:
- [ ] No `letta_starter` references: `! grep -q "letta_starter" pyproject.toml`
- [ ] No `letta-starter` references: `! grep -q "letta-starter" pyproject.toml`
- [ ] Package installs: `uv sync`

---

## Phase 3: Update All Python Imports

### Overview
Replace all `letta_starter` imports with `youlab_server` across all Python files.

### Changes Required:

**Bulk replacement** in all `.py` files under `src/` and `tests/`:

```bash
# Replace imports (underscore version)
find src tests -name "*.py" -exec sed -i '' 's/letta_starter/youlab_server/g' {} \;

# Replace LettaStarter in comments/docstrings
find src tests -name "*.py" -exec sed -i '' 's/LettaStarter/YouLab Server/g' {} \;
```

**Source files affected (46 files):**

```
src/youlab_server/__init__.py
src/youlab_server/main.py
src/youlab_server/agents/__init__.py
src/youlab_server/agents/base.py
src/youlab_server/agents/default.py
src/youlab_server/agents/templates.py
src/youlab_server/background/__init__.py
src/youlab_server/background/runner.py
src/youlab_server/config/__init__.py
src/youlab_server/config/settings.py
src/youlab_server/curriculum/__init__.py
src/youlab_server/curriculum/blocks.py
src/youlab_server/curriculum/loader.py
src/youlab_server/curriculum/schema.py
src/youlab_server/honcho/__init__.py
src/youlab_server/honcho/client.py
src/youlab_server/memory/__init__.py
src/youlab_server/memory/blocks.py
src/youlab_server/memory/enricher.py
src/youlab_server/memory/manager.py
src/youlab_server/memory/strategies.py
src/youlab_server/observability/__init__.py
src/youlab_server/observability/logging.py
src/youlab_server/observability/metrics.py
src/youlab_server/observability/tracing.py
src/youlab_server/pipelines/__init__.py
src/youlab_server/pipelines/letta_pipe.py
src/youlab_server/server/__init__.py
src/youlab_server/server/agents.py
src/youlab_server/server/background.py
src/youlab_server/server/cli.py
src/youlab_server/server/curriculum.py
src/youlab_server/server/main.py
src/youlab_server/server/schemas.py
src/youlab_server/server/tracing.py
src/youlab_server/server/strategy/__init__.py
src/youlab_server/server/strategy/manager.py
src/youlab_server/server/strategy/router.py
src/youlab_server/server/strategy/schemas.py
src/youlab_server/server/sync/__init__.py
src/youlab_server/server/sync/mappings.py
src/youlab_server/server/sync/openwebui_client.py
src/youlab_server/server/sync/router.py
src/youlab_server/server/sync/service.py
src/youlab_server/tools/__init__.py
src/youlab_server/tools/curriculum.py
src/youlab_server/tools/dialectic.py
src/youlab_server/tools/memory.py
```

**Test files affected (15+ files):**

```
tests/__init__.py
tests/conftest.py
tests/test_background_runner.py
tests/test_background_schema.py
tests/test_curriculum_loader.py
tests/test_honcho.py
tests/test_memory.py
tests/test_memory_enricher.py
tests/test_pipe.py
tests/test_server_honcho.py
tests/test_templates.py
tests/test_server/conftest.py
tests/test_server/test_agents.py
tests/test_server/test_background.py
tests/test_server/test_schemas.py
tests/test_server/test_tracing.py
tests/test_server/test_strategy/conftest.py
tests/test_server/test_strategy/test_manager.py
```

### Success Criteria:

#### Automated Verification:
- [ ] No old imports: `! grep -r "from letta_starter" src/ tests/`
- [ ] No old imports: `! grep -r "import letta_starter" src/ tests/`
- [ ] Lint passes: `make lint-fix`
- [ ] Type check passes: `uv run basedpyright`

---

## Phase 4: Update Default Service Names and Hardcoded Strings

### Overview
Update hardcoded default service names from `letta-starter` to `youlab` and `LettaStarter` to `YouLab Server`.

### Changes Required:

**File**: `src/youlab_server/config/settings.py`
```python
# Line ~61
default="letta-starter"  →  default="youlab"
```

**File**: `src/youlab_server/observability/logging.py`
```python
# Line ~31
service_name: str = "letta-starter"  →  service_name: str = "youlab"
```

**File**: `src/youlab_server/observability/tracing.py`
```python
# Line ~56
service_name: str = "letta-starter"  →  service_name: str = "youlab"
# Line ~286
service_name: str = "letta-starter"  →  service_name: str = "youlab"
```

**File**: `src/youlab_server/server/main.py`
```python
# Line ~104
title="LettaStarter Service"  →  title="YouLab Server"
```

**File**: `src/youlab_server/server/cli.py`
```python
# Line ~12 (already handled by Phase 3, but verify)
"letta_starter.server.main:app"  →  "youlab_server.server.main:app"
```

**File**: `src/youlab_server/pipelines/letta_pipe.py`
```python
description="URL of the LettaStarter HTTP service"  →  description="URL of the YouLab Server HTTP service"
```

### Success Criteria:

#### Automated Verification:
- [ ] No old defaults: `! grep -r "letta-starter" src/`
- [ ] No old branding: `! grep -r "LettaStarter" src/`

---

## Phase 5: Update Environment Example

### Overview
Update `.env.example` with new naming.

### Changes Required:

**File**: `.env.example`
```bash
# Line 1
# LettaStarter Environment Configuration  →  # YouLab Server Environment Configuration

# Line ~41
SERVICE_NAME=letta-starter  →  SERVICE_NAME=youlab
```

### Success Criteria:

#### Automated Verification:
- [ ] No old references: `! grep -i "lettastarter\|letta-starter" .env.example`

---

## Phase 6: Update Makefile

### Overview
Update coverage paths in Makefile.

### Changes Required:

**File**: `Makefile`
```makefile
# Line ~60
--cov=src/letta_starter  →  --cov=src/youlab_server

# Line ~63
--cov=src/letta_starter  →  --cov=src/youlab_server
```

### Success Criteria:

#### Automated Verification:
- [ ] No old paths: `! grep "letta_starter" Makefile`

---

## Phase 7: Update Documentation (20 files)

### Overview
Update all 20 documentation files with new naming.

### Changes Required:

**Bulk sed commands for docs:**
```bash
# Replace all variations in docs/
find docs -name "*.md" -exec sed -i '' 's/letta_starter/youlab_server/g' {} \;
find docs -name "*.md" -exec sed -i '' 's/letta-starter/youlab/g' {} \;
find docs -name "*.md" -exec sed -i '' 's/LettaStarter/YouLab Server/g' {} \;
find docs -name "*.md" -exec sed -i '' 's/uv run letta-server/uv run youlab-server/g' {} \;
```

**Files to update (20 total):**

| File | Changes |
|------|---------|
| `docs/Architecture.md` | `letta_starter` → `youlab_server`, `LettaStarter` → `YouLab Server` |
| `docs/Agent-System.md` | `letta_starter` → `youlab_server` |
| `docs/Agent-Tools.md` | `letta_starter` → `youlab_server` |
| `docs/Background-Agents.md` | `letta_starter` → `youlab_server` |
| `docs/Changelog.md` | Add rename entry, update CLI commands in migration notes |
| `docs/Configuration.md` | `letta-starter` → `youlab` in defaults table |
| `docs/config-schema.md` | `letta_starter` → `youlab_server` |
| `docs/Development.md` | `letta_starter` → `youlab_server`, CLI commands |
| `docs/Honcho.md` | `letta_starter` → `youlab_server` |
| `docs/HTTP-Service.md` | `letta_starter` → `youlab_server`, `LettaStarter` → `YouLab Server` |
| `docs/Letta-SDK.md` | `letta_starter` → `youlab_server` |
| `docs/Memory-System.md` | `letta_starter` → `youlab_server` |
| `docs/Pipeline.md` | `letta_starter` → `youlab_server`, `LettaStarter` → `YouLab Server` |
| `docs/Quickstart.md` | `letta_starter` → `youlab_server`, CLI commands |
| `docs/Roadmap.md` | `LettaStarter` → `YouLab Server` |
| `docs/Schemas.md` | `letta_starter` → `youlab_server` |
| `docs/Settings.md` | `letta-starter` → `youlab` in defaults |
| `docs/Strategy-Agent.md` | `letta_starter` → `youlab_server` |
| `docs/Testing.md` | `letta_starter` → `youlab_server` |
| `docs/Tooling.md` | `letta_starter` → `youlab_server` |

**File**: `README.md`
```markdown
# LettaStarter  →  # YouLab Server
cd LettaStarter  →  cd YouLab
uv run letta-starter  →  uv run youlab
from letta_starter import  →  from youlab_server import
src/letta_starter/  →  src/youlab_server/
```

**File**: `CLAUDE.md`
```markdown
LettaStarter  →  YouLab Server
letta-starter  →  youlab
letta_starter  →  youlab_server
uv run letta-starter  →  uv run youlab
uv run letta-server  →  uv run youlab-server
src/letta_starter/  →  src/youlab_server/
```

**File**: `docs/Changelog.md` (special handling)

Add new entry at top of `[Unreleased]`:
```markdown
### Changed
- **BREAKING**: Renamed project from `letta-starter` to `youlab-server`
  - Package name: `letta-starter` → `youlab-server`
  - Python module: `letta_starter` → `youlab_server`
  - CLI commands: `letta-starter` → `youlab`, `letta-server` → `youlab-server`
  - Default service name: `letta-starter` → `youlab`
```

Update the Migration Notes section to use new CLI command:
```markdown
# Start the service
uv run letta-server  →  uv run youlab-server
```

Keep historical context but update active instructions.

### Success Criteria:

#### Automated Verification:
- [ ] No old module refs in docs: `! grep -r "letta_starter" docs/`
- [ ] No old CLI refs in docs: `! grep -r "letta-starter" docs/` (except Changelog history)
- [ ] No old branding in docs: `! grep -r "LettaStarter" docs/` (except Changelog history)
- [ ] README clean: `! grep -E "letta_starter|letta-starter|LettaStarter" README.md`
- [ ] CLAUDE.md clean: `! grep -E "letta_starter|letta-starter" CLAUDE.md`

---

## Phase 8: Final Verification

### Overview
Run full test suite and verify no stale references remain.

### Verification Commands:

```bash
# 1. Reinstall package with new name
uv sync

# 2. Run full verification
make verify-agent

# 3. Test CLI commands
uv run youlab --help
uv run youlab-server --help

# 4. Comprehensive grep for stale references
echo "Checking src/..."
grep -rn "letta_starter\|letta-starter\|LettaStarter" src/ && echo "FAIL: Found in src/" || echo "OK: src/ clean"

echo "Checking tests/..."
grep -rn "letta_starter\|letta-starter\|LettaStarter" tests/ && echo "FAIL: Found in tests/" || echo "OK: tests/ clean"

echo "Checking docs/ (excluding Changelog history)..."
grep -rn "letta_starter" docs/ && echo "FAIL: Found letta_starter in docs/" || echo "OK: docs/ clean"

echo "Checking config files..."
grep -n "letta_starter\|letta-starter" pyproject.toml Makefile .env.example && echo "FAIL: Found in config" || echo "OK: config clean"

echo "Checking README.md..."
grep -n "letta_starter\|letta-starter\|LettaStarter" README.md && echo "FAIL: Found in README" || echo "OK: README clean"

echo "Checking CLAUDE.md..."
grep -n "letta_starter\|letta-starter" CLAUDE.md && echo "FAIL: Found in CLAUDE.md" || echo "OK: CLAUDE.md clean"
```

### Success Criteria:

#### Automated Verification:
- [ ] Full verification passes: `make verify-agent`
- [ ] CLI works: `uv run youlab --help`
- [ ] Server CLI works: `uv run youlab-server --help`
- [ ] All grep checks pass (no stale references outside Changelog historical notes)

#### Manual Verification:
- [ ] Interactive CLI starts successfully: `uv run youlab`
- [ ] Server starts successfully: `uv run youlab-server`

---

## Testing Strategy

### Unit Tests:
- All existing tests should pass after import updates
- No new tests needed (rename only)

### Integration Tests:
- Verify CLI entry points work
- Verify server starts and responds

### Manual Testing Steps:
1. Run `uv run youlab --help` and verify help output shows "YouLab"
2. Run `uv run youlab-server --help` and verify help output
3. Start server with `uv run youlab-server` and hit health endpoint at `localhost:8100/health`

---

## Rollback

If issues arise:
```bash
git checkout -- .
```

---

## Summary of All Files to Modify

| Category | Count | Files |
|----------|-------|-------|
| Source directory | 1 | `src/letta_starter/` → `src/youlab_server/` |
| Python source | 46 | All `.py` files in `src/youlab_server/` |
| Python tests | 18 | All `.py` files in `tests/` |
| Package config | 1 | `pyproject.toml` |
| Build config | 1 | `Makefile` |
| Environment | 1 | `.env.example` |
| Documentation | 20 | All `.md` files in `docs/` |
| Root docs | 2 | `README.md`, `CLAUDE.md` |
| **Total** | **81** | |

---

## References

- Original naming established in initial project setup
- No external dependencies on the `letta-starter` package name (internal project)
