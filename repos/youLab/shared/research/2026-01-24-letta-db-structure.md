---
date: 2026-01-25T02:32:04+0000
researcher: ARI
git_commit: 2611ba2
branch: main
repository: YouLab
topic: "Letta Database Structure Analysis for YouLab v2 Migration"
tags: [research, letta, database, postgresql, orm, migration, youlab-v2]
status: complete
last_updated: 2026-01-25
last_updated_by: ARI
---

# Research: Letta Database Structure Analysis for YouLab v2 Migration

**Date**: 2026-01-25T02:32:04+0000
**Researcher**: ARI
**Git Commit**: 2611ba2
**Branch**: main
**Repository**: YouLab

## Research Question

Analyze Letta's database structure for YouLab v2 migration. Examine Letta's PostgreSQL schema, agent state storage, memory block format, tool definitions, message/conversation storage, user/org models. Key questions: what needs migration, patterns to adopt, complexity to eliminate.

## Summary

Letta uses a complex SQLAlchemy-based ORM with **50+ database models** across PostgreSQL (with pgvector for embeddings) or SQLite. The schema covers agents, memory blocks, messages, tools, users/organizations, vector passages, jobs, runs, and extensive junction tables for many-to-many relationships.

**Key Findings for Migration:**
- **YouLab does NOT directly touch Letta's database** - all interaction is via HTTP SDK client
- **Core tables to understand**: `agents`, `block`, `messages`, `tools`, `users`, `organizations`
- **Complexity to eliminate**: Block sync layer, sandbox tool duplication, global state patterns
- **Patterns to adopt**: Block sharing via junction tables, optimistic locking, soft deletes

## Detailed Findings

### Letta ORM Architecture Overview

Letta's database layer is built on SQLAlchemy with these key characteristics:

**Base Classes** (`.venv/lib/python3.12/site-packages/letta/orm/`):
- `Base` - SQLAlchemy DeclarativeBase
- `CommonSqlalchemyMetaMixins` - Adds `created_at`, `updated_at`, `is_deleted` fields
- `SqlalchemyBase` - Full CRUD operations, pagination, access control predicates

**Common Mixins**:
- `OrganizationMixin` - Adds `organization_id` FK
- `ProjectMixin` - Adds optional `project_id` FK
- `TemplateMixin` - Adds template-related fields
- `AgentMixin` - Adds `agent_id` FK

### Core Entity Tables

#### 1. Agents Table (`letta/orm/agent.py`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | Auto-generated `agent-{uuid}` |
| `agent_type` | String | Agent classification (enum) |
| `name` | String | Human-readable identifier |
| `description` | String | Agent description |
| `system` | String | System prompt |
| `message_ids` | JSON | In-context memory message IDs |
| `llm_config` | Custom | LLM config (model, temperature, etc.) |
| `embedding_config` | Custom | Embedding model settings |
| `tool_rules` | Custom | Tool execution constraints |
| `metadata_` | JSON | Arbitrary agent metadata |
| `message_buffer_autoclear` | Boolean | Stateless mode flag |
| `enable_sleeptime` | Boolean | Background memory management |
| `last_run_completion` | DateTime | Last run timestamp |
| `last_run_duration_ms` | Integer | Last run duration |
| `last_stop_reason` | String | Last stop reason |
| `timezone` | String | Agent timezone |
| `hidden` | Boolean | Visibility flag |

**Indexes**:
- `ix_agents_created_at` (created_at, id)
- `ix_agents_organization_id_deployment_id` (organization_id, deployment_id)
- `ix_agents_project_id` (project_id)

**Key Relationships**:
- `core_memory` - Many-to-many with Block via `blocks_agents`
- `tools` - Many-to-many via `tools_agents`
- `sources` - Many-to-many via `sources_agents`
- `runs` - One-to-many execution records
- `identities` - Many-to-many via `identities_agents`
- `groups` - Many-to-many via `groups_agents`

