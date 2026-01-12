---
date: 2026-01-12T03:13:19Z
researcher: Claude
git_commit: 5921af254188eca4151966cebff1e202da4db885
branch: main
repository: YouLab
topic: "OpenWebUI Database Schema Research for YouLab Extensions"
tags: [research, codebase, openwebui, database, sqlalchemy, migrations, modules, memory-blocks]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude
---

# Research: OpenWebUI Database Schema for YouLab Extensions

**Date**: 2026-01-12T03:13:19Z
**Researcher**: Claude
**Git Commit**: 5921af254188eca4151966cebff1e202da4db885
**Branch**: main
**Repository**: YouLab
**Linear Ticket**: ARI-78

## Research Question

Understand the existing OpenWebUI database to plan extensions for:
- Module metadata (course, letta_agent_id, active status)
- Pending diffs table (block, old_value, new_value, agent, status)
- Memory block metadata (last_modified, version, etc.)

## Summary

OpenWebUI uses SQLAlchemy ORM with a dual migration system (legacy Peewee + modern Alembic). The database supports SQLite (default), PostgreSQL, and SQLCipher. **Chat-to-model associations are stored in JSON blobs within the `chat` column, not as foreign keys.** Most models include extensible JSON fields (`data`, `meta`, `settings`, `access_control`) that provide natural extension points. To extend the schema safely, we should:

1. Create new tables via Alembic migrations
2. Use JSON blob fields for flexible metadata
3. Follow OpenWebUI's existing patterns (table accessor classes, Pydantic validation, BigInteger timestamps)

## Detailed Findings

### 1. Core Database Models

#### User Table (`OpenWebUI/open-webui/backend/open_webui/models/users.py:45-76`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String (PK) | User identifier |
| `email` | String | User email |
| `name` | String | Display name |
| `role` | String | User role (admin/user/pending) |
| `info` | JSON | **Arbitrary user information** |
| `settings` | JSON | **User settings (UI prefs, function valves)** |
| `oauth` | JSON | OAuth provider data |
| `last_active_at` | BigInteger | Activity timestamp |
| `created_at`/`updated_at` | BigInteger | Timestamps |

**Extension Point**: The `info` and `settings` JSON fields can store module/memory metadata per-user.

#### Chat Table (`OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-65`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | String (PK) | Chat UUID |
| `user_id` | String | Chat owner |
| `title` | Text | Chat title |
| `chat` | JSON | **Complete chat object (messages, models, history)** |
| `meta` | JSON | **Chat metadata (tags, etc.)** |
| `folder_id` | Text | Folder organization |
| `archived` | Boolean | Archive status |
| `pinned` | Boolean | Pin status |

**Chat JSON Structure**:
```json
{
  "models": ["model-id-1", "model-id-2"],  // Selected models array
  "messages": [...],                        // Array format for API
  "history": {
    "messages": {
      "msg-id": {
        "model": "specific-model-id",       // Per-message model
        "modelName": "Display Name",
        "content": "...",
        "parentId": "...",
        "childrenIds": []
      }
    },
    "currentId": "msg-id"
  }
}
```

**Key Insight**: Model associations are NOT stored as foreign keys. They're embedded in the JSON `chat` blob. This means YouLab's module system cannot use direct table relationships to models.

#### Model Table (`OpenWebUI/open-webui/backend/open_webui/models/models.py:53-102`)

| Column | Type | Purpose |
|--------|------|---------|
| `id` | Text (PK) | Model identifier (e.g., "gpt-4") |
| `user_id` | Text | Model creator |
| `name` | Text | Display name |
| `base_model_id` | Text | Optional proxy target |
| `is_active` | Boolean | Activation status |
| `params` | JSONField | Model parameters |
| `meta` | JSONField | Model metadata (profile_image_url, description, capabilities) |
| `access_control` | JSON | Permission system (null=public, {}=private, or custom ACL) |

**Extension Point**: The `meta` field accepts extra fields via `ConfigDict(extra="allow")`. We could add `course_id`, `letta_agent_id`, `module_config` here.

### 2. Database/ORM Layer

#### Configuration (`OpenWebUI/open-webui/backend/open_webui/internal/db.py`)

```python
# Database URL default (SQLite)
DATABASE_URL = f"sqlite:///{DATA_DIR}/webui.db"

# Base class for all models
metadata_obj = MetaData(schema=DATABASE_SCHEMA)
Base = declarative_base(metadata=metadata_obj)

# Session factory
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine, expire_on_commit=False)
Session = scoped_session(SessionLocal)
```

