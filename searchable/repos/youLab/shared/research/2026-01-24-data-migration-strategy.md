---
date: 2026-01-25T02:32:29Z
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Data Migration Strategy for YouLab v2"
tags: [research, codebase, migration, dolt, git-storage, data-portability]
status: complete
last_updated: 2026-01-24
last_updated_by: ariasulin
---

# Research: Data Migration Strategy for YouLab v2

**Date**: 2026-01-25T02:32:29Z
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Design a data migration strategy for YouLab v2 that:
1. Migrates Git-backed user storage to Dolt
2. Preserves version history
3. Maintains data integrity with rollback capabilities
4. Keeps Honcho data unchanged
5. Provides a phased, testable approach

## Summary

The migration from Git-backed file storage to Dolt requires careful planning due to the fundamentally different versioning models. **Git uses commit-based versioning with full snapshots, while Dolt uses SQL with cell-level versioning**. The strategy involves extracting Git history into structured records, importing into Dolt with proper schema design, and maintaining parallel operation during transition.

**Key Decisions:**
- Honcho data remains unchanged (messages and sessions)
- Memory blocks migrate from markdown files to Dolt `blocks` table
- Pending diffs migrate from JSON files to Dolt `pending_diffs` table
- Version history imports as a separate `block_versions` table
- Phased rollout with shadow mode validation

---

## Current Data Architecture

### Git-Backed Storage

**Location**: `.data/users/{user_id}/` (one git repository per user)

**Directory Structure**:
```
.data/users/{user_id}/
    .git/                           # Full git repository
    memory-blocks/                  # Markdown files with YAML frontmatter
        student.md
        progress.md
        operating_manual.md
        course_syllabus.md
    pending_diffs/                  # JSON files for agent proposals
        {diff_id}.json
```

**Memory Block Format** (`src/youlab_server/storage/git.py:25-65`):
```markdown
---
block: student
updated_at: '2026-01-13T18:27:03.150665+00:00'
title: Student
schema: college-essay/student      # Optional
---

## About Me

Markdown content here...
```

**Pending Diff Format** (`src/youlab_server/storage/diffs.py:19-38`):
```json
{
  "id": "644daa8d-b8d7-4197-be9a-03bab55871df",
  "user_id": "7a41011b-5255-4225-b75e-1d8484d0e37f",
  "agent_id": "test-mock-agent",
  "block_label": "student",
  "field": null,
  "operation": "full_replace",
  "current_value": "...",
  "proposed_value": "...",
  "reasoning": "Updating student block",
  "confidence": "medium",
  "source_query": null,
  "status": "pending",
  "created_at": "2026-01-16T14:44:16.831174",
  "reviewed_at": null,
  "applied_commit": null
}
```

### Data Volumes

Based on existing user data and scalability analysis:

| Data Type | Per User | At 10,000 Users |
|-----------|----------|-----------------|
| Git repo overhead | 40-100KB | 400MB - 1GB |
| Memory blocks (3-5 blocks) | 3-15KB | 30-150MB |
| Block versions (10 avg) | 30-150KB | 300MB - 1.5GB |
| Pending diffs | 1-5KB | 10-50MB |
| **Total** | ~200KB | **~2-3GB** |

---

## Target Dolt Architecture

### Schema Design

Based on Dolt documentation and v2 architecture plan (`thoughts/shared/plans/2026-01-24-youlab-v2-greenfield.md`):

