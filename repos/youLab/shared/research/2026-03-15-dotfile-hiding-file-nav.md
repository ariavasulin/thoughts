---
date: 2026-03-15T12:00:00-05:00
researcher: ARI
git_commit: b6f1a40a6
branch: main
repository: Askademia
topic: "Where to implement dot file hiding in the file nav — Open Terminal vs Open WebUI"
tags: [research, codebase, file-nav, open-terminal, dotfiles]
status: complete
last_updated: 2026-03-15
last_updated_by: ARI
---

# Research: Dot File Hiding in File Nav

**Date**: 2026-03-15
**Researcher**: ARI
**Git Commit**: b6f1a40a6
**Branch**: main
**Repository**: Askademia

## Research Question
If I wanted to hide dot files in the file nav for the Open Terminal, would this make sense in Open Terminal itself or in Open WebUI? Is there already a way to do this?

## Summary

**No existing mechanism exists** for hiding dot files in the terminal file nav. Dot files are shown unfiltered at every layer of the pipeline. However, Open WebUI already has a dot file filtering pattern in the Knowledge Base upload code (`KnowledgeBase.svelte`) that could serve as a reference.

**Where to implement:** Either location works, but they serve different purposes:
- **Open Terminal backend** (`fs.py:listdir`) — server-side filtering, reduces payload, affects all consumers of the API
- **Open WebUI frontend** (`FileNav.svelte:loadDir`) — client-side filtering, allows a toggle UI, only affects the file nav display

## Detailed Findings

### Current Data Flow (no filtering at any layer)

1. `FileNav.svelte:246` → `loadDir(path)` called
2. `terminal/index.ts:58` → `listFiles(url, key, path)` HTTP GET to `/files/list?directory={path}`
3. `open-terminal/main.py:358` → `list_files()` endpoint
4. `fs.py:142` → `UserFS.listdir()` calls `os.listdir(path)` — returns ALL entries including dotfiles
5. `FileNav.svelte:268` → entries sorted (dirs first, then alpha) — no filtering
6. `FileNav.svelte:996` → all entries rendered via `FileEntryRow`

### Open Terminal Backend — `UserFS.listdir()` (`open-terminal/open_terminal/utils/fs.py:142-162`)

The `listdir` method iterates over `os.listdir(path)` and only filters on `is_path_allowed()` (multi-user isolation). No name-based filtering.

### Open WebUI Frontend — `FileNav.svelte` (`open-webui/src/lib/components/chat/FileNav.svelte`)

- `loadDir()` at line 246: fetches entries, sorts them, stores in `entries` array
- Rendering at line 994: `{#each entries as entry}` — renders all entries without filtering
- No toggle or setting for showing/hiding dotfiles

### Open Terminal Configuration — No Relevant Options

- `open-terminal/open_terminal/env.py` — no file filtering env vars
- `open-terminal/open_terminal/config.py` — TOML config supports no file filtering options

### Existing Dot File Filtering Pattern in Codebase

**KnowledgeBase.svelte** (`open-webui/src/lib/components/workspace/Knowledge/KnowledgeBase.svelte:360-476`):
- `hasHiddenFolder(path)` helper: `path.split('/').some(part => part.startsWith('.'))`
- `entry.name.startsWith('.')` checks during directory upload
- Used to skip hidden files/dirs when uploading folders to Knowledge Base

**PyodideFileNav** (`open-webui/src/lib/components/chat/PyodideFileNav.svelte:115`):
- Also has no dot file filtering — same pattern as FileNav

**Pyodide Worker** (`open-webui/src/lib/workers/pyodide.worker.ts:123`):
- Filters `.` and `..` only, not dotfiles

### Other Open Terminal File Operations — Also Unfiltered

- `/files/grep` (`main.py:721`) — no dotfile filtering
- `/files/glob` (`main.py:783`) — no dotfile filtering
- `fs.walk()` (`fs.py:164`) — no dotfile filtering

## Code References

- `open-terminal/open_terminal/utils/fs.py:142-162` — `listdir()` implementation
- `open-terminal/open_terminal/main.py:358-377` — `/files/list` endpoint
- `open-webui/src/lib/components/chat/FileNav.svelte:246-273` — `loadDir()` function
- `open-webui/src/lib/components/chat/FileNav.svelte:994-1009` — entry rendering loop
- `open-webui/src/lib/apis/terminal/index.ts:58-77` — `listFiles()` API client
- `open-webui/src/lib/components/workspace/Knowledge/KnowledgeBase.svelte:360-476` — existing dotfile filter pattern

## Architecture Documentation

The file nav pipeline is: **FileNav.svelte → terminal API client → Open Terminal `/files/list` → `UserFS.listdir()` → `os.listdir()`**. No layer currently applies dot file filtering. The only filtering in the pipeline is multi-user path isolation via `is_path_allowed()`.
