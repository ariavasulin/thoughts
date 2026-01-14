# Workspace Profile Tab Implementation Plan

## Overview

Add 'Profile' as a new tab in `/workspace` (alongside Models, Knowledge, Prompts, Tools) that displays memory blocks using the exact same UI patterns as Knowledge. The current `/you` page will redirect to `/workspace/profile`.

## Current State Analysis

### Existing `/you` Page
- **File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`
- Has Profile and Agents tabs (Agents to be dropped)
- Simple grid of BlockCard components
- Fetches from `/users/{user_id}/blocks` API
- No search, no ViewSelector, no infinite scroll

### Workspace Layout
- **File**: `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte`
- Tabs: Models, Knowledge, Prompts, Tools
- Each tab has permission check (`$user?.permissions?.workspace?.X`)
- Active tab styling via `$page.url.pathname.includes('/workspace/X')`

### Knowledge.svelte (Reference Pattern)
- **File**: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte`
- 328 lines with search, ViewSelector, item grid, infinite scroll
- Uses `searchKnowledgeBases()` API with pagination
- Badge, Tooltip, ItemMenu components for cards

### Memory Blocks API
- **Endpoint**: `GET /users/{user_id}/blocks`
- Returns `BlockSummary[]` with `label` and `pending_diffs` count
- No search/pagination support (client-side only for now)

## Desired End State

After implementation:
1. `/workspace` has 5 tabs: Models, Knowledge, Prompts, Tools, **Profile**
2. Profile tab displays memory blocks in Knowledge-style UI
3. Clicking a block navigates to `/you/blocks/{label}` (existing detail page)
4. `/you` redirects to `/workspace/profile`
5. Search box is visible but non-functional (TODO for later)
6. ViewSelector is functional (for future sharing support)

### Verification:
- Navigate to `/workspace/profile` → see memory blocks grid
- Navigate to `/you` → redirects to `/workspace/profile`
- Click a block → navigates to `/you/blocks/{label}`
- "New Memory Block" button visible but non-functional

## What We're NOT Doing

- Backend search/filter functionality for blocks
- Implementing "New Memory Block" create flow
- Moving `/you/blocks/[label]` route to workspace
- Agents tab (dropped entirely)
- Delete functionality for blocks (out of scope)

## Implementation Approach

Mirror the Knowledge.svelte component structure exactly, adapting for memory blocks:
1. Same header pattern with title + count + "New" button
2. Same search box (visible, non-functional)
3. Same ViewSelector (functional, persisted to localStorage)
4. Same item grid with responsive columns
5. Same loading/empty states
6. Adapt BlockCard to match Knowledge item card styling

---

## Phase 1: Create Profile Route & Component Structure

### Overview
Create the route and component files following the workspace pattern.

### Changes Required:

#### 1. Create Profile Route
**File**: `OpenWebUI/open-webui/src/routes/(app)/workspace/profile/+page.svelte`

```svelte
<script>
	import Profile from '$lib/components/workspace/Profile.svelte';
</script>

<Profile />
```

#### 2. Create Profile Component (Skeleton)
**File**: `OpenWebUI/open-webui/src/lib/components/workspace/Profile.svelte`

```svelte
<script lang="ts">
	import { onMount, getContext } from 'svelte';
	const i18n = getContext('i18n');

	import { WEBUI_NAME, user } from '$lib/stores';
	import { memoryBlocks } from '$lib/stores/memory';
	import { getBlocks } from '$lib/apis/memory';

	import Badge from '../common/Badge.svelte';
	import Search from '../icons/Search.svelte';
	import Plus from '../icons/Plus.svelte';
	import Spinner from '../common/Spinner.svelte';
	import Tooltip from '../common/Tooltip.svelte';
	import XMark from '../icons/XMark.svelte';
	import ViewSelector from './common/ViewSelector.svelte';

	let loaded = false;
	let loading = false;

	let query = '';
	let viewOption = '';

	let items: Array<{ label: string; pendingDiffs: number }> | null = null;

	onMount(async () => {
		viewOption = localStorage?.workspaceViewOption || '';
		loaded = true;
		await loadBlocks();
	});

	async function loadBlocks() {
		if (!$user) return;

		loading = true;
		try {
			const blocks = await getBlocks($user.id, localStorage.token);
			items = blocks.map((b) => ({ label: b.label, pendingDiffs: b.pending_diffs ?? 0 }));
			memoryBlocks.set(items);
		} catch (e) {
			console.error('Failed to load memory blocks:', e);
			items = [];
		}
		loading = false;
	}

	// TODO: Implement search filtering when backend supports it
	// For now, search box is visible but non-functional
	$: filteredItems = items;
</script>

<svelte:head>
	<title>
		{$i18n.t('Profile')} • {$WEBUI_NAME}
	</title>
</svelte:head>

{#if loaded}
	<!-- Content will be added in Phase 3 -->
	<div>Profile component skeleton</div>
{:else}
	<div class="w-full h-full flex justify-center items-center">
		<Spinner className="size-5" />
	</div>
{/if}
```