```sql
-- Memory blocks (current state)
CREATE TABLE blocks (
    user_id VARCHAR(255) NOT NULL,
    label VARCHAR(255) NOT NULL,
    content LONGTEXT NOT NULL,
    metadata JSON,                    -- YAML frontmatter as JSON
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, label),
    INDEX idx_user (user_id),
    INDEX idx_updated (updated_at)
);

-- Block version history (imported from git)
CREATE TABLE block_versions (
    id VARCHAR(255) PRIMARY KEY,      -- Git commit SHA or generated UUID
    user_id VARCHAR(255) NOT NULL,
    label VARCHAR(255) NOT NULL,
    content LONGTEXT NOT NULL,
    metadata JSON,
    author VARCHAR(255) NOT NULL,     -- 'user', 'system', or 'agent:{agent_id}'
    message VARCHAR(500),             -- Commit message
    created_at DATETIME NOT NULL,
    is_current BOOLEAN DEFAULT FALSE,
    INDEX idx_block (user_id, label),
    INDEX idx_created (created_at DESC)
);

-- Pending diffs (agent proposals)
CREATE TABLE pending_diffs (
    id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    agent_id VARCHAR(255) NOT NULL,
    block_label VARCHAR(255) NOT NULL,
    field VARCHAR(255),
    operation ENUM('append', 'replace', 'llm_diff', 'full_replace') NOT NULL,
    current_value LONGTEXT,
    proposed_value LONGTEXT NOT NULL,
    reasoning TEXT NOT NULL,
    confidence ENUM('low', 'medium', 'high') NOT NULL,
    source_query TEXT,
    status ENUM('pending', 'approved', 'rejected', 'superseded', 'expired') NOT NULL,
    created_at DATETIME NOT NULL,
    reviewed_at DATETIME,
    applied_commit VARCHAR(255),       -- Dolt commit hash after approval
    INDEX idx_user_pending (user_id, status),
    INDEX idx_block (block_label, status)
);

-- Module to chat mapping (new for v2)
CREATE TABLE module_chats (
    user_id VARCHAR(255) NOT NULL,
    agent_id VARCHAR(255) NOT NULL,
    module_id VARCHAR(255) NOT NULL,
    owui_chat_id VARCHAR(255) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, agent_id, module_id),
    UNIQUE INDEX idx_chat (owui_chat_id)
);

-- User API keys for OpenWebUI (new for v2)
CREATE TABLE user_api_keys (
    user_id VARCHAR(255) PRIMARY KEY,
    owui_api_key VARCHAR(500) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Performance Considerations

From Dolt documentation research:

| Consideration | Mitigation |
|---------------|------------|
| TEXT columns 2x slower than VARCHAR | Use `content LONGTEXT` only for body; metadata as JSON |
| No TEXT indexes | Add VARCHAR columns for indexed queries (user_id, label) |
| Batch inserts 130x faster | Migrate in batches of 500+ rows |
| BLOB separate storage | Expected for document content; acceptable tradeoff |

---

## Migration Strategy

### Phase 1: Schema Setup & Dry Run

**Goal**: Validate schema and test import with synthetic data

**Steps**:
1. Create Dolt database: `dolt init youlab_v2`
2. Apply schema DDL
3. Generate synthetic test data (100 users, 500 blocks)
4. Test import performance
5. Validate queries and indexes

**Validation Criteria**:
- Schema compiles without errors
- Indexes created successfully
- Test queries return expected results
- Import of 500 blocks < 30 seconds

### Phase 2: Export Script Development

**Goal**: Create reliable export from Git repos

**Export Script Pseudocode**:
```python
def export_user_data(user_storage: GitUserStorage) -> dict:
    """Export all user data to migration-ready format."""

    user_data = {
        "user_id": user_storage.user_id,
        "blocks": [],
        "block_versions": [],
        "pending_diffs": []
    }

    # Export current blocks
    for label in user_storage.list_blocks():
        content = user_storage.read_block(label)
        metadata, body = parse_frontmatter(content)

        user_data["blocks"].append({
            "label": label,
            "content": body,
            "metadata": json.dumps(metadata),
            "updated_at": metadata.get("updated_at")
        })

        # Export version history
        for version in user_storage.get_block_history(label, limit=100):
            historical_content = user_storage.get_block_at_version(label, version.commit_sha)
            if historical_content:
                hist_meta, hist_body = parse_frontmatter(historical_content)
                user_data["block_versions"].append({
                    "id": version.commit_sha,
                    "label": label,
                    "content": hist_body,
                    "metadata": json.dumps(hist_meta),
                    "author": version.author,
                    "message": version.message,
                    "created_at": version.timestamp.isoformat(),
                    "is_current": version.is_current
                })

    # Export pending diffs
    diffs_store = PendingDiffStore(user_storage.diffs_dir)
    for diff_file in user_storage.diffs_dir.glob("*.json"):
        diff = diffs_store.get(diff_file.stem)
        if diff:
            user_data["pending_diffs"].append(asdict(diff))

    return user_data
```

**Output Format**: NDJSON (newline-delimited JSON) for streaming import

### Phase 3: Import Script Development

**Goal**: Reliable import to Dolt with version control

**Import Strategy**:
```python
import mysql.connector

