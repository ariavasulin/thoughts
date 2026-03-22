---
date: 2026-03-15T00:00:00-05:00
researcher: ARI
git_commit: b6f1a40a6b4e98458cbe20f8b706b74aa672cbba
branch: main
repository: Askademia
topic: "How to rename the 'Models' header to 'Modules' in the sidebar"
tags: [research, codebase, sidebar, i18n, models, renaming]
status: complete
last_updated: 2026-03-15
last_updated_by: ARI
---

# Research: Renaming "Models" Header to "Modules" in the Sidebar

**Date**: 2026-03-15
**Researcher**: ARI
**Git Commit**: b6f1a40a6b4e98458cbe20f8b706b74aa672cbba
**Branch**: main
**Repository**: Askademia

## Research Question
In the menu sidebar on the left, how would I rename the "Models" header to "Modules"?

## Summary

The "Models" header in the left sidebar is rendered via the i18n translation system at `open-webui/src/lib/components/layout/Sidebar.svelte:1057`. It uses `$i18n.t('Models')` as the `name` prop of a `<Folder>` component. To rename just the sidebar header, change the i18n key in that one line. To rename all occurrences globally, override the translation value in the locale files.

## Detailed Findings

### Sidebar Implementation

The sidebar is implemented in `open-webui/src/lib/components/layout/Sidebar.svelte`. The "Models" section appears at lines 1052-1063:

```svelte
{#if ($models ?? []).length > 0 && (($settings?.pinnedModels ?? []).length > 0 || $config?.default_pinned_models)}
    <Folder
        id="sidebar-models"
        bind:open={showPinnedModels}
        className="px-2 mt-0.5"
        name={$i18n.t('Models')}
        chevron={false}
        dragAndDrop={false}
    >
        <PinnedModelList bind:selectedChatId {shiftKey} />
    </Folder>
{/if}
```

The `<Folder>` component (`open-webui/src/lib/components/common/Folder.svelte`) accepts a `name` prop (line 16: `export let name = ''`) and renders it as text at line 167: `{name}`.

### i18n System

- All UI text goes through `$i18n.t('Key')` where the key itself is the English fallback
- The English locale file (`open-webui/src/lib/i18n/locales/en-US/translation.json`) has `"Models": ""` at line 1256 — the empty value means the key "Models" is displayed as-is
- 59 locale files exist, each containing a `"Models"` translation key

### Scope of `$i18n.t('Models')`

The `"Models"` translation key is used in ~55 files across the codebase, including:
- **Sidebar header**: `Sidebar.svelte:1057`
- **Workspace tab**: `workspace/+layout.svelte:94`
- **Admin settings tab**: `admin/Settings.svelte:113`
- **Various admin, chat, and workspace components**

### Renaming Approaches

**Approach A — Sidebar-only rename (change the i18n key):**
Edit `Sidebar.svelte:1057`: change `$i18n.t('Models')` to `$i18n.t('Modules')`

**Approach B — Global rename (override translation value):**
Edit `en-US/translation.json:1256`: change `"Models": ""` to `"Models": "Modules"`
This changes every place that calls `$i18n.t('Models')` across the entire UI.

## Code References

- `open-webui/src/lib/components/layout/Sidebar.svelte:1057` — Sidebar "Models" header via `name={$i18n.t('Models')}`
- `open-webui/src/lib/components/common/Folder.svelte:16,167` — Folder component that renders the name prop
- `open-webui/src/lib/i18n/locales/en-US/translation.json:1256` — English translation key `"Models": ""`
- `open-webui/src/routes/(app)/workspace/+layout.svelte:94` — Workspace tab using same key
- `open-webui/src/lib/components/admin/Settings.svelte:113` — Admin settings tab config

## Architecture Documentation

The sidebar uses collapsible `<Folder>` components for each section (Models, Channels, Folders, Chats). Each section header is localized through the i18n system. The sidebar has two layout modes: collapsed (icon-only) and expanded (full text labels). The "Models" section specifically shows pinned models and is conditionally rendered based on whether models exist and are pinned.
