# ARI-82 Phase 2: Memory Block Editor Route Implementation Plan

## Overview

Create a new route `/you/blocks/[label]` for editing memory blocks, adapted from OpenWebUI's NoteEditor pattern. This provides a full-page editor experience with TipTap rich text editing, AI features (title generation, content enhancement), file/image upload support, and autosave functionality.

## Current State Analysis

### What Exists

| Component | Implementation | Location |
|-----------|----------------|----------|
| BlockDetailModal | Modal-based editing with Textarea | `OpenWebUI/.../components/you/BlockDetailModal.svelte` |
| BlockCard | Click handler opens modal | `OpenWebUI/.../components/you/BlockCard.svelte` |
| /you Page | Profile tab lists blocks, opens modal | `OpenWebUI/.../routes/(app)/you/+page.svelte` |
| Memory API | Full CRUD + history + diffs | `OpenWebUI/.../lib/apis/memory/index.ts` |
| RichTextInput | TipTap-based editor | `OpenWebUI/.../components/common/RichTextInput.svelte` |
| NoteEditor | Reference implementation | `OpenWebUI/.../components/notes/NoteEditor.svelte` |

### Key Discoveries

1. **NoteEditor is 1400+ lines** with:
   - TipTap RichTextInput integration
   - AI title generation via `generateOpenAIChatCompletion`
   - Content enhancement via `chatCompletion` streaming
   - 200ms debounce autosave
   - File/image upload with inline embedding
   - Version navigation (undo/redo via in-memory versions)
   - PaneGroup for sidebar panel (chat, settings)
   - Voice recording support

2. **/notes/[id] route pattern** is the template:
   ```svelte
   <NoteEditor id={$page.params.id} />
   ```

3. **Memory API already supports autosave** - just use the existing `updateBlock` with a short commit message.

4. **Diff view should remain accessible** - keep BlockDetailModal for diff approval, or integrate diff view into editor.

## Desired End State

After implementation:

1. Clicking a BlockCard navigates to `/you/blocks/{label}` instead of opening modal
2. Full-page editor with TipTap rich text editing
3. Autosave with 200ms debounce (commits to git)
4. AI title generation and content enhancement
5. File/image upload with inline display
6. Undo/redo via git history navigation
7. Diff approval accessible from editor header
8. Back navigation returns to /you

### Verification

```bash
# Navigate to block editor
curl -s $OPENWEBUI_URL/you/blocks/student
# Expected: 200 OK, full editor page

# Autosave working
# Edit content, wait 200ms, check git log
git -C .data/test-user-id log --oneline -1
# Expected: "Autosave student" commit

# File upload
# Drag image into editor
# Expected: Image appears inline, stored in user's files directory
```

## What We're NOT Doing

1. **Not removing BlockDetailModal** - keep for diff approval workflow (or can be accessed from editor)
2. **Not implementing WebSocket collaboration** - single-user editing is sufficient
3. **Not implementing voice recording initially** - can be added later
4. **Not changing backend storage format** - reuse existing TOML→MD conversion

## Implementation Approach

Fork and simplify NoteEditor for memory blocks:
1. Create route structure
2. Create simplified BlockEditor component (no collaboration, simpler data model)
3. Add autosave API function
4. Update navigation from BlockCard

---

## Phase 1: Create Route Structure

### Overview

Create the SvelteKit route files for `/you/blocks/[label]`.

### Changes Required:

#### 1. Create Route Directory

**Directory**: `OpenWebUI/open-webui/src/routes/(app)/you/blocks/[label]/`

Create the following files:

#### 2. Create Page Component

**File**: `OpenWebUI/open-webui/src/routes/(app)/you/blocks/[label]/+page.svelte`

```svelte
<script lang="ts">
	import { onMount } from 'svelte';
	import { page } from '$app/stores';
	import { showSidebar } from '$lib/stores';

	import BlockEditor from '$lib/components/you/BlockEditor.svelte';

	let loaded = false;

	onMount(async () => {
		loaded = true;
	});
</script>

{#if loaded}
	<div
		id="block-editor-container"
		class="w-full h-full {$showSidebar ? 'md:max-w-[calc(100%-var(--sidebar-width))]' : ''}"
	>
		<BlockEditor label={$page.params.label} />
	</div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] Route compiles without errors
- [ ] Navigating to `/you/blocks/student` loads the page

#### Manual Verification:
- [ ] Page renders without errors in browser
- [ ] URL parameter correctly extracted

---

## Phase 2: Create BlockEditor Component

### Overview

Create a simplified version of NoteEditor for memory blocks. Focus on essential features: rich text editing, autosave, AI assistance.

### Changes Required:

#### 1. Create BlockEditor Component

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`

