# ARI-82 Phase 1: Storage Format Migration (TOML → MD + YAML Frontmatter)

## Overview

Migrate memory block storage from TOML files to Markdown with YAML frontmatter. This eliminates the lossy TOML↔MD conversion layer and makes markdown the source of truth.

## Current State Analysis

### What Exists

| Component | Location | Current Behavior |
|-----------|----------|------------------|
| GitUserStorage | `storage/git.py:41-326` | Reads/writes `.toml` files in `blocks/` directory |
| UserBlockManager | `storage/blocks.py:21-395` | Converts TOML↔MD on every read/write, syncs to Letta |
| Converter | `storage/convert.py` | Forces section headers (`## Field`) even for freeform content |
| Server API | `server/blocks.py` | Returns both `content_toml` and `content_markdown` |

### Key Discoveries

1. **Conversion is lossy**: Booleans become strings, numbers become strings, empty fields are lost (`storage/convert.py:107-113` research doc)
2. **Section headers forced**: Single `body` field becomes awkward `## Body` section
3. **Double conversion on every edit**: User edits MD → convert to TOML → store → convert back to MD
4. **Letta receives parsed TOML**: `_toml_to_memory_string()` at `blocks.py:213-222`

### Files to Modify

| File | Lines | Changes |
|------|-------|---------|
| `src/youlab_server/storage/git.py` | 41-326 | New directory, `.md` file handling, YAML frontmatter parsing |
| `src/youlab_server/storage/blocks.py` | 21-395 | Remove conversion, direct MD read/write, update Letta sync |
| `src/youlab_server/storage/convert.py` | 1-148 | Delete or keep only for reference |
| `src/youlab_server/server/blocks.py` | 1-274 | Simplify API responses to MD-only |
| `tests/test_storage/test_blocks.py` | 1-418 | Update to use MD format |
| `tests/test_storage/test_convert.py` | 1-261 | Delete (no longer needed) |
| `tests/test_server/test_blocks_api.py` | 1-478 | Update to use MD format |

## Desired End State

After implementation:

1. **Memory blocks stored as `.md` files** in `memory-blocks/` directory
2. **YAML frontmatter** contains metadata (block label, optional schema, timestamp)
3. **Body is freeform markdown** - no forced section headers
4. **No TOML↔MD conversion** - markdown is the source of truth
5. **Letta receives raw markdown** body content

### File Format

```markdown
---
block: student
schema: college-essay/student
updated_at: 2026-01-13T10:00:00Z
---

Background and context about the student...

## Strengths
- Creativity
- Communication

## Goals
Working on college essays for top universities.
```

### Directory Structure

```
.data/{user-id}/
  memory-blocks/
    student.md
    journey.md
    engagement_strategy.md
  pending_diffs/
    {diff_id}.json
  agent_threads/
    {agent_name}/
      {chat_id}.json
```

### Verification

```bash
# Check file format
cat .data/test-user/memory-blocks/student.md
# Expected: YAML frontmatter + markdown body

# Verify git history works
git -C .data/test-user log --oneline memory-blocks/student.md
# Expected: commit history preserved

# API returns markdown
curl -s localhost:8000/users/test-user/blocks/student | jq '.content'
# Expected: raw markdown content
```

## What We're NOT Doing

1. **No backward compatibility** - manually convert existing TOML files
2. **No migration script** - few existing files, convert by hand
3. **No TOML support in API** - markdown-only going forward
4. **No structured field parsing** - freeform markdown is the format

## Implementation Approach

Bottom-up: Storage layer first, then manager, then API, then tests.

---

## Phase 1.1: GitUserStorage - Markdown Native

### Overview

Update GitUserStorage to read/write `.md` files with YAML frontmatter in `memory-blocks/` directory.

### Changes Required

#### 1. Update directory and file paths

**File**: `src/youlab_server/storage/git.py`

Update `__init__` and add `memory_blocks_dir` property (around line 61-76):

```python
def __init__(self, user_id: str, base_dir: Path | str) -> None:
    self.user_id = user_id
    self.base_dir = Path(base_dir)
    self.user_dir = self.base_dir / user_id
    self.memory_blocks_dir = self.user_dir / "memory-blocks"  # NEW
    self.diffs_dir = self.user_dir / "pending_diffs"
    self._repo: Repo | None = None
    self.logger = log.bind(user_id=user_id, component="git_storage")

# Remove old blocks_dir property or keep as alias:
@property
def blocks_dir(self) -> Path:
    """Alias for memory_blocks_dir (deprecated)."""
    return self.memory_blocks_dir
```

