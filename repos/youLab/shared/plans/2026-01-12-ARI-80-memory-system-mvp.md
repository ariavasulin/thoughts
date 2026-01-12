# ARI-80: Memory System MVP - Implementation Plan

## Overview

Implement the core infrastructure for user-scoped memory blocks with git-backed versioning and a basic UI for viewing/editing. This is Phase B of the YouLab product overhaul, running in parallel with Phase A (UI + Modules).

**Key deliverables:**
- GitUserStorage class for versioned file storage per user
- User initialization endpoint (`/users/init` webhook)
- UserBlockManager for user-scoped memory blocks
- Memory block CRUD API with version history
- "You" section in OpenWebUI sidebar with Profile + Agents tabs
- Agents tab showing background agent thread history

## Current State Analysis

### Backend

**No Git Integration**
- Codebase uses SHA256 content hashing for sync state (`server/sync/mappings.py:158-171`)
- No version control, no commit history, no diffs
- Need to add GitPython dependency

**Memory Blocks are Course-Scoped**
- Current naming: `youlab_shared_{course_id}_{block_label}` (`agents.py:53-55`)
- Shared across all users of a course, not per-user
- Cache key: `(course_id, block_label)` not `(user_id, block_label)`

**Deprecated Memory APIs**
- `MemoryManager` uses `client.get_agent_memory()` and `client.update_agent_core_memory()` (`memory/manager.py:94, 141`)
- Modern API: `client.agents.blocks.list()`, `client.agents.blocks.update()`
- `edit_memory_block` tool hardcoded to "human"/"persona" labels only (`tools/memory.py:140-152`)

**No User Initialization**
- Lazy agent creation on first interaction (`agents.py:252-260`)
- No central user registry or user storage directory
- No webhook receiver for OpenWebUI signup events

### Frontend (OpenWebUI)

**Relevant Patterns Identified:**
- Sidebar: `src/lib/components/layout/Sidebar.svelte`
- Settings modal: `src/lib/components/chat/SettingsModal.svelte` (tabbed interface)
- Memory editing: `src/lib/components/chat/Settings/Personalization/EditMemoryModal.svelte`
- Workspace cards: `src/lib/components/workspace/Knowledge.svelte` (grid layout)
- Badge component: `src/lib/components/common/Badge.svelte`

## Desired End State

After this plan is complete:

1. **Each user has a git-backed storage directory** at `.data/users/{user_id}/`
   - Memory blocks stored as TOML files in `blocks/` subdirectory
   - Full git history for every change
   - Can view history, diff versions, restore previous states

2. **User initialization on signup**
   - OpenWebUI webhook triggers `/users/init`
   - Creates user directory, initializes git repo
   - Populates default blocks from course schema

3. **Memory blocks are user-scoped**
   - Letta blocks named `youlab_user_{user_id}_{block_label}`
   - Each user has their own set of blocks
   - Agents attach to user's blocks (not course's)

4. **Agent memory edits require approval**
   - Agent calls `edit_memory_block` → creates pending diff
   - Diff stored in user's git directory
   - User approves/rejects in UI before change applies

5. **"You" section in UI**
   - Sidebar menu item navigates to profile
   - Two tabs: Profile (memory blocks) and Agents (background agent threads)
   - Profile tab shows memory block cards
   - Click card → detail view with markdown editor
   - Version history with restore buttons
   - Agents tab shows background agents with thread history
   - Click thread → navigates to OpenWebUI chat

### Verification

```bash
# Backend verification
curl -X POST localhost:8000/users/init -d '{"user_id": "test", "name": "Test User"}'
# Creates .data/users/test/ with git repo

curl localhost:8000/users/test/blocks
# Returns list of memory blocks

curl localhost:8000/users/test/blocks/student
# Returns block content with version history

# Git verification
cd .data/users/test && git log --oneline
# Shows commit history for block edits
```

## What We're NOT Doing

- **Diff approval UI** - Pending diffs will be stored but the approval/reject UI is a separate ticket
- **Real-time notifications** - Socket.IO integration for diff badges is out of scope
- **SCIM/LDAP user support** - Only standard signup and OAuth trigger initialization
- **Multi-course blocks** - Blocks are per-user, not per-user-per-course (simplified model)
- **Letta archival migration** - Existing archival memory stays in Letta, not migrated to git

## Implementation Approach

We'll build bottom-up: storage layer → API layer → UI layer. Each phase is independently testable.

**Storage format decision:**
- TOML files are source of truth (structured, validated)
- Markdown for user editing/display (human-readable)
- Bidirectional conversion with frontmatter for metadata

---

## Phase 1: Git Storage Infrastructure

### Overview
Add GitPython dependency and create the `GitUserStorage` class that manages per-user git repositories.

### Changes Required

#### 1. Add GitPython Dependency
**File**: `pyproject.toml`

```toml
[project]
dependencies = [
    # ... existing deps ...
    "gitpython>=3.1.0",
]
```

#### 2. Create GitUserStorage Class
**File**: `src/youlab_server/storage/__init__.py` (new directory)

```python
"""User storage with git versioning."""
```

**File**: `src/youlab_server/storage/git.py` (new file)

```python
"""Git-backed user storage."""

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
from typing import TYPE_CHECKING

import structlog
from git import Repo
from git.exc import InvalidGitRepositoryError

if TYPE_CHECKING:
    from git import Commit

log = structlog.get_logger()


@dataclass
class VersionInfo:
    """Information about a file version."""

    commit_sha: str
    message: str
    author: str
    timestamp: datetime
    is_current: bool = False


@dataclass
class FileDiff:
    """Diff between two versions of a file."""

    old_content: str
    new_content: str
    old_sha: str
    new_sha: str


class GitUserStorage:
    """
    Git-backed storage for a single user.

    Directory structure:
        .data/users/{user_id}/
            .git/
            blocks/
                student.toml
                engagement_strategy.toml
                journey.toml
            pending_diffs/
                {diff_id}.json

    Each edit creates a git commit for full version history.
    """

    def __init__(self, user_id: str, base_dir: Path | str) -> None:
        """
        Initialize user storage.

        Args:
            user_id: User identifier
            base_dir: Base directory for all user storage (e.g., .data/users)

        """
        self.user_id = user_id
        self.base_dir = Path(base_dir)
        self.user_dir = self.base_dir / user_id
        self.blocks_dir = self.user_dir / "blocks"
        self.diffs_dir = self.user_dir / "pending_diffs"
        self._repo: Repo | None = None
        self.logger = log.bind(user_id=user_id, component="git_storage")

    @property
    def repo(self) -> Repo:
        """Get or initialize the git repository."""
        if self._repo is None:
            try:
                self._repo = Repo(self.user_dir)
            except InvalidGitRepositoryError:
                self._repo = self._init_repo()
        return self._repo

    @property
    def exists(self) -> bool:
        """Check if user storage exists."""
        return self.user_dir.exists() and (self.user_dir / ".git").exists()

    def init(self) -> None:
        """Initialize user storage directory and git repo."""
        if self.exists:
            self.logger.debug("storage_already_exists")
            return

        # Create directories
        self.blocks_dir.mkdir(parents=True, exist_ok=True)
        self.diffs_dir.mkdir(parents=True, exist_ok=True)

        # Initialize git repo
        self._repo = self._init_repo()
        self.logger.info("storage_initialized")

    def _init_repo(self) -> Repo:
        """Initialize a new git repository."""
        repo = Repo.init(self.user_dir)

        # Configure repo
        with repo.config_writer() as config:
            config.set_value("user", "name", "YouLab System")
            config.set_value("user", "email", "system@youlab.local")

        # Create initial commit
        gitignore = self.user_dir / ".gitignore"
        gitignore.write_text("# YouLab user storage\n*.pyc\n__pycache__/\n")
        repo.index.add([".gitignore"])
        repo.index.commit("Initialize user storage")

        return repo

    def read_block(self, label: str) -> str | None:
        """
        Read a memory block file.

        Args:
            label: Block label (e.g., "student", "journey")

        Returns:
            File content or None if not found

        """
        path = self.blocks_dir / f"{label}.toml"
        if not path.exists():
            return None
        return path.read_text()

    def write_block(
        self,
        label: str,
        content: str,
        message: str | None = None,
        author: str = "user",
    ) -> str:
        """
        Write a memory block and commit.

        Args:
            label: Block label
            content: TOML content
            message: Commit message (auto-generated if None)
            author: Who made the change ("user", "system", or agent name)

        Returns:
            Commit SHA

        """
        path = self.blocks_dir / f"{label}.toml"
        path.write_text(content)

        # Stage and commit
        rel_path = path.relative_to(self.user_dir)
        self.repo.index.add([str(rel_path)])

        if message is None:
            message = f"Update {label} block"

        commit = self.repo.index.commit(
            f"{message}\n\nAuthor: {author}",
        )

        self.logger.info(
            "block_committed",
            block=label,
            sha=commit.hexsha[:8],
            author=author,
        )

        return commit.hexsha

    def get_block_history(
        self,
        label: str,
        limit: int = 20,
    ) -> list[VersionInfo]:
        """
        Get version history for a block.

        Args:
            label: Block label
            limit: Maximum versions to return

        Returns:
            List of VersionInfo, newest first

        """
        path = self.blocks_dir / f"{label}.toml"
        rel_path = path.relative_to(self.user_dir)

        if not path.exists():
            return []

        versions = []
        for i, commit in enumerate(
            self.repo.iter_commits(paths=str(rel_path), max_count=limit)
        ):
            versions.append(
                VersionInfo(
                    commit_sha=commit.hexsha,
                    message=commit.message.split("\n")[0],
                    author=self._extract_author(commit),
                    timestamp=datetime.fromtimestamp(commit.committed_date),
                    is_current=(i == 0),
                )
            )

        return versions

    def get_block_at_version(self, label: str, commit_sha: str) -> str | None:
        """
        Get block content at a specific version.

        Args:
            label: Block label
            commit_sha: Git commit SHA

        Returns:
            Content at that version or None

        """
        try:
            commit = self.repo.commit(commit_sha)
            rel_path = f"blocks/{label}.toml"
            blob = commit.tree / rel_path
            return blob.data_stream.read().decode("utf-8")
        except Exception as e:
            self.logger.warning(
                "version_read_failed",
                block=label,
                sha=commit_sha[:8],
                error=str(e),
            )
            return None

    def restore_block(
        self,
        label: str,
        commit_sha: str,
        message: str | None = None,
    ) -> str:
        """
        Restore a block to a previous version.

        Args:
            label: Block label
            commit_sha: Version to restore
            message: Commit message

        Returns:
            New commit SHA

        """
        content = self.get_block_at_version(label, commit_sha)
        if content is None:
            raise ValueError(f"Version {commit_sha[:8]} not found for {label}")

        if message is None:
            message = f"Restore {label} to version {commit_sha[:8]}"

        return self.write_block(label, content, message=message, author="user")

    def diff_versions(
        self,
        label: str,
        old_sha: str,
        new_sha: str,
    ) -> FileDiff:
        """
        Get diff between two versions.

        Args:
            label: Block label
            old_sha: Older version SHA
            new_sha: Newer version SHA

        Returns:
            FileDiff with old and new content

        """
        old_content = self.get_block_at_version(label, old_sha) or ""
        new_content = self.get_block_at_version(label, new_sha) or ""

        return FileDiff(
            old_content=old_content,
            new_content=new_content,
            old_sha=old_sha,
            new_sha=new_sha,
        )

    def _extract_author(self, commit: "Commit") -> str:
        """Extract author from commit message footer."""
        lines = commit.message.strip().split("\n")
        for line in lines:
            if line.startswith("Author: "):
                return line.replace("Author: ", "")
        return "unknown"

    def list_blocks(self) -> list[str]:
        """List all block labels in storage."""
        if not self.blocks_dir.exists():
            return []
        return [p.stem for p in self.blocks_dir.glob("*.toml")]


class GitUserStorageManager:
    """
    Factory for GitUserStorage instances.

    Manages the base directory and provides user storage instances.
    """

    def __init__(self, base_dir: Path | str) -> None:
        """
        Initialize the storage manager.

        Args:
            base_dir: Base directory for all user storage

        """
        self.base_dir = Path(base_dir)
        self.base_dir.mkdir(parents=True, exist_ok=True)
        self._cache: dict[str, GitUserStorage] = {}

    def get(self, user_id: str) -> GitUserStorage:
        """Get storage instance for a user (cached)."""
        if user_id not in self._cache:
            self._cache[user_id] = GitUserStorage(user_id, self.base_dir)
        return self._cache[user_id]

    def user_exists(self, user_id: str) -> bool:
        """Check if user storage exists."""
        return self.get(user_id).exists

    def list_users(self) -> list[str]:
        """List all user IDs with storage."""
        return [
            d.name
            for d in self.base_dir.iterdir()
            if d.is_dir() and (d / ".git").exists()
        ]
```

