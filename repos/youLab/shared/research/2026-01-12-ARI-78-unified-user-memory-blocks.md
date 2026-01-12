---
date: 2026-01-12T03:14:08+0000
researcher: Claude (Opus 4.5)
git_commit: 5921af2
branch: main
repository: YouLab
topic: "Unified User-Level Memory Blocks Implementation in Letta"
tags: [research, memory, letta, blocks, architecture, ARI-78]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude (Opus 4.5)
linear_ticket: ARI-78
---

# Research: Unified User-Level Memory Blocks in Letta

**Date**: 2026-01-12T03:14:08+0000
**Researcher**: Claude (Opus 4.5)
**Git Commit**: 5921af2
**Branch**: main
**Repository**: YouLab

## Research Question

How to implement unified user-level memory blocks in Letta that are:
- Shared across all courses/modules for a single user
- NOT per-agent (current system has per-agent blocks)
- Accessible by both fast agents (tutors) and slow agents (background)

## Summary

**Letta fully supports the unified memory architecture** described in the product spec. The core capability exists: multiple Letta agents can reference the same block instance via `block_ids`, and when one agent modifies it, all others see the change immediately. The implementation challenge is in YouLab's current codebase, which has several deprecated patterns that need updating:

1. **Current shared blocks are course-scoped, not user-scoped** (`agents.py:54`)
2. **Memory tools only support hardcoded "human"/"persona" blocks** (`memory.py:140-152`, `enricher.py:97-108`)
3. **Both tools use deprecated Letta APIs** (`get_agent_memory`, `update_agent_core_memory`)

The path forward requires: (1) changing the shared block naming convention from course-level to user-level, (2) updating memory tools to use the modern blocks API with dynamic label lookup, and (3) implementing a git-backed persistence layer for TOML storage.

---

## Detailed Findings

### 1. Letta's Native Shared Block Capabilities

**Letta supports true cross-agent shared memory.** Key API patterns:

| Operation | API |
|-----------|-----|
| Create shared block | `client.blocks.create(label=..., value=..., name=...)` |
| Attach to agent | `client.agents.create(..., block_ids=[block.id])` |
| Attach dynamically | `client.agents.blocks.attach(agent_id=..., block_id=...)` |
| Detach | `client.agents.blocks.detach(agent_id=..., block_id=...)` |
| Update block | `client.blocks.update(block_id=..., value=...)` |
| List blocks by agent | `client.agents.blocks.list(agent_id=...)` |
| List agents by block | `client.blocks.agents.list(block_id=...)` |

**Sharing behavior:**
- Blocks are stored once in the database with a unique `block_id`
- Multiple agents reference the same record
- Updates are **immediately visible** to all attached agents on their next context load
- **Last write wins** - no built-in conflict resolution
- `read_only=True` prevents agent modifications (useful for policies)

