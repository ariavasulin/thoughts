# Module-Thread Binding Implementation Plan

## Overview

Implement 1:1 binding between course modules and OpenWebUI threads. Each module a user enrolls in gets exactly one thread, created upfront on enrollment. The sidebar shows modules as direct links to their threads (not model selectors). Ralph determines which module a thread belongs to and loads the appropriate TOML configuration.

## Current State Analysis

**OpenWebUI Frontend** (custom YouLab fork):
- Modules displayed in sidebar via `ModuleList.svelte` and `ModuleItem.svelte`
- Currently treats modules as models with `youlab_module` metadata
- Creates folders per module, finds/creates threads in folders
- Complex indirection: Module → Model → Folder → Thread

**Ralph Backend**:
- Single hardcoded agent for all users (`src/ralph/server.py:283-297`)
- No course/module awareness - receives `user_id` and `chat_id` only
- No TOML configuration loading (exists in legacy YouLab Server)
- Background task infrastructure exists and works

**Honcho**:
- Already has 1:1 thread binding: `chat_id` → `chat_{chat_id}` session
- No changes needed

### Key Discoveries:
- OpenWebUI has native webhook support for user signup (`WEBHOOK_URL` env var)
- OpenWebUI has chat archiving (`POST /api/chats/{id}/archive`) - archived chats hidden from sidebar
- Pipe can access user's token via `__request__.cookies.get("token")`
- Admin API key available for service-level API calls

## Desired End State

1. **User signs up** → webhook fires to Ralph → Ralph creates module threads → threads archived (hidden from Chats)
2. **Sidebar modules** → direct links to thread IDs (fetched from Ralph API)
3. **User clicks module** → navigates to `/c/{thread_id}` → sees `first_message` content
4. **User sends message** → Ralph looks up `chat_id` → finds module → loads TOML config → builds appropriate agent
5. **Module progression** tracked in memory blocks (e.g., `onboarding_progress`)

### Verification:
- New user signup creates Welcome module thread automatically
- Thread appears in Module sidebar, not in Chats section
- First message shows welcome.toml `[first_message].content`
- Ralph uses welcome.toml config (system prompt, tools, blocks) for responses

## What We're NOT Doing

- Multi-module courses with progression/locking (future scope)
- LDAP/admin-created user enrollment (regular signup + OAuth only for MVP)
- Module-specific tools beyond what welcome.toml defines
- Migrating legacy YouLab Server curriculum loader (fresh implementation for Ralph)

## Implementation Approach

The implementation follows this dependency chain:

1. **Database schema** - Foundation for storing enrollments and module-thread mappings
2. **TOML loading** - Load course/module configurations
3. **Enrollment API** - Webhook endpoint + thread creation logic
4. **Module routing** - Ralph looks up module from chat_id, builds appropriate agent
5. **Frontend integration** - Sidebar fetches module list from Ralph, links to threads

---

## Phase 1: Database Schema & Configuration

### Overview
Add Dolt tables for course enrollment and module-thread mapping. Add configuration for OpenWebUI API access.

### Changes Required:

#### 1. Dolt Schema Migration
**File**: `src/ralph/dolt.py`
**Changes**: Add new tables and CRUD methods