#### 2. Block Table (`letta/orm/block.py`)

Memory blocks are the unit of structured agent memory.

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | Block identifier |
| `label` | String | Block type (e.g., 'human', 'persona', 'system') |
| `template_name` | String | Human-readable unique name |
| `description` | String | Context about block purpose |
| `value` | Text | Core memory content |
| `limit` | BigInteger | Character limit (default: `CORE_MEMORY_BLOCK_CHAR_LIMIT`) |
| `metadata_` | JSON | Arbitrary metadata |
| `is_template` | Boolean | Template flag |
| `preserve_on_migration` | Boolean | Preserve on template migration |
| `read_only` | Boolean | Prevent agent modifications |
| `hidden` | Boolean | Visibility flag |
| `version` | Integer | **Optimistic locking counter** |
| `current_history_entry_id` | FK | Points to current BlockHistory entry |

**Indexes**:
- `unique_block_id_label` - Unique constraint (id, label)
- `created_at_label_idx` (created_at, label)
- `ix_block_is_template`, `ix_block_hidden`
- `ix_block_org_project_template` (organization_id, project_id, is_template)

**Key Features**:
- **Optimistic locking** via `version` column (SQLAlchemy built-in)
- **History tracking** via `BlockHistory` relationship
- **Value validation** via SQLAlchemy `before_insert`/`before_update` events

**Key Relationships**:
- `agents` - Many-to-many via `blocks_agents`
- `identities` - Many-to-many via `identities_blocks`
- `groups` - Many-to-many via `groups_blocks`

#### 3. Message Table (`letta/orm/message.py`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | Message identifier |
| `role` | String | user/assistant/system/tool |
| `text` | String | Legacy message content |
| `content` | Custom JSON | List of MessageContent objects |
| `model` | String | LLM model used |
| `name` | String | Name for multi-agent scenarios |
| `tool_calls` | Custom JSON | OpenAI-style tool call objects |
| `tool_call_id` | String | Tool invocation identifier |
| `tool_returns` | Custom JSON | Tool execution results |
| `step_id` | FK | Reference to execution step |
| `run_id` | FK | Reference to execution run |
| `sequence_id` | BigInteger | **Monotonically increasing** for ordering |
| `group_id` | String | Multi-agent group identifier |
| `sender_id` | String | Sender (identity or agent) ID |
| `approve` | Boolean | Tool call approval status |
| `denial_reason` | String | Tool denial reason |
| `approvals` | Custom JSON | Approval responses |

**Indexes**:
- `ix_messages_agent_created_at` (agent_id, created_at)
- `ix_messages_created_at` (created_at, id)
- `ix_messages_agent_sequence` (agent_id, sequence_id)
- `ix_messages_org_agent` (organization_id, agent_id)
- `ix_messages_run_sequence` (run_id, sequence_id)
- `idx_messages_step_id` (step_id)

**Key Feature**: SQLite gets custom sequence handling via `message_sequence` table (atomic increment).

#### 4. Tool Table (`letta/orm/tool.py`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | Tool identifier |
| `name` | String | Tool display name |
| `tool_type` | Enum | CUSTOM, etc. |
| `description` | String | Tool description |
| `source_type` | Enum | Source code type (json) |
| `source_code` | String | Function source code |
| `json_schema` | JSON | OpenAI-compatible tool schema |
| `args_json_schema` | JSON | Function arguments schema |
| `return_char_limit` | Integer | Max return characters |
| `tags` | JSON | Metadata tags |
| `pip_requirements` | JSON | Python dependencies |
| `npm_requirements` | JSON | Node dependencies |
| `default_requires_approval` | Boolean | Human approval flag |
| `enable_parallel_execution` | Boolean | Parallel execution flag |
| `metadata_` | JSON | Additional metadata |

**Constraints**:
- `uix_name_organization` - Unique (name, organization_id)
- `uix_organization_project_name` - Unique (organization_id, project_id, name)

