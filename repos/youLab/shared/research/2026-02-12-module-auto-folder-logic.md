---
date: 2026-02-12T12:00:00-08:00
researcher: Claude
git_commit: 96e3e60
branch: main
repository: YouLab
topic: "Module auto-folder logic: how to remove it from YouLab and OpenWebUI fork"
tags: [research, codebase, folders, modules, openwebui]
status: complete
last_updated: 2026-02-12
last_updated_by: Claude
---

# Research: Module Auto-Folder Logic Removal

**Date**: 2026-02-12
**Git Commit**: 96e3e60
**Branch**: main

## Research Question

How do we remove the module auto-folder logic from YouLab and the OpenWebUI fork? Module chats currently go to specific folders automatically — the goal is to completely remove all that logic.

## Summary

The module auto-folder system lives **entirely in the OpenWebUI fork frontend** (`/Users/ariasulin/Git/YouLab/OpenWebUI/open-webui/`). The Ralph backend (`src/ralph/`) has **zero folder logic**. The legacy Letta stack (`src/youlab_server/`) has folder code but is deprecated and irrelevant.

The system works by:
1. Models with `info.meta.youlab_module` metadata are treated as "modules"
2. When a module chat is created, a folder is auto-created/found and the chat is assigned to it
3. Clicking a module in the sidebar sets `selectedFolder`, opens the most recent chat in that folder, or creates a new one

**Files to modify (8 files across the OpenWebUI fork):**

| File | What to change |
|------|---------------|
| `src/lib/utils/folders.ts` | Remove `ensureModuleFolder()`, `ensureAgentFolder()`, `getMostRecentThreadInFolder()`, `clearFolderCache()` |
| `src/lib/components/chat/Chat.svelte` | Remove module detection + folder assignment in `initChatHandler()` (~lines 2273-2296) |
| `src/lib/components/layout/Sidebar/ModuleItem.svelte` | Remove folder creation/selection in `handleModuleClick()` (~lines 28-52) |
| `src/lib/components/layout/Sidebar/ModuleList.svelte` | May need updates if ModuleItem interface changes |
| `src/lib/apis/chats/index.ts` | `createNewChat()` still accepts `folderId` param — make it always `null` or remove param |
| `src/lib/stores/index.ts` | `selectedFolder` store — possibly remove or stop using for module flow |
| `src/lib/apis/index.ts` | `YouLabModuleMeta` type — keep (used for module detection) but remove folder-related usage |
| `src/lib/apis/folders/index.ts` | No changes needed (generic folder CRUD, used by regular folders too) |

## Detailed Findings

### 1. `src/lib/utils/folders.ts` — Core Auto-Folder Logic

This file contains all the module-specific folder utilities. The entire file is YouLab custom code.

**`ensureModuleFolder()` (lines 22-54):**
- Takes `token`, `moduleId`, `moduleName`
- Uses in-memory cache (`folderCache`) to avoid duplicate API calls
- Fetches all folders via `getFolders(token)`
- Searches for folder matching `moduleName`
- If not found, creates folder with `meta: { type: 'module', moduleId }`
- Returns folder ID

**`ensureAgentFolder()` (lines 64-93):**
- Same pattern but for background agents
- Creates folder with `meta: { type: 'background_agent', agentId }`

**`getMostRecentThreadInFolder()` (lines 102-112):**
- Gets chats in folder sorted by `updated_at`
- Returns first (most recent) chat ID

**`clearFolderCache()` (lines 117-119):**
- Resets the in-memory cache

**Removal**: Delete the entire file or strip it to just `clearFolderCache()` if used elsewhere.

### 2. `src/lib/components/chat/Chat.svelte` — Chat Creation with Auto-Folder

**`initChatHandler()` (lines 2270-2332):**

Lines 2273-2284 — Module detection and folder assignment:
```typescript
const model = $models.find((m) => m.id === selectedModels[0]);
const hasYoulabModule = model?.info?.meta?.youlab_module;
let isModuleChat = $selectedFolder?.meta?.type === 'module' || hasYoulabModule;

let folderId = $selectedFolder?.id;
if (hasYoulabModule && !folderId) {
    folderId = await ensureModuleFolder(localStorage.token, model.id, model.name);
    isModuleChat = true;
}
```

Lines 2286-2296 — Module-specific title:
```typescript
if (isModuleChat) {
    const moduleName = model?.name || $selectedFolder?.name || 'Chat';
    const date = new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
    chatTitle = `${moduleName} - ${date}`;
}
```

Line 2311 — Passes `folderId` to `createNewChat()`:
```typescript
chat = await createNewChat(localStorage.token, { ... }, folderId);
```

Line 2324 — Resets selected folder:
```typescript
selectedFolder.set(null);
```

**Also at line 1988-1995** — Disables auto-title for modules:
```typescript
background_tasks: {
    title_generation: model?.info?.meta?.youlab_module
        ? false
        : ($settings?.title?.auto ?? true),
}
```

**Removal**:
- Remove `hasYoulabModule`, `isModuleChat`, and `folderId` logic from `initChatHandler()`
- Remove the `ensureModuleFolder` import
- Change `createNewChat()` call to always pass `null` for folderId
- Remove module-specific title generation (let normal title gen work)
- Remove auto-title disable for modules (line 1988-1995)
- Remove `selectedFolder.set(null)` cleanup
- Remove `selectedFolder` subscription at lines 636-645 that sets `selectedModels` from folder data

### 3. `src/lib/components/layout/Sidebar/ModuleItem.svelte` — Module Click Handler