```python
# Add to DoltClient class

# Table creation (add to _ensure_tables or create migration)
"""
CREATE TABLE IF NOT EXISTS user_courses (
    user_id VARCHAR(36) NOT NULL,
    course_id VARCHAR(64) NOT NULL,
    enrolled_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, course_id)
);

CREATE TABLE IF NOT EXISTS user_modules (
    user_id VARCHAR(36) NOT NULL,
    course_id VARCHAR(64) NOT NULL,
    module_id VARCHAR(64) NOT NULL,
    thread_id VARCHAR(36) NOT NULL,
    status ENUM('locked', 'available', 'in_progress', 'completed') DEFAULT 'available',
    started_at TIMESTAMP NULL,
    completed_at TIMESTAMP NULL,
    PRIMARY KEY (user_id, course_id, module_id),
    INDEX idx_thread_lookup (thread_id)
);
"""

# CRUD methods to add:
async def enroll_user_in_course(self, user_id: str, course_id: str) -> None: ...
async def is_user_enrolled(self, user_id: str, course_id: str) -> bool: ...
async def create_user_module(self, user_id: str, course_id: str, module_id: str, thread_id: str) -> None: ...
async def get_user_modules(self, user_id: str, course_id: str) -> list[dict]: ...
async def get_module_by_thread_id(self, thread_id: str) -> dict | None: ...
async def update_module_status(self, user_id: str, course_id: str, module_id: str, status: str) -> None: ...
```

#### 2. Configuration Updates
**File**: `src/ralph/config.py`
**Changes**: Add OpenWebUI API configuration

```python
# Add to Settings class
openwebui_base_url: str = "http://localhost:8080"
openwebui_admin_api_key: str = ""  # sk-... format

# Environment variables:
# RALPH_OPENWEBUI_BASE_URL
# RALPH_OPENWEBUI_ADMIN_API_KEY
```

### Success Criteria:

#### Automated Verification:
- [ ] Dolt migration applies cleanly: `docker exec youlab-dolt-1 dolt sql -q "SHOW TABLES"`
- [ ] Tables exist: `user_courses`, `user_modules`
- [ ] Type checking passes: `make check-agent`
- [ ] Unit tests pass for new Dolt methods

#### Manual Verification:
- [ ] Can insert/query enrollment records via SQL
- [ ] Config loads API key from environment

---

## Phase 2: TOML Course Loading

### Overview
Create a minimal TOML loader for Ralph that loads course configurations. Simpler than legacy loader - no v1/v2 schema complexity.

### Changes Required:

#### 1. Course Schema
**File**: `src/ralph/curriculum/schema.py` (new file)
**Changes**: Pydantic models for course configuration

```python
from pydantic import BaseModel, Field

class BlockConfig(BaseModel):
    """Memory block definition."""
    label: str
    title: str
    template: str = ""

class TaskTrigger(BaseModel):
    """Background task trigger configuration."""
    type: str  # "idle" or "cron"
    idle_minutes: int | None = None
    cooldown_minutes: int | None = None
    schedule: str | None = None  # cron expression

class TaskConfig(BaseModel):
    """Background task definition."""
    name: str
    trigger: TaskTrigger
    system_prompt: str
    tools: list[str] = Field(default_factory=list)
    blocks: list[str] = Field(default_factory=list)

class ModuleConfig(BaseModel):
    """Module definition (1 module = 1 thread)."""
    id: str
    name: str
    description: str = ""
    order: int = 0

class AgentConfig(BaseModel):
    """Agent configuration from TOML."""
    name: str
    model: str = "anthropic/claude-sonnet-4"
    system_prompt: str
    tools: list[str] = Field(default_factory=list)
    blocks: list[str] = Field(default_factory=list)

class FirstMessageConfig(BaseModel):
    """First message shown when user starts module."""
    content: str

class CourseConfig(BaseModel):
    """Complete course configuration."""
    agent: AgentConfig
    blocks: list[BlockConfig] = Field(default_factory=list)
    tasks: list[TaskConfig] = Field(default_factory=list)
    modules: list[ModuleConfig] = Field(default_factory=list)
    first_message: FirstMessageConfig | None = None
```

#### 2. Course Loader
**File**: `src/ralph/curriculum/loader.py` (new file)
**Changes**: TOML parsing and caching