#### 5. User Table (`letta/orm/user.py`)

Minimal user model:

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | User identifier |
| `name` | String | Display name |
| `organization_id` | FK | Parent organization |

**Relationships**: `organization`, `jobs`

#### 6. Organization Table (`letta/orm/organization.py`)

Top-level multi-tenancy unit:

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | Organization identifier |
| `name` | String | Display name |
| `privileged_tools` | Boolean | Access to privileged tools |

**Relationships** (cascade delete-orphan):
- `users`, `tools`, `blocks`, `agents`, `sources`, `messages`
- `source_passages`, `archival_passages`, `passage_tags`
- `providers`, `provider_models`, `identities`, `groups`
- `llm_batch_jobs`, `llm_batch_items`, `jobs`, `runs`
- `sandbox_configs`, `sandbox_environment_variables`, `agent_environment_variables`
- `mcp_servers`, `archives`, `provider_traces`

### Junction Tables (Many-to-Many Relationships)

| Table | Links | Key Constraints |
|-------|-------|-----------------|
| `blocks_agents` | Block ↔ Agent | Unique (agent_id, block_label), Unique (agent_id, block_id) |
| `tools_agents` | Tool ↔ Agent | Unique (agent_id, tool_id) |
| `sources_agents` | Source ↔ Agent | - |
| `files_agents` | File ↔ Agent | - |
| `archives_agents` | Archive ↔ Agent | - |
| `groups_agents` | Group ↔ Agent | - |
| `groups_blocks` | Group ↔ Block | - |
| `identities_agents` | Identity ↔ Agent | - |
| `identities_blocks` | Identity ↔ Block | - |
| `agents_tags` | Agent ↔ Tags | - |

**`blocks_agents` Detail** (`letta/orm/blocks_agents.py`):
- Composite FK: (block_id, block_label) → (block.id, block.label) with CASCADE
- Enables block sharing across multiple agents

### Block History (`letta/orm/block_history.py`)

Stores historical block states for undo/redo:

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | `block_hist-{uuid}` |
| `block_id` | FK | Parent block (CASCADE delete) |
| `sequence_number` | Integer | Monotonic sequence per block |
| `description` | Text | Snapshot of description |
| `label` | String | Snapshot of label |
| `value` | Text | Snapshot of value |
| `limit` | BigInteger | Snapshot of limit |
| `metadata_` | JSON | Snapshot of metadata |
| `actor_type` | String | Who made the change |
| `actor_id` | String | Editor ID |

**Index**: `ix_block_history_block_id_sequence` (block_id, sequence_number) UNIQUE

### Run & Step Models (`letta/orm/run.py`, `letta/orm/step.py`)

Execution tracking:

**Run Table**:
| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | `run-{uuid}` |
| `agent_id` | FK | Parent agent |
| `status` | Enum | created, running, completed, failed |
| `completed_at` | DateTime | Completion timestamp |
| `stop_reason` | String | Stop reason type |
| `background` | Boolean | Background mode flag |
| `metadata_` | JSON | Run metadata |
| `request_config` | JSON | Request configuration |
| `callback_url` | String | POST callback on completion |
| `ttft_ns` | BigInteger | Time to first token (nanoseconds) |
| `total_duration_ns` | BigInteger | Total duration (nanoseconds) |

**Relationships**: `agent`, `steps`, `messages`

### Vector Storage (Archival Memory)

Two passage tables with pgvector support:

**`source_passages`** - External data sources (files, documents)
**`archival_passages`** - Agent's long-term archival memory

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String PK | Passage identifier |
| `text` | Text | Passage content |
| `embedding` | Vector(MAX_EMBEDDING_DIM) | pgvector embedding |
| `embedding_config` | Custom | Embedding model config |
| `metadata_` | JSON | Supplementary info |
| `tags` | JSON | Associated tags |

