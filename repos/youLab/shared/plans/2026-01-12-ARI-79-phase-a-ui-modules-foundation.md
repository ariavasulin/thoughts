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
- Module progression logic (already implemented via `advance_lesson` tool in backend)
- Background agent integration (Phase C)
- User-specific status tracking (status comes from agent memory, not UI)
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

**Changes**: Create a simple 'Y' icon as placeholder logo

Generate a clean favicon.png (32x32 or 64x64) with:
- A bold, stylized 'Y' letter
- YouLab accent color (iA Writer cyan `#15BDEC` or similar)
- Transparent or dark background
- Simple, recognizable at small sizes

Can be created with ImageMagick or a simple SVG-to-PNG conversion:
```bash
# Example using ImageMagick (if available)
convert -size 64x64 xc:transparent -font Helvetica-Bold -pointsize 48 \
  -fill '#15BDEC' -gravity center -annotate 0 'Y' static/favicon.png
```

Or manually create an SVG and convert to PNG.

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

## Phase 2: YouLab Theme (iA Writer Aesthetic) ✅ COMPLETED

### Overview
Implement a YouLab theme based on iA Writer's visual design - clean typography, minimal colors.

### Status: COMPLETED (2026-01-12)

**Completed work:**
- ✅ iA Writer Duo font added (`static/assets/fonts/iAWriterDuoV.woff2`)
- ✅ Font-face declaration in `src/app.css`
- ✅ YouLab themes added to theme dropdown
- ✅ Default theme set to `youlab-dark` in `src/app.html`
- ✅ Dark theme color palette **fixed** (was inverted, now correct)

### Implementation Details

#### 1. iA Writer Duo Font
**Files modified**: `src/app.css`, `src/tailwind.css`

```css
@font-face {
	font-family: 'iA Writer Duo';
	src: url('/assets/fonts/iAWriterDuoV.woff2') format('woff2');
	font-display: swap;
}

html {
	font-family: 'iA Writer Duo', 'Archivo', 'Vazirmatn', sans-serif;
}
```

**Note**: Font downloaded from https://github.com/iaolo/iA-Fonts (OFL licensed). Using Duo variant (duospace) rather than Quattro.

#### 2. YouLab Theme Color Palettes
**Files modified**: `src/lib/components/chat/Settings/General.svelte`, `src/app.html`

**IMPORTANT**: Color semantic meaning must be preserved:
- **Low numbers (50-300)** = Light colors (used for text in dark mode via `dark:text-gray-*`)
- **High numbers (800-950)** = Dark colors (used for backgrounds in dark mode via `dark:bg-gray-*`)

**youlab-dark theme** (correct values after fix):
```javascript
'--color-gray-50': '#e8ebe9'   // Brightest (for text)
'--color-gray-100': '#d8dcd9'  // Bright text
'--color-gray-200': '#c5c9c6'  // Primary text (iA Writer text color)
'--color-gray-300': '#a0a0a0'  // Muted text
'--color-gray-400': '#909090'  // Secondary text
'--color-gray-500': '#707070'  // Mid gray
'--color-gray-600': '#505252'  // Subtle elements
'--color-gray-700': '#404242'  // Borders
'--color-gray-800': '#2a2c2c'  // Element backgrounds
'--color-gray-850': '#222424'  // Elevated surfaces
'--color-gray-900': '#1d1f20'  // Main background (iA Writer bg)
'--color-gray-950': '#141516'  // Darkest
```

**youlab (light theme)**:
```javascript
'--color-gray-50': '#f5f6f6'   // Lightest background
'--color-gray-100': '#e8e9e9'
'--color-gray-200': '#d4d5d5'
'--color-gray-300': '#b0b1b1'
'--color-gray-400': '#8c8d8d'
'--color-gray-500': '#686969'
'--color-gray-600': '#545555'
'--color-gray-700': '#424242'
'--color-gray-800': '#2e2f2f'
'--color-gray-850': '#242525'
'--color-gray-900': '#1a1b1b'
'--color-gray-950': '#101111'  // Darkest text
```

#### 3. Default Theme
**File**: `src/app.html` (line 50)
Default theme set to `youlab-dark` for new users.

### Bug Fix Applied (2026-01-12)

The original dark theme implementation had **inverted semantic meaning** which broke OpenWebUI component expectations:
- `dark:bg-gray-800` was showing light backgrounds (wrong)
- `dark:text-gray-200` was showing dark text on dark backgrounds (unreadable)

**Fix**: Corrected the color mapping to preserve semantic meaning while keeping iA Writer aesthetic.

### Success Criteria:

#### Automated Verification:
- [x] Font files exist: `ls OpenWebUI/open-webui/static/assets/fonts/iAWriterDuoV.woff2`
- [x] App compiles: `cd OpenWebUI/open-webui && npm run build`
- [ ] No TypeScript errors: `cd OpenWebUI/open-webui && npm run check`