def import_to_dolt(user_data: dict, conn: mysql.connector.Connection):
    """Import user data to Dolt with proper versioning."""
    cursor = conn.cursor()

    try:
        # Import current blocks (batched)
        blocks_sql = """
            INSERT INTO blocks (user_id, label, content, metadata, updated_at)
            VALUES (%s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE content = VALUES(content),
                                    metadata = VALUES(metadata),
                                    updated_at = VALUES(updated_at)
        """
        block_values = [
            (user_data["user_id"], b["label"], b["content"],
             b["metadata"], b["updated_at"])
            for b in user_data["blocks"]
        ]
        cursor.executemany(blocks_sql, block_values)

        # Import version history (batched)
        versions_sql = """
            INSERT INTO block_versions
            (id, user_id, label, content, metadata, author, message, created_at, is_current)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        version_values = [
            (v["id"], user_data["user_id"], v["label"], v["content"],
             v["metadata"], v["author"], v["message"], v["created_at"], v["is_current"])
            for v in user_data["block_versions"]
        ]
        cursor.executemany(versions_sql, version_values)

        # Import pending diffs
        diffs_sql = """
            INSERT INTO pending_diffs
            (id, user_id, agent_id, block_label, field, operation, current_value,
             proposed_value, reasoning, confidence, source_query, status,
             created_at, reviewed_at, applied_commit)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        diff_values = [
            (d["id"], d["user_id"], d["agent_id"], d["block_label"], d["field"],
             d["operation"], d["current_value"], d["proposed_value"], d["reasoning"],
             d["confidence"], d["source_query"], d["status"], d["created_at"],
             d["reviewed_at"], d["applied_commit"])
            for d in user_data["pending_diffs"]
        ]
        if diff_values:
            cursor.executemany(diffs_sql, diff_values)

        # Commit to Dolt
        cursor.execute("CALL DOLT_ADD('-A')")
        cursor.execute(f"CALL DOLT_COMMIT('-m', 'Import user {user_data['user_id']}')")

        conn.commit()

    except Exception as e:
        conn.rollback()
        raise
```

### Phase 4: Parallel Operation (Shadow Mode)

**Goal**: Validate migration without user impact

**Approach**:
1. Deploy v2 service alongside v1 (different port/path)
2. Enable write-through: v1 writes also export to Dolt
3. Compare reads: verify v1 and v2 return identical data
4. Monitor for discrepancies

**Shadow Mode Architecture**:
```
User Request
    │
    ├──> v1 Service (Git storage) ──> Response to user
    │         │
    │         └──> Async: Export to Dolt (shadow write)
    │
    └──> v2 Service (Dolt) ──> Log only (comparison)
```

**Validation Queries**:
```sql
-- Count blocks per user
SELECT user_id, COUNT(*) as block_count
FROM blocks
GROUP BY user_id;

-- Verify version history depth
SELECT user_id, label, COUNT(*) as version_count
FROM block_versions
GROUP BY user_id, label;

-- Check pending diff counts
SELECT user_id, status, COUNT(*) as diff_count
FROM pending_diffs
GROUP BY user_id, status;
```

### Phase 5: Cutover

**Goal**: Switch to Dolt as primary with rollback capability

**Pre-Cutover Checklist**:
- [ ] All users migrated to Dolt
- [ ] Shadow mode validation passed (7+ days)
- [ ] v2 service tested in staging
- [ ] Rollback procedure documented and tested
- [ ] User communication sent
- [ ] Maintenance window scheduled

**Cutover Steps**:
1. **T-30min**: Final sync of any new data
2. **T-15min**: Enable read-only mode on v1
3. **T-0**: Update DNS/routing to v2 service
4. **T+5min**: Verify v2 accepting traffic
5. **T+15min**: Smoke test all endpoints
6. **T+30min**: Declare cutover complete or rollback

**Post-Cutover**:
- Keep v1 Git repos for 30 days (rollback window)
- Monitor error rates and latency
- Disable shadow writes after 7 days stable

---

## Version History Preservation

### Git to Dolt Mapping

| Git Concept | Dolt Equivalent |
|-------------|-----------------|
| Commit SHA | `block_versions.id` (preserved) |
| Commit message | `block_versions.message` |
| Author | `block_versions.author` |
| Timestamp | `block_versions.created_at` |
| File content | `block_versions.content` |

### Dolt Native Versioning

After migration, new edits use Dolt's native versioning:

```sql
-- After user edits a block
UPDATE blocks SET content = 'new content' WHERE user_id = ? AND label = ?;
CALL DOLT_ADD('blocks');
CALL DOLT_COMMIT('-m', 'Update student block', '--author', 'user <user@youlab.ai>');

