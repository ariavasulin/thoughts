---
date: 2026-01-13T17:12:49+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "OpenWebUI Models Page and Chat Folder Organization"
tags: [research, codebase, openwebui, models, folders, chats, memory-blocks]
status: complete
last_updated: 2026-01-13
last_updated_by: ariasulin
linear_ticket: ARI-82
---

# Research: OpenWebUI Models Page and Chat Folder Organization

**Date**: 2026-01-13T17:12:49+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Research OpenWebUI's Models page and chat folder organization to understand:
1. Models page components and how models are displayed
2. How chat threads are organized into folders
3. The logic for assigning chats to model-based folders
4. How hidden models work (if any existing implementation)
5. Base model selection pattern (for YouLab as base-model)

**Goal**: Understand how modules should persist as models and how background agent threads get organized.

**Gap Analysis**: Compare to spec where modules = visible models, background agents = hidden models.

## Summary

OpenWebUI has a comprehensive model management and folder organization system that can support the YouLab architecture with minimal modifications:

1. **Models as Modules**: Custom "preset" models with `base_model_id` can wrap a base model (YouLab pipe) with custom system prompts and parameters - perfect for course modules
2. **Hidden Models**: The `meta.hidden` flag exists and is filtered from user-facing selectors while remaining visible in admin views - ideal for background agents
3. **Folders**: Manual folder organization exists but NO automatic model-based folder assignment - this is a gap that needs implementation
4. **Background Agent Type**: A `meta.type = "background_agent"` concept already exists and is filtered from tag groupings

## Detailed Findings

### 1. Models Page Components

**Location**: `OpenWebUI/open-webui/src/lib/components/workspace/Models/`

**Main Components**:
- `Models.svelte` - Main models list page with grid display
- `ModelEditor.svelte` - Model create/edit form
- `ModelMenu.svelte` - Dropdown menu for model actions (edit, hide, clone, delete, export)

**Routes**:
- `/workspace/models` - Models list
- `/workspace/models/create` - Create new model
- `/workspace/models/edit?id={model_id}` - Edit model

**Display Features**:
- 2-column grid layout for model cards
- Profile image, name, creator, description
- Enable/disable toggle (`is_active`)
- Pagination (30 per page)
- Tag filtering and search
- View options: All, Created by you, Shared with you

### 2. Model Data Schema

**Database Table** (`backend/open_webui/models/models.py:53-103`):

```python
class Model(Base):
    id = Column(Text, primary_key=True)        # Model identifier
    user_id = Column(Text)                      # Creator
    base_model_id = Column(Text, nullable=True) # Base model reference
    name = Column(Text)                         # Display name
    params = Column(JSONField)                  # System prompt, temperature, etc.
    meta = Column(JSONField)                    # Metadata (see below)
    access_control = Column(JSON, nullable=True)
    is_active = Column(Boolean, default=True)
    created_at = Column(BigInteger)
    updated_at = Column(BigInteger)
```

**Meta Schema** (stored in `meta` JSON field):
```python
{
    "profile_image_url": str,
    "description": str | None,
    "suggestion_prompts": list | None,
    "tags": [{"name": str}],
    "hidden": bool,              # KEY: Controls visibility in selectors
    "knowledge": list,           # Knowledge base attachments
    "toolIds": list,
    "filterIds": list,
    "actionIds": list,
    "capabilities": {
        "vision": bool,
        "file_upload": bool,
        "web_search": bool,
        "image_generation": bool,
        "code_interpreter": bool
    },
    "type": str | None           # KEY: Can be "background_agent"
}
```

### 3. Hidden Models Implementation

**Flag**: `meta.hidden` (boolean, default: false/undefined)

**Toggle Location**: `workspace/Models.svelte:163-191`
```javascript
const hideModelHandler = async (model) => {
    model.meta = {
        ...model.meta,
        hidden: !(model?.meta?.hidden ?? false)
    };
    const res = await updateModelById(localStorage.token, model.id, model);
}
```

**Frontend Filtering** (hidden models excluded):
- Chat interface: `chat/Chat.svelte:914`
- Model selector: `chat/ModelSelector/Selector.svelte:161, 323, 454`
- Command palette: `chat/MessageInput/Commands/Models.svelte:21, 41`
- Notes editor: `notes/NoteEditor.svelte:308, 804, 814`

**Filter Pattern**:
```javascript
.filter((model) => !(model?.info?.meta?.hidden ?? false))
```

**Visual Feedback**: Hidden models shown with 50% opacity in workspace view:
```javascript
class={model?.meta?.hidden ? 'opacity-50 dark:opacity-50' : ''}
```

**Background Agent Type**: Additional filtering exists for `meta.type === "background_agent"`:
```javascript
// ModelSelector/Selector.svelte:454
item.model?.info?.meta?.type !== 'background_agent'
```

