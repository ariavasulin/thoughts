# Memory Edit Tool Implementation Plan

## Overview

Create an Agno Toolkit that allows Ralph agents to propose memory block edits using a Claude Code-inspired `old_string`→`new_string` replacement interface. The tool will apply surgical string replacements and create proposals via the existing Dolt branch-based approval system.

## Current State Analysis

**Existing Infrastructure**:
- Memory blocks stored in Dolt with branch-based proposals (`src/ralph/dolt.py:323-399`)
- API endpoint `POST /users/{user_id}/blocks/{label}/propose` accepts full body replacement
- Agent receives memory blocks in system prompt via `build_memory_context()` (`src/ralph/memory.py`)
- Agno tools registered in `src/ralph/server.py:205-208` (FileTools, ShellTools)

**Gap**: No tool exists for agents to propose memory edits. Current system only supports full-body replacement via API.

## Desired End State

A `MemoryBlockTools` Agno Toolkit with three tools:
1. `propose_memory_edit` - Surgical string replacement (Claude Code pattern)
2. `read_memory_block` - Read current block content
3. `list_memory_blocks` - List available blocks

The tool validates `old_string` uniqueness and creates proposals through the existing Dolt system.

### Verification:
- Unit tests pass for all three tools
- Integration test: agent can read a block, propose an edit, and the proposal appears in Dolt
- `make check-agent` passes

## What We're NOT Doing

- Modifying the Dolt proposal system (it already works)
- Adding new API endpoints (tool calls DoltClient directly)
- Changing how memory context is passed to agents
- Adding section-based editing (future enhancement)

## Implementation Approach

Create a single new file `src/ralph/tools/memory_blocks.py` with an Agno Toolkit class. The toolkit:
1. Uses Agno's `RunContext` dependency injection for `user_id` (like HonchoTools)
2. Uses async DoltClient methods via thread pool (same pattern as HonchoTools)
3. Follows Claude Code error handling patterns exactly

**Key pattern from HonchoTools** (`src/ralph/tools/honcho_tools.py:34-57`):
- Tool methods receive `run_context: RunContext` as first param (auto-injected by Agno)
- Access user via `run_context.user_id` or `run_context.dependencies.get("user_id")`
- Handle async calls in sync tool via `concurrent.futures.ThreadPoolExecutor`

## Phase 1: Create MemoryBlockTools Toolkit

### Overview
Implement the core toolkit with all three tools.

### Changes Required:

#### 1. New Toolkit File
**File**: `src/ralph/tools/memory_blocks.py`

