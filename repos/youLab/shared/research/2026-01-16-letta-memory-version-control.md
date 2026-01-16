---
date: 2026-01-16T12:56:21-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Letta Memory Blocks Version Control"
tags: [research, letta, memory-blocks, version-control, git]
status: complete
last_updated: 2026-01-16
last_updated_by: ariasulin
---

# Research: Letta Memory Blocks Version Control

**Date**: 2026-01-16T12:56:21 PST
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Does Letta and its memory blocks system have native version control? How does it work?

## Summary

**Letta does NOT have native version control for individual memory blocks.** Memory blocks use **last-write-wins semantics** with no built-in mechanism to track changes, create snapshots, or restore previous versions. However, Letta provides **agent-level checkpointing** through the Agent File (.af) format and automatic state persistence at each agent step.

YouLab implements **application-layer version control** using git-backed storage (`storage/git.py`) that tracks every memory block edit with full commit history, enabling undo/restore functionality.

## Detailed Findings

### Letta Native Capabilities

#### What Letta Does NOT Provide

| Feature | Status |
|---------|--------|
| Memory block diff/history | Not available |
| Per-block rollback | Not available |
| Automatic versioning | Not available |
| Change tracking | Only `last_updated_by_id` field |

Key limitations from Letta documentation:
- "Setting `value` completely replaces the entire block content -- it is not an append operation"
- "If multiple processes (agents or external scripts) modify the same block concurrently, **the last write wins** and overwrites all earlier changes"

#### What Letta DOES Provide

**1. Agent-Level Checkpointing**
- "At each agent step (i.e. iteration of the loop), the state of the agent is checkpointed and persisted to the database"
- Agent state persists to Postgres or SQLite

**2. Agent File (.af) Format**
- Complete state serialization including memory blocks, message history, tools, and LLM settings
- Manual checkpointing at the agent level
- Import/export for moving agents between Letta servers

```python
# Export agent state
schema = client.agents.export_file(agent_id="<AGENT_ID>")

# Import agent from file
imported_agent = client.agents.import_file(file=open("/path/to/agent/file.af", "rb"))
```

**3. Recall Memory (Conversation History)**
- Automatically logs all conversational history
- Provides `conversation_search` and `conversation_search_date` tools
- Separate from memory blocks - this is message history only

### YouLab's Application-Layer Version Control

YouLab implements git-backed storage for memory blocks that provides full version control.

#### Architecture

```
.data/users/{user_id}/
  .git/                    # Git repository
  memory-blocks/
    student.md             # Memory block files
    journey.md
  pending_diffs/
    {diff_id}.json         # Agent-proposed edits awaiting approval
```

#### GitUserStorage (`storage/git.py`)

Every block edit creates a git commit with full version history:

- **Commit metadata** (`git.py:288-290`):
  ```
  {message}

  Author: {author}
  ```
  Author can be "user", "system", or "agent:{agent_id}"

- **History retrieval** (`git.py:324-335`): Uses GitPython's `repo.iter_commits(paths=file, max_count=limit)`

- **Version snapshots**: Any commit SHA can retrieve historical state

- **Restore operation**: Creates new commit with old content (preserves history, doesn't rewrite)

#### Key Operations

| Operation | Git Implementation |
|-----------|-------------------|
| Write block | Stage file + commit with author metadata |
| Get history | `repo.iter_commits(paths=rel_path)` |
| Get version | `commit.tree / rel_path` |
| Restore | Write old content as new commit |

#### API Endpoints (`server/blocks.py`)

```
GET  /users/{user_id}/blocks/{label}/history     # Get version history
GET  /users/{user_id}/blocks/{label}/versions/{sha}  # Get specific version
POST /users/{user_id}/blocks/{label}/restore     # Restore previous version
```

### Pending Diffs System

YouLab adds human-in-the-loop approval for agent-proposed memory edits:

1. Agent calls `edit_memory_block` tool
2. Tool creates `PendingDiff` (does NOT apply immediately)
3. User reviews via UI/API
4. On approval: applies edit, creates git commit, syncs to Letta
5. On rejection: marks diff as rejected, no changes

```python
# Agent proposes edit
edit_memory_block(
    block="student",
    field="goals",
    content="Get into Stanford",
    strategy="append",
    reasoning="Student mentioned Stanford 3 times"
)

# Creates pending diff - requires user approval before applying
```

### Data Flow: Memory Block Update

```
Agent Tool Call
      │
      ▼
PendingDiff Created (pending_diffs/{diff_id}.json)
      │
      ▼
User Reviews via API/UI
      │
      ├── Approve ─────────────────────────────────────┐
      │                                                │
      │   ┌────────────────────────────────────────────▼
      │   │ 1. Read current block content
      │   │ 2. Apply operation (append/replace)
      │   │ 3. Write to git with commit
      │   │ 4. Sync to Letta via API
      │   │ 5. Mark diff approved
      │   └────────────────────────────────────────────
      │
      └── Reject ──► Mark diff rejected, no changes
```

## Code References

- `src/youlab_server/storage/git.py:89-475` - GitUserStorage implementation
- `src/youlab_server/storage/blocks.py:19-364` - UserBlockManager bridging git and Letta
- `src/youlab_server/storage/diffs.py:1-144` - PendingDiff system
- `src/youlab_server/server/blocks.py:110-279` - REST API endpoints
- `src/youlab_server/tools/memory.py:27-108` - edit_memory_block tool

## Letta API Reference

Memory block operations:
- `client.blocks.list()` - List all blocks
- `client.blocks.create(label, name, value, description)` - Create block
- `client.blocks.update(block_id, value)` - Update block (last-write-wins)
- `client.agents.blocks.list(agent_id)` - List agent's blocks
- `client.agents.blocks.update(agent_id, block_id, value)` - Update agent's block

Agent checkpointing:
- `client.agents.export_file(agent_id)` - Export to .af file
- `client.agents.import_file(file)` - Import from .af file

## Historical Context (from thoughts/)

Extensive documentation exists on the memory system evolution:

- `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Core memory philosophy
- `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md` - Memory Blocks as Notes architecture
- `thoughts/shared/plans/2026-01-14-ARI-82-phase3-git-storage-adapter.md` - Git storage implementation plan
- `thoughts/shared/research/2026-01-13-ARI-82-user-storage-versioning.md` - User storage versioning design
- `thoughts/global/shared/reference/letta-archival-memory.md` - Letta archival memory reference

## Key Takeaways

1. **Letta is not a version control system** - it stores current state only
2. **YouLab adds version control** at the application layer using git
3. **Human-in-the-loop** via pending diffs ensures agent edits are reviewed
4. **Three-tier storage model**: Letta (runtime) ← Sync ← Git (source of truth) ← Pending Diffs (proposals)
5. **Agent File format** provides agent-level snapshots but not block-level history

## Sources

- [Memory blocks | Letta Docs](https://docs.letta.com/guides/agents/memory-blocks/)
- [Memory overview | Letta Docs](https://docs.letta.com/guides/agents/memory/)
- [AgentFile (.af) | Letta Docs](https://docs.letta.com/guides/agents/agent-file/)
- [Understanding memory management | Letta Docs](https://docs.letta.com/advanced/memory-management/)
- [API Reference | Letta Docs](https://docs.letta.com/api/)
