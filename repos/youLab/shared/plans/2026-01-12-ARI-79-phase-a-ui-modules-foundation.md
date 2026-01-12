# ARI-79 Phase A: UI + Modules Foundation Implementation Plan

## Overview

This plan implements the UI foundation and Modules system for YouLab, transforming OpenWebUI into a branded course delivery platform. Phase A runs in parallel with Phase B (Memory System).

## Current State Analysis

### OpenWebUI Structure (Key Files)
- **Branding**: `src/lib/constants.ts:4` - `APP_NAME = 'Open WebUI'`
- **Stores**: `src/lib/stores/index.ts:10` - `WEBUI_NAME` store
- **Sidebar**: `src/lib/components/layout/Sidebar.svelte` (1464 lines)
- **Theme System**: `src/lib/components/chat/Settings/General.svelte` (theme application)
- **Tailwind Config**: `tailwind.config.js` (CSS variable-based colors)
- **CSS**: `src/app.css` (fonts, custom styles)
- **Model Display**: `PinnedModelList.svelte`, `PinnedModelItem.svelte`
- **Folder Component**: `src/lib/components/common/Folder.svelte` (collapsible sections)

### Key Patterns Discovered
1. **Feature Flags**: Two-tier conditional `$config?.features?.enable_X && ($user?.role === 'admin' || $user?.permissions?.features?.X)`
2. **Section Visibility**: Gated with feature flags in Sidebar.svelte (lines 729-1112)
3. **Theme System**: CSS custom properties set via `applyTheme()` in General.svelte, stored in localStorage
4. **Model Metadata**: `ModelConfig.meta` supports `description`, `suggestion_prompts`, `tags`, `capabilities`, `toolIds`, `knowledge`

## Desired End State

After implementation:
1. App displays "YouLab" branding (logo, title, about page)
2. iA Writer theme applied (colors, fonts)
3. Workspace, Code Interpreter, Controls, Arena sections hidden
4. Files section simplified to "Knowledge" only
5. New "Modules" section in sidebar displaying course modules
6. Module metadata supports module-specific fields (course_id, module_index, description, status)

### How to Verify
- Visual inspection of sidebar shows "YouLab" branding and Modules section
- Feature flags disable OpenWebUI-specific features
- Module cards display correctly in sidebar
- Theme matches iA Writer aesthetic

## What We're NOT Doing

- Backend API changes (Phase A is frontend-only)
- Memory system integration (Phase B)
- User-level module progression tracking
- Module completion logic
- Background agent integration
- OpenWebUI upstream sync compatibility (we accept fork divergence)

## Implementation Approach

We'll modify OpenWebUI frontend directly (not as patches) since this is a fork. Changes are organized into:
1. **Branding** - Logo, title, constants
2. **Theme** - iA Writer colors/fonts
3. **Section Visibility** - Feature flags for hiding
4. **Modules System** - New sidebar section with module display

---

## Phase 1: Branding (YouLab Identity)

### Overview
Rename the application from "Open WebUI" to "YouLab" across all user-facing surfaces.

### Changes Required:

#### 1. Update App Name Constant
**File**: `OpenWebUI/open-webui/src/lib/constants.ts`
**Changes**: Change APP_NAME from 'Open WebUI' to 'YouLab'

```typescript
export const APP_NAME = 'YouLab';
```

#### 2. Update Favicon and Logo
**Files**:
- `OpenWebUI/open-webui/static/favicon.png`
- `OpenWebUI/open-webui/static/logo.png` (if exists)

**Changes**: Replace with YouLab logo (provide logo file)

#### 3. Update HTML Title
**File**: `OpenWebUI/open-webui/src/app.html`
**Changes**: Update `<title>` tag if hardcoded

#### 4. Update About Page
**File**: `OpenWebUI/open-webui/src/lib/components/chat/Settings/About.svelte`
**Changes**: Update description, links, and branding text

### Success Criteria:

