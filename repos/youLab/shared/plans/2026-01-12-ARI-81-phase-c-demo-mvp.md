# ARI-81 Phase C: Demo-Ready MVP Implementation Plan

## Overview

This plan completes the YouLab MVP by polishing frontend components, wiring modules to real data, implementing chat organization via folders, and connecting the diff approval UI to the backend. The end result is a demo-ready application where a user can complete a tutoring session and manage their memory profile.

## Current State Analysis

**Completed:**
- ARI-79: YouLab branding, iA Writer theme, Modules sidebar, hidden sections
- ARI-80 Phases 1-5: Git storage, TOML↔MD conversion, user init, UserBlockManager, CRUD API
- ARI-80 Phases 6-7: "You" page with Profile/Agents tabs, block cards, detail modal

**Issues Identified:**
1. Favicon has dark background fill (should be transparent)
2. BlockCard layout breaks with two-line titles, badge truncates
3. Profile/Agents tabs don't match Workspace toolbar pattern
4. Modules show demo data, clicking shows "Pull from Ollama.com" error
5. No chat organization - chats should auto-organize into module/agent folders
6. Diff approve/reject not wired to backend

## Desired End State

A first-time user can:
1. Sign up and land on YouLab with clean branding
2. See their module in the sidebar, click to start a chat
3. Chat is auto-organized into a module folder with proper naming
4. Click "You" to see memory blocks (Profile tab)
5. See pending diffs with approve/reject buttons that work
6. View background agent threads (Agents tab)

### How to Verify
- Visual inspection matches design spec
- Clicking module starts chat (no Ollama error)
- Chats appear in folders named after modules
- Approve/reject on diffs updates the memory block

## What We're NOT Doing

- Agent-speaks-first (welcome messages) - deferred
- Module catalog view (all modules) - deferred
- Real-time notifications/Socket.IO - deferred
- Background agent scheduler - deferred (manual trigger only)
- Per-user module progression - status is static metadata for now

## Implementation Approach

Five phases, ordered by dependency:
1. **Frontend Polish** - Favicon, BlockCard, toolbar consistency
2. **Module Integration** - Real module, remove demo data
3. **Chat Folders** - Auto-create folders, auto-assign chats
4. **Diff Approval UI** - Wire approve/reject to backend
5. **Integration Testing** - End-to-end verification

---

## Phase 1: Frontend Polish

### Overview
Fix visual issues: transparent favicon, BlockCard layout matching Knowledge pattern, consistent toolbar.

### Changes Required:

#### 1.1 Fix Favicon - Transparent Background
**File**: `OpenWebUI/open-webui/backend/open_webui/static/favicon.svg`
**Changes**: Remove the background rectangle, keep only the Y path

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <!-- Y letter as path - no background -->
  <path d="M16 12 L32 32 L32 52 L32 32 L48 12"
        stroke="#15BDEC"
        stroke-width="8"
        stroke-linecap="round"
        stroke-linejoin="round"
        fill="none"/>
</svg>
```

**Also update PNG versions** by regenerating from SVG:
- `backend/open_webui/static/favicon.png`
- `backend/open_webui/static/favicon-96x96.png`
- `backend/open_webui/static/favicon-dark.png`
- `static/favicon.png` (copy from backend)

Use ImageMagick or similar:
```bash
cd OpenWebUI/open-webui/backend/open_webui/static
convert -background none favicon.svg -resize 32x32 favicon.png
convert -background none favicon.svg -resize 96x96 favicon-96x96.png
cp favicon.png favicon-dark.png
```

#### 1.2 Restyle BlockCard to Match Knowledge Collections
**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockCard.svelte`
**Changes**: Match the Knowledge collection card pattern exactly

Reference pattern from `Knowledge.svelte:209-281`:
- Card: `flex space-x-4 px-3 py-2.5 hover:bg-gray-50 dark:hover:bg-gray-850/50 rounded-2xl`
- Badge row: `flex items-center justify-between -my-1 h-8`
- Name: `text-sm font-medium line-clamp-1 capitalize`

