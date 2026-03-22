---
date: 2026-01-16T11:45:58-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Evolution of specs across YouLab and OpenWebUI since Jan 11, 2026"
tags: [research, codebase, specs, evolution, memory-system, module-metadata]
status: complete
last_updated: 2026-01-16
last_updated_by: ariasulin
---

# Research: Evolution of Specs in YouLab and OpenWebUI

**Date**: 2026-01-16T11:45:58-08:00
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Analyze recent commits in both YouLab and OpenWebUI repos to understand the evolution of specs. "Recent" defined as commits made since the first commit to OpenWebUI (2026-01-11).

## Summary

From **January 11-16, 2026**, the YouLab platform evolved through **9 commits** (5 in YouLab, 4 in OpenWebUI) that established two major spec systems:

1. **Module Metadata Schema** - Frontend navigation spec for course modules
2. **Memory System** - Git-backed storage with diff approval workflow

The evolution progressed from basic UI scaffolding (branding, sidebar) through backend infrastructure (git storage, API endpoints) to full user-facing features (diff approval UI, rich text editing).

## Commit Timeline

### YouLab Backend (5 commits)

| Date | SHA | Description |
|------|-----|-------------|
| Jan 12 10:27 | `4c099b8` | CodeRabbit and Semgrep configuration |
| Jan 12 10:34 | `47f59dd` | CodeRabbit review feedback fixes |
| Jan 12 12:51 | `9c5639a` | Module metadata schema documentation |
| Jan 12 12:56 | `7e5649b` | Memory System MVP (Phases 1-5) |
| Jan 15 18:59 | `2611ba2` | Diff approval backend, storage refactor |

### OpenWebUI Frontend (4 commits)

| Date | SHA | Description |
|------|-----|-------------|
| Jan 11 17:25 | `22c9c3b` | Files menu item in sidebar |
| Jan 12 12:50 | `81c10f9` | YouLab branding, theme, modules UI |
| Jan 12 14:09 | `863b405` | You page, memory API client, agents tab |
| Jan 15 18:59 | `b3e763d` | Diff approval UI, block editor, notes API |

## Detailed Findings

### 1. Module Metadata Schema

**Introduced**: `81c10f9` (Jan 12) and documented in `9c5639a`

**TypeScript Interface** (`OpenWebUI/open-webui/src/lib/apis/index.ts:1666-1685`):

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

export interface ModelMeta {
  toolIds: never[];
  description?: string;
  capabilities?: object;
  profile_image_url?: string;
  youlab_module?: YouLabModuleMeta;
}
```

**Frontend Components**:
- `ModuleList.svelte` - Filters models with `youlab_module` metadata, sorts by `module_index`
- `ModuleItem.svelte` - Renders status icons (lock, clock, checkmark), handles folder creation

**Data Flow**:
1. Models loaded from OpenWebUI backend
2. Filtered by `youlab_module` presence
3. Sorted by `module_index`
4. Click creates/retrieves folder, navigates to existing thread or new chat

**Current Limitations** (Phase A):
- Status is global (stored in model metadata, not per-user)
- `unlock_criteria` defined but not enforced
- `welcome_message` not implemented in chat initialization

### 2. Memory System Spec

**Introduced**: `7e5649b` (Jan 12) - Backend MVP
**Enhanced**: `2611ba2` (Jan 15) - Diff approval, storage refactor

#### Storage Layer

**Directory Structure**:
```
.data/users/{user_id}/
    .git/                    # Git version control
    memory-blocks/           # Markdown files with YAML frontmatter
        student.md
        journey.md
    pending_diffs/           # Agent-proposed changes
        {diff_id}.json
```

**Core Classes**:

| Class | Location | Purpose |
|-------|----------|---------|
| `GitUserStorage` | `storage/git.py:89-441` | Git-backed file storage per user |
| `GitUserStorageManager` | `storage/git.py:443-475` | Factory with caching |
| `UserBlockManager` | `storage/blocks.py:19-364` | Orchestrates blocks, Letta sync, diffs |
| `PendingDiffStore` | `storage/diffs.py:71-144` | JSON-based diff persistence |

**PendingDiff Schema** (`storage/diffs.py:19-68`):

```python
@dataclass
class PendingDiff:
    id: str                 # UUID
    user_id: str
    agent_id: str
    block_label: str
    field: str | None
    operation: str          # append, replace, llm_diff, full_replace
    current_value: str
    proposed_value: str
    reasoning: str
    confidence: str         # low, medium, high
    source_query: str | None
    status: str             # pending, approved, rejected, superseded, expired
    created_at: str         # ISO timestamp
    reviewed_at: str | None
    applied_commit: str | None  # Git SHA
```

#### API Endpoints

**Blocks Router** (`/users/{user_id}/blocks`):

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/` | List blocks with pending diff counts |
| GET | `/{label}` | Get block detail (markdown, body, metadata) |
| PUT | `/{label}` | Update block (creates git commit) |
| GET | `/{label}/history` | Version history |
| GET | `/{label}/versions/{sha}` | Content at specific version |
| POST | `/{label}/restore` | Restore to previous version |
| GET | `/{label}/diffs` | List pending diffs |
| POST | `/{label}/diffs/{id}/approve` | Apply diff |
| POST | `/{label}/diffs/{id}/reject` | Reject diff |

**Notes API Adapter** (`/api/you/notes/*`):
- Translates between OpenWebUI's Notes API and git storage
- Enables TipTap editor compatibility
- Converts YAML frontmatter to/from note structure

