---
date: 2026-02-10T16:23:03-08:00
researcher: ariasulin
git_commit: cf5dcf7a2195108b0ea2823343599628bd12ed85
branch: main
repository: YouLab
topic: "Current status of syncing local workspace folders and OpenWebUI collections"
tags: [research, codebase, sync, workspace, openwebui, knowledge-base]
status: complete
last_updated: 2026-02-10
last_updated_by: ariasulin
---

# Research: Current Status of Workspace ↔ OpenWebUI Collection Sync

**Date**: 2026-02-10T16:23:03-08:00
**Researcher**: ariasulin
**Git Commit**: cf5dcf7a2195108b0ea2823343599628bd12ed85
**Branch**: main
**Repository**: YouLab

## Research Question

What is the current status of syncing local workspace folders and OpenWebUI collections? What has been built, what is planned, and what remains?

## Summary

There are **three separate sync systems** in different stages of completion:

1. **Ralph Sync Module** (`src/ralph/sync/`) - Python-based bidirectional sync between Ralph user workspaces and OpenWebUI knowledge bases. Core code is written and API endpoints are wired up, but **no background scheduler is active** - sync must be triggered manually via API.

2. **openwebui-content-sync** (`openwebui-content-sync/`) - Standalone Go application that syncs external content sources (GitHub, Confluence, Jira, Slack, local folders) into OpenWebUI knowledge bases on a cron schedule. This is an **external third-party tool** cloned into the repo but **not yet deployed** (explicitly deferred in the VPS deployment plan).

3. **youlab-sync daemon** (`tools/youlab-sync/`) - Go-based local daemon that runs on a user's machine, watches a local folder, and syncs files bidirectionally with Ralph workspaces via HTTP API. Code exists but **no installer or distribution** has been built.

**Net assessment**: The building blocks exist across all three systems, but none are actively running in production. The Ralph sync module is the most integrated (API endpoints mounted in server.py), but lacks the scheduler to make it automatic. The openwebui-content-sync tool is production-ready standalone but hasn't been deployed. The local daemon works but has no distribution mechanism.

## Detailed Findings

### 1. Ralph Sync Module (`src/ralph/sync/`)

**Status**: Code complete, API wired up, no automatic scheduling

#### Components

| File | Purpose |
|------|---------|
| `src/ralph/sync/workspace_sync.py` | Core sync logic - scan, hash, upload/download |
| `src/ralph/sync/openwebui_client.py` | HTTP client for OpenWebUI file and knowledge APIs |
| `src/ralph/sync/knowledge.py` | Per-user knowledge base creation and caching |
| `src/ralph/sync/models.py` | Pydantic models: FileMetadata, SyncState, SyncResult |

#### API Endpoints (`src/ralph/api/workspace.py`)