```svelte
<script lang="ts">
	import { createEventDispatcher, getContext } from 'svelte';
	import Badge from '$lib/components/common/Badge.svelte';
	import Document from '$lib/components/icons/Document.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let label: string;
	export let pendingDiffs: number = 0;

	// Convert snake_case to Title Case
	$: displayName = label
		.replace(/_/g, ' ')
		.replace(/\b\w/g, (c) => c.toUpperCase());
</script>

<button
	class="flex space-x-4 cursor-pointer text-left w-full px-3 py-2.5
		   hover:bg-gray-50 dark:hover:bg-gray-850/50 transition rounded-2xl"
	on:click={() => dispatch('click')}
>
	<div class="flex flex-col flex-1 min-w-0">
		<!-- Badge row - fixed height -->
		<div class="flex items-center justify-between h-6 mb-1">
			<Badge type="muted" content={$i18n.t('Memory Block')} />
			{#if pendingDiffs > 0}
				<Badge type="warning" content="{pendingDiffs} {$i18n.t('pending')}" />
			{/if}
		</div>

		<!-- Content row -->
		<div class="flex items-center gap-2">
			<div class="p-1.5 rounded-lg bg-gray-100 dark:bg-gray-800 shrink-0">
				<Document className="size-4 text-gray-600 dark:text-gray-400" />
			</div>
			<span class="text-sm font-medium line-clamp-1 text-gray-900 dark:text-gray-100">
				{displayName}
			</span>
		</div>
	</div>
</button>
```

