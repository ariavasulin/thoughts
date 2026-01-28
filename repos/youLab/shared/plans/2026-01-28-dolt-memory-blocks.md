# Dolt Memory Block System Implementation Plan

## Overview

Replace the current GitPython-based memory block storage with Dolt, a MySQL-compatible database with git-like version control. This provides SQL queryability while preserving version history, diffing, and the agent approval workflow.

## Current State Analysis

The existing system uses:
- **GitPython** for per-user repositories at `.data/users/{user_id}/`
- **Markdown files** with YAML frontmatter for memory blocks
- **JSON files** for pending diffs (agent proposals awaiting approval)
- **Letta sync** to inject blocks into agent memory (being deprecated)

Key files being replaced:
- `src/youlab_server/storage/git.py` - GitUserStorage
- `src/youlab_server/storage/blocks.py` - UserBlockManager
- `src/youlab_server/storage/diffs.py` - PendingDiff, PendingDiffStore

## Desired End State

A single Dolt database with:
- **One `memory_blocks` table** with `user_id` column (using OpenWebUI user IDs)
- **Branch-based approval workflow** - agent proposals are branches, approval = merge to main
- **Version history** via Dolt's built-in `dolt_history_*` and `dolt_diff()`
- **Ralph/Agno integration** - blocks injected into agent instructions
- **Same frontend API** - Notes adapter continues to work, just backed by Dolt

### Verification

After implementation:
1. `make verify-agent` passes
2. Memory blocks can be created, read, updated via API
3. Version history is accessible via API
4. Agent proposals create branches, show as pending
5. Approving a proposal merges the branch
6. Rejecting deletes the branch
7. OpenWebUI Notes UI works unchanged

## What We're NOT Doing

- Migration of existing git data (greenfield implementation)
- Agno tools for memory access (blocks injected into instructions only)
- Changes to OpenWebUI frontend components
- DoltgreSQL (using Dolt with MySQL protocol)

## Implementation Approach

Use Dolt branches as the primitive for the approval workflow:
- `main` branch holds all users' approved memory blocks
- `agent/{user_id}/{block_label}` branches hold pending proposals
- Multiple agent edits to the same block **append commits** to the existing proposal branch (preserving proposal history)
- Approval = `DOLT_MERGE()` + delete branch (brings all proposal commits to main)
- Rejection = delete branch

This eliminates the need for a separate `pending_diffs` table while preserving full diff/history capabilities.

**Proposal branch lifecycle:**
```
agent/user123/student (proposal branch)
  ├── commit 1: {"agent_id": "tutor", "reasoning": "Added CS interest", ...}
  ├── commit 2: {"agent_id": "tutor", "reasoning": "Clarified goals", ...}
  └── commit 3: {"agent_id": "tutor", "reasoning": "Updated timeline", ...}
```
When approved, all commits merge to main. The diff shown to the user is always `main...branch` (cumulative change).

---

## Phase 1: Dolt Infrastructure Setup

### Overview
Set up Dolt server, database, schema, and Python client infrastructure.

### Changes Required:

#### 1. Docker Compose Service
**File**: `docker-compose.yml` (create or add to existing)

```yaml
services:
  dolt:
    image: dolthub/dolt-sql-server:latest
    ports:
      - "3307:3306"  # Avoid conflict with local MySQL
    volumes:
      - dolt-data:/var/lib/dolt
      - ./config/dolt:/docker-entrypoint-initdb.d
    environment:
      DOLT_ROOT_PASSWORD: ${DOLT_ROOT_PASSWORD:-devpassword}
      DOLT_ROOT_HOST: "%"
    command: ["--config", "/etc/dolt/servercfg.d/config.yaml"]

volumes:
  dolt-data:
```

#### 2. Dolt Server Configuration
**File**: `config/dolt/config.yaml` (new)

```yaml
log_level: info

behavior:
  read_only: false
  autocommit: true
  dolt_transaction_commit: false

listener:
  host: 0.0.0.0
  port: 3306
  max_connections: 100

user:
  name: root
  password: ${DOLT_ROOT_PASSWORD}
```

#### 3. Database Initialization Script
**File**: `config/dolt/init.sql` (new)

```sql
-- Create the YouLab database
CREATE DATABASE IF NOT EXISTS youlab;
USE youlab;

-- Memory blocks table
-- Primary key (user_id, label) enables cell-wise versioning in Dolt
CREATE TABLE IF NOT EXISTS memory_blocks (
    user_id VARCHAR(255) NOT NULL COMMENT 'OpenWebUI user ID',
    label VARCHAR(100) NOT NULL COMMENT 'Block identifier (e.g., student, journey)',
    title VARCHAR(255) COMMENT 'Human-readable block title',
    body TEXT COMMENT 'Block content (markdown)',
    schema_ref VARCHAR(255) COMMENT 'Schema reference (e.g., college-essay/student)',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, label)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Initial commit
CALL DOLT_ADD('-A');
CALL DOLT_COMMIT('-m', 'Initial schema setup');
```

#### 4. Python Dependencies
**File**: `pyproject.toml` (add to dependencies)

```toml
[project]
dependencies = [
    # ... existing deps ...
    "sqlalchemy>=2.0",
    "aiomysql>=0.2.0",
]
```

#### 5. Dolt Configuration Settings
**File**: `src/ralph/config.py` (modify)

```python
class Settings(BaseSettings):
    # ... existing settings ...

    # Dolt database settings
    dolt_host: str = "localhost"
    dolt_port: int = 3307
    dolt_user: str = "root"
    dolt_password: str = "devpassword"
    dolt_database: str = "youlab"

    @property
    def dolt_url(self) -> str:
        """SQLAlchemy async connection URL for Dolt."""
        return f"mysql+aiomysql://{self.dolt_user}:{self.dolt_password}@{self.dolt_host}:{self.dolt_port}/{self.dolt_database}"

    model_config = SettingsConfigDict(env_prefix="RALPH_")
```

### Success Criteria:

#### Automated Verification:
- [x] `docker compose up dolt` starts Dolt server successfully
- [x] Can connect to Dolt via `mysql -h localhost -P 3307 -u root -p`
- [x] `USE youlab; SHOW TABLES;` shows `memory_blocks`
- [x] `SELECT active_branch();` returns `main`
- [x] `uv sync` installs sqlalchemy and aiomysql without errors

