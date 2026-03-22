---
date: 2026-01-14T16:11:55+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "Workspace Knowledge Page UI Patterns for Profile Tab Implementation"
tags: [research, codebase, openwebui, workspace, knowledge, svelte, ui-patterns]
status: complete
last_updated: 2026-01-14
last_updated_by: ariasulin
---

# Research: Workspace Knowledge Page UI Patterns for Profile Tab Implementation

**Date**: 2026-01-14T16:11:55+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Understand the workspace/knowledge page structure to inform adding a new 'Profile' tab that renders memory blocks using the same UI patterns as Knowledge.

## Summary

The OpenWebUI workspace uses a SvelteKit route-based architecture with:
1. A shared layout (`+layout.svelte`) that renders tab navigation
2. Route-specific pages that import main components from `$lib/components/workspace/`
3. Common patterns including search, view filtering, item grids, and CRUD operations

Adding a Profile tab requires:
1. Adding a route at `/workspace/profile/`
2. Adding a Profile component at `$lib/components/workspace/Profile.svelte`
3. Adding the tab link in the workspace layout
4. Optionally adding permission checks

## Detailed Findings

### 1. Tab Navigation Structure

**File**: `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte`

The workspace layout defines tabs at lines 81-124:

```svelte
<div class="flex gap-1 scrollbar-none overflow-x-auto ...">
  {#if $user?.role === 'admin' || $user?.permissions?.workspace?.models}
    <a class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/models') ? '' : 'text-gray-300...'}"
       href="/workspace/models">{$i18n.t('Models')}</a>
  {/if}

  {#if $user?.role === 'admin' || $user?.permissions?.workspace?.knowledge}
    <a href="/workspace/knowledge">{$i18n.t('Knowledge')}</a>
  {/if}

  <!-- Similar for Prompts, Tools -->
</div>
```

**Key patterns**:
- Each tab is wrapped in permission check (`$user?.role === 'admin' || $user?.permissions?.workspace?.X`)
- Active tab styling uses `$page.url.pathname.includes('/workspace/X')`
- Tab content renders via `<slot />` at line 135

**To add Profile tab**, insert after Tools (around line 123):
```svelte
{#if $user?.role === 'admin' || $user?.permissions?.workspace?.profile}
  <a class="min-w-fit p-1.5 {$page.url.pathname.includes('/workspace/profile')
    ? ''
    : 'text-gray-300 dark:text-gray-600 hover:text-gray-700 dark:hover:text-white'} transition"
    href="/workspace/profile">{$i18n.t('Profile')}</a>
{/if}
```

### 2. Route Structure

**Knowledge route files**:
```
src/routes/(app)/workspace/knowledge/
├── +page.svelte          # Main list view (imports Knowledge.svelte)
├── [id]/+page.svelte     # Detail view (imports KnowledgeBase.svelte)
└── create/+page.svelte   # Create view (imports CreateKnowledgeBase.svelte)
```

**`+page.svelte`** (line 1-5) - Simple wrapper:
```svelte
<script>
  import Knowledge from '$lib/components/workspace/Knowledge.svelte';
</script>

<Knowledge />
```

**For Profile**, create:
```
src/routes/(app)/workspace/profile/
├── +page.svelte          # Imports Profile.svelte
└── [blockId]/+page.svelte  # Optional: detail view for editing a block
```

### 3. Main List Component Pattern (Knowledge.svelte)

**File**: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte` (328 lines)

#### 3.1 State Management (lines 27-41)
```typescript
let loaded = false;
let page = 1;
let query = '';
let viewOption = '';
let items = null;
let total = null;
let allItemsLoaded = false;
let itemsLoading = false;
```

#### 3.2 Reactive Search (lines 43-64)
```typescript
$: if (loaded && query !== undefined && viewOption !== undefined) {
  init();
}

const init = async () => {
  reset();
  await getItemsPage();
};
```

#### 3.3 Header with Title + Count + New Button (lines 127-149)
```svelte
<div class="flex flex-col gap-1 px-1 mt-1.5 mb-3">
  <div class="flex justify-between items-center">
    <div class="flex items-center text-xl font-medium px-0.5 gap-2">
      <div>{$i18n.t('Knowledge')}</div>
      <div class="text-lg text-gray-500">{total}</div>
    </div>

    <div class="flex w-full justify-end gap-1.5">
      <a class="px-2 py-1.5 rounded-xl bg-black text-white..."
         href="/workspace/knowledge/create">
        <Plus className="size-3" strokeWidth="2.5" />
        <div class="hidden md:block md:ml-1 text-xs">{$i18n.t('New Knowledge')}</div>
      </a>
    </div>
  </div>