```python
import tomllib
from pathlib import Path
from functools import lru_cache

from ralph.curriculum.schema import CourseConfig, ModuleConfig, BlockConfig, ...

class CourseLoader:
    """Loads course configurations from TOML files."""

    def __init__(self, config_dir: Path | str = "config/courses"):
        self.config_dir = Path(config_dir)
        self._cache: dict[str, CourseConfig] = {}

    def list_courses(self) -> list[str]:
        """List available course IDs."""
        ...

    def load(self, course_id: str) -> CourseConfig:
        """Load a course configuration."""
        ...

    def get_module(self, course_id: str, module_id: str) -> ModuleConfig | None:
        """Get a specific module from a course."""
        ...

    def reload(self) -> None:
        """Clear cache and reload all courses."""
        self._cache.clear()

# Singleton instance
_loader: CourseLoader | None = None

def get_course_loader() -> CourseLoader:
    global _loader
    if _loader is None:
        _loader = CourseLoader()
    return _loader
```

#### 3. Welcome Course TOML
**File**: `config/courses/welcome/course.toml` (new file)
**Changes**: Move welcome.toml content to proper location

```toml
# Copy from /Users/ariasulin/Desktop/welcome.toml
# Adjust format if needed to match schema
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Unit tests for loader: `pytest tests/ralph/curriculum/`
- [ ] Welcome course loads without errors

#### Manual Verification:
- [ ] `CourseLoader().load("welcome")` returns valid config
- [ ] All 4 blocks parsed correctly
- [ ] First message content loaded

---

## Phase 3: Enrollment API & Thread Creation

### Overview
Add webhook endpoint that receives user signup events, creates module threads via OpenWebUI API, and stores mappings in Dolt.

### Changes Required:

#### 1. OpenWebUI API Client
**File**: `src/ralph/openwebui.py` (new file)
**Changes**: Client for OpenWebUI chat management API

```python
import httpx
from ralph.config import settings

class OpenWebUIClient:
    """Client for OpenWebUI API operations."""

    def __init__(self):
        self.base_url = settings.openwebui_base_url
        self.api_key = settings.openwebui_admin_api_key

    async def create_chat(
        self,
        user_id: str,
        title: str,
        model: str,
        initial_message: str | None = None,
    ) -> str:
        """Create a new chat and return its ID."""
        # POST /api/chats/new
        # Returns chat object with id
        ...

    async def archive_chat(self, chat_id: str) -> None:
        """Archive a chat (hide from sidebar)."""
        # POST /api/chats/{id}/archive
        ...

    async def create_and_archive_module_thread(
        self,
        user_id: str,
        module_name: str,
        model: str,
        first_message: str,
    ) -> str:
        """Create a module thread, inject first message, archive it."""
        chat_id = await self.create_chat(
            user_id=user_id,
            title=module_name,
            model=model,
            initial_message=first_message,
        )
        await self.archive_chat(chat_id)
        return chat_id
```

**Note**: The OpenWebUI API requires the request to be made on behalf of a user. We may need to:
- Use admin impersonation if supported, OR
- Pass user token through the webhook payload (requires OpenWebUI modification), OR
- Create chats as admin and transfer ownership

Research needed on exact API behavior for creating chats for other users.

#### 2. Enrollment Service
**File**: `src/ralph/enrollment.py` (new file)
**Changes**: Enrollment logic

```python
from ralph.curriculum.loader import get_course_loader
from ralph.dolt import get_dolt_client
from ralph.openwebui import OpenWebUIClient