All mounted under `/users/{user_id}/workspace/`:

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/files` | List workspace files with hashes |
| GET | `/files/{path}` | Download file content |
| PUT | `/files/{path}` | Upload/update file |
| DELETE | `/files/{path}` | Delete file |
| POST | `/sync` | Trigger manual bidirectional sync |

The router is imported and included in `src/ralph/server.py:35,163`.

#### Configuration (`src/ralph/config.py:47-52`)

```bash
RALPH_OPENWEBUI_URL=...           # Base URL for OpenWebUI
RALPH_OPENWEBUI_API_KEY=...       # API key
RALPH_SYNC_INTERVAL=300           # Default 5min (unused - no scheduler)
RALPH_SYNC_TO_OPENWEBUI=true      # Feature flag
RALPH_SYNC_KNOWLEDGE_PREFIX=workspace  # KB naming prefix
```

#### How Sync Works

- **State tracking**: JSON file (`.sync_state.json`) in each user's workspace directory
- **Change detection**: SHA256 content hashing (prefixed `sha256:`)
- **Knowledge base naming**: `workspace-{user_id}` auto-created on first sync
- **Upload flow**: Scan workspace → compare hashes → upload changed files → add to KB
- **Download flow**: Fetch KB files → compare hashes → write changed files to workspace
- **Conflict resolution**: Last-writer-wins (no merge)
- **File size limit**: 10MB per file

#### What's Missing

- **No background scheduler**: `RALPH_SYNC_INTERVAL` is defined but never used in any background task. All sync is manual via `POST /sync`.
- **No automatic trigger**: Files created by agents (via FileTools) don't automatically sync to OpenWebUI KB.
- **No RAG query tools**: Ralph agents cannot query OpenWebUI knowledge bases. The feasibility research (2026-01-28) identified this as a critical gap and proposed `KnowledgeQueryTools`, but it hasn't been built.

### 2. openwebui-content-sync (External Go Tool)

**Status**: Complete standalone tool, cloned but NOT deployed

**Location**: `/Users/ariasulin/Git/YouLab/openwebui-content-sync/` (untracked in git)

This is a third-party Go application with an adapter architecture for syncing external content sources into OpenWebUI knowledge bases.

#### Adapters

| Adapter | Source | Config |
|---------|--------|--------|
| GitHub | Repository files | `github.mappings[].{repository, knowledge_id}` |
| Confluence | Space pages / parent page children | `confluence.space_mappings[]`, `parent_page_mappings[]` |
| Local Folders | Local directories | `local_folders.mappings[].{folder_path, knowledge_id}` |
| Slack | Channel messages | `slack.channel_mappings[]` |
| Jira | Project issues | `jira.project_mappings[]` |

#### Architecture

- **Scheduler**: Cron-based (`@every {interval}`), configurable (default 1h)
- **Change detection**: SHA256 hashing, file index persisted to `file_index.json`
- **File processing**: Adaptive polling up to ~11 minutes for OpenWebUI to process uploaded files
- **Orphan cleanup**: Files removed from source are automatically deleted from KB
- **Health checks**: HTTP `/health` and `/ready` on port 8080
- **Deployment**: Kubernetes-native with PVC for storage, ConfigMap for config, Secrets for credentials

#### Local Folders Adapter Details

The local folders adapter (`internal/adapter/local.go`) is most relevant for workspace sync:
- Recursive directory walking via `filepath.WalkDir()`
- Binary file detection (null bytes + non-printable character ratio)
- Hidden file filtering (files starting with `.`)
- Ignore patterns: `node_modules`, `__pycache__`, `.git`, `*.log`, `*.tmp`, etc.
- Each folder maps to a specific knowledge base ID

#### Deployment Status

Per `thoughts/shared/plans/2026-02-09-vps-deployment.md`:
> "No `openwebui-content-sync` service -- that's for later"

The tool was explicitly excluded from the current VPS deployment phase.

### 3. youlab-sync Local Daemon (`tools/youlab-sync/`)

**Status**: Code exists, no distribution/installer

A Go-based daemon intended to run on student machines, providing real-time file watching and periodic sync with Ralph workspaces.

#### Components

| Path | Purpose |
|------|---------|
| `tools/youlab-sync/cmd/sync.go` | One-time bidirectional sync command |
| `tools/youlab-sync/cmd/watch.go` | Daemon mode with fsnotify file watching |
| `tools/youlab-sync/internal/ralph/client.go` | HTTP client calling Ralph workspace API |
| `tools/youlab-sync/config.example.yaml` | Configuration template |

#### How It Works

- **`sync` command**: Performs a single `FullSync()` (5-minute timeout), then exits
- **`watch` command**: Initial full sync → starts fsnotify watcher → handles file change events via `HandleLocalChange()` → periodic sync ticker at configured interval → graceful shutdown on SIGINT/SIGTERM
- **Communication**: All operations go through Ralph HTTP API (`GET/PUT/DELETE /users/{user_id}/workspace/files/...`)

#### Configuration

```yaml
server:
  url: "https://theyoulab.org"
  api_key: "${YOULAB_API_KEY}"
  user_id: "..."
sync:
  local_folder: "/Users/student/YouLab"
  interval: "30s"
  bidirectional: true
watch:
  enabled: true
  debounce: "500ms"
ignore:
  - ".git"
  - "node_modules"
  - "*.tmp"
