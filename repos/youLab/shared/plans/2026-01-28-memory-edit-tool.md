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
1. Receives `user_id` at construction (set per-request in server.py)
2. Uses async DoltClient methods directly
3. Follows Claude Code error handling patterns exactly

## Phase 1: Create MemoryBlockTools Toolkit

### Overview
Implement the core toolkit with all three tools.

### Changes Required:

#### 1. New Toolkit File
**File**: `src/ralph/tools/memory_blocks.py`

```python
"""Memory block tools for Ralph agent - Claude Code inspired editing."""

from __future__ import annotations

from typing import TYPE_CHECKING

import structlog
from agno.tools.toolkit import Toolkit

if TYPE_CHECKING:
    from ralph.dolt import DoltClient

log = structlog.get_logger()


class MemoryBlockTools(Toolkit):
    """
    Tools for reading and proposing edits to student memory blocks.

    Memory blocks contain persistent information about the student that helps
    personalize tutoring. Agents can read blocks and propose edits, but edits
    require user approval before being applied.

    Inspired by Claude Code's Edit tool - uses surgical string replacement.
    """

    def __init__(
        self,
        dolt: DoltClient,
        user_id: str,
        agent_id: str = "ralph",
        **kwargs,
    ) -> None:
        """
        Initialize memory block tools.

        Args:
            dolt: DoltClient instance for database operations
            user_id: The student's user ID
            agent_id: Identifier for this agent (used in proposals)
        """
        self.dolt = dolt
        self.user_id = user_id
        self.agent_id = agent_id

        tools = [
            self.list_memory_blocks,
            self.read_memory_block,
            self.propose_memory_edit,
        ]

        super().__init__(name="memory_block_tools", tools=tools, **kwargs)

    async def list_memory_blocks(self) -> str:
        """
        List all available memory blocks for the current student.

        Returns:
            A formatted list of memory blocks with their labels and titles,
            or a message if no blocks exist.
        """
        try:
            blocks = await self.dolt.list_blocks(self.user_id)

            if not blocks:
                return "No memory blocks exist for this student yet."

            lines = ["Available memory blocks:", ""]
            for block in blocks:
                title = block.title or block.label.replace("_", " ").title()
                lines.append(f"- {block.label}: {title}")

            return "\n".join(lines)

        except Exception as e:
            log.exception("list_memory_blocks_error", user_id=self.user_id, error=str(e))
            return f"Error listing memory blocks: {e}"

    async def read_memory_block(self, block_label: str) -> str:
        """
        Read the current content of a memory block.

        Use this before proposing edits to see the exact current content.

        Args:
            block_label: The label/identifier of the block to read (e.g., "student", "goals")

        Returns:
            The block's current content, or an error message if not found.
        """
        try:
            block = await self.dolt.get_block(self.user_id, block_label)

            if not block:
                return f"Memory block '{block_label}' not found."

            title = block.title or block_label.replace("_", " ").title()
            body = block.body or "(empty)"

            return f"# {title}\n\n{body}"

        except Exception as e:
            log.exception(
                "read_memory_block_error",
                user_id=self.user_id,
                block_label=block_label,
                error=str(e),
            )
            return f"Error reading memory block: {e}"

    async def propose_memory_edit(
        self,
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
            block_label: The label of the block to edit (e.g., "student", "goals")
            old_string: The exact text to find and replace. Must be unique unless replace_all=True.
            new_string: The text to replace it with. Must be different from old_string.
            reasoning: Brief explanation of why this edit is needed (shown to user for approval).
            replace_all: If True, replace all occurrences. If False (default), old_string must be unique.

        Returns:
            Success message if proposal created, or error message explaining what went wrong.
        """
        # Validate inputs
        if old_string == new_string:
            return "Error: old_string and new_string must be different."

        if not old_string:
            return "Error: old_string cannot be empty."

        if not reasoning:
            return "Error: reasoning is required to explain the edit to the user."

        try:
            # Get current block content
            block = await self.dolt.get_block(self.user_id, block_label)

            if not block:
                return f"Error: Memory block '{block_label}' not found."

            current_body = block.body or ""

            # Check if old_string exists
            if old_string not in current_body:
                return (
                    f"Error: old_string not found in block '{block_label}'. "
                    "Make sure you've read the block first and the text matches exactly "
                    "(including whitespace and newlines)."
                )

            # Check uniqueness unless replace_all
            occurrence_count = current_body.count(old_string)
            if occurrence_count > 1 and not replace_all:
                return (
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
            branch_name = await self.dolt.create_proposal(
                user_id=self.user_id,
                block_label=block_label,
                new_body=new_body,
                agent_id=self.agent_id,
                reasoning=reasoning,
                confidence="medium",
            )

            log.info(
                "memory_edit_proposed",
                user_id=self.user_id,
                block_label=block_label,
                branch_name=branch_name,
                occurrences_replaced=occurrence_count if replace_all else 1,
            )

            return (
                f"Edit proposal created for block '{block_label}'. "
                f"The user will be asked to approve this change. "
                f"Reasoning provided: {reasoning}"
            )

        except Exception as e:
            log.exception(
                "propose_memory_edit_error",
                user_id=self.user_id,
                block_label=block_label,
                error=str(e),
            )
            return f"Error creating edit proposal: {e}"
```