-- Query Dolt history directly
SELECT * FROM dolt_history_blocks
WHERE user_id = ? AND label = ?
ORDER BY commit_date DESC
LIMIT 10;
```

### History Query Migration

Current API (`src/youlab_server/storage/blocks.py:125-137`):
```python
def get_history(self, label: str, limit: int = 20) -> list[dict[str, Any]]:
    versions = self.storage.get_block_history(label, limit)
    return [{"sha": v.commit_sha, "message": v.message, ...} for v in versions]
```

New API (Dolt-backed):
```python
def get_history(self, label: str, limit: int = 20) -> list[dict[str, Any]]:
    # Combine imported history with Dolt native history
    cursor.execute("""
        SELECT id as sha, message, author, created_at, is_current
        FROM block_versions
        WHERE user_id = %s AND label = %s
        ORDER BY created_at DESC
        LIMIT %s
    """, (self.user_id, label, limit))
    return [dict(row) for row in cursor.fetchall()]
```

---

## Rollback Strategy

### Pre-Migration Backup

Before any migration:
```bash
# Backup entire .data directory
tar -czf youlab_backup_$(date +%Y%m%d).tar.gz .data/

# Store in durable location (S3, GCS, etc.)
aws s3 cp youlab_backup_*.tar.gz s3://youlab-backups/
```

### Dolt Rollback

Dolt provides built-in rollback via branches and reset:

```bash
# Create pre-migration branch
dolt checkout -b pre-migration-backup
dolt checkout main

# If migration fails, rollback
dolt reset --hard pre-migration-backup
```

### Full Rollback to Git

If Dolt migration must be completely reversed:

1. Stop v2 service
2. Restore DNS/routing to v1
3. Extract any new data from Dolt (if needed)
4. Resume v1 service with Git storage

**Data Recovery Script**:
```python
def export_dolt_to_git(dolt_conn, git_storage: GitUserStorage):
    """Export Dolt blocks back to Git storage."""
    cursor = dolt_conn.cursor()
    cursor.execute("""
        SELECT label, content, metadata, updated_at
        FROM blocks WHERE user_id = %s
    """, (git_storage.user_id,))

    for row in cursor.fetchall():
        content = format_frontmatter(json.loads(row['metadata']), row['content'])
        git_storage.write_block(
            label=row['label'],
            content=content,
            message="Restore from Dolt",
            author="system"
        )
```

---

## Testing Approach

### Unit Tests

```python
def test_export_user_data():
    """Test export produces valid migration format."""
    storage = create_test_git_storage()
    storage.write_block("test", "content", author="user")

    data = export_user_data(storage)

    assert data["user_id"] == storage.user_id
    assert len(data["blocks"]) == 1
    assert data["blocks"][0]["label"] == "test"
    assert len(data["block_versions"]) >= 1

def test_import_to_dolt():
    """Test import creates valid Dolt records."""
    user_data = create_test_export_data()

    import_to_dolt(user_data, dolt_conn)

    cursor = dolt_conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM blocks WHERE user_id = %s",
                   (user_data["user_id"],))
    assert cursor.fetchone()[0] == len(user_data["blocks"])

def test_roundtrip_integrity():
    """Test export->import preserves all data."""
    original = create_test_git_storage_with_history()

    exported = export_user_data(original)
    import_to_dolt(exported, dolt_conn)
    re_exported = export_from_dolt(dolt_conn, original.user_id)

    assert exported["blocks"] == re_exported["blocks"]
    assert len(exported["block_versions"]) == len(re_exported["block_versions"])
```

### Integration Tests

```python
def test_shadow_mode_consistency():
    """Test v1 and v2 return identical data."""
    user_id = create_test_user()

    # Write via v1
    v1_response = v1_client.update_block(user_id, "test", "content")

    # Wait for shadow write
    await asyncio.sleep(0.5)

    # Read from both
    v1_data = v1_client.get_block(user_id, "test")
    v2_data = v2_client.get_block(user_id, "test")

    assert v1_data["body"] == v2_data["body"]
    assert v1_data["metadata"] == v2_data["metadata"]

def test_version_history_preserved():
    """Test git history is accessible after migration."""
    user_id = setup_user_with_history(versions=10)

    migrate_user(user_id)

    history = v2_client.get_block_history(user_id, "test")
    assert len(history) == 10
    assert history[0]["is_current"] == True