#### 3. Add Storage to Settings
**File**: `src/youlab_server/config/settings.py`

Add to `Settings` class:

```python
# User storage
user_storage_dir: str = Field(
    default=".data/users",
    description="Base directory for user git storage",
)
```

### Success Criteria

#### Automated Verification:
- [ ] `uv sync` installs GitPython without errors
- [ ] `make check-agent` passes (lint + typecheck)
- [ ] Unit tests for GitUserStorage pass

#### Manual Verification:
- [ ] Create storage: `GitUserStorage("test", ".data/users").init()` creates `.data/users/test/.git`
- [ ] Write block creates git commit visible in `git log`
- [ ] Version history returns correct commits
- [ ] Restore creates new commit with old content

**Implementation Note**: After completing this phase and all automated verification passes, pause for manual verification before proceeding.

---

## Phase 2: TOML ↔ Markdown Conversion

### Overview
Create bidirectional converters between TOML (source of truth) and Markdown (user editing format).

### Changes Required

#### 1. Create Conversion Module
**File**: `src/youlab_server/storage/convert.py` (new file)

```python
"""TOML ↔ Markdown conversion for memory blocks."""

from __future__ import annotations

import re
from typing import Any

import tomli
import tomli_w


def toml_to_markdown(toml_content: str, block_label: str) -> str:
    """
    Convert TOML block to Markdown for user editing.

    Format:
        ---
        block: student
        ---

        ## Profile
        Background, CliftonStrengths, aspirations...

        ## Insights
        Key observations from conversation...

    Args:
        toml_content: TOML string
        block_label: Block label for frontmatter

    Returns:
        Markdown string

    """
    try:
        data = tomli.loads(toml_content)
    except tomli.TOMLDecodeError:
        # If invalid TOML, return as-is in code block
        return f"---\nblock: {block_label}\nerror: invalid_toml\n---\n\n```toml\n{toml_content}\n```"

    lines = [
        "---",
        f"block: {block_label}",
        "---",
        "",
    ]

    # Convert each field to a section
    for key, value in data.items():
        # Convert snake_case to Title Case
        title = key.replace("_", " ").title()
        lines.append(f"## {title}")
        lines.append("")

        if isinstance(value, list):
            for item in value:
                lines.append(f"- {item}")
        elif isinstance(value, str):
            # Multi-line strings get their own paragraph
            lines.append(value)
        elif isinstance(value, bool):
            lines.append("Yes" if value else "No")
        elif value is None:
            lines.append("*(not set)*")
        else:
            lines.append(str(value))

        lines.append("")

    return "\n".join(lines)


def markdown_to_toml(markdown_content: str) -> tuple[str, dict[str, Any]]:
    """
    Convert Markdown back to TOML.

    Parses sections and reconstructs structured data.

    Args:
        markdown_content: Markdown string

    Returns:
        Tuple of (toml_string, metadata_dict)

    """
    lines = markdown_content.strip().split("\n")

    # Parse frontmatter
    metadata: dict[str, Any] = {}
    content_start = 0

    if lines and lines[0].strip() == "---":
        for i, line in enumerate(lines[1:], start=1):
            if line.strip() == "---":
                content_start = i + 1
                break
            if ":" in line:
                key, value = line.split(":", 1)
                metadata[key.strip()] = value.strip()

    # Parse sections
    data: dict[str, Any] = {}
    current_section: str | None = None
    current_content: list[str] = []
    current_is_list = False

    for line in lines[content_start:]:
        # New section header
        if line.startswith("## "):
            # Save previous section
            if current_section is not None:
                data[current_section] = _finalize_section(
                    current_content, current_is_list
                )

            # Start new section
            title = line[3:].strip()
            current_section = _title_to_key(title)
            current_content = []
            current_is_list = False
        elif current_section is not None:
            stripped = line.strip()
            if stripped.startswith("- "):
                current_is_list = True
                current_content.append(stripped[2:])
            elif stripped and stripped != "*(not set)*":
                current_content.append(line)

    # Save last section
    if current_section is not None:
        data[current_section] = _finalize_section(current_content, current_is_list)

    toml_str = tomli_w.dumps(data)
    return toml_str, metadata


def _title_to_key(title: str) -> str:
    """Convert Title Case to snake_case."""
    return re.sub(r"\s+", "_", title.lower())


def _finalize_section(content: list[str], is_list: bool) -> str | list[str]:
    """Finalize section content."""
    if is_list:
        return content
    # Join non-list content as paragraphs
    text = "\n".join(content).strip()
    return text if text else ""
```

#### 2. Add tomli-w Dependency
**File**: `pyproject.toml`

```toml
dependencies = [
    # ... existing deps ...
    "tomli-w>=1.0.0",  # TOML writing (tomli is read-only)
]
```

### Success Criteria

#### Automated Verification:
- [x] `uv sync` installs tomli-w
- [x] `make check-agent` passes
- [x] Unit tests for conversion pass

#### Manual Verification:
- [ ] Round-trip: `markdown_to_toml(toml_to_markdown(toml))` produces equivalent TOML
- [ ] Lists convert correctly both ways
- [ ] Multi-line strings preserve formatting

---

## Phase 3: User Initialization Endpoint

### Overview
Create the `/users/init` webhook endpoint that OpenWebUI calls on user signup.

### Changes Required

#### 1. Create Users Router
**File**: `src/youlab_server/server/users.py` (new file)