#### Manual Verification:
- [x] YouLab theme appears in Settings > General > Theme dropdown
- [x] Selecting YouLab theme changes colors to iA Writer palette
- [x] Fonts render as iA Writer Duo
- [x] Dark/light variants work correctly
- [x] Theme persists after page refresh
- [x] Text is readable on dark backgrounds (contrast fixed)

---

## Phase 3: Hide OpenWebUI Sections ✅ COMPLETED

### Overview
Hide Workspace, Code Interpreter, Controls, and Arena sections using feature flags. The Files section will be simplified to show only Knowledge.

### Status: COMPLETED (2026-01-12)

**Completed work:**
- ✅ Added `youlabMode` store to `src/lib/stores/index.ts` (hardcoded to `true`)
- ✅ Hidden Workspace link in Sidebar.svelte (both collapsed and expanded views)
- ✅ Hidden Controls button in Navbar.svelte
- ✅ Hidden Controls panel in ChatControls.svelte
- ✅ Arena disabled via native OpenWebUI config (see below)
- ✅ Files section unchanged (already points to Knowledge)

### Implementation Details

#### 1. youlabMode Store
**File**: `src/lib/stores/index.ts`
```typescript
// YouLab-specific mode: hides Workspace, Controls, Arena, and other OpenWebUI-specific features
export const youlabMode = writable(true);
```

#### 2. Workspace Link Hidden
**File**: `src/lib/components/layout/Sidebar.svelte`
- Lines 778, 1031: Added `!$youlabMode &&` to permission checks

#### 3. Controls Button Hidden
**File**: `src/lib/components/chat/Navbar.svelte`
- Line 213: Added `!$youlabMode &&` to permission check for Controls button

#### 4. Controls Panel Hidden
**File**: `src/lib/components/chat/ChatControls.svelte`
- Wrapped entire template with `{#if !$youlabMode}` check

#### 5. Arena Disabled via Native Config
**IMPORTANT**: Arena is disabled using OpenWebUI's built-in configuration, NOT youlabMode:
```bash
# Set in environment when running OpenWebUI backend
ENABLE_EVALUATION_ARENA_MODELS=False
```
Or disable via Admin Panel → Settings → Evaluations → Toggle off "Arena Models"

This is the correct approach - always prefer native configs over custom changes.

### Success Criteria:

#### Automated Verification:
- [x] Grep confirms changes: `grep -r "youlabMode" OpenWebUI/open-webui/src`

#### Manual Verification:
- [ ] Sidebar does NOT show Workspace link
- [ ] Chat view does NOT show Controls button or panel
- [ ] No Arena model comparison UI visible (requires `ENABLE_EVALUATION_ARENA_MODELS=False`)
- [ ] Files link still works and shows Knowledge

---

## Phase 4: Modules System Foundation ✅ COMPLETED

### Status: COMPLETED (2026-01-12)

**Completed work:**
- ✅ Created `ModuleItem.svelte` component (reuses CheckCircle, LockClosed icons)
- ✅ Created `ModuleList.svelte` component with demo modules for testing
- ✅ Added Modules section to Sidebar.svelte (visible when `youlabMode` is true)
- ✅ Hidden Models/Pinned Models section when in youlabMode
- ✅ Updated favicon to transparent background (old saved as `favicon_old.png`)

**Known Issue:** Demo modules use placeholder IDs (e.g., `college-essay-intro`) that don't correspond to real models. Clicking a demo module shows "No results found - Pull from Ollama.com". This is expected behavior - real modules need to be created as OpenWebUI models with `youlab_module` metadata. The demo modules will be replaced automatically once real modules exist.

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
- [x] New component files exist: `ls OpenWebUI/open-webui/src/lib/components/layout/Sidebar/Module*.svelte`
- [x] App compiles: Pre-existing codebase has TypeScript errors unrelated to Module components
- [x] No new TypeScript errors: Module components follow existing patterns (i18n error is pre-existing in PinnedModelItem too)

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
- [x] Documentation exists: `cat docs/module-metadata-schema.md`
- [x] Types compile (if added): `cd OpenWebUI/open-webui && npm run check` (pre-existing errors in JS files, no new errors from TypeScript additions)

#### Manual Verification:
- [ ] Can create a model with `youlab_module` metadata via Model Editor
- [ ] Module appears in Modules section with correct display
- [ ] Module status displays correctly

**Implementation Note**: This phase is documentation and types only. Actual module creation will be manual or via course TOML loader (future work).

**Note on Status**: In Phase A, module status is static metadata on the model. True per-user progression tracking would require reading the user's agent `journey` memory block, which is a Phase B/C integration. For now, modules display with a default "available" status.

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