```

#### What's Missing

- No installer scripts
- No distribution mechanism (no releases, no Homebrew formula, etc.)
- No tray icon or desktop integration (listed as optional in plan)

## Evolution of Sync Plans (Historical Context)

The sync architecture has evolved through several iterations, tracked in the thoughts directory:

### Phase 1: One-Way OpenWebUI → Letta (Jan 10, 2026)

`thoughts/shared/plans/2026-01-10-file-knowledge-sync.md`

- Original plan: Users upload files in OpenWebUI → sync to Letta folders → agents read files
- MD files routed to Notes, PDFs to Knowledge collections
- 30-second polling-based sync
- Access control mapping: shared vs private folders
- **Status**: Superseded by bidirectional plan

### Phase 2: Bidirectional Letta ↔ OpenWebUI (Jan 11, 2026)

`thoughts/shared/plans/2026-01-11-bidirectional-realtime-sync.md`

- Extended to bidirectional: Letta folders ↔ OpenWebUI Knowledge collections
- Webhook-based forward sync (<1s) with polling fallback
- Phase 1 (reverse sync Letta→OpenWebUI): Marked complete
- Phase 2 (webhook sync): Partially complete (endpoint exists, sync methods incomplete)
- **Status**: Partially implemented, then architecture shifted to Ralph

### Phase 3: Three-Way Ralph Sync (Jan 28, 2026)

`thoughts/shared/plans/2026-01-28-three-way-workspace-sync.md`

- Complete redesign after migration from Letta to Ralph/Agno
- Hub-and-spoke: Ralph workspace (hub) ↔ OpenWebUI KB (spoke 1) ↔ Local folder (spoke 2)
- Produced the current `src/ralph/sync/` module and `tools/youlab-sync/` daemon
- Phase 1 (Ralph ↔ OpenWebUI): ~90% complete (scheduler pending)
- Phase 2 (Local daemon): ~75% complete (distribution pending)
- Phase 3 (UI polish): 0% complete

### Feasibility Study: openwebui-content-sync (Jan 28, 2026)

`thoughts/shared/research/2026-01-28-openwebui-content-sync-integration-feasibility.md`

- Assessed integrating the external Go tool into YouLab
- Recommended "Hybrid" approach: Deploy content-sync as-is + add RAG query tools to Ralph
- Identified critical gap: Ralph agents cannot query OpenWebUI knowledge bases at all
- Proposed `KnowledgeQueryTools` for Ralph agents
- **Status**: Recommendations not yet implemented

## Code References

- `src/ralph/sync/workspace_sync.py:322-413` - sync_to_openwebui() implementation
- `src/ralph/sync/workspace_sync.py:415-497` - sync_from_openwebui() implementation
- `src/ralph/sync/openwebui_client.py:32-290` - OpenWebUI HTTP client
- `src/ralph/sync/knowledge.py:34-120` - KnowledgeService with caching
- `src/ralph/sync/models.py:11-112` - All data models
- `src/ralph/api/workspace.py:86-291` - HTTP API endpoints
- `src/ralph/server.py:35,163` - Router integration
- `src/ralph/config.py:47-52` - Sync configuration
- `tools/youlab-sync/cmd/watch.go:34-143` - Daemon watch mode
- `tools/youlab-sync/internal/ralph/client.go:58-196` - Ralph API client
- `openwebui-content-sync/main.go:35-175` - Go sync app entry point
- `openwebui-content-sync/internal/adapter/local.go:42-238` - Local folders adapter
- `openwebui-content-sync/internal/sync/manager.go:179-250` - Sync manager

## Architecture Documentation

### Current Data Flow (When All Pieces Connect)

```
Student's Machine           Ralph Server (VPS)           OpenWebUI (VPS)
┌─────────────────┐    ┌──────────────────────┐    ┌──────────────────┐
│ Local Folder     │    │ User Workspace       │    │ Knowledge Base   │
│ ~/YouLab/        │◄──►│ /data/users/{id}/    │◄──►│ workspace-{id}   │
│                  │    │   workspace/          │    │                  │
│ youlab-sync      │    │                      │    │ Files for RAG    │
│ daemon (Go)      │    │ Ralph API endpoints  │    │                  │
└─────────────────┘    └──────────────────────┘    └──────────────────┘
      HTTP API              Python sync module         OpenWebUI API
   PUT/GET/DELETE           WorkspaceSync class       files, knowledge
   /workspace/files         .sync_state.json          endpoints
```

### Separate: openwebui-content-sync

```
External Sources            openwebui-content-sync        OpenWebUI
┌─────────────────┐    ┌──────────────────────────┐    ┌──────────────┐
│ GitHub repos     │    │ Go service (Kubernetes)  │    │ Knowledge    │
│ Confluence       │───►│ Adapter → Sync Manager   │───►│ Bases        │
│ Local folders    │    │ Cron scheduler           │    │              │
│ Slack channels   │    │ file_index.json          │    │              │
│ Jira projects    │    └──────────────────────────┘    └──────────────┘
└─────────────────┘           One-way push
```

These are **two independent systems** that both push files to OpenWebUI knowledge bases but serve different purposes:
- Ralph sync: Per-user workspace ↔ per-user KB (bidirectional)
- Content-sync: External sources → shared KBs (one-way)

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-28-three-way-workspace-sync.md` - Current canonical plan for sync architecture
- `thoughts/shared/research/2026-01-28-openwebui-content-sync-integration-feasibility.md` - Feasibility study for integrating the Go tool
- `thoughts/shared/plans/2026-02-09-vps-deployment.md` - VPS deployment explicitly defers content-sync
- `thoughts/shared/plans/2026-01-11-bidirectional-realtime-sync.md` - Earlier Letta-era bidirectional plan (partially superseded)
- `thoughts/shared/plans/2026-01-10-file-knowledge-sync.md` - Original one-way sync plan (superseded)
- `thoughts/shared/research/2026-01-11-openwebui-letta-folder-visibility.md` - Research on folder visibility
- `thoughts/shared/research/2026-01-10-documentation-verification-file-sync.md` - Documentation verification for sync
- `thoughts/shared/research/2026-01-14-workspace-knowledge-ui-patterns.md` - UI patterns for workspace

## Open Questions

1. **Background scheduler**: The `RALPH_SYNC_INTERVAL` config exists but no background task uses it. Should sync be triggered automatically after agent file operations, on a timer, or both?
2. **RAG query tools**: Ralph agents currently cannot query OpenWebUI knowledge bases. The proposed `KnowledgeQueryTools` hasn't been built. Is this still planned?
3. **openwebui-content-sync deployment**: When will this be added to the VPS? Will it use the local folders adapter to sync Ralph workspaces, or will it remain separate?
4. **youlab-sync distribution**: How will the local daemon be distributed to students? Binary releases? Homebrew? Direct download?
5. **Two sync systems overlap**: Both the Ralph sync module and openwebui-content-sync's local folders adapter can sync folders to OpenWebUI KBs. Is one intended to replace the other, or do they serve distinct roles?
