# ARI-82 Phase 5: Models and Folders Integration

## Overview

Create OpenWebUI models for both **modules** (visible) and **background agents** (hidden), with automatic folder organization for chat threads.

- **Module models**: Visible in model dropdown and modules tab. Users select to start/continue module work.
- **Background agent models**: Hidden from new chat picker, but threads are viewable and chattable after runs complete.

**Architecture principle**: Keep the pipe simple. The HTTP service handles model creation, folder management, and chat organization.

## Current State Analysis

### What Already Exists

| Component | Location | What It Does |
|-----------|----------|--------------|
| Folder + chat creation | `agents_threads.py:168-221` | `create_agent_thread()` - creates folder, archives old chats, creates new chat in folder |
| OpenWebUIClient folder/chat methods | `openwebui_client.py:306-374` | `ensure_folder()`, `create_chat(folder_id=...)`, `update_chat_folder()` |
| Letta folder attachment | `agents.py:118-153` | Attaches shared/private Letta folders to agents from course config |
| Course/Module config | `curriculum/schema.py` | Module metadata: `id`, `name`, `description`, `disabled_tools` |
| Background agent config | `curriculum/schema.py` | `BackgroundAgentConfig` with `id`, `name`, `schedule`, etc. |

### What's Missing

1. **OpenWebUIClient model methods** - No CRUD for OpenWebUI models
2. **Module model creation** - No mechanism to create visible models from module config
3. **Background agent model creation** - No mechanism to create hidden models for background agents
4. **Folder-Model linking** - No `folder_id` stored in model meta for chat auto-assignment
5. **Chat folder assignment on first message** - HTTP service doesn't assign folder when user starts chat with module model

## Desired End State

After implementation:

### Module Models (Visible)
- Each module appears in OpenWebUI model selector and modules sidebar tab
- Users select "First Impression" → start/continue chat in that module's folder
- Can return to previous module threads

### Background Agent Models (Hidden)
- NOT visible in new chat model picker
- Threads visible after agent runs complete (tool calls, chain of thought)
- Users can open existing threads and chat directly with the agent
- Changes proposed by agents are approved via profile/notes pages