```

### Load Tests

```python
def test_bulk_import_performance():
    """Test import of 1000 users completes in reasonable time."""
    users = [generate_test_user() for _ in range(1000)]

    start = time.time()
    for user in users:
        import_to_dolt(user, dolt_conn)
    elapsed = time.time() - start

    assert elapsed < 300  # 5 minutes for 1000 users

def test_concurrent_access():
    """Test multiple users accessing simultaneously."""
    user_ids = [create_migrated_user() for _ in range(100)]

    async def access_user(user_id):
        await v2_client.get_block(user_id, "test")
        await v2_client.update_block(user_id, "test", f"updated-{user_id}")

    await asyncio.gather(*[access_user(uid) for uid in user_ids])

    # Verify all updates persisted
    for uid in user_ids:
        block = await v2_client.get_block(uid, "test")
        assert block["body"] == f"updated-{uid}"
```

---

## User Communication

### Pre-Migration Notice (T-7 days)

> **Subject: YouLab Platform Upgrade - Enhanced Memory System**
>
> We're upgrading YouLab's memory system to improve performance and reliability.
>
> **What's changing:**
> - Faster access to your notes and progress
> - Better version history browsing
> - Improved reliability at scale
>
> **What's NOT changing:**
> - Your conversations and memories are preserved
> - All your progress and notes remain intact
> - Your experience stays the same
>
> **When:** [Maintenance window]
>
> **Action required:** None! We'll handle everything.

### During Migration (Status Page)

> **YouLab Maintenance in Progress**
>
> We're upgrading our systems. This typically takes 15-30 minutes.
> Your data is safe and will be available when we're done.
>
> Status: Migrating user data...
> Progress: 7,432 / 10,000 users

### Post-Migration Confirmation

> **Upgrade Complete!**
>
> YouLab's memory system has been upgraded. Everything is back to normal.
>
> If you notice anything unusual, please contact support@youlab.ai.

---

## Honcho Data (Unchanged)

Per the v2 architecture plan, **Honcho data remains unchanged**:

| Data | Location | Migration Status |
|------|----------|------------------|
| Messages | Honcho service | No migration needed |
| Sessions | Honcho service | No migration needed |
| Peers | Honcho service | No migration needed |

**Rationale**: Honcho provides dialectic query capabilities that are non-trivial to rebuild. Messages don't need version control. Both Dolt and Honcho are open source and self-hostable.

---

## Code References

### Current Implementation
- `src/youlab_server/storage/git.py:89-441` - GitUserStorage class
- `src/youlab_server/storage/git.py:443-475` - GitUserStorageManager
- `src/youlab_server/storage/blocks.py:19-364` - UserBlockManager
- `src/youlab_server/storage/diffs.py:19-143` - PendingDiff and PendingDiffStore
- `src/youlab_server/server/blocks.py` - HTTP API endpoints

### v2 Architecture
- `thoughts/shared/plans/2026-01-24-youlab-v2-greenfield.md` - Target architecture

### Related Research
- `thoughts/shared/research/2026-01-16-git-versioning-scalability-analysis.md` - Scalability concerns
- `thoughts/shared/research/2026-01-13-ARI-82-user-storage-versioning.md` - Original git design

---

## Open Questions

1. **Dolt hosting**: Self-hosted vs. DoltHub cloud?

2. **Connection pooling**: Use SQLAlchemy, aiomysql, or direct mysql.connector?

3. **Migration batch size**: How many users per batch to balance speed vs. memory?

4. **Downtime tolerance**: Can we do zero-downtime migration with dual-write?

5. **Export format preference**: Keep backup as NDJSON or also provide SQLite/Dolt clone?

---

## Appendix: Dolt Commands Reference

```bash
# Initialize database
dolt init youlab_v2

# Apply schema
dolt sql < schema.sql
dolt add .
dolt commit -m "Initial schema"

# Start SQL server
dolt sql-server -H 0.0.0.0 -P 3306

# Create backup branch
dolt checkout -b backup-$(date +%Y%m%d)
dolt checkout main

# View history
dolt log --oneline
dolt diff HEAD~1

# Rollback
dolt reset --hard HEAD~1

# Garbage collection after large import
dolt gc
```

---

*Last updated: 2026-01-24*
*Status: Complete*