#### 2. Update init() to create new directory

**File**: `src/youlab_server/storage/git.py`

Update `init()` method (around line 93-105):

```python
def init(self) -> None:
    """Initialize user storage directory and git repo."""
    if self.exists:
        self.logger.debug("storage_already_exists")
        return

    # Create directories
    self.memory_blocks_dir.mkdir(parents=True, exist_ok=True)
    self.diffs_dir.mkdir(parents=True, exist_ok=True)

    # Initialize git repo
    self._repo = self._init_repo()
    self.logger.info("storage_initialized")
```

#### 3. Add YAML frontmatter parsing utilities

**File**: `src/youlab_server/storage/git.py`

Add after imports (around line 17):

```python
import re
from datetime import datetime, timezone

import yaml

# Frontmatter regex pattern
FRONTMATTER_PATTERN = re.compile(r"^---\n(.*?)\n---\n", re.DOTALL)


def parse_frontmatter(content: str) -> tuple[dict[str, Any], str]:
    """
    Parse YAML frontmatter from markdown content.

    Returns:
        Tuple of (frontmatter_dict, body_content)
    """
    match = FRONTMATTER_PATTERN.match(content)
    if not match:
        return {}, content

    frontmatter_str = match.group(1)
    body = content[match.end():]

    try:
        frontmatter = yaml.safe_load(frontmatter_str) or {}
    except yaml.YAMLError:
        frontmatter = {}

    return frontmatter, body.lstrip("\n")


def format_frontmatter(metadata: dict[str, Any], body: str) -> str:
    """
    Format metadata and body into markdown with YAML frontmatter.

    Args:
        metadata: Dict with block, schema (optional), updated_at
        body: Markdown body content

    Returns:
        Complete markdown string with frontmatter
    """
    # Ensure updated_at is set
    if "updated_at" not in metadata:
        metadata["updated_at"] = datetime.now(timezone.utc).isoformat()

    frontmatter = yaml.dump(metadata, default_flow_style=False, sort_keys=False)
    return f"---\n{frontmatter}---\n\n{body}"
```

#### 4. Update read_block() for markdown

**File**: `src/youlab_server/storage/git.py`

Replace `read_block()` method (lines 124-138):

```python
def read_block(self, label: str) -> str | None:
    """
    Read a memory block file.

    Args:
        label: Block label (e.g., "student", "journey")

    Returns:
        Full markdown content (with frontmatter) or None if not found
    """
    path = self.memory_blocks_dir / f"{label}.md"
    if not path.exists():
        return None
    return path.read_text()

def read_block_body(self, label: str) -> str | None:
    """
    Read just the body content of a memory block (no frontmatter).

    Args:
        label: Block label

    Returns:
        Body content or None if not found
    """
    content = self.read_block(label)
    if content is None:
        return None
    _, body = parse_frontmatter(content)
    return body

def read_block_metadata(self, label: str) -> dict[str, Any] | None:
    """
    Read frontmatter metadata from a memory block.

    Args:
        label: Block label

    Returns:
        Frontmatter dict or None if not found
    """
    content = self.read_block(label)
    if content is None:
        return None
    metadata, _ = parse_frontmatter(content)
    return metadata
```

#### 5. Update write_block() for markdown

**File**: `src/youlab_server/storage/git.py`

Replace `write_block()` method (lines 140-184):

```python
def write_block(
    self,
    label: str,
    content: str,
    message: str | None = None,
    author: str = "user",
    schema: str | None = None,
) -> str:
    """
    Write a memory block and commit.

    Args:
        label: Block label
        content: Markdown content (with or without frontmatter)
        message: Commit message (auto-generated if None)
        author: Who made the change ("user", "system", or agent name)
        schema: Optional schema reference (e.g., "college-essay/student")

    Returns:
        Commit SHA
    """
    # Ensure memory_blocks directory exists
    self.memory_blocks_dir.mkdir(parents=True, exist_ok=True)

    # Parse existing frontmatter if present, or create new
    existing_meta, body = parse_frontmatter(content)

    # Build metadata
    metadata = {
        "block": label,
        **existing_meta,
        "updated_at": datetime.now(timezone.utc).isoformat(),
    }
    if schema:
        metadata["schema"] = schema

    # If content had no frontmatter, treat entire content as body
    if not existing_meta:
        body = content

    # Format and write
    full_content = format_frontmatter(metadata, body)
    path = self.memory_blocks_dir / f"{label}.md"
    path.write_text(full_content)

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
```

