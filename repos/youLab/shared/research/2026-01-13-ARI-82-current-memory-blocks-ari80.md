---
date: 2026-01-13T16:53:06+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "Current Memory Blocks Implementation from ARI-80"
tags: [research, codebase, memory-blocks, ari-80, ari-82, git-storage, openwebui]
status: complete
last_updated: 2026-01-13
last_updated_by: ariasulin
---

# Research: Current Memory Blocks Implementation (ARI-80)

**Date**: 2026-01-13T16:53:06+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Document the current memory blocks implementation from ARI-80 and identify what can be reused/refactored for the new Notes-based architecture (ARI-82).

## Summary

ARI-80 implemented a complete memory blocks system with:
1. **Backend**: Git-backed per-user storage with version history, TOML↔MD conversion, and Letta sync
2. **API**: Full CRUD endpoints with diff approval workflow
3. **Frontend**: Custom "You" section with BlockCard/BlockDetailModal components

The new spec (ARI-82) wants memory blocks to reuse OpenWebUI's existing Notes components. This requires bridging the gap between:
- **Memory blocks**: TOML source, git-backed versioning, agent diff approval
- **Notes**: JSON/HTML/MD, in-memory versions array, rich editor with collaboration

## Detailed Findings

### 1. Backend Storage Layer

#### GitUserStorage (`src/youlab_server/storage/git.py:41-360`)

Git-backed storage for a single user.

**Directory structure:**
```
.data/users/{user_id}/
    .git/
    blocks/
        student.toml
        engagement_strategy.toml
        journey.toml
    pending_diffs/
        {diff_id}.json
    agent_threads/
        {agent_name}/
            {chat_id}.json
```

**Key methods:**
- `read_block(label)` - Read TOML file
- `write_block(label, content, message, author)` - Write and commit
- `get_block_history(label, limit)` - Get version history from git log
- `get_block_at_version(label, commit_sha)` - Read file at specific commit
- `restore_block(label, commit_sha)` - Restore previous version
- `diff_versions(label, old_sha, new_sha)` - Compare two versions

**GitUserStorageManager** (`src/youlab_server/storage/git.py:328-360`):
- Factory class that creates/caches GitUserStorage instances
- Manages base directory for all users

#### UserBlockManager (`src/youlab_server/storage/blocks.py:21-395`)

User-scoped block operations with Letta sync and pending diffs.

**Responsibilities:**
- Read/write blocks from git storage
- Convert between TOML and Markdown
- Sync blocks to Letta agents
- Manage pending diffs for agent edits

**Key methods:**
- `get_block_toml(label)` / `get_block_markdown(label)` - Dual format access
- `update_block_from_markdown(label, markdown, message)` - User edits
- `update_block_from_toml(label, toml_content, message, author)` - Direct TOML update
- `get_history(label, limit)` / `restore_version(label, commit_sha)` - Version management
- `_sync_block_to_letta(label, toml_content)` - Sync to Letta blocks API
- `propose_edit(agent_id, block_label, ...)` - Create pending diff
- `approve_diff(diff_id)` / `reject_diff(diff_id)` - Diff workflow

#### TOML↔Markdown Conversion (`src/youlab_server/storage/convert.py:1-148`)

Bidirectional conversion for editing:

**TOML → Markdown:**
```markdown
---
block: student
---

## Profile
Background, CliftonStrengths, aspirations...

## Insights
Key observations from conversation...
```

**Markdown → TOML:**
- Parses frontmatter for metadata
- Converts `## Title` sections to snake_case keys
- Lists (`- item`) become arrays
- Paragraphs become string values

#### PendingDiffStore (`src/youlab_server/storage/diffs.py:71-144`)

JSON file storage for pending agent edits.

**PendingDiff dataclass:**
```python
@dataclass
class PendingDiff:
    id: str
    user_id: str
    agent_id: str
    block_label: str
    field: str | None
    operation: Literal["append", "replace", "llm_diff"]
    current_value: str
    proposed_value: str
    reasoning: str
    confidence: Literal["low", "medium", "high"]
    source_query: str | None
    status: Literal["pending", "approved", "rejected", "superseded", "expired"]
    created_at: str
    reviewed_at: str | None
    applied_commit: str | None
```