**`handleModuleClick()` (lines 28-52):**
```typescript
const folderId = await ensureModuleFolder(localStorage.token, module.id, module.name);
const folder = await getFolderById(localStorage.token, folderId);
if (folder) { selectedFolder.set(folder); }
const threadId = await getMostRecentThreadInFolder(localStorage.token, folderId);
if (threadId) { goto(`/c/${threadId}`); }
else { goto(`/?model=${module.id}`); }
```

**Removal**: Simplify to just navigate to a new chat with the module model:
```typescript
goto(`/?model=${module.id}`);
```
No folder creation, no folder selection, no thread lookup.

### 4. `src/lib/apis/chats/index.ts` — Chat Creation API

**`createNewChat()` (lines 4-34):**
- Accepts `folderId` as third parameter
- Sends `folder_id: folderId ?? null` in request body (line 16)

**`updateChatFolderIdById()` (lines 808-841):**
- POST to `/chats/${id}/folder` with `{ folder_id: folderId }`
- Used for manual folder moves (not just module auto-assignment)

**Removal**: Either remove `folderId` parameter from `createNewChat()` entirely, or just stop passing it. `updateChatFolderIdById()` can stay since it's used for general folder features.

### 5. `src/lib/stores/index.ts` — selectedFolder Store

**Line ~63:**
```typescript
export const selectedFolder = writable(null);
```

This store bridges module click → chat creation. It's set in `ModuleItem.handleModuleClick()` and consumed in `Chat.initChatHandler()`.

**Removal**: If the general folder UI still uses `selectedFolder` for non-module purposes, keep it. If only used for module auto-folder, remove it.

### 6. `src/lib/apis/index.ts` — Type Definitions

**`YouLabModuleMeta` (lines 1667-1676):**
```typescript
export interface YouLabModuleMeta {
  course_id: string;
  module_index: number;
  status?: 'locked' | 'available' | 'in_progress' | 'completed';
  welcome_message?: string;
  unlock_criteria?: { previous_module?: string; min_interactions?: number; };
}
```

**Keep this** — it's used for module display in `ModuleList.svelte`, not just folders.

### 7. Backend (No Changes Needed)

**OpenWebUI backend** (`backend/open_webui/routers/folders.py`, `models/folders.py`, `routers/chats.py`):
- Standard OpenWebUI folder system — not customized for YouLab
- `POST /chats/new` accepts optional `folder_id` — just don't send it
- No backend changes needed

**Ralph backend** (`src/ralph/`):
- Zero folder references anywhere
- `src/ralph/openwebui_client.py` creates chats without folder_id
- No changes needed

### 8. Legacy Letta Stack (Irrelevant)

`src/youlab_server/` has extensive folder logic for Letta agent folders, but this system is deprecated and unrelated to the module auto-folder behavior in the current UI.

## Removal Plan

### Step 1: Simplify ModuleItem Click Handler
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte`
- Remove `ensureModuleFolder`, `getFolderById`, `getMostRecentThreadInFolder` imports
- Replace `handleModuleClick()` body with just `goto(/?model=${module.id})`

### Step 2: Remove Folder Logic from Chat Creation
**File**: `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte`
- Remove `ensureModuleFolder` import
- In `initChatHandler()`: Remove module detection, folder assignment, module title logic
- Change `createNewChat()` call to pass `null` for folderId
- Remove auto-title disable for modules (line ~1991)
- Optionally remove `selectedFolder` subscription (lines 636-645) if only used for modules

### Step 3: Clean Up Folder Utilities
**File**: `OpenWebUI/open-webui/src/lib/utils/folders.ts`
- Remove `ensureModuleFolder()` and `ensureAgentFolder()` (or keep agent one if used by background tasks)
- Remove `getMostRecentThreadInFolder()` if no longer needed
- Remove cache if nothing uses it

### Step 4: Optionally Clean createNewChat Signature
**File**: `OpenWebUI/open-webui/src/lib/apis/chats/index.ts`
- Could remove `folderId` parameter if no other features use it
- Or just keep it and never pass a value from module flow

## Code References

- `OpenWebUI/open-webui/src/lib/utils/folders.ts:22-54` — `ensureModuleFolder()`
- `OpenWebUI/open-webui/src/lib/utils/folders.ts:64-93` — `ensureAgentFolder()`
- `OpenWebUI/open-webui/src/lib/utils/folders.ts:102-112` — `getMostRecentThreadInFolder()`
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:2270-2332` — `initChatHandler()` with folder logic
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:1988-1995` — Auto-title disable for modules
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:636-645` — `selectedFolder` subscription
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte:28-52` — `handleModuleClick()`
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleList.svelte:55-70` — Module filtering
- `OpenWebUI/open-webui/src/lib/apis/chats/index.ts:4-34` — `createNewChat()` with folderId
- `OpenWebUI/open-webui/src/lib/apis/index.ts:1667-1684` — `YouLabModuleMeta` interface
- `OpenWebUI/open-webui/src/lib/stores/index.ts:63` — `selectedFolder` store

## Open Questions

1. **Should we keep the module-specific title format** (`"Module Name - Feb 12, 2026"`) even without folders? Or let normal auto-title generate titles?
2. **Should ModuleItem clicking still open the most recent chat** for that model, or always create a new chat? (Currently it opens the most recent chat in the folder)
3. **Should `ensureAgentFolder()`** stay for background agent task threads, or should that be removed too?
4. **What about existing chats** already in module folders? Should they be moved out, or left as-is?
5. **General folder feature**: The standard OpenWebUI folder UI (drag-to-folder, folder sidebar) should remain untouched — only the _automatic_ module-to-folder assignment is being removed. Correct?