### 4. Base Model Selection Pattern

**Purpose**: Custom "preset" models can wrap a base model with custom parameters

**Field**: `Model.base_model_id` (nullable Text)

**Frontend Selector** (`ModelEditor.svelte:526-548`):
- Only shown when `preset={true}` prop
- Dropdown filters out: current model, other presets, arena models, direct connection models

**Request Proxying**:
When a chat uses a custom model with `base_model_id`:
1. Backend looks up custom model's `base_model_id`
2. Replaces `payload["model"]` with base model ID
3. Applies custom model's params and system prompt
4. Proxies to actual LLM provider

**Code Path** (`routers/openai.py:814-821`):
```python
if model_info.base_model_id:
    payload["model"] = model_info.base_model_id
```

**YouLab Pattern**: Create modules as preset models with `base_model_id = "youlab-pipe-id"`

### 5. Folder Organization System

**Database Schema** (`backend/open_webui/models/folders.py:22-33`):
```python
class Folder(Base):
    id = Column(Text, primary_key=True)
    parent_id = Column(Text, nullable=True)  # Hierarchical
    user_id = Column(Text)
    name = Column(Text)
    meta = Column(JSON, nullable=True)       # Icon, background_image_url
    data = Column(JSON, nullable=True)       # System prompt, files
    is_expanded = Column(Boolean, default=False)
```

**Chat-Folder Relationship** (`backend/open_webui/models/chats.py:51`):
```python
folder_id = Column(Text, nullable=True)
```

**Assignment Methods**:
1. **Menu**: Chat menu > Move > Select folder
2. **Drag-and-drop**: Drag chat onto folder component

**Key Behaviors**:
- Chats created with `folder_id = null` by default
- **NO automatic folder assignment** - requires explicit user action
- Moving to folder unpins the chat automatically
- Archiving a chat clears its `folder_id`
- Pinned chats excluded from folder views

### 6. Chat-to-Folder Assignment Logic

**Current State**: Manual only, no automatic assignment

**New Chat Creation** (`apis/chats/index.ts:4-34`):
```typescript
createNewChat(token: string, chat: object, folderId: string | null)
// folderId must be explicitly provided
```

**Backend** (`models/chats.py:241-264`):
- `folder_id` comes from `form_data.folder_id`
- Defaults to `None` if not specified
- No model-based, date-based, or tag-based auto-assignment

## Gap Analysis: Spec vs Current Implementation

| Feature | Spec Requirement | Current State | Gap |
|---------|------------------|---------------|-----|
| Modules as models | Modules persist as visible models | `meta.hidden=false` models visible | None - use preset models with `base_model_id` |
| Background agents as hidden | Agents persist as hidden models | `meta.hidden=true` filtering works | None - set `meta.hidden=true` on creation |
| YouLab as base model | All modules use YouLab pipe | `base_model_id` field exists | Need to create YouLab pipe model first |
| Auto folder by agent/module | Chat threads organized by agent | Manual folder assignment only | **GAP**: Need auto-assignment on chat creation |
| Model-based folder creation | Folders created per module/agent | Folders are user-created | **GAP**: Need auto-folder creation per model |
| Background agent type | Special type for background agents | `meta.type` field exists | Use `meta.type="background_agent"` |

### Implementation Gaps

1. **Auto-folder creation per model**
   - When a module/agent model is created, auto-create matching folder
   - Set folder metadata to link to model ID

2. **Auto-assign chats to model folder**
   - Modify chat creation to check model type
   - If model has associated folder, set `folder_id` automatically
   - Could add `meta.folder_id` to model schema

3. **Folder-model linking**
   - Add `model_id` field to folder schema OR
   - Add `folder_id` field to model meta
   - Query to find folder by model on chat creation

## Code References

### Models
- Model schema: `backend/open_webui/models/models.py:53-103`
- Model list page: `src/lib/components/workspace/Models.svelte`
- Model editor: `src/lib/components/workspace/Models/ModelEditor.svelte`
- Hide handler: `src/lib/components/workspace/Models.svelte:163-191`
- Model selector filtering: `src/lib/components/chat/ModelSelector/Selector.svelte:161`

### Folders
- Folder schema: `backend/open_webui/models/folders.py:22-33`
- Folder router: `backend/open_webui/routers/folders.py`
- Sidebar folders: `src/lib/components/layout/Sidebar/Folders.svelte`
- Recursive folder: `src/lib/components/layout/Sidebar/RecursiveFolder.svelte`

### Chats
- Chat schema: `backend/open_webui/models/chats.py:35-66`
- Chat-folder update: `backend/open_webui/models/chats.py:1039-1052`
- Folder assignment API: `backend/open_webui/routers/chats.py:956-969`