#### Manual Verification:
- [x] Dolt container stays healthy after 60 seconds
- [x] Can execute `CALL DOLT_BRANCH('test'); CALL DOLT_BRANCH('-d', 'test');` cycle

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Dolt Client Layer

### Overview
Create async Python client for Dolt operations with connection pooling.

### Changes Required:

#### 1. Dolt Client Module
**File**: `src/ralph/dolt.py` (new)

```python
"""Dolt database client for memory block storage."""

from __future__ import annotations

import json
from contextlib import asynccontextmanager
from dataclasses import dataclass
from datetime import datetime
from typing import AsyncIterator

from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine

from ralph.config import Settings, get_settings


@dataclass
class MemoryBlock:
    """A memory block record."""
    user_id: str
    label: str
    title: str | None
    body: str | None
    schema_ref: str | None
    updated_at: datetime


@dataclass
class VersionInfo:
    """Version history entry."""
    commit_hash: str
    message: str
    author: str
    timestamp: datetime
    is_current: bool = False


@dataclass
class PendingProposal:
    """A pending agent proposal (represented as a branch)."""
    branch_name: str
    user_id: str
    block_label: str
    agent_id: str
    reasoning: str
    confidence: str
    created_at: datetime


class DoltClient:
    """Async client for Dolt database operations."""

    def __init__(self, settings: Settings | None = None):
        self._settings = settings or get_settings()
        self._engine: AsyncEngine | None = None
        self._session_factory: async_sessionmaker[AsyncSession] | None = None

    async def connect(self) -> None:
        """Initialize connection pool."""
        self._engine = create_async_engine(
            self._settings.dolt_url,
            pool_size=20,
            max_overflow=10,
            pool_recycle=1800,
            pool_pre_ping=True,
        )
        self._session_factory = async_sessionmaker(
            self._engine,
            expire_on_commit=False,
        )

    async def disconnect(self) -> None:
        """Close connection pool."""
        if self._engine:
            await self._engine.dispose()
            self._engine = None
            self._session_factory = None

    @asynccontextmanager
    async def session(self) -> AsyncIterator[AsyncSession]:
        """Get a database session."""
        if not self._session_factory:
            raise RuntimeError("DoltClient not connected. Call connect() first.")
        async with self._session_factory() as session:
            yield session

    # -------------------------------------------------------------------------
    # Memory Block Operations (on main branch)
    # -------------------------------------------------------------------------

    async def list_blocks(self, user_id: str) -> list[MemoryBlock]:
        """List all memory blocks for a user."""
        async with self.session() as session:
            result = await session.execute(
                text("SELECT user_id, label, title, body, schema_ref, updated_at "
                     "FROM memory_blocks WHERE user_id = :user_id"),
                {"user_id": user_id}
            )
            return [
                MemoryBlock(
                    user_id=row.user_id,
                    label=row.label,
                    title=row.title,
                    body=row.body,
                    schema_ref=row.schema_ref,
                    updated_at=row.updated_at,
                )
                for row in result.fetchall()
            ]

    async def get_block(self, user_id: str, label: str) -> MemoryBlock | None:
        """Get a specific memory block."""
        async with self.session() as session:
            result = await session.execute(
                text("SELECT user_id, label, title, body, schema_ref, updated_at "
                     "FROM memory_blocks WHERE user_id = :user_id AND label = :label"),
                {"user_id": user_id, "label": label}
            )
            row = result.fetchone()
            if not row:
                return None
            return MemoryBlock(
                user_id=row.user_id,
                label=row.label,
                title=row.title,
                body=row.body,
                schema_ref=row.schema_ref,
                updated_at=row.updated_at,
            )

    async def update_block(
        self,
        user_id: str,
        label: str,
        body: str,
        title: str | None = None,
        schema_ref: str | None = None,
        author: str = "user",
        message: str | None = None,
    ) -> str:
        """Update a memory block and commit. Returns commit hash."""
        async with self.session() as session:
            # Upsert the block
            await session.execute(
                text("""
                    INSERT INTO memory_blocks (user_id, label, title, body, schema_ref)
                    VALUES (:user_id, :label, :title, :body, :schema_ref)
                    ON DUPLICATE KEY UPDATE
                        title = COALESCE(:title, title),
                        body = :body,
                        schema_ref = COALESCE(:schema_ref, schema_ref)
                """),
                {
                    "user_id": user_id,
                    "label": label,
                    "title": title,
                    "body": body,
                    "schema_ref": schema_ref,
                }
            )
            await session.commit()

            # Dolt commit with author attribution
            commit_msg = message or f"Update {label}"
            author_str = f"{author} <{author}@youlab>"

            await session.execute(
                text("CALL DOLT_ADD('-A')"),
            )
            result = await session.execute(
                text("CALL DOLT_COMMIT('--skip-empty', '--author', :author, '-m', :message)"),
                {"author": author_str, "message": commit_msg}
            )
            commit_hash = result.fetchone()[0]
            return commit_hash

    async def delete_block(self, user_id: str, label: str, author: str = "user") -> str | None:
        """Delete a memory block. Returns commit hash or None if not found."""
        async with self.session() as session:
            result = await session.execute(
                text("DELETE FROM memory_blocks WHERE user_id = :user_id AND label = :label"),
                {"user_id": user_id, "label": label}
            )
            if result.rowcount == 0:
                return None

            await session.commit()

            await session.execute(text("CALL DOLT_ADD('-A')"))
            result = await session.execute(
                text("CALL DOLT_COMMIT('--skip-empty', '--author', :author, '-m', :message)"),
                {"author": f"{author} <{author}@youlab>", "message": f"Delete {label}"}
            )
            return result.fetchone()[0]

    # -------------------------------------------------------------------------
    # Version History Operations
    # -------------------------------------------------------------------------

    async def get_block_history(
        self,
        user_id: str,
        label: str,
        limit: int = 20
    ) -> list[VersionInfo]:
        """Get version history for a block."""
        async with self.session() as session:
            # Query dolt_history_memory_blocks for this specific block
            result = await session.execute(
                text("""
                    SELECT DISTINCT
                        commit_hash,
                        commit_date,
                        committer
                    FROM dolt_history_memory_blocks
                    WHERE user_id = :user_id AND label = :label
                    ORDER BY commit_date DESC
                    LIMIT :limit
                """),
                {"user_id": user_id, "label": label, "limit": limit}
            )

            versions = []
            for i, row in enumerate(result.fetchall()):
                # Get commit message from dolt_log
                log_result = await session.execute(
                    text("SELECT message FROM dolt_log WHERE commit_hash = :hash LIMIT 1"),
                    {"hash": row.commit_hash}
                )
                log_row = log_result.fetchone()
                message = log_row.message if log_row else "No message"

                versions.append(VersionInfo(
                    commit_hash=row.commit_hash,
                    message=message,
                    author=row.committer,
                    timestamp=row.commit_date,
                    is_current=(i == 0),
                ))

            return versions

    async def get_block_at_version(
        self,
        user_id: str,
        label: str,
        commit_hash: str
    ) -> MemoryBlock | None:
        """Get a block's state at a specific commit."""
        async with self.session() as session:
            result = await session.execute(
                text("""
                    SELECT user_id, label, title, body, schema_ref, commit_date as updated_at
                    FROM dolt_history_memory_blocks
                    WHERE user_id = :user_id
                      AND label = :label
                      AND commit_hash = :commit_hash
                """),
                {"user_id": user_id, "label": label, "commit_hash": commit_hash}
            )
            row = result.fetchone()
            if not row:
                return None
            return MemoryBlock(
                user_id=row.user_id,
                label=row.label,
                title=row.title,
                body=row.body,
                schema_ref=row.schema_ref,
                updated_at=row.updated_at,
            )

    async def restore_block(
        self,
        user_id: str,
        label: str,
        commit_hash: str,
        author: str = "user",
    ) -> str:
        """Restore a block to a previous version. Returns new commit hash."""
        old_block = await self.get_block_at_version(user_id, label, commit_hash)
        if not old_block:
            raise ValueError(f"Block {label} not found at commit {commit_hash}")

        return await self.update_block(
            user_id=user_id,
            label=label,
            body=old_block.body or "",
            title=old_block.title,
            schema_ref=old_block.schema_ref,
            author=author,
            message=f"Restore {label} to {commit_hash[:8]}",
        )

    # -------------------------------------------------------------------------
    # Proposal Operations (Branch-based approval workflow)
    # -------------------------------------------------------------------------

    def _proposal_branch_name(self, user_id: str, block_label: str) -> str:
        """Generate branch name for a proposal."""
        return f"agent/{user_id}/{block_label}"

    def _parse_proposal_metadata(self, commit_message: str) -> dict:
        """Parse proposal metadata from commit message JSON."""
        try:
            return json.loads(commit_message)
        except (json.JSONDecodeError, TypeError):
            return {}

    async def create_proposal(
        self,
        user_id: str,
        block_label: str,
        new_body: str,
        agent_id: str,
        reasoning: str,
        confidence: str = "medium",
        title: str | None = None,
        schema_ref: str | None = None,
    ) -> str:
        """Create or append to a proposal branch. Returns branch name.

        If a proposal branch already exists for this block, appends a new
        commit to it (preserving proposal history). Otherwise creates a
        new branch from main.
        """
        branch_name = self._proposal_branch_name(user_id, block_label)

        async with self.session() as session:
            # Check if branch already exists
            result = await session.execute(
                text("SELECT name FROM dolt_branches WHERE name = :name"),
                {"name": branch_name}
            )
            branch_exists = result.fetchone() is not None

            if branch_exists:
                # Switch to existing proposal branch (append to it)
                await session.execute(
                    text("CALL DOLT_CHECKOUT(:branch)"),
                    {"branch": branch_name}
                )
            else:
                # Create new branch from main
                await session.execute(
                    text("CALL DOLT_CHECKOUT('-b', :branch)"),
                    {"branch": branch_name}
                )

            try:
                # Make the proposed edit
                await session.execute(
                    text("""
                        INSERT INTO memory_blocks (user_id, label, title, body, schema_ref)
                        VALUES (:user_id, :label, :title, :body, :schema_ref)
                        ON DUPLICATE KEY UPDATE
                            title = COALESCE(:title, title),
                            body = :body,
                            schema_ref = COALESCE(:schema_ref, schema_ref)
                    """),
                    {
                        "user_id": user_id,
                        "label": block_label,
                        "title": title,
                        "body": new_body,
                        "schema_ref": schema_ref,
                    }
                )
                await session.commit()

                # Commit with metadata in message
                metadata = json.dumps({
                    "agent_id": agent_id,
                    "reasoning": reasoning,
                    "confidence": confidence,
                    "block_label": block_label,
                    "user_id": user_id,
                })

                await session.execute(text("CALL DOLT_ADD('-A')"))
                await session.execute(
                    text("CALL DOLT_COMMIT('-m', :message, '--author', :author)"),
                    {
                        "message": metadata,
                        "author": f"agent:{agent_id} <agent@youlab>",
                    }
                )
            finally:
                # Always switch back to main
                await session.execute(text("CALL DOLT_CHECKOUT('main')"))

        return branch_name

    async def list_proposals(self, user_id: str) -> list[PendingProposal]:
        """List all pending proposals for a user."""
        prefix = f"agent/{user_id}/"

        async with self.session() as session:
            result = await session.execute(
                text("SELECT name FROM dolt_branches WHERE name LIKE :prefix"),
                {"prefix": f"{prefix}%"}
            )

            proposals = []
            for row in result.fetchall():
                branch_name = row.name
                block_label = branch_name.replace(prefix, "")

                # Get commit info from the branch
                log_result = await session.execute(
                    text("SELECT message, committer, date FROM dolt_log(:branch) LIMIT 1"),
                    {"branch": branch_name}
                )
                log_row = log_result.fetchone()

                if log_row:
                    metadata = self._parse_proposal_metadata(log_row.message)
                    proposals.append(PendingProposal(
                        branch_name=branch_name,
                        user_id=user_id,
                        block_label=block_label,
                        agent_id=metadata.get("agent_id", "unknown"),
                        reasoning=metadata.get("reasoning", ""),
                        confidence=metadata.get("confidence", "medium"),
                        created_at=log_row.date,
                    ))

            return proposals

    async def get_proposal_diff(
        self,
        user_id: str,
        block_label: str
    ) -> dict | None:
        """Get the diff for a pending proposal."""
        branch_name = self._proposal_branch_name(user_id, block_label)

        async with self.session() as session:
            # Check branch exists
            result = await session.execute(
                text("SELECT name FROM dolt_branches WHERE name = :name"),
                {"name": branch_name}
            )
            if not result.fetchone():
                return None

            # Get diff between main and proposal branch
            result = await session.execute(
                text("""
                    SELECT * FROM dolt_diff('main', :branch, 'memory_blocks')
                    WHERE to_user_id = :user_id AND to_label = :label
                """),
                {"branch": branch_name, "user_id": user_id, "label": block_label}
            )
            row = result.fetchone()
            if not row:
                return None

            # Get proposal metadata
            log_result = await session.execute(
                text("SELECT message, date FROM dolt_log(:branch) LIMIT 1"),
                {"branch": branch_name}
            )
            log_row = log_result.fetchone()
            metadata = self._parse_proposal_metadata(log_row.message) if log_row else {}

            return {
                "branch_name": branch_name,
                "block_label": block_label,
                "current_body": row.from_body,
                "proposed_body": row.to_body,
                "current_title": row.from_title,
                "proposed_title": row.to_title,
                "agent_id": metadata.get("agent_id"),
                "reasoning": metadata.get("reasoning"),
                "confidence": metadata.get("confidence"),
                "created_at": log_row.date if log_row else None,
            }

    async def approve_proposal(self, user_id: str, block_label: str) -> str:
        """Approve and merge a proposal. Returns merge commit hash."""
        branch_name = self._proposal_branch_name(user_id, block_label)

        async with self.session() as session:
            # Get proposal metadata for commit message
            log_result = await session.execute(
                text("SELECT message FROM dolt_log(:branch) LIMIT 1"),
                {"branch": branch_name}
            )
            log_row = log_result.fetchone()
            metadata = self._parse_proposal_metadata(log_row.message) if log_row else {}
            reasoning = metadata.get("reasoning", "No reasoning provided")

            # Merge the branch
            result = await session.execute(
                text("CALL DOLT_MERGE(:branch, '-m', :message)"),
                {
                    "branch": branch_name,
                    "message": f"Approve agent proposal: {reasoning[:50]}",
                }
            )
            merge_result = result.fetchone()
            commit_hash = merge_result[0] if merge_result else "unknown"

            # Delete the branch
            await session.execute(
                text("CALL DOLT_BRANCH('-d', :branch)"),
                {"branch": branch_name}
            )

            return commit_hash

    async def reject_proposal(self, user_id: str, block_label: str) -> bool:
        """Reject a proposal by deleting its branch. Returns True if deleted."""
        branch_name = self._proposal_branch_name(user_id, block_label)

        async with self.session() as session:
            result = await session.execute(
                text("SELECT name FROM dolt_branches WHERE name = :name"),
                {"name": branch_name}
            )
            if not result.fetchone():
                return False

            await session.execute(
                text("CALL DOLT_BRANCH('-D', :branch)"),
                {"branch": branch_name}
            )
            return True

    async def count_pending_proposals(self, user_id: str) -> int:
        """Count pending proposals for a user."""
        prefix = f"agent/{user_id}/"
        async with self.session() as session:
            result = await session.execute(
                text("SELECT COUNT(*) FROM dolt_branches WHERE name LIKE :prefix"),
                {"prefix": f"{prefix}%"}
            )
            return result.scalar() or 0


# Module-level singleton
_client: DoltClient | None = None


async def get_dolt_client() -> DoltClient:
    """Get the shared DoltClient instance."""
    global _client
    if _client is None:
        _client = DoltClient()
        await _client.connect()
    return _client


async def close_dolt_client() -> None:
    """Close the shared DoltClient instance."""
    global _client
    if _client:
        await _client.disconnect()
        _client = None
```

