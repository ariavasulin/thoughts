---
date: 2026-01-24T18:33:00-08:00
researcher: ariasulin
git_commit: 2611ba2
branch: main
repository: YouLab
openwebui_commit: b3e763d2d
openwebui_upstream: origin/main (0.6.43)
topic: "OpenWebUI Modifications for YouLab"
tags: [research, codebase, openwebui, frontend, patches, upgrade-compatibility]
status: complete
last_updated: 2026-01-24
last_updated_by: ariasulin
---

# Research: OpenWebUI Modifications for YouLab

**Date**: 2026-01-24T18:33:00-08:00
**Researcher**: ariasulin
**Git Commit**: 2611ba2 (YouLab main)
**OpenWebUI Fork Commit**: b3e763d2d
**Upstream Base**: origin/main (0.6.43 - commit a7271532f)
**Repository**: YouLab

## Research Question

Analyze OpenWebUI modifications for YouLab. Compare OpenWebUI/open-webui/ against upstream. Document: custom Svelte components, backend modifications, config changes. Assess: which patches are essential vs nice-to-have, what can be upstreamed, upgrade compatibility.

## Summary

YouLab maintains a fork of OpenWebUI at `OpenWebUI/open-webui/` with **4 commits ahead of upstream** adding:

1. **Memory Block System** - A git-backed content editing system with diff approval workflow
2. **Module Navigation** - Course module sidebar with folder-based chat organization
3. **YouLab Branding** - Custom themes, fonts, icons, and app name
4. **UI Simplifications** - Hide power-user features (Controls panel, Arena, etc.)

The modifications total **~3,700 lines added** across 65 files. The architecture is clean: backend modifications are minimal (branding only), with all YouLab logic in frontend components that call the separate YouLab HTTP service.

## Patch Classification

### Essential Patches (Core YouLab Functionality)

| Category | Files | Lines | Description |
|----------|-------|-------|-------------|
| Memory Block UI | 8 components | ~2,400 | Block list, editors, diff approval overlay |
| Memory API Client | 2 files | ~520 | TypeScript API for blocks/notes endpoints |
| Memory Store | 1 file | ~50 | Svelte stores for block state |
| Module Navigation | 3 files | ~320 | Sidebar modules, folder utilities |
| Routes | 4 files | ~50 | /you, /workspace/profile, block editor |
| Chat Integration | 2 files | ~80 | Module folder creation, title generation |

### Nice-to-Have Patches (Branding/UX)

| Category | Files | Lines | Description |
|----------|-------|-------|-------------|
| Theme System | 3 files | ~150 | youlab-dark/youlab themes with iA Writer colors |
| Custom Font | 2 files | ~10 | iA Writer Duo variable font |
| Branding Assets | 16 files | (binary) | Favicons, logos, splash screens |
| About Page | 1 file | -30 | YouLab attribution |
| UI Hiding | 4 files | ~30 | youlabMode store for feature hiding |

### Minimal/Isolated Patches

| Category | Files | Description |
|----------|-------|-------------|
| Backend Branding | 1 file | `WEBUI_NAME = "YouLab"` default |
| CodeRabbit Config | 1 file | `.coderabbit.yaml` for PR reviews |

## Detailed Findings

### 1. Memory Block System Components

**Location**: `src/lib/components/you/`

8 new Svelte components implementing a git-backed content editing system:

| Component | Lines | Purpose |
|-----------|-------|---------|
| `BlockEditor.svelte` | 539 | Full-page rich text editor with autosave, AI enhancement |
| `MemoryBlockEditor.svelte` | 423 | Notes API editor with git-based undo/redo |
| `DiffApprovalOverlay.svelte` | 353 | Full-screen diff review with keyboard shortcuts |
| `BlockDetailModal.svelte` | 384 | Modal editor with unified diff view |
| `AgentsTab.svelte` | 118 | Background agents list with thread links |
| `BlockEditorPanel.svelte` | 97 | Side panel for settings and pending diffs |
| `BlockCard.svelte` | 42 | Memory block card for list view |
| `DiffBadge.svelte` | 14 | Pending diff count badge |

**Key Features**:
- Dual editor variants: `BlockEditor` (legacy API) and `MemoryBlockEditor` (Notes API)
- Git-based version navigation with restore capability
- AI-generated diff approval workflow with confidence badges
- Keyboard shortcuts (A=approve, R=reject, E=edit, arrows=navigate)
- Autosave with 200ms debounce

### 2. Memory API Layer

**Location**: `src/lib/apis/memory/`

| File | Lines | Endpoints |
|------|-------|-----------|
| `index.ts` | 389 | Direct blocks API (`/users/{userId}/blocks/*`) |
| `notes.ts` | 134 | Notes adapter API (`/api/you/notes/*`) |