#### 2. Update Tools __init__.py
**File**: `src/ralph/tools/__init__.py`

```python
"""Custom tools for Ralph agent."""

from ralph.tools.memory_blocks import MemoryBlockTools
from ralph.tools.query_honcho import QueryHonchoTool

__all__ = ["MemoryBlockTools", "QueryHonchoTool"]
```

#### 3. Integrate Toolkit in Server
**File**: `src/ralph/server.py`

Add import at top (around line 35):
```python
from ralph.tools.memory_blocks import MemoryBlockTools
```

Update the agent tools list (around line 205-208):
```python
        # Build the agent with tools (strip Agno fields Mistral doesn't accept)
        agent = Agent(
            model=OpenRouter(
                id=settings.openrouter_model,
                api_key=settings.openrouter_api_key,
            ),
            tools=[
                strip_agno_fields(ShellTools(base_dir=workspace)),
                strip_agno_fields(FileTools(base_dir=workspace)),
                strip_agno_fields(
                    MemoryBlockTools(
                        dolt=dolt,
                        user_id=request.user_id,
                        agent_id="ralph",
                    )
                ),
            ],
            instructions=instructions,
            markdown=True,
        )
```

Note: The `dolt` variable is already available in scope from `await get_dolt_client()` at line 167.

### Success Criteria:

#### Automated Verification:
- [ ] Linting passes: `make lint-fix && make check-agent`
- [ ] Type checking passes: `uv run basedpyright src/ralph/`
- [ ] Unit tests pass (Phase 2)

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
from unittest.mock import AsyncMock, MagicMock

import pytest

from ralph.dolt import MemoryBlock
from ralph.tools.memory_blocks import MemoryBlockTools


@pytest.fixture
def mock_dolt() -> MagicMock:
    """Create a mock DoltClient."""
    return MagicMock()


@pytest.fixture
def tools(mock_dolt: MagicMock) -> MemoryBlockTools:
    """Create MemoryBlockTools instance with mock client."""
    return MemoryBlockTools(
        dolt=mock_dolt,
        user_id="test-user-123",
        agent_id="test-agent",
    )