#### Automated Verification:
- [ ] App compiles without errors: `cd OpenWebUI/open-webui && npm run build`
- [ ] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] Browser tab shows "YouLab" title
- [ ] Sidebar header displays "YouLab"
- [ ] About page shows YouLab branding
- [ ] Favicon displays YouLab logo

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: YouLab Theme (iA Writer Aesthetic)

### Overview
Implement a YouLab theme based on iA Writer's visual design - clean typography, minimal colors.

### Changes Required:

#### 1. Add iA Writer Fonts
**File**: `OpenWebUI/open-webui/src/app.css`
**Changes**: Add font-face declarations for iA Writer Quattro (OFL licensed)

```css
@font-face {
	font-family: 'iA Writer Quattro';
	src: url('/assets/fonts/iAWriterQuattroV.woff2') format('woff2');
	font-display: swap;
}

/* Update font-primary to use iA Writer Quattro */
.font-primary {
	font-family: 'iA Writer Quattro', 'Archivo', 'Vazirmatn', sans-serif;
}
```

**Note**: Download fonts from https://github.com/iaolo/iA-Fonts and place in `static/assets/fonts/`

#### 2. Add YouLab Theme to Theme List
**File**: `OpenWebUI/open-webui/src/lib/components/chat/Settings/General.svelte`
**Changes**: Add 'youlab' and 'youlab-dark' to themes array, add case in applyTheme()

```javascript
// Around line 17, add to themes array
const themes = ['system', 'light', 'dark', 'oled-dark', 'youlab', 'youlab-dark', 'her'];

// In applyTheme() function, add case for youlab themes
case 'youlab':
    // iA Writer light mode colors
    document.documentElement.style.setProperty('--color-gray-50', '#f5f6f6');
    document.documentElement.style.setProperty('--color-gray-100', '#e8e9e9');
    document.documentElement.style.setProperty('--color-gray-200', '#d4d5d5');
    document.documentElement.style.setProperty('--color-gray-300', '#b0b1b1');
    document.documentElement.style.setProperty('--color-gray-400', '#8c8d8d');
    document.documentElement.style.setProperty('--color-gray-500', '#686969');
    document.documentElement.style.setProperty('--color-gray-600', '#545555');
    document.documentElement.style.setProperty('--color-gray-700', '#424242');
    document.documentElement.style.setProperty('--color-gray-800', '#2e2f2f');
    document.documentElement.style.setProperty('--color-gray-850', '#242525');
    document.documentElement.style.setProperty('--color-gray-900', '#1a1b1b');
    document.documentElement.style.setProperty('--color-gray-950', '#101111');
    document.documentElement.classList.remove('dark');
    metaThemeColor = '#f5f6f6';
    break;

case 'youlab-dark':
    // iA Writer dark mode colors
    document.documentElement.style.setProperty('--color-gray-50', '#222424');
    document.documentElement.style.setProperty('--color-gray-100', '#2a2c2c');
    document.documentElement.style.setProperty('--color-gray-200', '#303232');
    document.documentElement.style.setProperty('--color-gray-300', '#404242');
    document.documentElement.style.setProperty('--color-gray-400', '#505252');
    document.documentElement.style.setProperty('--color-gray-500', '#707070');
    document.documentElement.style.setProperty('--color-gray-600', '#909090');
    document.documentElement.style.setProperty('--color-gray-700', '#a0a0a0');
    document.documentElement.style.setProperty('--color-gray-800', '#c5c9c6');
    document.documentElement.style.setProperty('--color-gray-850', '#d0d4d1');
    document.documentElement.style.setProperty('--color-gray-900', '#e0e4e1');
    document.documentElement.style.setProperty('--color-gray-950', '#f0f4f1');
    document.documentElement.classList.add('dark');
    metaThemeColor = '#1d1f20';
    break;
```

#### 3. Set Default Theme
**File**: `OpenWebUI/open-webui/src/lib/stores/index.ts`
**Changes**: Change default theme from 'system' to 'youlab-dark'

```typescript
export const theme = writable('youlab-dark');
```

### Success Criteria:

#### Automated Verification:
- [ ] Font files exist: `ls OpenWebUI/open-webui/static/assets/fonts/iAWriterQuattroV.woff2`
- [ ] App compiles: `cd OpenWebUI/open-webui && npm run build`
- [ ] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] YouLab theme appears in Settings > General > Theme dropdown
- [ ] Selecting YouLab theme changes colors to iA Writer palette
- [ ] Fonts render as iA Writer Quattro
- [ ] Dark/light variants work correctly
- [ ] Theme persists after page refresh

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Hide OpenWebUI Sections

### Overview
Hide Workspace, Code Interpreter, Controls, and Arena sections using feature flags. The Files section will be simplified to show only Knowledge.

### Changes Required:

#### 1. Add YouLab Mode Feature Flag
**File**: `OpenWebUI/open-webui/src/lib/stores/index.ts`
**Changes**: This is controlled via backend config. For now, we'll hardcode the flag.

**File**: `OpenWebUI/open-webui/src/routes/+layout.svelte`
**Changes**: Add a global `youlabMode` store or constant

```typescript
// Near other config initialization
export const youlabMode = writable(true); // YouLab-specific mode
```

#### 2. Hide Workspace Link in Sidebar
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
**Changes**: Wrap Workspace section (lines 777-812 and 1030-1062) with YouLab mode check

Find the Workspace sections and wrap with:
```svelte
{#if !youlabMode}
    <!-- Workspace section -->
{/if}
```

Or modify the existing permission check to always return false in YouLab mode:
```svelte
{#if !youlabMode && ($user?.role === 'admin' || $user?.permissions?.workspace?.models || ...)}
```

#### 3. Hide Controls Panel
**File**: `OpenWebUI/open-webui/src/lib/components/chat/ChatControls.svelte`
**Changes**: Add early return or conditional render for YouLab mode

```svelte
<script>
    // Add import
    import { youlabMode } from '$lib/stores';
</script>

{#if !$youlabMode}
    <!-- existing controls content -->
{/if}
```

Or simply hide the toggle in the chat navbar that opens controls.

#### 4. Hide Arena/Evaluation Features
**File**: `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte`
**Changes**: Hide arena model selection UI

Search for "arena" references and wrap with YouLab mode checks.

#### 5. Simplify Files Section
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
**Changes**: The Files link already points to `/workspace/knowledge`. No change needed unless we want to rename it.

### Success Criteria:

#### Automated Verification:
- [ ] App compiles: `cd OpenWebUI/open-webui && npm run build`
- [ ] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check`
- [ ] Grep confirms changes: `grep -r "youlabMode" OpenWebUI/open-webui/src`

#### Manual Verification:
- [ ] Sidebar does NOT show Workspace link
- [ ] Chat view does NOT show Controls panel
- [ ] No Arena model comparison UI visible
- [ ] Files link still works and shows Knowledge

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Modules System Foundation

### Overview
Add a "Modules" section to the sidebar that displays course modules. This leverages the existing Folder and PinnedModelItem patterns.

### Changes Required:

#### 1. Create ModuleItem Component
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte` (NEW)
**Changes**: Create component based on PinnedModelItem pattern

