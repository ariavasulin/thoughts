# Module 1 Local Proof of Concept

## Overview

Get the first module of the YouLab college essay course running excellently locally. This is a proof of concept to validate the end-to-end experience before deployment.

## Current State

### What EXISTS and Works
| Component | Status | Notes |
|-----------|--------|-------|
| Module 1 TOML config | Complete | 3 steps: strengths-upload → deepdive → bridge |
| System prompt | Complete | Coaching philosophy, playbook, tool guidance |
| Memory blocks | Complete | student, engagement_strategy, journey |
| Tools | Complete | edit_memory_block, advance_lesson, query_honcho |
| FileSyncService | Complete | OpenWebUI → Letta sync with PDF→MD |
| Background grader | Defined | Not yet tested |

### What NEEDS Work
| Component | Status | Issue |
|-----------|--------|-------|
| Local infrastructure | Untested | No .env file configured |
| Folder attachment | NOT IMPLEMENTED | TOML config exists but not wired to agent creation |
| PDF analysis flow | Untested | Need to verify CliftonStrengths PDF works |
| End-to-end experience | Untested | Full Module 1 walkthrough |

## Desired End State

A user can:
1. Open OpenWebUI at localhost:3000
2. Create account / sign in
3. Start chat with college-essay agent
4. Receive warm, engaging first impression
5. Upload CliftonStrengths PDF
6. Have PDF analyzed and insights reflected back
7. Continue through Module 1 steps with memory updating correctly
8. See curriculum progression work via advance_lesson

### Success Criteria

#### Automated Verification
- [ ] `make verify-agent` passes (lint + typecheck + tests)
- [ ] All services start without errors
- [ ] `/health` endpoint returns 200

#### Manual Verification
- [ ] Agent delivers warm opening message matching step config
- [ ] PDF upload via OpenWebUI syncs to Letta within 30s
- [ ] Agent can read and analyze uploaded PDF
- [ ] Memory blocks update correctly (check via Letta ADE)
- [ ] `advance_lesson` moves to next step with correct opening
- [ ] Full Module 1 walkthrough feels excellent

## What We're NOT Doing

- Deployment to VPS (tracked in ARI-73)
- Production environment configuration
- Honcho production setup (using demo mode)
- Background grader automation (manual trigger for testing)
- Multiple user testing

---

## Phase 1: Local Infrastructure Setup

### Overview
Get all services running locally and talking to each other.

### Prerequisites
- OpenAI API key
- Docker installed and running

### Steps

#### 1.1 Create .env file

**File**: `/Users/ariasulin/Git/YouLab/.env`

```bash
# Core LLM
OPENAI_API_KEY=sk-your-key-here

# Letta Connection (Docker container name)
LETTA_BASE_URL=http://localhost:8283

# YouLab HTTP Service
YOULAB_SERVICE_HOST=0.0.0.0
YOULAB_SERVICE_PORT=8100

# Honcho (demo mode - no API key needed)
YOULAB_SERVICE_HONCHO_ENABLED=true
YOULAB_SERVICE_HONCHO_ENVIRONMENT=demo

# File Sync (ENABLE THIS)
YOULAB_SERVICE_FILE_SYNC_ENABLED=true
YOULAB_SERVICE_OPENWEBUI_URL=http://localhost:3000
YOULAB_SERVICE_FILE_SYNC_INTERVAL=30
YOULAB_SERVICE_FILE_SYNC_EMBEDDING_MODEL=openai/text-embedding-3-small
YOULAB_SERVICE_DATA_DIR=.data

# Observability
LOG_LEVEL=INFO
LOG_JSON=false
LANGFUSE_ENABLED=false
```

#### 1.2 Start Letta Server (Docker)

```bash
docker run -d \
  --name letta-server \
  -p 8283:8283 \
  -e OPENAI_API_KEY=$OPENAI_API_KEY \
  -v letta-data:/root/.letta \
  letta/letta:latest
```

