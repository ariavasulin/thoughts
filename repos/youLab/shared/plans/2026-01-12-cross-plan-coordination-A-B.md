# Cross-Plan Coordination: Phase A (ARI-79) + Phase B (ARI-80)

## Overview

This document coordinates the parallel implementation of:
- **Phase A (ARI-79)**: UI + Modules Foundation - OpenWebUI branding, theme, section hiding, Modules system
- **Phase B (ARI-80)**: Memory System MVP - Git storage, block CRUD API, "You" section in UI

Both phases modify the OpenWebUI frontend and share integration points that require coordination.

---

## 1. File Ownership

### Phase A Owns (Do Not Modify in B)

| File | Changes |
|------|---------|
| `src/lib/constants.ts` | `APP_NAME = 'YouLab'` |
| `src/lib/stores/index.ts` | `youlabMode` store, default theme |
| `src/lib/components/chat/Settings/General.svelte` | YouLab theme cases |
| `src/lib/components/chat/ChatControls.svelte` | YouLab mode hide |
| `src/lib/components/layout/Sidebar/ModuleItem.svelte` | NEW - A creates |
| `src/lib/components/layout/Sidebar/ModuleList.svelte` | NEW - A creates |
| `static/favicon.png` | YouLab branding |
| `static/assets/fonts/` | iA Writer fonts |
| `src/app.css` | Font-face declarations |
| `src/app.html` | Title tag |
| `src/lib/components/chat/Settings/About.svelte` | YouLab branding |

### Phase B Owns (Do Not Modify in A)

| File | Changes |
|------|---------|
| `src/lib/stores/memory.ts` | NEW - B creates |
| `src/lib/apis/memory/index.ts` | NEW - B creates |
| `src/routes/(app)/you/+page.svelte` | NEW - B creates |
| `src/lib/components/you/BlockCard.svelte` | NEW - B creates |
| `src/lib/components/you/BlockDetailModal.svelte` | NEW - B creates |

### Shared Files (Coordinate Changes)

| File | Phase A Changes | Phase B Changes |
|------|-----------------|-----------------|
| `src/lib/components/layout/Sidebar.svelte` | Hide sections (lines 729-1112), add Modules section | Add "You" menu item |
| `src/lib/constants.ts` | `APP_NAME` | `YOULAB_API_BASE_URL` |

---

## 2. Shared Dependencies

### youlabMode Flag

**Created by**: Phase A (Phase 3)
**Location**: `src/lib/stores/index.ts`
**Used by**: Phase B (Phase 6)

**A must export from `$lib/stores/index.ts`**:
```typescript
// src/lib/stores/index.ts
export const youlabMode = writable(true);
```

**B imports and uses**:
```svelte
<script>
    import { youlabMode } from '$lib/stores';
</script>

{#if $youlabMode}
    <!-- "You" section only visible in YouLab mode -->
{/if}
```

**Interface Contract**:
- Type: `Writable<boolean>`
- Default: `true` (always YouLab mode for now)
- Future: May be controlled via backend config

### YOULAB_API_BASE_URL Constant

**Created by**: Phase A (Phase 3) - add during constants update
**Location**: `src/lib/constants.ts`
**Used by**: Phase B (Phase 6)

**A must add to `src/lib/constants.ts`**:
```typescript
// YouLab backend API base URL
export const YOULAB_API_BASE_URL = import.meta.env.YOULAB_API_BASE_URL || 'http://localhost:8000';
```

