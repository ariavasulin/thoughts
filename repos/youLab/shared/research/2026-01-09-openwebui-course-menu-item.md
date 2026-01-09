---
date: 2026-01-09T21:59:18+07:00
researcher: ariasulin
git_commit: 47ca115279c37e41f85e0dd3d7765dc2a1f7156d
branch: main
repository: YouLab
topic: "Adding a Course menu item to OpenWebUI sidebar"
tags: [research, codebase, openwebui, sidebar, menu, svelte]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: Adding a Course Menu Item to OpenWebUI Sidebar

**Date**: 2026-01-09T21:59:18+07:00
**Researcher**: ariasulin
**Git Commit**: 47ca115279c37e41f85e0dd3d7765dc2a1f7156d
**Branch**: main
**Repository**: YouLab

## Research Question

What would it take to add a new menu bar item between "Notes" and "Workspace" called "Course" in the OpenWebUI sidebar?

## Summary

Adding a "Course" menu item requires changes across 4-5 files:

1. **Frontend sidebar** - Add menu item blocks in 2 places (collapsed + expanded views)
2. **Icon component** - Create or choose an icon
3. **Translations** - Add "Course" to all locale files
4. **Route** - Create the `/course` route pages
5. **Backend config** - Add `ENABLE_COURSE` feature flag (optional but recommended)

The implementation follows an established pattern used by Notes and Workspace items.

## Detailed Findings

### Primary File: Sidebar.svelte

**Location**: `/Users/ariasulin/Git/YouLab/OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`

The sidebar component contains menu items in TWO places that must both be updated:

#### 1. Collapsed Sidebar (Icon-only view) - Lines 728-787

The collapsed sidebar shows only icons with tooltips. Notes is at lines 728-750, Workspace at 752-787.

**Insert new "Course" item between line 750 and 752:**

```svelte
{#if ($config?.features?.enable_course ?? false) && ($user?.role === 'admin' || ($user?.permissions?.features?.course ?? true))}
    <div class="">
        <Tooltip content={$i18n.t('Course')} placement="right">
            <a
                class=" cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group"
                href="/course"
                on:click={async (e) => {
                    e.stopImmediatePropagation();
                    e.preventDefault();
                    goto('/course');
                    itemClickHandler();
                }}
                draggable="false"
                aria-label={$i18n.t('Course')}
            >
                <div class=" self-center flex items-center justify-center size-9">
                    <CourseIcon className="size-4.5" />
                </div>
            </a>
        </Tooltip>
    </div>
{/if}
```

#### 2. Expanded Sidebar (Icon + Text view) - Lines 963-1016

The expanded sidebar shows icons with text labels. Notes is at lines 963-982, Workspace at 984-1016.

**Insert new "Course" item between line 982 and 984:**

```svelte
{#if ($config?.features?.enable_course ?? false) && ($user?.role === 'admin' || ($user?.permissions?.features?.course ?? true))}
    <div class="px-[0.4375rem] flex justify-center text-gray-800 dark:text-gray-200">
        <a
            id="sidebar-course-button"
            class="grow flex items-center space-x-3 rounded-2xl px-2.5 py-2 hover:bg-gray-100 dark:hover:bg-gray-900 transition"
            href="/course"
            on:click={itemClickHandler}
            draggable="false"
            aria-label={$i18n.t('Course')}
        >
            <div class="self-center">
                <CourseIcon className="size-4.5" strokeWidth="2" />
            </div>
            <div class="flex self-center translate-y-[0.5px]">
                <div class=" self-center text-sm font-primary">{$i18n.t('Course')}</div>
            </div>
        </a>
    </div>
{/if}
```

### Icon Component

**Location**: `/Users/ariasulin/Git/YouLab/OpenWebUI/open-webui/src/lib/components/icons/`

Create a new icon component (e.g., `Course.svelte`) following the pattern:

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
    aria-hidden="true"
>
    <!-- Use a Heroicons path, e.g., academic-cap -->
    <path
        stroke-linecap="round"
        stroke-linejoin="round"
        d="M4.26 10.147a60.438 60.438 0 0 0-.491 6.347A48.62 48.62 0 0 1 12 20.904a48.62 48.62 0 0 1 8.232-4.41 60.46 60.46 0 0 0-.491-6.347m-15.482 0a50.636 50.636 0 0 0-2.658-.813A59.906 59.906 0 0 1 12 3.493a59.903 59.903 0 0 1 10.399 5.84c-.896.248-1.783.52-2.658.814m-15.482 0A50.717 50.717 0 0 1 12 13.489a50.702 50.702 0 0 1 7.74-3.342M6.75 15a.75.75 0 1 0 0-1.5.75.75 0 0 0 0 1.5Zm0 0v-3.675A55.378 55.378 0 0 1 12 8.443m-7.007 11.55A5.981 5.981 0 0 0 6.75 15.75v-1.5"
    />