#### 1.3 Update You Page Grid to Match Knowledge
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`
**Changes**: Update grid container styling

```svelte
<!-- Around line 186-195, replace grid with Knowledge pattern -->
<div class="py-2 bg-white dark:bg-gray-900 rounded-3xl border border-gray-100/30 dark:border-gray-850/30">
	<div class="my-2 px-3 grid grid-cols-1 lg:grid-cols-2 gap-2">
		{#each $memoryBlocks as block (block.label)}
			<BlockCard
				label={block.label}
				pendingDiffs={block.pendingDiffs}
				on:click={() => openBlock(block.label)}
			/>
		{/each}
	</div>
</div>
```

#### 1.4 Update Profile/Agents Toolbar to Match Workspace
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`
**Changes**: Replace custom button tabs with Workspace-style anchor links

The Workspace pattern uses simple `<a>` tags with route-based active detection.
For the You page (single route with tabs), we'll use the same styling but with buttons/state:

```svelte
<!-- Replace tab bar around line 138-172 with Workspace pattern -->
<div class="flex gap-1 scrollbar-none overflow-x-auto w-fit text-center text-sm font-medium rounded-full bg-transparent py-1 touch-auto pointer-events-auto">
	<button
		class="min-w-fit p-1.5 {selectedTab === 'profile'
			? ''
			: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
		on:click={() => (selectedTab = 'profile')}
	>
		{$i18n.t('Profile')}
	</button>
	<button
		class="min-w-fit p-1.5 {selectedTab === 'agents'
			? ''
			: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
		on:click={() => (selectedTab = 'agents')}
	>
		{$i18n.t('Agents')}
	</button>
</div>
```

### Success Criteria:

#### Automated Verification:
- [x] SVG is valid: `xmllint --noout OpenWebUI/open-webui/backend/open_webui/static/favicon.svg`
- [x] PNG files exist: `ls OpenWebUI/open-webui/backend/open_webui/static/favicon*.png`
- [x] Frontend builds: `cd OpenWebUI/open-webui && npm run build`

#### Manual Verification:
- [x] Favicon shows Y on transparent background in browser tab
- [x] BlockCard with two-line title displays correctly (no truncated badge)
- [x] All block cards are equal height in grid
- [x] Profile/Agents tabs match Workspace toolbar styling
- [x] Hover states work correctly on tabs

---

## Phase 2: Module Integration

### Overview
Wire the Modules sidebar to display real course modules instead of demo data. Create an actual OpenWebUI model with `youlab_module` metadata.

### Changes Required:

#### 2.1 Create College Essay Module Model ✓
**Method**: `hack/seed-module.sh` script (created)

Create a model in OpenWebUI with:
- **ID**: `college-essay-intro` (must match what ModuleList expects)
- **Name**: `First Impressions`
- **Base Model**: `youlab-letta-pipe` (the pipe that routes to Letta)
- **Meta**:
```json
{
  "description": "Learn what makes a compelling college essay opening",
  "profile_image_url": "/static/favicon.png",
  "youlab_module": {
    "course_id": "college-essay",
    "module_index": 0,
    "status": "available"
  }
}
```

**Via API** (if programmatic creation needed):
```bash
curl -X POST "${WEBUI_API_BASE_URL}/api/models/add" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "college-essay-intro",
    "name": "First Impressions",
    "base_model_id": "youlab-letta-pipe",
    "meta": {
      "description": "Learn what makes a compelling college essay opening",
      "youlab_module": {
        "course_id": "college-essay",
        "module_index": 0,
        "status": "available"
      }
    }
  }'
```

#### 2.2 Remove Demo Module Fallback (Optional) ✓
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleList.svelte`
**Changes**: Updated to show demo modules only in dev mode (`import.meta.env.DEV`)

```svelte
<!-- Around line 70, change fallback behavior -->
$: modules = realModules.length > 0
	? realModules
	: []; // Empty instead of demo modules, or keep for dev
```

Or add environment check:
```svelte
$: modules = realModules.length > 0
	? realModules
	: (import.meta.env.DEV ? DEMO_MODULES : []);
```

### Success Criteria:

#### Automated Verification:
- [x] Model exists: `curl ${WEBUI_API_BASE_URL}/api/v1/models/model?id=college-essay-intro`

#### Manual Verification:
- [x] "First Impressions" module appears in Modules sidebar section
- [x] Clicking module opens chat (no "Pull from Ollama.com" error)
- [x] Module shows correct name and description
- [x] Chat functions with the configured base model/pipe

---

## Phase 3: Chat Folder Organization

### Overview
Implement automatic chat organization where:
- Each module and background agent has a dedicated folder
- ALL threads (active + archived) live in their respective folders
- Clicking a module in sidebar navigates to the most recent thread
- "New Chat" with same module archives current thread, creates new one in same folder
- Background agents are NOT selectable as models - only visible in You > Agents

### Thread Lifecycle

```
Folder: "First Impressions"
├─ First Impressions - Jan 12, 2026  ← active (most recent)
├─ First Impressions - Jan 10, 2026  ← archived
└─ First Impressions - Jan 8, 2026   ← archived

User clicks "First Impressions" module → opens "Jan 12" thread
User clicks "New Chat" → "Jan 12" becomes archived, new "Jan 13" created
```

### Changes Required:

#### 3.1 Create Module/Agent Folders Utility
**File**: `OpenWebUI/open-webui/src/lib/utils/folders.ts` (NEW)
**Changes**: Utility functions for folder management

```typescript
import { createNewFolder, getFolders } from '$lib/apis/folders';
import { getChatListByFolderId } from '$lib/apis/chats';

// Cache of folder IDs by name
let folderCache: Record<string, string> = {};

export async function ensureModuleFolder(
	token: string,
	moduleId: string,
	moduleName: string
): Promise<string> {
	const folderName = moduleName; // No emoji prefix

	// Check cache first
	if (folderCache[folderName]) {
		return folderCache[folderName];
	}

	// Fetch existing folders
	const folders = await getFolders(token);
	const existing = folders.find(f => f.name === folderName);

	if (existing) {
		folderCache[folderName] = existing.id;
		return existing.id;
	}

	// Create new folder
	const newFolder = await createNewFolder(token, {
		name: folderName,
		meta: {
			type: 'module',
			moduleId: moduleId
		}
	});

	folderCache[folderName] = newFolder.id;
	return newFolder.id;
}

export async function ensureAgentFolder(
	token: string,
	agentId: string,
	agentName: string
): Promise<string> {
	const folderName = agentName; // No emoji prefix

	if (folderCache[folderName]) {
		return folderCache[folderName];
	}

	const folders = await getFolders(token);
	const existing = folders.find(f => f.name === folderName);

	if (existing) {
		folderCache[folderName] = existing.id;
		return existing.id;
	}

	const newFolder = await createNewFolder(token, {
		name: folderName,
		meta: {
			type: 'background_agent',
			agentId: agentId
		}
	});

	folderCache[folderName] = newFolder.id;
	return newFolder.id;
}

export async function getMostRecentThreadInFolder(
	token: string,
	folderId: string
): Promise<string | null> {
	// Get chats in folder, sorted by updated_at desc
	const chats = await getChatListByFolderId(token, folderId);
	if (chats && chats.length > 0) {
		return chats[0].id; // Most recent
	}
	return null;
}

export function clearFolderCache() {
	folderCache = {};
}
```

#### 3.2 Module Click → Navigate to Most Recent Thread
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte`
**Changes**: Override navigation to go to most recent thread in module's folder

```svelte
<script lang="ts">
	import { goto } from '$app/navigation';
	import { ensureModuleFolder, getMostRecentThreadInFolder } from '$lib/utils/folders';

	export let module: { id: string; name: string; /* ... */ };
	export let onClick = () => {};

	async function handleModuleClick(e: Event) {
		e.preventDefault();

		// Ensure folder exists
		const folderId = await ensureModuleFolder(
			localStorage.token,
			module.id,
			module.name
		);

		// Find most recent thread
		const threadId = await getMostRecentThreadInFolder(localStorage.token, folderId);

		if (threadId) {
			// Navigate to existing thread
			goto(`/c/${threadId}`);
		} else {
			// No existing thread - create new chat with this model
			goto(`/?model=${module.id}`);
		}

		onClick();
	}
</script>

<a
	class="..."
	href="/?model={module.id}"
	on:click={handleModuleClick}
>
	<!-- ... -->
</a>
```

#### 3.3 New Chat → Archive Current, Create New in Folder
**File**: `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte`
**Changes**: When creating new chat with a module, handle folder assignment and archiving

```typescript
import { ensureModuleFolder } from '$lib/utils/folders';
import { updateChatFolderIdById, archiveChatById, getChatListByFolderId } from '$lib/apis/chats';

// When initializing a new chat with a module model
async function initModuleChat(model: Model) {
	if (!model?.info?.meta?.youlab_module) return;

	// Ensure folder exists
	const folderId = await ensureModuleFolder(
		localStorage.token,
		model.id,
		model.name
	);

	// Archive the current active thread (most recent non-archived)
	const existingChats = await getChatListByFolderId(localStorage.token, folderId);
	const activeChat = existingChats?.find(c => !c.archived);
	if (activeChat) {
		await archiveChatById(localStorage.token, activeChat.id);
	}

	// Assign new chat to folder with date-based title
	const date = new Date().toLocaleDateString('en-US', {
		month: 'short',
		day: 'numeric',
		year: 'numeric'
	});
	const title = `${model.name} - ${date}`;

	await updateChatFolderIdById(localStorage.token, $chatId, folderId);
	await updateChatById(localStorage.token, $chatId, { title });
}
```

#### 3.4 Background Agent Thread Creation (Backend)
**File**: `src/youlab_server/server/agents_threads.py`
**Changes**: When background agent runs, create thread in agent's folder

```python
from datetime import datetime

async def create_agent_thread(
    user_id: str,
    agent_id: str,
    agent_name: str,
    openwebui_client: OpenWebUIClient
) -> str:
    """Create a new chat thread for a background agent run.

    Archives the previous thread and creates a new one in the agent's folder.
    """
    # Ensure folder exists (no emoji)
    folder_name = agent_name
    folder_id = await openwebui_client.ensure_folder(
        user_id,
        folder_name,
        meta={"type": "background_agent", "agent_id": agent_id}
    )

    # Archive previous active thread in this folder
    existing_chats = await openwebui_client.get_chats_by_folder(user_id, folder_id)
    for chat in existing_chats:
        if not chat.get("archived"):
            await openwebui_client.archive_chat(chat["id"])

    # Create new thread
    chat = await openwebui_client.create_chat(user_id)

    # Assign to folder
    await openwebui_client.update_chat_folder(chat["id"], folder_id)

    # Name with date
    date_str = datetime.now().strftime("%b %d, %Y")
    title = f"{agent_name} - {date_str}"
    await openwebui_client.update_chat_title(chat["id"], title)

    return chat["id"]
```

#### 3.5 Hide Background Agents from Model Selector
**File**: `OpenWebUI/open-webui/src/lib/components/chat/ModelSelector.svelte` (or equivalent)
**Changes**: Filter out models with `meta.type === 'background_agent'`

```typescript
// In the model filtering logic
$: availableModels = $models.filter(model => {
	// Hide background agents from model selector
	if (model.info?.meta?.type === 'background_agent') {
		return false;
	}
	return true;
});
```

### Success Criteria:

#### Automated Verification:
- [x] Utility file exists: `ls OpenWebUI/open-webui/src/lib/utils/folders.ts`
- [x] Frontend builds: `cd OpenWebUI/open-webui && npm run build`

#### Manual Verification:
- [x] Clicking module navigates to most recent thread (or creates new if none)
- [x] Folder named "{Module Name}" (no emoji) contains all module threads
- [x] Multiple chats visible in folders (archiving removed)
- [x] All threads in folder named "{Module/Agent} - {Date}"
- [x] Folder renaming works correctly
- [ ] Background agents NOT visible in model selector
- [ ] Background agent threads appear in "{Agent Name}" folders

### Known Issues (Deferred):

#### Chat Title Renaming Not Persisting
**Status**: Not fixed - deferred for future investigation
**Symptom**: Renaming a chat title in the sidebar appears to work momentarily, then reverts when clicking elsewhere
**Root Cause**: OpenWebUI's aggressive state refresh cycle in `Folders.svelte` overwrites local changes. The reactive block `$: if (folders || ($selectedFolder && $chatId))` triggers `loadFolderItems()` on every navigation, fetching data that overwrites the rename.
**Attempted Fixes**:
1. Added `setFolderItems()` call in RecursiveFolder's on:change handler - didn't persist
2. Made handler async with await - still reverted
3. Removed `$chatId` from reactive block dependency - broke initial folder loading
4. Added 200ms delay before refresh - still didn't work
5. Modified Sidebar's on:change to skip `initFolders()` - didn't help

**Workaround**: Users cannot rename chat titles in folders for now. Titles are auto-generated as "{Module Name} - {Date}" which is acceptable for MVP.
**Future Fix**: Requires deeper investigation into OpenWebUI's state management, possibly updating local state optimistically without refetching, or finding where stale data is being cached.

---

## Phase 4: Diff Approval UI ✓

### Overview
Wire the approve/reject buttons in BlockDetailModal to call the backend endpoints.

**Implementation Note:** Redesigned to use GitHub-style unified diff view. When there are pending diffs, the modal shows the entire content with inline red/green diff highlighting. Approve/Reject buttons appear inline after each diff's addition line. Users can toggle to "Edit" mode for the regular textarea.

### Changes Required:

#### 4.1 Add Approve/Reject API Functions
**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`
**Changes**: Add functions to approve and reject diffs

```typescript
export async function approveDiff(
	userId: string,
	label: string,
	diffId: string,
	token: string
): Promise<boolean> {
	const res = await fetch(
		`${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}/diffs/${diffId}/approve`,
		{
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				Authorization: `Bearer ${token}`
			}
		}
	);

	if (!res.ok) {
		const error = await res.json();
		throw new Error(error.detail || 'Failed to approve diff');
	}

	return true;
}

