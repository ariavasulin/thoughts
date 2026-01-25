---
date: 2026-01-25T02:31:34Z
researcher: ARI
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Dolt Database Reference for YouLab v2 Architecture"
tags: [research, dolt, database, versioning, mysql, youlab-v2]
status: complete
last_updated: 2026-01-24
last_updated_by: ARI
---

# Research: Dolt Database Reference for YouLab v2 Architecture

**Date**: 2026-01-25T02:31:34Z
**Researcher**: ARI
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Comprehensive Dolt database reference for YouLab v2 architecture covering: Dolt vs DoltgreSQL, schema design patterns, connection pooling, branch/commit operations, Python clients/ORMs, diffing/merge capabilities, performance, and self-hosting.

## Summary

Dolt is a MySQL-compatible SQL database with Git-like version control. As of December 2025, Dolt has achieved **performance parity with MySQL** (0.99x multiplier on Sysbench). It's production-ready, backing thousands of applications, while DoltgreSQL (PostgreSQL variant) remains in Beta with 1.0 expected April 2026.

**Key Recommendation**: Use Dolt (MySQL protocol) over DoltgreSQL for YouLab v2:
- Production-ready since 2023
- Full CLI tooling and DoltHub integration
- Only 1.1x slower than MySQL (vs 5.2x for DoltgreSQL)
- Standard MySQL Python drivers work seamlessly

**YouLab Context**: The current YouLab codebase uses file-based git storage (GitPython) for user memory blocks. Dolt would provide the same versioning semantics with SQL queryability, replacing the custom `GitUserStorage` implementation.

---

## 1. Dolt vs DoltgreSQL: Why MySQL Protocol

### Core Differences

| Aspect | Dolt | DoltgreSQL |
|--------|------|------------|
| **Wire Protocol** | MySQL | PostgreSQL |
| **Maturity** | Production-ready (1.0+ since 2023) | Beta (1.0 expected ~April 2026) |
| **Performance** | 1.1x slower than MySQL | 5.2x slower than PostgreSQL |
| **CLI Tools** | Full Git-style CLI | SQL interface only |
| **DoltHub/DoltLab** | Full support | Not yet supported |

### What is Dolt?

Dolt is "Git for Data" - a SQL database you can fork, clone, branch, merge, push, and pull just like a Git repository. The tagline: **"Git versions files. Dolt versions tables."**

- MySQL-compatible drop-in replacement
- Version control accessible via CLI or SQL (system tables, stored procedures)
- Supports binlog replication from existing MySQL

### DoltHub's Official Recommendation

> "For people shopping for a version-controlled database and not choosy about the dialect or SQL feature set, the recommendation continues to be to choose Dolt. It's already feature complete and production quality, backing thousands of customer applications."

### Current Version: Dolt 1.75+ (October 2025)

Major 2025 features:
- **Automatic Garbage Collection** - Default in 1.75
- **Archive Storage Format** - 25-30% storage reduction
- **Concurrent GC** - No server disruption
- **Vector Indexes** - Vector search capabilities
- **Dolt MCP Server** - AI/LLM integration