```python
"""Memory block tools for Ralph agent - Claude Code inspired editing."""

from __future__ import annotations

import asyncio
import concurrent.futures
import logging
from typing import TYPE_CHECKING, Any

from agno.tools import Toolkit

if TYPE_CHECKING:
    from agno.run import RunContext

from ralph.dolt import get_dolt_client

logger = logging.getLogger(__name__)


def _run_async(coro: Any) -> Any:
    """Run async coroutine from sync context (handles nested event loops)."""
    try:
        asyncio.get_running_loop()
        # In async context, use thread pool to avoid blocking
        with concurrent.futures.ThreadPoolExecutor() as pool:
            return pool.submit(asyncio.run, coro).result(timeout=30)
    except RuntimeError:
        # No running loop, can use asyncio.run directly
        return asyncio.run(coro)


def _get_user_id(run_context: RunContext) -> str | None:
    """Extract user_id from RunContext (via user_id or dependencies)."""
    user_id = run_context.user_id
    if not user_id:
        deps = run_context.dependencies or {}
        user_id = deps.get("user_id")
    return user_id


class MemoryBlockTools(Toolkit):
    """
    Tools for reading and proposing edits to student memory blocks.

    Memory blocks contain persistent information about the student that helps
    personalize tutoring. Agents can read blocks and propose edits, but edits
    require user approval before being applied.

    Inspired by Claude Code's Edit tool - uses surgical string replacement.

    Uses Agno's RunContext dependency injection for user_id (same as HonchoTools).
    """

    def __init__(self, agent_id: str = "ralph", **kwargs: Any) -> None:
        """
        Initialize memory block tools.

        Args:
            agent_id: Identifier for this agent (used in proposals)
        """
        self.agent_id = agent_id

        tools = [
            self.list_memory_blocks,
            self.read_memory_block,
            self.propose_memory_edit,
        ]

        super().__init__(name="memory_block_tools", tools=tools, **kwargs)

    def list_memory_blocks(self, run_context: RunContext) -> str:
        """
        List all available memory blocks for the current student.

        Args:
            run_context: Agno run context with user_id (auto-injected).

        Returns:
            A formatted list of memory blocks with their labels and titles,
            or a message if no blocks exist.
        """
        user_id = _get_user_id(run_context)
        if not user_id:
            return "Unable to identify student. No user context available."

        try:
            async def _list() -> list:
                dolt = await get_dolt_client()
                return await dolt.list_blocks(user_id)

            blocks = _run_async(_list())

            if not blocks:
                return "No memory blocks exist for this student yet."

            lines = ["Available memory blocks:", ""]
            for block in blocks:
                title = block.title or block.label.replace("_", " ").title()
                lines.append(f"- {block.label}: {title}")

            return "\n".join(lines)

        except Exception as e:
            logger.warning("list_memory_blocks failed: %s", e)
            return f"Error listing memory blocks: {e}"

    def read_memory_block(self, run_context: RunContext, block_label: str) -> str:
        """
        Read the current content of a memory block.

        Use this before proposing edits to see the exact current content.

        Args:
            run_context: Agno run context with user_id (auto-injected).
            block_label: The label/identifier of the block to read (e.g., "student", "goals")

        Returns:
            The block's current content, or an error message if not found.
        """
        user_id = _get_user_id(run_context)
        if not user_id:
            return "Unable to identify student. No user context available."

        try:
            async def _read():
                dolt = await get_dolt_client()
                return await dolt.get_block(user_id, block_label)

            block = _run_async(_read())

            if not block:
                return f"Memory block '{block_label}' not found."

            title = block.title or block_label.replace("_", " ").title()
            body = block.body or "(empty)"

            return f"# {title}\n\n{body}"

        except Exception as e:
            logger.warning("read_memory_block failed for %s: %s", block_label, e)
            return f"Error reading memory block: {e}"

    def propose_memory_edit(
        self,
        run_context: RunContext,
        block_label: str,
        old_string: str,
        new_string: str,
        reasoning: str,
        replace_all: bool = False,
    ) -> str:
        """
        Propose an edit to a memory block using string replacement.

        The edit will be submitted as a proposal that requires user approval.
        The old_string must match exactly (including whitespace) and must be
        unique in the block unless replace_all is True.

        IMPORTANT: You must read the memory block first to see its exact content.
        The edit will FAIL if old_string is not found or is not unique.

        Args:
            run_context: Agno run context with user_id (auto-injected).
            block_label: The label of the block to edit (e.g., "student", "goals")
            old_string: The exact text to find and replace. Must be unique unless replace_all=True.
            new_string: The text to replace it with. Must be different from old_string.
            reasoning: Brief explanation of why this edit is needed (shown to user for approval).
            replace_all: If True, replace all occurrences. If False (default), old_string must be unique.

        Returns:
            Success message if proposal created, or error message explaining what went wrong.
        """
        user_id = _get_user_id(run_context)
        if not user_id:
            return "Unable to identify student. No user context available."

        # Validate inputs
        if old_string == new_string:
            return "Error: old_string and new_string must be different."

        if not old_string:
            return "Error: old_string cannot be empty."

        if not reasoning:
            return "Error: reasoning is required to explain the edit to the user."

        try:
            async def _propose():
                dolt = await get_dolt_client()

                # Get current block content
                block = await dolt.get_block(user_id, block_label)
                if not block:
                    return None, f"Error: Memory block '{block_label}' not found."

                current_body = block.body or ""

                # Check if old_string exists
                if old_string not in current_body:
                    return None, (
                        f"Error: old_string not found in block '{block_label}'. "
                        "Make sure you've read the block first and the text matches exactly "
                        "(including whitespace and newlines)."
                    )

                # Check uniqueness unless replace_all
                occurrence_count = current_body.count(old_string)
                if occurrence_count > 1 and not replace_all:
                    return None, (
                        f"Error: old_string appears {occurrence_count} times in block '{block_label}'. "
                        "Provide a larger unique string with more surrounding context, "
                        "or set replace_all=True to replace all occurrences."
                    )

                # Apply the replacement
                if replace_all:
                    new_body = current_body.replace(old_string, new_string)
                else:
                    new_body = current_body.replace(old_string, new_string, 1)

                # Create the proposal via Dolt
                branch_name = await dolt.create_proposal(
                    user_id=user_id,
                    block_label=block_label,
                    new_body=new_body,
                    agent_id=self.agent_id,
                    reasoning=reasoning,
                    confidence="medium",
                )

                return branch_name, None

            branch_name, error = _run_async(_propose())

            if error:
                return error

            logger.info(
                "Memory edit proposed: user=%s block=%s branch=%s",
                user_id,
                block_label,
                branch_name,
            )

            return (
                f"Edit proposal created for block '{block_label}'. "
                f"The user will be asked to approve this change. "
                f"Reasoning provided: {reasoning}"
            )

        except Exception as e:
            logger.warning("propose_memory_edit failed for %s: %s", block_label, e)
            return f"Error creating edit proposal: {e}"
```

