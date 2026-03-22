# Add `add_memory_block()` Agent Tool — Implementation Plan

## Overview

Add an `add_memory_block()` tool to `MemoryBlockTools` so the Ralph agent can create new memory blocks for a student during conversation. Currently the agent can only read and edit existing blocks — it cannot create new ones. This limits the agent's ability to organically grow the student's memory profile as new topics emerge.

## Current State Analysis

- `MemoryBlockTools` (`src/ralph/tools/memory_blocks.py:57-87`) registers 3 tools: `list_memory_blocks`, `read_memory_block`, `propose_memory_edit`
- `DoltClient.update_block()` (`src/ralph/dolt.py:153-195`) already does INSERT ... ON DUPLICATE KEY UPDATE — it supports creating new blocks natively
- New blocks are currently only created by `ensure_welcome_blocks()` (system, for new users) or the REST API `PUT /users/{user_id}/blocks/{label}`
- The proposal system (`create_proposal`) only supports editing existing block bodies (UPDATE on branch) — creating a new block on a proposal branch would silently fail

### Key Discoveries:
- `DoltClient.update_block()` already handles upserting — no DB layer changes needed (`src/ralph/dolt.py:166-174`)
- All tools use `_run_async_with_fresh_client()` to bridge sync Agno tools with async Dolt (`src/ralph/tools/memory_blocks.py:18-45`)
- User ID extraction uses `_get_user_id(run_context)` helper (`src/ralph/tools/memory_blocks.py:48-54`)
- Tool instructions in `server.py:227-258` will need updating to mention the new tool
- Tools are registered in `__init__` by adding to the `tools` list (`src/ralph/tools/memory_blocks.py:81-85`)

## Desired End State

The Ralph agent has an `add_memory_block` tool that:
1. Takes a `label`, `title`, and `body` (all required)
2. Validates the label doesn't already exist (returns error if it does — use `propose_memory_edit` instead)
3. Creates the block directly on main via `DoltClient.update_block()`
4. Returns a success message confirming creation

**Verification**: Agent can create a new block during conversation, which then appears in `list_memory_blocks` and in the student's memory context on next request.

## What We're NOT Doing

- Not adding proposal/approval flow for block creation (direct creation only — edits still require approval)
- Not adding block deletion as an agent tool
- Not adding schema_ref support (agents create free-form blocks)
- Not modifying the REST API or frontend
- Not adding constraints on label naming beyond basic validation

## Implementation Approach

Single-phase change — add one new method to `MemoryBlockTools` and update the tool instructions.

## Phase 1: Add `add_memory_block` Tool

### Overview
Add the tool method, register it, and update agent instructions.

### Changes Required:

#### 1. New tool method in `MemoryBlockTools`
**File**: `src/ralph/tools/memory_blocks.py`
**Changes**: Add `add_memory_block` method and register it in `__init__`

Register the new tool in `__init__`:
```python
tools = [
    self.list_memory_blocks,
    self.read_memory_block,
    self.propose_memory_edit,
    self.add_memory_block,
]
```

Add the method (after `list_memory_blocks`, before `read_memory_block` — keeping creation near listing):

