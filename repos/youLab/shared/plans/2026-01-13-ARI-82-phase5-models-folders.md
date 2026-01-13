# ARI-82 Phase 5: Models and Folders Integration

## Overview

Integrate YouLab modules and background agents with OpenWebUI's model and folder systems. Modules become visible models that users can select, background agents become hidden models for system use, and chats are automatically organized into folders by model.

**Parent Plan**: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md` (Phase 4)
**Research**: `thoughts/shared/research/2026-01-13-ARI-82-openwebui-models-chat-folders.md`

## Current State Analysis

### What Exists

| Component | Implementation | Location |
|-----------|----------------|----------|
| OpenWebUI Models API | Full CRUD with flexible meta field | `OpenWebUI/open-webui/backend/open_webui/routers/models.py` |
| Model hidden flag | `meta.hidden=true` filtered from selectors | `ModelSelector/Selector.svelte:161` |
| Background agent type | `meta.type="background_agent"` excluded | `ModelSelector/Selector.svelte:454` |
| Folders API | Full CRUD with arbitrary meta | `OpenWebUI/open-webui/backend/open_webui/routers/folders.py` |
| OpenWebUIClient | Notes, Knowledge, Folders, Chats | `src/youlab_server/server/sync/openwebui_client.py` |
| Agent creation | From curriculum with folder attachment | `src/youlab_server/server/agents.py:220-367` |

### Key Research Findings

1. **Model Schema Flexible**: `meta` field uses `ConfigDict(extra="allow")` - accepts any JSON
2. **Base Model Proxying**: `base_model_id` causes requests to proxy through base model
3. **Hidden Models Work**: Frontend filters `meta.hidden=true` from all user-facing selectors
4. **No Auto-Assignment**: Chats created with `folder_id=null` by default - explicit assignment required
5. **Folder Meta Flexible**: Can store `model_id` to link folder to model

## Desired End State

After implementation:

1. **YouLab Base Model** exists as a pipe model in OpenWebUI
2. **Modules** appear as selectable models with `base_model_id=youlab`
3. **Background Agents** exist as hidden models (`meta.hidden=true`, `meta.type=background_agent`)
4. **Folders** auto-created per model with `meta.model_id` linking
5. **New Chats** auto-assigned to model's folder via pipe middleware
6. **Hot Reload** creates new models when modules added to config

### Verification

```bash
# YouLab base model exists
curl -s $OPENWEBUI_URL/api/models | jq '.[] | select(.id == "youlab")'
# Expected: pipe model with is_active=true

# Module models visible
curl -s $OPENWEBUI_URL/api/models | jq '[.[] | select(.base_model_id == "youlab" and .meta.type == "module")]'
# Expected: array of visible module models

# Background agents hidden
curl -s $OPENWEBUI_URL/api/models | jq '[.[] | select(.meta.hidden == true and .meta.type == "background_agent")]'
# Expected: array of hidden background agent models

# Folders have model_id
curl -s $OPENWEBUI_URL/api/folders | jq '.[] | select(.meta.model_id != null)'
# Expected: folders linked to models

# Chat auto-assigned to folder (via pipe)
# Create chat with module model, verify folder_id set
```

## What We're NOT Doing

1. **Not modifying OpenWebUI backend** - Use existing APIs, no schema changes
2. **Not implementing real-time model sync** - Sync on startup and enrollment
3. **Not creating models for steps** - Only module-level models
4. **Not auto-archiving old model folders** - Manual cleanup
5. **Not implementing model deletion cascade** - Folders persist if model removed

## Implementation Approach

Bottom-up approach:
1. Add model management methods to OpenWebUIClient
2. Create YouLab base model on server startup
3. Create module/agent models during course enrollment
4. Create folders linked to models
5. Add auto-assignment middleware to pipe

---

## Phase 5.1: OpenWebUIClient Model Methods

### Overview

Extend `OpenWebUIClient` with model CRUD operations for creating base models, module models, and background agent models.

### Changes Required:

#### 1. Add model methods to OpenWebUIClient

**File**: `src/youlab_server/server/sync/openwebui_client.py`

Add after the folder methods (after line 374):

```python
# Model management methods for YouLab integration

