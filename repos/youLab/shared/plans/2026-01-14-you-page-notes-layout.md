# You Page Notes-Style Layout Implementation Plan

## Overview

Replace the existing `/you` page with a layout that mirrors the `/notes` page exactly, repurposing the Notes components for memory blocks with the only addition being pending diff badges.

## Current State Analysis

**Current `/you` page (`routes/(app)/you/+page.svelte`):**
- Has tabs: "Profile" and "Agents" (to be removed)
- Shows memory blocks in a simple 1-2 column grid using `BlockCard.svelte`
- `BlockCard` already has `Badge` component for pending diffs
- No search, no filters, no time-based grouping

**Current `/notes` page (`lib/components/notes/Notes.svelte`):**
- Search bar at the top
- Filter dropdowns (view: All/Created by you/Shared with you, permission: Write/Read Only)
- Display toggle (List/Grid)
- Time-based grouping (Today, Yesterday, Last 7 Days, etc.)
- List mode: compact row with title, timestamp, author, context menu
- Grid mode: cards with content preview
- Paginated with infinite scroll
- Drag-and-drop markdown file import

**Backend API (`server/blocks.py`):**
- `GET /users/{user_id}/blocks` returns `[{label, pending_diffs}]`
- Missing: `updated_at`, `title`, search/filter support, pagination

### Key Discoveries:
- `BlockSummary` model at `server/blocks.py:35` only returns `label` and `pending_diffs`
- Frontmatter includes `updated_at` (set in `storage/git.py:62`) but not exposed in list endpoint
- Memory blocks don't have "sharing" concept - filters will need adaptation
- Frontend API client at `lib/apis/memory/index.ts` needs matching updates

## Desired End State

The `/you` page renders identically to `/notes` with:
1. Search bar filtering blocks by label/title
2. Display toggle (List/Grid view)
3. Time-based grouping by `updated_at`
4. List view: title, timestamp, pending diff badge, context menu
5. Grid view: card with content preview, pending diff badge
6. Context menu with: Download (txt/md), Copy link, Delete (if applicable)
7. Infinite scroll pagination (for future when users have many blocks)

### Verification:
- Visual comparison: `/you` should look indistinguishable from `/notes` (except badges)
- All Notes features work: search, filter, list/grid toggle, time grouping
- Clicking a block navigates to `/you/blocks/${label}`
- Pending diff badges appear on blocks with pending changes

## What We're NOT Doing

