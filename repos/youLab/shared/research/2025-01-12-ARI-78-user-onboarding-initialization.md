---
date: 2026-01-12T00:00:00-08:00
researcher: Claude
git_commit: 5921af254188eca4151966cebff1e202da4db885
branch: main
repository: YouLab
topic: "User Onboarding and Initialization for YouLab Product Spec"
tags: [research, user-onboarding, initialization, git, memory-blocks, background-agents]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude
linear_ticket: ARI-78
---

# Research: User Onboarding and Initialization

**Date**: 2026-01-12
**Researcher**: Claude
**Git Commit**: 5921af254188eca4151966cebff1e202da4db885
**Branch**: main
**Repository**: YouLab
**Linear Ticket**: [ARI-78](https://linear.app/ariav/issue/ARI-78)

## Research Question

Design the flow for initializing new users with git repos, memory blocks, and background agents, as specified in the YouLab Product Spec (`thoughts/shared/plans/2025-01-12-youlab-product-spec.md`).

## Summary

The research reveals that **YouLab currently has no explicit user initialization system**. Users are identified by OpenWebUI user IDs, and resources are created lazily on first interaction. To implement the product spec's per-user sandbox architecture, we need to:

1. **Hook into OpenWebUI's webhook system** for user creation events
2. **Implement git repository initialization** (no existing git integration)
3. **Leverage existing memory block infrastructure** from curriculum loader
4. **Extend background agent runner** to support per-user agent associations

The product spec envisions per-user VPS folders at `/data/users/{user_id}/` with git-backed TOML files, which requires new infrastructure not currently present in the codebase.

---

## Detailed Findings

### 1. OpenWebUI User Creation Flow

**Entry Points for User Creation:**

| Method | Endpoint | Location |
|--------|----------|----------|
| Standard Signup | `POST /signup` | `OpenWebUI/open-webui/backend/open_webui/routers/auths.py:642` |
| Admin Add User | `POST /add` | `OpenWebUI/open-webui/backend/open_webui/routers/auths.py:840` |
| LDAP Auth | `POST /ldap` | `OpenWebUI/open-webui/backend/open_webui/routers/auths.py:216` |
| OAuth/OIDC | Callback handler | `OpenWebUI/open-webui/backend/open_webui/utils/oauth.py:1523` |
| SCIM Provisioning | `POST /Users` | `OpenWebUI/open-webui/backend/open_webui/routers/scim.py:569` |

**Core User Creation Process:**

1. Validates email/password (`auths.py:661-673`)
2. Calls `Auths.insert_new_auth()` (`auths.py:678`)
3. Creates `Auth` record (credentials) + `User` record (profile)
4. Returns JWT token
5. **Post-creation hooks:**
   - Webhook notification if `WEBHOOK_URL` configured (`auths.py:713-723`)
   - Default group assignment via `apply_default_group_assignment()` (`auths.py:733-736`)

**User Data Storage:**

- `auth` table: `id`, `email`, `password` (hash), `active`
- `user` table: `id`, `email`, `name`, `role`, `profile_image_url`, `settings` (JSON), `info` (JSON), `oauth` (JSON), timestamps

**Key Models:**
- `OpenWebUI/open-webui/backend/open_webui/models/auths.py:17-24` - Auth schema
- `OpenWebUI/open-webui/backend/open_webui/models/users.py:45-76` - User schema

### 2. Available User Lifecycle Hooks

**Webhook System (Best Integration Point):**

- **Configuration:** `WEBHOOK_URL` in app config (`config.py:1577-1578`)
- **Implementation:** `OpenWebUI/open-webui/backend/open_webui/utils/webhook.py:11` - `post_webhook()`
- **Triggered on:** Standard signup (`auths.py:713-723`) and OAuth signup (`oauth.py:1534-1542`)
- **NOT triggered on:** Admin add user, LDAP login, SCIM provisioning

**Webhook Payload Format:**
```json
{
  "action": "signup",
  "message": "New user signup: {name}",
  "user": { /* serialized user object */ }
}
```

**Alternative Hook Points:**

1. **Socket.IO Events** (`socket/main.py:316`): `user-join` event when user connects
2. **Middleware** (`main.py:1236-1345`): Can intercept all HTTP requests
3. **Database Event Listeners** (`internal/db.py:115-123`): SQLAlchemy connection events
4. **Pipeline Filters** (`routers/pipelines.py:59-144`): Inlet/outlet filters for chat processing

**Recommendation:** Use webhook system + add YouLab-specific endpoint for user initialization.

### 3. Current YouLab User Handling

**No Explicit User Initialization:**

YouLab implements a "just-in-time" model where resources are created lazily:

1. **Agent Creation** (`src/youlab_server/server/agents.py:220-367`):
   - Triggered via `POST /agents` or implicitly via OpenWebUI Pipe
   - Creates Letta agent with user ID in metadata
   - Creates memory blocks based on course config

2. **User ID Source:**
   - Always comes from OpenWebUI (`__user__.get("id")`)
   - Never created internally by YouLab

3. **User Discovery:**
   - Inferred from agent names: `youlab_{user_id}_{agent_type}`
   - No central user registry
   - `AgentManager.rebuild_cache()` (`agents.py:157-170`) rebuilds on startup

4. **Private Folder Creation:**
   - Per-user folders created as `user_{user_id}_notes` (`agents.py:148-153`)
   - Created in Letta, not filesystem

### 4. Git Integration Status

**No Git Integration Exists:**

- No GitPython, pygit2, dulwich imports
- No subprocess git calls
- No `.git` directory operations
- No git-related dependencies in `pyproject.toml`

**Referenced but Missing:**
- `hack/` directory mentioned in CLAUDE.md does not exist
- Worktree scripts referenced but not present

**Required Implementation:**

To implement product spec's git-backed memory:

```
/data/users/{user_id}/
├── .git/                     # Hidden version control
├── config/
│   ├── schema.toml          # Memory schema
│   └── agents/              # Background agent configs
├── memory/                   # Memory block TOML files
├── notes/                    # Active notes
└── files/                    # Archival files
```

**Options for Git Implementation:**
1. **GitPython** - High-level Python Git library (recommended for simplicity)
2. **pygit2** - libgit2 bindings (lower-level, better performance)
3. **dulwich** - Pure Python Git (no native deps)
4. **Subprocess** - Shell out to git command

### 5. Memory Block Initialization

**Existing Infrastructure:**

Memory blocks are fully supported via TOML configuration:

1. **Schema Definition** (`config/courses/college-essay/course.toml:86-109`):
```toml
[block.student]
label = "human"
shared = false
field.profile = { type = "string", default = "" }
field.insights = { type = "string", default = "" }
```

2. **Schema Classes** (`src/youlab_server/curriculum/schema.py:197-215`):
   - `FieldSchema`: type, default, options, max, description, required
   - `BlockSchema`: label, description, shared, fields

3. **Dynamic Model Generation** (`src/youlab_server/curriculum/blocks.py:119-169`):
   - `create_block_model()` generates Pydantic classes at runtime
   - `DynamicBlock.to_memory_string()` serializes to Letta format

4. **Agent Creation with Blocks** (`src/youlab_server/server/agents.py:267-306`):
   - Loads block registry from curriculum
   - Creates instances with defaults and overrides
   - Separates shared vs per-agent blocks

**For Product Spec Implementation:**

The existing system writes to Letta, not files. Need to:
1. Add file-based storage layer
2. Serialize to TOML files in `/data/users/{user_id}/memory/`
3. Initialize git repo with initial commit

**Default Memory Schema (from product spec):**
- Operating Manual
- Values
- Personality
- Listening Style
- Strengths
- Goals

### 6. Background Agent Setup

**Existing Infrastructure:**

1. **Configuration** (`config/courses/college-essay/course.toml:114-136`):
```toml
[[task]]
name = "progression-grader"
on_idle = true
manual = true
queries = [
    { target = "journey.grader_notes", question = "...", merge = "replace" }
]
```

2. **Runner** (`src/youlab_server/background/runner.py:46-133`):
   - `BackgroundAgentRunner` executes background agents
   - Discovers users by parsing agent names (`youlab_{user_id}_{agent_type}`)
   - Queries Honcho for insights
   - Applies to agent memory via `MemoryEnricher`

3. **Triggers** (configured but only manual implemented):
   - `manual = true` - HTTP endpoint (`POST /background/{agent_id}/run`)
   - `schedule` - Cron expression (not implemented)
   - `on_idle` - Idle detection (not implemented)

4. **Memory Enrichment** (`src/youlab_server/memory/enricher.py:47-254`):
   - Applies updates with merge strategies (append, replace, llm_diff)
   - Writes audit trail to archival memory

**For Product Spec Implementation:**

Background agents currently:
- Run against Letta agents (memory in Letta)
- Propose changes that are auto-applied

Product spec requires:
- Agents propose diffs to file-based memory
- Human approval required before applying
- Git commits for each approved change

---

## Proposed User Onboarding Flow

Based on the research, here is the recommended initialization sequence:

### Step 1: Hook into OpenWebUI User Creation

**Option A: Webhook Listener (Recommended)**

Configure `WEBHOOK_URL` to point to YouLab endpoint:

```python
# src/youlab_server/server/users.py (new file)
@router.post("/users/init")
async def initialize_user(webhook_payload: dict):
    """Initialize YouLab resources for new OpenWebUI user."""
    if webhook_payload.get("action") != "signup":
        return

    user = webhook_payload["user"]
    user_id = user["id"]
    user_name = user.get("name", "")

    await create_user_sandbox(user_id, user_name)
```

**Option B: OpenWebUI Code Modification**

Add YouLab initialization call after user creation in `auths.py:686`:

```python
# After user creation, before return
await youlab_initialize_user(user.id, user.name)
```

### Step 2: Create User Sandbox

```python
async def create_user_sandbox(user_id: str, user_name: str):
    """
    Create per-user sandbox directory structure.

    /data/users/{user_id}/
    ├── .git/
    ├── config/
    │   ├── schema.toml
    │   └── agents/
    ├── memory/
    ├── notes/
    └── files/
    """
    base_path = Path(f"/data/users/{user_id}")

    # Create directory structure
    for subdir in ["config", "config/agents", "memory", "notes", "files"]:
        (base_path / subdir).mkdir(parents=True, exist_ok=True)

    # Initialize git repository
    repo = git.Repo.init(base_path)

    # Create default schema.toml
    await create_default_schema(base_path / "config" / "schema.toml")

    # Create default memory blocks
    await create_default_memory_blocks(base_path / "memory", user_name)

    # Initial commit
    repo.index.add(["config/", "memory/"])
    repo.index.commit(f"Initialize user sandbox for {user_name}")
```

### Step 3: Initialize Default Memory Blocks

```python
async def create_default_memory_blocks(memory_dir: Path, user_name: str):
    """Create default memory block TOML files."""

    default_blocks = {
        "operating-manual.toml": {
            "content": "",
            "last_modified": datetime.now().isoformat(),
            "modified_by": "system",
        },
        "values.toml": {
            "content": "",
            "last_modified": datetime.now().isoformat(),
            "modified_by": "system",
        },
        "personality.toml": {
            "content": "",
            "last_modified": datetime.now().isoformat(),
            "modified_by": "system",
        },
        "listening-style.toml": {
            "content": "",
            "last_modified": datetime.now().isoformat(),
            "modified_by": "system",
        },
        "strengths.toml": {
            "content": "",
            "last_modified": datetime.now().isoformat(),
            "modified_by": "system",
        },
        "goals.toml": {
            "content": "",
            "last_modified": datetime.now().isoformat(),
            "modified_by": "system",
        },
    }

    for filename, content in default_blocks.items():
        with open(memory_dir / filename, "wb") as f:
            tomli_w.dump(content, f)
```

### Step 4: Register with Letta

Extend existing `AgentManager.create_agent_from_curriculum()` to:
1. Link agent metadata to user sandbox path
2. Sync memory blocks from TOML files to Letta
3. Set up bidirectional sync

### Step 5: Set Up Background Agent Associations

```python
async def setup_background_agents(user_id: str, course_id: str):
    """Associate background agents with user."""
    course = curriculum.get(course_id)

    for task_name, task_config in course.tasks.items():
        # Register user for background processing
        await background_runner.register_user_for_task(
            user_id=user_id,
            task_name=task_name,
            config=task_config,
        )
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         User Registration Flow                          │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   OpenWebUI  │───▶│   Webhook    │───▶│   YouLab     │───▶│  User        │
│   Signup     │    │   POST       │    │   /users/init│    │  Sandbox     │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                    │
                    ┌───────────────────────────────────────────────┼───────┐
                    │                                               │       │
                    ▼                                               ▼       ▼
            ┌──────────────┐                              ┌──────────────────┐
            │  Git Init    │                              │  Memory Blocks   │
            │  .git/       │                              │  TOML files      │
            └──────────────┘                              └──────────────────┘
                    │                                               │
                    │         ┌──────────────────────┐              │
                    └────────▶│   Initial Commit     │◀─────────────┘
                              │   "Initialize user"  │
                              └──────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
            ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
            │   Letta      │    │   Honcho     │    │  Background  │
            │   Agent      │    │   Peer       │    │  Agent Reg   │
            └──────────────┘    └──────────────┘    └──────────────┘
```

---

## Code References

### OpenWebUI User Creation
- `OpenWebUI/open-webui/backend/open_webui/routers/auths.py:642-736` - Standard signup
- `OpenWebUI/open-webui/backend/open_webui/routers/auths.py:678` - `Auths.insert_new_auth()` call
- `OpenWebUI/open-webui/backend/open_webui/utils/webhook.py:11` - Webhook implementation

### YouLab Agent Management
- `src/youlab_server/server/agents.py:220-367` - `create_agent_from_curriculum()`
- `src/youlab_server/server/agents.py:42-51` - Agent naming and metadata
- `src/youlab_server/server/agents.py:157-170` - Cache rebuild on startup

### Memory Block System
- `src/youlab_server/curriculum/schema.py:197-215` - BlockSchema, FieldSchema
- `src/youlab_server/curriculum/blocks.py:119-169` - `create_block_model()`
- `src/youlab_server/curriculum/blocks.py:48-117` - DynamicBlock base class

### Background Agents
- `src/youlab_server/background/runner.py:67-133` - `run_agent()` execution
- `src/youlab_server/background/runner.py:135-155` - User discovery
- `src/youlab_server/server/background.py:96-138` - HTTP endpoints

---

## Implementation Checklist

### Phase 1: Git Infrastructure
- [ ] Add GitPython to dependencies
- [ ] Create `UserSandboxService` class
- [ ] Implement `create_user_sandbox()` with directory structure
- [ ] Implement `init_git_repo()` with initial commit
- [ ] Add git operations for commits, diffs, history

### Phase 2: User Initialization Endpoint
- [ ] Create `/users/init` webhook endpoint
- [ ] Configure OpenWebUI `WEBHOOK_URL`
- [ ] Handle all signup methods (standard, OAuth, LDAP, SCIM)
- [ ] Add idempotency for repeated calls

### Phase 3: Memory Block File Storage
- [ ] Create `FileMemoryStore` class
- [ ] Implement TOML read/write for memory blocks
- [ ] Sync file changes to Letta memory
- [ ] Sync Letta changes to files (bidirectional)

### Phase 4: Background Agent Integration
- [ ] Extend `BackgroundAgentRunner` for file-based memory
- [ ] Implement diff proposal format
- [ ] Add pending changes tracking
- [ ] Implement approval workflow

### Phase 5: Version Control UI
- [ ] Add history endpoint for memory blocks
- [ ] Implement rollback functionality
- [ ] Create diff visualization helpers

---

## Open Questions

1. **Webhook Reliability:** What happens if YouLab endpoint is down during signup? Need retry/queue?

2. **Migration Path:** How to create sandboxes for existing users?

3. **Storage Location:** `/data/users/` on VPS - how to configure path? Environment variable?

4. **Git Library Choice:** GitPython vs pygit2 vs dulwich? Performance vs simplicity tradeoff.

5. **Sync Strategy:** How to handle conflicts between Letta memory and file-based memory?

6. **SCIM/Admin Users:** SCIM doesn't trigger webhook - need alternative hook?

---

## Related Research

- `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Product specification