class TestListMemoryBlocks:
    """Tests for list_memory_blocks tool."""

    @pytest.mark.anyio
    async def test_list_blocks_returns_formatted_list(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
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

        result = await tools.list_memory_blocks()

        assert "student: Student Profile" in result
        assert "goals: Learning Goals" in result

    @pytest.mark.anyio
    async def test_list_blocks_empty(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
    ) -> None:
        """Should handle no blocks gracefully."""
        mock_dolt.list_blocks = AsyncMock(return_value=[])

        result = await tools.list_memory_blocks()

        assert "No memory blocks exist" in result


class TestReadMemoryBlock:
    """Tests for read_memory_block tool."""

    @pytest.mark.anyio
    async def test_read_block_returns_content(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
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

        result = await tools.read_memory_block("student")

        assert "# Student Profile" in result
        assert "Test content here" in result

    @pytest.mark.anyio
    async def test_read_block_not_found(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
    ) -> None:
        """Should return error for missing block."""
        mock_dolt.get_block = AsyncMock(return_value=None)

        result = await tools.read_memory_block("nonexistent")

        assert "not found" in result


class TestProposeMemoryEdit:
    """Tests for propose_memory_edit tool."""

    @pytest.mark.anyio
    async def test_propose_edit_success(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
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

        result = await tools.propose_memory_edit(
            block_label="student",
            old_string="likes math",
            new_string="loves mathematics",
            reasoning="Student expressed stronger enthusiasm",
        )

        assert "proposal created" in result.lower()
        mock_dolt.create_proposal.assert_called_once()
        call_args = mock_dolt.create_proposal.call_args
        assert "loves mathematics" in call_args.kwargs["new_body"]

    @pytest.mark.anyio
    async def test_propose_edit_old_string_not_found(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
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

        result = await tools.propose_memory_edit(
            block_label="student",
            old_string="not in content",
            new_string="replacement",
            reasoning="test",
        )

        assert "not found" in result.lower()
        assert "read the block first" in result.lower()

    @pytest.mark.anyio
    async def test_propose_edit_non_unique_fails(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
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

        result = await tools.propose_memory_edit(
            block_label="student",
            old_string="The student",
            new_string="This student",
            reasoning="test",
        )

        assert "appears 2 times" in result
        assert "replace_all" in result.lower()

    @pytest.mark.anyio
    async def test_propose_edit_replace_all(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
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

        result = await tools.propose_memory_edit(
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

    @pytest.mark.anyio
    async def test_propose_edit_same_string_fails(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
    ) -> None:
        """Should fail when old_string equals new_string."""
        result = await tools.propose_memory_edit(
            block_label="student",
            old_string="same",
            new_string="same",
            reasoning="test",
        )

        assert "must be different" in result.lower()

    @pytest.mark.anyio
    async def test_propose_edit_empty_old_string_fails(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
    ) -> None:
        """Should fail when old_string is empty."""
        result = await tools.propose_memory_edit(
            block_label="student",
            old_string="",
            new_string="something",
            reasoning="test",
        )

        assert "cannot be empty" in result.lower()

    @pytest.mark.anyio
    async def test_propose_edit_missing_reasoning_fails(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
    ) -> None:
        """Should fail when reasoning is empty."""
        result = await tools.propose_memory_edit(
            block_label="student",
            old_string="old",
            new_string="new",
            reasoning="",
        )

        assert "reasoning is required" in result.lower()

    @pytest.mark.anyio
    async def test_propose_edit_block_not_found(
        self, tools: MemoryBlockTools, mock_dolt: MagicMock
    ) -> None:
        """Should fail when block doesn't exist."""
        mock_dolt.get_block = AsyncMock(return_value=None)

        result = await tools.propose_memory_edit(
            block_label="nonexistent",
            old_string="old",
            new_string="new",
            reasoning="test",
        )

        assert "not found" in result.lower()
```

### Success Criteria:

#### Automated Verification:
- [ ] All tests pass: `uv run pytest tests/ralph/tools/test_memory_blocks.py -v`
- [ ] Coverage is reasonable for the new code

#### Manual Verification:
- [ ] Tests cover all error cases from Claude Code pattern

**Implementation Note**: After completing this phase, run the tests to verify everything works.

---

## Phase 3: Update Agent Instructions

### Overview
Add documentation to the agent's system prompt about how to use the memory edit tools.

### Changes Required:

#### 1. Update Base Instructions in Server
**File**: `src/ralph/server.py`

Update the base instructions (around line 157-162) to include tool guidance:

```python
        # Build instructions
        base_instructions = f"""You are a helpful coding assistant with access to a workspace.

Your workspace directory is: {workspace}
All file operations and shell commands are automatically scoped to this directory.
When listing files or running commands, they execute in your workspace.
You can create, read, and edit files to help the user with their tasks.

## Memory Blocks

You have access to memory blocks that contain persistent information about the student.
These blocks are shown above in "Student Context" when available.

To update memory blocks, use the memory block tools:
1. First, use `read_memory_block` to see the exact current content
2. Then, use `propose_memory_edit` with exact string matching to suggest changes
3. Your edits will be submitted as proposals that require user approval

Important: The `old_string` in your edit must match the block content exactly,
including whitespace and newlines. If the string appears multiple times,
provide more surrounding context to make it unique, or use `replace_all=True`."""
```

### Success Criteria:

#### Automated Verification:
- [ ] `make check-agent` passes

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