async def list_models(self) -> list[dict]:
    """List all models."""
    resp = await self.client.get("/api/models")
    resp.raise_for_status()
    return resp.json()

async def get_model(self, model_id: str) -> dict | None:
    """Get a model by ID."""
    resp = await self.client.get(f"/api/models/{model_id}")
    if resp.status_code == 404:
        return None
    resp.raise_for_status()
    return resp.json()

async def create_model(
    self,
    model_id: str,
    name: str,
    base_model_id: str | None = None,
    system_prompt: str | None = None,
    meta: dict | None = None,
    is_active: bool = True,
) -> dict:
    """Create a new model."""
    payload = {
        "id": model_id,
        "name": name,
        "base_model_id": base_model_id,
        "params": {"system": system_prompt} if system_prompt else {},
        "meta": meta or {},
        "is_active": is_active,
    }
    resp = await self.client.post("/api/models/create", json=payload)
    resp.raise_for_status()
    return resp.json()

async def update_model(
    self,
    model_id: str,
    name: str | None = None,
    base_model_id: str | None = None,
    system_prompt: str | None = None,
    meta: dict | None = None,
    is_active: bool | None = None,
) -> dict:
    """Update an existing model."""
    # Get current model state
    current = await self.get_model(model_id)
    if current is None:
        raise ValueError(f"Model not found: {model_id}")

    payload = {
        "id": model_id,
        "name": name if name is not None else current.get("name", ""),
        "base_model_id": base_model_id if base_model_id is not None else current.get("base_model_id"),
        "params": current.get("params", {}),
        "meta": {**current.get("meta", {}), **(meta or {})},
        "is_active": is_active if is_active is not None else current.get("is_active", True),
    }
    if system_prompt is not None:
        payload["params"]["system"] = system_prompt

    resp = await self.client.post("/api/models/model/update", json=payload)
    resp.raise_for_status()
    return resp.json()

async def ensure_base_model(
    self,
    model_id: str,
    name: str,
    description: str | None = None,
) -> dict:
    """Ensure a base model exists, creating if needed."""
    existing = await self.get_model(model_id)
    if existing:
        return existing

    return await self.create_model(
        model_id=model_id,
        name=name,
        meta={
            "description": description,
            "hidden": False,
        },
    )

async def ensure_module_model(
    self,
    module_id: str,
    module_name: str,
    base_model_id: str,
    system_prompt: str | None = None,
    description: str | None = None,
) -> dict:
    """Ensure a module model exists, creating if needed."""
    model_id = f"module-{module_id}"
    existing = await self.get_model(model_id)
    if existing:
        return existing

    return await self.create_model(
        model_id=model_id,
        name=module_name,
        base_model_id=base_model_id,
        system_prompt=system_prompt,
        meta={
            "description": description or f"YouLab module: {module_name}",
            "hidden": False,
            "type": "module",
            "youlab_module_id": module_id,
        },
    )

async def ensure_background_agent_model(
    self,
    agent_id: str,
    agent_name: str,
    base_model_id: str,
    system_prompt: str | None = None,
    description: str | None = None,
) -> dict:
    """Ensure a background agent model exists as hidden."""
    model_id = f"agent-{agent_id}"
    existing = await self.get_model(model_id)
    if existing:
        return existing

    return await self.create_model(
        model_id=model_id,
        name=agent_name,
        base_model_id=base_model_id,
        system_prompt=system_prompt,
        meta={
            "description": description or f"Background agent: {agent_name}",
            "hidden": True,
            "type": "background_agent",
            "youlab_agent_id": agent_id,
        },
    )