Verify: http://localhost:8283/v1/health

#### 1.3 Start OpenWebUI (Docker)

```bash
docker run -d \
  --name open-webui \
  -p 3000:8080 \
  -v open-webui-data:/app/backend/data \
  -e WEBUI_AUTH=true \
  ghcr.io/open-webui/open-webui:main
```

Verify: http://localhost:3000

**Note**: The LettaPipe is already imported and configured in OpenWebUI. Just verify it's working.

#### 1.4 Start YouLab HTTP Service

```bash
# Terminal (or could also Dockerize later)
uv run letta-server
```

Verify: http://localhost:8100/health

#### 1.5 Verify Pipe Configuration

The pipe should already be configured. Verify in OpenWebUI Admin → Pipelines:
- `LETTA_SERVICE_URL`: `http://host.docker.internal:8100`
- `AGENT_TYPE`: `college-essay`
- `ENABLE_THINKING`: `true`

### Success Criteria

#### Automated
- [ ] All three services respond to health checks
- [ ] `GET /sync/status` shows `enabled: true, running: true`

#### Manual
- [ ] OpenWebUI loads without errors
- [ ] Pipe is working (already configured)

---

## Phase 2: Fix Folder Attachment Gap

### Overview
Wire up the TOML folder config to agent creation so uploaded files are accessible.

### The Problem
`config/courses/college-essay/course.toml` defines:
```toml
[agent.folders]
shared = ["college-essay-materials"]
private = true
```

But `AgentManager.create_agent_from_curriculum()` does NOT attach these folders.

### Changes Required

#### 2.1 Add folder attachment to agent creation

**File**: `src/letta_starter/server/agents.py`

After agent creation in `create_agent_from_curriculum()`, add folder attachment:

```python
# After agent is created (around line 295)
# Attach folders if sync service is available and folder config exists
if hasattr(course.agent, 'folders') and course.agent.folders:
    from letta_starter.server.sync.router import get_file_sync
    try:
        sync_service = get_file_sync()
        await self._attach_folders(
            agent_id=agent.id,
            user_id=user_id,
            folder_config=course.agent.folders,
            sync_service=sync_service,
        )
    except RuntimeError:
        # Sync service not enabled - skip folder attachment
        pass

async def _attach_folders(
    self,
    agent_id: str,
    user_id: str,
    folder_config: AgentFoldersConfig,
    sync_service: FileSyncService,
) -> list[str]:
    """Attach configured folders to agent."""
    attached = []

    # Attach shared folders
    for folder_name in folder_config.shared:
        folder_id = await sync_service.ensure_folder(folder_name)
        self.client.agents.folders.attach(agent_id=agent_id, folder_id=folder_id)
        attached.append(folder_name)
        self.log.info("attached_shared_folder", agent_id=agent_id, folder=folder_name)

    # Attach user's private folder
    if folder_config.private:
        private_folder = f"user_{user_id}_notes"
        folder_id = await sync_service.ensure_folder(private_folder)
        self.client.agents.folders.attach(agent_id=agent_id, folder_id=folder_id)
        attached.append(private_folder)
        self.log.info("attached_private_folder", agent_id=agent_id, folder=private_folder)

    return attached
```

#### 2.2 Make ensure_folder public

**File**: `src/letta_starter/server/sync/service.py`

Rename `_ensure_folder` to `ensure_folder` (remove underscore) to make it part of public API.

### Success Criteria

#### Automated
- [ ] `make verify-agent` passes
- [ ] Agent creation includes folder attachment

#### Manual
- [ ] Create new agent → folders appear in Letta ADE
- [ ] Upload file → syncs → agent can search it

---

## Phase 3: Test PDF Analysis Flow

### Overview
Verify the full CliftonStrengths PDF workflow works.

### Test Steps

1. **Upload PDF via OpenWebUI**
   - Create a Knowledge collection (or use existing)
   - Upload a sample CliftonStrengths PDF

