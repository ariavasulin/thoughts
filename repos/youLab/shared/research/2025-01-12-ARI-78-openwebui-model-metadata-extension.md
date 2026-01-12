---
date: 2026-01-12T03:33:10Z
researcher: claude
git_commit: 4c099b8eefca5a1c178847faf034af8047936438
branch: chore/coderabbit-setup
repository: YouLab
topic: "OpenWebUI Model Metadata Extension for Modules System"
tags: [research, codebase, openwebui, models, metadata, modules]
status: complete
last_updated: 2026-01-12
last_updated_by: claude
---

# Research: OpenWebUI Model Metadata Extension for Modules System

**Date**: 2026-01-12T03:33:10Z
**Researcher**: claude
**Git Commit**: 4c099b8eefca5a1c178847faf034af8047936438
**Branch**: chore/coderabbit-setup
**Repository**: YouLab

## Research Question

For the Modules system (replacing Models in sidebar), investigate how to extend OpenWebUI's model metadata to support module-specific data. Key questions:
1. Where is model metadata defined (schema, types)?
2. How is model.meta used currently?
3. What fields could we add for module-specific data?
4. How do models associate with folders/chats?
5. Is there a pattern for extending metadata without breaking upstream?

## Summary

OpenWebUI's model metadata system is **intentionally extensible** and well-suited for adding module-specific fields. The `model.meta` field uses a Pydantic model with `ConfigDict(extra="allow")`, storing data as JSON in the database without schema validation. This means we can add arbitrary fields like `module_id`, `course_id`, and step configurations without modifying the backend schema or risking upstream compatibility.

**Key findings:**
- Models are **NOT associated with folders** - folders only organize chats
- Chats store model references in their JSON `chat.models` array
- The `meta` field already stores 15+ custom fields beyond the 3 defined in schema
- Extension pattern: add fields in frontend, store in existing `meta` JSON column
- No namespacing convention exists - fields use flat names like `toolIds`, `actionIds`

## Detailed Findings

### 1. Backend Model Schema

**Location**: `OpenWebUI/open-webui/backend/open_webui/models/models.py`

The Model database schema (lines 53-103) stores metadata in two JSON columns:

```python
class Model(Base):
    __tablename__ = "model"
    id = Column(Text, primary_key=True)
    user_id = Column(Text)
    base_model_id = Column(Text, nullable=True)
    name = Column(Text)
    params = Column(JSONField)      # JSON blob for model parameters
    meta = Column(JSONField)        # JSON blob for metadata
    access_control = Column(JSON)   # Permission settings
    is_active = Column(Boolean)
    updated_at = Column(BigInteger)
    created_at = Column(BigInteger)
```

**ModelMeta Pydantic Schema** (lines 38-50):

```python
class ModelMeta(BaseModel):
    profile_image_url: Optional[str] = "/static/favicon.png"
    description: Optional[str] = None
    capabilities: Optional[dict] = None

    model_config = ConfigDict(extra="allow")  # Accepts ANY additional fields
```

The `extra="allow"` configuration means any field can be added without schema changes.

### 2. Current model.meta Usage

**All meta fields currently in use** (documented from codebase analysis):

| Field | Type | Purpose |
|-------|------|---------|
| `profile_image_url` | string | Model avatar (default: `/static/favicon.png`) |
| `description` | string | User-facing description |
| `capabilities` | dict | Feature flags: vision, file_upload, web_search, etc. |
| `knowledge` | array | References to attached knowledge bases |
| `toolIds` | string[] | IDs of attached tools |
| `filterIds` | string[] | Available filter function IDs |
| `defaultFilterIds` | string[] | Filters enabled by default |
| `actionIds` | string[] | Action function IDs |
| `defaultFeatureIds` | string[] | Features enabled by default |
| `tags` | `{name: string}[]` | Categorization tags |
| `suggestion_prompts` | array | Pre-written prompt suggestions |
| `hidden` | boolean | Controls visibility in selectors |
| `model_ids` | string[] | (Arena models) Routing target IDs |
| `filter_mode` | string | (Arena models) include/exclude mode |
| `access_control` | object | (Arena models) Permission overrides |

**Access patterns in code:**

```javascript
// Frontend safe access
model?.info?.meta?.capabilities?.vision ?? true

// Backend Pydantic access
if model.meta and hasattr(model.meta, "knowledge"):
    knowledge_list = model.meta.knowledge or []
```

### 3. Model-Folder-Chat Associations

**Critical finding: Models are NOT associated with folders.**

Database relationships:
- **Folder → Chats**: `Chat.folder_id` references `Folder.id`
- **Chat → Models**: `Chat.chat['models']` JSON array of model IDs
- **Model → Folders**: NO relationship exists

Sidebar structure (from `Sidebar.svelte`):
1. **Pinned Models** - from user settings `$settings.pinnedModels`
2. **Folders** - contain chats only (not models)
3. **Chats** - not in folders

```
┌─────────────────────────┐
│  Pinned Models          │  ← Not in folders
│    └─ [model list]      │
│  Folders                │  ← Contains chats
│    └─ Folder 1          │
│        └─ Chat A        │  ← Chat.models: ["model-1"]
│        └─ Chat B        │
│  Chats                  │  ← Not in folders
│    └─ Chat C            │
└─────────────────────────┘
```