async def ensure_model_folder(
    self,
    model_id: str,
    folder_name: str,
) -> str:
    """Ensure a folder exists for a model, return folder_id."""
    # Check existing folders for one linked to this model
    folders = await self.list_folders()

    for folder in folders:
        if folder.get("meta", {}).get("model_id") == model_id:
            return folder["id"]

    # Create new folder linked to model
    new_folder = await self.create_folder(
        name=folder_name,
        meta={"model_id": model_id},
    )
    return new_folder["id"]
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Model creation works:
  ```python
  # In test
  client = OpenWebUIClient(url, key)
  model = await client.create_model("test-model", "Test", meta={"hidden": True})
  assert model["id"] == "test-model"
  assert model["meta"]["hidden"] is True
  ```

#### Manual Verification:
- [ ] Create model via client, verify in OpenWebUI admin UI
- [ ] Create hidden model, verify not visible in chat selector

---

## Phase 5.2: YouLab Base Model Creation

### Overview

Create the YouLab base model on server startup. This pipe model serves as the `base_model_id` for all module and agent models.

### Changes Required:

#### 1. Add base model initialization to server startup

**File**: `src/youlab_server/server/main.py`

Add after curriculum initialization (after line 99), before sync service:

```python
# Ensure YouLab base model exists in OpenWebUI
if settings.openwebui_api_url and settings.openwebui_api_key:
    from youlab_server.server.sync.openwebui_client import OpenWebUIClient

    async with httpx.AsyncClient() as http_client:
        openwebui = OpenWebUIClient(
            settings.openwebui_api_url,
            settings.openwebui_api_key,
        )
        try:
            await openwebui.ensure_base_model(
                model_id="youlab",
                name="YouLab",
                description="YouLab tutoring platform - select a specific module for focused coaching",
            )
            log.info("youlab_base_model_ensured")
        except Exception as e:
            log.warning("youlab_base_model_failed", error=str(e))
        finally:
            await openwebui.close()
```

#### 2. Add settings for OpenWebUI integration

**File**: `src/youlab_server/config/settings.py`

Add new settings (if not present):

```python
# OpenWebUI integration
openwebui_api_url: str = Field(
    default="http://localhost:3000",
    description="OpenWebUI API base URL",
)
openwebui_api_key: str | None = Field(
    default=None,
    description="OpenWebUI API key for authentication",
)
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Server starts without error: `uv run youlab-server`
- [ ] Base model created on startup:
  ```bash
  curl -s $OPENWEBUI_URL/api/models/youlab | jq '.id'
  # Expected: "youlab"
  ```

#### Manual Verification:
- [ ] Start server, verify "youlab" model appears in OpenWebUI
- [ ] Model visible in admin Models page
- [ ] Model NOT visible in chat model selector (no base_model_id)

---

## Phase 5.3: Module and Background Agent Model Creation

### Overview

Create OpenWebUI models for modules and background agents during course enrollment. Modules are visible, background agents are hidden.

### Changes Required:

#### 1. Add model sync to agent creation

**File**: `src/youlab_server/server/agents.py`

Add new method after `_attach_folders` (around line 155):

```python
async def _sync_models_to_openwebui(
    self,
    course: CourseConfig,
    user_id: str,
    openwebui: OpenWebUIClient,
) -> dict[str, str]:
    """
    Sync course modules and background agents as OpenWebUI models.

    Returns dict mapping model_id -> folder_id.
    """
    model_folders: dict[str, str] = {}

    # Create module models (visible)
    for module in course.loaded_modules:
        try:
            model = await openwebui.ensure_module_model(
                module_id=module.id,
                module_name=module.name,
                base_model_id="youlab",
                system_prompt=course.agent.system,  # Use course system prompt
                description=f"Module: {module.name}",
            )
            model_id = model["id"]

            # Create folder for this module
            folder_id = await openwebui.ensure_model_folder(
                model_id=model_id,
                folder_name=module.name,
            )
            model_folders[model_id] = folder_id

            log.info(
                "module_model_synced",
                module_id=module.id,
                model_id=model_id,
                folder_id=folder_id,
            )
        except Exception as e:
            log.warning(
                "module_model_sync_failed",
                module_id=module.id,
                error=str(e),
            )

    # Create background agent models (hidden)
    for task in course.tasks:
        try:
            agent_name = task.name.replace("-", " ").replace("_", " ").title()
            model = await openwebui.ensure_background_agent_model(
                agent_id=task.name,
                agent_name=agent_name,
                base_model_id="youlab",
                system_prompt=task.system,
                description=f"Background agent: {agent_name}",
            )
            model_id = model["id"]

            # Create folder for background agent threads
            folder_id = await openwebui.ensure_model_folder(
                model_id=model_id,
                folder_name=f"[System] {agent_name}",
            )
            model_folders[model_id] = folder_id

            log.info(
                "background_agent_model_synced",
                agent_id=task.name,
                model_id=model_id,
                folder_id=folder_id,
            )
        except Exception as e:
            log.warning(
                "background_agent_model_sync_failed",
                agent_id=task.name,
                error=str(e),
            )

    return model_folders