**Sources**:
- [Dolt Documentation](https://docs.dolthub.com)
- [State of Doltgres (October 2025)](https://www.dolthub.com/blog/2025-10-16-state-of-doltgres/)
- [Dolt 1.75 Announcement](https://www.dolthub.com/blog/2025-10-20-dolt-1-75/)

---

## 2. Schema Design Patterns for Versioned Data

### How Dolt Stores Versioned Data

Dolt uses **Prolly Trees** (Probabilistic B-trees) - a content-addressed data structure combining:
- B-tree performance (logarithmic seeks)
- Merkle tree properties (content addressing, fast diffs)
- Git-style structural sharing between versions

**Key Properties**:
- Unchanged chunks share the same content address (stored once)
- Diff time scales with **size of changes**, not total table size
- Single modification affects only ~4KB × tree depth in new storage

### Primary Key Requirements (Critical)

**Always define primary keys** for tables where you need cell-wise versioning:
- With primary keys: Dolt tracks changes to individual columns
- Without primary keys (keyless tables): Only row additions/deletions tracked, not modifications
- Primary key changes appear as delete-insert pairs

**Best Practice**: Use keyless tables for initial data loading, then migrate to primary-key tables once data is validated.

### Schema Migration Patterns

**Recommended: Schema Branch and Merge**

```sql
-- Create branch for schema changes
CALL dolt_checkout('-b', 'schema_changes');

-- Make schema changes
ALTER TABLE employees ADD COLUMN start_date DATE;

-- Commit changes
CALL dolt_commit('-am', 'Added start_date column to employees');

-- Merge to main
CALL dolt_checkout('main');
CALL dolt_merge('schema_changes');
```

All changes are instantly recoverable using `dolt_reset()`.

### Temporal Data and Audit Trails

Dolt functions as a **uni-temporal database** through its Git-style versioning:
- Eliminates need for Slowly Changing Dimensions (SCD Type 2)
- Supports `AS OF` queries and time travel via commits/branches/tags
- Built-in queryable audit log of every cell

### Key System Tables for Auditing

| Table | Purpose |
|-------|---------|
| `dolt_history_$TABLENAME` | Row values at every commit in current branch |
| `dolt_diff_$TABLENAME` | Row changes across adjacent commits |
| `dolt_commit_diff_$TABLENAME` | Two-dot diff between any two commits |
| `dolt_blame_$TABLENAME` | Which user/commit is responsible for current values |
| `dolt_log` | Commit history reachable from current HEAD |

**Example Audit Queries**:

```sql
-- Full history of a specific row
SELECT * FROM dolt_history_employees
WHERE employee_id = 123
ORDER BY commit_date DESC;

-- Who changed what and when
SELECT * FROM dolt_blame_employees;

-- See uncommitted changes
SELECT * FROM dolt_commit_diff_employees
WHERE from_commit = 'HEAD' AND to_commit = 'WORKING';
```

### Dolt vs MySQL Differences

| Aspect | MySQL | Dolt |
|--------|-------|------|
| Version Control | None | Full Git-style |
| Primary Keys | Optional | Required for cell-wise versioning |
| Isolation Level | Multiple | REPEATABLE_READ only |
| Information Schema | Full | ~60% coverage |
| Performance Schema | Supported | Not supported |

**Sources**:
- [Storage Engine Documentation](https://docs.dolthub.com/architecture/storage-engine)
- [Primary Keys Documentation](https://docs.dolthub.com/concepts/dolt/sql/primary-key)
- [Schema Migrations in Dolt](https://www.dolthub.com/blog/2024-04-18-dolt-schema-migrations/)
- [System Tables Documentation](https://docs.dolthub.com/sql-reference/version-control/dolt-system-tables)

---

## 3. Connection Pooling (SQLAlchemy/aiomysql)

### Recommended Python Drivers

Dolt supports standard MySQL connectors:
1. **mysql-connector-python** (recommended) - Official MySQL connector
2. **PyMySQL** - Pure Python
3. **mysqlclient** - C-based (faster)

**Note**: DoltPy is deprecated. Use standard MySQL drivers.

### SQLAlchemy Connection Setup

```python
from sqlalchemy import create_engine

# Using mysql-connector-python
engine = create_engine("mysql+mysqlconnector://root@127.0.0.1:3306/doltdb")

# Using PyMySQL
engine = create_engine("mysql+pymysql://root@127.0.0.1:3306/doltdb")

# Connect to specific branch
engine = create_engine("mysql+pymysql://root@127.0.0.1:3306/doltdb/feature-branch")
```

### Production Connection Pool Configuration

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://user:password@dolt-server:3306/mydb",
    pool_size=20,              # Persistent connections (default: 5)
    max_overflow=10,           # Additional when exhausted (default: 10)
    pool_timeout=30,           # Seconds to wait (default: 30)
    pool_recycle=1800,         # Recycle after 30 min (< server timeout)
    pool_pre_ping=True,        # Validate before use
    pool_use_lifo=True,        # Allow idle timeout
    pool_reset_on_return='rollback',
    connect_args={
        'connect_timeout': 10,
        'read_timeout': 30,
        'write_timeout': 30,
    }
)
```

### Async with aiomysql

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(
    "mysql+aiomysql://user:password@dolt-server:3306/mydb",
    pool_size=20,
    max_overflow=30,
    pool_recycle=1800,
    pool_pre_ping=True,
)

# FastAPI dependency
async def get_session():
    async_session = async_sessionmaker(engine, expire_on_commit=False)
    async with async_session() as session:
        yield session
```

### Branch Selection Methods

**Method 1: Connection String**
```python
engine = create_engine("mysql+pymysql://root@127.0.0.1:3306/mydb/feature-branch")
```

**Method 2: USE Statement**
```sql
USE `mydb/feature-branch`;
```

**Method 3: DOLT_CHECKOUT Procedure**
```python
cursor.callproc("DOLT_CHECKOUT", ['feature-branch'])
```

**Method 4: Session Variable**
```sql
SET @@mydb_head_ref = 'feature-branch';
```

### Transaction Handling

**Key Distinction**: Dolt has two types of commits:
1. **SQL Transaction Commit** - Standard `COMMIT`
2. **Dolt Version Control Commit** - Creates point-in-time snapshot

```python
# Standard transaction
cursor.execute("BEGIN")
cursor.execute("INSERT INTO users (name) VALUES ('Alice')")
cursor.execute("COMMIT")  # SQL commit only

# Dolt version control commit
cursor.callproc("DOLT_COMMIT", ["-Am", "Added user Alice"])
```

**Auto-commit Dolt commits:**
```sql
SET @@dolt_transaction_commit = 1;
SET @@dolt_transaction_commit_message = 'Auto-commit message';
```

**Sources**:
- [Dolt Programmatic Clients](https://docs.dolthub.com/sql-reference/supported-clients/clients)
- [Use Dolt With MySQL Connector in Python](https://www.dolthub.com/blog/2024-11-01-dolt-with-mysql-connector-in-python/)
- [SQLAlchemy 2.0 Connection Pooling](https://docs.sqlalchemy.org/en/20/core/pooling.html)
- [aiomysql Pool Documentation](https://aiomysql.readthedocs.io/en/latest/pool.html)

---

## 4. Branch/Commit Operations

### Creating and Switching Branches

```sql
-- Create a new branch from current HEAD
CALL DOLT_BRANCH('myNewBranch');

-- Create and switch to new branch
CALL DOLT_CHECKOUT('-b', 'newBranchName');

-- Switch to existing branch
CALL DOLT_CHECKOUT('feature-branch');

-- Delete a branch
CALL DOLT_BRANCH('-d', 'branchToDelete');
```

### Commit Operations

```sql
-- Basic commit (staged changes only)
CALL DOLT_COMMIT('-m', 'commit message');

-- Stage and commit all modified tables
CALL DOLT_COMMIT('-a', '-m', 'commit message');

-- Stage all tables including new ones
CALL DOLT_COMMIT('-A', '-m', 'commit message');

-- With explicit author
CALL DOLT_COMMIT('-m', 'message', '--author', 'John Doe <john@example.com>');

-- Allow empty commits
CALL DOLT_COMMIT('-m', 'message', '--allow-empty');

-- Skip if nothing to commit (no error)
CALL DOLT_COMMIT('-m', 'message', '--skip-empty');
```

### Python Integration

```python
import mysql.connector

config = {
    'user': 'root',
    'password': '',
    'host': '127.0.0.1',
    'database': 'mydb/feature-branch',  # database/branch format
}

cnx = mysql.connector.connect(**config)
cursor = cnx.cursor()

# Check current branch
cursor.execute("SELECT active_branch()")
print(cursor.fetchone()[0])

# Create and switch to new branch
cursor.callproc("DOLT_CHECKOUT", ("-b", "my-feature"))

# Make changes and commit
cursor.execute("INSERT INTO users (name) VALUES ('Alice')")
cursor.callproc("DOLT_COMMIT", ("-Am", "Add user Alice"))

# View commit history
cursor.execute("SELECT commit_hash, committer, message FROM dolt_log")
for row in cursor:
    print(f'{row[0][:8]}: {row[2]} ({row[1]})')

# Merge branches
cursor.callproc("DOLT_CHECKOUT", ("main",))
cursor.callproc("DOLT_MERGE", ("my-feature",))

cursor.close()
cnx.close()
```

### Branch Management System Tables

```sql
-- List all branches
SELECT name, hash, latest_commit_message, dirty FROM dolt_branches;

-- View commit history
SELECT commit_hash, committer, message, date FROM dolt_log;

-- All commits in database (all branches)
SELECT * FROM dolt_commits;
```

### Working with Multiple Branches

- Each branch functions as an isolated database instance
- Multiple clients can connect to different branches simultaneously
- **Cannot commit changes to multiple branches in a single transaction**

```sql
-- Cross-branch queries (read-only)
SELECT * FROM `mydatabase/feature-branch`.accounts;
SELECT * FROM `mydatabase/main`.accounts;
```

**Sources**:
- [Dolt SQL Procedures](https://docs.dolthub.com/sql-reference/version-control/dolt-sql-procedures)
- [Using Branches](https://docs.dolthub.com/sql-reference/version-control/branches)
- [Dolt Basics: Branches](https://www.dolthub.com/blog/2025-03-10-dolt-basics-branches/)

---

## 5. Python Clients and ORMs

### DoltPy Status: Deprecated

DoltPy is in maintenance mode. The Dolt team recommends using standard MySQL Python clients instead.

### SQLAlchemy ORM Integration

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Date, text

class Base(DeclarativeBase):
    pass

class Employee(Base):
    __tablename__ = "employees"
    id: Mapped[int] = mapped_column(primary_key=True)
    last_name: Mapped[str] = mapped_column(String(255))
    first_name: Mapped[str] = mapped_column(String(255))
    start_date: Mapped[Date] = mapped_column(Date)

# Dolt operations via SQLAlchemy
def dolt_commit(engine, author, message):
    with engine.connect() as conn:
        conn.execute(text("CALL DOLT_ADD('-A')"))
        result = conn.execute(
            text(f"CALL DOLT_COMMIT('--skip-empty', '--author', '{author}', '-m', '{message}')")
        )
        for row in result:
            print(f"Created commit: {row[0]}")
```

### Django ORM Integration

```python
# settings.py
DATABASES = {
    'default': {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "mydatabase",
        "USER": "root",
        "HOST": "127.0.0.1",
        "PORT": "3306",
    }
}

# Read-only model for Dolt commit log
class Commit(models.Model):
    commit_hash = models.CharField(primary_key=True, max_length=20)
    committer = models.CharField(max_length=100)
    date = models.DateTimeField()
    message = models.TextField()

    class Meta:
        managed = False
        db_table = "dolt_log"
```

### Alembic Migrations

Alembic works with Dolt via SQLAlchemy. Dolt's version control operates independently from Alembic migrations.

```bash
alembic init alembic
alembic revision --autogenerate -m "Initial migration"
alembic upgrade head
```

### doltcli - Lightweight CLI Wrapper

Active development, minimal dependencies (~100KB):

```bash
pip install doltcli
```

Useful for data engineering scripts wrapping Dolt CLI commands.

### Key Dolt Stored Procedures

| Procedure | Purpose | Example |
|-----------|---------|---------|
| `DOLT_ADD()` | Stage changes | `CALL DOLT_ADD('-A')` |
| `DOLT_COMMIT()` | Create commit | `CALL DOLT_COMMIT('-m', 'message')` |
| `DOLT_BRANCH()` | Manage branches | `CALL DOLT_BRANCH('new-branch')` |
| `DOLT_CHECKOUT()` | Switch branches | `CALL DOLT_CHECKOUT('branch')` |
| `DOLT_MERGE()` | Merge branches | `CALL DOLT_MERGE('feature')` |
| `DOLT_RESET()` | Reset state | `CALL DOLT_RESET('--hard')` |

**Sources**:
- [Getting Started: SQLAlchemy and Dolt](https://www.dolthub.com/blog/2023-07-12-sql-alchemy-getting-started/)
- [Getting Started: Django and Dolt](https://www.dolthub.com/blog/2024-01-31-dolt-django/)
- [DoltCLI GitHub](https://github.com/dolthub/doltcli)
- [Dolt SQL Procedures](https://docs.dolthub.com/sql-reference/version-control/dolt-sql-procedures)

---

## 6. Diffing and Merge Capabilities

### DOLT_DIFF() Table Function

```sql
-- Compare branches
SELECT * FROM DOLT_DIFF('main', 'feature_branch', 'inventory');

-- Three-dot notation (diff from divergence point)
SELECT * FROM DOLT_DIFF('main...feature_branch', 'inventory');

-- With skinny option (only changed columns)
SELECT * FROM DOLT_DIFF('--skinny', 'HEAD~1', 'HEAD', 'table_name');
```

**Result Schema**:
```
from_commit, from_commit_date, to_commit, to_commit_date,
diff_type, from_<column>, to_<column>...
```

### Merge Operations

```sql
-- Basic merge
CALL DOLT_MERGE('branch-name');

-- Force merge commit even for fast-forward
CALL DOLT_MERGE('branch-name', '--no-ff', '-m', 'merge message');

-- Abort in-progress merge
CALL DOLT_MERGE('--abort');
```

**Return Values**:
| Field | Description |
|-------|-------------|
| `hash` | Resulting merge commit hash |
| `fast_forward` | 1 if fast-forward, 0 otherwise |
| `conflicts` | Number of conflicts detected |

### Three-Way Merge Process

1. Find merge base in commit graph
2. Merge schemas (table definitions)
3. Resolve schema conflicts
4. Merge table data using primary keys as identifiers
5. Resolve data conflicts
6. Resolve constraint violations

**Conflict Detection**:
- Conflicts occur when both branches modify same `[primary key, column]` pair to different values
- Identical modifications merge cleanly
- Different columns in the same row merge without conflict

### Conflict Resolution

```sql
-- View conflicts
SELECT * FROM dolt_conflicts;
SELECT * FROM dolt_conflicts_employees;

-- Accept "ours" (current branch)
DELETE FROM dolt_conflicts_employees;

-- Accept "theirs" (incoming)
REPLACE INTO employees
SELECT their_id, their_name, their_department
FROM dolt_conflicts_employees;
DELETE FROM dolt_conflicts_employees;

-- Resolve all tables
CALL DOLT_CONFLICTS_RESOLVE('--ours', '.');
CALL DOLT_CONFLICTS_RESOLVE('--theirs', '.');
```

### Diff Performance

Diff time scales with **size of changes**, not total table size - enabled by Prolly Tree architecture.

```sql
-- Paginate large diffs
SELECT * FROM DOLT_DIFF('main', 'feature', 'large_table')
WHERE diff_type = 'modified'
LIMIT 100 OFFSET 0;

-- Count changes efficiently
SELECT diff_type, COUNT(*)
FROM DOLT_DIFF('main', 'feature', 'large_table')
GROUP BY diff_type;
```

**Sources**:
- [Dolt SQL Functions](https://docs.dolthub.com/sql-reference/version-control/dolt-sql-functions)
- [Three-Way Merge in Dolt](https://www.dolthub.com/blog/2024-06-19-threeway-merge/)
- [Dolt Conflicts Documentation](https://docs.dolthub.com/concepts/dolt/git/conflicts)

---

## 7. Performance Characteristics

### Benchmark Results (December 2025)

Dolt has achieved **MySQL parity** on Sysbench:

| Benchmark Type | Dolt vs MySQL | Notes |
|----------------|---------------|-------|
| **Overall Mean** | 0.99x | Parity achieved |
| **Write Operations** | 0.91x (9% faster) | Dolt advantage |
| **Read Operations** | ~1.10x (10% slower) | MySQL advantage |
| **Point Selects** | 1.35x | MySQL advantage |
| **Index Scans** | 0.66x | Dolt advantage |

### Performance Evolution

| Year | Dolt vs MySQL |
|------|---------------|
| 2021 | ~15x slower |
| 2023 | ~2x slower |
| 2024 | 10% slower |
| Dec 2025 | **0.99x (parity)** |

### Storage Efficiency

**Prolly Trees** enable structural sharing:
- Single HEAD: Less storage than MySQL (better compression)
- Versioned history: Proportional to diffs, not full copies
- After GC: Minimal overhead for version metadata

### Scaling Considerations

| Factor | Limit/Recommendation |
|--------|---------------------|
| Database size | ~100GB documented limit |
| Write throughput | ~300 writes/second (serialized) |
| Concurrent connections | 1000 default, configurable |
| TPC-C throughput | ~40% of MySQL |

### Production Sizing

| Database Size | Recommended RAM |
|---------------|-----------------|
| Minimum | 2GB |
| Standard | 4-8GB |
| Rule of thumb | 10-20% of disk size |
| 100GB database | 10-16GB |

### Optimization Tips

1. **Batch writes** - Aggregate inserts into large batches
2. **Garbage collection** - Run `CALL dolt_gc()` after bulk imports (auto-GC default in 1.75+)
3. **Read replicas** - Scale reads horizontally
4. **Memory allocation** - 10-20% of database disk size as RAM

**Sources**:
- [Dolt is as Fast as MySQL on Sysbench](https://www.dolthub.com/blog/2025-12-04-dolt-is-as-fast-as-mysql/)
- [How Dolt Got as Fast as MySQL](https://www.dolthub.com/blog/2025-12-12-how-dolt-got-as-fast-as-mysql/)
- [Sizing Your Dolt Instance](https://www.dolthub.com/blog/2023-12-06-sizing-your-dolt-instance/)

---

## 8. Self-Hosting Options

### Running Dolt SQL Server

```bash
# Basic start
dolt sql-server

# With config file (recommended)
dolt sql-server --config config.yaml
```

### Configuration (config.yaml)

```yaml
log_level: info
behavior:
  read_only: false
  autocommit: true
  dolt_transaction_commit: false

listener:
  host: 0.0.0.0
  port: 3306
  max_connections: 1000

metrics:
  host: "0.0.0.0"
  port: 11228

remotesapi:
  port: 50051
```

### Docker Deployment

```bash
# Simple start
docker run -p 3306:3306 dolthub/dolt-sql-server:latest

# With password and persistent storage
docker run -e DOLT_ROOT_PASSWORD=secret \
           -e DOLT_ROOT_HOST=% \
           -p 3306:3306 \
           -v /path/to/data:/var/lib/dolt \
           dolthub/dolt-sql-server:latest
```

**Volume Mount Points**:
| Directory | Purpose |
|-----------|---------|
| `/var/lib/dolt/` | Data storage |
| `/etc/dolt/servercfg.d/` | Server config (.yaml) |
| `/docker-entrypoint-initdb.d/` | Init scripts |

### Kubernetes Deployment

Deploy as **StatefulSet** for:
- Stable network identities
- Persistent volume claims
- Ordered pod management

**doltclusterctl** - Cluster management tool:
```bash
kubectl run doltclusterctl --image=dolthub/doltclusterctl:latest \
  -- applyprimarylabels -n dolt-namespace
```

**Note**: No official Helm chart from DoltHub.

### Replication for High Availability

**Direct-to-Standby (Recommended for HA)**:

Primary (`dolt-1.db`):
```yaml
cluster:
  standby_remotes:
    - name: standby
      remote_url_template: http://dolt-2.db:50051/{database}
  bootstrap_role: primary
  bootstrap_epoch: 1
  remotesapi:
    port: 50051
```

Standby (`dolt-2.db`):
```yaml
cluster:
  standby_remotes:
    - name: standby
      remote_url_template: http://dolt-1.db:50051/{database}
  bootstrap_role: standby
  bootstrap_epoch: 1
  remotesapi:
    port: 50051
```

**Failover**:
```sql
-- On current primary: demote
CALL dolt_assume_cluster_role('standby', 2);

-- On target standby: promote
CALL dolt_assume_cluster_role('primary', 2);
```

### Backup Methods

1. **Remotes** (committed data):
   ```sql
   CALL dolt_push('backup', 'main');
   ```

2. **dolt_backup** (committed + uncommitted):
   ```sql
   CALL dolt_backup('add', 'my-backup', 's3://bucket/full-backup');
   CALL dolt_backup('sync', 'my-backup');
   ```

3. **mysqldump**:
   ```bash
   mysqldump -h localhost -P 3306 -u root database_name > backup.sql
   ```

### Monitoring (Prometheus)

Enable metrics in config.yaml, access at `http://localhost:11228/metrics`.

Key metrics:
- `dss_concurrent_connections`
- `dss_query_duration_bucket`
- `dss_replication_lag`
- `go_memstats_alloc_bytes`

**Sources**:
- [Server Configuration](https://docs.dolthub.com/sql-reference/server/configuration)
- [Replication](https://docs.dolthub.com/sql-reference/server/replication)
- [Docker Installation](https://docs.dolthub.com/introduction/installation/docker)
- [Productionizing Dolt](https://www.dolthub.com/blog/2024-11-27-productionizing-dolt/)

---

## 9. YouLab Current State: File-Based Storage

### Current Architecture

YouLab uses **file-based git storage** (GitPython) for user memory blocks:

```
.data/users/{user_id}/
    .git/                    # Git repository
    memory-blocks/           # Markdown files with YAML frontmatter
        student.md
        journey.md
    pending_diffs/           # JSON files for agent-proposed changes
        {diff_id}.json
```

### Key Implementation Files

- `src/youlab_server/storage/git.py` - GitUserStorage, GitUserStorageManager
- `src/youlab_server/storage/blocks.py` - UserBlockManager
- `src/youlab_server/storage/diffs.py` - PendingDiff, PendingDiffStore

### Patterns to Preserve in Migration

1. **Approval Workflow**: Agent edits create `PendingDiff` objects requiring user approval
2. **Version History**: Full commit history with author attribution
3. **Factory/Repository Pattern**: Storage managers per user
4. **Letta Sync**: Blocks sync to Letta's core memory

### No Database Currently

- No SQLAlchemy models in YouLab server
- No database engine or sessions
- OpenWebUI uses SQLAlchemy separately (not accessed by YouLab server)

---

## 10. Recommendations for YouLab v2

### Why Dolt Fits YouLab

1. **Version Control Semantics**: Matches current GitPython approach
2. **SQL Queryability**: Enables complex queries on user data
3. **Audit Trail**: Built-in with `dolt_history_*` tables
4. **Diff/Merge**: Native support for conflict resolution
5. **Python Integration**: SQLAlchemy works seamlessly

### Migration Considerations

| Current Pattern | Dolt Equivalent |
|-----------------|-----------------|
| `.git` directory | Dolt commit graph |
| Markdown files | SQL tables |
| YAML frontmatter | Table columns |
| `pending_diffs/*.json` | `pending_diffs` table |
| GitPython commits | `DOLT_COMMIT()` |
| File restore | `dolt_history_*` + rollback |

### Suggested Schema

```sql
-- Core memory blocks
CREATE TABLE memory_blocks (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(255) NOT NULL,
    label VARCHAR(100) NOT NULL,
    schema_ref VARCHAR(255),
    title VARCHAR(255),
    body TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY (user_id, label)
);

-- Pending agent edits (approval workflow)
CREATE TABLE pending_diffs (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    block_label VARCHAR(100) NOT NULL,
    operation ENUM('append', 'replace', 'full_replace') NOT NULL,
    new_value TEXT,
    current_value TEXT,
    reasoning TEXT,
    confidence ENUM('low', 'medium', 'high'),
    agent_id VARCHAR(255),
    status ENUM('pending', 'approved', 'rejected', 'superseded', 'expired'),
    created_at TIMESTAMP,
    reviewed_at TIMESTAMP
);
```

### Connection Pattern

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(
    f"mysql+aiomysql://root@localhost:3306/youlab/{user_id}",  # Per-user branch
    pool_size=20,
    pool_recycle=1800,
    pool_pre_ping=True,
)
```

---

## Additional Resources

### Official Documentation
- [Dolt Documentation](https://docs.dolthub.com)
- [Version Control SQL Reference](https://docs.dolthub.com/sql-reference/version-control)
- [System Tables](https://docs.dolthub.com/sql-reference/version-control/dolt-system-tables)
- [Stored Procedures](https://docs.dolthub.com/sql-reference/version-control/dolt-sql-procedures)

### GitHub Repositories
- [dolthub/dolt](https://github.com/dolthub/dolt) - Main repository
- [dolthub/doltgresql](https://github.com/dolthub/doltgresql) - PostgreSQL variant
- [dolthub/doltclusterctl](https://github.com/dolthub/doltclusterctl) - Kubernetes management
- [dolthub/doltcli](https://github.com/dolthub/doltcli) - Python CLI wrapper

### Key Blog Posts
- [Dolt is as Fast as MySQL (Dec 2025)](https://www.dolthub.com/blog/2025-12-04-dolt-is-as-fast-as-mysql/)
- [Schema Migrations in Dolt](https://www.dolthub.com/blog/2024-04-18-dolt-schema-migrations/)
- [SQLAlchemy and Dolt](https://www.dolthub.com/blog/2023-07-12-sql-alchemy-getting-started/)
- [Django and Dolt](https://www.dolthub.com/blog/2024-01-31-dolt-django/)
- [Productionizing Dolt](https://www.dolthub.com/blog/2024-11-27-productionizing-dolt/)

---

## Open Questions

1. **Per-user branches vs single database**: Should each user get a branch (YouLab current approach) or should we use a single database with user_id columns?

2. **Letta sync strategy**: How to maintain Letta block sync with Dolt as source of truth?

3. **Migration path**: Incremental migration from GitPython or full replacement?

4. **Hosted vs self-hosted**: DoltHub offers [Hosted Dolt](https://hosted.doltdb.com/) for managed deployment.