**Operations:**
- `append` - Add to existing content
- `replace` - Replace specific text
- `full_replace` - Replace entire block

### 2. Server API (`src/youlab_server/server/blocks.py:1-274`)

REST endpoints under `/users/{user_id}/blocks`:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | List all blocks with pending diff counts |
| `/diffs/counts` | GET | Get pending diff counts per block |
| `/{label}` | GET | Get block detail (TOML + MD + pending diffs) |
| `/{label}` | PUT | Update block (markdown or toml format) |
| `/{label}/history` | GET | Get version history |
| `/{label}/versions/{sha}` | GET | Get content at version |
| `/{label}/restore` | POST | Restore to previous version |
| `/{label}/diffs` | GET | List pending diffs for block |
| `/{label}/diffs/{id}/approve` | POST | Approve and apply diff |
| `/{label}/diffs/{id}/reject` | POST | Reject diff |

**Response Models:**
- `BlockSummary`: label, pending_diffs
- `BlockDetail`: label, content_toml, content_markdown, pending_diffs
- `VersionInfo`: sha, message, author, timestamp, is_current
- `DiffSummary`: id, block, field, operation, reasoning, confidence, oldValue, newValue

### 3. User Initialization (`src/youlab_server/server/users.py:57-161`)

`POST /users/init` webhook endpoint:
1. Check if user storage already exists (idempotent)
2. Initialize `GitUserStorage`
3. Load course configuration from curriculum
4. Create default blocks from course schema
5. Serialize Pydantic models to TOML

### 4. OpenWebUI Frontend - "You" Section

#### You Page (`OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte:1-210`)

Main page with two tabs:
- **Profile**: Grid of BlockCard components
- **Agents**: AgentsTab component

Features:
- Loads blocks via `getBlocks($user.id, token)`
- Mock data fallback when API unavailable
- Opens BlockDetailModal on card click

#### BlockCard (`OpenWebUI/open-webui/src/lib/components/you/BlockCard.svelte:1-43`)

Simple card showing:
- "Memory Block" badge
- Block label (snake_case → Title Case)
- Pending diffs count badge

#### BlockDetailModal (`OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte:1-385`)

Modal with:
- **View mode**: Unified diff view showing deletions (red) and additions (green)
- **Edit mode**: Simple Textarea for markdown editing
- **History sidebar**: Version list with restore buttons
- **Diff actions**: Approve/Reject buttons inline with additions

Diff view logic:
1. Builds `diffLines` array from TOML content + pending diffs
2. Maps old/new values to line numbers
3. Renders context/deletion/addition lines with styling

#### Memory Store (`OpenWebUI/open-webui/src/lib/stores/memory.ts:1-46`)

Svelte stores:
```typescript
export const memoryBlocks = writable<MemoryBlock[]>([]);
export const pendingDiffsCount = derived(memoryBlocks, ...);
export const selectedBlock = writable<BlockDetail | null>(null);
export const blockHistory = writable<VersionInfo[]>([]);
```

#### Memory API Client (`OpenWebUI/open-webui/src/lib/apis/memory/index.ts:1-388`)

TypeScript functions calling YouLab API:
- `getBlocks`, `getBlock`, `updateBlock`
- `getBlockHistory`, `getBlockVersion`, `restoreVersion`
- `getBlockDiffs`, `approveDiff`, `rejectDiff`
- `getBackgroundAgents`

### 5. OpenWebUI Notes Components (Comparison Target)

#### NoteEditor (`OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte`)

Full-featured editor with:
- **RichTextInput**: TipTap/ProseMirror-based rich text editor
- **Collaboration**: WebSocket-based real-time editing
- **Versions**: In-memory `data.versions` array with undo/redo navigation
- **Files**: Image/file upload with drag-drop
- **AI**: Title generation, content enhancement via LLM
- **Controls**: Chat panel, settings panel