```

#### 2. Call model sync during agent creation

**File**: `src/youlab_server/server/agents.py`

Update `create_agent_from_curriculum` to call model sync (after folder attachment, around line 354):

```python
# After the existing folder attachment block, add:

# Sync models to OpenWebUI
model_folders: dict[str, str] = {}
if settings.openwebui_api_url and settings.openwebui_api_key:
    from youlab_server.server.sync.openwebui_client import OpenWebUIClient

    openwebui = OpenWebUIClient(
        settings.openwebui_api_url,
        settings.openwebui_api_key,
    )
    try:
        model_folders = await self._sync_models_to_openwebui(
            course=course,
            user_id=user_id,
            openwebui=openwebui,
        )
    except Exception as e:
        log.warning("model_sync_failed", error=str(e))
    finally:
        await openwebui.close()
```

Update the final log statement to include model info:

```python
log.info(
    "agent_created_from_curriculum",
    user_id=user_id,
    course_id=course_id,
    agent_id=agent.id,
    blocks=list(course.blocks.keys()),
    shared_blocks=len(shared_block_ids),
    tools=tool_names,
    folders=attached_folders,
    models=list(model_folders.keys()),  # Add this
)
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Module models created:
  ```bash
  curl -s $OPENWEBUI_URL/api/models | jq '[.[] | select(.meta.type == "module")]'
  # Expected: array with module models
  ```
- [ ] Background agents hidden:
  ```bash
  curl -s $OPENWEBUI_URL/api/models | jq '[.[] | select(.meta.hidden == true)]'
  # Expected: array with background agent models
  ```
- [ ] Folders linked to models:
  ```bash
  curl -s $OPENWEBUI_URL/api/folders | jq '.[] | select(.meta.model_id != null) | {name, model_id: .meta.model_id}'
  ```

#### Manual Verification:
- [ ] Enroll in course, verify module models appear in chat selector
- [ ] Verify background agents NOT in chat selector
- [ ] Verify folders created with module/agent names
- [ ] Check OpenWebUI admin > Models shows all models

**Implementation Note**: After completing this phase, pause for manual verification before proceeding.

---

## Phase 5.4: Chat Auto-Assignment to Folders

### Overview

Automatically assign new chats to the appropriate folder based on the selected model. This happens in the pipe middleware when a new chat is detected.

### Changes Required:

#### 1. Add chat folder assignment to pipe

**File**: `src/youlab_server/pipelines/letta_pipe.py`

Add method to handle folder assignment (after `_ensure_agent_exists`, around line 136):

