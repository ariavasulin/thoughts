# ARI-82 Phase 5: Models and Folders Integration

## Overview

Create course modules as visible OpenWebUI models and background agents as hidden models, with automatic folder organization for chat threads. This builds on the existing background agent folder mechanism in `agents_threads.py`.

## Current State Analysis

### What Already Exists

| Component | Location | What It Does |
|-----------|----------|--------------|
| Background agent folders | `agents_threads.py:168-221` | Creates OpenWebUI folders with `meta.type = "background_agent"`, auto-assigns chats |
| OpenWebUIClient folder methods | `openwebui_client.py:306-374` | `ensure_folder()`, `create_chat(folder_id=...)`, `update_chat_folder()` |
| Letta folder attachment | `agents.py:118-153` | Attaches shared/private Letta folders to agents from course config |
| Course/Module config | `curriculum/schema.py` | Module metadata: `id`, `name`, `description`, `disabled_tools` |

### What's Missing

1. **OpenWebUIClient model methods** - No CRUD for OpenWebUI models
2. **Module → Model sync** - No mechanism to create models from module config
3. **Background Agent → Hidden Model** - Only folders exist, not hidden models
4. **Folder-Model linking** - No `folder_id` stored in model meta
5. **Module chat auto-assignment** - Only background agents have auto-folder assignment

## Desired End State

After implementation:

1. **Each module appears as a model** in OpenWebUI model selector
   - Users select "First Impression" model to start that module's chat
   - Model wraps the YouLab pipe with module-specific system prompt

2. **Background agents are hidden models** in workspace but not in selector
   - Admin can see them in `/workspace/models`
   - Hidden from user-facing model selectors

3. **Chats auto-organize into folders**
   - Module chats → Module folder
   - Background agent chats → Background agent folder

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Course Enrollment                             │
├─────────────────────────────────────────────────────────────────┤
│  For each Module:                                                │
│    1. Create OpenWebUI Folder (name = module.name)              │
│    2. Create OpenWebUI Model:                                   │
│       - base_model_id = "youlab-pipe"                           │
│       - params.system = course.system + module.system           │
│       - meta.type = "module"                                    │
│       - meta.module_id = module.id                              │
│       - meta.folder_id = folder.id                              │
│       - meta.hidden = false                                     │
├─────────────────────────────────────────────────────────────────┤
│  For each Background Task:                                       │
│    1. Create OpenWebUI Folder (if not exists)                   │
│    2. Create OpenWebUI Model:                                   │
│       - meta.type = "background_agent"                          │
│       - meta.hidden = true                                      │
│       - meta.folder_id = folder.id                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Chat Auto-Assignment                          │
├─────────────────────────────────────────────────────────────────┤
│  When user starts chat with Module Model:                        │
│    1. Pipe detects model.meta.type = "module"                   │
│    2. Looks up model.meta.folder_id                             │
│    3. Assigns chat to folder via update_chat_folder()           │
└─────────────────────────────────────────────────────────────────┘
```

## What We're NOT Doing

1. **Not implementing nested folders** - Flat structure (one folder per module)
2. **Not creating models for individual steps** - Only module-level models
3. **Not modifying OpenWebUI frontend** - Use existing model selector and folder UI
4. **Not bidirectional sync** - Models created once on enrollment, not continuously synced

---

## Phase 5.1: OpenWebUIClient Model Methods

### Overview
Add model CRUD methods to the existing OpenWebUIClient.

### Changes Required

#### 1. OpenWebUIClient Model Methods
**File**: `src/youlab_server/server/sync/openwebui_client.py`

Add after line 374 (after `archive_chat`):

```python
# Model management methods

async def list_models(self, user_id: str | None = None) -> list[dict]:
    """List all models, optionally filtered by user."""
    resp = await self.client.get("/api/models/list")
    resp.raise_for_status()
    data = resp.json()
    models = data.get("items", [])
    if user_id:
        models = [m for m in models if m.get("user_id") == user_id]
    return models