class EnrollmentService:
    """Handles user enrollment in courses."""

    def __init__(self):
        self.loader = get_course_loader()
        self.dolt = get_dolt_client()
        self.openwebui = OpenWebUIClient()

    async def enroll_user(self, user_id: str, course_id: str = "welcome") -> dict:
        """
        Enroll a user in a course.

        1. Check if already enrolled
        2. Load course config
        3. Create thread for each module
        4. Archive threads
        5. Initialize memory blocks from templates
        6. Store enrollment in Dolt

        Returns dict with module_id -> thread_id mapping.
        """
        ...

    async def _create_module_thread(
        self,
        user_id: str,
        course: CourseConfig,
        module: ModuleConfig,
    ) -> str:
        """Create and archive a thread for a module."""
        first_message = course.first_message.content if course.first_message else ""

        thread_id = await self.openwebui.create_and_archive_module_thread(
            user_id=user_id,
            module_name=module.name,
            model="ralph-wiggum",  # The pipe model ID
            first_message=first_message,
        )
        return thread_id

    async def _initialize_blocks(self, user_id: str, course: CourseConfig) -> None:
        """Initialize memory blocks from course templates."""
        for block in course.blocks:
            existing = await self.dolt.get_memory_block(user_id, block.label)
            if existing is None:
                await self.dolt.upsert_memory_block(
                    user_id=user_id,
                    label=block.label,
                    title=block.title,
                    body=block.template,
                    schema_ref=f"{course.agent.name}/{block.label}",
                )
```

#### 3. Webhook Endpoint
**File**: `src/ralph/server.py`
**Changes**: Add enrollment webhook endpoint

```python
from ralph.enrollment import EnrollmentService

@app.post("/webhook/user-signup")
async def handle_user_signup(request: Request) -> dict:
    """
    Webhook called by OpenWebUI on user signup.

    Expected payload:
    {
        "action": "signup",
        "user": "{\"id\": \"...\", \"email\": \"...\", \"name\": \"...\"}"
    }
    """
    payload = await request.json()

    if payload.get("action") != "signup":
        return {"status": "ignored", "reason": "not a signup event"}

    user_data = json.loads(payload.get("user", "{}"))
    user_id = user_data.get("id")

    if not user_id:
        raise HTTPException(400, "Missing user ID in payload")

    service = EnrollmentService()
    result = await service.enroll_user(user_id, course_id="welcome")

    return {"status": "enrolled", "modules": result}
```

#### 4. OpenWebUI Webhook Configuration
**Changes**: Set environment variable

```bash
# In OpenWebUI environment
WEBHOOK_URL=http://ralph-server:8200/webhook/user-signup
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Unit tests for enrollment service
- [ ] Integration test: POST to webhook endpoint creates records in Dolt

#### Manual Verification:
- [ ] Sign up new user in OpenWebUI
- [ ] Verify webhook fires (check Ralph logs)
- [ ] Verify thread created in OpenWebUI (check via API or DB)
- [ ] Verify thread is archived (not visible in Chats)
- [ ] Verify `user_courses` and `user_modules` rows created in Dolt
- [ ] Verify memory blocks initialized with templates

**Implementation Note**: This phase has the most uncertainty around OpenWebUI API behavior. May need to iterate on how threads are created for users.

---

## Phase 4: Module-Aware Agent Routing

### Overview
When Ralph receives a chat request, look up which module owns that thread and build an agent using the module's course configuration.

### Changes Required:

#### 1. Module Lookup in Chat Handler
**File**: `src/ralph/server.py`
**Changes**: Look up module before building agent

```python
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest) -> StreamingResponse:
    user_id = request.user_id
    chat_id = request.chat_id

    # Look up module for this thread
    dolt = get_dolt_client()
    module_info = await dolt.get_module_by_thread_id(chat_id)

    if module_info:
        # This is a module thread - use course config
        course_id = module_info["course_id"]
        module_id = module_info["module_id"]

        loader = get_course_loader()
        course = loader.load(course_id)

        agent = await build_agent_from_course(
            user_id=user_id,
            course=course,
            workspace=workspace,
        )

        # Update module status if first message
        if module_info["status"] == "available":
            await dolt.update_module_status(
                user_id, course_id, module_id, "in_progress"
            )
    else:
        # Not a module thread - use default agent (existing behavior)
        agent = build_default_agent(user_id, workspace)

    # ... rest of streaming logic
```

#### 2. Course-Based Agent Builder
**File**: `src/ralph/agent_builder.py` (new file)
**Changes**: Build agent from course configuration