```python
async def _ensure_chat_folder(
    self,
    client: httpx.AsyncClient,
    chat_id: str,
    model_id: str,
) -> str | None:
    """
    Ensure chat is in the correct folder for its model.

    Returns folder_id if assigned, None if no folder found.
    """
    if not self.valves.YOULAB_SERVICE_URL:
        return None

    try:
        # Look up folder for this model
        response = await client.get(
            f"{self.valves.YOULAB_SERVICE_URL}/models/{model_id}/folder",
        )
        if response.status_code != HTTPStatus.OK:
            return None

        folder_id = response.json().get("folder_id")
        if not folder_id:
            return None

        # Assign chat to folder
        response = await client.post(
            f"{self.valves.OPENWEBUI_API_URL}/api/chats/{chat_id}/folder",
            json={"folder_id": folder_id},
            headers={"Authorization": f"Bearer {self.valves.OPENWEBUI_API_KEY}"},
        )
        if response.status_code == HTTPStatus.OK:
            return folder_id

    except Exception as e:
        log.warning("chat_folder_assignment_failed", chat_id=chat_id, error=str(e))

    return None
```

#### 2. Add model-folder lookup endpoint

**File**: `src/youlab_server/server/main.py`

Add new endpoint (after existing endpoints):

```python
@app.get("/models/{model_id}/folder")
async def get_model_folder(
    model_id: str,
    settings: Annotated[Settings, Depends(get_settings)],
) -> dict[str, str | None]:
    """Get the folder ID associated with a model."""
    if not settings.openwebui_api_url or not settings.openwebui_api_key:
        return {"folder_id": None}

    from youlab_server.server.sync.openwebui_client import OpenWebUIClient

    openwebui = OpenWebUIClient(
        settings.openwebui_api_url,
        settings.openwebui_api_key,
    )
    try:
        folders = await openwebui.list_folders()
        for folder in folders:
            if folder.get("meta", {}).get("model_id") == model_id:
                return {"folder_id": folder["id"]}
        return {"folder_id": None}
    finally:
        await openwebui.close()
```

#### 3. Call folder assignment in pipe

**File**: `src/youlab_server/pipelines/letta_pipe.py`

Update `pipe()` method to assign folder for new chats (after agent exists check, around line 180):

```python
# After ensuring agent exists
agent_id = await self._ensure_agent_exists(client, user_id, user_name)

# Assign chat to model's folder if this is a new chat
if chat_id and len(messages) <= 1:  # New chat (first message)
    model_id = body.get("model", "")
    if model_id.startswith("module-") or model_id.startswith("agent-"):
        await self._ensure_chat_folder(client, chat_id, model_id)
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Folder lookup endpoint works:
  ```bash
  curl -s $YOULAB_URL/models/module-01-first-impression/folder
  # Expected: {"folder_id": "..."}
  ```

#### Manual Verification:
- [ ] Select module model in OpenWebUI
- [ ] Send first message to create chat
- [ ] Verify chat appears in module's folder in sidebar
- [ ] Existing chats remain unaffected

**Implementation Note**: This phase requires the pipe to have access to OpenWebUI API for folder assignment. Ensure `OPENWEBUI_API_URL` and `OPENWEBUI_API_KEY` valves are configured.

---

## Phase 5.5: Hot Reload Model Sync

### Overview

When curriculum is hot-reloaded (modules added/changed), sync new models to OpenWebUI.

### Changes Required:

#### 1. Add model sync to curriculum reload

**File**: `src/youlab_server/curriculum/__init__.py`

Add method to sync models after reload (after `reload` method):

```python
async def sync_models_to_openwebui(
    self,
    course_id: str,
    openwebui: "OpenWebUIClient",
) -> list[str]:
    """
    Sync course models to OpenWebUI after reload.

    Returns list of model IDs that were synced.
    """
    course = self.get(course_id)
    if course is None:
        return []

    synced: list[str] = []

    # Sync module models
    for module in course.loaded_modules:
        try:
            model = await openwebui.ensure_module_model(
                module_id=module.id,
                module_name=module.name,
                base_model_id="youlab",
                system_prompt=course.agent.system,
            )
            synced.append(model["id"])
        except Exception:
            pass

    # Sync background agent models
    for task in course.tasks:
        try:
            model = await openwebui.ensure_background_agent_model(
                agent_id=task.name,
                agent_name=task.name.replace("-", " ").title(),
                base_model_id="youlab",
                system_prompt=task.system,
            )
            synced.append(model["id"])
        except Exception:
            pass

    return synced