#### 6. Update list_blocks() for markdown

**File**: `src/youlab_server/storage/git.py`

Update `list_blocks()` method (lines 321-325):

```python
def list_blocks(self) -> list[str]:
    """List all block labels in storage."""
    if not self.memory_blocks_dir.exists():
        return []
    return [p.stem for p in self.memory_blocks_dir.glob("*.md")]
```

#### 7. Update get_block_history() path

**File**: `src/youlab_server/storage/git.py`

Update `get_block_history()` method (lines 186-221):

```python
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
    path = self.memory_blocks_dir / f"{label}.md"
    rel_path = path.relative_to(self.user_dir)

    if not path.exists():
        return []

    versions = []
    for i, commit in enumerate(self.repo.iter_commits(paths=str(rel_path), max_count=limit)):
        msg = self._get_message(commit)
        versions.append(
            VersionInfo(
                commit_sha=commit.hexsha,
                message=msg.split("\n")[0],
                author=self._extract_author(commit),
                timestamp=datetime.fromtimestamp(commit.committed_date),
                is_current=(i == 0),
            )
        )

    return versions
```

#### 8. Update get_block_at_version() path

**File**: `src/youlab_server/storage/git.py`

Update `get_block_at_version()` method (lines 223-247):

```python
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
        rel_path = f"memory-blocks/{label}.md"
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
```

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes (lint + typecheck)
- [x] New user init creates `memory-blocks/` directory
- [x] `write_block()` creates `.md` files with YAML frontmatter
- [x] `read_block()` returns full markdown content
- [x] `read_block_body()` returns body without frontmatter
- [x] `list_blocks()` finds `.md` files
- [x] `get_block_history()` works with new path
- [x] `get_block_at_version()` works with new path

#### Manual Verification:
- [ ] Create a block, verify `.md` file has correct format
- [ ] Check git log shows commits for block changes

---

## Phase 1.2: UserBlockManager - Direct Markdown

### Overview

Update UserBlockManager to work directly with markdown files, removing TOML conversion layer.

### Changes Required

#### 1. Remove convert imports and simplify methods

**File**: `src/youlab_server/storage/blocks.py`

Update imports (lines 1-18):

```python
"""User-scoped block management with Letta sync."""

from __future__ import annotations

from typing import TYPE_CHECKING, Any

import structlog

from youlab_server.storage.diffs import PendingDiff, PendingDiffStore
from youlab_server.storage.git import parse_frontmatter

if TYPE_CHECKING:
    from letta_client import Letta

    from youlab_server.storage.git import GitUserStorage

log = structlog.get_logger()
```

#### 2. Simplify get_block methods

**File**: `src/youlab_server/storage/blocks.py`

Replace CRUD methods (lines 52-98):

```python
def list_blocks(self) -> list[str]:
    """List all block labels for this user."""
    return self.storage.list_blocks()

def get_block_markdown(self, label: str) -> str | None:
    """Get block content as full markdown (with frontmatter)."""
    return self.storage.read_block(label)

def get_block_body(self, label: str) -> str | None:
    """Get block body content (without frontmatter)."""
    return self.storage.read_block_body(label)

def get_block_metadata(self, label: str) -> dict[str, Any] | None:
    """Get block frontmatter metadata."""
    return self.storage.read_block_metadata(label)

def update_block(
    self,
    label: str,
    content: str,
    message: str | None = None,
    author: str = "user",
    schema: str | None = None,
    sync_to_letta: bool = True,
) -> str:
    """
    Update block from markdown content.

    Args:
        label: Block label
        content: Markdown content (body only or with frontmatter)
        message: Commit message
        author: Who made the change
        schema: Optional schema reference
        sync_to_letta: Whether to sync to Letta

    Returns:
        Commit SHA
    """
    commit_sha = self.storage.write_block(
        label=label,
        content=content,
        message=message or f"Update {label}",
        author=author,
        schema=schema,
    )

    if sync_to_letta and self.letta:
        self._sync_block_to_letta(label)

    return commit_sha

# Keep old method names as aliases for backward compatibility during transition
def update_block_from_markdown(
    self,
    label: str,
    markdown: str,
    message: str | None = None,
    sync_to_letta: bool = True,
) -> str:
    """Update block from markdown content (alias for update_block)."""
    return self.update_block(
        label=label,
        content=markdown,
        message=message,
        sync_to_letta=sync_to_letta,
    )
```