### Success Criteria:

#### Automated Verification:
- [x] `uv run basedpyright src/ralph/dolt.py` passes
- [x] `uv run ruff check src/ralph/dolt.py` passes
- [ ] Unit tests for DoltClient pass (see Phase 3)

#### Manual Verification:
- [ ] Can instantiate DoltClient and connect to Dolt
- [ ] CRUD operations work on memory_blocks table
- [ ] Branch operations create/delete branches correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Unit Tests for Dolt Client

### Overview
Write comprehensive tests for the DoltClient.

### Changes Required:

#### 1. Test Module
**File**: `tests/test_dolt_client.py` (new)

```python
"""Tests for DoltClient."""

import pytest
from ralph.dolt import DoltClient, MemoryBlock


@pytest.fixture
async def dolt_client():
    """Create a DoltClient for testing."""
    client = DoltClient()
    await client.connect()
    yield client
    # Cleanup: delete test data
    async with client.session() as session:
        await session.execute(
            "DELETE FROM memory_blocks WHERE user_id LIKE 'test_%'"
        )
        await session.commit()
        await session.execute("CALL DOLT_COMMIT('-Am', 'Test cleanup')")
        # Delete any test branches
        result = await session.execute(
            "SELECT name FROM dolt_branches WHERE name LIKE 'agent/test_%'"
        )
        for row in result.fetchall():
            await session.execute(f"CALL DOLT_BRANCH('-D', '{row.name}')")
    await client.disconnect()


class TestMemoryBlockCRUD:
    """Test basic CRUD operations."""

    async def test_create_and_read_block(self, dolt_client: DoltClient):
        """Test creating and reading a block."""
        user_id = "test_user_1"
        label = "student"
        body = "Student name: Test User\nInterests: Testing"

        commit_hash = await dolt_client.update_block(
            user_id=user_id,
            label=label,
            body=body,
            title="Student Profile",
        )

        assert commit_hash is not None
        assert len(commit_hash) > 0

        block = await dolt_client.get_block(user_id, label)
        assert block is not None
        assert block.user_id == user_id
        assert block.label == label
        assert block.body == body
        assert block.title == "Student Profile"

    async def test_list_blocks(self, dolt_client: DoltClient):
        """Test listing blocks for a user."""
        user_id = "test_user_2"

        await dolt_client.update_block(user_id, "block1", "Content 1")
        await dolt_client.update_block(user_id, "block2", "Content 2")

        blocks = await dolt_client.list_blocks(user_id)
        assert len(blocks) == 2
        labels = {b.label for b in blocks}
        assert labels == {"block1", "block2"}

    async def test_delete_block(self, dolt_client: DoltClient):
        """Test deleting a block."""
        user_id = "test_user_3"

        await dolt_client.update_block(user_id, "to_delete", "Will be deleted")

        block = await dolt_client.get_block(user_id, "to_delete")
        assert block is not None

        commit_hash = await dolt_client.delete_block(user_id, "to_delete")
        assert commit_hash is not None

        block = await dolt_client.get_block(user_id, "to_delete")
        assert block is None


class TestVersionHistory:
    """Test version history operations."""

    async def test_get_block_history(self, dolt_client: DoltClient):
        """Test getting version history."""
        user_id = "test_user_4"
        label = "versioned"

        # Create multiple versions
        await dolt_client.update_block(user_id, label, "Version 1", message="First version")
        await dolt_client.update_block(user_id, label, "Version 2", message="Second version")
        await dolt_client.update_block(user_id, label, "Version 3", message="Third version")

        history = await dolt_client.get_block_history(user_id, label)

        assert len(history) >= 3
        assert history[0].is_current
        assert "Third" in history[0].message or "Version 3" in history[0].message

    async def test_get_block_at_version(self, dolt_client: DoltClient):
        """Test retrieving a specific version."""
        user_id = "test_user_5"
        label = "time_travel"

        await dolt_client.update_block(user_id, label, "Original content")
        history = await dolt_client.get_block_history(user_id, label)
        original_hash = history[0].commit_hash

        await dolt_client.update_block(user_id, label, "Modified content")

        old_block = await dolt_client.get_block_at_version(user_id, label, original_hash)
        assert old_block is not None
        assert old_block.body == "Original content"

    async def test_restore_block(self, dolt_client: DoltClient):
        """Test restoring a block to a previous version."""
        user_id = "test_user_6"
        label = "restorable"

        await dolt_client.update_block(user_id, label, "Original")
        history = await dolt_client.get_block_history(user_id, label)
        original_hash = history[0].commit_hash

        await dolt_client.update_block(user_id, label, "Changed")

        await dolt_client.restore_block(user_id, label, original_hash)

        block = await dolt_client.get_block(user_id, label)
        assert block.body == "Original"


class TestProposalWorkflow:
    """Test the branch-based approval workflow."""

    async def test_create_proposal(self, dolt_client: DoltClient):
        """Test creating a proposal."""
        user_id = "test_user_7"
        label = "proposable"

        # Create initial block
        await dolt_client.update_block(user_id, label, "Initial content")

        # Create proposal
        branch_name = await dolt_client.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body="Proposed content",
            agent_id="test_agent",
            reasoning="Test reasoning",
            confidence="high",
        )

        assert branch_name == f"agent/{user_id}/{label}"

        # Verify proposal exists
        proposals = await dolt_client.list_proposals(user_id)
        assert len(proposals) == 1
        assert proposals[0].block_label == label
        assert proposals[0].agent_id == "test_agent"

    async def test_proposal_appends_to_existing(self, dolt_client: DoltClient):
        """Test that new proposals append commits to existing branch."""
        user_id = "test_user_8"
        label = "append_test"

        await dolt_client.update_block(user_id, label, "Initial")

        # First proposal
        await dolt_client.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body="First proposal",
            agent_id="agent_1",
            reasoning="First reason",
        )

        # Second proposal (should append to same branch)
        await dolt_client.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body="Second proposal",
            agent_id="agent_2",
            reasoning="Second reason",
        )

        # Still one proposal branch
        proposals = await dolt_client.list_proposals(user_id)
        assert len(proposals) == 1

        # Latest commit has agent_2's info
        assert proposals[0].agent_id == "agent_2"

        # Diff shows cumulative change (Initial -> Second proposal)
        diff = await dolt_client.get_proposal_diff(user_id, label)
        assert diff["current_body"] == "Initial"
        assert diff["proposed_body"] == "Second proposal"

    async def test_get_proposal_diff(self, dolt_client: DoltClient):
        """Test getting diff for a proposal."""
        user_id = "test_user_9"
        label = "diff_test"

        await dolt_client.update_block(user_id, label, "Original body")
        await dolt_client.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body="Modified body",
            agent_id="diff_agent",
            reasoning="Testing diffs",
        )

        diff = await dolt_client.get_proposal_diff(user_id, label)

        assert diff is not None
        assert diff["current_body"] == "Original body"
        assert diff["proposed_body"] == "Modified body"
        assert diff["agent_id"] == "diff_agent"

    async def test_approve_proposal(self, dolt_client: DoltClient):
        """Test approving a proposal."""
        user_id = "test_user_10"
        label = "approvable"

        await dolt_client.update_block(user_id, label, "Before approval")
        await dolt_client.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body="After approval",
            agent_id="approve_agent",
            reasoning="Should be approved",
        )

        commit_hash = await dolt_client.approve_proposal(user_id, label)
        assert commit_hash is not None

        # Verify block is updated
        block = await dolt_client.get_block(user_id, label)
        assert block.body == "After approval"

        # Verify no pending proposals
        proposals = await dolt_client.list_proposals(user_id)
        assert len(proposals) == 0

    async def test_reject_proposal(self, dolt_client: DoltClient):
        """Test rejecting a proposal."""
        user_id = "test_user_11"
        label = "rejectable"

        await dolt_client.update_block(user_id, label, "Should stay")
        await dolt_client.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body="Should be rejected",
            agent_id="reject_agent",
            reasoning="Bad proposal",
        )

        result = await dolt_client.reject_proposal(user_id, label)
        assert result is True

        # Verify block is unchanged
        block = await dolt_client.get_block(user_id, label)
        assert block.body == "Should stay"

        # Verify no pending proposals
        proposals = await dolt_client.list_proposals(user_id)
        assert len(proposals) == 0

    async def test_count_pending_proposals(self, dolt_client: DoltClient):
        """Test counting pending proposals."""
        user_id = "test_user_12"

        await dolt_client.update_block(user_id, "block1", "Content 1")
        await dolt_client.update_block(user_id, "block2", "Content 2")

        await dolt_client.create_proposal(user_id, "block1", "New 1", "agent", "reason")
        await dolt_client.create_proposal(user_id, "block2", "New 2", "agent", "reason")

        count = await dolt_client.count_pending_proposals(user_id)
        assert count == 2
```