```python
def add_memory_block(
    self,
    run_context: RunContext,
    block_label: str,
    title: str,
    body: str,
) -> str:
    """
    Create a new memory block for the current student.

    Use this when you discover a new topic, interest, or pattern worth
    tracking that doesn't fit in existing blocks. The block is created
    immediately (no approval needed). Future edits to this block will
    still require user approval via propose_memory_edit.

    Args:
        run_context: Agno run context with user_id (auto-injected).
        block_label: A short snake_case identifier (e.g., "math_journey", "writing_goals").
                     Must be unique — cannot match an existing block label.
        title: Human-readable title (e.g., "Math Journey", "Writing Goals").
        body: Initial content for the block in markdown format.

    Returns:
        Success message if created, or error message if the label already exists.

    """
    user_id = _get_user_id(run_context)
    if not user_id:
        return "Unable to identify student. No user context available."

    # Validate label format
    if not block_label.replace("_", "").isalnum():
        return (
            "Error: block_label must be alphanumeric with underscores only "
            "(e.g., 'math_journey', 'writing_goals')."
        )

    if not title.strip():
        return "Error: title cannot be empty."

    if not body.strip():
        return "Error: body cannot be empty."

    try:

        async def _create(dolt: DoltClient) -> str | None:
            # Check if block already exists
            existing = await dolt.get_block(user_id, block_label)
            if existing:
                return (
                    f"Error: Block '{block_label}' already exists. "
                    "Use propose_memory_edit to modify existing blocks."
                )

            # Create the block directly on main
            await dolt.update_block(
                user_id=user_id,
                label=block_label,
                body=body,
                title=title,
                author=self.agent_id,
                message=f"Create {block_label}: {title}",
            )
            return None  # No error

        error = _run_async_with_fresh_client(_create)

        if error:
            return error

        logger.info(
            "Memory block created: user=%s block=%s title=%s",
            user_id,
            block_label,
            title,
        )

        return (
            f"Memory block '{block_label}' created successfully with title '{title}'. "
            f"Future edits to this block will require user approval."
        )

    except Exception as e:
        logger.warning("add_memory_block failed for %s: %s", block_label, e)
        return f"Error creating memory block: {e}"
```

#### 2. Update tool instructions in server.py
**File**: `src/ralph/server.py`
**Changes**: Update the "Memory Blocks" section of `tool_instructions` (around line 233)

Replace the existing memory block instructions with:

```python
### Memory Blocks

You have access to memory blocks that contain persistent information about the student.
These blocks are shown below in "Student Context" when available.

To work with memory blocks:

**Creating new blocks:**
- Use `add_memory_block` to create a new block when you discover a topic, interest,
  or pattern worth tracking that doesn't fit in existing blocks
- Choose a descriptive snake_case label (e.g., "math_journey", "study_habits")
- New blocks are created immediately (no approval needed)

**Editing existing blocks:**
1. First, use `read_memory_block` to see the exact current content
2. Then, use `propose_memory_edit` with exact string matching to suggest changes
3. Your edits will be submitted as proposals that require user approval

Important: The `old_string` in your edit must match the block content exactly,
including whitespace and newlines. If the string appears multiple times,
provide more surrounding context to make it unique, or use `replace_all=True`.
```

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `make lint-fix && uv run ruff check src/ralph/`
- [ ] Type checking passes: `uv run basedpyright src/ralph/`
- [ ] Tests pass: `make test-agent`
- [ ] Full verify: `make verify-agent`

#### Manual Verification:
- [ ] Start dev server (`make dev`), send a message asking the agent to create a new memory block
- [ ] Verify the new block appears in `list_memory_blocks` output
- [ ] Verify the new block appears in the student's memory context on the next message
- [ ] Verify attempting to create a duplicate label returns an error
- [ ] Verify `propose_memory_edit` still works on the newly created block

## Testing Strategy

### Existing Tests:
- Check if there are existing tests for `MemoryBlockTools` — if so, add a test for `add_memory_block`
- The tool returns plain strings, so testing is straightforward with mocked `DoltClient`

### Key Edge Cases:
- Label already exists → returns error directing to `propose_memory_edit`
- Invalid label format (spaces, special chars) → returns validation error
- Empty title or body → returns validation error
- Dolt connection failure → returns error message (doesn't crash)

### Manual Testing (curl):
```bash
# After agent creates a block, verify via API:
USER_ID="ed6d1437-7b38-47a4-bd49-670267f0a7ce"
curl -s "http://localhost:8200/users/$USER_ID/blocks" | jq '.[] | .label'
```

## References

- `MemoryBlockTools`: `src/ralph/tools/memory_blocks.py`
- `DoltClient.update_block()`: `src/ralph/dolt.py:153-195`
- Agent construction: `src/ralph/server.py:307-320`
- Tool instructions: `src/ralph/server.py:227-258`
- Memory context builder: `src/ralph/memory.py:103-136`