#### 3. Simplify Letta sync to send raw markdown

**File**: `src/youlab_server/storage/blocks.py`

Replace Letta sync methods (lines 159-222):

```python
def _sync_block_to_letta(self, label: str) -> None:
    """Sync block content to Letta."""
    if not self.letta:
        return

    block_name = self._letta_block_name(label)

    # Get body content (without frontmatter)
    body = self.storage.read_block_body(label)
    if body is None:
        self.logger.warning("sync_skipped_no_content", label=label)
        return

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
                value=body,
            )
            self.logger.debug("letta_block_updated", label=label)
        else:
            # Create new block
            self.letta.blocks.create(
                label=label,
                name=block_name,
                value=body,
            )
            self.logger.info("letta_block_created", label=label)

    except Exception as e:
        self.logger.error(
            "letta_sync_failed",
            label=label,
            error=str(e),
        )

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
    body = self.storage.read_block_body(label)
    if body is None:
        return None

    block = self.letta.blocks.create(
        label=label,
        name=block_name,
        value=body,
    )
    return block.id
```

#### 4. Update pending diff approval to work with markdown

**File**: `src/youlab_server/storage/blocks.py`

Update `approve_diff()` method (around lines 304-366):

```python
def approve_diff(self, diff_id: str) -> str:
    """
    Approve and apply a pending diff.

    Returns commit SHA.
    """
    diff = self.diffs.get(diff_id)
    if diff is None:
        msg = f"Diff {diff_id} not found"
        raise ValueError(msg)
    if diff.status != "pending":
        msg = f"Diff {diff_id} is not pending (status: {diff.status})"
        raise ValueError(msg)

    # Get current block body
    current_body = self.storage.read_block_body(diff.block_label) or ""

    # Apply the edit based on operation type
    if diff.operation == "append":
        new_body = current_body.rstrip() + "\n\n" + diff.proposed_value
    elif diff.operation == "replace":
        if diff.current_value and diff.current_value in current_body:
            new_body = current_body.replace(diff.current_value, diff.proposed_value, 1)
        else:
            msg = (
                f"Cannot apply diff {diff_id}: current_value not found in block. "
                "The block may have been modified since the diff was created."
            )
            raise ValueError(msg)
    elif diff.operation == "full_replace":
        new_body = diff.proposed_value
    else:
        msg = f"Unknown diff operation: {diff.operation}"
        raise ValueError(msg)

    commit_sha = self.update_block(
        label=diff.block_label,
        content=new_body,
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
```

#### 5. Remove _toml_to_memory_string helper

**File**: `src/youlab_server/storage/blocks.py`

Delete the `_toml_to_memory_string()` method (was at lines 213-222) - no longer needed.

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes
- [x] `get_block_markdown()` returns full content
- [x] `get_block_body()` returns body without frontmatter
- [x] `update_block()` writes markdown and syncs to Letta
- [x] `approve_diff()` works with markdown content

#### Manual Verification:
- [ ] Edit block via manager, verify `.md` file updated correctly
- [ ] Verify Letta receives raw markdown body

---

## Phase 1.3: Server API - Markdown Only

### Overview

Simplify the API to return markdown content only, removing TOML fields.

### Changes Required

#### 1. Update response models

**File**: `src/youlab_server/server/blocks.py`

Update models (lines 35-64):

```python
class BlockSummary(BaseModel):
    """Summary of a memory block."""

    label: str
    pending_diffs: int


class BlockDetail(BaseModel):
    """Detailed block information."""

    label: str
    content: str  # Full markdown with frontmatter
    body: str  # Body only (for editor)
    metadata: dict[str, Any]  # Parsed frontmatter
    pending_diffs: int


class BlockUpdateRequest(BaseModel):
    """Request to update a block."""

    content: str  # Markdown content (body or full)
    message: str | None = None
    schema: str | None = None  # Optional schema reference


class BlockUpdateResponse(BaseModel):
    """Response after updating a block."""

    commit_sha: str
    label: str
```

#### 2. Update get_block endpoint