### Success Criteria:

#### Automated Verification:
- [ ] `make test-agent` passes (or pytest runs successfully)
- [ ] All tests in `test_dolt_client.py` pass
- [ ] No flaky tests on repeated runs

#### Manual Verification:
- [ ] Tests can be run against a real Dolt instance
- [ ] Test cleanup properly removes test data

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: HTTP API Layer

### Overview
Create FastAPI endpoints that replace the current blocks API, backed by Dolt.

### Changes Required:

#### 1. API Router
**File**: `src/ralph/api/blocks.py` (new)

```python
"""Memory blocks API endpoints backed by Dolt."""

from __future__ import annotations

from datetime import datetime
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from ralph.dolt import DoltClient, get_dolt_client


router = APIRouter(prefix="/users/{user_id}/blocks", tags=["blocks"])


# Request/Response Models
class BlockResponse(BaseModel):
    """Memory block response."""
    user_id: str
    label: str
    title: str | None
    body: str | None
    schema_ref: str | None
    updated_at: datetime
    pending_diffs: int = 0


class BlockListResponse(BaseModel):
    """List of memory blocks."""
    blocks: list[BlockResponse]


class BlockUpdateRequest(BaseModel):
    """Request to update a block."""
    body: str
    title: str | None = None
    schema_ref: str | None = None
    message: str | None = None


class VersionResponse(BaseModel):
    """Version history entry."""
    commit_sha: str
    message: str
    author: str
    timestamp: datetime
    is_current: bool


class VersionListResponse(BaseModel):
    """List of versions."""
    versions: list[VersionResponse]


class ProposalResponse(BaseModel):
    """Pending proposal response."""
    branch_name: str
    block_label: str
    agent_id: str
    reasoning: str
    confidence: str
    created_at: datetime


class ProposalDiffResponse(BaseModel):
    """Diff for a pending proposal."""
    branch_name: str
    block_label: str
    current_body: str | None
    proposed_body: str | None
    current_title: str | None
    proposed_title: str | None
    agent_id: str | None
    reasoning: str | None
    confidence: str | None
    created_at: datetime | None


class ProposeEditRequest(BaseModel):
    """Request to propose an edit (from agent)."""
    agent_id: str
    body: str
    reasoning: str
    confidence: str = "medium"
    title: str | None = None
    schema_ref: str | None = None


class ProposeEditResponse(BaseModel):
    """Response from proposing an edit."""
    branch_name: str
    success: bool
    error: str | None = None


class RestoreRequest(BaseModel):
    """Request to restore a block to a previous version."""
    commit_sha: str


# Dependency
DoltDep = Annotated[DoltClient, Depends(get_dolt_client)]


# Endpoints
@router.get("", response_model=BlockListResponse)
async def list_blocks(user_id: str, dolt: DoltDep) -> BlockListResponse:
    """List all memory blocks for a user."""
    blocks = await dolt.list_blocks(user_id)
    pending_count = await dolt.count_pending_proposals(user_id)

    # Get per-block pending counts
    proposals = await dolt.list_proposals(user_id)
    pending_by_block = {p.block_label: 1 for p in proposals}

    return BlockListResponse(
        blocks=[
            BlockResponse(
                user_id=b.user_id,
                label=b.label,
                title=b.title,
                body=b.body,
                schema_ref=b.schema_ref,
                updated_at=b.updated_at,
                pending_diffs=pending_by_block.get(b.label, 0),
            )
            for b in blocks
        ]
    )


@router.get("/{label}", response_model=BlockResponse)
async def get_block(user_id: str, label: str, dolt: DoltDep) -> BlockResponse:
    """Get a specific memory block."""
    block = await dolt.get_block(user_id, label)
    if not block:
        raise HTTPException(status_code=404, detail=f"Block {label} not found")

    pending = 1 if await dolt.get_proposal_diff(user_id, label) else 0

    return BlockResponse(
        user_id=block.user_id,
        label=block.label,
        title=block.title,
        body=block.body,
        schema_ref=block.schema_ref,
        updated_at=block.updated_at,
        pending_diffs=pending,
    )


@router.put("/{label}", response_model=BlockResponse)
async def update_block(
    user_id: str,
    label: str,
    request: BlockUpdateRequest,
    dolt: DoltDep,
) -> BlockResponse:
    """Update a memory block (user edit)."""
    await dolt.update_block(
        user_id=user_id,
        label=label,
        body=request.body,
        title=request.title,
        schema_ref=request.schema_ref,
        author="user",
        message=request.message,
    )

    block = await dolt.get_block(user_id, label)
    return BlockResponse(
        user_id=block.user_id,
        label=block.label,
        title=block.title,
        body=block.body,
        schema_ref=block.schema_ref,
        updated_at=block.updated_at,
        pending_diffs=0,
    )


@router.delete("/{label}")
async def delete_block(user_id: str, label: str, dolt: DoltDep) -> dict:
    """Delete a memory block."""
    result = await dolt.delete_block(user_id, label)
    if not result:
        raise HTTPException(status_code=404, detail=f"Block {label} not found")
    return {"deleted": True, "commit_sha": result}


@router.get("/{label}/history", response_model=VersionListResponse)
async def get_block_history(
    user_id: str,
    label: str,
    dolt: DoltDep,
    limit: int = 20,
) -> VersionListResponse:
    """Get version history for a block."""
    versions = await dolt.get_block_history(user_id, label, limit=limit)
    return VersionListResponse(
        versions=[
            VersionResponse(
                commit_sha=v.commit_hash,
                message=v.message,
                author=v.author,
                timestamp=v.timestamp,
                is_current=v.is_current,
            )
            for v in versions
        ]
    )


@router.get("/{label}/versions/{commit_sha}", response_model=BlockResponse)
async def get_block_at_version(
    user_id: str,
    label: str,
    commit_sha: str,
    dolt: DoltDep,
) -> BlockResponse:
    """Get a block at a specific version."""
    block = await dolt.get_block_at_version(user_id, label, commit_sha)
    if not block:
        raise HTTPException(
            status_code=404,
            detail=f"Block {label} not found at commit {commit_sha}"
        )
    return BlockResponse(
        user_id=block.user_id,
        label=block.label,
        title=block.title,
        body=block.body,
        schema_ref=block.schema_ref,
        updated_at=block.updated_at,
        pending_diffs=0,
    )


@router.post("/{label}/restore", response_model=BlockResponse)
async def restore_block(
    user_id: str,
    label: str,
    request: RestoreRequest,
    dolt: DoltDep,
) -> BlockResponse:
    """Restore a block to a previous version."""
    try:
        await dolt.restore_block(user_id, label, request.commit_sha)
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))

    block = await dolt.get_block(user_id, label)
    return BlockResponse(
        user_id=block.user_id,
        label=block.label,
        title=block.title,
        body=block.body,
        schema_ref=block.schema_ref,
        updated_at=block.updated_at,
        pending_diffs=0,
    )


# Proposal endpoints
@router.get("/{label}/diffs", response_model=list[ProposalDiffResponse])
async def get_pending_diffs(
    user_id: str,
    label: str,
    dolt: DoltDep,
) -> list[ProposalDiffResponse]:
    """Get pending diffs for a block."""
    diff = await dolt.get_proposal_diff(user_id, label)
    if not diff:
        return []
    return [ProposalDiffResponse(**diff)]


@router.post("/{label}/propose", response_model=ProposeEditResponse)
async def propose_edit(
    user_id: str,
    label: str,
    request: ProposeEditRequest,
    dolt: DoltDep,
) -> ProposeEditResponse:
    """Propose an edit to a block (called by agents)."""
    try:
        branch_name = await dolt.create_proposal(
            user_id=user_id,
            block_label=label,
            new_body=request.body,
            agent_id=request.agent_id,
            reasoning=request.reasoning,
            confidence=request.confidence,
            title=request.title,
            schema_ref=request.schema_ref,
        )
        return ProposeEditResponse(branch_name=branch_name, success=True)
    except Exception as e:
        return ProposeEditResponse(
            branch_name="",
            success=False,
            error=str(e)
        )


@router.post("/{label}/diffs/{diff_id}/approve")
async def approve_diff(
    user_id: str,
    label: str,
    diff_id: str,  # Ignored - we use label as key
    dolt: DoltDep,
) -> dict:
    """Approve a pending diff."""
    try:
        commit_sha = await dolt.approve_proposal(user_id, label)
        return {"approved": True, "commit_sha": commit_sha}
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.post("/{label}/diffs/{diff_id}/reject")
async def reject_diff(
    user_id: str,
    label: str,
    diff_id: str,  # Ignored - we use label as key
    dolt: DoltDep,
) -> dict:
    """Reject a pending diff."""
    result = await dolt.reject_proposal(user_id, label)
    if not result:
        raise HTTPException(status_code=404, detail="No pending proposal found")
    return {"rejected": True}
```

