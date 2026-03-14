---
date: 2026-03-14T12:00:00-05:00
researcher: ARI
git_commit: 470afd4214978159e7b45b41cfa6a640707918e0
branch: main
repository: Askademia
topic: "How to replace Open WebUI logo/icon and title with Askademia branding"
tags: [research, codebase, branding, openwebui, logo, favicon, customization]
status: complete
last_updated: 2026-03-14
last_updated_by: ARI
---

# Research: Open WebUI Branding Customization (Logo, Icon, Title)

**Date**: 2026-03-14
**Researcher**: ARI
**Git Commit**: 470afd42
**Branch**: main
**Repository**: Askademia

## Research Question
How do we replace the icon in the top-left corner of Open WebUI and the title from "Open WebUI" to "Askademia"? What file format and size should the new logo/icon be?

## Summary

Rebranding requires two categories of changes: (1) replacing image assets and (2) changing the app name. The logo/icon displayed in the sidebar top-left is `favicon.png` (500x500 PNG). The app name is controlled by the `WEBUI_NAME` environment variable and a frontend constant. A total of 8 PNG image files, 1 SVG, 1 ICO, and several text references need updating.

## Detailed Findings

### 1. Image Assets to Replace

All source images live in `open-webui/static/static/`. The backend copies them to `open-webui/backend/open_webui/static/` at startup. The build output in `open-webui/build/static/` is generated.

| File | Dimensions | Format | Purpose |
|------|-----------|--------|---------|
| `favicon.png` | 500x500 | PNG RGBA | **Primary logo** - sidebar top-left, browser tab icon, fallback icon everywhere |
| `logo.png` | 500x500 | PNG RGBA | Application logo (used in onboarding, admin settings) |
| `splash.png` | 500x500 | PNG RGBA | Loading splash screen (light mode), app sidebar home button |
| `splash-dark.png` | 500x500 | PNG RGBA | Loading splash screen (dark mode) |
| `favicon-96x96.png` | 96x96 | PNG RGBA | High-res browser favicon |
| `apple-touch-icon.png` | 180x180 | PNG RGBA | iOS home screen icon |
| `web-app-manifest-192x192.png` | 192x192 | PNG RGBA | PWA icon (maskable) |
| `web-app-manifest-512x512.png` | 512x512 | PNG RGBA | PWA icon (maskable) |
| `favicon.svg` | scalable | SVG | Vector favicon for modern browsers |
| `favicon.ico` | 48x48, 32x32 | ICO | Legacy favicon (multi-resolution) |

**Recommended approach**: Create the Askademia logo as a 512x512 PNG (or larger), then generate all required sizes from it.

### 2. Where the Top-Left Icon Is Rendered

**Sidebar component** (`open-webui/src/lib/components/layout/Sidebar.svelte`):

- **Expanded sidebar** (lines 895-907): Shows `{WEBUI_BASE_URL}/static/favicon.png` as a 24px (`size-6`) rounded circle next to the app name
- **Collapsed sidebar** (lines 707-711): Shows the same `favicon.png` as a small icon button
- **App name** (lines 909-916): Displays `{$WEBUI_NAME}` as text next to the logo

### 3. App Name Configuration

#### Backend (env.py:135-137)
```python
WEBUI_NAME = os.environ.get("WEBUI_NAME", "Open WebUI")
if WEBUI_NAME != "Open WebUI":
    WEBUI_NAME += " (Open WebUI)"
```
**Important**: Setting `WEBUI_NAME=Askademia` via env var results in "Askademia (Open WebUI)" due to the suffix logic. To get just "Askademia", you must either:
- Modify `env.py` to remove the suffix append (lines 136-137)
- Or set `WEBUI_NAME` to "Open WebUI" and change the `APP_NAME` constant on the frontend

#### Frontend constant (src/lib/constants.ts:4)
```typescript
export const APP_NAME = 'Open WebUI';
```

#### Frontend store (src/lib/stores/index.ts:10-11)
```typescript
export const WEBUI_NAME = writable(APP_NAME);
```
The store gets overwritten with the backend value on layout load (`+layout.svelte:862`).