**File**: `src/youlab_server/server/blocks.py`

Update `get_block()` endpoint (lines 132-152):

```python
@router.get("/{label}", response_model=BlockDetail)
async def get_block(
    user_id: str,
    label: str,
    storage: StorageDep,
) -> BlockDetail:
    """Get a specific memory block."""
    manager = get_block_manager(user_id, storage)
    content = manager.get_block_markdown(label)
    if content is None:
        raise HTTPException(status_code=404, detail=f"Block {label} not found")

    body = manager.get_block_body(label) or ""
    metadata = manager.get_block_metadata(label) or {}
    counts = manager.count_pending_diffs()

    return BlockDetail(
        label=label,
        content=content,
        body=body,
        metadata=metadata,
        pending_diffs=counts.get(label, 0),
    )
```

#### 3. Update update_block endpoint

**File**: `src/youlab_server/server/blocks.py`

Update `update_block()` endpoint (lines 155-177):

```python
@router.put("/{label}", response_model=BlockUpdateResponse)
async def update_block(
    user_id: str,
    label: str,
    request: BlockUpdateRequest,
    storage: StorageDep,
) -> BlockUpdateResponse:
    """Update a memory block (user edit)."""
    manager = get_block_manager(user_id, storage)
    commit_sha = manager.update_block(
        label=label,
        content=request.content,
        message=request.message,
        schema=request.schema,
    )

    return BlockUpdateResponse(commit_sha=commit_sha, label=label)
```

#### 4. Update get_block_version to return markdown

**File**: `src/youlab_server/server/blocks.py`

Update `get_block_version()` endpoint (lines 198-210):

```python
@router.get("/{label}/versions/{commit_sha}")
async def get_block_version(
    user_id: str,
    label: str,
    commit_sha: str,
    storage: StorageDep,
) -> dict[str, Any]:
    """Get block content at a specific version."""
    manager = get_block_manager(user_id, storage)
    content = manager.get_version(label, commit_sha)
    if content is None:
        raise HTTPException(status_code=404, detail="Version not found")

    # Parse frontmatter from historical content
    from youlab_server.storage.git import parse_frontmatter
    metadata, body = parse_frontmatter(content)

    return {
        "content": content,
        "body": body,
        "metadata": metadata,
        "sha": commit_sha,
    }
```

### Success Criteria

#### Automated Verification:
- [x] `make check-agent` passes
- [x] `GET /blocks/{label}` returns `content`, `body`, `metadata`
- [x] `PUT /blocks/{label}` accepts markdown content
- [x] `GET /blocks/{label}/versions/{sha}` returns markdown

#### Manual Verification:
- [ ] API responses have correct structure
- [ ] Frontend can consume new response format

---

## Phase 1.4: Delete convert.py

### Overview

Remove the TOML↔MD converter module since it's no longer needed.

### Changes Required

#### 1. Delete the file

**File**: `src/youlab_server/storage/convert.py`

Delete the entire file.

#### 2. Remove any remaining imports

Search for and remove any imports of `convert.py` functions:

```bash
grep -r "from youlab_server.storage.convert" src/
grep -r "from \.convert import" src/
```

### Success Criteria

#### Automated Verification:
- [x] `convert.py` deleted
- [x] No import errors when running `make check-agent`
- [x] No references to `toml_to_markdown` or `markdown_to_toml`

---

## Phase 1.5: Update Tests

### Overview

Update test suites to use the new markdown format and remove TOML-specific tests.

### Changes Required

#### 1. Delete test_convert.py

**File**: `tests/test_storage/test_convert.py`

Delete the entire file - no longer needed.

#### 2. Update test_blocks.py fixtures

**File**: `tests/test_storage/test_blocks.py`

Update fixtures and tests to use markdown format:

```python
@pytest.fixture
def sample_markdown(self):
    """Sample markdown content for a student block."""
    return """---
block: student
---

Background and context about Alice.

## Name
Alice

## Background
Computer science student
"""

# Update tests to use markdown instead of TOML
def test_get_block_markdown(self, manager, user_storage, sample_markdown):
    """Get block markdown returns content."""
    user_storage.write_block("student", sample_markdown, author="test")

    markdown = manager.get_block_markdown("student")
    assert markdown is not None
    assert "---" in markdown
    assert "block: student" in markdown
    assert "Alice" in markdown

def test_update_block(self, manager):
    """Update block creates commit."""
    content = """---
block: student
---

Bob is a developer.
"""
    commit_sha = manager.update_block(
        label="student",
        content=content,
        message="Add student block",
    )

    assert commit_sha is not None
    assert len(commit_sha) == 40

    # Verify content
    result = manager.get_block_markdown("student")
    assert "Bob" in result
```