**Sources:**
- [Multi-Agent Shared Memory | Letta Docs](https://docs.letta.com/guides/agents/multi-agent-shared-memory)
- [Memory Blocks Guide | Letta Docs](https://docs.letta.com/guides/agents/memory-blocks/)
- [Shared Memory Blocks Tutorial | Letta Docs](https://docs.letta.com/tutorials/shared-memory-blocks)

---

### 2. Current Shared Block Implementation in YouLab

**Location**: `src/youlab_server/server/agents.py:57-116`

The current implementation creates shared blocks keyed by `(course_id, block_label)`:

```python
# agents.py:53-55
def _shared_block_name(self, course_id: str, block_label: str) -> str:
    """Generate unique name for a shared block."""
    return f"youlab_shared_{course_id}_{block_label}"
```

**Current flow** (agents.py:273-305):
1. Course config defines `shared = true` in block schema
2. `_get_or_create_shared_block()` creates/retrieves block by course+label
3. Block ID added to `shared_block_ids` list
4. Agent created with `block_ids=shared_block_ids` parameter

**Key limitation**: Sharing is at **course level**, not **user level**. All users of the same course share the same block, which is NOT the desired behavior for unified user memory.

---

### 3. Required Architecture Change: User-Level Blocks

**Current naming**: `youlab_shared_{course_id}_{block_label}`
**Required naming**: `youlab_user_{user_id}_{block_label}`

The product spec requires:
> "Memory blocks are a **unified architecture of the student** that all courses and modules use. They are NOT per-course or per-agent."

**Implementation approach:**

```python
# Proposed: User-level shared block naming
def _user_block_name(self, user_id: str, block_label: str) -> str:
    return f"youlab_user_{user_id}_{block_label}"

def _get_or_create_user_block(self, user_id: str, block_label: str, ...) -> str:
    cache_key = (user_id, block_label)  # Changed from (course_id, block_label)
    # ... rest of implementation
```

**Block ownership model:**
- Each user has ONE set of unified memory blocks
- All agents for that user (across all courses) attach to the same blocks
- Background agents also attach to and modify these blocks

---

### 4. Current Memory Tool Limitations

#### 4.1 edit_memory_block Tool (`tools/memory.py:36-228`)

**Problem 1: Hardcoded block names** (lines 140-152):
```python
if block == "human":
    # ...
if block == "persona":
    # ...
return f"Unknown block '{block}'. Use 'human' or 'persona'."
```

Cannot edit custom blocks like "operating_manual", "values", "strengths", etc.

**Problem 2: Deprecated API** (via `MemoryManager`):
- `client.get_agent_memory(agent_id)` - returns dict with "persona"/"human" keys
- `client.update_agent_core_memory(agent_id, persona=..., human=...)` - only these labels

#### 4.2 MemoryEnricher for Background Agents (`memory/enricher.py:67-153`)

**Same limitations** (lines 97-108):
- Only supports "human" and "persona" blocks
- Uses same deprecated `MemoryManager` pattern

---

### 5. Modern Block Access Pattern

The `advance_lesson` tool in `tools/curriculum.py:307-314` demonstrates the correct pattern:

```python
# Find block by label using modern API
blocks = _letta_client.agents.blocks.list(agent_id=agent_id)
for block in blocks:
    if getattr(block, "label", None) == "journey":
        new_value = _serialize_block_value(data)
        _letta_client.agents.blocks.update(
            agent_id=agent_id,
            block_id=block.id,
            value=new_value,
        )
```

**This pattern works with ANY block label**, not just hardcoded ones.

---

### 6. Proposed Implementation for Unified Memory

#### 6.1 User Block Manager (New)

Create a new manager that handles user-level blocks:

```python
class UserBlockManager:
    """Manages unified memory blocks per user."""

    def __init__(self, letta_client, user_id: str):
        self.client = letta_client
        self.user_id = user_id
        self._block_cache: dict[str, str] = {}  # label -> block_id

    def get_or_create_block(self, label: str, schema: BlockSchema) -> str:
        """Get or create a user-level block by label."""
        block_name = f"youlab_user_{self.user_id}_{label}"

        # Check cache
        if label in self._block_cache:
            return self._block_cache[label]

        # Search existing blocks
        blocks = self.client.blocks.list(name=block_name)
        for block in blocks:
            if block.name == block_name:
                self._block_cache[label] = block.id
                return block.id

        # Create new block from schema
        model_class = create_block_model(label, schema)
        instance = model_class()

        block = self.client.blocks.create(
            label=label,
            name=block_name,
            value=instance.to_memory_string(),
            description=schema.description,
        )

        self._block_cache[label] = block.id
        return block.id

    def attach_to_agent(self, agent_id: str, labels: list[str]) -> None:
        """Attach user blocks to an agent."""
        for label in labels:
            block_id = self._block_cache.get(label)
            if block_id:
                self.client.agents.blocks.attach(
                    agent_id=agent_id,
                    block_id=block_id,
                )
```

#### 6.2 Updated Agent Creation Flow

```python
async def create_agent_from_curriculum(self, user_id: str, course_id: str, ...):
    # 1. Get or create unified user blocks (shared across all courses)
    user_block_mgr = UserBlockManager(self.client, user_id)
    unified_block_ids = []
    for label in DEFAULT_UNIFIED_BLOCKS:  # operating_manual, values, etc.
        block_id = user_block_mgr.get_or_create_block(label, UNIFIED_SCHEMA[label])
        unified_block_ids.append(block_id)

    # 2. Get or create course-specific blocks (e.g., journey)
    course_block_ids = []
    for name, schema in course.blocks.items():
        if schema.shared and not schema.unified:  # course-level shared
            block_id = self._get_or_create_shared_block(course_id, schema.label, ...)
            course_block_ids.append(block_id)

    # 3. Create agent with all block_ids
    agent = self.client.agents.create(
        name=self._agent_name(user_id, course_id),
        memory_blocks=per_agent_blocks,
        block_ids=unified_block_ids + course_block_ids,
        ...
    )
```

#### 6.3 Updated Memory Tool

```python
def edit_memory_block(
    block: str,        # Any block label
    field: str,        # Field within block
    content: str,
    strategy: str = "append",
    agent_state: dict | None = None,
) -> str:
    agent_id = agent_state.get("agent_id")

    # Find block by label using modern API
    blocks = _letta_client.agents.blocks.list(agent_id=agent_id)
    target_block = None
    for b in blocks:
        if getattr(b, "label", None) == block:
            target_block = b
            break

    if not target_block:
        return f"Block '{block}' not found."

    # Load block registry for this agent's course
    course_id = agent_state.get("metadata", {}).get("course_id")
    registry = curriculum.get_block_registry(course_id)

    # Deserialize, modify, serialize
    BlockModel = registry.get(block) or DynamicBlock
    instance = BlockModel.from_memory_string(target_block.value)

    # Apply field update based on strategy
    _update_field(instance, field, content, strategy)

    # Write back
    _letta_client.agents.blocks.update(
        agent_id=agent_id,
        block_id=target_block.id,
        value=instance.to_memory_string(),
    )

    return f"Updated {block}.{field}"
```

---

### 7. Git-Backed Storage Layer

The product spec requires TOML storage with git version control:

```
/data/users/{user_id}/
├── .git/
├── memory/
│   ├── operating-manual.toml
│   ├── values.toml
│   ├── personality.toml
│   └── ...
└── config/
    └── schema.toml
```

**Integration approach:**
1. Primary storage: Letta blocks (for real-time agent access)
2. Secondary storage: Git-backed TOML (for version control, diffs, rollbacks)
3. Sync mechanism: Write-through on approved changes

**Sync triggers:**
- User approves background agent diff → write to git, commit
- User manually edits → write to git, commit
- Rollback requested → revert git, sync to Letta

---

### 8. Background Agent Access to Unified Memory

Background agents already access memory via `MemoryEnricher`. Updates needed:

1. **Use UserBlockManager** to locate user's unified blocks
2. **Support arbitrary labels** (not just "human"/"persona")
3. **Propose diffs** instead of direct writes (per product spec)

**Diff proposal flow:**
1. Background agent queries Honcho for insights
2. Generates proposed diff (new value vs current value)
3. Writes diff to pending storage (not applied yet)
4. User sees diff badge in "You" section
5. User approves → apply to Letta block + git commit
6. User rejects → discard diff

---

## Code References

| File | Line | Description |
|------|------|-------------|
| `server/agents.py` | 57-116 | Current `_get_or_create_shared_block()` implementation |
| `server/agents.py` | 273-305 | Block creation during agent setup |
| `server/agents.py` | 324-338 | Agent creation with `block_ids` parameter |
| `tools/memory.py` | 36-228 | `edit_memory_block` tool (hardcoded blocks) |
| `tools/memory.py` | 140-152 | Block name switch statement (limitation) |
| `memory/manager.py` | 91-127 | Deprecated `get_agent_memory` usage |
| `memory/enricher.py` | 67-153 | Background agent memory enrichment |
| `memory/enricher.py` | 97-108 | Hardcoded block support |
| `tools/curriculum.py` | 307-314 | Modern block access pattern (correct) |
| `curriculum/schema.py` | 208-214 | `BlockSchema` with `shared` attribute |
| `curriculum/blocks.py` | 119-168 | Dynamic block model generation |

---

## Architecture Documentation

### Current Architecture

```
User → Agent Creation → Per-agent blocks + Course-shared blocks
                                    ↓
             [course_id, block_label] cache in AgentManager
                                    ↓
                        Block attached via block_ids
                                    ↓
Agent Tool (edit_memory_block) → Deprecated API → Only "human"/"persona"
                                    ↓
Background Agent → MemoryEnricher → Same deprecated API
```

### Proposed Architecture

```
User → UserBlockManager → User-level unified blocks
                               ↓
              [user_id, block_label] block registry
                               ↓
        All user's agents attach same unified blocks
                               ↓
Agent Tool (edit_memory_block) → Modern blocks API → Any label
                               ↓
Background Agent → Diff Proposal → Pending approval queue
                               ↓
User Approval → Git commit + Letta block update
```

---

## Open Questions

1. **Conflict resolution**: How to handle concurrent writes from multiple agents to the same unified block?
   - Option A: Last write wins (current Letta behavior)
   - Option B: Optimistic locking with version check
   - Option C: Queue writes and apply sequentially

2. **Block migration**: How to migrate existing per-agent blocks to unified user blocks?
   - One-time migration script needed
   - Merge strategy for conflicting data

3. **Performance**: Will listing all blocks for label lookup impact latency?
   - Consider caching block_id by label per agent
   - Consider metadata-based filtering if Letta supports it

4. **Schema evolution**: How to handle schema changes to unified blocks?
   - Add new fields: fill with defaults
   - Remove fields: preserve in archival?
   - Type changes: migration strategy

---

## Related Research

- `thoughts/shared/research/2026-01-10-curriculum-agent-architecture.md` - Agent creation patterns
- `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Memory design principles
- `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Full product specification

---

## Implementation Phases

### Phase 1: User Block Infrastructure
- [ ] Create `UserBlockManager` class
- [ ] Define `DEFAULT_UNIFIED_BLOCKS` schema
- [ ] Update `AgentManager.create_agent_from_curriculum()` to use user blocks
- [ ] Add migration script for existing users

### Phase 2: Tool Updates
- [ ] Update `edit_memory_block` to use modern blocks API
- [ ] Support arbitrary block labels
- [ ] Update `MemoryEnricher` for background agents
- [ ] Deprecate `MemoryManager`

### Phase 3: Git Integration
- [ ] Implement git-backed TOML storage
- [ ] Sync mechanism between Letta and git
- [ ] Version history retrieval

### Phase 4: Diff Approval Workflow
- [ ] Pending diff storage
- [ ] Approval/rejection API
- [ ] UI integration for "You" section