- Keeping the "Agents" tab (removed entirely)
- Implementing "Shared with you" filter (memory blocks aren't shared)
- Implementing "New Block" button (blocks are created by the system, not users)
- Implementing delete functionality (blocks are system-managed)
- Drag-and-drop file import (blocks aren't created from markdown files)

## Implementation Approach

Mirror the Notes architecture exactly:
1. Extend backend to return data needed for Notes-style display
2. Create `Blocks.svelte` component by adapting `Notes.svelte`
3. Create `BlockMenu.svelte` by adapting `NoteMenu.svelte`
4. Update `/you/+page.svelte` to use `Blocks.svelte`
5. Update frontend API client for new response shape

---

## Phase 1: Backend API Extensions

### Overview
Extend the blocks API to return the data needed for Notes-style display (timestamps, titles, content preview) and add search/pagination support.

### Changes Required:

#### 1. Update BlockSummary Response Model
**File**: `src/youlab_server/server/blocks.py`
**Changes**: Add `updated_at`, `title`, and `content_preview` to BlockSummary

```python
class BlockSummary(BaseModel):
    """Summary of a memory block."""

    label: str
    title: str  # Display title (from frontmatter or derived from label)
    updated_at: int  # Unix timestamp in nanoseconds (matching notes format)
    pending_diffs: int
    content_preview: str | None = None  # First ~100 chars of body for grid view
```

#### 2. Update list_blocks Endpoint
**File**: `src/youlab_server/server/blocks.py`
**Changes**: Fetch metadata for each block to populate new fields, add search/pagination

```python
@router.get("", response_model=list[BlockSummary])
async def list_blocks(
    user_id: str,
    storage: StorageDep,
    query: Annotated[str | None, Query()] = None,
    page: Annotated[int, Query(ge=1)] = 1,
    limit: Annotated[int, Query(ge=1, le=100)] = 50,
) -> list[BlockSummary]:
    """List all memory blocks for a user with optional search."""
    manager = get_block_manager(user_id, storage)
    labels = manager.list_blocks()
    counts = manager.count_pending_diffs()

    results = []
    for label in labels:
        metadata = manager.get_block_metadata(label) or {}
        body = manager.get_block_body(label) or ""

        # Get title from metadata or derive from label
        title = metadata.get("title") or label.replace("_", " ").title()

        # Parse updated_at from ISO string to nanosecond timestamp
        updated_at_str = metadata.get("updated_at", "")
        try:
            from datetime import datetime
            dt = datetime.fromisoformat(updated_at_str.replace("Z", "+00:00"))
            updated_at = int(dt.timestamp() * 1_000_000_000)  # nanoseconds
        except (ValueError, AttributeError):
            updated_at = 0

        # Content preview for grid view
        content_preview = body[:150] if body else None

        # Filter by query if provided
        if query:
            query_lower = query.lower()
            if query_lower not in label.lower() and query_lower not in title.lower():
                continue

        results.append(BlockSummary(
            label=label,
            title=title,
            updated_at=updated_at,
            pending_diffs=counts.get(label, 0),
            content_preview=content_preview,
        ))

    # Sort by updated_at descending (newest first)
    results.sort(key=lambda x: x.updated_at, reverse=True)

    # Pagination
    start = (page - 1) * limit
    end = start + limit
    return results[start:end]
```

#### 3. Add Total Count Endpoint
**File**: `src/youlab_server/server/blocks.py`
**Changes**: Add endpoint for total count (for pagination UI)

```python
@router.get("/count")
async def get_block_count(
    user_id: str,
    storage: StorageDep,
    query: Annotated[str | None, Query()] = None,
) -> dict[str, int]:
    """Get total count of blocks (optionally filtered by query)."""
    manager = get_block_manager(user_id, storage)
    labels = manager.list_blocks()

    if query:
        query_lower = query.lower()
        labels = [l for l in labels if query_lower in l.lower()]

    return {"total": len(labels)}
```

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Existing block tests pass: `make test-agent`
- [ ] New endpoint returns correct shape: manual curl test

#### Manual Verification:
- [ ] `GET /users/{userId}/blocks` returns `updated_at`, `title`, `content_preview`
- [ ] Search query filters results correctly
- [ ] Results sorted by `updated_at` descending

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 2: Frontend API Client Updates

### Overview
Update the frontend API client to handle the new response shape and add search/pagination support.

### Changes Required:

#### 1. Update Types and Transform Function
**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`
**Changes**: Update MemoryBlock type and add search/pagination

```typescript
// Update the transform function
function transformBlock(data: any): MemoryBlock {
    return {
        label: data.label,
        title: data.title,
        updatedAt: data.updated_at,  // nanosecond timestamp
        pendingDiffs: data.pending_diffs ?? 0,
        contentPreview: data.content_preview ?? null
    };
}

// Add search function
export async function searchBlocks(
    userId: string,
    token: string,
    query: string = '',
    page: number = 1
): Promise<{ items: MemoryBlock[]; total: number }> {
    let error = null;

    const params = new URLSearchParams();
    if (query) params.set('query', query);
    params.set('page', String(page));

    const res = await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/blocks?${params}`, {
        method: 'GET',
        headers: {
            Accept: 'application/json',
            'Content-Type': 'application/json',
            ...(token && { Authorization: `Bearer ${token}` })
        }
    })
        .then(async (res) => {
            if (!res.ok) throw await res.json();
            return res.json();
        })
        .catch((err) => {
            error = err;
            console.error('Failed to search blocks:', err);
            return null;
        });

    if (error) throw error;

    // Get total count
    const countRes = await fetch(
        `${YOULAB_API_BASE_URL}/users/${userId}/blocks/count?${query ? `query=${encodeURIComponent(query)}` : ''}`,
        {
            method: 'GET',
            headers: {
                Accept: 'application/json',
                'Content-Type': 'application/json',
                ...(token && { Authorization: `Bearer ${token}` })
            }
        }
    ).then(r => r.json()).catch(() => ({ total: 0 }));

    return {
        items: (res ?? []).map(transformBlock),
        total: countRes.total
    };
}
```

#### 2. Update Store Types
**File**: `OpenWebUI/open-webui/src/lib/stores/memory.ts`
**Changes**: Update MemoryBlock interface

```typescript
export interface MemoryBlock {
    label: string;
    title: string;
    updatedAt: number;  // nanosecond timestamp
    pendingDiffs: number;
    contentPreview: string | null;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors
- [ ] No lint errors in modified files

#### Manual Verification:
- [ ] `searchBlocks()` returns correctly shaped data
- [ ] Pagination works correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 3: Create Blocks.svelte Component

### Overview
Create a new `Blocks.svelte` component that mirrors `Notes.svelte` exactly, adapted for memory blocks with pending diff badges.

### Changes Required:

#### 1. Create BlockMenu Component
**File**: `OpenWebUI/open-webui/src/lib/components/you/Blocks/BlockMenu.svelte`
**Changes**: Create menu component based on NoteMenu (without Delete since blocks are system-managed)

```svelte
<script lang="ts">
    import { DropdownMenu } from 'bits-ui';
    import { getContext } from 'svelte';
    import { fade } from 'svelte/transition';
    import { flyAndScale } from '$lib/utils/transitions';

    import Download from '$lib/components/icons/Download.svelte';
    import Share from '$lib/components/icons/Share.svelte';
    import Link from '$lib/components/icons/Link.svelte';
    import DocumentDuplicate from '$lib/components/icons/DocumentDuplicate.svelte';

    const i18n = getContext('i18n');

    export let show = false;
    export let className = 'max-w-[180px]';

    export let onDownload = (type: string) => {};
    export let onCopyLink: (() => void) | null = null;
    export let onCopyToClipboard: (() => void) | null = null;
    export let onChange = (state: boolean) => {};
</script>

<DropdownMenu.Root
    bind:open={show}
    onOpenChange={(state) => {
        onChange(state);
    }}
>
    <DropdownMenu.Trigger>
        <slot />
    </DropdownMenu.Trigger>

    <slot name="content">
        <DropdownMenu.Content
            class="w-full {className} text-sm rounded-2xl px-1 py-1 border border-gray-100 dark:border-gray-800 z-50 bg-white dark:bg-gray-850 dark:text-white shadow-lg"
            sideOffset={6}
            side="bottom"
            align="end"
            transition={(e) => fade(e, { duration: 100 })}
        >
            <DropdownMenu.Sub>
                <DropdownMenu.SubTrigger
                    class="flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
                >
                    <Download strokeWidth="2" />
                    <div class="flex items-center">{$i18n.t('Download')}</div>
                </DropdownMenu.SubTrigger>
                <DropdownMenu.SubContent
                    class="w-full rounded-xl p-1 z-50 bg-white dark:bg-gray-850 dark:text-white shadow-lg"
                    transition={flyAndScale}
                    sideOffset={8}
                    align="end"
                >
                    <DropdownMenu.Item
                        class="flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
                        on:click={() => onDownload('txt')}
                    >
                        <div class="flex items-center line-clamp-1">{$i18n.t('Plain text (.txt)')}</div>
                    </DropdownMenu.Item>

                    <DropdownMenu.Item
                        class="flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
                        on:click={() => onDownload('md')}
                    >
                        <div class="flex items-center line-clamp-1">{$i18n.t('Markdown (.md)')}</div>
                    </DropdownMenu.Item>
                </DropdownMenu.SubContent>
            </DropdownMenu.Sub>

            {#if onCopyLink || onCopyToClipboard}
                <DropdownMenu.Sub>
                    <DropdownMenu.SubTrigger
                        class="flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
                    >
                        <Share strokeWidth="2" />
                        <div class="flex items-center">{$i18n.t('Share')}</div>
                    </DropdownMenu.SubTrigger>
                    <DropdownMenu.SubContent
                        class="w-full rounded-xl p-1 z-50 bg-white dark:bg-gray-850 dark:text-white shadow-lg"
                        transition={flyAndScale}
                        sideOffset={8}
                        align="end"
                    >
                        {#if onCopyLink}
                            <DropdownMenu.Item
                                class="flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
                                on:click={() => onCopyLink()}
                            >
                                <Link />
                                <div class="flex items-center">{$i18n.t('Copy link')}</div>
                            </DropdownMenu.Item>
                        {/if}

                        {#if onCopyToClipboard}
                            <DropdownMenu.Item
                                class="flex gap-2 items-center px-3 py-1.5 text-sm cursor-pointer hover:bg-gray-50 dark:hover:bg-gray-800 rounded-xl"
                                on:click={() => onCopyToClipboard()}
                            >
                                <DocumentDuplicate strokeWidth="2" />
                                <div class="flex items-center">{$i18n.t('Copy to clipboard')}</div>
                            </DropdownMenu.Item>
                        {/if}
                    </DropdownMenu.SubContent>
                </DropdownMenu.Sub>
            {/if}
        </DropdownMenu.Content>
    </slot>
</DropdownMenu.Root>
```

#### 2. Create Blocks.svelte Component
**File**: `OpenWebUI/open-webui/src/lib/components/you/Blocks.svelte`
**Changes**: Create component mirroring Notes.svelte structure

This is a large component (~500 lines). Key differences from Notes.svelte:
- Uses `searchBlocks` API instead of `searchNotes`
- Links to `/you/blocks/${label}` instead of `/notes/${id}`
- Adds pending diff badge to list and grid items
- Removes "New Block" button (blocks are system-created)
- Removes filter dropdowns (no sharing concept for blocks)
- Removes delete functionality
- Download handler fetches block content via `getBlock` API

The component will:
1. Import `searchBlocks`, `getBlock` from memory API
2. Import `Badge` from common components for diff badges
3. Import `BlockMenu` for context menu
4. Use same time-grouping logic (`getTimeRange`)
5. Use same list/grid display toggle
6. Add badge next to title when `pendingDiffs > 0`

See full implementation in the code below (truncated for plan readability).

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors
- [ ] No lint errors in new files
- [ ] Svelte component compiles

#### Manual Verification:
- [ ] Blocks.svelte renders identically to Notes.svelte layout
- [ ] Search filters blocks by title/label
- [ ] List/Grid toggle works
- [ ] Time grouping displays correctly
- [ ] Pending diff badges appear on blocks with diffs
- [ ] Context menu shows Download and Share options
- [ ] Clicking block navigates to `/you/blocks/${label}`

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 4: Update /you Route

### Overview
Replace the current `/you/+page.svelte` with the new Notes-style layout using `Blocks.svelte`.

### Changes Required:

#### 1. Simplify +page.svelte
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`
**Changes**: Remove tabs, use Blocks component (mirror notes/+page.svelte structure)

```svelte
<script lang="ts">
    import { getContext, onMount } from 'svelte';

    const i18n = getContext('i18n');

    import { mobile, showArchivedChats, showSidebar, user } from '$lib/stores';

    import UserMenu from '$lib/components/layout/Sidebar/UserMenu.svelte';
    import Blocks from '$lib/components/you/Blocks.svelte';
    import Tooltip from '$lib/components/common/Tooltip.svelte';
    import Sidebar from '$lib/components/icons/Sidebar.svelte';
    import { WEBUI_API_BASE_URL } from '$lib/constants';

    let loaded = false;

    onMount(async () => {
        loaded = true;
    });
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
                    <div class="">
                        <div
                            class="flex gap-1 scrollbar-none overflow-x-auto w-fit text-center text-sm font-medium bg-transparent py-1 touch-auto pointer-events-auto"
                        >
                            <a class="min-w-fit transition" href="/you">
                                {$i18n.t('You')}
                            </a>
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

        <div class="flex-1 max-h-full overflow-y-auto @container">
            <Blocks />
        </div>
    </div>
{/if}
```

#### 2. Remove Unused Components
**Files to potentially remove/deprecate**:
- `BlockCard.svelte` - replaced by inline rendering in Blocks.svelte
- `AgentsTab.svelte` - Agents tab removed

Note: Keep these files for now but mark as deprecated with comments. They may be useful for the block detail page.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors
- [ ] No lint errors
- [ ] Route loads without errors

#### Manual Verification:
- [ ] `/you` page renders with Notes-style layout
- [ ] Header shows "You" title
- [ ] Search, list/grid toggle work
- [ ] Blocks display with time grouping
- [ ] Pending diff badges visible
- [ ] Navigation to block detail works

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to the next phase.

---

## Phase 5: Final Testing and Cleanup

### Overview
Comprehensive testing and cleanup of deprecated code.

### Changes Required:

#### 1. Add/Update Tests
**File**: `tests/test_server/test_blocks_api.py`
**Changes**: Add tests for new endpoint fields and search

```python
def test_list_blocks_returns_extended_fields(client, user_id):
    """Test that list_blocks returns updated_at, title, content_preview."""
    response = client.get(f"/users/{user_id}/blocks")
    assert response.status_code == 200
    blocks = response.json()
    if blocks:
        block = blocks[0]
        assert "label" in block
        assert "title" in block
        assert "updated_at" in block
        assert "pending_diffs" in block
        # content_preview is optional
        assert isinstance(block["updated_at"], int)

def test_list_blocks_search(client, user_id):
    """Test that search query filters blocks."""
    # Create blocks with known labels first...
    response = client.get(f"/users/{user_id}/blocks?query=student")
    assert response.status_code == 200
    # Verify filtering works

def test_block_count_endpoint(client, user_id):
    """Test the count endpoint."""
    response = client.get(f"/users/{user_id}/blocks/count")
    assert response.status_code == 200
    assert "total" in response.json()
```

#### 2. Remove Mock Data
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`
**Changes**: The `MOCK_BLOCKS` constant is no longer needed with real API integration.

#### 3. Update Document Title
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+layout.svelte` (create if needed)
**Changes**: Set page title to "You • {WEBUI_NAME}"

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `make test-agent`
- [ ] Lint passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Full verification: `make verify-agent`

#### Manual Verification:
- [ ] Side-by-side comparison: `/you` looks like `/notes`
- [ ] Search functionality works
- [ ] List/Grid toggle preserves preference in localStorage
- [ ] Time grouping is accurate
- [ ] Pending diff badges appear correctly
- [ ] Download functionality works (txt, md)
- [ ] Copy link functionality works
- [ ] Block detail navigation works
- [ ] No console errors
- [ ] Mobile responsive layout works

---

## Testing Strategy

### Unit Tests:
- `test_server/test_blocks_api.py`: New response fields, search, pagination, count endpoint
- Frontend: Svelte component rendering (if test framework available)

### Integration Tests:
- API returns correct data shape
- Frontend correctly displays data from API
- Search filters work end-to-end

### Manual Testing Steps:
1. Navigate to `/you` - should show Notes-style layout
2. Verify blocks display with titles and timestamps
3. Search for a block by partial label - results should filter
4. Toggle List/Grid view - both should work
5. Verify time grouping (Today, Yesterday, etc.)
6. Check a block with pending diffs has badge visible
7. Click context menu - Download submenu should work
8. Click block - should navigate to `/you/blocks/{label}`
9. Resize browser - verify responsive layout
10. Compare side-by-side with `/notes` page

## Performance Considerations

- The list endpoint now fetches metadata for each block, which reads files from disk
- For users with many blocks (>50), pagination will be important
- Consider caching block metadata in memory if performance becomes an issue
- Time-based grouping is done client-side, which is fine for <100 items

## Migration Notes

- No database migrations needed (git-based storage)
- Existing block data is preserved
- Frontend localStorage keys for display preferences: `blockDisplayOption`
- Old `BlockCard.svelte` can be deprecated but kept for backward compatibility

## References

- Notes component: `OpenWebUI/open-webui/src/lib/components/notes/Notes.svelte`
- Notes API: `OpenWebUI/open-webui/src/lib/apis/notes/index.ts`
- Blocks API: `src/youlab_server/server/blocks.py`
- Memory API client: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`
- Time range utility: `OpenWebUI/open-webui/src/lib/utils/index.ts` (`getTimeRange`)