#### 2. Register Router in Server
**File**: `src/ralph/server.py` (modify)

Add to imports and app setup:
```python
from ralph.api.blocks import router as blocks_router

# In create_app() or wherever app is configured:
app.include_router(blocks_router)
```

#### 3. Lifecycle Management
**File**: `src/ralph/server.py` (modify)

Add startup/shutdown hooks:
```python
from ralph.dolt import get_dolt_client, close_dolt_client

@app.on_event("startup")
async def startup():
    await get_dolt_client()  # Initialize connection pool

@app.on_event("shutdown")
async def shutdown():
    await close_dolt_client()  # Close connection pool
```

### Success Criteria:

#### Automated Verification:
- [ ] `uv run basedpyright src/ralph/api/blocks.py` passes
- [ ] `uv run ruff check src/ralph/api/` passes
- [ ] API endpoint tests pass (see Phase 5)

#### Manual Verification:
- [ ] `curl http://localhost:8200/users/test/blocks` returns empty list
- [ ] Can create, update, delete blocks via API
- [ ] Version history endpoint works
- [ ] Proposal workflow works end-to-end

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 5.

---

## Phase 5: Notes API Adapter Update

### Overview
Update the Notes API adapter to use Dolt instead of git storage.

### Changes Required:

#### 1. Update Notes Adapter
**File**: `src/ralph/api/notes_adapter.py` (new, replaces `src/youlab_server/server/notes_adapter.py`)