### Folder Organization
- Each module gets a folder (e.g., "First Impression")
- Each background agent gets a folder (e.g., "Memory Enricher")
- Chats auto-organize into appropriate folders

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Course Enrollment                             │
├─────────────────────────────────────────────────────────────────┤
│  For each Module:                                                │
│    1. Create OpenWebUI Folder (name = module.name)              │
│    2. Create OpenWebUI Model (VISIBLE):                         │
│       - base_model_id = "youlab-pipe"                           │
│       - meta.type = "module"                                    │
│       - meta.folder_id = folder.id                              │
│       - meta.hidden = false                                     │
│                                                                  │
│  For each Background Agent:                                      │
│    1. Create OpenWebUI Folder (name = agent.name)               │
│    2. Create OpenWebUI Model (HIDDEN):                          │
│       - base_model_id = "youlab-pipe"                           │
│       - meta.type = "background_agent"                          │
│       - meta.folder_id = folder.id                              │
│       - meta.hidden = true                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Chat Folder Assignment (HTTP Service)               │
├─────────────────────────────────────────────────────────────────┤
│  On first message to a module/agent model:                       │
│    1. HTTP service receives chat request with model_id          │
│    2. Looks up model metadata → gets folder_id                  │
│    3. Assigns chat to folder via update_chat_folder()           │
│    4. Subsequent messages skip assignment (already done)        │
│                                                                  │
│  Pipe remains simple - just forwards to HTTP service            │
└─────────────────────────────────────────────────────────────────┘
```

## What We're NOT Doing

1. **Not implementing nested folders** - Flat structure (one folder per module/agent)
2. **Not creating models for individual steps** - Only module-level models
3. **Not modifying OpenWebUI frontend** - Use existing model selector and folder UI
4. **Not bidirectional sync** - Models created once on enrollment, not continuously synced
5. **Not handling folder assignment in the pipe** - HTTP service handles all folder logic

---

## Phase 5.1: OpenWebUIClient Model Methods

### Overview
Add model CRUD methods to the existing OpenWebUIClient.

**API Note**: OpenWebUI uses `/api/v1/models/*` endpoints (not `/api/models/*`).

### Changes Required

#### 1. OpenWebUIClient Model Methods
**File**: `src/youlab_server/server/sync/openwebui_client.py`

Add after line 374 (after `archive_chat`):

```python
# Model management methods

async def list_models(self) -> list[dict]:
    """List all models visible to the authenticated user."""
    resp = await self.client.get("/api/v1/models/list")
    resp.raise_for_status()
    data = resp.json()
    return data.get("items", [])

async def get_model(self, model_id: str) -> dict | None:
    """Get a model by ID."""
    resp = await self.client.get("/api/v1/models/model", params={"id": model_id})
    if resp.status_code == 404:
        return None
    resp.raise_for_status()
    return resp.json()

async def create_model(
    self,
    model_id: str,
    name: str,
    base_model_id: str,
    params: dict | None = None,
    meta: dict | None = None,
    access_control: dict | None = None,
) -> dict:
    """
    Create a new model.

    Args:
        model_id: Unique model ID (max 256 chars).
        name: Display name.
        base_model_id: The underlying model/pipe ID (e.g., "youlab-pipe").
        params: Model params (system prompt, etc).
        meta: Metadata (type, folder_id, hidden, etc).
        access_control: None=public, {}=owner-only, or custom grants.
    """
    resp = await self.client.post(
        "/api/v1/models/create",
        json={
            "id": model_id,
            "name": name,
            "base_model_id": base_model_id,
            "params": params or {},
            "meta": meta or {},
            "access_control": access_control,
            "is_active": True,
        },
    )
    resp.raise_for_status()
    return resp.json()

async def update_model(
    self,
    model_id: str,
    name: str | None = None,
    params: dict | None = None,
    meta: dict | None = None,
) -> dict:
    """Update an existing model (preserves unspecified fields)."""
    current = await self.get_model(model_id)
    if not current:
        raise ValueError(f"Model {model_id} not found")

    resp = await self.client.post(
        "/api/v1/models/model/update",
        json={
            "id": model_id,
            "name": name or current["name"],
            "base_model_id": current.get("base_model_id"),
            "params": params if params is not None else current.get("params", {}),
            "meta": meta if meta is not None else current.get("meta", {}),
            "access_control": current.get("access_control"),
            "is_active": current.get("is_active", True),
        },
    )
    resp.raise_for_status()
    return resp.json()

async def delete_model(self, model_id: str) -> bool:
    """Delete a model."""
    resp = await self.client.post(
        "/api/v1/models/model/delete",
        json={"id": model_id},
    )
    return resp.status_code == 200

async def ensure_model(
    self,
    model_id: str,
    name: str,
    base_model_id: str,
    params: dict | None = None,
    meta: dict | None = None,
    access_control: dict | None = None,
) -> dict:
    """Ensure a model exists, creating or updating as needed."""
    existing = await self.get_model(model_id)
    if existing:
        return await self.update_model(model_id, name, params, meta)
    return await self.create_model(
        model_id, name, base_model_id, params, meta, access_control
    )
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] Unit test for model CRUD methods passes

#### Manual Verification:
- [ ] Can call `ensure_model()` and see model in OpenWebUI workspace

---

## Phase 5.2: Models Service (Modules + Background Agents)

### Overview
Create a service that syncs course modules AND background agents to OpenWebUI models on enrollment.

- **Module models**: Visible in model picker (`hidden: false`)
- **Background agent models**: Hidden from new chat picker (`hidden: true`), but threads remain accessible

### Changes Required

#### 1. Models Service
**File**: `src/youlab_server/server/sync/models.py` (new file)

```python
"""Course models synchronization service."""

from __future__ import annotations

import structlog

from youlab_server.curriculum import CourseConfig, ModuleConfig, BackgroundAgentConfig
from youlab_server.server.sync.openwebui_client import OpenWebUIClient

log = structlog.get_logger()


def build_model_id(course_id: str, item_id: str, user_id: str) -> str:
    """Build unique model ID.

    Format: {course_id}:{item_id}:{user_id}
    Example: college-essay:01-first-impression:user123
    """
    return f"{course_id}:{item_id}:{user_id}"


def build_module_system_prompt(course: CourseConfig, module: ModuleConfig) -> str:
    """Build system prompt for a module model."""
    base_prompt = course.agent.system

    module_context = f"""
## Current Module: {module.name}

{module.description}

### Module Objectives:
"""
    for step in module.steps:
        for obj in step.objectives:
            module_context += f"- {obj}\n"

    if module.disabled_tools:
        module_context += f"\n### Disabled Tools for This Module:\n"
        module_context += f"Do not use: {', '.join(module.disabled_tools)}\n"

    return base_prompt + "\n" + module_context


async def create_module_models(
    course: CourseConfig,
    user_id: str,
    base_model_id: str,
    openwebui_client: OpenWebUIClient,
) -> list[str]:
    """Create VISIBLE models for all modules in a course."""
    created_models = []

    for module in course.loaded_modules:
        model_id = build_model_id(course.id, module.id, user_id)

        folder_id = await openwebui_client.ensure_folder(
            module.name,
            meta={"type": "module", "module_id": module.id, "course_id": course.id},
        )

        system_prompt = build_module_system_prompt(course, module)

        await openwebui_client.ensure_model(
            model_id=model_id,
            name=module.name,
            base_model_id=base_model_id,
            params={"system": system_prompt},
            meta={
                "type": "module",
                "module_id": module.id,
                "course_id": course.id,
                "folder_id": folder_id,
                "description": module.description,
                "hidden": False,  # Visible in model picker
            },
            access_control={},  # Private to owner
        )

        created_models.append(model_id)
        log.info("module_model_created", model_id=model_id, folder_id=folder_id)

    return created_models


async def create_background_agent_models(
    course: CourseConfig,
    user_id: str,
    base_model_id: str,
    openwebui_client: OpenWebUIClient,
) -> list[str]:
    """Create HIDDEN models for all background agents in a course."""
    created_models = []

    for agent_id, agent_config in course.background.items():
        model_id = build_model_id(course.id, agent_id, user_id)

        folder_id = await openwebui_client.ensure_folder(
            agent_config.name,
            meta={"type": "background_agent", "agent_id": agent_id, "course_id": course.id},
        )

        # Background agents use their own system prompt from config
        system_prompt = agent_config.system or course.agent.system

        await openwebui_client.ensure_model(
            model_id=model_id,
            name=agent_config.name,
            base_model_id=base_model_id,
            params={"system": system_prompt},
            meta={
                "type": "background_agent",
                "agent_id": agent_id,
                "course_id": course.id,
                "folder_id": folder_id,
                "description": agent_config.description or "",
                "hidden": True,  # Hidden from new chat picker
            },
            access_control={},  # Private to owner
        )

        created_models.append(model_id)
        log.info("background_agent_model_created", model_id=model_id, folder_id=folder_id)

    return created_models


async def create_course_models(
    course: CourseConfig,
    user_id: str,
    base_model_id: str,
    openwebui_client: OpenWebUIClient,
) -> dict[str, list[str]]:
    """Create all models for a course (modules + background agents)."""
    module_models = await create_module_models(
        course, user_id, base_model_id, openwebui_client
    )
    agent_models = await create_background_agent_models(
        course, user_id, base_model_id, openwebui_client
    )

    return {
        "modules": module_models,
        "background_agents": agent_models,
    }
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Unit test for `build_model_id()` passes
- [ ] Unit test for `build_module_system_prompt()` passes
- [ ] Unit test for `create_course_models()` passes (mocked OpenWebUI client)

#### Manual Verification:
- [ ] After enrollment, module models appear in OpenWebUI model selector
- [ ] Background agent models are hidden from new chat picker

---

## Phase 5.3: Integration with Enrollment Flow

### Overview
Wire the model creation into the agent enrollment flow.

### Changes Required

#### 1. Update AgentManager
**File**: `src/youlab_server/server/agents.py`

Add import at top:
```python
from youlab_server.server.sync.models import create_course_models
```

Update `create_agent_from_curriculum()` method (around line 340, after folder attachment):

```python
# After attached_folders logging

# Create all models (modules + background agents) in OpenWebUI
if sync_service and sync_service.openwebui:
    try:
        from youlab_server.config.settings import service_settings
        base_model_id = service_settings.openwebui_pipe_model_id

        created_models = await create_course_models(
            course=course,
            user_id=user_id,
            base_model_id=base_model_id,
            openwebui_client=sync_service.openwebui,
        )

        log.info(
            "enrollment_models_created",
            agent_id=agent_id,
            module_models=len(created_models["modules"]),
            background_agent_models=len(created_models["background_agents"]),
        )
    except Exception as e:
        log.warning("model_creation_failed", error=str(e))
```

#### 2. Add Settings
**File**: `src/youlab_server/config/settings.py`

Add to `ServiceSettings`:

```python
openwebui_pipe_model_id: str = Field(
    default="youlab-pipe",
    description="Model ID of the YouLab pipe in OpenWebUI",
)
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Integration test: create agent → both module and background agent models created

#### Manual Verification:
- [ ] New user enrollment creates module models (visible)
- [ ] New user enrollment creates background agent models (hidden)
- [ ] Models appear correctly in OpenWebUI

**Implementation Note**: Pause for manual verification before proceeding.

---

## Phase 5.4: Chat Auto-Assignment to Folders (HTTP Service)

### Overview
When a user starts a chat with a module/agent model, the HTTP service assigns it to the appropriate folder. **The pipe remains simple** - it just forwards model_id to the HTTP service.

### Architecture

```
User selects module model → OpenWebUI creates chat → User sends message
    → Pipe forwards to HTTP service (includes model_id, chat_id)
    → HTTP service checks model metadata
    → HTTP service assigns chat to folder (if not already assigned)
```

### Changes Required

#### 1. Add Folder Assignment to HTTP Service
**File**: `src/youlab_server/server/chat.py` (or wherever chat handling lives)

```python
async def assign_chat_to_model_folder(
    chat_id: str,
    model_id: str,
    openwebui_client: OpenWebUIClient,
) -> bool:
    """
    Assign a chat to its model's folder (if applicable).

    Called on each message. Checks model metadata for folder_id and assigns
    the chat if it has a type of "module" or "background_agent".

    Args:
        chat_id: The OpenWebUI chat ID.
        model_id: The model being used for this chat.
        openwebui_client: OpenWebUI API client.

    Returns:
        True if assigned (or already assigned), False if not applicable.
    """
    if not chat_id or not model_id:
        return False

    # Get model metadata
    model = await openwebui_client.get_model(model_id)
    if not model:
        return False

    meta = model.get("meta", {})
    model_type = meta.get("type")
    folder_id = meta.get("folder_id")

    # Only assign for module/background_agent types
    if model_type not in ("module", "background_agent"):
        return False

    if not folder_id:
        return False

    # Assign chat to folder
    await openwebui_client.update_chat_folder(chat_id, folder_id)
    return True
```

#### 2. Call from Chat Endpoint
**File**: `src/youlab_server/server/main.py` (chat endpoint)

In the chat handling flow, after receiving a request:

```python
# Early in chat handling, assign folder if needed
if openwebui_client and chat_id and model_id:
    try:
        await assign_chat_to_model_folder(chat_id, model_id, openwebui_client)
    except Exception as e:
        log.warning("folder_assignment_failed", chat_id=chat_id, error=str(e))
```

#### 3. Pipe Changes (Minimal)
**File**: `src/youlab_server/pipelines/letta_pipe.py`

Ensure the pipe passes `model_id` to the HTTP service. The pipe should already be doing this, but verify:

```python
# In pipe(), ensure model_id is included in the request to HTTP service
model_id = body.get("model", "")
# ... pass model_id to HTTP service
```

**No folder assignment logic in the pipe.** The pipe just forwards the request.

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Unit test for `assign_chat_to_model_folder()` passes

#### Manual Verification:
- [ ] Start chat with module model → chat appears in module folder
- [ ] Start chat with base YouLab model → chat NOT auto-assigned (stays in root)
- [ ] Open existing background agent thread → can continue chatting

**Implementation Note**: Pause for manual verification before proceeding.

---

## Phase 5.5: Cleanup and Edge Cases

### Overview
Handle model cleanup on user deletion and edge cases.

### Changes Required

#### 1. Add Model Cleanup
**File**: `src/youlab_server/server/users.py`

Add cleanup when user is deleted. Since model IDs include user_id, we can filter by prefix:

```python
async def cleanup_user_models(
    user_id: str,
    course_id: str,
    openwebui_client: OpenWebUIClient,
) -> int:
    """Delete all course models for a user.

    Model IDs follow format: {course_id}:{item_id}:{user_id}
    So we can identify user's models by checking the suffix.

    Returns:
        Number of models deleted.
    """
    models = await openwebui_client.list_models()
    deleted = 0

    user_suffix = f":{user_id}"
    for model in models:
        model_id = model.get("id", "")
        # Check if this is a course model belonging to this user
        if model_id.startswith(f"{course_id}:") and model_id.endswith(user_suffix):
            await openwebui_client.delete_model(model_id)
            deleted += 1

    return deleted
```

#### 2. Handle Re-enrollment
**File**: `src/youlab_server/server/sync/models.py`

The `ensure_model()` method already handles updates via create-or-update pattern. No additional changes needed.

### Success Criteria

#### Automated Verification:
- [ ] `make verify-agent` passes (full lint + typecheck + tests)
- [ ] All new code has docstrings

#### Manual Verification:
- [ ] Re-enrollment updates existing models (doesn't create duplicates)
- [ ] User deletion cleans up models

---

## Testing Strategy

### Unit Tests

| Test | File | Coverage |
|------|------|----------|
| Model CRUD methods | `tests/test_server/test_openwebui_client.py` | Mock HTTP responses |
| `build_model_id()` | `tests/test_server/test_models_sync.py` | ID format |
| `build_module_system_prompt()` | `tests/test_server/test_models_sync.py` | Prompt generation |
| `create_course_models()` | `tests/test_server/test_models_sync.py` | Model creation (mocked) |
| `assign_chat_to_model_folder()` | `tests/test_server/test_chat.py` | Folder assignment |

### Integration Tests

| Test | Description |
|------|-------------|
| `test_enrollment_creates_models` | Create agent → verify module + background agent models |
| `test_chat_folder_assignment` | Send message with module model → chat in folder |
| `test_model_update_on_reenroll` | Re-enroll → models updated, not duplicated |
| `test_background_agent_thread_chat` | Can chat in existing background agent thread |

### Manual Testing Checklist

- [ ] Fresh user enrollment: module models appear in selector
- [ ] Fresh user enrollment: background agent models are hidden
- [ ] Start chat with "First Impression" → chat in "First Impression" folder
- [ ] Start chat with base "YouLab Tutor" → chat NOT auto-assigned (stays in root)
- [ ] Open background agent thread → can continue chatting
- [ ] Background agent not visible when creating new chat

---

## Performance Considerations

1. **Enrollment Time**: Model creation adds ~100-200ms per module/agent (acceptable)
2. **Model Lookup**: HTTP service does one model lookup per message - can cache if needed
3. **Folder Assignment**: Single API call per new chat (idempotent, so repeated calls are fine)

---

## Migration Notes

### Existing Users

Existing users enrolled before Phase 5 will not have module models. Options:
1. **Migration script**: Batch create models for existing users (recommended)
2. **Manual re-enrollment**: User deletes agent and re-enrolls
3. **Lazy creation in HTTP service**: Create models on first chat if missing

**Recommendation**: Provide migration script. Lazy creation adds complexity and latency.

---

## Open Questions Resolved

| Question | Decision |
|----------|----------|
| Module system prompts | Module context appended to course prompt |
| Flat vs nested folders | Flat - one folder per module/agent |
| Auto-assignment mechanism | HTTP service (not pipe) handles folder assignment |
| Folder naming | Display name (e.g., "First Impression", "Memory Enricher") |
| Background agent models | **Needed** (hidden) - users can chat in existing threads |
| Model visibility | Modules: visible (`hidden: false`), Background agents: hidden (`hidden: true`) |
| Pipe complexity | Minimal - pipe just forwards to HTTP service |

---

## Access Control Notes

OpenWebUI models have an `access_control` field:
- `None` = Public (all users see it) - **not recommended for per-user models**
- `{}` = Private (only owner sees it) - **use this for course models**
- Custom grants possible but not needed for this use case

Since models are created per-user with unique IDs (`{course}:{module}:{user}`), setting `access_control={}` ensures each user only sees their own models.

---

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Research: `thoughts/shared/research/2026-01-13-ARI-82-openwebui-models-chat-folders.md`
- Existing folder logic: `src/youlab_server/server/agents_threads.py:168-221`
- OpenWebUI models API: `OpenWebUI/open-webui/backend/open_webui/routers/models.py`
- OpenWebUI models schema: `OpenWebUI/open-webui/backend/open_webui/models/models.py:53-103`
- Linear: [ARI-82](https://linear.app/ariav/issue/ARI-82)