**For Modules system**: We would add "Modules" section that displays models filtered by module-specific metadata (e.g., `meta.module_id`, `meta.is_active_module`).

### 4. Frontend Model Handling

**TypeScript types** (`OpenWebUI/open-webui/src/lib/apis/index.ts:1658-1671`):

```typescript
export interface ModelConfig {
    id: string;
    name: string;
    meta: ModelMeta;
    base_model_id?: string;
    params: ModelParams;
}

export interface ModelMeta {
    toolIds: never[];
    description?: string;
    capabilities?: object;
    profile_image_url?: string;
}
```

Note: The TypeScript interface is incomplete - actual usage includes many more fields.

**Key components:**
- `ModelSelector.svelte:60-64` - Main model picker
- `Selector.svelte:160` - Filters out `meta.hidden` models
- `ModelItem.svelte:201-224` - Displays description, tags
- `ModelEditor.svelte:73-193` - Edit interface for all meta fields
- `Capabilities.svelte:9-44` - Capability toggles

### 5. Safe Extension Patterns

**Pattern 1: Frontend-Driven Extension** (recommended)
```javascript
// ModelEditor.svelte pattern - add field conditionally
if (moduleConfig) {
    info.meta.module_id = moduleConfig.id;
    info.meta.course_id = moduleConfig.courseId;
} else {
    delete info.meta.module_id;
    delete info.meta.course_id;
}
```

**Pattern 2: Merge on Update** (backend)
```python
# functions.py:297 - shallow merge pattern
function.meta = {**function.meta, **new_metadata}
```

**Pattern 3: No Namespacing**
Existing fields use flat names (`toolIds`, `actionIds`). No `x-` or `custom.` prefix convention exists.

## Proposed Module-Specific Fields

Based on the product spec and existing patterns:

```typescript
interface ModuleMeta extends ModelMeta {
    // Module identity
    module_id?: string;           // References TOML module config
    course_id?: string;           // Parent course identifier

    // Display in Modules section
    is_module?: boolean;          // Show in Modules sidebar section
    module_order?: number;        // Sort order in sidebar

    // Step configuration
    current_step_id?: string;     // Active step within module
    step_configs?: StepConfig[];  // Per-step overrides

    // Module state
    module_status?: 'locked' | 'active' | 'completed';
    progress_percent?: number;
}

interface StepConfig {
    step_id: string;
    title: string;
    system_prompt_override?: string;
}
```

These fields would:
1. Store in existing `meta` JSON column without migrations
2. Be filtered in a new "Modules" sidebar component
3. Support per-user module state tracking
4. Enable step-based navigation within modules

## Code References

### Backend
- `OpenWebUI/open-webui/backend/open_webui/models/models.py:38-50` - ModelMeta schema
- `OpenWebUI/open-webui/backend/open_webui/models/models.py:53-103` - Model table
- `OpenWebUI/open-webui/backend/open_webui/internal/db.py:29-48` - JSONField serialization
- `OpenWebUI/open-webui/backend/open_webui/routers/models.py:291-323` - Profile image endpoint
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-52` - Chat table with folder_id
- `OpenWebUI/open-webui/backend/open_webui/models/folders.py:22-34` - Folder table (no model FK)

### Frontend
- `OpenWebUI/open-webui/src/lib/apis/index.ts:1658-1671` - TypeScript types
- `OpenWebUI/open-webui/src/lib/stores/index.ts:105-144` - Model store types
- `OpenWebUI/open-webui/src/lib/components/workspace/Models/ModelEditor.svelte:73-193` - Meta editing
- `OpenWebUI/open-webui/src/lib/components/chat/ModelSelector/Selector.svelte:160` - Hidden filter
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte:1065-1234` - Sidebar sections
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/PinnedModelList.svelte:40-89` - Pinned models

## Architecture Documentation

### Extension Safety

The OpenWebUI metadata system is designed for extension:

1. **Database Layer**: JSONField accepts any JSON structure
2. **Backend Layer**: Pydantic `extra="allow"` preserves unknown fields
3. **API Layer**: Endpoints merge dicts without validation
4. **Frontend Layer**: TypeScript types are incomplete by design

**No changes needed for basic field additions:**
- Add fields to frontend components
- Store in existing `meta` column
- No migrations required
- No backend code changes for storage

**Changes needed for filtering/querying:**
- Backend: Add query parameters to `search_models()` if filtering by new fields
- Frontend: Add new sidebar component for Modules view

### Upstream Compatibility

Extending `meta` is **upstream-safe** because:
1. OpenWebUI's schema explicitly allows extra fields
2. Unknown fields are preserved through all operations (import/export/sync)
3. No validation rejects unknown fields
4. Cleanup pattern removes empty fields, keeping JSON clean

## Open Questions

1. **User-specific module state**: Should module progress (current_step, completion status) live in `meta` or a separate user-module table?

2. **Sidebar component reuse**: Can we reuse `PinnedModelList` for Modules, or do we need a new `ModuleList` component?

3. **Model-module mapping**: Should one model = one module, or can multiple modules share a model (as suggested in the spec)?

4. **Hidden models**: How do we handle models that should appear in Modules but not in the regular Models catalog?

## Related Research

- Product spec: `thoughts/shared/plans/2025-01-12-youlab-product-spec.md`
- Linear ticket: ARI-78 (YouLab Product Spec)