```python
"""Notes API adapter - bridges OpenWebUI Notes format to Dolt-backed blocks."""

from __future__ import annotations

from datetime import datetime
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from ralph.dolt import DoltClient, get_dolt_client


router = APIRouter(prefix="/you/notes", tags=["notes-adapter"])


# OpenWebUI-compatible models
class NoteContent(BaseModel):
    """Note content in OpenWebUI format."""
    html: str
    md: str
    json: None = None  # Always null for our use case


class NoteVersion(BaseModel):
    """Version entry in OpenWebUI format."""
    sha: str
    message: str
    timestamp: int  # Nanoseconds epoch


class NoteResponse(BaseModel):
    """Full note response for OpenWebUI."""
    id: str
    title: str
    content: NoteContent
    versions: list[NoteVersion]
    user_id: str
    created_at: int  # Nanoseconds epoch
    updated_at: int  # Nanoseconds epoch


class NoteListItem(BaseModel):
    """Note list item for OpenWebUI."""
    id: str
    title: str
    user_id: str
    created_at: int
    updated_at: int
    pending_diffs: int = 0


class NoteListResponse(BaseModel):
    """List of notes response."""
    notes: list[NoteListItem]


class NoteUpdateRequest(BaseModel):
    """Request to update a note."""
    content: NoteContent


DoltDep = Annotated[DoltClient, Depends(get_dolt_client)]


def _datetime_to_nanos(dt: datetime) -> int:
    """Convert datetime to nanoseconds epoch."""
    return int(dt.timestamp() * 1_000_000_000)


def _md_to_html(md: str) -> str:
    """Simple markdown to HTML conversion."""
    # For now, just wrap in a div.
    # Could use markdown library for proper conversion.
    return f"<div>{md}</div>"


@router.get("/", response_model=NoteListResponse)
async def list_notes(dolt: DoltDep, user_id: str | None = None) -> NoteListResponse:
    """List all notes for a user."""
    # TODO: Get user_id from auth context instead of query param
    if not user_id:
        return NoteListResponse(notes=[])

    blocks = await dolt.list_blocks(user_id)
    proposals = await dolt.list_proposals(user_id)
    pending_by_block = {p.block_label: 1 for p in proposals}

    return NoteListResponse(
        notes=[
            NoteListItem(
                id=b.label,
                title=b.title or b.label,
                user_id=b.user_id,
                created_at=_datetime_to_nanos(b.updated_at),  # TODO: track created_at
                updated_at=_datetime_to_nanos(b.updated_at),
                pending_diffs=pending_by_block.get(b.label, 0),
            )
            for b in blocks
        ]
    )


@router.get("/{note_id}", response_model=NoteResponse)
async def get_note(note_id: str, dolt: DoltDep, user_id: str | None = None) -> NoteResponse:
    """Get a note by ID (block label)."""
    if not user_id:
        raise HTTPException(status_code=400, detail="user_id required")

    block = await dolt.get_block(user_id, note_id)
    if not block:
        raise HTTPException(status_code=404, detail=f"Note {note_id} not found")

    history = await dolt.get_block_history(user_id, note_id)

    body = block.body or ""

    return NoteResponse(
        id=block.label,
        title=block.title or block.label,
        content=NoteContent(
            html=_md_to_html(body),
            md=body,
        ),
        versions=[
            NoteVersion(
                sha=v.commit_hash,
                message=v.message,
                timestamp=_datetime_to_nanos(v.timestamp),
            )
            for v in history
        ],
        user_id=block.user_id,
        created_at=_datetime_to_nanos(block.updated_at),  # TODO: track created_at
        updated_at=_datetime_to_nanos(block.updated_at),
    )


@router.post("/{note_id}/update", response_model=NoteResponse)
async def update_note(
    note_id: str,
    request: NoteUpdateRequest,
    dolt: DoltDep,
    user_id: str | None = None,
) -> NoteResponse:
    """Update a note."""
    if not user_id:
        raise HTTPException(status_code=400, detail="user_id required")

    await dolt.update_block(
        user_id=user_id,
        label=note_id,
        body=request.content.md,
        author="user",
        message=f"Update {note_id}",
    )

    return await get_note(note_id, dolt, user_id)
```

