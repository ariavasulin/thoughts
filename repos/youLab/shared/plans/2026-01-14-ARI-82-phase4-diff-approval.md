# ARI-82 Phase 4: Diff Approval Integration in BlockEditor

## Overview

Add pending diff approval UI directly to the `BlockEditor.svelte` component at `/you/blocks/[label]`. Users will see inline diff visualization with approve/reject buttons, confidence indicators, and agent reasoning without leaving the editor.

## Current State Analysis

### What Already Exists

| Component | Implementation | Location |
|-----------|----------------|----------|
| Route `/you/blocks/[label]` | Uses BlockEditor | `routes/(app)/you/blocks/[label]/+page.svelte:20` |
| BlockEditor | Loads diffs, shows DiffBadge, side panel | `components/you/BlockEditor.svelte` |
| BlockEditorPanel | Shows diff summary (no approve/reject) | `components/you/BlockEditorPanel.svelte:52-74` |
| Backend diff API | Full CRUD for pending diffs | `server/blocks.py:240-279` |
| Frontend API client | `getBlockDiffs`, `approveDiff`, `rejectDiff` | `apis/memory/index.ts:162-334` |
| Diff storage | `PendingDiff` model with confidence, reasoning | `storage/diffs.py:19-68` |

### Current User Flow

1. User opens block at `/you/blocks/student`
2. `BlockEditor` loads pending diffs via `getBlockDiffs()`
3. If diffs exist, `DiffBadge` shows count linking to `/you`
4. Side panel shows diffs with reasoning (no approve/reject)
5. User must navigate away to `/you` to review and approve diffs

### What's Missing

1. **Inline diff visualization** - No red/green deletion/addition view in editor
2. **Confidence indicators** - Backend returns `confidence` but UI ignores it
3. **Approve/Reject buttons** - Only available in BlockDetailModal at `/you`
4. **Multi-diff navigation** - No way to step through multiple diffs
5. **Agent reasoning** - Shows in panel but not with inline diff view

## Desired End State

After implementation:

1. **Diff overlay** appears over the editor when pending diffs exist
2. **Inline visualization** shows deletions (red) and additions (green)
3. **Confidence badges** show low (amber), medium (blue), high (green)
4. **Approve/Reject buttons** for each diff with keyboard shortcuts
5. **Diff navigation** shows "1 of N" with prev/next for multiple diffs
6. **Agent reasoning** displayed below each diff
7. **Auto-refresh** reloads diffs after approve/reject
8. **Toggle** button to dismiss overlay and edit directly

### Verification

```bash
# Manual testing checklist:
# 1. Create pending diff for test block
curl -X POST http://localhost:8100/users/test-user/blocks/student/propose \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "agent_id": "test-agent",
    "operation": "replace",
    "current_value": "old content",
    "proposed_value": "new content",
    "reasoning": "Test diff for verification",
    "confidence": "high"
  }'

# 2. Navigate to /you/blocks/student
# 3. Verify diff overlay appears with:
#    - Confidence badge (green for "high")
#    - Red line showing "old content"
#    - Green line showing "new content"
#    - Reasoning text visible
#    - Approve (green) and Reject (gray) buttons
# 4. Press 'A' key - diff should approve
# 5. Verify UI refreshes and diff is gone
```

## What We're NOT Doing

1. **Not creating MemoryBlockEditor.svelte** - Use existing BlockEditor.svelte
2. **Not modifying notes_adapter.py** - Diff API stays in blocks.py
3. **Not implementing character-level diff** - Line-level is sufficient
4. **Not adding batch approve/reject all** - Per-diff gives user control
5. **Not changing backend diff format** - Current format works

## Implementation Approach

Create a new `DiffApprovalOverlay.svelte` component that:
1. Renders as an overlay on top of the editor
2. Shows unified diff view with line-by-line changes
3. Includes confidence badge, reasoning, and action buttons
4. Handles keyboard shortcuts for efficient review

---

## Phase 4.1: Create DiffApprovalOverlay Component

### Overview

Create a new Svelte component for displaying and approving diffs inline.

### Changes Required:

#### 1. Create DiffApprovalOverlay Component

**File**: `OpenWebUI/open-webui/src/lib/components/you/DiffApprovalOverlay.svelte` (new file)