**Database Types Supported**:
- **SQLite** (default): `sqlite:///{DATA_DIR}/webui.db`
- **PostgreSQL**: Connection pooling with `QueuePool`
- **SQLCipher**: Encrypted SQLite via `DATABASE_PASSWORD`

#### Custom JSONField (`db.py:29-49`)
```python
class JSONField(types.TypeDecorator):
    impl = types.Text  # Serializes to text for cross-DB compatibility

    def process_bind_param(self, value, dialect):
        return json.dumps(value)

    def process_result_value(self, value, dialect):
        return json.loads(value) if value else None
```

### 3. Migration System

OpenWebUI uses a **dual migration system**:

#### Legacy Peewee Migrations (`internal/migrations/001-018_*.py`)
- Executes first on startup at `db.py:78`
- Sequential numbering (001, 002, etc.)
- Tracks applied migrations in `migratehistory` table
- Used for early schema changes

#### Modern Alembic Migrations (`migrations/versions/*.py`)
- Executes after Peewee at `config.py:70`
- Hash-based revision IDs (e.g., `7e5b5dc7342b`)
- Uses `down_revision` dependency chain
- Tracks in `alembic_version` table

**Creating New Migrations**:
```bash
DATABASE_URL=<url> alembic revision --autogenerate -m "description"
```

#### Migration Patterns

**Adding New Table** (`migrations/versions/af906e964978_add_feedback_table.py`):
```python
def upgrade():
    op.create_table(
        "feedback",
        sa.Column("id", sa.Text(), primary_key=True),
        sa.Column("user_id", sa.Text(), nullable=True),
        sa.Column("data", sa.JSON(), nullable=True),
        sa.Column("meta", sa.JSON(), nullable=True),
        sa.Column("created_at", sa.BigInteger(), nullable=False),
        sa.Column("updated_at", sa.BigInteger(), nullable=False),
    )

def downgrade():
    op.drop_table("feedback")
```

**Adding Column** (`migrations/versions/c69f45358db4_add_folder_table.py`):
```python
def upgrade():
    op.add_column("chat", sa.Column("folder_id", sa.Text(), nullable=True))

def downgrade():
    op.drop_column("chat", "folder_id")
```

**Safe Table Creation** (`migrations/versions/7e5b5dc7342b_init.py`):
```python
from open_webui.migrations.util import get_existing_tables

def upgrade():
    existing_tables = set(get_existing_tables())
    if "my_table" not in existing_tables:
        op.create_table(...)
```

### 4. All Model Files Inventory

| File | Models | Key JSON Fields |
|------|--------|-----------------|
| `users.py` | User, ApiKey | `info`, `settings`, `oauth` |
| `chats.py` | Chat, ChatFile | `chat`, `meta` |
| `models.py` | Model | `params`, `meta`, `access_control` |
| `functions.py` | Function | `meta`, `valves` |
| `tools.py` | Tool | `specs`, `meta`, `valves`, `access_control` |
| `knowledge.py` | Knowledge, KnowledgeFile | `meta`, `access_control` |
| `files.py` | File | `data`, `meta`, `access_control` |
| `folders.py` | Folder | `items`, `meta`, `data` |
| `groups.py` | Group, GroupMember | `data`, `meta`, `permissions` |
| `channels.py` | Channel, ChannelMember, ChannelFile, ChannelWebhook | `data`, `meta`, `access_control` |
| `messages.py` | Message, MessageReaction | `data`, `meta` |
| `notes.py` | Note | `data`, `meta`, `access_control` |
| `feedbacks.py` | Feedback | `data`, `meta`, `snapshot` |
| `memories.py` | Memory | (no JSON fields) |
| `tags.py` | Tag | `meta` |
| `auths.py` | Auth | (no JSON fields) |
| `prompts.py` | Prompt | `access_control` |
| `oauth_sessions.py` | OAuthSession | (no JSON fields) |

### 5. Existing Extension Patterns

#### Table Accessor Pattern
Every model has a singleton table class:
```python
class ModelsTable:
    def get_model_by_id(self, id: str) -> Optional[ModelModel]:
        with get_db() as db:
            model = db.query(Model).filter_by(id=id).first()
            return ModelModel.model_validate(model)

    def insert_new_model(self, form_data: ModelForm, ...):
        with get_db() as db:
            model = Model(**form_data.model_dump())
            db.add(model)
            db.commit()
            return model

Models = ModelsTable()  # Singleton
```

#### Pydantic Integration
```python
class ModelModel(BaseModel):
    id: str
    name: str
    meta: ModelMeta

    model_config = ConfigDict(from_attributes=True)
```

#### Access Control Pattern
```json
{
  "read": {
    "group_ids": ["group1"],
    "user_ids": ["user1"]
  },
  "write": {
    "group_ids": ["group2"],
    "user_ids": ["user2"]
  }
}
```

