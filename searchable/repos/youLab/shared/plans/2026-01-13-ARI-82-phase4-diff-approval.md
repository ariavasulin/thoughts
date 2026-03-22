# ARI-82 Phase 4: Diff Approval Integration Enhancement

## Overview

Enhance the existing diff approval UI in `BlockDetailModal.svelte` with confidence indicators, overlay-style visualization, and improved multi-diff handling. The current implementation already supports line-based diffs with approve/reject - this phase adds polish and UX improvements.

## Current State Analysis

### What Already Exists

| Feature | Status | Location |
|---------|--------|----------|
| Pending diff display | Done | `BlockDetailModal.svelte:267-320` |
| Red/green line diff view | Done | Lines 276-318 |
| Approve/Reject buttons | Done | Lines 298-312 |
| Reasoning display | Done | Lines 293-297 |
| Auto-refresh after action | Done | `loadBlock()` in handlers |
| Backend API endpoints | Done | `server/blocks.py:234-273` |

### What's Missing

1. **Confidence indicator** - Backend returns `confidence` but UI doesn't display it
2. **Overlay-style visualization** - Currently switches between modes, not overlay
3. **Multiple diff navigation** - No diff counter or navigation
4. **Keyboard shortcuts** - No hotkeys for approve/reject
5. **Character-level highlighting** - Current is line-level only

## Desired End State

After implementation:

1. **Confidence badges** shown next to each diff (low=yellow, medium=blue, high=green)
2. **Inline diff overlay** that shows diffs without hiding the editor completely
3. **Diff navigator** showing "1 of N" with prev/next buttons
4. **Keyboard shortcuts**: `A` to approve, `R` to reject, `←/→` to navigate diffs
5. **Smoother transitions** between diff view and edit view

### Verification

```bash
# Manual testing checklist:
# 1. Open block with pending diff
# 2. Verify confidence badge visible
# 3. Press 'A' key - diff should approve
# 4. Create block with 3+ pending diffs
# 5. Verify diff counter and navigation
# 6. Verify editor still partially visible behind overlay
```

## What We're NOT Doing

1. **Not implementing character-level diff** - Line-level is sufficient for MVP
2. **Not adding batch approve/reject all** - Per-diff approval gives user control
3. **Not changing backend diff format** - Current oldValue/newValue works
4. **Not adding diff preview before apply** - User can see the change inline

## Implementation Approach

Enhance the existing `BlockDetailModal.svelte` incrementally:
1. Add confidence badge (quick win)
2. Add diff navigation for multiple diffs
3. Convert to overlay-style presentation
4. Add keyboard shortcuts

---

## Phase 4.1: Confidence Indicator

### Overview

Display confidence level badge for each pending diff using the existing `confidence` field.

### Changes Required:

#### 1. Add confidence badge styling

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add helper function after `formatDate` (around line 229):

```typescript
function getConfidenceBadge(confidence: string): { color: string; text: string } {
    switch (confidence) {
        case 'high':
            return { color: 'bg-green-100 text-green-700 dark:bg-green-900/30 dark:text-green-300', text: 'High' };
        case 'medium':
            return { color: 'bg-blue-100 text-blue-700 dark:bg-blue-900/30 dark:text-blue-300', text: 'Medium' };
        case 'low':
            return { color: 'bg-amber-100 text-amber-700 dark:bg-amber-900/30 dark:text-amber-300', text: 'Low' };
        default:
            return { color: 'bg-gray-100 text-gray-700 dark:bg-gray-800 dark:text-gray-300', text: confidence };
    }
}
```

#### 2. Update diff line rendering to show confidence

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Update the reasoning/actions block (around line 293-315) to include confidence badge:

```svelte
{#if line.isLastInDiff && line.diff}
    <div class="px-3 py-1.5 border-t border-green-200 dark:border-green-800/50 flex items-center justify-between gap-2 bg-green-50 dark:bg-green-900/20">
        <div class="flex items-center gap-2 min-w-0">
            <!-- Confidence Badge -->
            {@const badge = getConfidenceBadge(line.diff.confidence)}
            <span class="px-1.5 py-0.5 rounded text-xs font-medium {badge.color} shrink-0">
                {badge.text}
            </span>
            <span class="text-xs text-green-700 dark:text-green-300 italic truncate">
                {line.diff.reasoning}
            </span>
        </div>
        <div class="flex gap-1.5 shrink-0">
            <button
                class="px-2 py-1 bg-green-600 hover:bg-green-700 text-white rounded text-xs transition disabled:opacity-50"
                on:click={() => handleApprove(line.diff?.id || '')}
                disabled={processing}
            >
                {$i18n.t('Approve')}
            </button>
            <button
                class="px-2 py-1 bg-gray-500 hover:bg-gray-600 text-white rounded text-xs transition disabled:opacity-50"
                on:click={() => handleReject(line.diff?.id || '')}
                disabled={processing}
            >
                {$i18n.t('Reject')}
            </button>
        </div>
    </div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`