2. **Verify Sync**
   - Wait 30 seconds (or trigger manually: `POST /sync/trigger`)
   - Check `GET /sync/mappings` for the file
   - Verify Letta processed it (check folder in ADE)

3. **Verify Agent Access**
   - Start chat with college-essay agent
   - Ask: "Can you see my uploaded files?"
   - Agent should be able to search/read the PDF content

4. **Test Analysis Quality**
   - Upload real CliftonStrengths PDF
   - Ask agent to analyze it
   - Verify insights match PDF content

### Potential Issues

| Issue | Solution |
|-------|----------|
| PDF not syncing | Check OpenWebUI API key in .env |
| PDF not converted | Ensure MISTRAL_API_KEY set for Letta |
| Agent can't see files | Verify folder attachment worked |
| Poor analysis | May need prompt tuning |

### Success Criteria

#### Manual
- [ ] PDF appears in Letta folder as markdown
- [ ] Agent correctly identifies top 5 strengths
- [ ] Agent provides meaningful insights about strength combinations

---

## Phase 4: Full Module 1 Walkthrough

### Overview
Test the complete user journey through Module 1's three steps.

### Test Script

#### Step 1: strengths-upload
**Expected Opening**:
> "Welcome to YouLab! I'm thrilled to be your guide..."

**Test Flow**:
1. Receive warm opening message
2. Respond with initial greeting
3. Upload CliftonStrengths PDF (via Knowledge collection)
4. Agent should analyze and reflect back insights
5. Agent should update `student.profile` memory

**Verify**:
- [ ] Opening matches `01-first-impression.toml:33-37`
- [ ] Agent is warm and enthusiastic
- [ ] PDF analysis is thorough
- [ ] Memory block updated (check Letta ADE)

#### Step 2: strengths-deepdive
**Trigger**: Agent calls `advance_lesson` after PDF analyzed

**Expected Opening**:
> "Now that I've seen your strengths profile, I'm curious how these show up..."

**Test Flow**:
1. Share a story about top strength
2. Agent probes deeper ("What did that feel like?")
3. Notice energy shifts and follow them
4. Agent captures themes in `student.insights`

**Verify**:
- [ ] advance_lesson moved to step 2
- [ ] Opening matches config
- [ ] Agent asks for concrete stories
- [ ] Insights captured in memory

#### Step 3: strengths-bridge
**Trigger**: Agent calls `advance_lesson` after sufficient exploration

**Expected Opening**:
> "You've shared some powerful moments. I'm starting to see threads..."

**Test Flow**:
1. Reflect on patterns across stories
2. Identify 2-3 essay angles
3. Validate which resonates most
4. End with clarity on direction

**Verify**:
- [ ] Agent helps see patterns
- [ ] Doesn't overwhelm with options
- [ ] Student feels understood
- [ ] `auto_advance = false` respected

### Experience Quality Checklist

- [ ] Agent never writes essay content
- [ ] Agent goes deep, not wide
- [ ] Agent celebrates small wins
- [ ] Memory blocks feel accurate
- [ ] Progression feels natural, not rushed

---

## Phase 5: Polish and Document

### Overview
Fix any issues discovered and document the local setup.

### Tasks

1. **Fix discovered issues** - based on Phase 4 testing
2. **Tune prompts** - if agent behavior needs adjustment
3. **Add inline TODOs** - for any deferred improvements
4. **Update quickstart docs** - if setup process changed

### Success Criteria

- [ ] Module 1 experience feels excellent
- [ ] No blocking bugs
- [ ] Clear path to production deployment

---

## References

- Course config: `config/courses/college-essay/course.toml`
- Module 1 config: `config/courses/college-essay/modules/01-first-impression.toml`
- File sync plan: `thoughts/shared/plans/2026-01-10-file-knowledge-sync.md`
- Deployment ticket: ARI-73