```svelte
<script lang="ts">
	import { getContext } from 'svelte';
	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';
	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n = getContext('i18n');

	export let module: {
		id: string;
		name: string;
		description?: string;
		status?: 'locked' | 'available' | 'in_progress' | 'completed';
		icon_url?: string;
	};
	export let onClick = () => {};

	let mouseOver = false;

	$: statusColor = {
		locked: 'text-gray-400',
		available: 'text-blue-500',
		in_progress: 'text-yellow-500',
		completed: 'text-green-500'
	}[module.status ?? 'available'];
</script>

{#if module}
	<div
		class="flex justify-center text-gray-800 dark:text-gray-200 cursor-pointer relative group"
		data-id={module.id}
		on:mouseenter={() => mouseOver = true}
		on:mouseleave={() => mouseOver = false}
	>
		<a
			class="grow flex items-center space-x-2.5 rounded-xl px-2.5 py-[7px] group-hover:bg-gray-100 dark:group-hover:bg-gray-900 transition"
			href="/?model={module.id}"
			on:click={onClick}
			draggable="false"
		>
			<div class="self-center shrink-0">
				{#if module.icon_url}
					<img
						src={module.icon_url}
						class="size-5 rounded-full"
						alt={module.name}
					/>
				{:else}
					<div class="size-5 rounded-full bg-gray-200 dark:bg-gray-700 flex items-center justify-center text-xs">
						{module.name.charAt(0)}
					</div>
				{/if}
			</div>

			<div class="flex flex-col flex-1">
				<div class="self-center text-sm font-primary line-clamp-1">
					{module.name}
				</div>
				{#if module.description}
					<div class="text-xs text-gray-500 line-clamp-1">
						{module.description}
					</div>
				{/if}
			</div>

			<!-- Status indicator -->
			<div class="self-center {statusColor}">
				{#if module.status === 'completed'}
					<svg class="size-4" fill="currentColor" viewBox="0 0 20 20">
						<path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
					</svg>
				{:else if module.status === 'in_progress'}
					<svg class="size-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
					</svg>
				{:else if module.status === 'locked'}
					<svg class="size-4" fill="currentColor" viewBox="0 0 20 20">
						<path fill-rule="evenodd" d="M5 9V7a5 5 0 0110 0v2a2 2 0 012 2v5a2 2 0 01-2 2H5a2 2 0 01-2-2v-5a2 2 0 012-2zm8-2v2H7V7a3 3 0 016 0z" clip-rule="evenodd" />
					</svg>
				{/if}
			</div>
		</a>
	</div>
{/if}
```

#### 2. Create ModuleList Component
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleList.svelte` (NEW)
**Changes**: Create list component based on PinnedModelList pattern

```svelte
<script>
	import { onMount } from 'svelte';
	import { models, chatId, mobile, showSidebar } from '$lib/stores';
	import ModuleItem from './ModuleItem.svelte';

	// Filter models that are YouLab modules
	// Modules are identified by having youlab_module metadata
	$: modules = $models
		.filter(model => model.info?.meta?.youlab_module)
		.map(model => ({
			id: model.id,
			name: model.name,
			description: model.info?.meta?.description,
			status: model.info?.meta?.youlab_module?.status ?? 'available',
			icon_url: model.info?.meta?.profile_image_url,
			module_index: model.info?.meta?.youlab_module?.module_index ?? 0
		}))
		.sort((a, b) => a.module_index - b.module_index);
</script>

