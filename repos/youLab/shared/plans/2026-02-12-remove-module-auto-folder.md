# Remove Module Auto-Folder Logic

## Overview

Remove the automatic folder creation/assignment for module chats in the OpenWebUI fork. Module chats should behave like normal chats (no auto-folder), but keep the custom title format and the "resume most recent chat" behavior via title search instead of folder lookup.

## Current State Analysis

The module auto-folder system lives entirely in the OpenWebUI fork frontend (`OpenWebUI/open-webui/src/`). The Ralph backend has zero folder code. Three files contain YouLab-custom folder logic:

1. **`src/lib/utils/folders.ts`** — Entire file is YouLab custom. Contains `ensureModuleFolder()`, `ensureAgentFolder()`, `getMostRecentThreadInFolder()`, `clearFolderCache()`.
2. **`src/lib/components/chat/Chat.svelte`** — Custom logic in `initChatHandler()` (lines 2243-2253) that detects module models and auto-assigns folders. Also disables auto-title for modules (line 1959-1962).
3. **`src/lib/components/layout/Sidebar/ModuleItem.svelte`** — **Already updated** to use search-based chat resume instead of folders. No changes needed.

### Key Discoveries:
- `ModuleItem.svelte` has already been migrated away from folder logic — it uses `getChatListBySearchText()` to find the most recent chat by title prefix matching (`module.name + ' - '`).
- `ensureModuleFolder` is called at `Chat.svelte:2251` but **never imported** — this is a runtime error waiting to happen.
- `selectedFolder` store (line 43 import, lines 936-937, 2246, 2249, 2258, 2293, 2405-2408, 2487) is base OpenWebUI — used by general folder UI, not just modules.
- `createNewChat` in `src/lib/apis/chats/index.ts` takes `folderId` as a base OpenWebUI parameter — no changes needed.
- `moveChatHandler` (Chat.svelte:2346) is base OpenWebUI drag-to-folder — keep it.

## Desired End State

- Clicking a module in sidebar → searches for most recent chat by title → resumes it OR navigates to new chat page with correct model selected (already works via ModuleItem.svelte)
- Creating a new module chat → no folder assignment, but keeps custom title format (`"Module Name - Feb 12, 2026"`)
- Auto-title generation stays disabled for module chats (they already have the custom title)
- `src/lib/utils/folders.ts` is deleted entirely (all functions are YouLab custom)
- No base OpenWebUI code is modified (only YouLab additions removed)
- Existing chats in folders left as-is

### Verification:
1. Click a module in sidebar → goes to most recent chat or new chat page with model pre-selected
2. Send first message in a new module chat → chat is created WITHOUT a folder_id, with title "Module Name - Feb 12, 2026"
3. Auto-title does NOT overwrite the custom module title
4. General folder features (drag-to-folder, folder sidebar) still work
5. No console errors about missing `ensureModuleFolder`

## What We're NOT Doing