export async function rejectDiff(
	userId: string,
	label: string,
	diffId: string,
	token: string
): Promise<boolean> {
	const res = await fetch(
		`${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}/diffs/${diffId}/reject`,
		{
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				Authorization: `Bearer ${token}`
			}
		}
	);

	if (!res.ok) {
		const error = await res.json();
		throw new Error(error.detail || 'Failed to reject diff');
	}

	return true;
}
```

#### 4.2 Wire BlockDetailModal Approve/Reject Buttons
**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`
**Changes**: Add click handlers that call the API

```svelte
<script lang="ts">
	// Add imports
	import { approveDiff, rejectDiff } from '$lib/apis/memory';
	import { user } from '$lib/stores';
	import { toast } from 'svelte-sonner';

	// Add state
	let processing = false;

	// Add handlers
	async function handleApprove(diffId: string) {
		if (processing) return;
		processing = true;

		try {
			await approveDiff($user.id, label, diffId, localStorage.token);
			toast.success('Change approved');
			// Refresh the block data
			await loadBlockDetail();
		} catch (e) {
			toast.error(e.message || 'Failed to approve');
		} finally {
			processing = false;
		}
	}

	async function handleReject(diffId: string) {
		if (processing) return;
		processing = true;

		try {
			await rejectDiff($user.id, label, diffId, localStorage.token);
			toast.success('Change rejected');
			await loadBlockDetail();
		} catch (e) {
			toast.error(e.message || 'Failed to reject');
		} finally {
			processing = false;
		}
	}
</script>

<!-- In the pending diffs section, add click handlers -->
{#each pendingDiffs as diff}
	<div class="...">
		<!-- Diff display -->
		<div class="flex gap-2 mt-2">
			<button
				class="px-3 py-1.5 bg-green-500 hover:bg-green-600 text-white rounded-lg text-sm transition disabled:opacity-50"
				on:click={() => handleApprove(diff.id)}
				disabled={processing}
			>
				{$i18n.t('Approve')}
			</button>
			<button
				class="px-3 py-1.5 bg-red-500 hover:bg-red-600 text-white rounded-lg text-sm transition disabled:opacity-50"
				on:click={() => handleReject(diff.id)}
				disabled={processing}
			>
				{$i18n.t('Reject')}
			</button>
		</div>
	</div>
{/each}
```