Note data structure:
```typescript
{
  title: '',
  data: {
    content: {
      json: null,  // TipTap JSON
      html: '',
      md: ''
    },
    versions: [],  // Array of content snapshots
    files: null
  },
  meta: null,
  access_control: {}
}
```

#### Notes List (`OpenWebUI/open-webui/src/lib/components/notes/Notes.svelte`)

List view with:
- Search functionality
- View filters (All, Created by you, Shared with you)
- Display options (List, Grid)
- Time-based grouping
- Delete, download, copy link actions

## Gap Analysis: Memory Blocks vs Notes

| Feature | Memory Blocks (Current) | Notes (OpenWebUI) | Gap |
|---------|------------------------|-------------------|-----|
| **Storage** | Git-backed TOML files | SQLite via OpenWebUI API | Different backends |
| **Versioning** | Git commit history | In-memory versions array | Git is more robust |
| **Editor** | Simple Textarea | RichTextInput (TipTap) | Notes is richer |
| **Format** | TOML source | JSON/HTML/MD | Need TOML↔JSON bridge |
| **Diff Approval** | Full workflow | None | Notes needs this |
| **Collaboration** | None | WebSocket | Blocks needs this |
| **Files** | None | Full upload support | Blocks needs this |
| **AI Enhancement** | None | Title gen, content enhance | Blocks could use |

## What Can Be Reused

### From ARI-80 (Keep):
1. **GitUserStorage** - Git-backed versioning is more robust than in-memory
2. **PendingDiffStore** - Agent edit approval workflow is core differentiator
3. **UserBlockManager** - Business logic for blocks + Letta sync
4. **Server API endpoints** - Well-structured REST API

### From Notes (Adopt):
1. **RichTextInput** - Replace Textarea with rich editor
2. **Version navigation UI** - Undo/redo buttons pattern
3. **File handling** - Image/file upload infrastructure
4. **List view patterns** - Search, filters, grid/list toggle

### Needs Integration Work:
1. **TOML↔JSON conversion** - Bridge for rich editor
2. **Diff overlay in rich editor** - Show agent proposals inline
3. **Git-backed persistence for Notes format** - Store JSON versions in git
4. **Schema validation** - Opt-in schema enforcement for structured blocks

## Architecture Decisions for ARI-82

Based on ARI-82 spec:
1. Memory blocks default to freeform unless schema specified
2. Schema enforcement is opt-in per block
3. TOML → MD round-trip must be robust (or switch to MD-first?)
4. Diffs shown inline in markdown editor for approve/deny

**Key Question**: Should memory blocks switch from TOML source to Markdown source?
- TOML is structured but harder to edit in rich editor
- Markdown is native to Notes components
- Schema enforcement would need custom validation layer

## Code References

- `src/youlab_server/storage/git.py:41-360` - GitUserStorage class
- `src/youlab_server/storage/blocks.py:21-395` - UserBlockManager class
- `src/youlab_server/storage/diffs.py:19-144` - PendingDiff + PendingDiffStore
- `src/youlab_server/storage/convert.py:12-148` - TOML↔MD conversion
- `src/youlab_server/server/blocks.py:109-274` - REST API endpoints
- `src/youlab_server/server/users.py:57-139` - User initialization
- `OpenWebUI/open-webui/src/lib/components/you/*` - You section components
- `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte` - Rich editor
- `OpenWebUI/open-webui/src/lib/stores/memory.ts` - Memory stores

## Related Research

- ARI-80: Phase B Memory System MVP (parent implementation)
- ARI-79: Module metadata schema documentation
- ARI-81: Related architecture work

## Open Questions

1. **Format decision**: Keep TOML source or switch to Markdown-first for Notes compatibility?
2. **Schema enforcement**: How to validate structured blocks in a freeform editor?
3. **Diff visualization**: How to show TOML-level diffs in a rich text editor?
4. **Migration path**: How to migrate existing TOML blocks to new format?
5. **Collaboration**: Do memory blocks need real-time collaboration like Notes?