#### 2. Update Tools __init__.py
**File**: `src/ralph/tools/__init__.py`

Add the new export:
```python
"""Custom tools for Ralph agent."""

from ralph.tools.honcho_tools import HonchoTools
from ralph.tools.memory_blocks import MemoryBlockTools

__all__ = ["HonchoTools", "MemoryBlockTools"]
```

#### 3. Integrate Toolkit in Server
**File**: `src/ralph/server.py`

Add import at top (around line 37):
```python
from ralph.tools import HonchoTools, MemoryBlockTools
```

Update the agent tools list (around line 205-208):
```python
            tools=[
                strip_agno_fields(ShellTools(base_dir=workspace)),
                strip_agno_fields(FileTools(base_dir=workspace)),
                HonchoTools(),
                MemoryBlockTools(),  # Uses RunContext for user_id
            ],
```

Note: `MemoryBlockTools` gets `user_id` via Agno's `RunContext` injection (same as `HonchoTools`), so no constructor parameters needed. The `dependencies={"user_id": ...}` passed to `agent.arun()` at line 262-265 makes it available.

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `make lint-fix && make check-agent`
- [x] Type checking passes: `uv run basedpyright src/ralph/`
- [x] Unit tests pass (Phase 2)

#### Manual Verification:
- [ ] Agent can call `list_memory_blocks` and see available blocks
- [ ] Agent can call `read_memory_block` and see content
- [ ] Agent can propose an edit and it appears in Dolt branches

**Implementation Note**: After completing this phase, run `make check-agent` to verify linting and types pass before proceeding.

---

## Phase 2: Add Unit Tests

### Overview
Create comprehensive unit tests for the MemoryBlockTools toolkit.

### Changes Required:

#### 1. Test File
**File**: `tests/ralph/tools/test_memory_blocks.py`