</div>
```

#### 3.4 Search Box (lines 152-178)
```svelte
<div class="py-2 bg-white dark:bg-gray-900 rounded-3xl border border-gray-100/30...">
  <div class="flex w-full space-x-2 py-0.5 px-3.5 pb-2">
    <div class="flex flex-1">
      <div class="self-center ml-1 mr-3">
        <Search className="size-3.5" />
      </div>
      <input
        class="w-full text-sm py-1 rounded-r-xl outline-hidden bg-transparent"
        bind:value={query}
        placeholder={$i18n.t('Search Knowledge')}
      />
      {#if query}
        <button on:click={() => { query = ''; }}>
          <XMark className="size-3" />
        </button>
      {/if}
    </div>
  </div>
```

#### 3.5 View Filter Selector (lines 180-202)
```svelte
<div class="px-3 flex w-full bg-transparent overflow-x-auto scrollbar-none -mx-1">
  <div class="flex gap-0.5 w-fit text-center text-sm rounded-full...">
    <ViewSelector
      bind:value={viewOption}
      onChange={async (value) => {
        localStorage.workspaceViewOption = value;
        await tick();
      }}
    />
  </div>
</div>
```

**ViewSelector** (`src/lib/components/workspace/common/ViewSelector.svelte`) provides:
- All
- Created by you
- Shared with you

#### 3.6 Item Grid (lines 204-286)
```svelte
{#if items !== null && total !== null}
  {#if (items ?? []).length !== 0}
    <div class="my-2 px-3 grid grid-cols-1 lg:grid-cols-2 gap-2">
      {#each items as item}
        <button
          class="flex space-x-4 cursor-pointer text-left w-full px-3 py-2.5
                 dark:hover:bg-gray-850/50 hover:bg-gray-50 transition rounded-2xl"
          on:click={() => goto(`/workspace/knowledge/${item.id}`)}
        >
          <!-- Item content with Badge, name, timestamp, author -->
        </button>
      {/each}
    </div>

    <!-- Infinite scroll loader -->
    {#if !allItemsLoaded}
      <Loader on:visible={() => loadMoreItems()}>
        <Spinner />
      </Loader>
    {/if}
  {:else}
    <!-- Empty state -->
    <div class="w-full h-full flex flex-col justify-center items-center my-16">
      <div class="text-3xl mb-3">😕</div>
      <div class="text-lg font-medium">{$i18n.t('No knowledge found')}</div>
    </div>
  {/if}
{:else}
  <!-- Loading state -->
  <Spinner className="size-4" />
{/if}
```

#### 3.7 Item Card Content Pattern (lines 208-284)
```svelte
<button class="flex space-x-4 cursor-pointer...">
  <div class="w-full">
    <div class="flex items-center justify-between -my-1 h-8">
      <div class="flex gap-2 items-center">
        <Badge type="success" content={$i18n.t('Collection')} />
        {#if !item?.write_access}
          <Badge type="muted" content={$i18n.t('Read Only')} />
        {/if}
      </div>

      {#if item?.write_access || $user?.role === 'admin'}
        <ItemMenu on:delete={() => { selectedItem = item; showDeleteConfirm = true; }} />
      {/if}
    </div>

    <div class="flex items-center gap-1 justify-between px-1.5">
      <Tooltip content={item?.description ?? item.name}>
        <div class="text-sm font-medium line-clamp-1 capitalize">{item.name}</div>
      </Tooltip>

      <div class="flex items-center gap-2">
        <Tooltip content={dayjs(item.updated_at * 1000).format('LLLL')}>
          <div class="text-xs text-gray-500">
            {$i18n.t('Updated')} {dayjs(item.updated_at * 1000).fromNow()}
          </div>
        </Tooltip>

        <div class="text-xs text-gray-500">
          {$i18n.t('By {{name}}', { name: item?.user?.name })}
        </div>
      </div>
    </div>
  </div>
</button>
```

### 4. Item Menu Component

**File**: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge/ItemMenu.svelte`

Simple dropdown menu with delete option using `bits-ui` DropdownMenu:

```svelte
<Dropdown bind:show>
  <Tooltip content={$i18n.t('More')}>
    <button><EllipsisHorizontal /></button>
  </Tooltip>

  <div slot="content">
    <DropdownMenu.Content>
      <DropdownMenu.Item on:click={() => dispatch('delete')}>
        <GarbageBin />
        <div>{$i18n.t('Delete')}</div>
      </DropdownMenu.Item>
    </DropdownMenu.Content>
  </div>
</Dropdown>
```

### 5. Common Components Used

| Component | Location | Purpose |
|-----------|----------|---------|
| `ViewSelector` | `workspace/common/ViewSelector.svelte` | Filter by All/Created/Shared |
| `Search` | `icons/Search.svelte` | Search icon |
| `Plus` | `icons/Plus.svelte` | Add button icon |
| `XMark` | `icons/XMark.svelte` | Clear/close icon |
| `Badge` | `common/Badge.svelte` | Status badges |
| `Tooltip` | `common/Tooltip.svelte` | Hover tooltips |
| `Spinner` | `common/Spinner.svelte` | Loading indicator |
| `Loader` | `common/Loader.svelte` | Infinite scroll trigger |
| `ConfirmDialog` | `common/ConfirmDialog.svelte` | Delete confirmation |
| `EllipsisHorizontal` | `icons/EllipsisHorizontal.svelte` | Menu trigger icon |
| `Dropdown` | `common/Dropdown.svelte` | Dropdown wrapper |

### 6. Models.svelte Comparison

**File**: `OpenWebUI/open-webui/src/lib/components/workspace/Models.svelte` (689 lines)

Models uses the same patterns with additions:
- **TagSelector** for filtering by tags (line 428-435)
- **Pagination** component for page-based navigation (line 637-639)
- **Import/Export** buttons (lines 340-364)
- **Switch** for enable/disable toggle (line 578-592)

Key difference: Models uses debounced search (lines 73-83):
```typescript
$: if (page !== undefined && query !== undefined && ...) {
  clearTimeout(searchDebounceTimer);
  searchDebounceTimer = setTimeout(() => {
    getModelList();
  }, 300);
}
```

### 7. Permissions Structure

Layout checks permissions at route level (lines 24-40):
```typescript
onMount(async () => {
  if ($user?.role !== 'admin') {
    if ($page.url.pathname.includes('/models') && !$user?.permissions?.workspace?.models) {
      goto('/');
    }
    // Similar for knowledge, prompts, tools
  }
});
```

Permission path: `$user.permissions.workspace.{tab_name}`

## Code References

- Tab navigation: `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte:81-124`
- Knowledge route: `OpenWebUI/open-webui/src/routes/(app)/workspace/knowledge/+page.svelte:1-5`
- Knowledge component: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte:1-328`
- Header pattern: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte:127-149`
- Search pattern: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte:152-178`
- View filter: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte:180-202`
- Item grid: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge.svelte:204-286`
- ItemMenu: `OpenWebUI/open-webui/src/lib/components/workspace/Knowledge/ItemMenu.svelte:1-68`
- ViewSelector: `OpenWebUI/open-webui/src/lib/components/workspace/common/ViewSelector.svelte:1-96`
- Models comparison: `OpenWebUI/open-webui/src/lib/components/workspace/Models.svelte:1-689`

## Architecture Documentation

### File Organization Pattern

```
src/routes/(app)/workspace/
├── +layout.svelte              # Shared navigation, permission checks
├── models/
│   ├── +page.svelte            # <Models /> wrapper
│   ├── create/+page.svelte     # Create form
│   └── edit/+page.svelte       # Edit form (?id= query param)
├── knowledge/
│   ├── +page.svelte            # <Knowledge /> wrapper
│   ├── create/+page.svelte     # <CreateKnowledgeBase />
│   └── [id]/+page.svelte       # <KnowledgeBase /> detail view
├── prompts/
│   └── ...
└── tools/
    └── ...

src/lib/components/workspace/
├── Models.svelte               # Main Models list component
├── Knowledge.svelte            # Main Knowledge list component
├── Models/
│   └── ModelMenu.svelte        # Model-specific dropdown menu
├── Knowledge/
│   ├── ItemMenu.svelte         # Knowledge item dropdown menu
│   ├── CreateKnowledgeBase.svelte
│   └── KnowledgeBase.svelte    # Detail view component
│       └── KnowledgeBase/
│           ├── Files.svelte    # File list subcomponent
│           ├── AddContentMenu.svelte
│           └── AddTextContentModal.svelte
└── common/
    └── ViewSelector.svelte     # Shared view filter dropdown
```

### State Management Pattern

Components use local reactive state with Svelte stores for shared data:
- Local: `loaded`, `items`, `total`, `page`, `query`, `viewOption`
- Shared stores: `$user`, `$knowledge`, `$i18n` (from context)

### API Integration Pattern

```typescript
// Fetch with pagination
const getItemsPage = async () => {
  const res = await searchKnowledgeBases(localStorage.token, query, viewOption, page);
  if (res) {
    total = res.total;
    items = [...items, ...res.items];
  }
};

// CRUD operations via toast feedback
const deleteHandler = async (item) => {
  const res = await deleteKnowledgeById(localStorage.token, item.id).catch((e) => {
    toast.error(`${e}`);
  });
  if (res) {
    toast.success($i18n.t('Knowledge deleted successfully.'));
    init();
  }
};
```

## Implementation Blueprint for Profile Tab

### Required Files to Create

1. **Route**: `src/routes/(app)/workspace/profile/+page.svelte`
```svelte
<script>
  import Profile from '$lib/components/workspace/Profile.svelte';
</script>

<Profile />
```

2. **Component**: `src/lib/components/workspace/Profile.svelte`
   - Follow Knowledge.svelte structure
   - Replace `searchKnowledgeBases` with memory block API
   - Adapt item card to show block name, type, content preview

3. **Item Menu**: `src/lib/components/workspace/Profile/BlockMenu.svelte`
   - Edit, Delete options

4. **Layout update**: `src/routes/(app)/workspace/+layout.svelte`
   - Add Profile tab link around line 123

### Data Shape Considerations

Memory blocks have different shape than knowledge items:
```typescript
// Knowledge item
{ id, name, description, updated_at, user, write_access }

// Memory block (example)
{ name, type, value, label, description, limit, updated_at }
```

Adapt item card rendering accordingly.

## Open Questions

1. Should Profile tab require specific permission (`workspace.profile`) or be available to all users?
2. What memory block operations should be supported (view, edit, delete, create)?
3. Should blocks be editable inline or via modal/separate page?
4. Does the backend need new endpoints for listing/searching user memory blocks?