#### 3. Update test_blocks_api.py

**File**: `tests/test_server/test_blocks_api.py`

Update API tests to use markdown format:

```python
@pytest.fixture
def initialized_user(storage_manager):
    """Create an initialized user with some blocks."""
    user_storage = storage_manager.get("test-user")
    user_storage.init()

    # Create a student block (markdown format)
    user_storage.write_block(
        "student",
        """---
block: student
---

Alice is a computer science student.

## Name
Alice

## Background
Computer science student
""",
        message="Initialize student block",
        author="system",
    )

    # Create a journey block
    user_storage.write_block(
        "journey",
        """---
block: journey
---

## Progress
Module 1
""",
        message="Initialize journey block",
        author="system",
    )

    return "test-user"


def test_get_block_returns_detail(self, blocks_test_client, initialized_user):
    """Get block returns markdown content."""
    response = blocks_test_client.get(f"/users/{initialized_user}/blocks/student")

    assert response.status_code == 200
    data = response.json()

    assert data["label"] == "student"
    assert "---" in data["content"]
    assert "block: student" in data["content"]
    assert "Alice" in data["body"]
    assert data["metadata"]["block"] == "student"


def test_update_block(self, blocks_test_client, initialized_user):
    """Update block from markdown content."""
    content = """---
block: student
---

Bob is an updated student.
"""
    response = blocks_test_client.put(
        f"/users/{initialized_user}/blocks/student",
        json={"content": content},
    )

    assert response.status_code == 200
    data = response.json()
    assert data["label"] == "student"
    assert len(data["commit_sha"]) == 40

    # Verify content updated
    get_response = blocks_test_client.get(f"/users/{initialized_user}/blocks/student")
    assert "Bob" in get_response.json()["body"]
```

### Success Criteria

#### Automated Verification:
- [x] `make test-agent` passes
- [x] All tests use markdown format
- [x] No references to TOML content in tests
- [x] `test_convert.py` deleted

---

## Testing Strategy

### Unit Tests

Update `tests/test_storage/test_blocks.py`:
- Test `write_block()` creates valid markdown with frontmatter
- Test `read_block()` returns full content
- Test `read_block_body()` strips frontmatter
- Test `read_block_metadata()` parses frontmatter
- Test `update_block()` preserves/updates frontmatter
- Test Letta sync sends body content

### Integration Tests

Update `tests/test_server/test_blocks_api.py`:
- Test all endpoints return markdown format
- Test version history works with `.md` files
- Test diff approval works with markdown content

### Manual Testing Steps

1. Create new user via API, verify `memory-blocks/` directory created
2. Create block, verify `.md` file has YAML frontmatter
3. Edit block, verify frontmatter preserved and `updated_at` changes
4. View git log for block, verify history works
5. Restore previous version, verify content restored

## Performance Considerations

1. **No conversion overhead**: Direct read/write eliminates TOML↔MD parsing
2. **Simpler Letta sync**: Raw markdown sent, no intermediate parsing

## Migration Notes

### Manual File Conversion

For existing TOML files, manually convert to markdown:

**Before** (`student.toml`):
```toml
name = "Alice"
background = "Computer science student"
strengths = ["Creativity", "Communication"]
```

**After** (`student.md`):
```markdown
---
block: student
updated_at: 2026-01-13T10:00:00Z
---

## Name
Alice

## Background
Computer science student

## Strengths
- Creativity
- Communication
```

Or for freeform blocks:

```markdown
---
block: student
updated_at: 2026-01-13T10:00:00Z
---

Alice is a computer science student with strengths in creativity and communication.
```

## References

- Parent plan: `thoughts/shared/plans/2026-01-13-ARI-82-memory-blocks-as-notes-plan.md`
- Research: `thoughts/searchable/shared/research/2026-01-13-ARI-82-toml-md-conversion.md`
- Research: `thoughts/searchable/shared/research/2026-01-13-ARI-82-user-storage-versioning.md`
- Linear ticket: [ARI-82](https://linear.app/ariav/issue/ARI-82)