#### Frontend Memory Components

**Introduced**: `863b405` (Jan 12) - Basic UI
**Enhanced**: `b3e763d` (Jan 15) - Diff approval, editors

| Component | Purpose |
|-----------|---------|
| `BlockCard.svelte` | Clickable card for block navigation |
| `BlockDetailModal.svelte` | View/edit block with unified diff view |
| `BlockEditor.svelte` | Rich editor with autosave, AI enhancement |
| `MemoryBlockEditor.svelte` | Notes API-based editor with git undo/redo |
| `DiffApprovalOverlay.svelte` | Full-screen diff review with keyboard nav |
| `DiffBadge.svelte` | Pending diff count badge |
| `AgentsTab.svelte` | Background agent threads display |

**Memory API Client** (`apis/memory/index.ts`):

```typescript
// Block operations
getBlocks(userId, token): Promise<MemoryBlock[]>
getBlock(userId, label, token): Promise<BlockDetail>
updateBlock(userId, label, content, token, format, message?): Promise<{commitSha, label}>

// Version control
getBlockHistory(userId, label, token, limit?): Promise<VersionInfo[]>
getBlockVersion(userId, label, sha, token): Promise<{content, sha}>
restoreVersion(userId, label, sha, token): Promise<{commitSha, label}>

// Diff approval
getBlockDiffs(userId, label, token): Promise<PendingDiff[]>
approveDiff(userId, label, diffId, token): Promise<{diffId, commitSha}>
rejectDiff(userId, label, diffId, token, reason?): Promise<{status, diffId}>
```

**Keyboard Shortcuts** (DiffApprovalOverlay):
- `A` - Approve current diff
- `R` - Reject current diff
- `Arrow Left/Up` - Previous diff
- `Arrow Right/Down` - Next diff
- `Escape/E` - Dismiss overlay

### 3. Storage Layer Refactoring

**Commit**: `2611ba2` (Jan 15)

The final commit consolidated the storage layer:

| Before | After |
|--------|-------|
| Separate `convert.py` for TOML<->MD | Integrated into `blocks.py` and `git.py` |
| Tests in `test_convert.py` | Removed (260 lines deleted) |
| `full_replace` operation missing | Added to supported operations |

**New Files**:
- `notes_adapter.py` - Notes API compatibility layer
- `agents_threads.py` - Background agent thread history
- `config/courses/default/course.toml` - Default course configuration

## Architecture Evolution

```
Phase 1 (Jan 11-12): UI Scaffolding
├── YouLab branding and theming
├── Module navigation components
└── Basic memory page structure

Phase 2 (Jan 12): Backend Infrastructure
├── Git storage layer
├── Block CRUD API
├── User initialization webhook
└── PendingDiff system

Phase 3 (Jan 15): Full Integration
├── Diff approval UI components
├── Rich text block editors
├── Notes API adapter
└── Background agent integration
```

## Code References

### Module Metadata
- `OpenWebUI/open-webui/src/lib/apis/index.ts:1666-1685` - TypeScript interfaces
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleList.svelte:55-67` - Filtering logic
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ModuleItem.svelte:21-26` - Status styling
- `docs/module-metadata-schema.md` - Full schema documentation

### Memory System
- `src/youlab_server/storage/git.py:89-441` - GitUserStorage implementation
- `src/youlab_server/storage/blocks.py:19-364` - UserBlockManager
- `src/youlab_server/storage/diffs.py:19-68` - PendingDiff dataclass
- `src/youlab_server/server/blocks.py` - REST API endpoints
- `src/youlab_server/server/notes_adapter.py` - Notes API adapter
- `OpenWebUI/open-webui/src/lib/apis/memory/index.ts` - Memory API client
- `OpenWebUI/open-webui/src/lib/components/you/*.svelte` - UI components

## Key Design Decisions

1. **Git as Version Control**: Every memory block edit creates a git commit, providing full history with restore capability

2. **Pending Diff Workflow**: Agent edits don't apply immediately - they require user approval through the UI

3. **Dual Content Formats**: TOML for structured diff operations, Markdown for human editing

4. **Notes API Compatibility**: Reuses OpenWebUI's TipTap editor by implementing Notes-compatible endpoints

5. **Frontend State Management**: Svelte stores (`memoryBlocks`, `selectedBlock`, `blockHistory`) with derived `pendingDiffsCount`

6. **Supersession Logic**: Approving a diff supersedes older pending diffs for the same block

## Open Questions

1. **Per-user module status**: Currently global - Phase B will add per-user progression via `journey` memory block

2. **Letta sync reliability**: The `_sync_block_to_letta()` method logs errors but doesn't fail operations

3. **Notes API bug**: `MemoryBlockEditor.svelte:67` documents that refreshing page auto-rejects pending diffs

4. **Background agent discovery**: `agents_threads.py` endpoint exists but agent discovery mechanism not fully documented

## Related Research

- `thoughts/shared/research/2026-01-12-ARI-78-openwebui-model-metadata-extension.md` - Earlier model metadata research
- `thoughts/shared/research/2026-01-12-ARI-78-unified-user-memory-blocks.md` - Memory blocks design
- `thoughts/shared/research/2026-01-12-ARI-81-phase-4-diff-approval-analysis.md` - Diff approval design
- `thoughts/shared/research/2026-01-13-ARI-82-user-storage-versioning.md` - Version control design