```python
"""User management endpoints."""

from __future__ import annotations

from typing import TYPE_CHECKING, Any

import structlog
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from youlab_server.curriculum import curriculum

if TYPE_CHECKING:
    from youlab_server.storage.git import GitUserStorageManager

log = structlog.get_logger()
router = APIRouter(prefix="/users", tags=["users"])

# Dependency injection
_storage_manager: GitUserStorageManager | None = None


def get_storage_manager() -> "GitUserStorageManager":
    """Get the storage manager instance."""
    if _storage_manager is None:
        raise HTTPException(status_code=503, detail="Storage not initialized")
    return _storage_manager


def set_storage_manager(manager: "GitUserStorageManager") -> None:
    """Set the storage manager (called on startup)."""
    global _storage_manager
    _storage_manager = manager


class UserInitRequest(BaseModel):
    """Request body for user initialization."""

    user_id: str
    name: str | None = None
    email: str | None = None
    course_id: str = "college-essay"  # Default course


class UserInitResponse(BaseModel):
    """Response for user initialization."""

    user_id: str
    created: bool
    blocks_initialized: list[str]
    message: str


@router.post("/init", response_model=UserInitResponse)
async def init_user(
    request: UserInitRequest,
    storage: "GitUserStorageManager" = Depends(get_storage_manager),
) -> UserInitResponse:
    """
    Initialize user storage and default memory blocks.

    This endpoint is called by OpenWebUI webhook on user signup.
    It is idempotent - calling multiple times is safe.

    Args:
        request: User initialization data

    Returns:
        Initialization result

    """
    user_storage = storage.get(request.user_id)

    # Check if already initialized
    if user_storage.exists:
        log.info("user_already_initialized", user_id=request.user_id)
        return UserInitResponse(
            user_id=request.user_id,
            created=False,
            blocks_initialized=[],
            message="User already initialized",
        )

    # Initialize storage
    user_storage.init()

    # Get course configuration for default blocks
    course = curriculum.get(request.course_id)
    if course is None:
        # Fallback to default course
        course = curriculum.get("default")

    blocks_created = []

    if course:
        # Create default blocks from course schema
        block_registry = curriculum.get_block_registry(request.course_id)

        for block_name, block_schema in course.blocks.items():
            model_class = block_registry.get(block_name) if block_registry else None
            if model_class is None:
                continue

            # Create instance with defaults
            overrides: dict[str, Any] = {}
            if block_schema.label == "human" and request.name:
                # Try to inject user name if block has a name-like field
                for field_name in ["name", "profile"]:
                    if hasattr(model_class, field_name):
                        overrides[field_name] = request.name
                        break

            try:
                instance = model_class(**overrides)
                # Serialize to TOML-like format
                toml_content = _block_to_toml(instance, block_schema)
                user_storage.write_block(
                    label=block_name,
                    content=toml_content,
                    message=f"Initialize {block_name} block",
                    author="system",
                )
                blocks_created.append(block_name)
            except Exception as e:
                log.warning(
                    "block_init_failed",
                    user_id=request.user_id,
                    block=block_name,
                    error=str(e),
                )

    log.info(
        "user_initialized",
        user_id=request.user_id,
        blocks=blocks_created,
    )

    return UserInitResponse(
        user_id=request.user_id,
        created=True,
        blocks_initialized=blocks_created,
        message=f"Initialized {len(blocks_created)} memory blocks",
    )


def _block_to_toml(instance: Any, schema: Any) -> str:
    """Convert a block instance to TOML string."""
    import tomli_w

    # Get field values from instance
    data = {}
    for field_name in instance.model_fields:
        value = getattr(instance, field_name)
        if value is not None and value != "" and value != []:
            data[field_name] = value

    return tomli_w.dumps(data)


@router.get("/{user_id}/exists")
async def user_exists(
    user_id: str,
    storage: "GitUserStorageManager" = Depends(get_storage_manager),
) -> dict[str, bool]:
    """Check if user storage exists."""
    return {"exists": storage.user_exists(user_id)}
```

#### 2. Register Router in Main
**File**: `src/youlab_server/server/main.py`

Add imports and router registration:

```python
from youlab_server.server.users import router as users_router, set_storage_manager
from youlab_server.storage.git import GitUserStorageManager

# In lifespan or startup:
storage_manager = GitUserStorageManager(settings.user_storage_dir)
set_storage_manager(storage_manager)

# Register router:
app.include_router(users_router)
```

#### 3. Add Lazy Init Fallback to Chat Endpoint
**File**: `src/youlab_server/server/main.py`

In the chat endpoint, add a check before processing:

```python
# Ensure user storage exists (fallback for admin-created users)
if not storage_manager.user_exists(user_id):
    from youlab_server.server.users import init_user, UserInitRequest
    await init_user(
        UserInitRequest(user_id=user_id, course_id="college-essay"),
        storage=storage_manager,
    )
```

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes
- [x] API tests for `/users/init` endpoint pass
- [x] Idempotent: calling twice doesn't error

#### Manual Verification:
- [ ] `curl -X POST localhost:8000/users/init -H "Content-Type: application/json" -d '{"user_id":"test123","name":"Test"}'` creates storage
- [ ] `.data/users/test123/blocks/` contains TOML files
- [ ] `git log` in user directory shows initialization commit

---

## Phase 4: UserBlockManager - User-Scoped Blocks

### Overview
Create UserBlockManager that manages user-scoped blocks with Letta sync and pending diff support.

### Changes Required

#### 1. Create Pending Diff Model
**File**: `src/youlab_server/storage/diffs.py` (new file)

```python
"""Pending diff storage for agent-proposed changes."""

from __future__ import annotations

import json
import uuid
from dataclasses import asdict, dataclass, field
from datetime import datetime
from pathlib import Path
from typing import Literal

import structlog

log = structlog.get_logger()


@dataclass
class PendingDiff:
    """A proposed change from an agent awaiting user approval."""

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
    reviewed_at: str | None = None
    applied_commit: str | None = None

    @classmethod
    def create(
        cls,
        user_id: str,
        agent_id: str,
        block_label: str,
        field: str | None,
        operation: str,
        current_value: str,
        proposed_value: str,
        reasoning: str,
        confidence: str = "medium",
        source_query: str | None = None,
    ) -> "PendingDiff":
        """Create a new pending diff."""
        return cls(
            id=str(uuid.uuid4()),
            user_id=user_id,
            agent_id=agent_id,
            block_label=block_label,
            field=field,
            operation=operation,  # type: ignore
            current_value=current_value,
            proposed_value=proposed_value,
            reasoning=reasoning,
            confidence=confidence,  # type: ignore
            source_query=source_query,
            status="pending",
            created_at=datetime.now().isoformat(),
        )


class PendingDiffStore:
    """
    JSON file storage for pending diffs.

    Storage: {user_storage}/pending_diffs/{diff_id}.json
    """

    def __init__(self, diffs_dir: Path) -> None:
        self.diffs_dir = diffs_dir
        self.diffs_dir.mkdir(parents=True, exist_ok=True)

    def _diff_path(self, diff_id: str) -> Path:
        return self.diffs_dir / f"{diff_id}.json"

    def save(self, diff: PendingDiff) -> None:
        """Save a diff to storage."""
        path = self._diff_path(diff.id)
        path.write_text(json.dumps(asdict(diff), indent=2))

    def get(self, diff_id: str) -> PendingDiff | None:
        """Get a diff by ID."""
        path = self._diff_path(diff_id)
        if not path.exists():
            return None
        data = json.loads(path.read_text())
        return PendingDiff(**data)

    def list_pending(self, block_label: str | None = None) -> list[PendingDiff]:
        """List all pending diffs, optionally filtered by block."""
        diffs = []
        for path in self.diffs_dir.glob("*.json"):
            try:
                data = json.loads(path.read_text())
                diff = PendingDiff(**data)
                if diff.status == "pending":
                    if block_label is None or diff.block_label == block_label:
                        diffs.append(diff)
            except Exception:
                continue
        return sorted(diffs, key=lambda d: d.created_at, reverse=True)

    def count_pending(self) -> dict[str, int]:
        """Count pending diffs per block."""
        counts: dict[str, int] = {}
        for diff in self.list_pending():
            counts[diff.block_label] = counts.get(diff.block_label, 0) + 1
        return counts

    def update_status(
        self,
        diff_id: str,
        status: str,
        applied_commit: str | None = None,
    ) -> None:
        """Update diff status."""
        diff = self.get(diff_id)
        if diff:
            diff.status = status  # type: ignore
            diff.reviewed_at = datetime.now().isoformat()
            if applied_commit:
                diff.applied_commit = applied_commit
            self.save(diff)

    def supersede_older(self, block_label: str, keep_id: str) -> int:
        """Mark older pending diffs for a block as superseded."""
        count = 0
        for diff in self.list_pending(block_label):
            if diff.id != keep_id:
                self.update_status(diff.id, "superseded")
                count += 1
        return count
```

#### 2. Create UserBlockManager
**File**: `src/youlab_server/storage/blocks.py` (new file)