#### Valves System (Functions/Tools)
- Global valves: stored in function/tool record
- Per-user valves: stored in `user.settings.functions.valves[function_id]`

### 6. Planned YouLab Extensions

Based on the research, here's how the extensions map to existing patterns:

#### Module Metadata
**Option A**: Extend Model's `meta` JSON field
```python
# In meta JSONField:
{
  "profile_image_url": "...",
  "description": "...",
  "youlab": {
    "course_id": "college-essay",
    "letta_agent_id": "letta-agent-uuid",
    "module_config": {...},
    "is_module": true,
    "active": true
  }
}
```

**Option B**: New `module` table with foreign key to `model`
```python
class Module(Base):
    __tablename__ = "module"
    id = Column(Text, primary_key=True)
    model_id = Column(Text, ForeignKey("model.id"))
    course_id = Column(Text)
    letta_agent_id = Column(Text)
    config = Column(JSON)
    is_active = Column(Boolean)
    created_at = Column(BigInteger)
    updated_at = Column(BigInteger)
```

#### Pending Diffs Table
```python
class PendingDiff(Base):
    __tablename__ = "pending_diff"
    id = Column(Text, primary_key=True)
    user_id = Column(Text, ForeignKey("user.id"))
    block_name = Column(Text)           # e.g., "operating_manual"
    old_value = Column(Text)
    new_value = Column(Text)
    agent_id = Column(Text)             # Background agent that proposed
    status = Column(Text)               # pending/approved/rejected
    thread_id = Column(Text)            # OpenWebUI chat thread
    reasoning = Column(Text)            # Agent's reasoning
    created_at = Column(BigInteger)
    updated_at = Column(BigInteger)
```

#### Memory Block Metadata
**Option A**: Extend User's `settings` JSON
```python
# In user.settings:
{
  "memory_blocks": {
    "operating_manual": {
      "last_modified": 1736647999,
      "last_modified_by": "insight-synthesizer",
      "version": 5
    }
  }
}
```

**Option B**: New `memory_block` table
```python
class MemoryBlock(Base):
    __tablename__ = "memory_block"
    id = Column(Text, primary_key=True)
    user_id = Column(Text, ForeignKey("user.id"))
    name = Column(Text)                 # e.g., "operating_manual"
    content = Column(Text)              # Current value
    version = Column(Integer)
    last_modified_by = Column(Text)     # agent_id or "manual"
    created_at = Column(BigInteger)
    updated_at = Column(BigInteger)
```

## Code References

- Database base class: `OpenWebUI/open-webui/backend/open_webui/internal/db.py:147-148`
- Migration execution: `OpenWebUI/open-webui/backend/open_webui/internal/db.py:78` (Peewee), `config.py:70` (Alembic)
- User model: `OpenWebUI/open-webui/backend/open_webui/models/users.py:45-76`
- Chat model: `OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-65`
- Model model: `OpenWebUI/open-webui/backend/open_webui/models/models.py:53-102`
- Alembic config: `OpenWebUI/open-webui/backend/open_webui/alembic.ini`
- Migration utility: `OpenWebUI/open-webui/backend/open_webui/migrations/util.py`

## Architecture Documentation

### Design Patterns Used
1. **Table Accessor Pattern**: Singleton classes wrapping CRUD operations
2. **Pydantic Validation**: `ConfigDict(from_attributes=True)` for ORM integration
3. **JSON Blob Storage**: Flexible metadata via JSON/JSONField columns
4. **Access Control**: Consistent ACL structure across models
5. **BigInteger Timestamps**: Unix timestamps (seconds) for created_at/updated_at
6. **Dual Migration System**: Peewee (legacy) + Alembic (current)

### Timestamp Convention
- All timestamps are BigInteger (Unix epoch seconds)
- Channels use nanosecond precision (`time.time_ns()`)

### JSON Query Dialect Differences
- **SQLite**: JSON1 extension (`json_each`, `->`, `->>`)
- **PostgreSQL**: JSONB type and `json_array_elements`

## Open Questions

1. **Extension Strategy**: Should we use JSON blob extensions (easier, no migrations) or new tables (cleaner, requires migrations)?
2. **Git Backend Integration**: How will memory block versioning integrate with the VPS git backend described in the spec?
3. **Upgrade Compatibility**: How do we handle OpenWebUI upgrades that might conflict with our schema changes?
4. **Migration Ownership**: Should YouLab migrations be in a separate directory to avoid merge conflicts with upstream OpenWebUI?

## Related Research

- Product Spec: `thoughts/shared/plans/2025-01-12-youlab-product-spec.md`
- Linear Ticket: ARI-78