**B imports and uses**:
```typescript
import { YOULAB_API_BASE_URL } from '$lib/constants';

await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/blocks`);
```

### pendingDiffsCount Store

**Created by**: Phase B (Phase 6)
**Location**: `src/lib/stores/memory.ts`
**Used by**: Sidebar.svelte (Phase B adds import)

This is B-only; no coordination needed with A.

---

## 3. Integration Points

### Sidebar.svelte Modifications

Both phases modify `Sidebar.svelte`. To avoid merge conflicts:

**Phase A adds** (around line 1076, after Models section):
```svelte
<!-- === PHASE A: MODULES SECTION === -->
{#if $youlabMode}
    <Folder
        id="sidebar-modules"
        bind:open={showModules}
        className="px-2 mt-0.5"
        name={$i18n.t('Modules')}
    >
        <ModuleList />
    </Folder>
{/if}
<!-- === END PHASE A === -->
```

**Phase B adds** (after Notes/Knowledge items, before workspace section):
```svelte
<!-- === PHASE B: YOU SECTION === -->
{#if $youlabMode}
    <a
        class="px-2.5 py-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-850 flex items-center gap-2"
        href="/you"
    >
        <User class="size-4" />
        <span>You</span>
        {#if $pendingDiffsCount > 0}
            <Badge type="error" size="sm">{$pendingDiffsCount}</Badge>
        {/if}
    </a>
{/if}
<!-- === END PHASE B === -->
```

**Section Order in Sidebar** (after both merge):
1. Chat history (existing)
2. **"You" link** (Phase B) - navigation item
3. Notes/Knowledge (existing, shown if enabled)
4. Models (existing, hidden by youlabMode in Phase A)
5. **Modules** (Phase A) - collapsible folder with module list
6. Workspace (hidden by youlabMode in Phase A)

### "You" Menu Item Placeholder

**Phase A does NOT create a placeholder** for "You".

- Phase A focuses on: Branding, theme, hiding OpenWebUI sections, adding Modules
- Phase B owns the entire "You" section: menu item, page, components

If Phase A merges first and Phase B is still in progress, there will simply be no "You" link until Phase B merges.

---

## 4. Merge Strategy

### Recommended Git Workflow

```
main
├── feature/ARI-79-phase-a-ui-modules     (Phase A branch)
└── feature/ARI-80-memory-system-mvp      (Phase B branch)
```

### Merge Order

1. **Phase A merges first** (creates foundation)
   - Establishes `youlabMode` store
   - Adds `YOULAB_API_BASE_URL` constant
   - Sets up Sidebar modifications pattern

2. **Phase B rebases on main** (after A merges)
   - `git fetch origin && git rebase origin/main`
   - Resolve conflicts in Sidebar.svelte:
     - Keep A's Modules section
     - Add B's "You" link in correct location
   - Verify imports work (`youlabMode`, `YOULAB_API_BASE_URL`)

3. **Phase B merges**

### Handling Parallel Development

If both branches need to be active simultaneously:

**Option 1: B branches from A**
```bash
git checkout feature/ARI-79-phase-a-ui-modules
git checkout -b feature/ARI-80-memory-system-mvp
# B can see A's changes directly
```

**Option 2: B tracks A** (if already separate)
```bash
# In Phase B branch, periodically:
git fetch origin feature/ARI-79-phase-a-ui-modules
git merge origin/feature/ARI-79-phase-a-ui-modules
# Keep B up-to-date with A's progress
```

### Conflict Resolution Guide

**Sidebar.svelte conflicts**:
- Both add imports at top → combine import lists
- Both add sections → preserve order (You before Modules)
- Both check `youlabMode` → no conflict (different sections)

**constants.ts conflicts**:
- Keep both additions (`APP_NAME` and `YOULAB_API_BASE_URL`)

**stores/index.ts conflicts**:
- Phase A adds `youlabMode`, `theme` default change
- Phase B adds nothing to this file (creates separate `memory.ts`)
- Should be conflict-free

---

## 5. Integration Testing

### Pre-Merge Checklist (Each Phase)

**Phase A before merge**:
- [ ] `npm run build` succeeds
- [ ] `npm run check` shows no errors
- [ ] App loads with YouLab branding
- [ ] Modules section visible (with test module)
- [ ] Hidden sections not visible (Workspace, Controls, Arena)
- [ ] `YOULAB_API_BASE_URL` constant exported

**Phase B before merge**:
- [ ] `npm run build` succeeds
- [ ] `npm run check` shows no errors
- [ ] "You" link appears in sidebar
- [ ] `/you` route loads
- [ ] Block cards display
- [ ] Block editing works (requires backend running)

### Post-Integration Smoke Test

After both phases are merged:

**UI Verification**:
- [ ] App shows "YouLab" branding (title, logo)
- [ ] YouLab theme active by default (dark mode)
- [ ] Sidebar shows "You" link with badge (if pending diffs)
- [ ] Sidebar shows "Modules" collapsible section
- [ ] Sidebar does NOT show: Workspace, Controls, Arena
- [ ] Knowledge link still works

**Functional Verification**:
- [ ] Click "You" → navigates to `/you`
- [ ] Block cards load from API
- [ ] Click block → opens detail modal
- [ ] Edit block → saves to backend
- [ ] Version history displays
- [ ] Restore version works
- [ ] Click module → opens chat with that model
- [ ] Module status displays correctly

**API Integration**:
- [ ] `YOULAB_API_BASE_URL` correctly points to YouLab server
- [ ] API calls authenticated (token passed)
- [ ] Error states handled (backend down, 404, etc.)

### Regression Checks

- [ ] Chat functionality still works
- [ ] Model selection still works
- [ ] Settings modal opens correctly
- [ ] Theme switching works (can switch away from YouLab theme and back)
- [ ] Mobile responsive behavior intact

---

## 6. Communication Protocol

### During Development

1. **Daily sync**: If both phases active, quick check on:
   - Which files each is currently modifying
   - Any blockers from missing dependencies

2. **Before modifying shared files**:
   - Comment in Linear ticket
   - Or quick Slack message
   - "About to modify Sidebar.svelte for X"

3. **When creating shared dependencies**:
   - Phase A creates `youlabMode` → update this doc with exact export
   - Phase A creates `YOULAB_API_BASE_URL` → update this doc with exact value

### At Merge Time

1. **Phase A about to merge**:
   - Notify Phase B team
   - Confirm all shared dependencies are exported correctly
   - Wait for confirmation before merging

2. **Phase B rebasing**:
   - Pull latest main (includes A)
   - Resolve conflicts per guide above
   - Run full test suite
   - Get review approval
   - Merge

---

## References

- [ARI-79 Phase A Plan](./2026-01-12-ARI-79-phase-a-ui-modules-foundation.md)
- [ARI-80 Phase B Plan](./2026-01-12-ARI-80-memory-system-mvp.md)
- Parent ticket: ARI-78

---

*Coordination document created: 2026-01-12*