```python
"""User-scoped block management with Letta sync."""

from __future__ import annotations

from typing import TYPE_CHECKING, Any

import structlog
import tomli
import tomli_w

from youlab_server.storage.convert import markdown_to_toml, toml_to_markdown
from youlab_server.storage.diffs import PendingDiff, PendingDiffStore

if TYPE_CHECKING:
    from letta_client import Letta

    from youlab_server.storage.git import GitUserStorage

log = structlog.get_logger()


class UserBlockManager:
    """
    Manages memory blocks for a single user.

    Responsibilities:
    - Read/write blocks from git storage
    - Convert between TOML and Markdown
    - Sync blocks to Letta agents
    - Manage pending diffs for agent edits
    """

    def __init__(
        self,
        user_id: str,
        storage: "GitUserStorage",
        letta_client: "Letta | None" = None,
    ) -> None:
        self.user_id = user_id
        self.storage = storage
        self.letta = letta_client
        self.diffs = PendingDiffStore(storage.diffs_dir)
        self.logger = log.bind(user_id=user_id, component="user_block_manager")

    def _letta_block_name(self, label: str) -> str:
        """Generate Letta block name for user-scoped block."""
        return f"youlab_user_{self.user_id}_{label}"

    # =========================================================================
    # Block CRUD Operations
    # =========================================================================

    def list_blocks(self) -> list[str]:
        """List all block labels for this user."""
        return self.storage.list_blocks()

    def get_block_toml(self, label: str) -> str | None:
        """Get block content as TOML."""
        return self.storage.read_block(label)

    def get_block_markdown(self, label: str) -> str | None:
        """Get block content as Markdown for editing."""
        toml_content = self.storage.read_block(label)
        if toml_content is None:
            return None
        return toml_to_markdown(toml_content, label)

    def update_block_from_markdown(
        self,
        label: str,
        markdown: str,
        message: str | None = None,
        sync_to_letta: bool = True,
    ) -> str:
        """
        Update block from Markdown content (user edit).

        Args:
            label: Block label
            markdown: Markdown content
            message: Commit message
            sync_to_letta: Whether to sync to Letta immediately

        Returns:
            Commit SHA

        """
        toml_content, _ = markdown_to_toml(markdown)
        commit_sha = self.storage.write_block(
            label=label,
            content=toml_content,
            message=message or f"Update {label}",
            author="user",
        )

        if sync_to_letta and self.letta:
            self._sync_block_to_letta(label, toml_content)

        return commit_sha

    def update_block_from_toml(
        self,
        label: str,
        toml_content: str,
        message: str | None = None,
        author: str = "user",
        sync_to_letta: bool = True,
    ) -> str:
        """Update block from TOML content."""
        commit_sha = self.storage.write_block(
            label=label,
            content=toml_content,
            message=message or f"Update {label}",
            author=author,
        )

        if sync_to_letta and self.letta:
            self._sync_block_to_letta(label, toml_content)

        return commit_sha

    # =========================================================================
    # Version History
    # =========================================================================

    def get_history(self, label: str, limit: int = 20) -> list[dict[str, Any]]:
        """Get version history for a block."""
        versions = self.storage.get_block_history(label, limit)
        return [
            {
                "sha": v.commit_sha,
                "message": v.message,
                "author": v.author,
                "timestamp": v.timestamp.isoformat(),
                "is_current": v.is_current,
            }
            for v in versions
        ]

    def get_version(self, label: str, commit_sha: str) -> str | None:
        """Get block content at a specific version."""
        return self.storage.get_block_at_version(label, commit_sha)

    def restore_version(
        self,
        label: str,
        commit_sha: str,
        sync_to_letta: bool = True,
    ) -> str:
        """Restore block to a previous version."""
        new_sha = self.storage.restore_block(label, commit_sha)

        if sync_to_letta and self.letta:
            content = self.storage.read_block(label)
            if content:
                self._sync_block_to_letta(label, content)

        return new_sha

    # =========================================================================
    # Letta Sync
    # =========================================================================

    def _sync_block_to_letta(self, label: str, toml_content: str) -> None:
        """Sync block content to Letta."""
        if not self.letta:
            return

        block_name = self._letta_block_name(label)

        # Parse TOML and convert to Letta memory string format
        try:
            data = tomli.loads(toml_content)
            memory_str = self._toml_to_memory_string(data)
        except Exception as e:
            self.logger.warning(
                "toml_parse_failed",
                label=label,
                error=str(e),
            )
            memory_str = toml_content

        # Find or create block in Letta
        try:
            blocks = self.letta.blocks.list()
            existing = next(
                (b for b in blocks if getattr(b, "name", None) == block_name),
                None,
            )

            if existing:
                # Update existing block
                self.letta.blocks.update(
                    block_id=existing.id,
                    value=memory_str,
                )
                self.logger.debug("letta_block_updated", label=label)
            else:
                # Create new block
                self.letta.blocks.create(
                    label=label,
                    name=block_name,
                    value=memory_str,
                )
                self.logger.info("letta_block_created", label=label)

        except Exception as e:
            self.logger.error(
                "letta_sync_failed",
                label=label,
                error=str(e),
            )

    def _toml_to_memory_string(self, data: dict[str, Any]) -> str:
        """Convert TOML data to Letta memory string format."""
        lines = []
        for key, value in data.items():
            if isinstance(value, list):
                items = "\n".join(f"- {item}" for item in value)
                lines.append(f"{key}:\n{items}")
            elif value:
                lines.append(f"{key}: {value}")
        return "\n\n".join(lines)

    def get_or_create_letta_block_id(self, label: str) -> str | None:
        """Get or create a Letta block for this user/label."""
        if not self.letta:
            return None

        block_name = self._letta_block_name(label)

        # Check if exists
        blocks = self.letta.blocks.list()
        existing = next(
            (b for b in blocks if getattr(b, "name", None) == block_name),
            None,
        )

        if existing:
            return existing.id

        # Create from current content
        content = self.storage.read_block(label)
        if content is None:
            return None

        try:
            data = tomli.loads(content)
            memory_str = self._toml_to_memory_string(data)
        except Exception:
            memory_str = content

        block = self.letta.blocks.create(
            label=label,
            name=block_name,
            value=memory_str,
        )
        return block.id

    # =========================================================================
    # Pending Diffs (Agent Edits)
    # =========================================================================

    def propose_edit(
        self,
        agent_id: str,
        block_label: str,
        field: str | None,
        operation: str,
        proposed_value: str,
        reasoning: str,
        confidence: str = "medium",
        source_query: str | None = None,
    ) -> PendingDiff:
        """
        Create a pending diff for an agent-proposed edit.

        This does NOT apply the edit - it creates a diff for user approval.
        """
        current = self.storage.read_block(block_label) or ""

        diff = PendingDiff.create(
            user_id=self.user_id,
            agent_id=agent_id,
            block_label=block_label,
            field=field,
            operation=operation,
            current_value=current,
            proposed_value=proposed_value,
            reasoning=reasoning,
            confidence=confidence,
            source_query=source_query,
        )

        self.diffs.save(diff)
        self.logger.info(
            "diff_proposed",
            diff_id=diff.id,
            block=block_label,
            agent=agent_id,
        )

        return diff

    def approve_diff(self, diff_id: str) -> str:
        """
        Approve and apply a pending diff.

        Returns commit SHA.
        """
        diff = self.diffs.get(diff_id)
        if diff is None:
            raise ValueError(f"Diff {diff_id} not found")
        if diff.status != "pending":
            raise ValueError(f"Diff {diff_id} is not pending (status: {diff.status})")

        # Apply the edit
        # For now, simple replace. TODO: handle append/llm_diff
        commit_sha = self.update_block_from_toml(
            label=diff.block_label,
            toml_content=diff.proposed_value,
            message=f"Apply agent suggestion: {diff.reasoning[:50]}",
            author=f"agent:{diff.agent_id}",
            sync_to_letta=True,
        )

        # Update diff status
        self.diffs.update_status(diff_id, "approved", commit_sha)

        # Supersede older diffs for same block
        superseded = self.diffs.supersede_older(diff.block_label, diff_id)

        self.logger.info(
            "diff_approved",
            diff_id=diff_id,
            commit=commit_sha[:8],
            superseded=superseded,
        )

        return commit_sha

    def reject_diff(self, diff_id: str, reason: str | None = None) -> None:
        """Reject a pending diff."""
        self.diffs.update_status(diff_id, "rejected")
        self.logger.info("diff_rejected", diff_id=diff_id, reason=reason)

    def list_pending_diffs(
        self, block_label: str | None = None
    ) -> list[dict[str, Any]]:
        """List pending diffs as dicts."""
        diffs = self.diffs.list_pending(block_label)
        return [
            {
                "id": d.id,
                "block": d.block_label,
                "field": d.field,
                "operation": d.operation,
                "reasoning": d.reasoning,
                "confidence": d.confidence,
                "created_at": d.created_at,
            }
            for d in diffs
        ]

    def count_pending_diffs(self) -> dict[str, int]:
        """Count pending diffs per block."""
        return self.diffs.count_pending()
```

#### 3. Update edit_memory_block Tool
**File**: `src/youlab_server/tools/memory.py`

Replace the tool to create pending diffs instead of direct edits:

```python
def edit_memory_block(
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
    agent_state: dict[str, Any] | None = None,
) -> str:
    """
    Propose an update to a memory block.

    This creates a pending diff that the user must approve.
    The change is NOT applied immediately.

    Args:
        block: Which memory block to edit (e.g., "student", "journey")
        field: Which field to update
        content: The content to add or replace
        strategy: How to merge: "append", "replace", or "llm_diff"
        reasoning: Explain WHY you're proposing this change
        agent_state: Agent state injected by Letta

    Returns:
        Confirmation that the proposal was created

    """
    from youlab_server.server.users import get_storage_manager
    from youlab_server.storage.blocks import UserBlockManager

    agent_id = agent_state.get("agent_id") if agent_state else None
    user_id = agent_state.get("user_id") if agent_state else None

    if not agent_id or not user_id:
        return "Unable to identify agent or user. Proposal not created."

    try:
        storage_manager = get_storage_manager()
        user_storage = storage_manager.get(user_id)
        manager = UserBlockManager(user_id, user_storage)

        diff = manager.propose_edit(
            agent_id=agent_id,
            block_label=block,
            field=field,
            operation=strategy,
            proposed_value=content,
            reasoning=reasoning or "No reasoning provided",
        )

        return (
            f"Proposed change to {block}.{field} (ID: {diff.id[:8]}). "
            f"The user will review and approve/reject this suggestion."
        )

    except Exception as e:
        log.error("propose_edit_failed", error=str(e))
        return f"Failed to create proposal: {e}"
```

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes
- [x] Unit tests for UserBlockManager pass
- [x] Unit tests for PendingDiffStore pass