#### 2. Register Notes Router
**File**: `src/ralph/server.py` (modify)

```python
from ralph.api.notes_adapter import router as notes_router

# Mount at /api for OpenWebUI compatibility
app.include_router(notes_router, prefix="/api")
```

### Success Criteria:

#### Automated Verification:
- [x] `uv run basedpyright src/ralph/api/notes_adapter.py` passes
- [x] `uv run ruff check src/ralph/api/notes_adapter.py` passes

#### Manual Verification:
- [ ] OpenWebUI can list notes via `/api/you/notes/`
- [ ] OpenWebUI can read a note via `/api/you/notes/{id}`
- [ ] OpenWebUI can update a note via `/api/you/notes/{id}/update`
- [ ] Version history shows in OpenWebUI UI

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 6.

---

## Phase 6: Ralph Agent Integration

### Overview
Inject memory blocks into Agno agent instructions.

### Changes Required:

#### 1. Memory Block Loader
**File**: `src/ralph/memory.py` (new)

```python
"""Memory block loading for agent instructions."""

from __future__ import annotations

from ralph.dolt import DoltClient


async def build_memory_context(dolt: DoltClient, user_id: str) -> str:
    """Build memory context string for agent instructions."""
    blocks = await dolt.list_blocks(user_id)

    if not blocks:
        return ""

    sections = ["## Student Memory\n"]

    for block in blocks:
        title = block.title or block.label.replace("_", " ").title()
        body = block.body or "(empty)"
        sections.append(f"### {title}\n\n{body}\n")

    return "\n".join(sections)


async def get_block_for_agent(
    dolt: DoltClient,
    user_id: str,
    label: str
) -> str | None:
    """Get a specific block's content for agent use."""
    block = await dolt.get_block(user_id, label)
    if not block:
        return None
    return block.body
```