- [ ] Svelte check passes: `npm run check`

#### Manual Verification:
- [ ] Open block with low confidence diff - yellow badge visible
- [ ] Open block with medium confidence diff - blue badge visible
- [ ] Open block with high confidence diff - green badge visible

**Implementation Note**: This is a quick enhancement. Proceed to Phase 4.2 after verification.

---

## Phase 4.2: Diff Navigation

### Overview

Add navigation controls for blocks with multiple pending diffs, allowing users to focus on one diff at a time.

### Changes Required:

#### 1. Add navigation state

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add state variables after existing let declarations (around line 34):

```typescript
let currentDiffIndex = 0;

// Compute focused diff for navigation
$: focusedDiff = diffsWithValues.length > 0 ? diffsWithValues[currentDiffIndex] : null;
$: hasMultipleDiffs = diffsWithValues.length > 1;

function nextDiff() {
    if (currentDiffIndex < diffsWithValues.length - 1) {
        currentDiffIndex++;
    }
}

function prevDiff() {
    if (currentDiffIndex > 0) {
        currentDiffIndex--;
    }
}

// Reset index when diffs change
$: if (diffsWithValues) {
    currentDiffIndex = 0;
}
```

#### 2. Add navigation controls to header

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Update the diff count display in header (around line 270-273):

```svelte
<div class="bg-gray-50 dark:bg-gray-850 px-3 py-2 border-b border-gray-200 dark:border-gray-700 flex items-center justify-between">
    <span class="text-sm font-medium text-gray-700 dark:text-gray-300">
        {diffsWithValues.length} {$i18n.t('pending')} {diffsWithValues.length === 1 ? $i18n.t('change') : $i18n.t('changes')}
    </span>
    {#if hasMultipleDiffs}
        <div class="flex items-center gap-2">
            <button
                class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-30 transition"
                disabled={currentDiffIndex === 0}
                on:click={prevDiff}
                title={$i18n.t('Previous change')}
            >
                <svg class="size-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                </svg>
            </button>
            <span class="text-xs text-gray-500">
                {currentDiffIndex + 1} / {diffsWithValues.length}
            </span>
            <button
                class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-30 transition"
                disabled={currentDiffIndex === diffsWithValues.length - 1}
                on:click={nextDiff}
                title={$i18n.t('Next change')}
            >
                <svg class="size-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                </svg>
            </button>
        </div>
    {/if}
</div>
```

#### 3. Highlight focused diff

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Update buildDiffView to track which diff is focused:

```typescript
function buildDiffView() {
    if (diffsWithValues.length === 0 || editMode) {
        diffLines = [];
        return;
    }

    const lines = tomlContent.split('\n');
    const result: DiffLine[] = [];

    // Create a map of line numbers to diffs
    const diffsByLine: Map<number, DiffWithValues> = new Map();
    for (const diff of diffsWithValues) {
        if (diff.lineNumber !== undefined) {
            diffsByLine.set(diff.lineNumber, diff);
        } else if (diff.oldValue) {
            const lineIdx = lines.findIndex(line => line.trim() === diff.oldValue?.trim());
            if (lineIdx >= 0) {
                diff.lineNumber = lineIdx;
                diffsByLine.set(lineIdx, diff);
            }
        }
    }

    // Build the unified view
    for (let i = 0; i < lines.length; i++) {
        const diff = diffsByLine.get(i);
        if (diff) {
            const isFocused = focusedDiff?.id === diff.id;

            if (diff.oldValue) {
                result.push({
                    type: 'deletion',
                    content: diff.oldValue,
                    lineNum: i + 1,
                    diffId: diff.id,
                    diff: diff,
                    isFocused
                });
            }
            if (diff.newValue) {
                result.push({
                    type: 'addition',
                    content: diff.newValue,
                    diffId: diff.id,
                    diff: diff,
                    isLastInDiff: true,
                    isFocused
                });
            }
        } else {
            result.push({
                type: 'context',
                content: lines[i],
                lineNum: i + 1
            });
        }
    }

    diffLines = result;
}

// Rebuild when focused diff changes
$: if (tomlContent && diffsWithValues && focusedDiff) {
    buildDiffView();
}
```