</svg>
```

Add import to Sidebar.svelte:
```svelte
import CourseIcon from '$lib/components/icons/Course.svelte';
```

### Translations

**Location**: `/Users/ariasulin/Git/YouLab/OpenWebUI/open-webui/src/lib/i18n/locales/*/translation.json`

Add `"Course": ""` (or translated value) to each locale file. The English entry at `en-US/translation.json` line ~1167 would be:

```json
"Course": "",
```

### Route Structure

**Location**: `/Users/ariasulin/Git/YouLab/OpenWebUI/open-webui/src/routes/(app)/course/`

Create route files following the Notes pattern:

```
src/routes/(app)/course/
├── +layout.svelte    # Permission check + page title
├── +page.svelte      # Main course page
└── [id]/
    └── +page.svelte  # Individual course view (if needed)
```

**`+layout.svelte`** example (based on notes pattern):

```svelte
<script lang="ts">
    import { onMount, getContext } from 'svelte';
    import { WEBUI_NAME, config, user } from '$lib/stores';
    import { goto } from '$app/navigation';

    const i18n = getContext('i18n');
    let loaded = false;

    onMount(async () => {
        if (
            !(
                ($config?.features?.enable_course ?? false) &&
                ($user?.role === 'admin' || ($user?.permissions?.features?.course ?? true))
            )
        ) {
            goto('/');
        }
        loaded = true;
    });
</script>

<svelte:head>
    <title>{$i18n.t('Course')} - {$WEBUI_NAME}</title>
</svelte:head>

{#if loaded}
    <slot />
{/if}
```

### Backend Configuration (Optional but Recommended)

If you want admin-controlled visibility:

#### 1. Config Definition

**File**: `backend/open_webui/config.py` (~line 1550)

```python
ENABLE_COURSE = PersistentConfig(
    "ENABLE_COURSE",
    "course.enable",
    os.environ.get("ENABLE_COURSE", "True").lower() == "true",
)
```

#### 2. Main App State

**File**: `backend/open_webui/main.py`

- Import at line ~362: Add `ENABLE_COURSE` to imports
- Set app state at line ~778: `app.state.config.ENABLE_COURSE = ENABLE_COURSE`
- Add to features dict at line ~1906: `"enable_course": app.state.config.ENABLE_COURSE,`

#### 3. Auth Router

**File**: `backend/open_webui/routers/auths.py`

- Add to AdminConfig class (~line 966): `ENABLE_COURSE: bool`
- Add to GET response (~line 944): `"ENABLE_COURSE": request.app.state.config.ENABLE_COURSE,`
- Add to POST handler (~line 991): `request.app.state.config.ENABLE_COURSE = form_data.ENABLE_COURSE`

## Code References

| File | Line | Description |
|------|------|-------------|
| `src/lib/components/layout/Sidebar.svelte` | 728-750 | Notes collapsed menu item |
| `src/lib/components/layout/Sidebar.svelte` | 752-787 | Workspace collapsed menu item |
| `src/lib/components/layout/Sidebar.svelte` | 963-982 | Notes expanded menu item |
| `src/lib/components/layout/Sidebar.svelte` | 984-1016 | Workspace expanded menu item |
| `src/routes/(app)/notes/+layout.svelte` | 1-33 | Notes layout pattern |
| `src/lib/components/icons/Note.svelte` | 1-20 | Icon component pattern |
| `src/lib/i18n/locales/en-US/translation.json` | 1167 | Translation entry location |
| `backend/open_webui/config.py` | 1550-1553 | ENABLE_NOTES config pattern |
| `backend/open_webui/main.py` | 1906 | Feature flag exposure |

## Architecture Documentation

### Menu Item Pattern

Each sidebar menu item consists of:

1. **Permission check**: `{#if condition}` using `$config?.features?.enable_*` and `$user?.permissions`
2. **Tooltip wrapper**: `<Tooltip content={$i18n.t('Label')} placement="right">`
3. **Link element**: `<a href="/route">` with click handler
4. **Icon container**: `<div class="size-9">` containing the icon component
5. **Text label** (expanded only): `<div class="text-sm font-primary">`

### Styling Classes

- **Collapsed icon container**: `cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group`
- **Expanded container**: `px-[0.4375rem] flex justify-center text-gray-800 dark:text-gray-200`
- **Expanded link**: `grow flex items-center space-x-3 rounded-2xl px-2.5 py-2 hover:bg-gray-100 dark:hover:bg-gray-900 transition`
- **Icon size**: `size-4.5` (collapsed: strokeWidth 1.5, expanded: strokeWidth 2)

### Feature Flag Flow

```
Environment Variable (ENABLE_COURSE)
    ↓
backend/config.py (PersistentConfig)
    ↓
backend/main.py (app.state.config)
    ↓
API /api/config (features dict)
    ↓
Frontend $config store
    ↓
Sidebar.svelte permission check
```

## Minimum Changes Checklist

For a basic implementation without backend feature flags:

- [ ] Create `src/lib/components/icons/Course.svelte`
- [ ] Edit `src/lib/components/layout/Sidebar.svelte`:
  - Add import for Course icon
  - Add collapsed menu item after line 750
  - Add expanded menu item after line 982
- [ ] Add "Course" to `src/lib/i18n/locales/en-US/translation.json`
- [ ] Create `src/routes/(app)/course/+layout.svelte`
- [ ] Create `src/routes/(app)/course/+page.svelte`

For full feature-flag support, also update the backend files listed above.

## Open Questions

1. Should Course be visible to all users or permission-gated?
2. What icon best represents "Course"? (academic-cap, book-open, presentation-chart-bar)
3. Does Course need sub-routes like `/course/[id]`?
4. Should there be a backend API for course management?