### Base Model
- Base model substitution: `backend/open_webui/routers/openai.py:814-821`
- Model aggregation: `backend/open_webui/utils/models.py:150-242`
- Base model filter: `src/lib/components/workspace/Models/ModelEditor.svelte:542`

## Architecture Documentation

### Model Visibility Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Model Visibility                      │
├─────────────────────────────────────────────────────────┤
│  Layer 1: is_active (Database)                          │
│    - false = completely removed from API                │
│    - Filtered in backend utils/models.py                │
├─────────────────────────────────────────────────────────┤
│  Layer 2: access_control (Database)                     │
│    - null = public, {} = private, {...} = group-based   │
│    - Filtered in backend by user permissions            │
├─────────────────────────────────────────────────────────┤
│  Layer 3: meta.hidden (Metadata)                        │
│    - true = hidden from user selectors, visible admin   │
│    - Filtered in frontend components                    │
├─────────────────────────────────────────────────────────┤
│  Layer 4: meta.type (Metadata)                          │
│    - "background_agent" = excluded from tag groups      │
│    - Filtered in frontend ModelSelector                 │
└─────────────────────────────────────────────────────────┘
```

### Model Inheritance Pattern

```
┌──────────────────┐     base_model_id      ┌──────────────────┐
│  Custom Model    │ ───────────────────▶   │   Base Model     │
│  (Module/Agent)  │                        │  (YouLab Pipe)   │
├──────────────────┤                        ├──────────────────┤
│ id: "module-1"   │                        │ id: "youlab"     │
│ name: "Essay..."│                        │ name: "YouLab"   │
│ params.system:   │                        │ pipe: true       │
│   "You are..."   │                        │ owned_by: openai │
│ meta.hidden:     │                        └──────────────────┘
│   false          │
│ base_model_id:   │
│   "youlab"       │
└──────────────────┘
```

### Folder-Chat Organization (Current)

```
┌─────────────────────────────────────────────────────────┐
│                     Sidebar                              │
├─────────────────────────────────────────────────────────┤
│  📁 Folders (user-created)                              │
│    └─ 📁 My Essays                                      │
│         └─ Chat: "Personal Statement Draft"             │
│    └─ 📁 Research                                       │
│         └─ Chat: "Topic Ideas"                          │
├─────────────────────────────────────────────────────────┤
│  💬 Chats (no folder, folder_id=null)                   │
│    └─ Chat: "Random conversation"                       │
├─────────────────────────────────────────────────────────┤
│  📌 Pinned (excluded from folders)                      │
│    └─ Chat: "Important notes"                           │
└─────────────────────────────────────────────────────────┘
```

### Proposed Folder-Chat Organization (With Spec)

```
┌─────────────────────────────────────────────────────────┐
│                     Sidebar                              │
├─────────────────────────────────────────────────────────┤
│  📁 Modules (auto-created from visible models)          │
│    └─ 📁 First Impression                               │
│         └─ Chat: "Session 1" (auto-assigned)            │
│         └─ Chat: "Session 2"                            │
│    └─ 📁 Deep Dive                                      │
│         └─ Chat: "Session 1"                            │
├─────────────────────────────────────────────────────────┤
│  📁 Background (hidden, for hidden models)              │
│    └─ 📁 Memory Enricher (meta.type=background_agent)   │
│         └─ Chat: "Enrichment run 2026-01-13"            │
│    └─ 📁 Reflections Agent                              │
│         └─ Chat: "Daily reflection"                     │
├─────────────────────────────────────────────────────────┤
│  💬 General Chats                                       │
│    └─ Chat: "Questions about course"                    │
└─────────────────────────────────────────────────────────┘
```

## Open Questions

1. **Model-Folder Linking Schema**: Should we add `model_id` to folders or `folder_id` to models?
   - Adding to folders: `Folder.model_id` - simpler, one folder per model
   - Adding to models: `Model.meta.folder_id` - more flexible, models could share folders

2. **Auto-folder Creation Timing**: When should model folders be created?
   - On model creation (proactive)
   - On first chat with model (lazy)
   - On course enrollment (bulk)

3. **Background Folder Visibility**: Should background agent folders be completely hidden or collapsed by default?
   - Completely hidden: Cleaner UX, users don't see system operations
   - Collapsed: Power users can inspect background agent activity

4. **Folder Hierarchy**: Should module folders be nested under a course folder?
   - Flat: `[Module 1] [Module 2] [Module 3]`
   - Nested: `Course > Module 1, Module 2, Module 3`

## Related Research

- ARI-80: Memory System MVP implementation
- ARI-79: Module metadata schema documentation
- ARI-81: Related architecture decisions

---

*Research completed as part of ARI-82 Research Swarm: Memory Blocks as Notes Architecture*
