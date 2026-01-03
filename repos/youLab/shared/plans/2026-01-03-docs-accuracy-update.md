# Documentation Accuracy Update Plan

## Overview

Update all documentation files to be 100% accurate based on the verification report from 2026-01-03. This involves fixing minor discrepancies across 5 docs files and adding missing structural coverage to CLAUDE.md.

## Current State Analysis

Based on verification report `thoughts/shared/research/2026-01-03-docs-verification.md`:
- 5 docs are mostly accurate with minor discrepancies
- 1 doc (CLAUDE.md) has a structural omission (missing `server/strategy/` directory)

### Key Discoveries:
- `HealthResponse` includes `version: str = "0.1.0"` not documented anywhere (`schemas.py:70`)
- `AgentResponse.created_at` accepts `datetime | int | None` not just `datetime | None` (`schemas.py:28`)
- POST /agents can return 500 error (`main.py:104-107`)
- All strategy endpoints return 503 on Letta unavailable
- Pipeline lifecycle hooks have conditional logging (`letta_pipe.py:45-55`)

## Desired End State

All documentation files accurately reflect the current implementation:
- Every schema field matches its Pydantic definition
- Every endpoint documents all possible error codes
- CLAUDE.md lists all directories and key files
- All code examples match actual implementation

### Verification:
- Re-run `/verify_docs` - should report 0 discrepancies

## What We're NOT Doing

- Adding new documentation sections
- Changing documentation structure/format
- Documenting internal implementation details (already decided to skip)
- Adding Langfuse tracing details (low priority per report)

## Implementation Approach

Make minimal, surgical edits to fix each documented discrepancy. Changes are independent and can be verified individually.

---

## Phase 1: Fix Schema Documentation

### Overview
Update `docs/Schemas.md` to match actual Pydantic definitions.

### Changes Required:

#### 1. Fix HealthResponse (line 82-86)

**File**: `docs/Schemas.md`

**Current** (line 82-86):
```python
class HealthResponse(BaseModel):
    status: str
    letta_connected: bool
```

**Updated**:
```python
class HealthResponse(BaseModel):
    status: str
    letta_connected: bool
    version: str = "0.1.0"
```

#### 2. Fix AgentResponse.created_at (line 28-34)

**File**: `docs/Schemas.md`

**Current** (line 33):
```python
    created_at: datetime | None = None
```

**Updated**:
```python
    created_at: datetime | int | None = None
```

### Success Criteria:

#### Automated Verification:
- [ ] `make lint-fix` passes

#### Manual Verification:
- [ ] Schema definitions match `src/letta_starter/server/schemas.py`

---

## Phase 2: Fix API Reference

### Overview
Update `docs/API.md` to document all error codes and response fields.

### Changes Required:

#### 1. Add version to GET /health response (line 26-30)

**File**: `docs/API.md`

**Current** (line 26-30):
```json
{
  "status": "ok",
  "letta_connected": true
}
```

**Updated**:
```json
{
  "status": "ok",
  "letta_connected": true,
  "version": "0.1.0"
}
```

#### 2. Add version to health table (line 33-36)

**File**: `docs/API.md`

**Current** (line 33-36):
```markdown
| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` or `"degraded"` |
| `letta_connected` | boolean | Letta server reachable |
```

**Updated**:
```markdown
| Field | Type | Description |
|-------|------|-------------|
| `status` | string | `"ok"` or `"degraded"` |
| `letta_connected` | boolean | Letta server reachable |
| `version` | string | Service version (e.g., `"0.1.0"`) |
```

#### 3. Add 500 error to POST /agents (line 72-74)

**File**: `docs/API.md`

**Current** (line 72-74):
```markdown
**Errors**:
- `400` - Unknown agent type
- `503` - Letta unavailable
```

**Updated**:
```markdown
**Errors**:
- `400` - Unknown agent type
- `500` - Agent created but info retrieval failed
- `503` - Letta unavailable
```

#### 4. Add 503 errors to strategy endpoints

**File**: `docs/API.md`

After POST /strategy/documents response (after line 227), add:
```markdown
**Errors**:
- `503` - Letta unavailable
```

After POST /strategy/ask response (after line 247), add:
```markdown
**Errors**:
- `503` - Letta unavailable
```

After GET /strategy/documents response (after line 268), add:
```markdown
**Errors**:
- `503` - Letta unavailable
```

### Success Criteria:

#### Automated Verification:
- [ ] `make lint-fix` passes

#### Manual Verification:
- [ ] All documented error codes match implementation

---

## Phase 3: Fix HTTP Service Documentation

### Overview
Update `docs/HTTP-Service.md` to add version field to health response.

### Changes Required:

#### 1. Add version to health response (line 39-44)

**File**: `docs/HTTP-Service.md`

**Current** (line 39-44):
```json
{
  "status": "ok",
  "letta_connected": true
}
```

**Updated**:
```json
{
  "status": "ok",
  "letta_connected": true,
  "version": "0.1.0"
}
```

### Success Criteria:

#### Automated Verification:
- [ ] `make lint-fix` passes

#### Manual Verification:
- [ ] Health response matches implementation

---

## Phase 4: Fix Pipeline Documentation

### Overview
Update `docs/Pipeline.md` to show conditional logging and error defaults.

### Changes Required:

#### 1. Fix lifecycle hooks (line 211-220)

**File**: `docs/Pipeline.md`

**Current** (line 211-220):
```python
async def on_startup(self):
    print(f"YouLab Pipe started. Service: {LETTA_SERVICE_URL}")