#### Manual Verification:
- [ ] User edit creates git commit and syncs to Letta
- [ ] Agent calling `edit_memory_block` creates pending diff (not direct edit)
- [ ] Approving diff applies change and creates commit
- [ ] Rejecting diff updates status without applying

---

## Phase 5: Memory Block CRUD API

### Overview
Create REST API endpoints for memory block operations.

### Changes Required

#### 1. Create Blocks Router
**File**: `src/youlab_server/server/blocks.py` (new file)

```python
"""Memory block CRUD API endpoints."""

from __future__ import annotations

from typing import Any

import structlog
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from youlab_server.server.users import get_storage_manager
from youlab_server.storage.blocks import UserBlockManager
from youlab_server.storage.git import GitUserStorageManager

log = structlog.get_logger()
router = APIRouter(prefix="/users/{user_id}/blocks", tags=["blocks"])


def get_block_manager(
    user_id: str,
    storage: GitUserStorageManager = Depends(get_storage_manager),
) -> UserBlockManager:
    """Get UserBlockManager for a user."""
    user_storage = storage.get(user_id)
    if not user_storage.exists:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")
    return UserBlockManager(user_id, user_storage)


# =============================================================================
# Request/Response Models
# =============================================================================


class BlockSummary(BaseModel):
    """Summary of a memory block."""

    label: str
    pending_diffs: int


class BlockDetail(BaseModel):
    """Detailed block information."""

    label: str
    content_toml: str
    content_markdown: str
    pending_diffs: int


class BlockUpdateRequest(BaseModel):
    """Request to update a block."""

    content: str
    format: str = "markdown"  # "markdown" or "toml"
    message: str | None = None


class BlockUpdateResponse(BaseModel):
    """Response after updating a block."""

    commit_sha: str
    label: str


class VersionInfo(BaseModel):
    """Information about a block version."""

    sha: str
    message: str
    author: str
    timestamp: str
    is_current: bool


class RestoreRequest(BaseModel):
    """Request to restore a previous version."""

    commit_sha: str


class DiffSummary(BaseModel):
    """Summary of a pending diff."""

    id: str
    block: str
    field: str | None
    operation: str
    reasoning: str
    confidence: str
    created_at: str


class DiffDetail(BaseModel):
    """Detailed diff information."""

    id: str
    block: str
    field: str | None
    operation: str
    current_value: str
    proposed_value: str
    reasoning: str
    confidence: str
    created_at: str


class ApproveResponse(BaseModel):
    """Response after approving a diff."""

    diff_id: str
    commit_sha: str


# =============================================================================
# Block Endpoints
# =============================================================================


@router.get("", response_model=list[BlockSummary])
async def list_blocks(
    manager: UserBlockManager = Depends(get_block_manager),
) -> list[BlockSummary]:
    """List all memory blocks for a user."""
    labels = manager.list_blocks()
    counts = manager.count_pending_diffs()

    return [
        BlockSummary(label=label, pending_diffs=counts.get(label, 0))
        for label in labels
    ]


@router.get("/{label}", response_model=BlockDetail)
async def get_block(
    label: str,
    manager: UserBlockManager = Depends(get_block_manager),
) -> BlockDetail:
    """Get a specific memory block."""
    toml_content = manager.get_block_toml(label)
    if toml_content is None:
        raise HTTPException(status_code=404, detail=f"Block {label} not found")

    markdown = manager.get_block_markdown(label) or ""
    counts = manager.count_pending_diffs()

    return BlockDetail(
        label=label,
        content_toml=toml_content,
        content_markdown=markdown,
        pending_diffs=counts.get(label, 0),
    )


@router.put("/{label}", response_model=BlockUpdateResponse)
async def update_block(
    label: str,
    request: BlockUpdateRequest,
    manager: UserBlockManager = Depends(get_block_manager),
) -> BlockUpdateResponse:
    """Update a memory block (user edit)."""
    if request.format == "markdown":
        commit_sha = manager.update_block_from_markdown(
            label=label,
            markdown=request.content,
            message=request.message,
        )
    else:
        commit_sha = manager.update_block_from_toml(
            label=label,
            toml_content=request.content,
            message=request.message,
        )

    return BlockUpdateResponse(commit_sha=commit_sha, label=label)


# =============================================================================
# Version History Endpoints
# =============================================================================


@router.get("/{label}/history", response_model=list[VersionInfo])
async def get_block_history(
    label: str,
    limit: int = 20,
    manager: UserBlockManager = Depends(get_block_manager),
) -> list[VersionInfo]:
    """Get version history for a block."""
    history = manager.get_history(label, limit)
    return [VersionInfo(**v) for v in history]


@router.get("/{label}/versions/{commit_sha}")
async def get_block_version(
    label: str,
    commit_sha: str,
    manager: UserBlockManager = Depends(get_block_manager),
) -> dict[str, str]:
    """Get block content at a specific version."""
    content = manager.get_version(label, commit_sha)
    if content is None:
        raise HTTPException(status_code=404, detail="Version not found")
    return {"content": content, "sha": commit_sha}


@router.post("/{label}/restore", response_model=BlockUpdateResponse)
async def restore_block_version(
    label: str,
    request: RestoreRequest,
    manager: UserBlockManager = Depends(get_block_manager),
) -> BlockUpdateResponse:
    """Restore a block to a previous version."""
    try:
        commit_sha = manager.restore_version(label, request.commit_sha)
        return BlockUpdateResponse(commit_sha=commit_sha, label=label)
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))


# =============================================================================
# Pending Diff Endpoints
# =============================================================================


@router.get("/{label}/diffs", response_model=list[DiffSummary])
async def list_block_diffs(
    label: str,
    manager: UserBlockManager = Depends(get_block_manager),
) -> list[DiffSummary]:
    """List pending diffs for a block."""
    diffs = manager.list_pending_diffs(label)
    return [DiffSummary(**d) for d in diffs]


@router.get("/diffs/counts")
async def get_diff_counts(
    manager: UserBlockManager = Depends(get_block_manager),
) -> dict[str, int]:
    """Get pending diff counts per block."""
    return manager.count_pending_diffs()


@router.post("/diffs/{diff_id}/approve", response_model=ApproveResponse)
async def approve_diff(
    diff_id: str,
    manager: UserBlockManager = Depends(get_block_manager),
) -> ApproveResponse:
    """Approve and apply a pending diff."""
    try:
        commit_sha = manager.approve_diff(diff_id)
        return ApproveResponse(diff_id=diff_id, commit_sha=commit_sha)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@router.post("/diffs/{diff_id}/reject")
async def reject_diff(
    diff_id: str,
    reason: str | None = None,
    manager: UserBlockManager = Depends(get_block_manager),
) -> dict[str, str]:
    """Reject a pending diff."""
    manager.reject_diff(diff_id, reason)
    return {"status": "rejected", "diff_id": diff_id}
```

#### 2. Register Router
**File**: `src/youlab_server/server/main.py`

```python
from youlab_server.server.blocks import router as blocks_router

app.include_router(blocks_router)
```

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes
- [x] API integration tests pass
- [x] OpenAPI docs generate correctly

#### Manual Verification:
- [ ] `GET /users/{id}/blocks` returns block list
- [ ] `GET /users/{id}/blocks/{label}` returns block with markdown
- [ ] `PUT /users/{id}/blocks/{label}` updates and creates commit
- [ ] `GET /users/{id}/blocks/{label}/history` returns version list
- [ ] `POST /users/{id}/blocks/{label}/restore` creates new commit with old content

---

## Phase 6: Frontend - "You" Section

### Overview
Add the "You" section to OpenWebUI's sidebar with profile tab and block editing.

### Changes Required

#### 1. Add "You" Menu Item to Sidebar
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`

Add after the Notes/Knowledge items:

```svelte
<!-- You Section -->
<a
    class="px-2.5 py-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-850 flex items-center gap-2"
    href="/you"
>
    <User class="size-4" />
    <span>You</span>
    {#if $pendingDiffsCount > 0}
        <Badge type="error" size="sm">{$pendingDiffsCount}</Badge>
    {/if}
</a>
```

Add store import and subscription:

```svelte
<script>
    import { pendingDiffsCount } from '$lib/stores/memory';
    // ... existing imports
</script>
```

#### 2. Create Memory Store
**File**: `OpenWebUI/open-webui/src/lib/stores/memory.ts` (new file)

```typescript
import { writable, derived } from 'svelte/store';

export interface MemoryBlock {
    label: string;
    pendingDiffs: number;
}

export interface BlockDetail {
    label: string;
    contentToml: string;
    contentMarkdown: string;
    pendingDiffs: number;
}

export interface VersionInfo {
    sha: string;
    message: string;
    author: string;
    timestamp: string;
    isCurrent: boolean;
}

// Store for all blocks
export const memoryBlocks = writable<MemoryBlock[]>([]);

// Derived store for total pending diffs
export const pendingDiffsCount = derived(
    memoryBlocks,
    ($blocks) => $blocks.reduce((sum, b) => sum + b.pendingDiffs, 0)
);

// Currently selected block
export const selectedBlock = writable<BlockDetail | null>(null);

// Version history for selected block
export const blockHistory = writable<VersionInfo[]>([]);
```

#### 3. Create Memory API Functions
**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts` (new file)

```typescript
import { YOULAB_API_BASE_URL } from '$lib/constants';

export async function getBlocks(userId: string, token: string) {
    const res = await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/blocks`, {
        headers: { Authorization: `Bearer ${token}` },
    });
    if (!res.ok) throw new Error('Failed to fetch blocks');
    return res.json();
}