```python
"""Tests for MemoryBlockTools."""

from __future__ import annotations

from datetime import datetime, timezone
from typing import Any
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from ralph.dolt import MemoryBlock
from ralph.tools.memory_blocks import MemoryBlockTools


@pytest.fixture
def mock_run_context() -> MagicMock:
    """Create a mock RunContext with user_id."""
    ctx = MagicMock()
    ctx.user_id = "test-user-123"
    ctx.dependencies = {"user_id": "test-user-123"}
    return ctx


@pytest.fixture
def mock_dolt() -> MagicMock:
    """Create a mock DoltClient."""
    return MagicMock()


@pytest.fixture
def tools() -> MemoryBlockTools:
    """Create MemoryBlockTools instance."""
    return MemoryBlockTools(agent_id="test-agent")


def make_run_async_mock(mock_dolt: MagicMock) -> Any:
    """Create a mock for _run_async that uses the mock_dolt."""
    def _mock_run_async(coro: Any) -> Any:
        # Extract the coroutine and run it with our mock
        import asyncio
        return asyncio.get_event_loop().run_until_complete(coro)
    return _mock_run_async


class TestListMemoryBlocks:
    """Tests for list_memory_blocks tool."""

    def test_list_blocks_returns_formatted_list(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should return formatted list of blocks."""
        mock_dolt.list_blocks = AsyncMock(
            return_value=[
                MemoryBlock(
                    user_id="test-user-123",
                    label="student",
                    title="Student Profile",
                    body="content",
                    schema_ref=None,
                    updated_at=datetime.now(timezone.utc),
                ),
                MemoryBlock(
                    user_id="test-user-123",
                    label="goals",
                    title="Learning Goals",
                    body="content",
                    schema_ref=None,
                    updated_at=datetime.now(timezone.utc),
                ),
            ]
        )

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.list_memory_blocks(mock_run_context)

        assert "student: Student Profile" in result
        assert "goals: Learning Goals" in result

    def test_list_blocks_empty(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should handle no blocks gracefully."""
        mock_dolt.list_blocks = AsyncMock(return_value=[])

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.list_memory_blocks(mock_run_context)

        assert "No memory blocks exist" in result

    def test_list_blocks_no_user(self, tools: MemoryBlockTools) -> None:
        """Should fail gracefully when no user_id available."""
        ctx = MagicMock()
        ctx.user_id = None
        ctx.dependencies = {}

        result = tools.list_memory_blocks(ctx)

        assert "Unable to identify student" in result


class TestReadMemoryBlock:
    """Tests for read_memory_block tool."""

    def test_read_block_returns_content(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should return block content with title."""
        mock_dolt.get_block = AsyncMock(
            return_value=MemoryBlock(
                user_id="test-user-123",
                label="student",
                title="Student Profile",
                body="## About\n\nTest content here.",
                schema_ref=None,
                updated_at=datetime.now(timezone.utc),
            )
        )

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.read_memory_block(mock_run_context, "student")

        assert "# Student Profile" in result
        assert "Test content here" in result

    def test_read_block_not_found(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should return error for missing block."""
        mock_dolt.get_block = AsyncMock(return_value=None)

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.read_memory_block(mock_run_context, "nonexistent")

        assert "not found" in result


class TestProposeMemoryEdit:
    """Tests for propose_memory_edit tool."""

    def test_propose_edit_success(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should create proposal when old_string is unique."""
        mock_dolt.get_block = AsyncMock(
            return_value=MemoryBlock(
                user_id="test-user-123",
                label="student",
                title="Student Profile",
                body="The student likes math.",
                schema_ref=None,
                updated_at=datetime.now(timezone.utc),
            )
        )
        mock_dolt.create_proposal = AsyncMock(return_value="agent/test-user-123/student")

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.propose_memory_edit(
                    mock_run_context,
                    block_label="student",
                    old_string="likes math",
                    new_string="loves mathematics",
                    reasoning="Student expressed stronger enthusiasm",
                )

        assert "proposal created" in result.lower()
        mock_dolt.create_proposal.assert_called_once()
        call_args = mock_dolt.create_proposal.call_args
        assert "loves mathematics" in call_args.kwargs["new_body"]

    def test_propose_edit_old_string_not_found(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should fail when old_string not in block."""
        mock_dolt.get_block = AsyncMock(
            return_value=MemoryBlock(
                user_id="test-user-123",
                label="student",
                title="Student Profile",
                body="The student likes math.",
                schema_ref=None,
                updated_at=datetime.now(timezone.utc),
            )
        )

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.propose_memory_edit(
                    mock_run_context,
                    block_label="student",
                    old_string="not in content",
                    new_string="replacement",
                    reasoning="test",
                )

        assert "not found" in result.lower()
        assert "read the block first" in result.lower()

    def test_propose_edit_non_unique_fails(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should fail when old_string appears multiple times."""
        mock_dolt.get_block = AsyncMock(
            return_value=MemoryBlock(
                user_id="test-user-123",
                label="student",
                title="Student Profile",
                body="The student likes math. The student also likes science.",
                schema_ref=None,
                updated_at=datetime.now(timezone.utc),
            )
        )

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.propose_memory_edit(
                    mock_run_context,
                    block_label="student",
                    old_string="The student",
                    new_string="This student",
                    reasoning="test",
                )

        assert "appears 2 times" in result
        assert "replace_all" in result.lower()

    def test_propose_edit_replace_all(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should replace all occurrences when replace_all=True."""
        mock_dolt.get_block = AsyncMock(
            return_value=MemoryBlock(
                user_id="test-user-123",
                label="student",
                title="Student Profile",
                body="The student likes math. The student also likes science.",
                schema_ref=None,
                updated_at=datetime.now(timezone.utc),
            )
        )
        mock_dolt.create_proposal = AsyncMock(return_value="agent/test-user-123/student")

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.propose_memory_edit(
                    mock_run_context,
                    block_label="student",
                    old_string="The student",
                    new_string="This student",
                    reasoning="test",
                    replace_all=True,
                )

        assert "proposal created" in result.lower()
        call_args = mock_dolt.create_proposal.call_args
        new_body = call_args.kwargs["new_body"]
        assert new_body.count("This student") == 2
        assert "The student" not in new_body

    def test_propose_edit_same_string_fails(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock
    ) -> None:
        """Should fail when old_string equals new_string."""
        result = tools.propose_memory_edit(
            mock_run_context,
            block_label="student",
            old_string="same",
            new_string="same",
            reasoning="test",
        )

        assert "must be different" in result.lower()

    def test_propose_edit_empty_old_string_fails(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock
    ) -> None:
        """Should fail when old_string is empty."""
        result = tools.propose_memory_edit(
            mock_run_context,
            block_label="student",
            old_string="",
            new_string="something",
            reasoning="test",
        )

        assert "cannot be empty" in result.lower()

    def test_propose_edit_missing_reasoning_fails(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock
    ) -> None:
        """Should fail when reasoning is empty."""
        result = tools.propose_memory_edit(
            mock_run_context,
            block_label="student",
            old_string="old",
            new_string="new",
            reasoning="",
        )

        assert "reasoning is required" in result.lower()

    def test_propose_edit_block_not_found(
        self, tools: MemoryBlockTools, mock_run_context: MagicMock, mock_dolt: MagicMock
    ) -> None:
        """Should fail when block doesn't exist."""
        mock_dolt.get_block = AsyncMock(return_value=None)

        with patch("ralph.tools.memory_blocks.get_dolt_client", AsyncMock(return_value=mock_dolt)):
            with patch("ralph.tools.memory_blocks._run_async", make_run_async_mock(mock_dolt)):
                result = tools.propose_memory_edit(
                    mock_run_context,
                    block_label="nonexistent",
                    old_string="old",
                    new_string="new",
                    reasoning="test",
                )

        assert "not found" in result.lower()
```