#### 4.3 Verify Backend Endpoints Exist
**File**: `src/youlab_server/server/blocks.py`
**Check**: Ensure approve/reject endpoints are implemented

The endpoints should be:
- `POST /users/{user_id}/blocks/{label}/diffs/{diff_id}/approve`
- `POST /users/{user_id}/blocks/{label}/diffs/{diff_id}/reject`

If not implemented, add them:

```python
@router.post("/users/{user_id}/blocks/{label}/diffs/{diff_id}/approve")
async def approve_diff(
    user_id: str,
    label: str,
    diff_id: str,
    user_block_manager: UserBlockManager = Depends(get_user_block_manager)
) -> dict:
    """Approve a pending diff and apply it to the block."""
    success = await user_block_manager.approve_diff(user_id, label, diff_id)
    if not success:
        raise HTTPException(status_code=404, detail="Diff not found")
    return {"status": "approved"}

@router.post("/users/{user_id}/blocks/{label}/diffs/{diff_id}/reject")
async def reject_diff(
    user_id: str,
    label: str,
    diff_id: str,
    user_block_manager: UserBlockManager = Depends(get_user_block_manager)
) -> dict:
    """Reject a pending diff."""
    success = await user_block_manager.reject_diff(user_id, label, diff_id)
    if not success:
        raise HTTPException(status_code=404, detail="Diff not found")
    return {"status": "rejected"}
```