#### 3. Create BlockMenu Component (Placeholder)
**File**: `OpenWebUI/open-webui/src/lib/components/workspace/Profile/BlockMenu.svelte`

```svelte
<script lang="ts">
	import { getContext, createEventDispatcher } from 'svelte';
	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	import Dropdown from '$lib/components/common/Dropdown.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import EllipsisHorizontal from '$lib/components/icons/EllipsisHorizontal.svelte';

	let show = false;
</script>

<Dropdown bind:show>
	<Tooltip content={$i18n.t('More')}>
		<button
			class="self-center hover:bg-black/5 dark:hover:bg-white/5 dark:text-gray-300 rounded-md"
			on:click={() => {
				show = !show;
			}}
		>
			<EllipsisHorizontal className="size-5" />
		</button>
	</Tooltip>

	<div slot="content">
		<div
			class="min-w-[140px] text-sm font-medium z-50 bg-white dark:bg-gray-850 rounded-lg shadow-lg border border-gray-100 dark:border-gray-800"
		>
			<!-- TODO: Add menu items (Edit, Delete) when functionality is needed -->
			<div class="px-3 py-2 text-xs text-gray-500">
				{$i18n.t('No actions available')}
			</div>
		</div>
	</div>
</Dropdown>
```

### Success Criteria:

#### Automated Verification:
- [x] Route file exists: `ls OpenWebUI/open-webui/src/routes/(app)/workspace/profile/+page.svelte`
- [x] Component file exists: `ls OpenWebUI/open-webui/src/lib/components/workspace/Profile.svelte`
- [x] BlockMenu file exists: `ls OpenWebUI/open-webui/src/lib/components/workspace/Profile/BlockMenu.svelte`
- [x] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check` (i18n type warnings match existing patterns in Knowledge.svelte)

#### Manual Verification:
- [ ] Navigate to `/workspace/profile` shows "Profile component skeleton" text
- [ ] No console errors on page load

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 2: Add Profile Tab to Workspace Navigation

### Overview
Add the Profile tab link to the workspace layout navigation.

### Changes Required:

#### 1. Update Workspace Layout
**File**: `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte`

Add Profile tab after Tools (around line 123). Since Profile is available to all authenticated users, no permission check needed:

```svelte
<!-- Add after the Tools tab block (around line 123) -->
<a
	class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/profile')
		? ''
		: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
	href="/workspace/profile"
>
	{$i18n.t('Profile')}
