---
date: 2026-01-16T17:43:50-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Git-Based Versioning Scalability Analysis for 10,000 Users"
tags: [research, codebase, git-storage, scalability, performance, memory-blocks]
status: complete
last_updated: 2026-01-16
last_updated_by: ariasulin
---

# Research: Git-Based Versioning Scalability Analysis for 10,000 Users

**Date**: 2026-01-16T17:43:50-08:00
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Does the git-based versioning system create scalability or performance issues? Can it work effectively with 10,000 users?

## Summary

The current git-based versioning system uses **one git repository per user** stored at `.data/users/{user_id}/`. Each memory block edit creates a git commit for full version history. While this architecture provides excellent versioning capabilities and simplicity for small deployments, it has **significant scalability limitations** at 10,000+ users.

**Key Concerns:**
1. **10,000 separate git repositories** on disk - filesystem overhead and inode usage
2. **In-memory caching** of `GitUserStorage` instances without limits or eviction
3. **Synchronous git operations** (add, commit) block the event loop
4. **No connection pooling** or async support for concurrent operations
5. **Linear disk growth** - every edit creates a new git commit object

**Assessment**: The current implementation is suitable for **early-stage / POC** deployments (tens to hundreds of users) but would require architectural changes for 10,000+ users.

## Detailed Findings

### Current Architecture

**Location**: `src/youlab_server/storage/git.py`

The storage layer consists of three main components:

| Component | Location | Purpose |
|-----------|----------|---------|
| `GitUserStorage` | `git.py:89-441` | Per-user git repository manager |
| `GitUserStorageManager` | `git.py:443-475` | Factory with in-memory caching |
| `UserBlockManager` | `blocks.py:19-364` | High-level API with Letta sync |

**Directory Structure Per User** (`git.py:93-105`):
```
.data/users/{user_id}/
    .git/                    # Full git repository (~40-100KB minimum)
    memory-blocks/           # Markdown files with YAML frontmatter
        student.md
        journey.md
        engagement_strategy.md
    pending_diffs/           # JSON files for agent proposals
    agent_threads/           # Chat history (future use)
```

### Scalability Analysis

#### 1. Disk Space & Filesystem

**Per-user overhead:**
- Git repository minimum: ~40-100KB (`.git/` directory with objects, refs, hooks)
- Memory blocks: 1-10KB each (markdown with frontmatter)
- Pending diffs: ~1KB each (JSON files)

**At 10,000 users:**
| Resource | Estimate |
|----------|----------|
| Git repo overhead | 400MB - 1GB |
| User content (3 blocks avg) | ~150MB |
| Total disk minimum | ~600MB - 1.2GB |
| Inodes used | 30,000+ (3+ dirs per user) |

**Git history growth:**
- Each edit creates a new git object (blob + commit)
- With 10 edits per user: ~500KB additional per user
- At scale: **5GB+** for history alone

**Filesystem concerns:**
- Large number of directories degrades `ls` and `find` performance
- Backup systems slow down with millions of small files
- No sharding - all users in single `.data/users/` directory

#### 2. Memory Usage

**GitUserStorageManager caching** (`git.py:460-466`):
```python
class GitUserStorageManager:
    def __init__(self, base_dir: Path | str) -> None:
        self._cache: dict[str, GitUserStorage] = {}

    def get(self, user_id: str) -> GitUserStorage:
        if user_id not in self._cache:
            self._cache[user_id] = GitUserStorage(user_id, self.base_dir)
        return self._cache[user_id]
```

**Issues:**
- **No eviction policy** - cache grows unbounded
- Each `GitUserStorage` holds a `git.Repo` reference
- GitPython `Repo` objects hold file handles and memory buffers
- With 10,000 users: potential for **hundreds of MB** in cached objects

**Estimated memory per cached user:**
- `GitUserStorage` object: ~1KB
- `git.Repo` object: ~50-100KB (varies with repo size)
- At 10,000 concurrent users: **500MB - 1GB** just for repo caches

#### 3. I/O Performance

**Write operation** (`git.py:228-299`):
```python
def write_block(self, label, content, message, author, schema, title):
    # 1. Parse frontmatter (CPU)
    existing_meta, body = parse_frontmatter(content)

    # 2. Write file (I/O)
    path.write_text(full_content)

    # 3. Git add (I/O + spawns git process)
    self.repo.index.add([str(rel_path)])

    # 4. Git commit (I/O + spawns git process)
    commit = self.repo.index.commit(f"{message}\n\nAuthor: {author}")
```

**Performance characteristics:**
- **Synchronous file I/O** - blocks the Python thread
- **GitPython uses subprocess** for some operations
- **No async/await** support - each request blocks
- **No batching** - every small edit = full commit cycle

**Estimated latency per write:**
| Operation | Time |
|-----------|------|
| File write | 1-5ms |
| Git add | 10-50ms |
| Git commit | 20-100ms |
| **Total** | **30-150ms** |