#### 4. Update DiffLine interface

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add `isFocused` to interface (around line 46):

```typescript
interface DiffLine {
    type: 'context' | 'deletion' | 'addition';
    content: string;
    lineNum?: number;
    diffId?: string;
    diff?: DiffWithValues;
    isLastInDiff?: boolean;
    isFocused?: boolean;
}
```

#### 5. Style focused diff

Update the diff line rendering to highlight focused diff:

```svelte
{#if line.type === 'deletion'}
    <div class="px-3 py-0.5 {line.isFocused ? 'bg-red-200 dark:bg-red-800/50 ring-2 ring-red-400' : 'bg-red-100 dark:bg-red-900/30'} text-red-800 dark:text-red-200 flex">
        <!-- ... existing content ... -->
    </div>
{:else if line.type === 'addition'}
    <div class="{line.isFocused ? 'bg-green-200 dark:bg-green-800/50 ring-2 ring-green-400' : 'bg-green-100 dark:bg-green-900/30'}">
        <!-- ... existing content ... -->
    </div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] Create block with 3 pending diffs
- [ ] Verify "1 / 3" counter appears
- [ ] Click next arrow - counter shows "2 / 3", diff 2 highlighted
- [ ] Click prev arrow - counter shows "1 / 3", diff 1 highlighted

**Implementation Note**: Proceed to Phase 4.3 after verification.

---

## Phase 4.3: Keyboard Shortcuts

### Overview

Add keyboard shortcuts for efficient diff review: `A` to approve, `R` to reject, arrow keys to navigate.

### Changes Required:

#### 1. Add keyboard event handler

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add keyboard handler and mount/destroy lifecycle:

```typescript
import { onMount, onDestroy, createEventDispatcher, getContext } from 'svelte';

// Add after other handlers
function handleKeydown(event: KeyboardEvent) {
    // Only handle shortcuts when viewing diffs (not editing)
    if (editMode || !showDiffView || processing) return;

    // Ignore if user is typing in an input
    if (event.target instanceof HTMLInputElement || event.target instanceof HTMLTextAreaElement) {
        return;
    }

    switch (event.key.toLowerCase()) {
        case 'a':
            // Approve focused diff
            if (focusedDiff) {
                event.preventDefault();
                handleApprove(focusedDiff.id);
            }
            break;
        case 'r':
            // Reject focused diff
            if (focusedDiff) {
                event.preventDefault();
                handleReject(focusedDiff.id);
            }
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
        case 'e':
            // Toggle edit mode
            event.preventDefault();
            editMode = !editMode;
            break;
    }
}

onMount(() => {
    window.addEventListener('keydown', handleKeydown);
});

onDestroy(() => {
    window.removeEventListener('keydown', handleKeydown);
});
```

#### 2. Add keyboard shortcut hints

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add hint text below the diff viewer (after line 319):

```svelte
{#if showDiffView && !processing}
    <div class="px-3 py-1.5 text-xs text-gray-400 dark:text-gray-500 border-t border-gray-100 dark:border-gray-800">
        <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">A</kbd> {$i18n.t('approve')} ·
        <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">R</kbd> {$i18n.t('reject')} ·
        <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">←</kbd><kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">→</kbd> {$i18n.t('navigate')} ·
        <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">E</kbd> {$i18n.t('edit')}
    </div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] Open block with pending diffs
- [ ] Press 'A' key - focused diff approved, removed from view
- [ ] Press 'R' key - focused diff rejected, removed from view
- [ ] Press arrow keys - navigation works
- [ ] Press 'E' key - toggles to edit mode
- [ ] Keyboard hints visible at bottom

**Implementation Note**: Proceed to Phase 4.4 after verification.

---

## Phase 4.4: Overlay-Style Presentation

### Overview

Change from full mode-switch to overlay that shows diffs while keeping editor visible in background. Uses a semi-transparent backdrop with focused diff panel.

### Changes Required:

#### 1. Restructure the main content area

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Replace the conditional diff view / edit view with overlaid structure:

```svelte
<div class="flex-1 min-w-0 relative">
    <!-- Base Editor (always visible when not loading) -->
    <Textarea
        bind:value={markdownContent}
        placeholder={$i18n.t('Memory block content (Markdown)...')}
        className="w-full rounded-lg px-3.5 py-2 text-sm bg-gray-50 dark:text-gray-300 dark:bg-gray-850 outline-hidden font-mono min-h-[300px] {showDiffView ? 'opacity-30 pointer-events-none' : ''}"
        disabled={showDiffView}
    />

    <!-- Diff Overlay (when diffs exist and not editing) -->
    {#if showDiffView}
        <div class="absolute inset-0 flex flex-col rounded-lg overflow-hidden bg-white/95 dark:bg-gray-900/95 backdrop-blur-sm shadow-lg border border-gray-200 dark:border-gray-700">
            <!-- Header -->
            <div class="bg-gray-50 dark:bg-gray-850 px-3 py-2 border-b border-gray-200 dark:border-gray-700 flex items-center justify-between">
                <span class="text-sm font-medium text-gray-700 dark:text-gray-300">
                    {diffsWithValues.length} {$i18n.t('pending')} {diffsWithValues.length === 1 ? $i18n.t('change') : $i18n.t('changes')}
                </span>
                {#if hasMultipleDiffs}
                    <div class="flex items-center gap-2">
                        <button
                            class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-30 transition"
                            disabled={currentDiffIndex === 0}
                            on:click={prevDiff}
                        >
                            <svg class="size-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7" />
                            </svg>
                        </button>
                        <span class="text-xs text-gray-500">{currentDiffIndex + 1} / {diffsWithValues.length}</span>
                        <button
                            class="p-1 rounded hover:bg-gray-200 dark:hover:bg-gray-700 disabled:opacity-30 transition"
                            disabled={currentDiffIndex === diffsWithValues.length - 1}
                            on:click={nextDiff}
                        >
                            <svg class="size-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                            </svg>
                        </button>
                    </div>
                {/if}
            </div>

            <!-- Diff Content -->
            <div class="flex-1 font-mono text-sm overflow-auto">
                {#each diffLines as line, idx (idx)}
                    <!-- ... existing diff line rendering ... -->
                {/each}
            </div>

            <!-- Keyboard Hints -->
            <div class="px-3 py-1.5 text-xs text-gray-400 dark:text-gray-500 border-t border-gray-100 dark:border-gray-800">
                <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">A</kbd> {$i18n.t('approve')} ·
                <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">R</kbd> {$i18n.t('reject')} ·
                <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">←</kbd><kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">→</kbd> {$i18n.t('navigate')} ·
                <kbd class="px-1 py-0.5 bg-gray-100 dark:bg-gray-800 rounded">E</kbd> {$i18n.t('edit')}
            </div>
        </div>
    {/if}
</div>
```

#### 2. Update toggle button

Change the "Edit" / "View Changes" toggle to a cleaner dismiss action:

```svelte
{#if hasDiffs}
    <button
        class="text-sm transition {editMode
            ? 'text-amber-600 dark:text-amber-400 font-medium'
            : 'text-gray-500 hover:text-gray-700 dark:hover:text-gray-300'}"
        on:click={() => (editMode = !editMode)}
    >
        {editMode ? $i18n.t('Review Changes') : $i18n.t('Edit Directly')}
    </button>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] Open block with diffs - overlay appears over editor
- [ ] Editor content dimly visible behind overlay
- [ ] Click "Edit Directly" - overlay hides, editor fully visible
- [ ] Click "Review Changes" - overlay returns

---

## Testing Strategy

### Unit Tests

No new backend tests needed - this phase is frontend-only.

### Integration Tests

Not applicable - UI changes only.

### Manual Testing Steps

1. **Setup**: Create test block with agent-proposed diffs of varying confidence
2. **Confidence badges**: Verify low/medium/high display correctly
3. **Multiple diffs**: Create 3+ diffs, test navigation
4. **Keyboard shortcuts**: Test A/R/arrows/E keys
5. **Overlay behavior**: Verify editor visible behind overlay
6. **Auto-refresh**: Approve diff, verify list updates

### Test Data Setup

```bash
# Create test diffs via API
curl -X POST http://localhost:8080/users/test-user/blocks/student/propose \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "test-agent",
    "field": null,
    "operation": "replace",
    "current_value": "old text",
    "proposed_value": "new text",
    "reasoning": "Test diff with low confidence",
    "confidence": "low"
  }'
```

## Performance Considerations

1. **Keyboard listener**: Added/removed with component lifecycle, no memory leaks
2. **Diff rebuild**: Only triggers when content or focused diff changes
3. **Overlay rendering**: CSS-only blur/opacity, no JavaScript animation overhead

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Current UI: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`
- Backend API: `src/youlab_server/server/blocks.py:234-273`
- Diff storage: `src/youlab_server/storage/diffs.py`