```

#### 2. Add reload endpoint with model sync

**File**: `src/youlab_server/server/main.py`

Update reload endpoint to sync models:

```python
@app.post("/curriculum/reload")
async def reload_curriculum(
    settings: Annotated[Settings, Depends(get_settings)],
) -> dict[str, Any]:
    """Reload curriculum configuration and sync models."""
    courses = curriculum.reload()

    synced_models: dict[str, list[str]] = {}

    if settings.openwebui_api_url and settings.openwebui_api_key:
        from youlab_server.server.sync.openwebui_client import OpenWebUIClient

        openwebui = OpenWebUIClient(
            settings.openwebui_api_url,
            settings.openwebui_api_key,
        )
        try:
            for course_id in courses:
                synced = await curriculum.sync_models_to_openwebui(course_id, openwebui)
                synced_models[course_id] = synced
        finally:
            await openwebui.close()

    return {
        "courses_reloaded": courses,
        "models_synced": synced_models,
    }
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Reload syncs models:
  ```bash
  # Add new module to config
  curl -X POST $YOULAB_URL/curriculum/reload
  # Expected: {"courses_reloaded": [...], "models_synced": {"college-essay": [...]}}
  ```

#### Manual Verification:
- [ ] Add new module TOML to config/courses/
- [ ] Call reload endpoint
- [ ] Verify new model appears in OpenWebUI
- [ ] Verify folder created for new model

---

## Testing Strategy

### Unit Tests

Add to `tests/test_server/`:

1. **test_openwebui_models.py** - OpenWebUIClient model methods
   - Test create_model with various meta fields
   - Test ensure_module_model idempotency
   - Test ensure_background_agent_model hides model
   - Test ensure_model_folder creates linked folder

2. **test_model_sync.py** - Model sync during agent creation
   - Test models created for modules
   - Test models created for background agents
   - Test folders linked to models

### Integration Tests

1. **test_chat_folder_assignment.py** - End-to-end folder assignment
   - Create chat with module model
   - Verify chat assigned to folder
   - Verify folder has correct model_id in meta

### Manual Testing Steps

1. Start fresh OpenWebUI instance
2. Start YouLab server
3. Verify "youlab" base model created
4. Enroll user in course
5. Verify module models visible in chat selector
6. Verify background agents NOT visible
7. Start chat with module, verify folder assignment
8. Add new module to config, reload, verify model created

## Performance Considerations

1. **Model Caching**: `ensure_*` methods check existence before creating
2. **Folder Lookup**: O(n) scan of folders - acceptable for small number of models
3. **Async Operations**: All OpenWebUI calls are async, non-blocking
4. **Connection Reuse**: Client reuses httpx.AsyncClient with connection pooling

## Migration Notes

### Existing Users

- Models created on next enrollment/agent creation
- Existing chats remain in root (no folder)
- Manual folder assignment still works

### Model ID Schema

| Type | ID Pattern | Example |
|------|------------|---------|
| Base | `youlab` | `youlab` |
| Module | `module-{module_id}` | `module-01-first-impression` |
| Agent | `agent-{task_name}` | `agent-memory-enricher` |

### Folder Naming

| Type | Name Pattern | Example |
|------|--------------|---------|
| Module | `{module_name}` | `First Impression` |
| Agent | `[System] {agent_name}` | `[System] Memory Enricher` |

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Research: `thoughts/shared/research/2026-01-13-ARI-82-openwebui-models-chat-folders.md`
- OpenWebUI Models API: `OpenWebUI/open-webui/backend/open_webui/routers/models.py`
- OpenWebUI Folders API: `OpenWebUI/open-webui/backend/open_webui/routers/folders.py`
- Current OpenWebUIClient: `src/youlab_server/server/sync/openwebui_client.py`
- Agent creation: `src/youlab_server/server/agents.py:220-367`
- Linear ticket: [ARI-82](https://linear.app/ariav/issue/ARI-82)