### Success Criteria:

#### Automated Verification:
- [x] All tests pass: `uv run pytest tests/ralph/tools/test_memory_blocks.py -v`
- [x] Coverage is reasonable for the new code

#### Manual Verification:
- [x] Tests cover all error cases from Claude Code pattern

**Implementation Note**: After completing this phase, run the tests to verify everything works.

---

## Phase 3: Update Agent Instructions

### Overview
Add documentation to the agent's system prompt about how to use the memory edit tools.

### Changes Required:

#### 1. Update Base Instructions in Server
**File**: `src/ralph/server.py`

Update the base instructions (around line 157-166) to add memory block tool guidance:

```python
        # Build instructions
        base_instructions = f"""You are a helpful AI tutor assistant.
Your workspace is: {workspace}
You can read and write files, and execute shell commands within this workspace.

You also have access to a `query_student` tool that lets you ask questions about
the current student's learning history and context. Use this tool when you need
to recall previous interactions, understand their learning style, or get context
about what you've discussed before.

## Memory Blocks

You have access to memory blocks that contain persistent information about the student.
These blocks are shown below in "Student Context" when available.

To update memory blocks, use the memory block tools:
1. First, use `read_memory_block` to see the exact current content
2. Then, use `propose_memory_edit` with exact string matching to suggest changes
3. Your edits will be submitted as proposals that require user approval

Important: The `old_string` in your edit must match the block content exactly,
including whitespace and newlines. If the string appears multiple times,
provide more surrounding context to make it unique, or use `replace_all=True`.

Always be helpful, encouraging, and focused on the student's learning goals."""
```

### Success Criteria:

#### Automated Verification:
- [x] `make check-agent` passes

#### Manual Verification:
- [ ] Agent receives helpful guidance about memory tools in its instructions

---

## Testing Strategy

### Unit Tests:
- Mock DoltClient to test tool logic in isolation
- Test all error cases (not found, non-unique, same string, empty inputs)
- Test replace_all behavior

### Integration Tests:
- Start Ralph server with Dolt container
- Create a test memory block
- Have agent propose an edit
- Verify proposal branch created in Dolt

### Manual Testing Steps:
1. Start Dolt: `docker compose up -d dolt`
2. Start Ralph: `uv run ralph-server`
3. Create a test block via curl
4. Open OpenWebUI and ask the agent to update the block
5. Verify proposal appears in `dolt_branches` table

## Performance Considerations

- Tool calls are async and use the shared DoltClient connection pool
- No additional network calls beyond existing Dolt queries
- Memory blocks are small (KB), so string operations are fast

## References

- Claude Code Edit tool: https://gist.github.com/wong2/e0f34aac66caf890a332f7b6f9e2ba8f
- Existing Dolt client: `src/ralph/dolt.py`
- Agno Toolkit patterns: `.venv/lib/python3.12/site-packages/agno/tools/toolkit.py`
- Ralph server: `src/ralph/server.py`