async def get_model(self, model_id: str) -> dict | None:
    """Get a model by ID."""
    resp = await self.client.get("/api/models/model", params={"id": model_id})
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
    """Create a new model."""
    resp = await self.client.post(
        "/api/models/create",
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
    """Update an existing model."""
    # First get current model to preserve fields
    current = await self.get_model(model_id)
    if not current:
        raise ValueError(f"Model {model_id} not found")

    resp = await self.client.post(
        "/api/models/model/update",
        json={
            "id": model_id,
            "name": name or current["name"],
            "base_model_id": current.get("base_model_id"),
            "params": params or current.get("params", {}),
            "meta": meta or current.get("meta", {}),
            "access_control": current.get("access_control"),
            "is_active": current.get("is_active", True),
        },
    )
    resp.raise_for_status()
    return resp.json()

async def delete_model(self, model_id: str) -> bool:
    """Delete a model."""
    resp = await self.client.post(
        "/api/models/model/delete",
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
) -> dict:
    """Ensure a model exists, creating or updating as needed."""
    existing = await self.get_model(model_id)
    if existing:
        return await self.update_model(model_id, name, params, meta)
    return await self.create_model(model_id, name, base_model_id, params, meta)
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] Unit test for model CRUD methods passes

#### Manual Verification:
- [ ] Can call `ensure_model()` and see model in OpenWebUI workspace

---

## Phase 5.2: Module Models Service

### Overview
Create a service that syncs course modules to OpenWebUI models on enrollment.

### Changes Required

#### 1. Module Models Service
**File**: `src/youlab_server/server/sync/models.py` (new file)

```python
"""Module-to-Model synchronization service."""

from __future__ import annotations

import structlog
from dataclasses import dataclass

from youlab_server.curriculum import CourseConfig, ModuleConfig
from youlab_server.server.sync.openwebui_client import OpenWebUIClient

log = structlog.get_logger()


@dataclass
class ModuleModelConfig:
    """Configuration for a module model."""

    model_id: str
    name: str
    base_model_id: str
    system_prompt: str
    module_id: str
    folder_id: str | None = None
    description: str = ""


def build_module_model_id(course_id: str, module_id: str, user_id: str) -> str:
    """Build unique model ID for a module.

    Format: {course_id}:{module_id}:{user_id}
    Example: college-essay:01-first-impression:user123
    """
    return f"{course_id}:{module_id}:{user_id}"


def build_module_system_prompt(course: CourseConfig, module: ModuleConfig) -> str:
    """Build system prompt for a module model.

    Combines course-level system prompt with module-specific context.
    """
    base_prompt = course.agent.system

    # Add module context
    module_context = f"""
## Current Module: {module.name}

{module.description}

### Module Objectives:
"""

    # Collect objectives from all steps
    for step in module.steps:
        for obj in step.objectives:
            module_context += f"- {obj}\n"

    # Add disabled tools notice if any
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
    """
    Create OpenWebUI models for all modules in a course.

    Args:
        course: The course configuration.
        user_id: User ID for model ownership and ID generation.
        base_model_id: The base model (YouLab pipe) ID.
        openwebui_client: OpenWebUI API client.

    Returns:
        List of created model IDs.
    """
    created_models = []

    for module in course.loaded_modules:
        model_id = build_module_model_id(course.id, module.id, user_id)

        # Create folder for module chats
        folder_id = await openwebui_client.ensure_folder(
            module.name,
            meta={"type": "module", "module_id": module.id, "course_id": course.id},
        )

        # Build system prompt
        system_prompt = build_module_system_prompt(course, module)

        # Create model
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
                "hidden": False,
            },
        )

        created_models.append(model_id)
        log.info(
            "module_model_created",
            model_id=model_id,
            module=module.name,
            folder_id=folder_id,
        )

    return created_models


async def create_background_agent_models(
    course: CourseConfig,
    user_id: str,
    base_model_id: str,
    openwebui_client: OpenWebUIClient,
) -> list[str]:
    """
    Create hidden OpenWebUI models for background agents.

    Args:
        course: The course configuration.
        user_id: User ID for model ownership.
        base_model_id: The base model (YouLab pipe) ID.
        openwebui_client: OpenWebUI API client.

    Returns:
        List of created model IDs.
    """
    created_models = []

    for i, task in enumerate(course.tasks):
        task_name = getattr(task, "name", f"background-task-{i}")
        model_id = f"{course.id}:bg:{task_name}:{user_id}"

        # Create folder for background agent chats
        folder_id = await openwebui_client.ensure_folder(
            task_name,
            meta={"type": "background_agent", "task_name": task_name},
        )

        # Build system prompt from task config
        system_prompt = task.system or f"Background agent: {task_name}"

        # Create hidden model
        await openwebui_client.ensure_model(
            model_id=model_id,
            name=task_name,
            base_model_id=base_model_id,
            params={"system": system_prompt},
            meta={
                "type": "background_agent",
                "task_name": task_name,
                "course_id": course.id,
                "folder_id": folder_id,
                "hidden": True,  # Hidden from model selector
            },
        )

        created_models.append(model_id)
        log.info(
            "background_model_created",
            model_id=model_id,
            task=task_name,
            folder_id=folder_id,
        )

    return created_models
```