```svelte
<script lang="ts">
	import { createEventDispatcher, getContext, onDestroy, onMount } from 'svelte';
	import type { PendingDiff } from '$lib/apis/memory';
	import { approveDiff, rejectDiff } from '$lib/apis/memory';
	import { user } from '$lib/stores';

	import CheckCircle from '$lib/components/icons/CheckCircle.svelte';
	import XCircle from '$lib/components/icons/XCircle.svelte';
	import ChevronLeft from '$lib/components/icons/ChevronLeft.svelte';
	import ChevronRight from '$lib/components/icons/ChevronRight.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let label: string;
	export let diffs: PendingDiff[] = [];
	export let currentContent: string = '';

	// Navigation state
	let currentDiffIndex = 0;
	let processing = false;

	// Computed
	$: currentDiff = diffs.length > 0 ? diffs[currentDiffIndex] : null;
	$: hasMultipleDiffs = diffs.length > 1;
	$: canGoPrev = currentDiffIndex > 0;
	$: canGoNext = currentDiffIndex < diffs.length - 1;

	// Confidence badge colors
	function getConfidenceBadge(confidence: string): { bg: string; text: string; label: string } {
		switch (confidence) {
			case 'high':
				return {
					bg: 'bg-green-100 dark:bg-green-900/30',
					text: 'text-green-700 dark:text-green-300',
					label: 'High'
				};
			case 'medium':
				return {
					bg: 'bg-blue-100 dark:bg-blue-900/30',
					text: 'text-blue-700 dark:text-blue-300',
					label: 'Medium'
				};
			case 'low':
				return {
					bg: 'bg-amber-100 dark:bg-amber-900/30',
					text: 'text-amber-700 dark:text-amber-300',
					label: 'Low'
				};
			default:
				return {
					bg: 'bg-gray-100 dark:bg-gray-800',
					text: 'text-gray-700 dark:text-gray-300',
					label: confidence
				};
		}
	}

	// Build diff lines for visualization
	interface DiffLine {
		type: 'context' | 'deletion' | 'addition';
		content: string;
		lineNum?: number;
	}

	function buildDiffLines(diff: PendingDiff | null, content: string): DiffLine[] {
		if (!diff) return [];

		const lines: DiffLine[] = [];
		const contentLines = content.split('\n');

		// Find where the old value appears in content
		const oldValue = (diff as any).oldValue || '';
		const newValue = (diff as any).newValue || '';

		// Simple approach: show context around the change
		let foundIndex = -1;
		if (oldValue) {
			for (let i = 0; i < contentLines.length; i++) {
				if (contentLines[i].includes(oldValue.trim())) {
					foundIndex = i;
					break;
				}
			}
		}

		// Show 2 lines of context before
		const startContext = Math.max(0, foundIndex - 2);
		const endContext = Math.min(contentLines.length - 1, foundIndex + 2);

		for (let i = startContext; i <= endContext; i++) {
			if (i === foundIndex && oldValue) {
				// Deletion line
				lines.push({ type: 'deletion', content: oldValue, lineNum: i + 1 });
				// Addition line
				if (newValue) {
					lines.push({ type: 'addition', content: newValue });
				}
			} else if (i >= 0 && i < contentLines.length) {
				lines.push({ type: 'context', content: contentLines[i], lineNum: i + 1 });
			}
		}

		// If we couldn't find context, just show old/new directly
		if (lines.length === 0) {
			if (oldValue) {
				lines.push({ type: 'deletion', content: oldValue });
			}
			if (newValue) {
				lines.push({ type: 'addition', content: newValue });
			}
		}

		return lines;
	}

	$: diffLines = buildDiffLines(currentDiff, currentContent);
	$: confidenceBadge = currentDiff ? getConfidenceBadge(currentDiff.confidence) : null;

	// Navigation
	function prevDiff() {
		if (canGoPrev) currentDiffIndex--;
	}

	function nextDiff() {
		if (canGoNext) currentDiffIndex++;
	}

	// Actions
	async function handleApprove() {
		if (!currentDiff || processing) return;

		processing = true;
		try {
			await approveDiff($user.id, label, currentDiff.id, localStorage.token);
			dispatch('approved', { diffId: currentDiff.id });

			// Adjust index if needed
			if (currentDiffIndex >= diffs.length - 1 && currentDiffIndex > 0) {
				currentDiffIndex--;
			}
		} catch (e) {
			console.error('Approve failed:', e);
			dispatch('error', { message: `${e}` });
		} finally {
			processing = false;
		}
	}

	async function handleReject() {
		if (!currentDiff || processing) return;

		processing = true;
		try {
			await rejectDiff($user.id, label, currentDiff.id, localStorage.token);
			dispatch('rejected', { diffId: currentDiff.id });

			// Adjust index if needed
			if (currentDiffIndex >= diffs.length - 1 && currentDiffIndex > 0) {
				currentDiffIndex--;
			}
		} catch (e) {
			console.error('Reject failed:', e);
			dispatch('error', { message: `${e}` });
		} finally {
			processing = false;
		}
	}

	function handleDismiss() {
		dispatch('dismiss');
	}

	// Keyboard shortcuts
	function handleKeydown(event: KeyboardEvent) {
		if (processing) return;

		// Ignore if typing in input
		if (event.target instanceof HTMLInputElement || event.target instanceof HTMLTextAreaElement) {
			return;
		}

		switch (event.key.toLowerCase()) {
			case 'a':
				event.preventDefault();
				handleApprove();
				break;
			case 'r':
				event.preventDefault();
				handleReject();
				break;
			case 'arrowleft':
			case 'arrowup':
				event.preventDefault();
				prevDiff();
				break;
			case 'arrowright':
			case 'arrowdown':
				event.preventDefault();
				nextDiff();
				break;
			case 'escape':
			case 'e':
				event.preventDefault();
				handleDismiss();
				break;
		}
	}

	onMount(() => {
		window.addEventListener('keydown', handleKeydown);
	});

	onDestroy(() => {
		window.removeEventListener('keydown', handleKeydown);
	});

	// Reset index when diffs change
	$: if (diffs) {
		currentDiffIndex = Math.min(currentDiffIndex, Math.max(0, diffs.length - 1));
	}
</script>

<div
	class="absolute inset-0 flex flex-col bg-white/95 dark:bg-gray-900/95 backdrop-blur-sm rounded-lg border border-gray-200 dark:border-gray-700 shadow-lg z-20 overflow-hidden"
>
	<!-- Header -->
	<div
		class="flex items-center justify-between px-4 py-3 border-b border-gray-200 dark:border-gray-700 bg-gray-50 dark:bg-gray-850"
	>
		<div class="flex items-center gap-3">
			<span class="text-sm font-medium">
				{diffs.length}
				{diffs.length === 1 ? $i18n.t('pending change') : $i18n.t('pending changes')}
			</span>

			{#if hasMultipleDiffs}
				<div class="flex items-center gap-1">
					<button
						class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-30 transition"
						disabled={!canGoPrev}
						on:click={prevDiff}
						title={$i18n.t('Previous')}
					>
						<ChevronLeft className="size-4" />
					</button>
					<span class="text-xs text-gray-500 min-w-[3rem] text-center">
						{currentDiffIndex + 1} / {diffs.length}
					</span>
					<button
						class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-30 transition"
						disabled={!canGoNext}
						on:click={nextDiff}
						title={$i18n.t('Next')}
					>
						<ChevronRight className="size-4" />
					</button>
				</div>
			{/if}
		</div>

		<button
			class="text-sm text-gray-500 hover:text-gray-700 dark:hover:text-gray-300 transition"
			on:click={handleDismiss}
		>
			{$i18n.t('Edit Directly')} (E)
		</button>
	</div>

	<!-- Diff Content -->
	<div class="flex-1 overflow-auto">
		{#if currentDiff}
			<!-- Confidence and reasoning -->
			<div class="px-4 py-3 border-b border-gray-100 dark:border-gray-800">
				<div class="flex items-center gap-2 mb-2">
					{#if confidenceBadge}
						<span
							class="px-2 py-0.5 rounded text-xs font-medium {confidenceBadge.bg} {confidenceBadge.text}"
						>
							{confidenceBadge.label} confidence
						</span>
					{/if}
					<span class="text-xs text-gray-500">{currentDiff.operation}</span>
				</div>
				<p class="text-sm text-gray-600 dark:text-gray-400 italic">
					"{currentDiff.reasoning}"
				</p>
			</div>

			<!-- Diff lines -->
			<div class="font-mono text-sm">
				{#each diffLines as line}
					{#if line.type === 'context'}
						<div class="px-4 py-0.5 flex text-gray-600 dark:text-gray-400">
							<span class="w-8 text-right mr-3 text-gray-400 select-none">{line.lineNum || ''}</span>
							<span class="flex-1 whitespace-pre-wrap">{line.content}</span>
						</div>
					{:else if line.type === 'deletion'}
						<div
							class="px-4 py-0.5 flex bg-red-50 dark:bg-red-900/20 text-red-800 dark:text-red-200"
						>
							<span class="w-8 text-right mr-3 text-red-400 select-none"
								>{line.lineNum || ''}</span
							>
							<span class="w-4 text-center text-red-500 select-none">-</span>
							<span class="flex-1 whitespace-pre-wrap">{line.content}</span>
						</div>
					{:else if line.type === 'addition'}
						<div
							class="px-4 py-0.5 flex bg-green-50 dark:bg-green-900/20 text-green-800 dark:text-green-200"
						>
							<span class="w-8 text-right mr-3 text-green-400 select-none"></span>
							<span class="w-4 text-center text-green-500 select-none">+</span>
							<span class="flex-1 whitespace-pre-wrap">{line.content}</span>
						</div>
					{/if}
				{/each}
			</div>
		{/if}
	</div>

	<!-- Action buttons -->
	<div
		class="flex items-center justify-between px-4 py-3 border-t border-gray-200 dark:border-gray-700 bg-gray-50 dark:bg-gray-850"
	>
		<div class="text-xs text-gray-400">
			<kbd class="px-1.5 py-0.5 bg-gray-200 dark:bg-gray-700 rounded">A</kbd> approve ·
			<kbd class="px-1.5 py-0.5 bg-gray-200 dark:bg-gray-700 rounded">R</kbd> reject ·
			<kbd class="px-1.5 py-0.5 bg-gray-200 dark:bg-gray-700 rounded">←</kbd><kbd
				class="px-1.5 py-0.5 bg-gray-200 dark:bg-gray-700 rounded">→</kbd
			> navigate
		</div>

		<div class="flex items-center gap-2">
			<button
				class="flex items-center gap-1.5 px-3 py-1.5 bg-gray-500 hover:bg-gray-600 text-white rounded text-sm transition disabled:opacity-50"
				disabled={processing}
				on:click={handleReject}
			>
				<XCircle className="size-4" />
				{$i18n.t('Reject')}
			</button>
			<button
				class="flex items-center gap-1.5 px-3 py-1.5 bg-green-600 hover:bg-green-700 text-white rounded text-sm transition disabled:opacity-50"
				disabled={processing}
				on:click={handleApprove}
			>
				<CheckCircle className="size-4" />
				{$i18n.t('Approve')}
			</button>
		</div>
	</div>
</div>
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`
- [ ] Component exports correctly