At 10,000 concurrent writes: **potential bottleneck**

#### 4. Concurrent Access

**Current model:**
- No locking mechanism per user
- Same user from multiple devices = race condition
- `GitUserStorageManager._cache` is not thread-safe

**Risk scenarios:**
```
User A (Device 1): write_block("student", "content v1")
User A (Device 2): write_block("student", "content v2")
                   └─ Both run concurrently
                   └─ Git may produce merge conflict or corruption
```

#### 5. Read Performance

**Version history lookup** (`git.py:301-336`):
```python
def get_block_history(self, label: str, limit: int = 20):
    for i, commit in enumerate(self.repo.iter_commits(paths=str(rel_path), max_count=limit)):
        # Extract metadata from each commit
```

- Iterates git log per request
- **O(commits)** time complexity
- No indexing or caching of history

### Comparison: Current vs. Database-Backed

| Aspect | Git-Backed (Current) | Database-Backed |
|--------|---------------------|-----------------|
| Storage | Filesystem | PostgreSQL/SQLite |
| Versioning | Git commits | Version table |
| Query performance | O(n) scan | O(1) indexed |
| Concurrent access | No locking | ACID transactions |
| Memory | Unbounded cache | Connection pool |
| Async support | No | Yes (asyncpg) |
| Horizontal scaling | No | Read replicas |
| Backup | Complex (many files) | pg_dump |

### Theoretical Limits

Based on the analysis, estimated limits for current architecture:

| Metric | Safe Limit | Degraded | Critical |
|--------|------------|----------|----------|
| Concurrent users | 100 | 500 | 1,000 |
| Total users (disk) | 1,000 | 5,000 | 10,000 |
| Writes/second | 10 | 50 | 100 |
| Memory usage | 500MB | 2GB | 5GB |

**At 10,000 users**: System would likely experience:
- Slow startup (loading all repos)
- Memory pressure from cache
- I/O bottlenecks on writes
- Filesystem slowdown from directory size

## Code References

- `src/youlab_server/storage/git.py:89-441` - GitUserStorage class
- `src/youlab_server/storage/git.py:443-475` - GitUserStorageManager with cache
- `src/youlab_server/storage/git.py:228-299` - write_block (commit creation)
- `src/youlab_server/storage/git.py:301-336` - get_block_history
- `src/youlab_server/storage/blocks.py:19-364` - UserBlockManager
- `src/youlab_server/server/blocks.py` - HTTP API endpoints

## Architecture Documentation

### Current Data Flow

```
[HTTP Request: Update Block]
         ↓
[FastAPI endpoint - sync]
         ↓
[UserBlockManager.update_block()]
         ↓
[GitUserStorage.write_block()]
    │   1. parse_frontmatter() - CPU
    │   2. path.write_text() - I/O (sync)
    │   3. repo.index.add() - I/O (sync, may spawn git)
    │   4. repo.index.commit() - I/O (sync, may spawn git)
         ↓
[Optional: _sync_block_to_letta() - HTTP to Letta]
         ↓
[Return commit SHA]
```

### Caching Model

```
GitUserStorageManager._cache
    └── {user_id_1}: GitUserStorage
    │       └── _repo: git.Repo (file handles, buffers)
    └── {user_id_2}: GitUserStorage
    │       └── _repo: git.Repo
    └── ... (grows unbounded)
```

## Historical Context (from thoughts/)

- `thoughts/shared/research/2026-01-13-ARI-82-user-storage-versioning.md` - Original design research for git-backed versioning
- `thoughts/shared/plans/2026-01-14-ARI-82-phase3-git-storage-adapter.md` - Implementation plan for git storage adapter
- `thoughts/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md` - Initial memory system design

## Related Research

- `thoughts/shared/research/2026-01-16-letta-memory-version-control.md` - Letta memory versioning
- `thoughts/shared/research/2026-01-04-team-deployment-options.md` - Deployment scalability considerations

## Open Questions

1. **What is the actual target scale?** - Is 10,000 users a near-term goal or theoretical?

2. **Read vs. write ratio?** - If mostly reads, caching could help significantly

3. **User concurrency pattern?** - Are all 10,000 users active simultaneously or spread over time?

4. **Acceptable latency?** - Is 100-150ms write latency acceptable for the use case?

5. **Deployment environment?** - SSD vs. HDD, available RAM, etc.

## Conclusion

The git-based versioning system is **not designed for 10,000 concurrent users**. It works well for:
- Early-stage development
- POC deployments (tens of users)
- Pilot programs (hundreds of users)

For 10,000+ users, the system would need architectural changes such as:
- Database-backed version storage (PostgreSQL with version table)
- Async I/O throughout the stack
- Proper connection pooling
- LRU caching with eviction
- Horizontal scaling capability

The current implementation prioritizes **simplicity and auditability** over scale, which is appropriate for the project's current stage.