<div class="mt-0.5 pb-1.5" id="module-list">
	{#each modules as module (module.id)}
		<ModuleItem
			{module}
			onClick={() => {
				chatId.set('');
				if ($mobile) {
					showSidebar.set(false);
				}
			}}
		/>
	{/each}

	{#if modules.length === 0}
		<div class="px-2.5 py-2 text-sm text-gray-500 dark:text-gray-400">
			No modules available
		</div>
	{/if}
</div>
```

#### 3. Add Modules Section to Sidebar
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
**Changes**: Import ModuleList and add Modules folder section

Add import:
```svelte
import ModuleList from './Sidebar/ModuleList.svelte';
```

Add state variable:
```svelte
let showModules = true;
```

Add Modules section after the Models section (around line 1076):
```svelte
<!-- YouLab Modules Section -->
{#if youlabMode}
	<Folder
		id="sidebar-modules"
		bind:open={showModules}
		className="px-2 mt-0.5"
		name={$i18n.t('Modules')}
		chevron={false}
		dragAndDrop={false}
	>
		<ModuleList />
	</Folder>
{/if}
```

### Success Criteria:

#### Automated Verification:
- [ ] New component files exist: `ls OpenWebUI/open-webui/src/lib/components/layout/Sidebar/Module*.svelte`
- [ ] App compiles: `cd OpenWebUI/open-webui && npm run build`
- [ ] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] Sidebar shows "Modules" collapsible section
- [ ] Models with `youlab_module` metadata appear in Modules section
- [ ] Module items display name, description, status icon
- [ ] Clicking a module opens chat with that model
- [ ] Empty state shows when no modules configured

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 5: Module Metadata Schema

### Overview
Define the metadata schema for YouLab modules. This extends the existing model metadata with module-specific fields.

### Changes Required:

#### 1. Document Module Metadata Schema
**File**: `docs/module-metadata-schema.md` (NEW or update existing)
**Changes**: Document the expected metadata structure

```markdown
# YouLab Module Metadata Schema

Modules are OpenWebUI models with additional `youlab_module` metadata.

## Schema

```typescript
interface ModuleMetadata {
  // Standard OpenWebUI model meta fields
  profile_image_url?: string;
  description?: string;
  suggestion_prompts?: Array<{content: string, title: [string, string]}>;
  capabilities?: object;

  // YouLab-specific fields
  youlab_module?: {
    course_id: string;           // e.g., "college-essay"
    module_index: number;        // Display order (0-based)
    status?: 'locked' | 'available' | 'in_progress' | 'completed';
    welcome_message?: string;    // Agent-speaks-first message
    unlock_criteria?: {          // Future: automatic unlocking
      previous_module?: string;
      min_interactions?: number;
    };
  };
}
```

## Example

Creating a module via OpenWebUI Model Editor or API:

```json
{
  "id": "college-essay-intro",
  "name": "First Impressions",
  "base_model_id": "youlab-letta-pipe",
  "meta": {
    "profile_image_url": "/static/modules/intro.png",
    "description": "Learn what makes a compelling college essay opening",
    "youlab_module": {
      "course_id": "college-essay",
      "module_index": 0,
      "status": "available",
      "welcome_message": "Hi! I'm excited to help you craft the perfect opening for your college essay. What school are you applying to?"
    }
  }
}
```
```

#### 2. Update TypeScript Types (Optional)
**File**: `OpenWebUI/open-webui/src/lib/stores/index.ts` or new types file
**Changes**: Add TypeScript interface for YouLab module metadata

```typescript
export interface YouLabModuleMeta {
  course_id: string;
  module_index: number;
  status?: 'locked' | 'available' | 'in_progress' | 'completed';
  welcome_message?: string;
  unlock_criteria?: {
    previous_module?: string;
    min_interactions?: number;
  };
}

// Extend ModelMeta
export interface ModelMeta {
  // ... existing fields
  youlab_module?: YouLabModuleMeta;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Documentation exists: `cat docs/module-metadata-schema.md`
- [ ] Types compile (if added): `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [ ] Can create a model with `youlab_module` metadata via Model Editor
- [ ] Module appears in Modules section with correct display
- [ ] Module status displays correctly

**Implementation Note**: This phase is documentation and types only. Actual module creation will be manual or via course TOML loader (future work).

---

## Testing Strategy

### Unit Tests
- No unit tests needed for Phase A (pure frontend changes)

### Integration Tests
- Test module filtering logic in ModuleList component

### Manual Testing Steps
1. Start OpenWebUI dev server: `cd OpenWebUI/open-webui && npm run dev`
2. Verify branding changes (logo, title)
3. Verify theme selection works
4. Verify hidden sections (Workspace, Controls, Arena)
5. Create test model with `youlab_module` metadata
6. Verify module appears in Modules section
7. Click module and verify chat opens

## Performance Considerations

- Module list filtering is O(n) on models array - acceptable for typical model counts (<100)
- No additional API calls needed - modules are filtered from existing models store

## Migration Notes

- No data migration needed
- Existing OpenWebUI models without `youlab_module` metadata will not appear in Modules section
- Theme preference will reset for existing users (localStorage key unchanged)

## References

- Linear ticket: ARI-79
- Parent ticket with research: ARI-78 (ID: 719d97dd-ab58-4068-86ec-d8c9405acf9f)
- iA Writer fonts: https://github.com/iaolo/iA-Fonts (OFL licensed)
- Research documents:
  - `thoughts/shared/research/2026-01-12-ARI-78-openwebui-sidebar-components.md`
  - `thoughts/shared/research/2026-01-12-ARI-78-ia-writer-styling.md`

---

*Plan created by Claude | 2026-01-12*