</a>
```

The full tab section should look like:

```svelte
<div class="">
	<div
		class="flex gap-1 scrollbar-none overflow-x-auto w-fit text-center text-sm font-medium rounded-full bg-transparent py-1 touch-auto pointer-events-auto"
	>
		{#if $user?.role === 'admin' || $user?.permissions?.workspace?.models}
			<a
				class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/models')
					? ''
					: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
				href="/workspace/models">{$i18n.t('Models')}</a
			>
		{/if}

		{#if $user?.role === 'admin' || $user?.permissions?.workspace?.knowledge}
			<a
				class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/knowledge')
					? ''
					: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
				href="/workspace/knowledge"
			>
				{$i18n.t('Knowledge')}
			</a>
		{/if}

		{#if $user?.role === 'admin' || $user?.permissions?.workspace?.prompts}
			<a
				class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/prompts')
					? ''
					: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
				href="/workspace/prompts">{$i18n.t('Prompts')}</a
			>
		{/if}

		{#if $user?.role === 'admin' || $user?.permissions?.workspace?.tools}
			<a
				class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/tools')
					? ''
					: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
				href="/workspace/tools"
			>
				{$i18n.t('Tools')}
			</a>
		{/if}

		<!-- Profile tab - available to all authenticated users -->
		<a
			class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/profile')
				? ''
				: 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
			href="/workspace/profile"
		>
			{$i18n.t('Profile')}
		</a>
	</div>
</div>
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check` (i18n type warnings match existing tabs in layout)
- [x] Layout file contains Profile link: `grep -l "workspace/profile" OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte`

#### Manual Verification:
- [ ] Navigate to `/workspace/models` and see "Profile" tab in navigation
- [ ] Click "Profile" tab navigates to `/workspace/profile`
- [ ] Profile tab shows active styling when on `/workspace/profile`

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 3: Implement Profile Component UI

### Overview
Build out the full Profile.svelte component matching Knowledge.svelte patterns.

### Changes Required:

#### 1. Complete Profile.svelte Implementation
**File**: `OpenWebUI/open-webui/src/lib/components/workspace/Profile.svelte`

Replace the skeleton with the full implementation:

```svelte
<script lang="ts">
	import { onMount, getContext, tick } from 'svelte';
	const i18n = getContext('i18n');

	import { goto } from '$app/navigation';
	import { WEBUI_NAME, user } from '$lib/stores';
	import { memoryBlocks } from '$lib/stores/memory';
	import { getBlocks } from '$lib/apis/memory';

	import Badge from '../common/Badge.svelte';
	import Search from '../icons/Search.svelte';
	import Plus from '../icons/Plus.svelte';
	import Spinner from '../common/Spinner.svelte';
	import Tooltip from '../common/Tooltip.svelte';
	import XMark from '../icons/XMark.svelte';
	import ViewSelector from './common/ViewSelector.svelte';
	import BlockMenu from './Profile/BlockMenu.svelte';

	let loaded = false;
	let loading = false;

	let query = '';
	let viewOption = '';

	let items: Array<{ label: string; pendingDiffs: number }> | null = null;

	onMount(async () => {
		viewOption = localStorage?.workspaceViewOption || '';
		loaded = true;
		await loadBlocks();
	});

	async function loadBlocks() {
		if (!$user) return;

		loading = true;
		try {
			const blocks = await getBlocks($user.id, localStorage.token);
			items = blocks.map((b) => ({ label: b.label, pendingDiffs: b.pending_diffs ?? 0 }));
			memoryBlocks.set(items);
		} catch (e) {
			console.error('Failed to load memory blocks:', e);
			items = [];
		}
		loading = false;
	}

	// Convert snake_case to Title Case for display
	function formatLabel(label: string): string {
		return label
			.replace(/_/g, ' ')
			.replace(/\b\w/g, (c) => c.toUpperCase());
	}

	// TODO(search): Implement search filtering when backend supports it
	// For now, search box is visible but non-functional
	$: filteredItems = items;

	// Reactive reload when viewOption changes (for future use)
	$: if (loaded && viewOption !== undefined) {
		// TODO(filter): Implement view filtering when blocks support sharing
	}
</script>

<svelte:head>
	<title>
		{$i18n.t('Profile')} • {$WEBUI_NAME}
	</title>
</svelte:head>

{#if loaded}
	<!-- Header section -->
	<div class="flex flex-col gap-1 px-1 mt-1.5 mb-3">
		<div class="flex justify-between items-center">
			<div class="flex items-center md:self-center text-xl font-medium px-0.5 gap-2 shrink-0">
				<div>
					{$i18n.t('Profile')}
				</div>

				<div class="text-lg font-medium text-gray-500 dark:text-gray-500">
					{items?.length ?? ''}
				</div>
			</div>

			<div class="flex w-full justify-end gap-1.5">
				<!-- TODO(create): Wire up to create memory block flow -->
				<button
					class="px-2 py-1.5 rounded-xl bg-black text-white dark:bg-white dark:text-black transition font-medium text-sm flex items-center opacity-50 cursor-not-allowed"
					disabled
					title="Coming soon"
				>
					<Plus className="size-3" strokeWidth="2.5" />
					<div class="hidden md:block md:ml-1 text-xs">{$i18n.t('New Memory Block')}</div>
				</button>
			</div>
		</div>
	</div>

	<!-- Search and filter container -->
	<div
		class="py-2 bg-white dark:bg-gray-900 rounded-3xl border border-gray-100/30 dark:border-gray-850/30"
	>
		<!-- Search box -->
		<div class="flex w-full space-x-2 py-0.5 px-3.5 pb-2">
			<div class="flex flex-1">
				<div class="self-center ml-1 mr-3">
					<Search className="size-3.5" />
				</div>
				<input
					class="w-full text-sm py-1 rounded-r-xl outline-hidden bg-transparent"
					bind:value={query}
					placeholder={$i18n.t('Search Profile')}
					disabled
					title="Search coming soon"
				/>
				{#if query}
					<div class="self-center pl-1.5 translate-y-[0.5px] rounded-l-xl bg-transparent">
						<button
							class="p-0.5 rounded-full hover:bg-gray-100 dark:hover:bg-gray-900 transition"
							on:click={() => {
								query = '';
							}}
						>
							<XMark className="size-3" strokeWidth="2" />
						</button>
					</div>
				{/if}
			</div>
		</div>

		<!-- View selector -->
		<div
			class="px-3 flex w-full bg-transparent overflow-x-auto scrollbar-none -mx-1"
			on:wheel={(e) => {
				if (e.deltaY !== 0) {
					e.preventDefault();
					e.currentTarget.scrollLeft += e.deltaY;
				}
			}}
		>
			<div class="flex gap-0.5 w-fit text-center text-sm rounded-full bg-transparent px-1.5 whitespace-nowrap">
				<ViewSelector
					bind:value={viewOption}
					onChange={async (value) => {
						localStorage.workspaceViewOption = value;
						await tick();
					}}
				/>
			</div>
		</div>

		<!-- Items grid -->
		{#if items !== null}
			{#if (filteredItems ?? []).length !== 0}
				<div class="my-2 px-3 grid grid-cols-1 lg:grid-cols-2 gap-2">
					{#each filteredItems as item (item.label)}
						<button
							class="flex space-x-4 cursor-pointer text-left w-full px-3 py-2.5 dark:hover:bg-gray-850/50 hover:bg-gray-50 transition rounded-2xl"
							on:click={() => {
								goto(`/you/blocks/${item.label}`);
							}}
						>
							<div class="w-full">
								<div class="self-center flex-1 justify-between">
									<!-- Badge row -->
									<div class="flex items-center justify-between -my-1 h-8">
										<div class="flex gap-2 items-center">
											<Badge type="muted" content={$i18n.t('Memory Block')} />
											{#if item.pendingDiffs > 0}
												<Badge type="warning" content="{item.pendingDiffs} {$i18n.t('pending')}" />
											{/if}
										</div>

										<div class="flex items-center gap-2">
											<div class="flex self-center">
												<BlockMenu />
											</div>
										</div>
									</div>

									<!-- Content row -->
									<div class="flex items-center gap-1 justify-between px-1.5">
										<Tooltip content={item.label}>
											<div class="flex items-center gap-2">
												<div class="text-sm font-medium line-clamp-1 capitalize">
													{formatLabel(item.label)}
												</div>
											</div>
										</Tooltip>
									</div>
								</div>
							</div>
						</button>
					{/each}
				</div>
			{:else}
				<!-- Empty state -->
				<div class="w-full h-full flex flex-col justify-center items-center my-16 mb-24">
					<div class="max-w-md text-center">
						<div class="text-3xl mb-3">😕</div>
						<div class="text-lg font-medium mb-1">{$i18n.t('No memory blocks found')}</div>
						<div class="text-gray-500 text-center text-xs">
							{$i18n.t('Start a conversation to begin building your profile.')}
						</div>
					</div>
				</div>
			{/if}
		{:else if loading}
			<!-- Loading state -->
			<div class="w-full h-full flex justify-center items-center py-10">
				<Spinner className="size-4" />
			</div>
		{/if}
	</div>
{:else}
	<div class="w-full h-full flex justify-center items-center">
		<Spinner className="size-5" />
	</div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check` (i18n type warnings match existing codebase patterns)
- [x] Component imports resolve correctly

#### Manual Verification:
- [ ] Navigate to `/workspace/profile` shows memory blocks in grid layout
- [ ] Each block card shows "Memory Block" badge
- [ ] Blocks with pending diffs show warning badge with count
- [ ] Clicking a block navigates to `/you/blocks/{label}`
- [ ] "New Memory Block" button is visible but disabled
- [ ] Search box is visible but disabled
- [ ] ViewSelector is visible and toggleable
- [ ] Empty state shows when no blocks exist
- [ ] Loading spinner shows while fetching

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 4: Redirect /you to /workspace/profile

### Overview
Replace the `/you` page content with a redirect to `/workspace/profile`.

### Changes Required:

#### 1. Update /you Page to Redirect
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`

Replace the entire file with a simple redirect:

```svelte
<script lang="ts">
	import { goto } from '$app/navigation';
	import { onMount } from 'svelte';

	onMount(() => {
		goto('/workspace/profile', { replaceState: true });
	});
</script>

<!-- Redirect to /workspace/profile -->
<div class="w-full h-full flex justify-center items-center">
	<div class="animate-spin size-4 border-2 border-gray-300 border-t-gray-600 dark:border-gray-600 dark:border-t-gray-300 rounded-full" />
</div>
```

#### 2. Keep /you/blocks/[label] Route Unchanged
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/blocks/[label]/+page.svelte`

No changes needed - this route stays as-is.

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check`
- [x] Redirect code present: `grep -l "workspace/profile" OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`

#### Manual Verification:
- [ ] Navigate to `/you` → automatically redirects to `/workspace/profile`
- [ ] URL changes to `/workspace/profile` (not just content)
- [ ] Navigate to `/you/blocks/student` → shows block detail page (no redirect)
- [ ] Browser back button works correctly after redirect

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Testing Strategy

### Unit Tests:
- No new unit tests required (frontend components)

### Integration Tests:
- Verify API endpoint `/users/{user_id}/blocks` returns expected data shape

### Manual Testing Steps:
1. Log in as any user
2. Navigate to `/workspace` → Profile tab visible in navigation
3. Click Profile tab → shows memory blocks grid
4. Click a block → navigates to `/you/blocks/{label}` detail page
5. Navigate directly to `/you` → redirects to `/workspace/profile`
6. Navigate to `/you/blocks/student` → shows detail page (no redirect)
7. Verify pending diff badges display correctly on blocks with diffs
8. Test on mobile viewport → responsive grid (1 column)
9. Test on desktop viewport → responsive grid (2 columns)

## Performance Considerations

- Memory blocks are fetched client-side without pagination (acceptable for MVP since users typically have <10 blocks)
- ViewSelector persists to localStorage to avoid re-fetching on option change
- No infinite scroll needed (unlike Knowledge which can have many items)

## Migration Notes

- No database migrations required
- No backend changes required
- Existing `/you/blocks/[label]` route preserved for backwards compatibility

## Future Work (TODOs in code)

1. `TODO(search)`: Implement search filtering when backend supports it
2. `TODO(filter)`: Implement view filtering when blocks support sharing
3. `TODO(create)`: Wire up "New Memory Block" button to create flow
4. Add delete functionality to BlockMenu
5. Consider moving `/you/blocks/[label]` to `/workspace/profile/{label}` for consistency

## References

- Research document: `thoughts/shared/research/2026-01-14-workspace-knowledge-ui-patterns.md`
- Knowledge.svelte (reference pattern): `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte`
- Memory blocks API: `src/youlab_server/server/blocks.py`
- Existing BlockCard: `OpenWebUI/open-webui/src/lib/components/you/BlockCard.svelte`