export async function getBlock(userId: string, label: string, token: string) {
    const res = await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}`, {
        headers: { Authorization: `Bearer ${token}` },
    });
    if (!res.ok) throw new Error('Failed to fetch block');
    return res.json();
}

export async function updateBlock(
    userId: string,
    label: string,
    content: string,
    token: string,
    format: 'markdown' | 'toml' = 'markdown'
) {
    const res = await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}`, {
        method: 'PUT',
        headers: {
            Authorization: `Bearer ${token}`,
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ content, format }),
    });
    if (!res.ok) throw new Error('Failed to update block');
    return res.json();
}

export async function getBlockHistory(userId: string, label: string, token: string) {
    const res = await fetch(
        `${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}/history`,
        { headers: { Authorization: `Bearer ${token}` } }
    );
    if (!res.ok) throw new Error('Failed to fetch history');
    return res.json();
}

export async function restoreVersion(
    userId: string,
    label: string,
    commitSha: string,
    token: string
) {
    const res = await fetch(
        `${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}/restore`,
        {
            method: 'POST',
            headers: {
                Authorization: `Bearer ${token}`,
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ commit_sha: commitSha }),
        }
    );
    if (!res.ok) throw new Error('Failed to restore version');
    return res.json();
}

export async function approveDiff(userId: string, diffId: string, token: string) {
    const res = await fetch(
        `${YOULAB_API_BASE_URL}/users/${userId}/blocks/diffs/${diffId}/approve`,
        {
            method: 'POST',
            headers: { Authorization: `Bearer ${token}` },
        }
    );
    if (!res.ok) throw new Error('Failed to approve diff');
    return res.json();
}

export async function rejectDiff(userId: string, diffId: string, token: string) {
    const res = await fetch(
        `${YOULAB_API_BASE_URL}/users/${userId}/blocks/diffs/${diffId}/reject`,
        {
            method: 'POST',
            headers: { Authorization: `Bearer ${token}` },
        }
    );
    if (!res.ok) throw new Error('Failed to reject diff');
    return res.json();
}
```

#### 4. Create You Page Route
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte` (new file)

```svelte
<script lang="ts">
    import { onMount } from 'svelte';
    import { user, config } from '$lib/stores';
    import { getBlocks } from '$lib/apis/memory';
    import { memoryBlocks } from '$lib/stores/memory';
    import BlockCard from '$lib/components/you/BlockCard.svelte';
    import BlockDetailModal from '$lib/components/you/BlockDetailModal.svelte';

    let loading = true;
    let selectedLabel: string | null = null;

    onMount(async () => {
        if ($user) {
            try {
                const blocks = await getBlocks($user.id, localStorage.token);
                memoryBlocks.set(blocks);
            } catch (e) {
                console.error('Failed to load blocks:', e);
            }
        }
        loading = false;
    });

    function openBlock(label: string) {
        selectedLabel = label;
    }

    function closeModal() {
        selectedLabel = null;
    }
</script>

<div class="min-h-screen max-h-screen w-full flex flex-col">
    <!-- Header -->
    <div class="px-4 pt-3 pb-2.5 flex items-center justify-between">
        <h1 class="text-xl font-semibold">You</h1>
    </div>

    <!-- Content -->
    <div class="flex-1 overflow-y-auto px-4 pb-4">
        {#if loading}
            <div class="flex justify-center py-8">
                <div class="animate-spin size-6 border-2 border-gray-300 border-t-primary rounded-full" />
            </div>
        {:else if $memoryBlocks.length === 0}
            <div class="text-center text-gray-500 py-8">
                No memory blocks found. Start a conversation to begin building your profile.
            </div>
        {:else}
            <div class="grid gap-3 sm:grid-cols-2 lg:grid-cols-3">
                {#each $memoryBlocks as block}
                    <BlockCard
                        label={block.label}
                        pendingDiffs={block.pendingDiffs}
                        on:click={() => openBlock(block.label)}
                    />
                {/each}
            </div>
        {/if}
    </div>
</div>

{#if selectedLabel}
    <BlockDetailModal
        label={selectedLabel}
        on:close={closeModal}
    />
{/if}
```

