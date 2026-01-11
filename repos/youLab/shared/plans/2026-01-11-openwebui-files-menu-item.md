# OpenWebUI "Files" Menu Item Implementation Plan

## Overview

Add a new "Files" menu item to the OpenWebUI sidebar, positioned between "Notes" and "Workspace". This item links to the existing Knowledge page (`/workspace/knowledge`) but with the user-friendly label "Files".

## Current State Analysis

The OpenWebUI sidebar is implemented in `Sidebar.svelte` and has menu items defined in **two places**:
1. **Collapsed view** (lines 728-787): Icon-only buttons with tooltips
2. **Expanded view** (lines 963-1016): Icons with text labels

The Knowledge page already exists at `/workspace/knowledge` and is gated by the `$user?.permissions?.workspace?.knowledge` permission.

### Key Discoveries:
- Notes menu item: lines 728-750 (collapsed), 963-982 (expanded)
- Workspace menu item: lines 752-787 (collapsed), 984-1016 (expanded)
- `Document.svelte` icon exists and is appropriate for "Files"
- Workspace permissions already include `knowledge` check
- No new backend changes required - reusing existing permission system

## Desired End State

After implementation:
- A "Files" menu item appears between "Notes" and "Workspace" in the sidebar
- Clicking "Files" navigates to `/workspace/knowledge`
- The item uses the Document icon for visual consistency with file/document semantics
- The item respects existing `workspace.knowledge` permissions
- Changes persist across container restarts via custom Docker image build

### Verification:
- Menu item visible to users with `workspace.knowledge` permission
- Menu item hidden from users without permission
- Click navigation works in both collapsed and expanded sidebar states
- Styling matches adjacent Notes and Workspace items

## What We're NOT Doing

- Creating a new route (using existing `/workspace/knowledge`)
- Adding new backend permissions or feature flags
- Changing the Knowledge page itself
- Adding translations for languages other than English (can be done later)

## Implementation Approach

Since OpenWebUI runs in a Docker container, we'll:
1. Modify the local OpenWebUI source files
2. Rebuild the Docker image
3. Restart the container with the new image

## Phase 1: Modify Sidebar Component

### Overview
Add the "Files" menu item to both collapsed and expanded sidebar views.

### Changes Required:

#### 1. Add Import for Document Icon

**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
**Location**: Near other icon imports (around line 67)
**Changes**: Add Document icon import

```svelte
import Document from '../icons/Document.svelte';
```

#### 2. Add Collapsed Menu Item (Icon Only)

**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
**Location**: After line 750 (after Notes item, before Workspace item)
**Changes**: Insert new Files menu item

```svelte
{#if $user?.role === 'admin' || $user?.permissions?.workspace?.knowledge}
	<div class="">
		<Tooltip content={$i18n.t('Files')} placement="right">
			<a
				class=" cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group"
				href="/workspace/knowledge"
				on:click={async (e) => {
					e.stopImmediatePropagation();
					e.preventDefault();

					goto('/workspace/knowledge');
					itemClickHandler();
				}}
				draggable="false"
				aria-label={$i18n.t('Files')}
			>
				<div class=" self-center flex items-center justify-center size-9">
					<Document className="size-4.5" />
				</div>
			</a>
		</Tooltip>
	</div>
{/if}
```

#### 3. Add Expanded Menu Item (Icon + Text)

**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
**Location**: After line 982 (after Notes item, before Workspace item)
**Changes**: Insert new Files menu item

```svelte
{#if $user?.role === 'admin' || $user?.permissions?.workspace?.knowledge}
	<div class="px-[0.4375rem] flex justify-center text-gray-800 dark:text-gray-200">
		<a
			id="sidebar-files-button"
			class="grow flex items-center space-x-3 rounded-2xl px-2.5 py-2 hover:bg-gray-100 dark:hover:bg-gray-900 transition"
			href="/workspace/knowledge"
			on:click={itemClickHandler}
			draggable="false"
			aria-label={$i18n.t('Files')}
		>
			<div class="self-center">
				<Document className="size-4.5" strokeWidth="2" />
			</div>

			<div class="flex self-center translate-y-[0.5px]">
				<div class=" self-center text-sm font-primary">{$i18n.t('Files')}</div>
			</div>
		</a>
	</div>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] Svelte syntax is valid (no compilation errors during Docker build)
- [ ] Docker image builds successfully: `docker compose build open-webui`

#### Manual Verification:
- [ ] "Files" menu item appears between "Notes" and "Workspace" in sidebar
- [ ] Icon displays correctly in collapsed sidebar view
- [ ] Text label "Files" displays in expanded sidebar view
- [ ] Clicking navigates to Knowledge page
- [ ] Styling matches Notes and Workspace items

---

## Phase 2: Add Translation Entry

### Overview
Add the "Files" translation key to the English locale file.

### Changes Required:

#### 1. Add Translation Key

**File**: `OpenWebUI/open-webui/src/lib/i18n/locales/en-US/translation.json`
**Changes**: Add "Files" entry (alphabetically near "File" entries)

Find the alphabetical position (likely near line ~686 after "File*" entries) and add:

```json
"Files": "",
```

Note: Empty string value means the key itself ("Files") is used as the display text.

### Success Criteria:

#### Automated Verification:
- [ ] JSON syntax is valid
- [ ] Docker image builds successfully

#### Manual Verification:
- [ ] Menu item displays "Files" text in expanded sidebar
- [ ] Tooltip shows "Files" in collapsed sidebar

---

## Phase 3: Rebuild Docker Image and Deploy

### Overview
Rebuild the OpenWebUI Docker image with the changes and restart the container.

### Commands:

```bash
# Navigate to OpenWebUI directory
cd OpenWebUI/open-webui

# Rebuild the Docker image
docker compose build open-webui

# Restart with new image
docker compose up -d open-webui
```

### Success Criteria:

#### Automated Verification:
- [ ] `docker compose build open-webui` completes successfully
- [ ] `docker compose up -d open-webui` starts container without errors
- [ ] Container health check passes

#### Manual Verification:
- [ ] OpenWebUI accessible at expected URL
- [ ] "Files" menu item visible and functional
- [ ] Navigation to Knowledge page works correctly

---

## Testing Strategy

### Manual Testing Steps:
1. Log in as admin user
2. Verify "Files" appears in collapsed sidebar (icon only with tooltip)
3. Expand sidebar - verify "Files" appears with text label
4. Click "Files" - verify navigation to Knowledge page
5. Log in as user with `workspace.knowledge` permission - verify visibility
6. Log in as user without permission - verify "Files" is hidden

### Edge Cases:
- Sidebar collapsed state navigation
- Sidebar expanded state navigation
- Permission-based visibility
- Mobile/responsive behavior (if applicable)

## Performance Considerations

None - this is a simple DOM addition that reuses existing infrastructure.

## References

- Research document: `thoughts/shared/research/2026-01-09-openwebui-course-menu-item.md`
- Sidebar component: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
- Document icon: `OpenWebUI/open-webui/src/lib/components/icons/Document.svelte`
- Knowledge page: `OpenWebUI/open-webui/src/routes/(app)/workspace/knowledge/+page.svelte`