**index.ts endpoints**:
- `GET /users/{userId}/blocks` - List all blocks
- `GET/PUT /users/{userId}/blocks/{label}` - Get/update block
- `GET /users/{userId}/blocks/{label}/history` - Version history
- `GET /users/{userId}/blocks/{label}/diffs` - Pending changes
- `POST /users/{userId}/blocks/{label}/diffs/{diffId}/approve` - Approve diff
- `POST /users/{userId}/blocks/{label}/diffs/{diffId}/reject` - Reject diff
- `GET /users/{userId}/agents` - Background agents

**notes.ts endpoints**:
- `GET /api/you/notes/` - List blocks as notes
- `GET /api/you/notes/{noteId}` - Get single block
- `POST /api/you/notes/{noteId}/update` - Update block

**Configuration**: `YOULAB_API_BASE_URL` from `src/lib/constants.ts:17-19` (defaults to `http://localhost:8100`)

### 3. Module Navigation System

**Location**: `src/lib/components/layout/Sidebar/`

| File | Lines | Purpose |
|------|-------|---------|
| `ModuleList.svelte` | 91 | Module list container, discovers modules from model metadata |
| `ModuleItem.svelte` | 101 | Individual module with status icons and folder navigation |

**Sidebar.svelte modifications** (~130 lines added):
- Import `youlabMode` store and `ModuleList` component
- Add "You" navigation link with pending diffs badge
- Add "Modules" collapsible section (visible when `$youlabMode`)

**Folder utilities** (`src/lib/utils/folders.ts`):
- `ensureModuleFolder()` - Create/retrieve folder for module
- `getMostRecentThreadInFolder()` - Find latest chat in folder
- In-memory folder cache to prevent redundant API calls

**Module Discovery**:
- Modules identified by `model.info.meta.youlab_module` metadata
- Status: `locked`, `available`, `in_progress`, `completed`
- Ordering via `module_index` metadata field

### 4. Chat.svelte Integration

**Lines changed**: ~80 (additions only)

Two integration points for module-based chat:

**Auto-folder assignment** (lines 2254-2291):
```javascript
// If model has youlab_module, ensure folder exists
if (hasYoulabModule && !folderId) {
  folderId = await ensureModuleFolder(localStorage.token, model.id, model.name);
}

// Generate title: "Module Name - Jan 24, 2026"
if (isModuleChat) {
  chatTitle = `${moduleName} - ${date}`;
}
```

**Disable auto-title for modules** (lines 1970-1975):
```javascript
title_generation: model?.info?.meta?.youlab_module
  ? false
  : ($settings?.title?.auto ?? true),
```

### 5. YouLab Mode Feature Hiding

**Store**: `src/lib/stores/index.ts:35`
```javascript
export const youlabMode = writable(true);
```

**Components that check `$youlabMode`**:
- `ChatControls.svelte` - Hides entire Controls panel
- `Navbar.svelte` - Hides Controls button
- `Sidebar.svelte` - Shows Modules section, hides Pinned Models

**ModelSelector modifications**:
- Filters out models with `meta.type === 'background_agent'` from UI

### 6. Theme System

**Custom themes added**: `youlab-dark`, `youlab`

**Implementation**:
- `src/app.html` - Theme initialization on page load
- `src/lib/components/chat/Settings/General.svelte` - Theme application logic
- `src/lib/stores/index.ts:32` - Default theme: `'youlab-dark'`

**Color palette** (iA Writer inspired):
```javascript
// youlab-dark
'--color-gray-900': '#1d1f20'  // Background
'--color-gray-50': '#e8ebe9'   // Text
```

**Custom font**: iA Writer Duo variable font
- `static/assets/fonts/iAWriterDuoV.woff2`
- Applied via `src/app.css` and `src/tailwind.css`

### 7. Branding Assets

16 binary files replaced:
- Favicons (multiple sizes and formats)
- Apple touch icon
- Web manifest icons (192x192, 512x512)
- Splash screens (light and dark)
- Logo

**site.webmanifest changes**:
- `name`: "YouLab"
- `short_name`: "YouLab"

### 8. Backend Modifications

**Minimal changes** - 2 files total:

| File | Change |
|------|--------|
| `backend/open_webui/env.py:90` | `WEBUI_NAME = os.environ.get("WEBUI_NAME", "YouLab")` |
| `backend/open_webui/main.py` | No changes (standard OpenWebUI) |

The YouLab integration happens entirely through the **Pipeline (Pipe) system** - see `src/youlab_server/pipelines/letta_pipe.py` in the main YouLab repo.

## Upgrade Compatibility Assessment

### Low Risk (Isolated Changes)

| Category | Risk | Notes |
|----------|------|-------|
| New components (`you/`, `Profile/`) | Low | No upstream conflicts possible |
| New API modules (`apis/memory/`) | Low | No upstream conflicts possible |
| New routes (`you/`, `workspace/profile/`) | Low | No upstream conflicts possible |
| New store (`stores/memory.ts`) | Low | No upstream conflicts possible |
| New utilities (`utils/folders.ts`) | Low | No upstream conflicts possible |
| Binary assets | Low | Simply re-apply after upgrade |