```svelte
<script lang="ts">
	import { getContext, onDestroy, onMount, tick } from 'svelte';
	import { toast } from 'svelte-sonner';
	import { goto } from '$app/navigation';

	import dayjs from '$lib/dayjs';
	import { marked } from 'marked';

	import { PaneGroup, Pane, PaneResizer } from 'paneforge';
	import { compressImage, copyToClipboard, convertHeicToJpeg } from '$lib/utils';
	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';
	import { getFileById, uploadFile } from '$lib/apis/files';
	import { chatCompletion, generateOpenAIChatCompletion } from '$lib/apis/openai';

	import {
		config,
		mobile,
		models,
		settings,
		showSidebar,
		user,
		WEBUI_NAME
	} from '$lib/stores';
	import { selectedBlock, blockHistory } from '$lib/stores/memory';
	import {
		getBlock,
		updateBlock,
		getBlockHistory,
		getBlockDiffs,
		type PendingDiff
	} from '$lib/apis/memory';

	import RichTextInput from '$lib/components/common/RichTextInput.svelte';
	import Spinner from '$lib/components/common/Spinner.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import ArrowLeft from '$lib/components/icons/ArrowLeft.svelte';
	import ArrowUturnLeft from '$lib/components/icons/ArrowUturnLeft.svelte';
	import ArrowUturnRight from '$lib/components/icons/ArrowUturnRight.svelte';
	import Sparkles from '$lib/components/icons/Sparkles.svelte';
	import SparklesSolid from '$lib/components/icons/SparklesSolid.svelte';
	import EllipsisHorizontal from '$lib/components/icons/EllipsisHorizontal.svelte';
	import AiMenu from '$lib/components/notes/AIMenu.svelte';
	import BlockEditorPanel from './BlockEditorPanel.svelte';
	import DiffBadge from './DiffBadge.svelte';

	const i18n = getContext('i18n');

	export let label: string;

	let editor = null;
	let inputElement = null;
	let loading = true;
	let saving = false;

	// Block data
	let markdownContent = '';
	let displayName = '';
	let files = [];
	let pendingDiffs: PendingDiff[] = [];

	// Editor state
	let wordCount = 0;
	let charCount = 0;
	let editing = false;
	let streaming = false;
	let stopResponseFlag = false;

	// UI state
	let showPanel = false;
	let selectedPanel = 'settings';
	let selectedModelId = '';
	let titleGenerating = false;

	// Autosave
	let debounceTimeout: ReturnType<typeof setTimeout> | null = null;
	let lastSavedContent = '';
	let autosaveStatus: 'idle' | 'saving' | 'saved' | 'error' = 'idle';

	$: displayName = label
		.replace(/_/g, ' ')
		.replace(/\b\w/g, (c) => c.toUpperCase());

	const init = async () => {
		loading = true;
		try {
			const block = await getBlock($user.id, label, localStorage.token);
			selectedBlock.set(block);
			markdownContent = block.contentMarkdown;
			lastSavedContent = markdownContent;

			const history = await getBlockHistory($user.id, label, localStorage.token);
			blockHistory.set(history);

			const diffs = await getBlockDiffs($user.id, label, localStorage.token);
			pendingDiffs = diffs;
		} catch (e) {
			toast.error($i18n.t('Failed to load block'));
			console.error(e);
			goto('/you');
			return;
		}
		loading = false;
	};

	// Autosave handler
	const scheduleAutosave = () => {
		if (debounceTimeout) {
			clearTimeout(debounceTimeout);
		}

		debounceTimeout = setTimeout(async () => {
			if (markdownContent !== lastSavedContent && markdownContent.trim()) {
				autosaveStatus = 'saving';
				try {
					await updateBlock(
						$user.id,
						label,
						markdownContent,
						localStorage.token,
						'markdown',
						`Autosave ${label}`
					);
					lastSavedContent = markdownContent;
					autosaveStatus = 'saved';
					setTimeout(() => {
						autosaveStatus = 'idle';
					}, 2000);
				} catch (e) {
					autosaveStatus = 'error';
					console.error('Autosave failed:', e);
				}
			}
		}, 200);
	};

	// AI title generation
	const generateTitleHandler = async () => {
		const content = markdownContent;
		const DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE = `### Task:
Generate a concise, 3-5 word title summarizing the memory block content.
### Guidelines:
- Title should represent the main theme or subject
- Keep it clear and simple
- Output only the title, no quotes or formatting
### Content:
${content}`;

		titleGenerating = true;
		try {
			const res = await generateOpenAIChatCompletion(
				localStorage.token,
				{
					model: selectedModelId,
					stream: false,
					messages: [{ role: 'user', content: DEFAULT_TITLE_GENERATION_PROMPT_TEMPLATE }]
				},
				`${WEBUI_BASE_URL}/api`
			);

			if (res?.choices?.[0]?.message?.content) {
				toast.success($i18n.t('Title suggestion: ') + res.choices[0].message.content.trim());
			}
		} catch (e) {
			toast.error($i18n.t('Failed to generate title'));
		}
		titleGenerating = false;
	};

	// AI content enhancement
	const enhanceContentHandler = async () => {
		if (!selectedModelId) {
			toast.error($i18n.t('Please select a model'));
			return;
		}

		editing = true;
		streaming = true;
		stopResponseFlag = false;

		const systemPrompt = `Enhance the memory block content while preserving its structure and key information.
Improve clarity, add relevant details, and fix any grammar issues.
Return only the enhanced markdown content.`;

		try {
			const [res, controller] = await chatCompletion(
				localStorage.token,
				{
					model: selectedModelId,
					stream: true,
					messages: [
						{ role: 'system', content: systemPrompt },
						{ role: 'user', content: markdownContent }
					]
				},
				`${WEBUI_BASE_URL}/api`
			);

			if (res && res.ok) {
				let enhancedContent = '';
				const reader = res.body
					.pipeThrough(new TextDecoderStream())
					.getReader();

				while (true) {
					const { value, done } = await reader.read();
					if (done || stopResponseFlag) {
						if (stopResponseFlag) controller.abort();
						break;
					}

					const lines = value.split('\n');
					for (const line of lines) {
						if (line.startsWith('data: ') && line !== 'data: [DONE]') {
							try {
								const data = JSON.parse(line.slice(6));
								if (data.choices?.[0]?.delta?.content) {
									enhancedContent += data.choices[0].delta.content;
									markdownContent = enhancedContent;
								}
							} catch {}
						}
					}
				}
			}
		} catch (e) {
			toast.error($i18n.t('Enhancement failed'));
		}

		editing = false;
		streaming = false;
		if (editor) {
			editor.commands.setContent(marked.parse(markdownContent));
		}
	};

	const stopResponseHandler = () => {
		stopResponseFlag = true;
	};

	const handleBack = () => {
		goto('/you');
	};

	onMount(async () => {
		if ($settings?.models) {
			selectedModelId = $settings.models[0];
		} else if ($config?.default_models) {
			selectedModelId = $config.default_models.split(',')[0];
		}

		if (label) {
			await init();
		}
	});

	onDestroy(() => {
		if (debounceTimeout) {
			clearTimeout(debounceTimeout);
		}
	});
</script>

<svelte:head>
	<title>{displayName} - {$WEBUI_NAME}</title>
</svelte:head>

<PaneGroup direction="horizontal" class="w-full h-full">
	<Pane defaultSize={70} minSize={30} class="h-full flex flex-col w-full relative">
		<div class="relative flex-1 w-full h-full flex justify-center pt-3" id="block-editor">
			{#if loading}
				<div class="absolute inset-0 flex">
					<div class="m-auto">
						<Spinner className="size-5" />
					</div>
				</div>
			{:else}
				<div class="w-full flex flex-col">
					<!-- Header -->
					<div class="shrink-0 w-full flex justify-between items-center px-3.5 mb-2">
						<div class="flex items-center gap-2">
							<Tooltip content={$i18n.t('Back')}>
								<button
									class="p-1.5 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition"
									on:click={handleBack}
								>
									<ArrowLeft className="size-5" />
								</button>
							</Tooltip>

							<h1 class="text-xl font-medium">{displayName}</h1>

							{#if pendingDiffs.length > 0}
								<DiffBadge count={pendingDiffs.length} {label} />
							{/if}
						</div>

						<div class="flex items-center gap-1">
							<!-- Autosave status -->
							{#if autosaveStatus === 'saving'}
								<span class="text-xs text-gray-400">{$i18n.t('Saving...')}</span>
							{:else if autosaveStatus === 'saved'}
								<span class="text-xs text-green-500">{$i18n.t('Saved')}</span>
							{:else if autosaveStatus === 'error'}
								<span class="text-xs text-red-500">{$i18n.t('Save failed')}</span>
							{/if}

							<!-- Undo/Redo -->
							{#if editor}
								<button
									class="p-1.5 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition disabled:opacity-30"
									on:click={() => editor.chain().focus().undo().run()}
									disabled={!editor.can().undo()}
								>
									<ArrowUturnLeft className="size-4" />
								</button>
								<button
									class="p-1.5 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition disabled:opacity-30"
									on:click={() => editor.chain().focus().redo().run()}
									disabled={!editor.can().redo()}
								>
									<ArrowUturnRight className="size-4" />
								</button>
							{/if}

							<!-- Settings panel toggle -->
							<Tooltip content={$i18n.t('Settings')}>
								<button
									class="p-1.5 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition"
									on:click={() => {
										showPanel = !showPanel;
										selectedPanel = 'settings';
									}}
								>
									<EllipsisHorizontal className="size-5" />
								</button>
							</Tooltip>
						</div>
					</div>

					<!-- Metadata bar -->
					<div class="px-4 flex items-center gap-2 text-xs text-gray-500 mb-2">
						<span>{$i18n.t('Memory Block')}</span>
						{#if editor}
							<span>|</span>
							<span>{$i18n.t('{{COUNT}} words', { COUNT: wordCount })}</span>
							<span>{$i18n.t('{{COUNT}} characters', { COUNT: charCount })}</span>
						{/if}
					</div>

					<!-- Editor -->
					<div class="flex-1 w-full h-full overflow-auto px-3.5 relative" id="block-content-container">
						{#if editing}
							<div
								class="w-full h-full fixed top-0 left-0 {streaming
									? ''
									: 'backdrop-blur-xs bg-white/10 dark:bg-gray-900/10'} flex items-center justify-center z-10"
							></div>
						{/if}

						<RichTextInput
							bind:this={inputElement}
							bind:editor
							id={`block-${label}`}
							className="input-prose-sm px-0.5 h-[calc(100%-2rem)]"
							html={marked.parse(markdownContent)}
							placeholder={$i18n.t('Start writing...')}
							editable={!editing}
							onChange={(content) => {
								markdownContent = content.md;

								if (editor) {
									wordCount = editor.storage.characterCount.words();
									charCount = editor.storage.characterCount.characters();
								}

								scheduleAutosave();
							}}
						/>
					</div>
				</div>
			{/if}
		</div>

		<!-- AI FAB -->
		<div class="absolute z-50 bottom-0 right-0 p-3.5 flex select-none">
			{#if editing}
				<button
					class="p-2.5 flex rounded-full bg-white dark:bg-gray-850 hover:bg-gray-50 dark:hover:bg-gray-800 transition shadow-xl"
					on:click={stopResponseHandler}
				>
					<Spinner className="size-5" />
				</button>
			{:else}
				<AiMenu
					onEdit={enhanceContentHandler}
					onChat={() => {
						showPanel = true;
						selectedPanel = 'chat';
					}}
				>
					<div
						class="cursor-pointer p-2.5 flex rounded-full border border-gray-50 bg-white dark:border-none dark:bg-gray-850 hover:bg-gray-50 dark:hover:bg-gray-800 transition shadow-xl"
					>
						<SparklesSolid />
					</div>
				</AiMenu>
			{/if}
		</div>
	</Pane>

	<!-- Side Panel -->
	{#if showPanel}
		<PaneResizer class="w-1 bg-gray-100 dark:bg-gray-800" />
		<Pane defaultSize={30} minSize={20}>
			<BlockEditorPanel
				bind:show={showPanel}
				bind:selectedModelId
				{selectedPanel}
				{files}
				{label}
				{pendingDiffs}
			/>
		</Pane>
	{/if}
</PaneGroup>
```