#### 2. Add `name` field to TaskConfig
**File**: `src/youlab_server/curriculum/schema.py`

Add to `TaskConfig` class (around line 169):

```python
class TaskConfig(BaseModel):
    """Background task configuration (v2 format)."""

    name: str = ""  # Add this field

    # ... rest of existing fields
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Unit test for `build_module_system_prompt()` passes
- [ ] Unit test for `build_module_model_id()` passes

#### Manual Verification:
- [ ] After enrollment, module models appear in OpenWebUI model selector

---

## Phase 5.3: Integration with Enrollment Flow

### Overview
Wire the model creation into the agent enrollment flow.

### Changes Required

#### 1. Update AgentManager
**File**: `src/youlab_server/server/agents.py`

Add import at top:
```python
from youlab_server.server.sync.models import (
    create_module_models,
    create_background_agent_models,
)
```

Update `create_agent_from_curriculum()` method (around line 340, after folder attachment):

```python
# After line 364 (after attached_folders logging)

# Create module models in OpenWebUI
module_models: list[str] = []
background_models: list[str] = []

if sync_service and sync_service.openwebui:
    try:
        # Get the YouLab pipe model ID from settings
        from youlab_server.config.settings import settings
        base_model_id = settings.OPENWEBUI_PIPE_MODEL_ID  # Add this setting

        module_models = await create_module_models(
            course=course,
            user_id=user_id,
            base_model_id=base_model_id,
            openwebui_client=sync_service.openwebui,
        )

        background_models = await create_background_agent_models(
            course=course,
            user_id=user_id,
            base_model_id=base_model_id,
            openwebui_client=sync_service.openwebui,
        )

        log.info(
            "enrollment_models_created",
            agent_id=agent_id,
            module_models=len(module_models),
            background_models=len(background_models),
        )
    except Exception as e:
        log.warning("model_creation_failed", error=str(e))
```

#### 2. Add Settings
**File**: `src/youlab_server/config/settings.py`

Add new setting:

```python
OPENWEBUI_PIPE_MODEL_ID: str = Field(
    default="youlab-pipe",
    description="Model ID of the YouLab pipe in OpenWebUI",
)
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Integration test: create agent → models created

#### Manual Verification:
- [ ] New user enrollment creates module models
- [ ] Models appear in OpenWebUI model selector
- [ ] Background agent models are hidden

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the models appear correctly in OpenWebUI before proceeding to the next phase.

---

## Phase 5.4: Chat Auto-Assignment to Folders

### Overview
When a user starts a chat with a module model, automatically assign it to the module's folder.

### Changes Required

#### 1. Update Letta Pipe
**File**: `src/youlab_server/pipelines/letta_pipe.py`

Add method to handle folder assignment:

```python
def _assign_chat_to_folder(self, chat_id: str, model_id: str) -> bool:
    """
    Assign chat to folder based on model metadata.

    Args:
        chat_id: The chat ID to assign.
        model_id: The model ID to look up folder from.

    Returns:
        True if assigned, False otherwise.
    """
    if not chat_id or chat_id.startswith("local:"):
        return False

    try:
        from open_webui.models.models import Models
        from open_webui.models.chats import Chats

        # Get model metadata
        model = Models.get_model_by_id(model_id)
        if not model:
            return False

        # Check if model has folder assignment
        meta = model.meta.model_dump() if model.meta else {}
        folder_id = meta.get("folder_id")

        if not folder_id:
            return False

        # Check model type (only assign for module/background_agent types)
        model_type = meta.get("type")
        if model_type not in ("module", "background_agent"):
            return False

        # Assign chat to folder
        Chats.update_chat_folder_id_by_id_and_user_id(chat_id, folder_id)

        if self.valves.ENABLE_LOGGING:
            print(f"YouLab: assigned chat {chat_id} to folder {folder_id}")

        return True

    except ImportError:
        return False
    except Exception as e:
        if self.valves.ENABLE_LOGGING:
            print(f"Failed to assign chat to folder: {e}")
        return False
```