### Medium Risk (Modifications to Existing Files)

| File | Lines Changed | Conflict Risk |
|------|---------------|---------------|
| `Sidebar.svelte` | +130 | Medium - Sidebar frequently updated upstream |
| `Chat.svelte` | +34 | Medium - Core chat component |
| `app.html` | +31 | Low - Theme init is isolated section |
| `General.svelte` | +51 | Low - Theme options are additive |
| `constants.ts` | +7 | Low - Adding new constant |
| `stores/index.ts` | +5 | Low - Adding new stores |
| `app.css` | +9 | Low - Font additions |
| `tailwind.css` | +2 | Low - Font stack modification |

### Upgrade Strategy

1. **Rebase or cherry-pick** YouLab commits onto new upstream tag
2. **Resolve conflicts** - likely in `Sidebar.svelte` and `Chat.svelte`
3. **Re-apply binary assets** - Copy over favicon/logo files
4. **Test theme system** - Upstream CSS variable changes may affect YouLab themes
5. **Test module navigation** - Verify folder API compatibility

## Upstreaming Potential

### Could Be Upstreamed

| Feature | Value to Upstream | Effort |
|---------|-------------------|--------|
| Theme system architecture | High - flexible theming | Low |
| youlabMode pattern | Medium - feature flag pattern | Low |
| Folder utilities | Low - specific to module use case | Low |

### YouLab-Specific (Not for Upstream)

| Feature | Reason |
|---------|--------|
| Memory block system | Requires YouLab backend |
| Module navigation | Requires `youlab_module` metadata |
| Branding assets | YouLab-specific |
| About page changes | YouLab-specific |

## Architecture Documentation

### Integration Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenWebUI Frontend                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   Modules    │  │  Memory UI   │  │  Chat + Sidebar  │   │
│  │  (Sidebar)   │  │ (/you pages) │  │  (youlabMode)    │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
│         │                 │                    │             │
│         ▼                 ▼                    ▼             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Memory API Client Layer                  │   │
│  │         (src/lib/apis/memory/index.ts)               │   │
│  └──────────────────────────┬───────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────┘
                              │ HTTP (port 8100)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    YouLab HTTP Service                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  /users/...  │  │ /api/you/... │  │   Letta Pipe     │   │
│  │   /blocks    │  │    /notes    │  │   Integration    │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### File Organization

```
OpenWebUI/open-webui/
├── src/lib/
│   ├── apis/memory/           # NEW: Memory API client
│   │   ├── index.ts           # Direct blocks API
│   │   └── notes.ts           # Notes adapter API
│   ├── components/
│   │   ├── you/               # NEW: Memory block components
│   │   ├── layout/Sidebar/    # NEW: Module navigation
│   │   └── workspace/Profile/ # NEW: Profile components
│   ├── stores/
│   │   ├── index.ts           # MODIFIED: youlabMode, theme
│   │   └── memory.ts          # NEW: Memory stores
│   ├── utils/
│   │   └── folders.ts         # NEW: Folder utilities
│   └── constants.ts           # MODIFIED: YOULAB_API_BASE_URL
├── src/routes/(app)/
│   ├── you/                   # NEW: Memory routes
│   └── workspace/profile/     # NEW: Profile route
├── static/assets/fonts/       # NEW: iA Writer Duo font
└── backend/open_webui/
    └── env.py                 # MODIFIED: WEBUI_NAME default
```

## Code References

### Essential Files
- `src/lib/components/you/BlockEditor.svelte` - Main block editor
- `src/lib/components/you/DiffApprovalOverlay.svelte` - Diff review UI
- `src/lib/components/layout/Sidebar/ModuleList.svelte` - Module navigation
- `src/lib/apis/memory/index.ts` - Memory API client
- `src/lib/components/chat/Chat.svelte:2250-2291` - Module folder integration

### Configuration
- `src/lib/constants.ts:17-19` - `YOULAB_API_BASE_URL`
- `src/lib/stores/index.ts:32-35` - Theme and youlabMode defaults
- `src/app.html:50-90` - Theme initialization

## Commits on Fork

```
b3e763d2d feat: add block diff approval UI and memory editing
863b40594 feat: add You page for user memory blocks and pending diffs
81c10f9fd feat(ARI-79): YouLab branding, theme, and modules UI
22c9c3b32 feat: add Files menu item to sidebar
```

## Open Questions

1. **Upgrade frequency** - How often should the fork be rebased onto upstream?
2. **Feature flags** - Should `youlabMode` be configurable per-user or remain global?
3. **Theme naming** - Should YouLab themes be prefixed differently to avoid conflicts?
4. **Dual editor** - Why two editor variants (BlockEditor vs MemoryBlockEditor)?