#### 2. Update Server to Inject Memory
**File**: `src/ralph/server.py` (modify)

In the chat endpoint, inject memory into instructions:

```python
from ralph.memory import build_memory_context
from ralph.dolt import get_dolt_client

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    # ... existing code ...

    # Build instructions with memory context
    dolt = await get_dolt_client()
    memory_context = await build_memory_context(dolt, request.user_id)

    instructions = f"""
{base_instructions}

{memory_context}
"""

    agent = Agent(
        # ... existing config ...
        instructions=instructions,
    )

    # ... rest of handler ...
```

### Success Criteria:

#### Automated Verification:
- [x] `uv run basedpyright src/ralph/memory.py` passes
- [x] `uv run ruff check src/ralph/memory.py` passes
- [x] `make verify-agent` passes (ralph code passes; legacy youlab_server/ has pre-existing type errors)

#### Manual Verification:
- [ ] Agent can see student memory blocks in its context
- [ ] Changes to memory blocks are reflected in subsequent agent conversations

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 7.

---

## Phase 7: Cleanup and Documentation

### Overview
Remove old git-based storage code and update documentation.

### Changes Required:

#### 1. Remove Deprecated Files
Delete the following files (they're replaced by Dolt implementation):
- `src/youlab_server/storage/git.py`
- `src/youlab_server/storage/blocks.py`
- `src/youlab_server/storage/diffs.py`
- `src/youlab_server/server/notes_adapter.py`
- `src/youlab_server/tools/memory.py` (the Letta-based one)

#### 2. Update CLAUDE.md
Add Dolt information to the architecture section.

#### 3. Update pyproject.toml
Remove GitPython dependency if no longer needed.

### Success Criteria:

#### Automated Verification:
- [ ] `make verify-agent` passes after cleanup
- [ ] No import errors for removed modules
- [ ] `uv sync` still works

#### Manual Verification:
- [ ] Documentation accurately reflects new architecture
- [ ] No dead code references

---

## Testing Strategy

### Unit Tests:
- DoltClient CRUD operations
- Version history retrieval
- Proposal create/approve/reject workflow
- Branch name generation and parsing

### Integration Tests:
- Full API endpoint tests
- Notes adapter compatibility
- Agent memory injection

### Manual Testing Steps:
1. Create a memory block via API
2. Update the block, verify version history
3. Create a proposal via agent endpoint
4. View proposal diff in UI
5. Approve proposal, verify block updated
6. Reject a different proposal, verify block unchanged
7. Restore block to previous version
8. Verify agent sees updated memory in conversation

## Performance Considerations

- Connection pooling configured with 20 persistent connections
- Dolt diff operations scale with change size, not table size
- Branch operations are lightweight (Dolt branches are cheap)
- Consider read replicas if read load becomes high

## References

- Research: `thoughts/shared/research/2026-01-28-memory-block-system-state.md`
- Dolt Reference: `thoughts/shared/research/2026-01-24-dolt-reference.md`
- Current implementation: `src/youlab_server/storage/` (being replaced)