#### 2. Create Support Components

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditorPanel.svelte`

```svelte
<script lang="ts">
	import { getContext } from 'svelte';
	import { models, settings } from '$lib/stores';
	import type { PendingDiff } from '$lib/apis/memory';

	import XMark from '$lib/components/icons/XMark.svelte';

	const i18n = getContext('i18n');

	export let show = false;
	export let selectedModelId = '';
	export let selectedPanel: 'settings' | 'chat' = 'settings';
	export let files: any[] = [];
	export let label: string;
	export let pendingDiffs: PendingDiff[] = [];

	$: visibleModels = $models.filter((m) => !(m?.info?.meta?.hidden ?? false));
</script>

<div class="h-full flex flex-col bg-white dark:bg-gray-900 border-l border-gray-200 dark:border-gray-800">
	<div class="flex items-center justify-between p-3 border-b border-gray-200 dark:border-gray-800">
		<span class="font-medium">
			{selectedPanel === 'settings' ? $i18n.t('Settings') : $i18n.t('Chat')}
		</span>
		<button
			class="p-1 rounded hover:bg-gray-100 dark:hover:bg-gray-800 transition"
			on:click={() => (show = false)}
		>
			<XMark className="size-5" />
		</button>
	</div>

	<div class="flex-1 overflow-y-auto p-4">
		{#if selectedPanel === 'settings'}
			<!-- Model selector -->
			<div class="mb-4">
				<label class="block text-sm font-medium mb-1">{$i18n.t('AI Model')}</label>
				<select
					bind:value={selectedModelId}
					class="w-full p-2 rounded-lg border border-gray-200 dark:border-gray-700 bg-transparent"
				>
					{#each visibleModels as model}
						<option value={model.id}>{model.name}</option>
					{/each}
				</select>
			</div>

			<!-- Pending diffs -->
			{#if pendingDiffs.length > 0}
				<div class="mb-4">
					<label class="block text-sm font-medium mb-2">{$i18n.t('Pending Changes')}</label>
					<div class="space-y-2">
						{#each pendingDiffs as diff}
							<div class="p-2 rounded-lg bg-amber-50 dark:bg-amber-900/20 text-sm">
								<div class="font-medium text-amber-700 dark:text-amber-300">
									{diff.operation}
								</div>
								<div class="text-amber-600 dark:text-amber-400 text-xs mt-1">
									{diff.reasoning}
								</div>
							</div>
						{/each}
					</div>
					<a
						href="/you"
						class="block text-center text-sm text-blue-600 dark:text-blue-400 mt-2 hover:underline"
					>
						{$i18n.t('Review in full view')}
					</a>
				</div>
			{/if}

			<!-- Attached files -->
			{#if files.length > 0}
				<div>
					<label class="block text-sm font-medium mb-2">{$i18n.t('Attached Files')}</label>
					<div class="space-y-1">
						{#each files as file}
							<div class="text-sm text-gray-600 dark:text-gray-400 truncate">
								{file.name}
							</div>
						{/each}
					</div>
				</div>
			{/if}
		{:else}
			<!-- Chat panel placeholder -->
			<div class="text-center text-gray-500 py-8">
				{$i18n.t('AI chat coming soon')}
			</div>
		{/if}
	</div>
</div>
```

**File**: `OpenWebUI/open-webui/src/lib/components/you/DiffBadge.svelte`

```svelte
<script lang="ts">
	import { getContext } from 'svelte';
	import Badge from '$lib/components/common/Badge.svelte';

	const i18n = getContext('i18n');

	export let count: number;
	export let label: string;
</script>

<a href="/you" class="inline-block">
	<Badge type="warning" content="{count} {$i18n.t('pending')}" />
</a>
```

**File**: `OpenWebUI/open-webui/src/lib/components/icons/ArrowLeft.svelte`

```svelte
<script lang="ts">
	export let className = 'size-4';
	export let strokeWidth = '1.5';
</script>

<svg
	xmlns="http://www.w3.org/2000/svg"
	fill="none"
	viewBox="0 0 24 24"
	stroke-width={strokeWidth}
	stroke="currentColor"
	class={className}
>
	<path stroke-linecap="round" stroke-linejoin="round" d="M10.5 19.5 3 12m0 0 7.5-7.5M3 12h18" />
</svg>
```

### Success Criteria:

#### Automated Verification:
- [ ] BlockEditor component compiles without TypeScript errors
- [ ] All imports resolve correctly

#### Manual Verification:
- [ ] Navigate to `/you/blocks/student`
- [ ] Editor loads with block content
- [ ] TipTap editor is functional
- [ ] Autosave triggers after 200ms of inactivity
- [ ] AI enhancement button works

---

## Phase 3: Update Navigation Flow

### Overview

Update BlockCard and /you page to navigate to the new route instead of opening a modal.

### Changes Required:

#### 1. Update BlockCard to Use Navigation

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockCard.svelte`

Replace the click handler to use navigation:

```svelte
<script lang="ts">
	import { getContext } from 'svelte';
	import { goto } from '$app/navigation';
	import Badge from '$lib/components/common/Badge.svelte';
	import Document from '$lib/components/icons/Document.svelte';

	const i18n = getContext('i18n');

	export let label: string;
	export let pendingDiffs: number = 0;

	// Convert snake_case to Title Case
	$: displayName = label
		.replace(/_/g, ' ')
		.replace(/\b\w/g, (c) => c.toUpperCase());

	function handleClick() {
		goto(`/you/blocks/${label}`);
	}
</script>

<button
	class="flex space-x-4 cursor-pointer text-left w-full px-3 py-2.5
		   hover:bg-gray-50 dark:hover:bg-gray-850/50 transition rounded-2xl"
	on:click={handleClick}
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

#### 2. Update /you Page

**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`

Remove the modal-related code and simplify:

```svelte
<script lang="ts">
	import { getContext, onMount } from 'svelte';

	const i18n = getContext('i18n');

	import { mobile, showArchivedChats, showSidebar, user } from '$lib/stores';
	import { memoryBlocks, pendingDiffsCount } from '$lib/stores/memory';
	import { getBlocks } from '$lib/apis/memory';

	import UserMenu from '$lib/components/layout/Sidebar/UserMenu.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import Sidebar from '$lib/components/icons/Sidebar.svelte';
	import { WEBUI_API_BASE_URL } from '$lib/constants';

	import BlockCard from '$lib/components/you/BlockCard.svelte';
	import AgentsTab from '$lib/components/you/AgentsTab.svelte';

	let loaded = false;
	let loading = true;
	let selectedTab: 'profile' | 'agents' = 'profile';

	onMount(async () => {
		loaded = true;
		await loadBlocks();
	});

	const MOCK_BLOCKS = [
		{ label: 'student', pendingDiffs: 1 },
		{ label: 'engagement_strategy', pendingDiffs: 1 },
		{ label: 'journey', pendingDiffs: 1 }
	];

	async function loadBlocks() {
		if (!$user) return;

		loading = true;
		try {
			const blocks = await getBlocks($user.id, localStorage.token);
			memoryBlocks.set(blocks);
		} catch (e) {
			console.error('Failed to load blocks, using mock data:', e);
			memoryBlocks.set(MOCK_BLOCKS);
		}
		loading = false;
	}
</script>

{#if loaded}
	<div
		class="flex flex-col w-full h-screen max-h-[100dvh] transition-width duration-200 ease-in-out {$showSidebar
			? 'md:max-w-[calc(100%-var(--sidebar-width))]'
			: ''} max-w-full"
	>
		<nav class="px-2 pt-1.5 backdrop-blur-xl w-full drag-region">
			<div class="flex items-center">
				{#if $mobile}
					<div class="{$showSidebar ? 'md:hidden' : ''} flex flex-none items-center">
						<Tooltip
							content={$showSidebar ? $i18n.t('Close Sidebar') : $i18n.t('Open Sidebar')}
							interactive={true}
						>
							<button
								id="sidebar-toggle-button"
								class="cursor-pointer flex rounded-lg hover:bg-gray-100 dark:hover:bg-gray-850 transition"
								on:click={() => {
									showSidebar.set(!$showSidebar);
								}}
							>
								<div class="self-center p-1.5">
									<Sidebar />
								</div>
							</button>
						</Tooltip>
					</div>
				{/if}

				<div class="ml-2 py-0.5 self-center flex items-center justify-between w-full">
					<div>
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
					</div>

					<div class="self-center flex items-center gap-1">
						{#if $user !== undefined && $user !== null}
							<UserMenu
								className="max-w-[240px]"
								role={$user?.role}
								help={true}
								on:show={(e) => {
									if (e.detail === 'archived-chat') {
										showArchivedChats.set(true);
									}
								}}
							>
								<button
									class="select-none flex rounded-xl p-1.5 w-full hover:bg-gray-50 dark:hover:bg-gray-850 transition"
									aria-label="User Menu"
								>
									<div class="self-center">
										<img
											src={`${WEBUI_API_BASE_URL}/users/${$user?.id}/profile/image`}
											class="size-6 object-cover rounded-full"
											alt="User profile"
											draggable="false"
										/>
									</div>
								</button>
							</UserMenu>
						{/if}
					</div>
				</div>
			</div>
		</nav>

		<div class="pb-1 px-3 md:px-[18px] flex-1 max-h-full overflow-y-auto">
			{#if selectedTab === 'profile'}
				<div class="flex flex-col gap-1 px-1 mt-1.5 mb-3">
					<div class="flex justify-between items-center">
						<div class="flex items-center md:self-center text-xl font-medium px-0.5 gap-2 shrink-0">
							<div>{$i18n.t('Profile')}</div>
							<div class="text-lg font-medium text-gray-500 dark:text-gray-500">
								{$memoryBlocks.length}
							</div>
						</div>
					</div>
				</div>

				{#if loading}
					<div class="w-full h-full flex justify-center items-center py-10">
						<div class="animate-spin size-4 border-2 border-gray-300 border-t-gray-600 dark:border-gray-600 dark:border-t-gray-300 rounded-full" />
					</div>
				{:else if $memoryBlocks.length === 0}
					<div class="w-full h-full flex flex-col justify-center items-center my-16 mb-24">
						<div class="max-w-md text-center">
							<div class="text-3xl mb-3">Blocks</div>
							<div class="text-lg font-medium mb-1">{$i18n.t('No memory blocks found')}</div>
							<div class="text-gray-500 text-center text-xs">
								{$i18n.t('Start a conversation to begin building your profile.')}
							</div>
						</div>
					</div>
				{:else}
					<div class="py-2 bg-white dark:bg-gray-900 rounded-3xl border border-gray-100/30 dark:border-gray-850/30">
						<div class="my-2 px-3 grid grid-cols-1 lg:grid-cols-2 gap-2">
							{#each $memoryBlocks as block (block.label)}
								<BlockCard
									label={block.label}
									pendingDiffs={block.pendingDiffs}
								/>
							{/each}
						</div>
					</div>
				{/if}
			{:else if selectedTab === 'agents'}
				<div class="flex flex-col gap-1 px-1 mt-1.5 mb-3">
					<div class="flex justify-between items-center">
						<div class="flex items-center md:self-center text-xl font-medium px-0.5 gap-2 shrink-0">
							<div>{$i18n.t('Agents')}</div>
						</div>
					</div>
				</div>

				<AgentsTab />
			{/if}
		</div>
	</div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] No TypeScript errors in modified files
- [ ] Route navigation works without errors

#### Manual Verification:
- [ ] Click on BlockCard navigates to `/you/blocks/{label}`
- [ ] Back button returns to `/you`
- [ ] Diff badge count shows correctly
- [ ] Tab state preserved when returning

---

## Phase 4: Add File Upload Support

### Overview

Add file and image upload support to BlockEditor, similar to NoteEditor.

### Changes Required:

#### 1. Add Upload Handler to BlockEditor

Add the following to BlockEditor.svelte after the existing state variables:

```typescript
// File upload handling
const uploadFileHandler = async (file: File) => {
	const tempItemId = uuidv4();
	const fileItem = {
		type: 'file',
		file: '',
		id: null,
		url: '',
		name: file.name,
		status: 'uploading',
		size: file.size,
		error: '',
		itemId: tempItemId
	};

	if (fileItem.size === 0) {
		toast.error($i18n.t('You cannot upload an empty file.'));
		return null;
	}

	files = [...files, fileItem];

	try {
		const uploadedFile = await uploadFile(localStorage.token, file, null);

		if (uploadedFile) {
			if (uploadedFile.error) {
				toast.warning(uploadedFile.error);
			}

			fileItem.status = 'uploaded';
			fileItem.file = await getFileById(localStorage.token, uploadedFile.id);
			fileItem.id = uploadedFile.id;
			fileItem.url = `${uploadedFile.id}`;
			files = files;
		} else {
			files = files.filter((item) => item?.itemId !== tempItemId);
		}
	} catch (e) {
		toast.error(`${e}`);
		files = files.filter((item) => item?.itemId !== tempItemId);
	}

	return fileItem;
};

const inputFileHandler = async (file: File) => {
	if (file.type.startsWith('image/')) {
		const reader = new FileReader();
		return new Promise((resolve) => {
			reader.onload = async (event) => {
				let imageUrl = event.target?.result as string;
				// Optional: compress image
				const fileId = uuidv4();
				const fileItem = {
					id: fileId,
					type: 'image',
					url: imageUrl
				};
				files = [...files, fileItem];
				resolve(fileItem);
			};
			reader.readAsDataURL(file);
		});
	} else {
		return await uploadFileHandler(file);
	}
};
```

#### 2. Enable File Handler in RichTextInput

Update the RichTextInput usage in BlockEditor:

```svelte
<RichTextInput
	bind:this={inputElement}
	bind:editor
	id={`block-${label}`}
	className="input-prose-sm px-0.5 h-[calc(100%-2rem)]"
	html={marked.parse(markdownContent)}
	placeholder={$i18n.t('Start writing...')}
	editable={!editing}
	fileHandler={true}
	image={true}
	{files}
	onFileDrop={(currentEditor, droppedFiles, pos) => {
		droppedFiles.forEach(async (file) => {
			const fileItem = await inputFileHandler(file);
			if (fileItem?.type === 'image') {
				currentEditor
					.chain()
					.insertContentAt(pos, {
						type: 'image',
						attrs: { src: `data://${fileItem.id}` }
					})
					.focus()
					.run();
			}
		});
	}}
	onChange={(content) => {
		markdownContent = content.md;
		if (editor) {
			wordCount = editor.storage.characterCount.words();
			charCount = editor.storage.characterCount.characters();
		}
		scheduleAutosave();
	}}
/>
```

### Success Criteria:

#### Automated Verification:
- [ ] File upload code compiles without errors

#### Manual Verification:
- [ ] Drag and drop image into editor
- [ ] Image appears inline
- [ ] Paste image from clipboard works
- [ ] Autosave includes file references

---

## Testing Strategy

### Unit Tests

These would be added to `OpenWebUI/open-webui/` test directory if one exists:

1. **BlockEditor load test** - Verify block loads correctly
2. **Autosave debounce test** - Verify 200ms debounce works
3. **Navigation test** - Verify BlockCard navigates to correct route

### Integration Tests

1. **Full edit flow** - Load block, edit, verify autosave
2. **AI enhancement** - Trigger enhancement, verify streaming
3. **History navigation** - Use undo/redo, verify state

### Manual Testing Steps

1. Navigate to `/you`, click a BlockCard
2. Verify `/you/blocks/{label}` route loads
3. Edit content in TipTap editor
4. Wait 200ms, verify "Saved" status appears
5. Check git log for autosave commit
6. Click AI button, select "Enhance"
7. Verify streaming enhancement works
8. Click back button, verify return to `/you`
9. Verify pending diff badge shows correctly
10. Drag/drop image, verify inline display

## Performance Considerations

1. **Debounce autosave at 200ms** - Prevents excessive commits
2. **Lazy load TipTap extensions** - Only load what's needed
3. **Efficient history loading** - Limit to 20 versions by default

## Migration Notes

### Backwards Compatibility

- BlockDetailModal.svelte remains available for diff approval workflow
- Modal can still be opened from BlockEditorPanel for diff review
- No backend changes required - uses existing API

### Breaking Changes

- BlockCard no longer emits `click` event - uses `goto()` instead
- /you page no longer manages `selectedLabel` state

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Linear ticket: [ARI-82](https://linear.app/ariav/issue/ARI-82)
- NoteEditor reference: `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte`
- RichTextInput: `OpenWebUI/open-webui/src/lib/components/common/RichTextInput.svelte`
- Current blocks API: `src/youlab_server/server/blocks.py`