#### 5. Create BlockCard Component
**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockCard.svelte` (new file)

```svelte
<script lang="ts">
    import { createEventDispatcher } from 'svelte';
    import Badge from '$lib/components/common/Badge.svelte';
    import Document from '$lib/components/icons/Document.svelte';

    export let label: string;
    export let pendingDiffs: number = 0;

    const dispatch = createEventDispatcher();

    // Convert snake_case to Title Case
    $: displayName = label.replace(/_/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
</script>

<button
    class="text-left w-full px-4 py-3 rounded-xl border border-gray-200 dark:border-gray-800
           hover:bg-gray-50 dark:hover:bg-gray-850 transition cursor-pointer"
    on:click={() => dispatch('click')}
>
    <div class="flex items-start justify-between">
        <div class="flex items-center gap-2">
            <Document class="size-5 text-gray-500" />
            <span class="font-medium">{displayName}</span>
        </div>
        {#if pendingDiffs > 0}
            <Badge type="warning" size="sm">{pendingDiffs} pending</Badge>
        {/if}
    </div>
</button>
```

#### 6. Create BlockDetailModal Component
**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte` (new file)

```svelte
<script lang="ts">
    import { createEventDispatcher, onMount } from 'svelte';
    import { user } from '$lib/stores';
    import { getBlock, updateBlock, getBlockHistory, restoreVersion } from '$lib/apis/memory';
    import { selectedBlock, blockHistory } from '$lib/stores/memory';
    import Modal from '$lib/components/common/Modal.svelte';
    import Textarea from '$lib/components/common/Textarea.svelte';

    export let label: string;

    const dispatch = createEventDispatcher();

    let loading = true;
    let saving = false;
    let content = '';
    let showHistory = false;
    let error = '';

    onMount(async () => {
        await loadBlock();
    });

    async function loadBlock() {
        loading = true;
        error = '';
        try {
            const block = await getBlock($user.id, label, localStorage.token);
            selectedBlock.set(block);
            content = block.contentMarkdown;

            const history = await getBlockHistory($user.id, label, localStorage.token);
            blockHistory.set(history);
        } catch (e) {
            error = 'Failed to load block';
            console.error(e);
        }
        loading = false;
    }

    async function save() {
        saving = true;
        error = '';
        try {
            await updateBlock($user.id, label, content, localStorage.token);
            await loadBlock();
        } catch (e) {
            error = 'Failed to save';
            console.error(e);
        }
        saving = false;
    }

    async function restore(sha: string) {
        if (!confirm('Restore this version? Current content will be replaced.')) return;

        saving = true;
        try {
            await restoreVersion($user.id, label, sha, localStorage.token);
            await loadBlock();
        } catch (e) {
            error = 'Failed to restore';
            console.error(e);
        }
        saving = false;
    }

    // Convert snake_case to Title Case
    $: displayName = label.replace(/_/g, ' ').replace(/\b\w/g, c => c.toUpperCase());
</script>

<Modal size="lg" on:close={() => dispatch('close')}>
    <div class="p-6">
        <!-- Header -->
        <div class="flex items-center justify-between mb-4">
            <h2 class="text-xl font-semibold">{displayName}</h2>
            <div class="flex gap-2">
                <button
                    class="text-sm text-gray-500 hover:text-gray-700"
                    on:click={() => showHistory = !showHistory}
                >
                    {showHistory ? 'Hide' : 'Show'} History
                </button>
            </div>
        </div>

        {#if loading}
            <div class="flex justify-center py-8">
                <div class="animate-spin size-6 border-2 border-gray-300 border-t-primary rounded-full" />
            </div>
        {:else}
            {#if error}
                <div class="text-red-500 mb-4">{error}</div>
            {/if}

            <div class="flex gap-4">
                <!-- Editor -->
                <div class="flex-1">
                    <Textarea
                        bind:value={content}
                        rows={16}
                        placeholder="Memory block content (Markdown)..."
                        class="font-mono text-sm"
                    />

                    <div class="mt-4 flex justify-end gap-2">
                        <button
                            class="px-4 py-2 text-gray-600 hover:bg-gray-100 rounded-lg"
                            on:click={() => dispatch('close')}
                        >
                            Cancel
                        </button>
                        <button
                            class="px-4 py-2 bg-primary text-white rounded-lg disabled:opacity-50"
                            disabled={saving}
                            on:click={save}
                        >
                            {saving ? 'Saving...' : 'Save'}
                        </button>
                    </div>
                </div>

                <!-- History Sidebar -->
                {#if showHistory}
                    <div class="w-64 border-l pl-4">
                        <h3 class="font-medium mb-2">Version History</h3>
                        <div class="space-y-2 max-h-96 overflow-y-auto">
                            {#each $blockHistory as version}
                                <div class="text-sm p-2 rounded hover:bg-gray-50 dark:hover:bg-gray-850">
                                    <div class="font-medium truncate">{version.message}</div>
                                    <div class="text-gray-500 text-xs">
                                        {version.author} · {new Date(version.timestamp).toLocaleDateString()}
                                    </div>
                                    {#if !version.isCurrent}
                                        <button
                                            class="text-xs text-primary hover:underline mt-1"
                                            on:click={() => restore(version.sha)}
                                        >
                                            Restore
                                        </button>
                                    {:else}
                                        <span class="text-xs text-gray-400">Current</span>
                                    {/if}
                                </div>
                            {/each}
                        </div>
                    </div>
                {/if}
            </div>
        {/if}
    </div>
</Modal>
```

### Success Criteria

#### Automated Verification:
- [x] TypeScript compiles without errors
- [x] `npm run build` succeeds in OpenWebUI directory

#### Manual Verification:
- [x] "You" menu item appears in sidebar
- [x] Clicking "You" shows block cards (Profile tab)
- [x] Clicking a card opens detail modal with markdown content
- [x] Editing and saving creates new version (mock data fallback working)
- [x] Version history shows commits (mock data fallback working)
- [x] Restore button replaces content with old version (mock data fallback working)
- [x] Badge shows pending diff count when diffs exist

---

## Phase 7: Frontend - "You" Section Agents Tab

### Overview
Add the Agents tab to the "You" section, showing background agents with their thread history. This completes the two-tab structure specified in the product spec (Profile + Agents).

Per spec section 2.3-2.4:
- **Tab 2: Agents** shows background agents grouped by agent name
- Each agent row shows a list of thread runs with dates
- Thread naming: `{Agent Name} - {Date}` (e.g., "Insight Synthesizer - Jan 12, 2025")
- Clicking a thread navigates to that OpenWebUI chat
- Badge on agent row shows pending diff count

### Changes Required

#### 1. Add Tab Switching to You Page
**File**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`

Update to support Profile/Agents tabs using state-driven tab pattern (like SettingsModal.svelte):

```svelte
<script lang="ts">
    import { onMount } from 'svelte';
    import { user } from '$lib/stores';
    import { getBlocks } from '$lib/apis/memory';
    import { getBackgroundAgentThreads } from '$lib/apis/memory';
    import { memoryBlocks } from '$lib/stores/memory';
    import BlockCard from '$lib/components/you/BlockCard.svelte';
    import BlockDetailModal from '$lib/components/you/BlockDetailModal.svelte';
    import AgentsTab from '$lib/components/you/AgentsTab.svelte';
    import User from '$lib/components/icons/User.svelte';
    import Bot from '$lib/components/icons/Bot.svelte';

    let loading = true;
    let selectedLabel: string | null = null;
    let selectedTab: 'profile' | 'agents' = 'profile';

    onMount(async () => {
        if ($user) {
            try {
                const blocks = await getBlocks($user.id, localStorage.token);
                memoryBlocks.set(blocks);
            } catch (e) {
                console.error('Failed to load blocks:', e);
            }
        }
        loading = false;
    });

    function openBlock(label: string) {
        selectedLabel = label;
    }

    function closeModal() {
        selectedLabel = null;
    }
</script>

<div class="min-h-screen max-h-screen w-full flex flex-col">
    <!-- Header with tabs -->
    <div class="px-4 pt-3 pb-2.5 flex items-center justify-between">
        <h1 class="text-xl font-semibold">You</h1>
    </div>

    <!-- Tab bar -->
    <div class="px-4 pb-2">
        <div role="tablist" class="flex gap-2">
            <button
                role="tab"
                aria-selected={selectedTab === 'profile'}
                class="px-3 py-1.5 rounded-lg flex items-center gap-2 transition
                    {selectedTab === 'profile'
                        ? 'bg-gray-100 dark:bg-gray-800'
                        : 'text-gray-500 hover:text-gray-700 dark:hover:text-gray-300'}"
                on:click={() => selectedTab = 'profile'}
            >
                <User class="size-4" />
                <span>Profile</span>
            </button>
            <button
                role="tab"
                aria-selected={selectedTab === 'agents'}
                class="px-3 py-1.5 rounded-lg flex items-center gap-2 transition
                    {selectedTab === 'agents'
                        ? 'bg-gray-100 dark:bg-gray-800'
                        : 'text-gray-500 hover:text-gray-700 dark:hover:text-gray-300'}"
                on:click={() => selectedTab = 'agents'}
            >
                <Bot class="size-4" />
                <span>Agents</span>
            </button>
        </div>
    </div>

    <!-- Tab content -->
    <div class="flex-1 overflow-y-auto px-4 pb-4">
        {#if selectedTab === 'profile'}
            {#if loading}
                <div class="flex justify-center py-8">
                    <div class="animate-spin size-6 border-2 border-gray-300 border-t-primary rounded-full" />
                </div>
            {:else if $memoryBlocks.length === 0}
                <div class="text-center text-gray-500 py-8">
                    No memory blocks found. Start a conversation to begin building your profile.
                </div>
            {:else}
                <div class="grid gap-3 sm:grid-cols-2 lg:grid-cols-3">
                    {#each $memoryBlocks as block}
                        <BlockCard
                            label={block.label}
                            pendingDiffs={block.pendingDiffs}
                            on:click={() => openBlock(block.label)}
                        />
                    {/each}
                </div>
            {/if}
        {:else if selectedTab === 'agents'}
            <AgentsTab />
        {/if}
    </div>
</div>

{#if selectedLabel}
    <BlockDetailModal
        label={selectedLabel}
        on:close={closeModal}
    />
{/if}
```

#### 2. Create AgentsTab Component
**File**: `OpenWebUI/open-webui/src/lib/components/you/AgentsTab.svelte` (new file)

```svelte
<script lang="ts">
    import { onMount } from 'svelte';
    import { user } from '$lib/stores';
    import { getBackgroundAgents } from '$lib/apis/memory';
    import Collapsible from '$lib/components/common/Collapsible.svelte';
    import Badge from '$lib/components/common/Badge.svelte';
    import Bot from '$lib/components/icons/Bot.svelte';
    import ChevronRight from '$lib/components/icons/ChevronRight.svelte';
    import ChevronDown from '$lib/components/icons/ChevronDown.svelte';

    interface ThreadRun {
        id: string;
        chatId: string;
        date: string;
        displayDate: string;
    }

    interface BackgroundAgent {
        name: string;
        pendingDiffs: number;
        threads: ThreadRun[];
    }

    let agents: BackgroundAgent[] = [];
    let loading = true;
    let error = '';
    let expandedAgents: Set<string> = new Set();

    onMount(async () => {
        await loadAgents();
    });

    async function loadAgents() {
        loading = true;
        error = '';
        try {
            agents = await getBackgroundAgents($user.id, localStorage.token);
        } catch (e) {
            error = 'Failed to load agents';
            console.error(e);
        }
        loading = false;
    }

    function toggleAgent(name: string) {
        if (expandedAgents.has(name)) {
            expandedAgents.delete(name);
        } else {
            expandedAgents.add(name);
        }
        expandedAgents = expandedAgents; // trigger reactivity
    }

    function navigateToThread(chatId: string) {
        window.location.href = `/c/${chatId}`;
    }
</script>

<div class="space-y-2">
    <h2 class="text-sm font-medium text-gray-500 dark:text-gray-400 mb-3">
        Background Agents
    </h2>

    {#if loading}
        <div class="flex justify-center py-8">
            <div class="animate-spin size-6 border-2 border-gray-300 border-t-primary rounded-full" />
        </div>
    {:else if error}
        <div class="text-red-500 text-center py-4">{error}</div>
    {:else if agents.length === 0}
        <div class="text-center text-gray-500 py-8">
            No background agents have run yet.
        </div>
    {:else}
        {#each agents as agent}
            <div class="border border-gray-200 dark:border-gray-800 rounded-xl overflow-hidden">
                <!-- Agent header (collapsible trigger) -->
                <button
                    class="w-full px-4 py-3 flex items-center justify-between
                           hover:bg-gray-50 dark:hover:bg-gray-850 transition"
                    on:click={() => toggleAgent(agent.name)}
                >
                    <div class="flex items-center gap-3">
                        <div class="p-1.5 rounded-lg bg-gray-100 dark:bg-gray-800">
                            <Bot class="size-4 text-gray-600 dark:text-gray-400" />
                        </div>
                        <span class="font-medium">{agent.name}</span>
                        {#if agent.pendingDiffs > 0}
                            <Badge type="warning" size="sm">{agent.pendingDiffs}</Badge>
                        {/if}
                    </div>
                    <div class="text-gray-400">
                        {#if expandedAgents.has(agent.name)}
                            <ChevronDown class="size-4" />
                        {:else}
                            <ChevronRight class="size-4" />
                        {/if}
                    </div>
                </button>

                <!-- Thread list (expanded content) -->
                {#if expandedAgents.has(agent.name)}
                    <div class="border-t border-gray-200 dark:border-gray-800">
                        {#if agent.threads.length === 0}
                            <div class="px-4 py-3 text-gray-500 text-sm">
                                No thread runs yet
                            </div>
                        {:else}
                            {#each agent.threads as thread, idx}
                                <a
                                    href="/c/{thread.chatId}"
                                    class="block px-4 py-2.5 hover:bg-gray-50 dark:hover:bg-gray-850
                                           {idx > 0 ? 'border-t border-gray-100 dark:border-gray-850' : ''}"
                                >
                                    <div class="flex items-center justify-between">
                                        <span class="text-sm">
                                            {agent.name} - {thread.displayDate}
                                        </span>
                                        {#if idx === 0}
                                            <span class="text-xs text-gray-400">(latest)</span>
                                        {/if}
                                    </div>
                                </a>
                            {/each}
                        {/if}
                    </div>
                {/if}
            </div>
        {/each}
    {/if}
</div>
```

#### 3. Add Backend API for Background Agent Threads
**File**: `src/youlab_server/server/agents_threads.py` (new file)

```python
"""Background agent thread management endpoints."""

from __future__ import annotations

from datetime import datetime
from typing import Any

import structlog
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from youlab_server.server.users import get_storage_manager
from youlab_server.storage.git import GitUserStorageManager

log = structlog.get_logger()
router = APIRouter(prefix="/users/{user_id}/agents", tags=["agents"])


class ThreadRun(BaseModel):
    """A single background agent thread run."""

    id: str
    chat_id: str
    date: str
    display_date: str


class BackgroundAgent(BaseModel):
    """Background agent with thread history."""

    name: str
    pending_diffs: int
    threads: list[ThreadRun]


@router.get("", response_model=list[BackgroundAgent])
async def list_background_agents(
    user_id: str,
    storage: GitUserStorageManager = Depends(get_storage_manager),
) -> list[BackgroundAgent]:
    """
    List background agents with their thread history.

    Returns agents grouped by name, each with:
    - List of thread runs (OpenWebUI chat IDs)
    - Pending diff count for this agent
    - Threads sorted by date (newest first)
    """
    user_storage = storage.get(user_id)
    if not user_storage.exists:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")

    # Get pending diffs to count per agent
    from youlab_server.storage.blocks import UserBlockManager

    manager = UserBlockManager(user_id, user_storage)
    pending_diffs = manager.list_pending_diffs()

    # Group diffs by agent
    agent_diffs: dict[str, int] = {}
    for diff in pending_diffs:
        agent_id = diff.get("agent_id", "unknown")
        agent_diffs[agent_id] = agent_diffs.get(agent_id, 0) + 1

    # Get background agent threads from storage
    # Thread metadata stored in user's git storage: threads/{agent_name}/{chat_id}.json
    agents_dir = user_storage.user_dir / "agent_threads"
    agents: list[BackgroundAgent] = []

    if agents_dir.exists():
        import json

        for agent_dir in agents_dir.iterdir():
            if not agent_dir.is_dir():
                continue

            agent_name = agent_dir.name.replace("_", " ").title()
            threads: list[ThreadRun] = []

            for thread_file in sorted(
                agent_dir.glob("*.json"),
                key=lambda p: p.stat().st_mtime,
                reverse=True,
            ):
                try:
                    data = json.loads(thread_file.read_text())
                    run_date = datetime.fromisoformat(data.get("created_at", ""))
                    threads.append(
                        ThreadRun(
                            id=thread_file.stem,
                            chat_id=data.get("chat_id", thread_file.stem),
                            date=data.get("created_at", ""),
                            display_date=run_date.strftime("%b %d, %Y"),
                        )
                    )
                except Exception as e:
                    log.warning(
                        "thread_parse_failed",
                        agent=agent_name,
                        file=thread_file.name,
                        error=str(e),
                    )

            agents.append(
                BackgroundAgent(
                    name=agent_name,
                    pending_diffs=agent_diffs.get(agent_dir.name, 0),
                    threads=threads,
                )
            )

    # Sort agents by name
    agents.sort(key=lambda a: a.name)

    return agents


@router.post("/{agent_name}/threads")
async def register_agent_thread(
    user_id: str,
    agent_name: str,
    chat_id: str,
    storage: GitUserStorageManager = Depends(get_storage_manager),
) -> dict[str, str]:
    """
    Register a new background agent thread run.

    Called when a background agent job starts to create the thread mapping.
    """
    import json

    user_storage = storage.get(user_id)
    if not user_storage.exists:
        raise HTTPException(status_code=404, detail=f"User {user_id} not found")

    # Create agent threads directory
    agent_dir = user_storage.user_dir / "agent_threads" / agent_name.lower().replace(" ", "_")
    agent_dir.mkdir(parents=True, exist_ok=True)

    # Save thread metadata
    thread_file = agent_dir / f"{chat_id}.json"
    thread_file.write_text(
        json.dumps(
            {
                "chat_id": chat_id,
                "created_at": datetime.now().isoformat(),
                "agent_name": agent_name,
            },
            indent=2,
        )
    )

    log.info(
        "agent_thread_registered",
        user_id=user_id,
        agent=agent_name,
        chat_id=chat_id,
    )

    return {"status": "registered", "chat_id": chat_id}
```

#### 4. Register Router in Main
**File**: `src/youlab_server/server/main.py`

Add import and router registration:

```python
from youlab_server.server.agents_threads import router as agents_router

# In router registration section:
app.include_router(agents_router)
```

#### 5. Add Frontend API Function
**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`

Add to existing memory API:

```typescript
export interface ThreadRun {
    id: string;
    chatId: string;
    date: string;
    displayDate: string;
}

export interface BackgroundAgent {
    name: string;
    pendingDiffs: number;
    threads: ThreadRun[];
}

export async function getBackgroundAgents(
    userId: string,
    token: string
): Promise<BackgroundAgent[]> {
    const res = await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/agents`, {
        headers: { Authorization: `Bearer ${token}` },
    });
    if (!res.ok) throw new Error('Failed to fetch background agents');

    const data = await res.json();

    // Transform snake_case to camelCase
    return data.map((agent: any) => ({
        name: agent.name,
        pendingDiffs: agent.pending_diffs,
        threads: agent.threads.map((t: any) => ({
            id: t.id,
            chatId: t.chat_id,
            date: t.date,
            displayDate: t.display_date,
        })),
    }));
}
```

### Implementation Notes

**Completed 2026-01-12:**
- Frontend components (You page with tabs, AgentsTab) were already implemented in Phase 6 with mock data fallbacks
- Backend `agents_threads.py` API implemented using `GitUserStorageManager` and `UserBlockManager` from Phases 1-5
- Properly integrated with storage layer for pending diffs counting via `UserBlockManager.list_pending_diffs()`
- Thread metadata stored in: `{user_storage_dir}/users/{user_id}/agent_threads/{agent_name}/{chat_id}.json`
- Returns 404 if user not found, empty list if user exists but has no agent threads

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes (Python lint + typecheck)
- [x] TypeScript compiles without errors in OpenWebUI
- [x] `npm run build` succeeds in OpenWebUI directory

#### Manual Verification:
- [ ] "You" section shows two tabs: Profile and Agents
- [ ] Clicking "Profile" tab shows memory block cards
- [ ] Clicking "Agents" tab shows background agents list
- [ ] Each agent row is collapsible (click to expand/collapse)
- [ ] Expanded agent shows thread runs with dates (e.g., "Insight Synthesizer - Jan 12, 2025")
- [ ] Clicking a thread navigates to `/c/{chatId}` (OpenWebUI chat)
- [ ] Badge on agent row shows pending diff count when diffs exist
- [ ] Empty state shown when no agents have run

---

## Testing Strategy

### Unit Tests

**Git Storage** (`tests/test_storage/test_git.py`):
- `test_init_creates_repo`
- `test_write_block_creates_commit`
- `test_get_history_returns_versions`
- `test_restore_creates_new_commit`

**TOML/Markdown Conversion** (`tests/test_storage/test_convert.py`):
- `test_toml_to_markdown_basic`
- `test_markdown_to_toml_basic`
- `test_roundtrip_preserves_data`
- `test_list_conversion`

**UserBlockManager** (`tests/test_storage/test_blocks.py`):
- `test_list_blocks`
- `test_update_creates_commit`
- `test_propose_edit_creates_diff`
- `test_approve_diff_applies_change`

### Integration Tests

**API Tests** (`tests/test_server/test_blocks_api.py`):
- `test_list_blocks_endpoint`
- `test_get_block_returns_markdown`
- `test_update_block_creates_commit`
- `test_restore_version`
- `test_approve_diff`

### Manual Testing Steps

1. **User Signup Flow**:
   - Create new user in OpenWebUI
   - Verify `.data/users/{id}/` created with git repo
   - Check default blocks exist

2. **Block Editing**:
   - Open "You" section
   - Edit a block in markdown
   - Save and verify git commit created
   - Check Letta block updated

3. **Version History**:
   - Make multiple edits
   - View history
   - Restore older version
   - Verify new commit created

4. **Agent Diff Workflow**:
   - Chat with agent that triggers `edit_memory_block`
   - Verify pending diff created (not applied)
   - Approve in UI
   - Verify change applied and committed

---

## Performance Considerations

- **Git operations**: GitPython is synchronous. For production, consider async wrappers or background tasks for large repos.
- **Letta sync**: Sync on every edit may be slow. Consider debouncing or batch sync.
- **Block listing**: Cache block list in memory, invalidate on write.

---

## Migration Notes

- **Existing users**: No migration needed. Storage initialized on first access via lazy init fallback.
- **Existing Letta blocks**: Course-scoped blocks remain. New user-scoped blocks coexist.
- **Deprecated modules**: `memory/manager.py`, `memory/blocks.py` remain for backwards compatibility but are not used by new code.

---

## References

- Linear ticket: ARI-80
- Parent ticket: ARI-78 (research comments contain detailed technical analysis)
- Related research:
  - `thoughts/shared/research/2026-01-12-ARI-78-unified-user-memory-blocks.md`
  - `thoughts/shared/research/2025-01-12-ARI-78-user-onboarding-initialization.md`
- Current implementation:
  - `src/youlab_server/server/agents.py:57-116` (shared block logic)
  - `src/youlab_server/tools/memory.py` (edit_memory_block tool)
  - `OpenWebUI/open-webui/src/lib/components/chat/Settings/Personalization/EditMemoryModal.svelte` (existing modal pattern)
