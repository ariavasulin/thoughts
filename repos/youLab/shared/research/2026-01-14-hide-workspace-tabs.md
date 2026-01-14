---
date: 2026-01-14T17:09:40+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "How to hide Tools and Prompts tabs on the workspace page"
tags: [research, openwebui, workspace, permissions, configuration]
status: complete
last_updated: 2026-01-14
last_updated_by: ariasulin
---

# Research: How to Hide Tools and Prompts Tabs on the Workspace Page

**Date**: 2026-01-14T17:09:40+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question
Find out if there is a way in config or settings to hide the Tools and Prompts tabs on the workspace page.

## Summary

Yes, there are **two primary methods** to hide the Tools and Prompts tabs:

1. **Environment Variables** - Set user permissions to control tab visibility for non-admin users
2. **youlabMode Flag** - A custom flag that hides the entire Workspace section from the sidebar

Admin users (`role === 'admin'`) always see all tabs regardless of permission settings.

## Detailed Findings

### Method 1: Environment Variables (Recommended)

Set these environment variables to hide specific tabs for non-admin users:

```bash
# Hide Prompts tab
USER_PERMISSIONS_WORKSPACE_PROMPTS_ACCESS=False

# Hide Tools tab
USER_PERMISSIONS_WORKSPACE_TOOLS_ACCESS=False
```

**Location of config definition**: `OpenWebUI/open-webui/backend/open_webui/config.py:1257-1264`

These can be set in:
- `.env` file in the OpenWebUI backend directory
- Docker environment variables
- System environment before starting the server

### Method 2: youlabMode Flag (Hides Entire Workspace)

A custom `youlabMode` store flag exists that hides the entire Workspace section:

**Location**: `OpenWebUI/open-webui/src/lib/stores/index.ts:35`
```typescript
export const youlabMode = writable(true);
```

When `youlabMode` is `true`, the Workspace button is hidden from the sidebar entirely.

**Sidebar check location**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte:811-826`

### Tab Visibility Logic

The workspace tabs are rendered conditionally in the layout component:

**File**: `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte:85-123`

```svelte
{#if $user?.role === 'admin' || $user?.permissions?.workspace?.prompts}
    <a href="/workspace/prompts">{$i18n.t('Prompts')}</a>
{/if}

{#if $user?.role === 'admin' || $user?.permissions?.workspace?.tools}
    <a href="/workspace/tools">{$i18n.t('Tools')}</a>
{/if}
```

**Access control redirect**: Lines 23-43 also redirect users without permissions who try to access tabs directly via URL.

### Permission Resolution Flow

1. Environment variables set default permissions (`config.py:1472-1536`)
2. Group permissions can override defaults (`utils/access_control.py:28-68`)
3. User's resolved permissions are stored in `$user.permissions` on frontend
4. Each tab checks `$user?.role === 'admin' || $user?.permissions?.workspace?.{feature}`

### All Workspace Tab Environment Variables

| Variable | Default | Tab Affected |
|----------|---------|--------------|
| `USER_PERMISSIONS_WORKSPACE_MODELS_ACCESS` | False | Models |
| `USER_PERMISSIONS_WORKSPACE_KNOWLEDGE_ACCESS` | False | Knowledge |
| `USER_PERMISSIONS_WORKSPACE_PROMPTS_ACCESS` | False | Prompts |
| `USER_PERMISSIONS_WORKSPACE_TOOLS_ACCESS` | False | Tools |

## Code References

- `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte:85-123` - Tab rendering logic
- `OpenWebUI/open-webui/src/routes/(app)/workspace/+layout.svelte:23-43` - Access control redirect
- `OpenWebUI/open-webui/backend/open_webui/config.py:1257-1264` - Prompts/Tools permission env vars
- `OpenWebUI/open-webui/backend/open_webui/config.py:1472-1536` - DEFAULT_USER_PERMISSIONS structure
- `OpenWebUI/open-webui/src/lib/stores/index.ts:35` - youlabMode flag
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte:811-826` - Sidebar workspace visibility
- `OpenWebUI/open-webui/backend/open_webui/utils/access_control.py:28-68` - Permission resolution

## Architecture Documentation

### Permission System

```
Environment Variables
        ↓
  DEFAULT_USER_PERMISSIONS (config.py)
        ↓
  Group Permissions Override (optional)
        ↓
  get_permissions() resolves final permissions
        ↓
  Frontend checks $user.permissions for visibility
```

### Important Notes

1. **Admin Bypass**: Admin users always see all tabs regardless of permission settings
2. **Persistent Config**: If `ENABLE_PERSISTENT_CONFIG=True`, database values override environment variables
3. **No Admin UI**: There's no web interface to set these permissions - must use environment variables or direct database modification
4. **Profile Tab**: Always visible to all authenticated users (not permission-gated)

## Open Questions

- Should there be an Admin UI to configure default user permissions?
- Should youlabMode be configurable via environment variable instead of hardcoded?