async def on_shutdown(self):
    print("YouLab Pipe stopped")

async def on_valves_updated(self):
    print("YouLab Pipe valves updated")
```

**Updated**:
```python
async def on_startup(self):
    if self.valves.ENABLE_LOGGING:
        print(f"YouLab Pipe started. Service: {self.valves.LETTA_SERVICE_URL}")

async def on_shutdown(self):
    if self.valves.ENABLE_LOGGING:
        print("YouLab Pipe stopped")

async def on_valves_updated(self):
    if self.valves.ENABLE_LOGGING:
        print("YouLab Pipe valves updated")
```

#### 2. Fix error event default (line 159-163)

**File**: `docs/Pipeline.md`

**Current** (line 159-163):
```python
    elif event_type == "error":
        await emitter({
            "type": "message",
            "data": {"content": f"Error: {event.get('message')}"}
        })
```

**Updated**:
```python
    elif event_type == "error":
        await emitter({
            "type": "message",
            "data": {"content": f"Error: {event.get('message', 'Unknown')}"}
        })
```

### Success Criteria:

#### Automated Verification:
- [ ] `make lint-fix` passes

#### Manual Verification:
- [ ] Code examples match `src/letta_starter/pipelines/letta_pipe.py`

---

## Phase 5: Update CLAUDE.md Structure

### Overview
Add missing `server/strategy/` directory and key files to CLAUDE.md.

### Changes Required:

#### 1. Add strategy directory to Project Structure (after line 22)

**File**: `CLAUDE.md`

**Current** (line 17-27):
```
src/letta_starter/       # Python backend
  agents/                # BaseAgent + factory functions + AgentRegistry + templates
  memory/                # Memory blocks, rotation strategies, manager
  pipelines/             # OpenWebUI Pipe integration
  server/                # FastAPI HTTP service (agent management, chat endpoints)
  observability/         # Logging, metrics, tracing (Langfuse)
  config/                # Pydantic settings from env
  main.py                # CLI entry point
tests/                   # Pytest suite (including tests/test_server/)
```

**Updated**:
```
src/letta_starter/       # Python backend
  agents/                # BaseAgent + factory functions + AgentRegistry + templates
  memory/                # Memory blocks, rotation strategies, manager
  pipelines/             # OpenWebUI Pipe integration
  server/                # FastAPI HTTP service (agent management, chat endpoints)
    strategy/            # RAG-enabled strategy agent (project knowledge queries)
  observability/         # Logging, metrics, tracing (Langfuse)
  config/                # Pydantic settings from env
  main.py                # CLI entry point
tests/                   # Pytest suite (including tests/test_server/)
```

#### 2. Add strategy key files (after line 72)

**File**: `CLAUDE.md`

After `- `src/letta_starter/server/agents.py` - AgentManager for per-user Letta agents` (line 72), add:
```markdown
- `src/letta_starter/server/strategy/manager.py` - StrategyManager singleton for RAG queries
- `src/letta_starter/server/strategy/router.py` - FastAPI router: /strategy/documents, /ask, /health
```

### Success Criteria:

#### Automated Verification:
- [ ] `make lint-fix` passes
- [ ] All listed files exist: `ls -la src/letta_starter/server/strategy/`

#### Manual Verification:
- [ ] Project structure matches actual directory layout

---

## Phase 6: Verify All Changes

### Overview
Run full verification to confirm all discrepancies resolved.

### Steps:
1. Run `make verify-agent` to ensure no regressions
2. Run `/verify_docs` to generate new verification report
3. Confirm 0 discrepancies in new report

### Success Criteria:

#### Automated Verification:
- [ ] `make verify-agent` passes
- [ ] `/verify_docs` shows "Documentation is in sync" or 0 discrepancies

#### Manual Verification:
- [ ] Review new verification report for any remaining issues

---

## Testing Strategy

### Automated:
- `make lint-fix` after each phase to catch markdown issues
- `make verify-agent` at end to ensure no code regressions

### Manual:
- Cross-reference each changed section against source code
- Spot-check file:line references are accurate

## References

- Verification report: `thoughts/shared/research/2026-01-03-docs-verification.md`
- Source schemas: `src/letta_starter/server/schemas.py`
- Source pipeline: `src/letta_starter/pipelines/letta_pipe.py`
- Strategy implementation: `src/letta_starter/server/strategy/`