### Success Criteria:

#### Automated Verification:
- [x] API functions exist: `grep -n "approveDiff\|rejectDiff" OpenWebUI/open-webui/src/lib/apis/memory/index.ts`
- [x] Backend endpoints work: `curl -X POST ${YOULAB_API_BASE_URL}/users/test/blocks/student/diffs/test/approve`
- [x] Frontend builds: `cd OpenWebUI/open-webui && npm run build`

#### Manual Verification:
- [ ] Open a block with pending diffs
- [ ] Click "Approve" - diff is applied, block content updates
- [ ] Click "Reject" on another diff - diff is removed
- [ ] Toast notifications show success/error states
- [ ] Pending diff count updates after approve/reject

### Known Issues (Blocking - Must Fix):

#### 4.4 Diff View Not Displaying Green/Red Lines
**Status**: NOT WORKING - diffs don't render visually
**Symptom**: When opening a block with pending diffs, the inline diff view (red deletions, green additions) does not appear at all. The modal shows content but no diff highlighting.
**Root Cause**: TBD - may be related to `buildDiffView()` logic, line matching, or the diff data structure from the API.
**Files to investigate**:
- `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte` - `buildDiffView()` function
- API response for `/blocks/{label}/diffs` - verify `oldValue`/`newValue` fields are populated

#### 4.5 Diff View Shows TOML Instead of Markdown
**Status**: NOT WORKING - wrong format displayed
**Symptom**: When viewing diffs, the content is displayed as raw TOML instead of user-friendly Markdown. The edit mode textarea placeholder says "Markdown" but shows TOML content.
**Root Cause**: `BlockDetailModal.svelte` loads `tomlContent` for diff view, but users expect to see Markdown. The conversion flow is broken or the wrong content variable is being used.
**Design Decision Needed**: Should diffs be TOML-based (backend) but displayed as Markdown (frontend)? Or should the entire flow use Markdown?