```python
from agno import Agent
from agno.models.openrouter import OpenRouter

from ralph.curriculum.schema import CourseConfig
from ralph.tools import get_tool_by_name
from ralph.memory import build_memory_context

async def build_agent_from_course(
    user_id: str,
    course: CourseConfig,
    workspace: Path,
) -> Agent:
    """Build an Agno agent from course configuration."""

    # Build tools from course config
    tools = []
    for tool_name in course.agent.tools:
        tool = get_tool_by_name(tool_name, workspace=workspace, user_id=user_id)
        if tool:
            tools.append(tool)

    # Build memory context from course blocks
    memory_context = await build_memory_context(
        user_id=user_id,
        block_labels=course.agent.blocks,
    )

    # Compose instructions
    instructions = f"""
{course.agent.system_prompt}

---

## Your Memory

{memory_context}
"""

    return Agent(
        model=OpenRouter(id=course.agent.model, ...),
        tools=tools,
        instructions=instructions,
        markdown=True,
    )
```

#### 3. Tool Registry
**File**: `src/ralph/tools/__init__.py`
**Changes**: Add tool lookup by name

```python
from ralph.tools.memory_blocks import MemoryBlockTools
from ralph.tools.honcho_tools import HonchoTools
from ralph.tools.latex_tools import LaTeXTools
# ... other tools

def get_tool_by_name(name: str, workspace: Path, user_id: str) -> object | None:
    """Get a tool instance by name."""
    registry = {
        "memory_blocks": lambda: MemoryBlockTools(user_id=user_id),
        "honcho_tools": lambda: HonchoTools(user_id=user_id),
        "file_tools": lambda: FileTools(base_dir=workspace),
        "shell_tools": lambda: ShellTools(base_dir=workspace),
        "latex_tools": lambda: LaTeXTools(workspace=workspace),
    }

    factory = registry.get(name)
    return factory() if factory else None
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Unit tests for agent builder
- [ ] Unit tests for tool registry

#### Manual Verification:
- [ ] Send message to module thread
- [ ] Verify Ralph uses course system prompt (check logs or response style)
- [ ] Verify memory blocks from course are included in context
- [ ] Verify module status changes to "in_progress" after first message

---

## Phase 5: Frontend Integration

### Overview
Update OpenWebUI sidebar to fetch module list from Ralph API and link directly to thread IDs.

### Changes Required:

#### 1. Module List API
**File**: `src/ralph/server.py` (or `src/ralph/api/modules.py`)
**Changes**: Add endpoint for frontend to fetch user's modules

```python
@app.get("/users/{user_id}/modules")
async def get_user_modules(user_id: str) -> list[dict]:
    """
    Get user's enrolled modules with thread IDs.

    Returns:
    [
        {
            "course_id": "welcome",
            "module_id": "welcome",
            "module_name": "Welcome",
            "thread_id": "abc-123",
            "status": "available" | "in_progress" | "completed",
            "order": 0
        }
    ]
    """
    dolt = get_dolt_client()
    loader = get_course_loader()

    modules = await dolt.get_user_modules(user_id, course_id=None)  # All courses

    result = []
    for m in modules:
        course = loader.load(m["course_id"])
        module_config = loader.get_module(m["course_id"], m["module_id"])

        result.append({
            "course_id": m["course_id"],
            "module_id": m["module_id"],
            "module_name": module_config.name if module_config else m["module_id"],
            "thread_id": m["thread_id"],
            "status": m["status"],
            "order": module_config.order if module_config else 0,
        })

    return sorted(result, key=lambda x: x["order"])
```

#### 2. Update ModuleList.svelte
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleList.svelte`
**Changes**: Fetch modules from Ralph API instead of models