### 4. All Locations That Reference "Open WebUI" Text

| Location | File | Line(s) |
|----------|------|---------|
| HTML `<title>` | `src/app.html` | 106 |
| Svelte `<title>` | `src/routes/+layout.svelte` | 958-970 |
| Frontend constant | `src/lib/constants.ts` | 4 |
| Backend env default | `backend/open_webui/env.py` | 135 |
| Web manifest (static) | `static/static/site.webmanifest` | 2-3 |
| Web manifest (backend) | `backend/open_webui/static/site.webmanifest` | 2-3 |
| OpenSearch XML | `static/opensearch.xml` | 2-3 |
| Desktop notifications | `src/routes/+layout.svelte` | 451, 654 |
| PWA manifest (dynamic) | `backend/open_webui/main.py` | 2555-2557 |

### 5. Icon File Format & Size Requirements

**Required image set for full rebranding:**

1. **Master source**: 512x512+ PNG with transparency (RGBA), square aspect ratio
2. From the master, generate:
   - `favicon.png` — 500x500 PNG RGBA
   - `logo.png` — 500x500 PNG RGBA (can be same as favicon or include wordmark)
   - `splash.png` — 500x500 PNG RGBA (light mode variant)
   - `splash-dark.png` — 500x500 PNG RGBA (dark mode variant, or inverted)
   - `favicon-96x96.png` — 96x96 PNG RGBA
   - `apple-touch-icon.png` — 180x180 PNG RGBA
   - `web-app-manifest-192x192.png` — 192x192 PNG RGBA (maskable: safe zone is inner 80%)
   - `web-app-manifest-512x512.png` — 512x512 PNG RGBA (maskable: safe zone is inner 80%)
   - `favicon.svg` — SVG version of the icon
   - `favicon.ico` — ICO with 48x48 and 32x32 variants (32-bit color)

**Key constraints**:
- All PNG files use RGBA (transparency support)
- Square aspect ratio (1:1)
- PWA icons are `maskable` — keep important content within the inner 80% circle/area
- The sidebar renders the icon at 24px (`size-6`) with `rounded-full` (circular crop)
- Dark mode: if `favicon-dark.png` doesn't exist, the auth page applies CSS `invert(1)` as fallback

## Code References

- `open-webui/static/static/` — Source image assets directory
- `open-webui/backend/open_webui/static/` — Backend-served static assets (copied at startup)
- `open-webui/src/lib/components/layout/Sidebar.svelte:895-916` — Top-left logo + name rendering
- `open-webui/backend/open_webui/env.py:135-137` — WEBUI_NAME env var with suffix logic
- `open-webui/src/lib/constants.ts:4` — APP_NAME frontend constant
- `open-webui/src/app.html:106` — Static HTML title
- `open-webui/static/static/site.webmanifest:1-3` — PWA manifest name fields
- `open-webui/backend/open_webui/config.py:874-912` — Static file copy logic at startup

## Architecture Documentation

The branding system follows this flow:
1. **Startup**: Backend reads `WEBUI_NAME` from env, copies frontend build assets to `STATIC_DIR`
2. **API**: `/api/config` endpoint exposes `app.state.WEBUI_NAME` to frontend
3. **Frontend init**: Layout component fetches config and sets the `WEBUI_NAME` Svelte store
4. **Rendering**: All components read from `$WEBUI_NAME` store and `{WEBUI_BASE_URL}/static/` for images
5. **Static serving**: FastAPI mounts `STATIC_DIR` at `/static/` URL path

## Open Questions

- Should the `env.py` suffix logic (`+ " (Open WebUI)"`) be removed entirely for a clean "Askademia" display, or kept for attribution?
- Does the `CUSTOM_NAME` legacy API mechanism (config.py:927-963) need to be disabled to prevent overwriting custom assets?
- Should a dark-mode-specific icon (`favicon-dark.png`) be created, or rely on the CSS `invert(1)` fallback?