- Not changing the backend (Ralph or OpenWebUI)
- Not modifying existing chats already in folders
- Not removing the general OpenWebUI folder system
- Not changing `selectedFolder` store (it's base OpenWebUI)
- Not changing `createNewChat` API signature (it's base OpenWebUI)
- Not removing `ModuleList.svelte` or `ModuleItem.svelte` (they stay)

## Implementation Approach

3 files to change, 1 file to delete. All changes are in the OpenWebUI fork at `OpenWebUI/open-webui/`.

## Phase 1: Delete `src/lib/utils/folders.ts`

### Overview
Remove the entire file. All 4 exported functions are YouLab custom code, none are used anywhere except the broken (unimported) reference in Chat.svelte which we fix in Phase 2.

### Changes Required:

#### 1. Delete file
**File**: `src/lib/utils/folders.ts`
**Action**: Delete entirely

### Success Criteria:

#### Automated Verification:
- [x] File no longer exists: `! test -f src/lib/utils/folders.ts`
- [x] No remaining imports of the file: `grep -r "utils/folders" src/` returns nothing

---

## Phase 2: Remove folder logic from `Chat.svelte` `initChatHandler()`

### Overview
Remove the module detection + folder assignment block, but keep the custom title generation. Remove the broken `ensureModuleFolder` call.

### Changes Required:

#### 1. `src/lib/components/chat/Chat.svelte` — `initChatHandler()` (lines 2239-2301)

**Current code** (lines 2242-2293):
```typescript
if (!$temporaryChatEnabled) {
    // Check if this is a module chat that needs special handling
    const model = $models.find((m) => m.id === selectedModels[0]);
    const hasYoulabModule = model?.info?.meta?.youlab_module;
    let isModuleChat = $selectedFolder?.meta?.type === 'module' || hasYoulabModule;

    // If model has youlab_module but selectedFolder isn't set, ensure the folder exists
    let folderId = $selectedFolder?.id;
    if (hasYoulabModule && !folderId) {
        folderId = await ensureModuleFolder(localStorage.token, model.id, model.name);
        isModuleChat = true;
    }

    // Generate title - use module name + date for module chats
    let chatTitle = $i18n.t('New Chat');
    if (isModuleChat) {
        const moduleName = model?.name || $selectedFolder?.name || 'Chat';
        const date = new Date().toLocaleDateString('en-US', {
            month: 'short',
            day: 'numeric',
            year: 'numeric'
        });
        chatTitle = `${moduleName} - ${date}`;
    }

    chat = await createNewChat(
        localStorage.token,
        {
            id: _chatId,
            title: chatTitle,
            ...
        },
        folderId
    );
    ...
    selectedFolder.set(null);
```

**New code**:
```typescript
if (!$temporaryChatEnabled) {
    // Check if this is a YouLab module chat (for custom title)
    const model = $models.find((m) => m.id === selectedModels[0]);
    const hasYoulabModule = model?.info?.meta?.youlab_module;

    // Generate title - use module name + date for module chats
    let chatTitle = $i18n.t('New Chat');
    if (hasYoulabModule) {
        const moduleName = model?.name || 'Chat';
        const date = new Date().toLocaleDateString('en-US', {
            month: 'short',
            day: 'numeric',
            year: 'numeric'
        });
        chatTitle = `${moduleName} - ${date}`;
    }

    chat = await createNewChat(
        localStorage.token,
        {
            id: _chatId,
            title: chatTitle,
            ...
        },
        $selectedFolder?.id ?? null
    );
    ...
    selectedFolder.set(null);
```

**What changed**:
- Removed `isModuleChat` variable (replaced with direct `hasYoulabModule` check)
- Removed `folderId` variable and `ensureModuleFolder()` call
- Removed `$selectedFolder?.meta?.type === 'module'` check
- Changed `folderId` in `createNewChat()` to `$selectedFolder?.id ?? null` (preserves base OpenWebUI folder behavior for non-module flows)
- Removed `$selectedFolder?.name` fallback in title (modules always have a model name)
- Kept custom title format for module chats
- Kept `selectedFolder.set(null)` (base OpenWebUI cleanup)

#### 2. `src/lib/components/chat/Chat.svelte` — Keep auto-title disable (lines 1959-1962)

**Current code** (keep as-is):
```typescript
// Disable auto title for youlab modules (they use custom titles)
title_generation: model?.info?.meta?.youlab_module
    ? false
    : ($settings?.title?.auto ?? true),
```

This is correct — module chats already get a custom title, so auto-title should stay disabled.

### Success Criteria:

#### Automated Verification:
- [x] No references to `ensureModuleFolder` in Chat.svelte: `grep ensureModuleFolder Chat.svelte` returns nothing
- [x] No references to `ensureAgentFolder` anywhere: `grep -r ensureAgentFolder src/` returns nothing
- [x] Build succeeds: `npm run build` (from OpenWebUI directory)

#### Manual Verification:
- [ ] Click a module → navigates to most recent chat or new chat page with correct model
- [ ] Send first message as new module chat → chat created without folder, with custom title "Module Name - Feb 12, 2026"
- [ ] Regular (non-module) new chat still works normally
- [ ] Drag-and-drop chat to folder still works
- [ ] No console errors

---

## Phase 3: Verify no remaining references

### Overview
Ensure there are no dangling imports or references to the deleted utilities.

### Changes Required:

#### 1. Search and clean
```bash
# From OpenWebUI/open-webui/
grep -r "ensureModuleFolder\|ensureAgentFolder\|getMostRecentThreadInFolder\|clearFolderCache\|utils/folders" src/
```

If any results found, remove the imports/references.

### Success Criteria:

#### Automated Verification:
- [x] Zero results from the grep above
- [x] `npm run build` succeeds
- [x] `npm run check` — 8009 errors pre-existing (baseline upstream OpenWebUI). No new errors introduced. Build passes.

## Testing Strategy

### Manual Testing Steps:
1. Click a module in sidebar with existing chats → should navigate to most recent chat
2. Click a module with no existing chats → should go to `/?model=<module-id>` (new chat page with module selected)
3. Send a message on the new chat page → chat title should be "Module Name - Feb 12, 2026"
4. Verify the chat is NOT in any folder (check sidebar — it should appear in the main chat list, not inside a folder)
5. Create a regular (non-module) chat → should work exactly as before
6. Drag a chat into a folder → should still work
7. Check browser console for errors during all the above

## References

- Research doc: `thoughts/shared/research/2026-02-12-module-auto-folder-logic.md`
- `OpenWebUI/open-webui/src/lib/utils/folders.ts` — File to delete (entire file is YouLab custom)
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:2239-2301` — `initChatHandler()` with folder logic to remove
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:1959-1962` — Auto-title disable (keep)
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte:26-43` — Already uses search-based resume (no changes)