```svelte
<script lang="ts">
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import ModuleItem from './ModuleItem.svelte';

    // Fetch from Ralph API instead of $models
    let modules: Module[] = [];
    let loading = true;

    onMount(async () => {
        const userId = /* get from store */;
        const response = await fetch(`${RALPH_API_URL}/users/${userId}/modules`, {
            headers: { Authorization: `Bearer ${localStorage.token}` }
        });
        modules = await response.json();
        loading = false;
    });
</script>

{#if loading}
    <div>Loading modules...</div>
{:else}
    {#each modules as module (module.module_id)}
        <ModuleItem
            {module}
            on:click={() => goto(`/c/${module.thread_id}`)}
        />
    {/each}
{/if}
```

#### 3. Simplify ModuleItem.svelte
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte`
**Changes**: Remove folder logic, just navigate to thread

```svelte
<script lang="ts">
    import { goto } from '$app/navigation';
    import { showSidebar } from '$lib/stores';

    export let module: {
        module_id: string;
        module_name: string;
        thread_id: string;
        status: string;
    };

    function handleClick() {
        // Direct navigation - no folder creation needed
        goto(`/c/${module.thread_id}`);

        // Hide sidebar on mobile
        if (window.innerWidth < 768) {
            showSidebar.set(false);
        }
    }
</script>

<button on:click={handleClick}>
    <!-- Module display with status icon -->
</button>
```

#### 4. Remove Folder Utilities (Optional Cleanup)
**File**: `OpenWebUI/open-webui/src/lib/utils/folders.ts`
**Changes**: Can remove `ensureModuleFolder` and `getMostRecentThreadInFolder` if no longer needed

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`
- [ ] Ralph API returns correct module list

#### Manual Verification:
- [ ] Sidebar shows modules fetched from Ralph
- [ ] Clicking module navigates to correct thread
- [ ] Thread shows first_message content
- [ ] Status icons reflect actual module status
- [ ] Module threads don't appear in Chats section

---

## Testing Strategy

### Unit Tests:
- Dolt CRUD methods for `user_courses` and `user_modules`
- CourseLoader parsing of welcome.toml
- EnrollmentService logic (mocking OpenWebUI client)
- Agent builder from course config
- Tool registry lookup

### Integration Tests:
- Webhook endpoint creates enrollment records
- Chat stream uses correct agent for module threads
- Module API returns correct data

### Manual Testing Steps:
1. Start fresh (clear Dolt, new OpenWebUI user)
2. Sign up new user
3. Verify webhook fires (Ralph logs)
4. Verify module appears in sidebar
5. Click module, verify thread loads with first_message
6. Send a message, verify Ralph uses welcome.toml config
7. Verify memory blocks initialized with templates
8. Verify module status updates to "in_progress"

---

## Migration Notes

**Existing users**: Need a migration script to enroll existing users in the welcome course. Run once after deployment:

```python
async def migrate_existing_users():
    """Enroll all existing users in welcome course."""
    # Get all users from OpenWebUI (or Dolt user_activity table)
    # For each user, call EnrollmentService.enroll_user()
```

**Thread archiving**: Any existing module-related threads should be archived to avoid duplicates in sidebar.

---

## Performance Considerations

- Course configs cached in memory (CourseLoader._cache)
- Module lookup by thread_id uses indexed query
- OpenWebUI API calls happen once per enrollment (not per message)

---

## Open Questions Resolved

1. ~~Course routing mechanism~~ → Thread ID lookup in Dolt
2. ~~First message delivery~~ → Injected when creating thread via OpenWebUI API
3. ~~Phase persistence~~ → In memory blocks (onboarding_progress)
4. ~~TOML location~~ → `config/courses/welcome/course.toml`
5. ~~API access~~ → Admin API key for service-level calls

---

## References

- Research: `thoughts/shared/research/2026-01-30-welcome-course-implementation-gap-analysis.md`
- Welcome TOML: `/Users/ariasulin/Desktop/welcome.toml`
- OpenWebUI API: `/Users/ariasulin/Downloads/Complete API Reference...`
- Legacy loader: `src/youlab_server/curriculum/loader.py`