#### 4.6 Manual Edits Do Not Persist
**Status**: NOT WORKING - save is broken
**Symptom**: When editing a block manually (e.g., adding a line like "Test" to the journey block) and clicking Save, the changes do not persist. The content reverts to the previous state.
**Root Cause**: The modal uses `markdownContent` for editing but there may be:
1. Format mismatch between what's sent and what backend expects
2. TOML↔Markdown conversion issues in `storage/blocks.py`
3. API not receiving the correct payload
**Files to investigate**:
- `BlockDetailModal.svelte:save()` - verify `markdownContent` and format param
- `src/youlab_server/server/blocks.py:update_block()` - verify request handling
- `src/youlab_server/storage/blocks.py:update_block_from_markdown()` - verify conversion

#### 4.7 Approve Diff Deletes Entire Block (FIXED but verify)
**Status**: Fix applied, needs verification after restart
**Symptom**: Clicking "Approve" on a diff would delete all block content except the proposed value.
**Fix Applied**: Updated `approve_diff()` in `storage/blocks.py` to:
1. Match `current_value` in current content and replace only that line
2. Raise error if `current_value` not found (stale diff protection)
3. Support explicit `full_replace` operation for intentional full replacements
**Verification**: Restart server and test approve flow with matching diffs.

---

## Phase 5: Integration Testing

### Overview
End-to-end verification of the complete demo flow.

### Manual Testing Script:

#### Test 1: New User Onboarding
1. Create new user account in OpenWebUI
2. Verify user initialization creates git storage (check `./data/users/{user_id}/`)
3. Verify default memory blocks are created

#### Test 2: Module Chat Flow
1. Log in as test user
2. Click "First Impressions" module in sidebar
3. Verify folder "First Impressions" is created (no emoji)
4. Verify new chat created in that folder
5. Verify chat title is "First Impressions - {today's date}"
6. Send a message, verify Letta agent responds
7. Click "New Chat", select same module
8. Verify previous chat is archived (still in folder)
9. Verify new chat created with new date
10. Click module in sidebar again - should go to most recent thread

#### Test 3: Memory Block Viewing
1. Click "You" in sidebar
2. Verify memory blocks display (Student, Engagement Strategy, Journey)
3. Click a block, verify detail modal opens
4. Verify version history shows

#### Test 4: Diff Approval Flow
1. Manually create a pending diff via API or background agent
2. Navigate to You > Profile
3. Verify badge shows pending count
4. Click block with pending diff
5. Verify diff is displayed
6. Click "Approve" - verify block updates
7. Click another diff's "Reject" - verify diff removed

#### Test 5: Background Agent Threads (if implemented)
1. Trigger a background agent run
2. Verify folder "{Agent Name}" is created (no emoji)
3. Verify thread appears in folder with date naming
4. Navigate to You > Agents tab
5. Verify agent and thread are displayed
6. Trigger agent again - verify previous thread archived, new one created
7. Verify background agent NOT visible in model selector (New Chat)

### Success Criteria:

#### All Tests Pass:
- [ ] New user onboarding works
- [ ] Module chat flow works end-to-end
- [ ] Memory blocks display correctly
- [ ] Diff approval/rejection works
- [ ] (Optional) Background agent threads work

---

## Testing Strategy

### Unit Tests
- `folders.ts` utility functions (mock API calls)
- BlockCard component rendering with various label lengths

### Integration Tests
- API endpoint tests for approve/reject diffs
- Folder creation idempotency

### Manual Testing
- Full demo flow as described in Phase 5

## Performance Considerations

- Folder cache prevents repeated API calls for same folder
- Lazy loading of folder contents (already implemented in OpenWebUI)
- Badge counts fetched with block list (single API call)

## Migration Notes

- No database migration needed
- Existing chats remain in root (no folder) - only new chats get organized
- Users can manually drag old chats into folders if desired

## References

- Parent ticket: ARI-78
- Product spec: `thoughts/shared/plans/2025-01-12-youlab-product-spec.md`
- Phase A plan: `thoughts/shared/plans/2026-01-12-ARI-79-phase-a-ui-modules-foundation.md`
- Phase B plan: `thoughts/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md`
- Knowledge UI pattern: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte:207-281`
- Folder API: `OpenWebUI/open-webui/src/lib/apis/folders/index.ts`

---

*Plan created: 2026-01-12*