Update `pipe()` method to call folder assignment (after line 171):

```python
# After: if self.valves.ENABLE_LOGGING:
#            print(f"YouLab: user={user_id}, chat={chat_id}")

# Assign chat to folder based on model
model_id = body.get("model", "")
if chat_id and model_id:
    self._assign_chat_to_folder(chat_id, model_id)
```

### Success Criteria

#### Automated Verification:
- [ ] `make check-agent` passes
- [ ] Pipe loads without errors

#### Manual Verification:
- [ ] Start chat with module model → chat appears in module folder
- [ ] Start chat with base YouLab model → chat NOT auto-assigned (stays in root)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that chat auto-assignment works correctly before proceeding.

---

## Phase 5.5: Cleanup and Edge Cases

### Overview
Handle model cleanup on user deletion and edge cases.

### Changes Required

#### 1. Add Model Cleanup
**File**: `src/youlab_server/server/users.py`

Add cleanup when user is deleted:

```python
async def cleanup_user_models(user_id: str, openwebui_client: OpenWebUIClient) -> int:
    """Delete all models owned by a user.

    Returns:
        Number of models deleted.
    """
    models = await openwebui_client.list_models(user_id=user_id)
    deleted = 0

    for model in models:
        model_id = model.get("id")
        if model_id and model.get("user_id") == user_id:
            await openwebui_client.delete_model(model_id)
            deleted += 1

    return deleted
```

#### 2. Handle Model Conflicts
**File**: `src/youlab_server/server/sync/models.py`

The `ensure_model()` method already handles updates, but add conflict logging:

```python
# In create_module_models(), after ensure_model call:
log.info(
    "module_model_synced",
    model_id=model_id,
    module=module.name,
    action="created_or_updated",
)
```

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
| `build_module_system_prompt()` | `tests/test_server/test_models_sync.py` | Prompt generation |
| `build_module_model_id()` | `tests/test_server/test_models_sync.py` | ID format |

### Integration Tests

| Test | Description |
|------|-------------|
| `test_enrollment_creates_models` | Create agent → verify models in OpenWebUI |
| `test_chat_folder_assignment` | Send message with module model → chat in folder |
| `test_model_update_on_reenroll` | Re-enroll → models updated, not duplicated |

### Manual Testing Checklist

- [ ] Fresh user enrollment: all module models appear
- [ ] Model selector shows module names
- [ ] Background agent models NOT in selector
- [ ] Start chat with "First Impression" → chat in "First Impression" folder
- [ ] Start chat with base "YouLab Tutor" → chat NOT auto-assigned
- [ ] Admin can see hidden background agent models in workspace

---

## Performance Considerations

1. **Enrollment Time**: Model creation adds ~100-200ms per module (acceptable)
2. **Model Lookup**: Pipe does one model lookup per chat - cached by OpenWebUI
3. **Folder Assignment**: Single DB update per new chat

---

## Migration Notes

### Existing Users

Existing users enrolled before Phase 5 will not have module models. Options:
1. **Lazy creation**: Create models on next chat (adds latency)
2. **Migration script**: Batch create models for existing users
3. **Manual re-enrollment**: User deletes agent and re-enrolls

**Recommendation**: Implement lazy creation in pipe as fallback, provide migration script.

---

## Open Questions Resolved

| Question | Decision |
|----------|----------|
| Module system prompts | Module context appended to course prompt |
| Flat vs nested folders | Flat - one folder per module |
| Auto-assignment mechanism | Pipe middleware (after chat creation) |
| Folder naming | Module display name (e.g., "First Impression") |
| Background folder visibility | Visible but can be collapsed by user |

---

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Research: `thoughts/shared/research/2026-01-13-ARI-82-openwebui-models-chat-folders.md`
- Existing folder logic: `src/youlab_server/server/agents_threads.py:168-221`
- OpenWebUI models schema: `OpenWebUI/open-webui/backend/open_webui/models/models.py:53-103`
- Linear: [ARI-82](https://linear.app/ariav/issue/ARI-82)