#### Manual Verification:
- [ ] Component renders without errors when imported
- [ ] Keyboard shortcuts work (test in isolation)

**Implementation Note**: After completing this phase, proceed to Phase 4.2 to integrate the component.

---

## Phase 4.2: Integrate Overlay into BlockEditor

### Overview

Add the `DiffApprovalOverlay` to `BlockEditor.svelte` and wire up the approve/reject handlers.

### Changes Required:

#### 1. Import and Add Overlay to BlockEditor

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`

Add import after existing imports (around line 38):

```typescript
import DiffApprovalOverlay from './DiffApprovalOverlay.svelte';
```

Add state variable after existing state (around line 65):

```typescript
// Diff approval UI state
let showDiffOverlay = true; // Show by default when diffs exist
```

Add handler functions after existing handlers (around line 280):

```typescript
// Diff approval handlers
async function handleDiffApproved(event: CustomEvent<{ diffId: string }>) {
	toast.success($i18n.t('Change approved'));
	await init(); // Reload block and diffs
}

async function handleDiffRejected(event: CustomEvent<{ diffId: string }>) {
	toast.success($i18n.t('Change rejected'));
	await init(); // Reload block and diffs
}

async function handleDiffError(event: CustomEvent<{ message: string }>) {
	toast.error(event.detail.message);
}