**Database-specific handling**:
- PostgreSQL: Uses `pgvector.sqlalchemy.Vector`
- SQLite: Uses custom `CommonVector` with cosine distance function

### Custom Column Types (`letta/orm/custom_columns.py`)

Letta defines custom SQLAlchemy column types for complex JSON structures:
- `LLMConfigColumn` - LLM configuration
- `EmbeddingConfigColumn` - Embedding configuration
- `ToolRulesColumn` - Tool execution rules
- `ResponseFormatColumn` - Response format union
- `CompactionSettingsColumn` - Memory compaction settings
- `MessageContentColumn` - Message content parts
- `ToolCallColumn` - Tool call objects
- `ToolReturnColumn` - Tool return objects
- `ApprovalsColumn` - Approval responses

## How YouLab Currently Uses Letta

**Key Finding**: YouLab does NOT directly access Letta's database. All interaction is via the Letta SDK HTTP client.

### Current Integration Points

| Component | Letta Interaction |
|-----------|-------------------|
| `server/agents.py:AgentManager` | Creates/retrieves agents via SDK |
| `storage/blocks.py:UserBlockManager` | Syncs blocks to Letta via SDK |
| `tools/sandbox.py` | HTTP tools that call back to YouLab server |
| `pipelines/letta_pipe.py` | Streams messages via SDK |
| `background/factory.py` | Creates background agents via SDK |

### YouLab-Specific Patterns

**Agent Naming**: `youlab_{user_id}_{agent_type}`
**Block Naming**: `youlab_user_{user_id}_{label}` or `youlab_shared_{course_id}_{block_label}`
**Agent Metadata**: Stores `youlab_user_id`, `youlab_agent_type`, `course_id`, `course_version`

## Migration Analysis for YouLab v2

### What Needs Migration

From the v2 greenfield plan (`thoughts/shared/plans/2026-01-24-youlab-v2-greenfield.md`):

| Current | New (v2) | Migration Approach |
|---------|----------|-------------------|
| Letta blocks | Dolt blocks table | Export block content, reimport to Dolt |
| Git user repos | Dolt single database | Data migration tooling |
| Letta messages | Keep in Honcho | No change |
| Pending diffs | Dolt pending_diffs table | Migrate format |
| Block sync layer | Removed | Delete code |

### Patterns to Adopt from Letta

1. **Optimistic Locking** - Block's `version` column pattern for concurrent edits
2. **Soft Deletes** - `is_deleted` flag instead of hard deletes
3. **Junction Tables for Sharing** - `blocks_agents` pattern for multi-agent block sharing
4. **History Tracking** - `BlockHistory` pattern for undo/redo
5. **Sequence Numbers** - Monotonic ordering for messages
6. **Actor Tracking** - `created_by_id`, `last_updated_by_id` fields

### Complexity to Eliminate

1. **Block Sync Layer** - `storage/blocks.py` Letta sync code
2. **Sandbox Tool Duplication** - HTTP tools duplicated for Docker sandbox
3. **Global State Patterns** - `set_letta_client()`, `set_user_context()` globals
4. **Agent Caching** - `AgentManager._cache` in-memory agent ID cache
5. **Complex Streaming** - Multiple streaming layers (Letta SDK → SSE → OpenWebUI)
6. **Organization Model** - Letta's multi-org overhead (YouLab is single-org)

### Dolt Schema Recommendations

Based on Letta patterns and v2 needs:

```sql
-- Core tables (from v2 plan)
CREATE TABLE blocks (
    id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    label VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    metadata JSON,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, label)
);

CREATE TABLE pending_diffs (
    id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    block_label VARCHAR(255) NOT NULL,
    diff_content TEXT NOT NULL,
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id, block_label) REFERENCES blocks(user_id, label)
);

CREATE TABLE module_chats (
    id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    agent_id VARCHAR(255) NOT NULL,
    module_id VARCHAR(255) NOT NULL,
    owui_chat_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (user_id, agent_id, module_id)
);

CREATE TABLE user_api_keys (
    id VARCHAR(255) PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL UNIQUE,
    owui_api_key VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Optional: block history for undo/redo
CREATE TABLE block_history (
    id VARCHAR(255) PRIMARY KEY,
    block_id VARCHAR(255) NOT NULL,
    sequence_number INT NOT NULL,
    value TEXT NOT NULL,
    metadata JSON,
    actor_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (block_id) REFERENCES blocks(id) ON DELETE CASCADE,
    UNIQUE (block_id, sequence_number)
);
```

## Code References

- `.venv/lib/python3.12/site-packages/letta/orm/agent.py` - Agent ORM model
- `.venv/lib/python3.12/site-packages/letta/orm/block.py` - Block ORM model
- `.venv/lib/python3.12/site-packages/letta/orm/message.py` - Message ORM model
- `.venv/lib/python3.12/site-packages/letta/orm/tool.py` - Tool ORM model
- `.venv/lib/python3.12/site-packages/letta/orm/user.py` - User ORM model
- `.venv/lib/python3.12/site-packages/letta/orm/organization.py` - Organization ORM model
- `.venv/lib/python3.12/site-packages/letta/orm/blocks_agents.py` - Block-Agent junction
- `.venv/lib/python3.12/site-packages/letta/orm/block_history.py` - Block history
- `.venv/lib/python3.12/site-packages/letta/orm/passage.py` - Vector passage storage
- `.venv/lib/python3.12/site-packages/letta/orm/run.py` - Execution run model
- `.venv/lib/python3.12/site-packages/letta/orm/sqlalchemy_base.py` - Base ORM with CRUD
- `src/youlab_server/server/agents.py` - AgentManager (Letta SDK usage)
- `src/youlab_server/storage/blocks.py` - UserBlockManager (Letta sync)

## Historical Context (from thoughts/)

### YouLab v2 Migration Plans
- `thoughts/shared/plans/2026-01-24-youlab-v2-greenfield.md` - Primary v2 architecture plan
- `thoughts/shared/plans/2026-01-16-letta-to-agno-greenfield.md` - Letta to Agno rebuild
- `thoughts/shared/research/2026-01-16-letta-to-agno-migration.md` - Migration research

### Letta Framework Analysis
- `thoughts/shared/research/2026-01-16-letta-framework-analysis.md` - Framework analysis
- `thoughts/shared/research/2026-01-13-ARI-82-letta-memory-integration.md` - Memory blocks integration
- `thoughts/shared/research/2026-01-01-letta-sdk-api-reference.md` - SDK API reference

### Memory System Design
- `thoughts/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md` - Memory System MVP
- `thoughts/shared/research/ari-85-memory-block-schema-research.md` - Block schema research
- `thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Memory philosophy

## External Resources

- [Letta Database Configuration](https://docs.letta.com/guides/selfhosting/postgres) - PostgreSQL setup
- [Letta Core Concepts](https://docs.letta.com/core-concepts/) - State persistence
- [Letta Memory Blocks](https://docs.letta.com/guides/agents/memory-blocks/) - Block documentation
- [Letta ORM Source](https://github.com/letta-ai/letta/tree/main/letta/orm) - GitHub ORM directory
- [Letta on AWS Aurora](https://aws.amazon.com/blogs/database/how-letta-builds-production-ready-ai-agents-with-amazon-aurora-postgresql/) - Production patterns

## Open Questions

1. **Block migration scope** - Migrate all historical block versions or just current state?
2. **Optimistic locking in Dolt** - How to implement version-based locking in Dolt?
3. **Vector storage** - If archival memory is needed, where does it live in v2? (Honcho handles dialectic)
4. **Background task queue** - TOML-based, Dolt-backed, or external (Redis)?
5. **Multi-agent blocks** - Is block sharing needed in v2, or is per-user sufficient?