function handleDiffDismiss() {
	showDiffOverlay = false;
}
```

#### 2. Update Template to Include Overlay

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`

Replace the header section (lines 321-380) to include a "Review Changes" button and overlay:

```svelte
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
		<!-- Review changes button (when diffs exist and overlay is hidden) -->
		{#if pendingDiffs.length > 0 && !showDiffOverlay}
			<button
				class="px-2 py-1 text-sm text-amber-600 dark:text-amber-400 hover:bg-amber-50 dark:hover:bg-amber-900/20 rounded transition"
				on:click={() => (showDiffOverlay = true)}
			>
				{$i18n.t('Review Changes')}
			</button>
		{/if}

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
```

#### 3. Add Overlay to Editor Container

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`

Update the editor container (around line 393) to include the overlay:

```svelte
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
		className="input-prose-sm px-0.5 h-[calc(100%-2rem)] {pendingDiffs.length > 0 && showDiffOverlay
			? 'opacity-30 pointer-events-none'
			: ''}"
		html={marked.parse(markdownContent)}
		placeholder={$i18n.t('Start writing...')}
		editable={!editing && !(pendingDiffs.length > 0 && showDiffOverlay)}
		fileHandler={true}
		image={true}
		{files}
		onFileDrop={(currentEditor, droppedFiles, pos) => {
			droppedFiles.forEach(async (file: File) => {
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

	<!-- Diff Approval Overlay -->
	{#if pendingDiffs.length > 0 && showDiffOverlay}
		<DiffApprovalOverlay
			{label}
			diffs={pendingDiffs}
			currentContent={markdownContent}
			on:approved={handleDiffApproved}
			on:rejected={handleDiffRejected}
			on:error={handleDiffError}
			on:dismiss={handleDiffDismiss}
		/>
	{/if}
</div>
```

#### 4. Reset Overlay State When Navigating

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`

Add reactive statement after state declarations (around line 75):

```typescript
// Reset diff overlay when diffs change
$: if (pendingDiffs.length > 0) {
	showDiffOverlay = true;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`
- [ ] Svelte check passes: `npm run check`

#### Manual Verification:
- [ ] Navigate to `/you/blocks/student` with pending diffs
- [ ] Diff overlay appears automatically
- [ ] Editor is dimmed behind overlay
- [ ] "Edit Directly" button dismisses overlay
- [ ] "Review Changes" button shows overlay again
- [ ] Approve button works and refreshes
- [ ] Reject button works and refreshes
- [ ] Keyboard shortcuts (A/R/arrows/E) work

**Implementation Note**: After completing this phase, proceed to Phase 4.3 for extended diff data.

---

## Phase 4.3: Add oldValue/newValue to PendingDiff Type

### Overview

The frontend API client expects `oldValue` and `newValue` on diffs. Ensure the `PendingDiff` type includes these values properly.

### Changes Required:

#### 1. Verify API Response Includes Extended Fields

**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`

The `getBlockDiffs` function (lines 162-202) already maps these fields:

```typescript
// Around lines 199-200, verify this mapping exists:
oldValue: (diff as any).oldValue,
newValue: (diff as any).newValue,
```

#### 2. Extend PendingDiff Type

**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`

Update the `PendingDiff` interface (lines 151-160) to include optional value fields:

```typescript
export interface PendingDiff {
	id: string;
	block: string;
	field: string | null;
	operation: string;
	reasoning: string;
	confidence: string;
	createdAt: string;
	agentId: string;
	oldValue?: string;  // Add this
	newValue?: string;  // Add this
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors
- [ ] Type includes `oldValue` and `newValue`

#### Manual Verification:
- [ ] API response includes oldValue/newValue for replace operations
- [ ] Diff overlay shows correct old/new content

---

## Phase 4.4: Update BlockEditorPanel to Link to Overlay

### Overview

Update the side panel to open the diff overlay instead of linking to `/you`.

### Changes Required:

#### 1. Change Panel Link to Dispatch Event

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditorPanel.svelte`

Add event dispatcher and replace the link:

```svelte
<script lang="ts">
	import { createEventDispatcher, getContext } from 'svelte';
	// ... existing imports ...

	const dispatch = createEventDispatcher();

	// ... existing code ...
</script>

<!-- Replace the link (lines 67-72) with: -->
{#if pendingDiffs.length > 0}
	<div class="mb-4">
		<span class="block text-sm font-medium mb-2">{$i18n.t('Pending Changes')}</span>
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
		<button
			class="block w-full text-center text-sm text-blue-600 dark:text-blue-400 mt-2 hover:underline"
			on:click={() => dispatch('showDiffOverlay')}
		>
			{$i18n.t('Review Changes')}
		</button>
	</div>
{/if}
```

#### 2. Handle Event in BlockEditor

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`

Update the BlockEditorPanel usage (around line 475):

```svelte
<BlockEditorPanel
	bind:show={showPanel}
	bind:selectedModelId
	{selectedPanel}
	{files}
	{label}
	{pendingDiffs}
	on:showDiffOverlay={() => {
		showDiffOverlay = true;
		showPanel = false;
	}}
/>
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles
- [ ] Event handling works

#### Manual Verification:
- [ ] Click "Review Changes" in side panel
- [ ] Diff overlay appears
- [ ] Side panel closes

---

## Testing Strategy

### Unit Tests

No new backend tests needed - this phase is frontend-only.

### Manual Testing Steps

1. **Setup test data**:
   ```bash
   # Create a test diff
   curl -X POST http://localhost:8100/users/$USER_ID/blocks/student/propose \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
       "agent_id": "test-agent",
       "operation": "replace",
       "current_value": "I am a student",
       "proposed_value": "I am an enthusiastic learner",
       "reasoning": "Made the description more engaging",
       "confidence": "high"
     }'
   ```

2. **Test overlay appearance**:
   - Navigate to `/you/blocks/student`
   - Verify overlay appears with diff visualization
   - Verify confidence badge shows "High" in green
   - Verify reasoning text is visible

3. **Test diff visualization**:
   - Red line shows "I am a student" with minus sign
   - Green line shows "I am an enthusiastic learner" with plus sign
   - Context lines visible around changes

4. **Test approval workflow**:
   - Click "Approve" or press 'A'
   - Verify toast shows success
   - Verify overlay disappears (no more diffs)
   - Verify block content updated

5. **Test rejection workflow**:
   - Create another test diff
   - Click "Reject" or press 'R'
   - Verify toast shows success
   - Verify overlay disappears
   - Verify block content unchanged

6. **Test multi-diff navigation**:
   - Create 3 test diffs
   - Verify "1 / 3" counter appears
   - Press → to navigate to diff 2
   - Press ← to navigate back
   - Approve diff 2, verify counter updates to "1 / 2"

7. **Test edit mode toggle**:
   - Press 'E' or click "Edit Directly"
   - Verify overlay hides
   - Verify editor is editable
   - Click "Review Changes"
   - Verify overlay returns

8. **Test keyboard shortcuts**:
   - 'A' approves current diff
   - 'R' rejects current diff
   - '←' goes to previous diff
   - '→' goes to next diff
   - 'E' or 'Escape' dismisses overlay

## Performance Considerations

1. **Keyboard listener lifecycle**: Added/removed with component mount/destroy
2. **Diff line building**: Only rebuilds when currentDiff or content changes
3. **Auto-refresh**: Reloads entire block data after approve/reject (acceptable for MVP)

## Migration Notes

- No data migration needed
- Existing diffs work with new UI immediately
- BlockDetailModal at `/you` remains functional (parallel path)

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Phase 3 plan: `thoughts/shared/plans/2026-01-14-ARI-82-phase3-git-storage-adapter.md`
- BlockEditor: `OpenWebUI/open-webui/src/lib/components/you/BlockEditor.svelte`
- Backend diff API: `src/youlab_server/server/blocks.py:240-279`
- Frontend diff API: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts:162-334`
- Diff storage: `src/youlab_server/storage/diffs.py`
- Linear ticket: [ARI-82](https://linear.app/ariav/issue/ARI-82)
