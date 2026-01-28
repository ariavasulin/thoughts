# Ralph Background Agents Implementation Plan

## Overview

Implement an in-process background task system for Ralph that runs Agno agents on schedules (cron-style and idle-based triggers), processing explicit user lists in batches with configurable concurrency. The architecture leaves the door open for future migration to a task queue (ARQ/Redis) without significant refactoring.

## Current State Analysis

### What Exists
- **Per-request ephemeral agents**: Agno agents created fresh for each chat request in `server.py:200-211`
- **Memory block system**: Dolt-backed storage with proposal/approval workflow in `dolt.py`
- **Honcho integration**: Message persistence in `honcho.py` (can be used for conversation history queries)
- **Tools infrastructure**: `FileTools`, `ShellTools` scoped to user workspaces; `strip_agno_fields()` helper

### What's Missing
- No mechanism to run agents outside HTTP request context
- No scheduling infrastructure (cron or idle-based)
- No tracking of user activity timestamps for idle triggers
- No background task execution or history logging

### Key Discoveries
- Agent creation pattern in `server.py:200-211` can be extracted and reused
- Memory context building already exists in `memory.py:build_memory_context()`
- Dolt client is a singleton with connection pooling (`dolt.py:551-561`)
- FastAPI lifespan (`server.py:85-104`) is the right place to start/stop the scheduler

## Desired End State

A fully functional background task system where:

1. **Tasks are registered programmatically** with a name, system prompt, tools, memory block labels, trigger config, and explicit user list
2. **Cron triggers** fire at specified schedules (e.g., "0 3 * * *" for 3 AM daily)
3. **Idle triggers** fire N minutes after a user's last message
4. **Tasks process users in batches** with configurable concurrency (e.g., 5 users at a time)
5. **Background agents are full Agno agents** capable of multi-turn reasoning
6. **Memory block edits go through the proposal workflow** (same as chat agents)
7. **Execution history is persisted** for debugging and monitoring

### Verification
- Register a test task with cron trigger, verify it fires at the scheduled time
- Register a test task with idle trigger, send a message, verify task fires after idle threshold
- Verify task processes multiple users in parallel up to batch size
- Verify background agent can make multiple tool calls and propose memory block edits
- Verify execution history is queryable via API

## What We're NOT Doing

- TOML configuration for task definitions (future)
- External task queue (Redis/ARQ) - architecture supports it, not implemented now
- Creating the `edit_memory_block` tool (assumed to exist or out of scope)
- UI for task management
- User filtering logic (explicit user lists only)
- Retry/dead-letter handling (basic error logging only)

## Implementation Approach

We'll build the system in layers:
1. **Data layer**: Schemas and Dolt tables for tasks and runs
2. **Registry layer**: In-memory task registration with persistence
3. **Scheduler layer**: Async scheduler running in FastAPI lifespan
4. **Executor layer**: Background agent creation and execution
5. **API layer**: HTTP endpoints for management and manual triggers

The architecture separates "what to run" (task definitions) from "how to run it" (scheduler/executor), making future migration to ARQ straightforward.

---

## Phase 1: Core Data Models & Storage

### Overview
Define the data structures for background tasks and execution history, plus Dolt schema for persistence.

### Changes Required

#### 1. Background Task Models
**File**: `src/ralph/background/models.py` (new file)

```python
"""Background task data models."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Any


class TriggerType(str, Enum):
    """Trigger types for background tasks."""

    CRON = "cron"
    IDLE = "idle"


@dataclass
class CronTrigger:
    """Cron-style trigger configuration."""

    schedule: str  # Cron expression, e.g., "0 3 * * *" (3 AM daily)


@dataclass
class IdleTrigger:
    """Idle-based trigger configuration."""

    idle_minutes: int  # Minutes after last message to trigger
    cooldown_minutes: int = 60  # Minimum minutes between runs per user


@dataclass
class BackgroundTask:
    """Definition of a background task."""

    name: str
    system_prompt: str
    tools: list[str]  # Tool names to include (e.g., ["query_honcho", "edit_memory_block"])
    memory_blocks: list[str]  # Block labels to include in context
    trigger: CronTrigger | IdleTrigger
    user_ids: list[str]  # Explicit list of users to process
    batch_size: int = 5  # Number of users to process concurrently
    max_turns: int = 10  # Maximum agent turns per user
    enabled: bool = True


class RunStatus(str, Enum):
    """Status of a task run."""

    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    PARTIAL = "partial"  # Some users succeeded, some failed


@dataclass
class UserRunResult:
    """Result of running a task for a single user."""

    user_id: str
    status: RunStatus
    started_at: datetime
    completed_at: datetime | None = None
    turns_used: int = 0
    error: str | None = None
    proposals_created: int = 0


@dataclass
class TaskRun:
    """Record of a task execution."""

    id: str  # UUID
    task_name: str
    trigger_type: TriggerType
    status: RunStatus
    started_at: datetime
    completed_at: datetime | None = None
    user_results: list[UserRunResult] = field(default_factory=list)
    error: str | None = None  # Top-level error (e.g., task not found)
```

#### 2. User Activity Tracking Model
**File**: `src/ralph/background/models.py` (append)

```python
@dataclass
class UserActivity:
    """Tracks user activity for idle triggers."""

    user_id: str
    last_message_at: datetime
    last_task_run_at: dict[str, datetime] = field(default_factory=dict)  # task_name -> last_run
```

#### 3. Dolt Schema for Background Tasks
**File**: `migrations/003_background_tasks.sql` (new file)

```sql
-- Background task definitions
CREATE TABLE IF NOT EXISTS background_tasks (
    name VARCHAR(255) PRIMARY KEY,
    system_prompt TEXT NOT NULL,
    tools JSON NOT NULL,  -- ["query_honcho", "edit_memory_block"]
    memory_blocks JSON NOT NULL,  -- ["student", "progress"]
    trigger_type ENUM('cron', 'idle') NOT NULL,
    trigger_config JSON NOT NULL,  -- {"schedule": "0 3 * * *"} or {"idle_minutes": 30, "cooldown_minutes": 60}
    user_ids JSON NOT NULL,  -- ["user-1", "user-2"]
    batch_size INT DEFAULT 5,
    max_turns INT DEFAULT 10,
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Task execution history
CREATE TABLE IF NOT EXISTS task_runs (
    id VARCHAR(36) PRIMARY KEY,  -- UUID
    task_name VARCHAR(255) NOT NULL,
    trigger_type ENUM('cron', 'idle') NOT NULL,
    status ENUM('pending', 'running', 'success', 'failed', 'partial') NOT NULL,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP NULL,
    user_results JSON NULL,  -- Array of UserRunResult
    error TEXT NULL,
    FOREIGN KEY (task_name) REFERENCES background_tasks(name) ON DELETE CASCADE,
    INDEX idx_task_runs_task_name (task_name),
    INDEX idx_task_runs_started_at (started_at)
);

-- User activity tracking for idle triggers
CREATE TABLE IF NOT EXISTS user_activity (
    user_id VARCHAR(255) PRIMARY KEY,
    last_message_at TIMESTAMP NOT NULL,
    last_task_runs JSON DEFAULT '{}',  -- {"task_name": "2024-01-28T12:00:00Z"}
    INDEX idx_user_activity_last_message (last_message_at)
);
```

#### 4. Package Init
**File**: `src/ralph/background/__init__.py` (new file)

```python
"""Background task system for Ralph."""

from ralph.background.models import (
    BackgroundTask,
    CronTrigger,
    IdleTrigger,
    RunStatus,
    TaskRun,
    TriggerType,
    UserActivity,
    UserRunResult,
)

__all__ = [
    "BackgroundTask",
    "CronTrigger",
    "IdleTrigger",
    "RunStatus",
    "TaskRun",
    "TriggerType",
    "UserActivity",
    "UserRunResult",
]
```

### Success Criteria

#### Automated Verification
- [x] Models import without error: `python -c "from ralph.background import *"`
- [x] Type checking passes: `uv run basedpyright src/ralph/background/`
- [x] Linting passes: `uv run ruff check src/ralph/background/`
- [x] Migration SQL is valid syntax

#### Manual Verification
- [ ] Migration applies cleanly to Dolt: `docker exec youlab-dolt-1 dolt sql < migrations/003_background_tasks.sql`

---

## Phase 2: Dolt Repository Layer

### Overview
Add CRUD operations to `DoltClient` for background tasks, task runs, and user activity tracking.

### Changes Required

#### 1. Background Task Repository Methods
**File**: `src/ralph/dolt.py` (append to DoltClient class)

```python
    # -------------------------------------------------------------------------
    # Background Task Operations
    # -------------------------------------------------------------------------

    async def create_task(self, task: BackgroundTask) -> None:
        """Create or update a background task definition."""
        trigger_type = "cron" if isinstance(task.trigger, CronTrigger) else "idle"
        trigger_config = (
            {"schedule": task.trigger.schedule}
            if isinstance(task.trigger, CronTrigger)
            else {"idle_minutes": task.trigger.idle_minutes, "cooldown_minutes": task.trigger.cooldown_minutes}
        )

        async with self.session() as session:
            await session.execute(
                text("""
                    INSERT INTO background_tasks
                        (name, system_prompt, tools, memory_blocks, trigger_type,
                         trigger_config, user_ids, batch_size, max_turns, enabled)
                    VALUES
                        (:name, :system_prompt, :tools, :memory_blocks, :trigger_type,
                         :trigger_config, :user_ids, :batch_size, :max_turns, :enabled)
                    ON DUPLICATE KEY UPDATE
                        system_prompt = :system_prompt,
                        tools = :tools,
                        memory_blocks = :memory_blocks,
                        trigger_type = :trigger_type,
                        trigger_config = :trigger_config,
                        user_ids = :user_ids,
                        batch_size = :batch_size,
                        max_turns = :max_turns,
                        enabled = :enabled
                """),
                {
                    "name": task.name,
                    "system_prompt": task.system_prompt,
                    "tools": json.dumps(task.tools),
                    "memory_blocks": json.dumps(task.memory_blocks),
                    "trigger_type": trigger_type,
                    "trigger_config": json.dumps(trigger_config),
                    "user_ids": json.dumps(task.user_ids),
                    "batch_size": task.batch_size,
                    "max_turns": task.max_turns,
                    "enabled": task.enabled,
                },
            )
            await session.commit()

    async def get_task(self, name: str) -> BackgroundTask | None:
        """Get a background task by name."""
        async with self.session() as session:
            result = await session.execute(
                text("SELECT * FROM background_tasks WHERE name = :name"),
                {"name": name},
            )
            row = result.fetchone()
            if not row:
                return None
            return self._row_to_task(row)

    async def list_tasks(self, enabled_only: bool = False) -> list[BackgroundTask]:
        """List all background tasks."""
        async with self.session() as session:
            query = "SELECT * FROM background_tasks"
            if enabled_only:
                query += " WHERE enabled = TRUE"
            result = await session.execute(text(query))
            return [self._row_to_task(row) for row in result.fetchall()]

    async def delete_task(self, name: str) -> bool:
        """Delete a background task. Returns True if deleted."""
        async with self.session() as session:
            result = await session.execute(
                text("DELETE FROM background_tasks WHERE name = :name"),
                {"name": name},
            )
            await session.commit()
            return result.rowcount > 0

    def _row_to_task(self, row: Any) -> BackgroundTask:
        """Convert a database row to a BackgroundTask."""
        trigger_config = json.loads(row.trigger_config)
        if row.trigger_type == "cron":
            trigger = CronTrigger(schedule=trigger_config["schedule"])
        else:
            trigger = IdleTrigger(
                idle_minutes=trigger_config["idle_minutes"],
                cooldown_minutes=trigger_config.get("cooldown_minutes", 60),
            )

        return BackgroundTask(
            name=row.name,
            system_prompt=row.system_prompt,
            tools=json.loads(row.tools),
            memory_blocks=json.loads(row.memory_blocks),
            trigger=trigger,
            user_ids=json.loads(row.user_ids),
            batch_size=row.batch_size,
            max_turns=row.max_turns,
            enabled=row.enabled,
        )
```

#### 2. Task Run Repository Methods
**File**: `src/ralph/dolt.py` (append to DoltClient class)

```python
    async def create_task_run(self, run: TaskRun) -> None:
        """Create a task run record."""
        async with self.session() as session:
            await session.execute(
                text("""
                    INSERT INTO task_runs
                        (id, task_name, trigger_type, status, started_at, completed_at, user_results, error)
                    VALUES
                        (:id, :task_name, :trigger_type, :status, :started_at, :completed_at, :user_results, :error)
                """),
                {
                    "id": run.id,
                    "task_name": run.task_name,
                    "trigger_type": run.trigger_type.value,
                    "status": run.status.value,
                    "started_at": run.started_at,
                    "completed_at": run.completed_at,
                    "user_results": json.dumps([self._user_result_to_dict(r) for r in run.user_results]),
                    "error": run.error,
                },
            )
            await session.commit()

    async def update_task_run(self, run: TaskRun) -> None:
        """Update a task run record."""
        async with self.session() as session:
            await session.execute(
                text("""
                    UPDATE task_runs SET
                        status = :status,
                        completed_at = :completed_at,
                        user_results = :user_results,
                        error = :error
                    WHERE id = :id
                """),
                {
                    "id": run.id,
                    "status": run.status.value,
                    "completed_at": run.completed_at,
                    "user_results": json.dumps([self._user_result_to_dict(r) for r in run.user_results]),
                    "error": run.error,
                },
            )
            await session.commit()

    async def get_task_run(self, run_id: str) -> TaskRun | None:
        """Get a task run by ID."""
        async with self.session() as session:
            result = await session.execute(
                text("SELECT * FROM task_runs WHERE id = :id"),
                {"id": run_id},
            )
            row = result.fetchone()
            if not row:
                return None
            return self._row_to_task_run(row)

    async def list_task_runs(
        self, task_name: str | None = None, limit: int = 50
    ) -> list[TaskRun]:
        """List task runs, optionally filtered by task name."""
        async with self.session() as session:
            if task_name:
                result = await session.execute(
                    text("""
                        SELECT * FROM task_runs
                        WHERE task_name = :task_name
                        ORDER BY started_at DESC
                        LIMIT :limit
                    """),
                    {"task_name": task_name, "limit": limit},
                )
            else:
                result = await session.execute(
                    text("SELECT * FROM task_runs ORDER BY started_at DESC LIMIT :limit"),
                    {"limit": limit},
                )
            return [self._row_to_task_run(row) for row in result.fetchall()]

    def _user_result_to_dict(self, result: UserRunResult) -> dict[str, Any]:
        """Convert UserRunResult to dict for JSON storage."""
        return {
            "user_id": result.user_id,
            "status": result.status.value,
            "started_at": result.started_at.isoformat(),
            "completed_at": result.completed_at.isoformat() if result.completed_at else None,
            "turns_used": result.turns_used,
            "error": result.error,
            "proposals_created": result.proposals_created,
        }

    def _row_to_task_run(self, row: Any) -> TaskRun:
        """Convert a database row to a TaskRun."""
        user_results_data = json.loads(row.user_results) if row.user_results else []
        user_results = [
            UserRunResult(
                user_id=r["user_id"],
                status=RunStatus(r["status"]),
                started_at=datetime.fromisoformat(r["started_at"]),
                completed_at=datetime.fromisoformat(r["completed_at"]) if r.get("completed_at") else None,
                turns_used=r.get("turns_used", 0),
                error=r.get("error"),
                proposals_created=r.get("proposals_created", 0),
            )
            for r in user_results_data
        ]

        return TaskRun(
            id=row.id,
            task_name=row.task_name,
            trigger_type=TriggerType(row.trigger_type),
            status=RunStatus(row.status),
            started_at=row.started_at,
            completed_at=row.completed_at,
            user_results=user_results,
            error=row.error,
        )
```

#### 3. User Activity Repository Methods
**File**: `src/ralph/dolt.py` (append to DoltClient class)

```python
    async def update_user_activity(self, user_id: str, message_time: datetime) -> None:
        """Update user's last message time."""
        async with self.session() as session:
            await session.execute(
                text("""
                    INSERT INTO user_activity (user_id, last_message_at, last_task_runs)
                    VALUES (:user_id, :last_message_at, '{}')
                    ON DUPLICATE KEY UPDATE last_message_at = :last_message_at
                """),
                {"user_id": user_id, "last_message_at": message_time},
            )
            await session.commit()

    async def record_task_run_for_user(
        self, user_id: str, task_name: str, run_time: datetime
    ) -> None:
        """Record when a task was last run for a user."""
        async with self.session() as session:
            # Get current task runs
            result = await session.execute(
                text("SELECT last_task_runs FROM user_activity WHERE user_id = :user_id"),
                {"user_id": user_id},
            )
            row = result.fetchone()
            task_runs = json.loads(row.last_task_runs) if row else {}
            task_runs[task_name] = run_time.isoformat()

            await session.execute(
                text("""
                    INSERT INTO user_activity (user_id, last_message_at, last_task_runs)
                    VALUES (:user_id, NOW(), :last_task_runs)
                    ON DUPLICATE KEY UPDATE last_task_runs = :last_task_runs
                """),
                {"user_id": user_id, "last_task_runs": json.dumps(task_runs)},
            )
            await session.commit()

    async def get_users_idle_for(
        self, minutes: int, task_name: str, cooldown_minutes: int
    ) -> list[str]:
        """Get users who have been idle for at least `minutes` and haven't had task run within cooldown."""
        cutoff = datetime.utcnow() - timedelta(minutes=minutes)
        cooldown_cutoff = datetime.utcnow() - timedelta(minutes=cooldown_minutes)

        async with self.session() as session:
            result = await session.execute(
                text("""
                    SELECT user_id, last_task_runs
                    FROM user_activity
                    WHERE last_message_at <= :cutoff
                """),
                {"cutoff": cutoff},
            )

            eligible_users = []
            for row in result.fetchall():
                task_runs = json.loads(row.last_task_runs) if row.last_task_runs else {}
                last_run = task_runs.get(task_name)
                if last_run:
                    last_run_dt = datetime.fromisoformat(last_run)
                    if last_run_dt > cooldown_cutoff:
                        continue  # Still in cooldown
                eligible_users.append(row.user_id)

            return eligible_users

    async def get_user_activity(self, user_id: str) -> UserActivity | None:
        """Get activity record for a user."""
        async with self.session() as session:
            result = await session.execute(
                text("SELECT * FROM user_activity WHERE user_id = :user_id"),
                {"user_id": user_id},
            )
            row = result.fetchone()
            if not row:
                return None
            return UserActivity(
                user_id=row.user_id,
                last_message_at=row.last_message_at,
                last_task_run_at={
                    k: datetime.fromisoformat(v)
                    for k, v in json.loads(row.last_task_runs or "{}").items()
                },
            )
```

#### 4. Import Updates
**File**: `src/ralph/dolt.py` (update imports at top)

Add to imports:
```python
from datetime import datetime, timedelta

from ralph.background.models import (
    BackgroundTask,
    CronTrigger,
    IdleTrigger,
    RunStatus,
    TaskRun,
    TriggerType,
    UserActivity,
    UserRunResult,
)
```

### Success Criteria

#### Automated Verification
- [x] Type checking passes: `uv run basedpyright src/ralph/dolt.py`
- [x] Linting passes: `uv run ruff check src/ralph/dolt.py`
- [x] Imports work: `python -c "from ralph.dolt import DoltClient"`

#### Manual Verification
- [ ] CRUD operations work against Dolt database (test with curl or Python script)

---

## Phase 3: Task Registry

### Overview
Create an in-memory task registry with synchronization to Dolt. Tasks can be registered programmatically and are loaded from database on startup.

### Changes Required

#### 1. Task Registry
**File**: `src/ralph/background/registry.py` (new file)

```python
"""Background task registry."""

from __future__ import annotations

import structlog

from ralph.background.models import BackgroundTask, CronTrigger, IdleTrigger
from ralph.dolt import DoltClient

log = structlog.get_logger()


class TaskRegistry:
    """In-memory registry of background tasks with Dolt persistence."""

    def __init__(self) -> None:
        self._tasks: dict[str, BackgroundTask] = {}
        self._dolt: DoltClient | None = None

    async def initialize(self, dolt: DoltClient) -> None:
        """Initialize registry and load tasks from database."""
        self._dolt = dolt
        await self._load_from_database()

    async def _load_from_database(self) -> None:
        """Load all tasks from Dolt."""
        if not self._dolt:
            return
        tasks = await self._dolt.list_tasks()
        for task in tasks:
            self._tasks[task.name] = task
            log.info("task_loaded", name=task.name, enabled=task.enabled)

    async def register(self, task: BackgroundTask, persist: bool = True) -> None:
        """
        Register a background task.

        Args:
            task: The task definition
            persist: If True, save to Dolt database
        """
        self._tasks[task.name] = task
        log.info(
            "task_registered",
            name=task.name,
            trigger_type="cron" if isinstance(task.trigger, CronTrigger) else "idle",
            user_count=len(task.user_ids),
            enabled=task.enabled,
        )

        if persist and self._dolt:
            await self._dolt.create_task(task)
            log.info("task_persisted", name=task.name)

    async def unregister(self, name: str, persist: bool = True) -> bool:
        """
        Unregister a background task.

        Returns True if task existed and was removed.
        """
        if name not in self._tasks:
            return False

        del self._tasks[name]
        log.info("task_unregistered", name=name)

        if persist and self._dolt:
            await self._dolt.delete_task(name)

        return True

    def get(self, name: str) -> BackgroundTask | None:
        """Get a task by name."""
        return self._tasks.get(name)

    def list_all(self) -> list[BackgroundTask]:
        """List all registered tasks."""
        return list(self._tasks.values())

    def list_enabled(self) -> list[BackgroundTask]:
        """List only enabled tasks."""
        return [t for t in self._tasks.values() if t.enabled]

    def list_cron_tasks(self) -> list[BackgroundTask]:
        """List enabled tasks with cron triggers."""
        return [
            t for t in self._tasks.values()
            if t.enabled and isinstance(t.trigger, CronTrigger)
        ]

    def list_idle_tasks(self) -> list[BackgroundTask]:
        """List enabled tasks with idle triggers."""
        return [
            t for t in self._tasks.values()
            if t.enabled and isinstance(t.trigger, IdleTrigger)
        ]

    async def set_enabled(self, name: str, enabled: bool) -> bool:
        """Enable or disable a task. Returns False if task not found."""
        task = self._tasks.get(name)
        if not task:
            return False

        # Create new task with updated enabled status
        updated_task = BackgroundTask(
            name=task.name,
            system_prompt=task.system_prompt,
            tools=task.tools,
            memory_blocks=task.memory_blocks,
            trigger=task.trigger,
            user_ids=task.user_ids,
            batch_size=task.batch_size,
            max_turns=task.max_turns,
            enabled=enabled,
        )
        await self.register(updated_task, persist=True)
        return True


# Module-level singleton
_registry: TaskRegistry | None = None


def get_registry() -> TaskRegistry:
    """Get the task registry singleton."""
    global _registry
    if _registry is None:
        _registry = TaskRegistry()
    return _registry
```

#### 2. Update Package Init
**File**: `src/ralph/background/__init__.py` (update)

```python
"""Background task system for Ralph."""

from ralph.background.models import (
    BackgroundTask,
    CronTrigger,
    IdleTrigger,
    RunStatus,
    TaskRun,
    TriggerType,
    UserActivity,
    UserRunResult,
)
from ralph.background.registry import TaskRegistry, get_registry

__all__ = [
    "BackgroundTask",
    "CronTrigger",
    "IdleTrigger",
    "RunStatus",
    "TaskRegistry",
    "TaskRun",
    "TriggerType",
    "UserActivity",
    "UserRunResult",
    "get_registry",
]
```

### Success Criteria

#### Automated Verification
- [x] Type checking passes: `uv run basedpyright src/ralph/background/`
- [x] Linting passes: `uv run ruff check src/ralph/background/`
- [x] Imports work: `python -c "from ralph.background import get_registry, BackgroundTask"`

#### Manual Verification
- [ ] Registry loads tasks from database on initialization
- [ ] Tasks can be registered and unregistered programmatically

---

## Phase 4: Background Agent Executor

### Overview
Create the executor that runs Agno agents for background tasks, processing users in batches with configurable concurrency.

### Changes Required

#### 1. Tool Factory
**File**: `src/ralph/background/tools.py` (new file)

```python
"""Tool factory for background agents."""

from __future__ import annotations

from pathlib import Path
from typing import TYPE_CHECKING

from agno.tools.file import FileTools
from agno.tools.shell import ShellTools

if TYPE_CHECKING:
    from agno.tools.toolkit import Toolkit

from ralph.config import get_settings


def strip_agno_fields(toolkit: Toolkit) -> Toolkit:
    """Strip Agno-specific fields that some models don't accept."""
    for func in toolkit.functions.values():
        func.requires_confirmation = None  # type: ignore[assignment]
        func.external_execution = None  # type: ignore[assignment]
    return toolkit


def get_workspace_path(user_id: str) -> Path:
    """Get workspace directory for a user."""
    settings = get_settings()
    if settings.agent_workspace:
        return Path(settings.agent_workspace)
    workspace = Path(settings.user_data_dir) / user_id / "workspace"
    workspace.mkdir(parents=True, exist_ok=True)
    return workspace


def create_tools_for_task(
    tool_names: list[str],
    user_id: str,
) -> list[Toolkit]:
    """
    Create tool instances for a background task.

    Args:
        tool_names: List of tool names to include
        user_id: User ID for workspace scoping

    Returns:
        List of Toolkit instances
    """
    workspace = get_workspace_path(user_id)
    tools: list[Toolkit] = []

    for name in tool_names:
        if name == "file_tools":
            tools.append(strip_agno_fields(FileTools(base_dir=workspace)))
        elif name == "shell_tools":
            tools.append(strip_agno_fields(ShellTools(base_dir=workspace)))
        # Add more tool mappings here as needed:
        # elif name == "query_honcho":
        #     tools.append(QueryHonchoTool(...))
        # elif name == "edit_memory_block":
        #     tools.append(EditMemoryBlockTool(...))

    return tools
```

#### 2. Background Agent Executor
**File**: `src/ralph/background/executor.py` (new file)

```python
"""Background agent executor."""

from __future__ import annotations

import asyncio
import uuid
from datetime import datetime
from typing import TYPE_CHECKING

import structlog
from agno.agent import Agent
from agno.models.openrouter import OpenRouter

from ralph.background.models import (
    BackgroundTask,
    RunStatus,
    TaskRun,
    TriggerType,
    UserRunResult,
)
from ralph.background.tools import create_tools_for_task
from ralph.config import get_settings
from ralph.dolt import DoltClient
from ralph.memory import build_memory_context

if TYPE_CHECKING:
    pass

log = structlog.get_logger()


class BackgroundExecutor:
    """Executes background tasks for users."""

    def __init__(self, dolt: DoltClient) -> None:
        self._dolt = dolt
        self._settings = get_settings()

    async def execute_task(
        self,
        task: BackgroundTask,
        trigger_type: TriggerType,
        user_ids: list[str] | None = None,
    ) -> TaskRun:
        """
        Execute a background task for specified users.

        Args:
            task: The task definition
            trigger_type: What triggered this run
            user_ids: Override user list (defaults to task.user_ids)

        Returns:
            TaskRun with results for all users
        """
        run_id = str(uuid.uuid4())
        users_to_process = user_ids or task.user_ids

        run = TaskRun(
            id=run_id,
            task_name=task.name,
            trigger_type=trigger_type,
            status=RunStatus.RUNNING,
            started_at=datetime.utcnow(),
            user_results=[],
        )

        # Persist initial run record
        await self._dolt.create_task_run(run)

        log.info(
            "task_run_started",
            run_id=run_id,
            task_name=task.name,
            user_count=len(users_to_process),
            batch_size=task.batch_size,
        )

        try:
            # Process users in batches
            for i in range(0, len(users_to_process), task.batch_size):
                batch = users_to_process[i : i + task.batch_size]
                batch_results = await self._process_batch(task, batch)
                run.user_results.extend(batch_results)

                # Update run record after each batch
                await self._dolt.update_task_run(run)

            # Determine final status
            statuses = {r.status for r in run.user_results}
            if statuses == {RunStatus.SUCCESS}:
                run.status = RunStatus.SUCCESS
            elif statuses == {RunStatus.FAILED}:
                run.status = RunStatus.FAILED
            else:
                run.status = RunStatus.PARTIAL

        except Exception as e:
            log.exception("task_run_failed", run_id=run_id, error=str(e))
            run.status = RunStatus.FAILED
            run.error = str(e)

        run.completed_at = datetime.utcnow()
        await self._dolt.update_task_run(run)

        log.info(
            "task_run_completed",
            run_id=run_id,
            task_name=task.name,
            status=run.status.value,
            duration_seconds=(run.completed_at - run.started_at).total_seconds(),
        )

        return run

    async def _process_batch(
        self,
        task: BackgroundTask,
        user_ids: list[str],
    ) -> list[UserRunResult]:
        """Process a batch of users concurrently."""
        tasks = [self._run_for_user(task, user_id) for user_id in user_ids]
        return await asyncio.gather(*tasks)

    async def _run_for_user(
        self,
        task: BackgroundTask,
        user_id: str,
    ) -> UserRunResult:
        """Run a background task for a single user."""
        started_at = datetime.utcnow()
        log.info("user_run_started", task_name=task.name, user_id=user_id)

        try:
            # Build memory context for this user
            memory_context = ""
            if task.memory_blocks:
                memory_context = await build_memory_context(
                    self._dolt, user_id, labels=task.memory_blocks
                )

            # Build instructions
            instructions = task.system_prompt
            if memory_context:
                instructions += f"\n\n---\n\n# Student Context\n\n{memory_context}"

            # Create tools for this user
            tools = create_tools_for_task(task.tools, user_id)

            # Create agent
            agent = Agent(
                model=OpenRouter(
                    id=self._settings.openrouter_model,
                    api_key=self._settings.openrouter_api_key,
                ),
                tools=tools,
                instructions=instructions,
                markdown=True,
            )

            # Run agent (non-streaming, let it iterate)
            # The agent will make tool calls and reason through the task
            turns_used = 0
            async for _chunk in agent.arun(
                "Execute your background task now. Review the student context and take appropriate action.",
                stream=True,
            ):
                turns_used += 1
                if turns_used >= task.max_turns:
                    log.warning(
                        "user_run_max_turns",
                        task_name=task.name,
                        user_id=user_id,
                        max_turns=task.max_turns,
                    )
                    break

            # Record that this task ran for this user
            await self._dolt.record_task_run_for_user(user_id, task.name, datetime.utcnow())

            log.info(
                "user_run_completed",
                task_name=task.name,
                user_id=user_id,
                turns_used=turns_used,
            )

            return UserRunResult(
                user_id=user_id,
                status=RunStatus.SUCCESS,
                started_at=started_at,
                completed_at=datetime.utcnow(),
                turns_used=turns_used,
                proposals_created=0,  # TODO: Track this when edit_memory_block tool exists
            )

        except Exception as e:
            log.exception("user_run_failed", task_name=task.name, user_id=user_id, error=str(e))
            return UserRunResult(
                user_id=user_id,
                status=RunStatus.FAILED,
                started_at=started_at,
                completed_at=datetime.utcnow(),
                error=str(e),
            )
```

#### 3. Update memory.py to support label filtering
**File**: `src/ralph/memory.py` (update)

```python
"""Memory context building for agents."""

from __future__ import annotations

from ralph.dolt import DoltClient


async def build_memory_context(
    dolt: DoltClient,
    user_id: str,
    labels: list[str] | None = None,
) -> str:
    """
    Build memory context string from user's memory blocks.

    Args:
        dolt: Dolt client
        user_id: User ID
        labels: Optional list of block labels to include (None = all)

    Returns:
        Formatted markdown string with memory blocks
    """
    blocks = await dolt.list_blocks(user_id)

    # Filter by labels if specified
    if labels:
        blocks = [b for b in blocks if b.label in labels]

    if not blocks:
        return ""

    sections = ["## Student Memory\n"]
    for block in blocks:
        title = block.title or block.label.replace("_", " ").title()
        body = block.body or "(empty)"
        sections.append(f"### {title}\n\n{body}")

    return "\n\n".join(sections)


async def get_block_for_agent(
    dolt: DoltClient,
    user_id: str,
    label: str,
) -> str | None:
    """Get a single memory block's body for an agent."""
    block = await dolt.get_block(user_id, label)
    if not block:
        return None
    return block.body
```

#### 4. Update Package Init
**File**: `src/ralph/background/__init__.py` (update)

```python
"""Background task system for Ralph."""

from ralph.background.executor import BackgroundExecutor
from ralph.background.models import (
    BackgroundTask,
    CronTrigger,
    IdleTrigger,
    RunStatus,
    TaskRun,
    TriggerType,
    UserActivity,
    UserRunResult,
)
from ralph.background.registry import TaskRegistry, get_registry

__all__ = [
    "BackgroundExecutor",
    "BackgroundTask",
    "CronTrigger",
    "IdleTrigger",
    "RunStatus",
    "TaskRegistry",
    "TaskRun",
    "TriggerType",
    "UserActivity",
    "UserRunResult",
    "get_registry",
]
```

### Success Criteria

#### Automated Verification
- [x] Type checking passes: `uv run basedpyright src/ralph/background/`
- [x] Linting passes: `uv run ruff check src/ralph/background/`
- [x] Imports work: `python -c "from ralph.background import BackgroundExecutor"`

#### Manual Verification
- [ ] Executor can run a task for a single user
- [ ] Batch processing works with multiple users
- [ ] Task runs are persisted to Dolt

---

## Phase 5: Scheduler

### Overview
Create the async scheduler that monitors triggers and dispatches task executions. Runs as part of FastAPI lifespan.

### Changes Required

#### 1. Scheduler Core
**File**: `src/ralph/background/scheduler.py` (new file)

```python
"""Background task scheduler."""

from __future__ import annotations

import asyncio
from datetime import datetime
from typing import TYPE_CHECKING

import structlog
from croniter import croniter

from ralph.background.executor import BackgroundExecutor
from ralph.background.models import CronTrigger, IdleTrigger, TriggerType
from ralph.background.registry import TaskRegistry

if TYPE_CHECKING:
    from ralph.dolt import DoltClient

log = structlog.get_logger()


class BackgroundScheduler:
    """
    Async scheduler for background tasks.

    Monitors cron and idle triggers and dispatches executions.
    Designed to run within FastAPI lifespan context.
    """

    def __init__(
        self,
        registry: TaskRegistry,
        executor: BackgroundExecutor,
        dolt: DoltClient,
        check_interval_seconds: int = 60,
    ) -> None:
        self._registry = registry
        self._executor = executor
        self._dolt = dolt
        self._check_interval = check_interval_seconds
        self._running = False
        self._task: asyncio.Task[None] | None = None
        self._last_cron_check: dict[str, datetime] = {}

    async def start(self) -> None:
        """Start the scheduler loop."""
        if self._running:
            log.warning("scheduler_already_running")
            return

        self._running = True
        self._task = asyncio.create_task(self._run_loop())
        log.info("scheduler_started", check_interval=self._check_interval)

    async def stop(self) -> None:
        """Stop the scheduler loop."""
        self._running = False
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        log.info("scheduler_stopped")

    async def _run_loop(self) -> None:
        """Main scheduler loop."""
        while self._running:
            try:
                await self._check_triggers()
            except Exception:
                log.exception("scheduler_check_failed")

            await asyncio.sleep(self._check_interval)

    async def _check_triggers(self) -> None:
        """Check all triggers and dispatch tasks as needed."""
        now = datetime.utcnow()

        # Check cron tasks
        for task in self._registry.list_cron_tasks():
            if await self._should_run_cron(task.name, task.trigger, now):
                log.info("cron_trigger_fired", task_name=task.name)
                asyncio.create_task(
                    self._executor.execute_task(task, TriggerType.CRON)
                )
                self._last_cron_check[task.name] = now

        # Check idle tasks
        for task in self._registry.list_idle_tasks():
            trigger = task.trigger
            if not isinstance(trigger, IdleTrigger):
                continue

            # Find users who are idle and not in cooldown
            # Only consider users in the task's user_ids list
            idle_users = await self._dolt.get_users_idle_for(
                minutes=trigger.idle_minutes,
                task_name=task.name,
                cooldown_minutes=trigger.cooldown_minutes,
            )

            # Filter to only users in this task's list
            eligible_users = [u for u in idle_users if u in task.user_ids]

            if eligible_users:
                log.info(
                    "idle_trigger_fired",
                    task_name=task.name,
                    user_count=len(eligible_users),
                )
                asyncio.create_task(
                    self._executor.execute_task(
                        task, TriggerType.IDLE, user_ids=eligible_users
                    )
                )

    async def _should_run_cron(
        self,
        task_name: str,
        trigger: CronTrigger,
        now: datetime,
    ) -> bool:
        """Check if a cron task should run now."""
        last_check = self._last_cron_check.get(task_name)

        if last_check is None:
            # First check - initialize but don't run immediately
            self._last_cron_check[task_name] = now
            return False

        # Check if cron would have fired between last check and now
        cron = croniter(trigger.schedule, last_check)
        next_run = cron.get_next(datetime)

        return next_run <= now

    async def run_task_now(self, task_name: str) -> str | None:
        """
        Manually trigger a task to run immediately.

        Returns the run ID if task found, None otherwise.
        """
        task = self._registry.get(task_name)
        if not task:
            log.warning("manual_trigger_task_not_found", task_name=task_name)
            return None

        log.info("manual_trigger_fired", task_name=task_name)
        run = await self._executor.execute_task(task, TriggerType.CRON)  # Use CRON for manual
        return run.id


# Module-level singleton
_scheduler: BackgroundScheduler | None = None


async def get_scheduler(
    registry: TaskRegistry,
    executor: BackgroundExecutor,
    dolt: DoltClient,
) -> BackgroundScheduler:
    """Get or create the scheduler singleton."""
    global _scheduler
    if _scheduler is None:
        _scheduler = BackgroundScheduler(registry, executor, dolt)
    return _scheduler


async def stop_scheduler() -> None:
    """Stop and clear the scheduler singleton."""
    global _scheduler
    if _scheduler:
        await _scheduler.stop()
        _scheduler = None
```

#### 2. Integrate with FastAPI Lifespan
**File**: `src/ralph/server.py` (update lifespan)

Update the imports at the top:
```python
from ralph.background import BackgroundExecutor, get_registry
from ralph.background.scheduler import get_scheduler, stop_scheduler
```

Update the lifespan function:
```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """Application lifespan handler."""
    settings = get_settings()
    log.info("ralph_server_starting", model=settings.openrouter_model)

    # Initialize Dolt connection pool
    dolt = None
    try:
        dolt = await get_dolt_client()
        log.info("dolt_client_connected")
    except Exception as e:
        log.warning("dolt_client_connection_failed", error=str(e))
        # Continue without Dolt - blocks API will fail but chat will work

    # Initialize background task system
    if dolt:
        try:
            registry = get_registry()
            await registry.initialize(dolt)

            executor = BackgroundExecutor(dolt)
            scheduler = await get_scheduler(registry, executor, dolt)
            await scheduler.start()
            log.info("background_scheduler_started")
        except Exception as e:
            log.warning("background_scheduler_failed", error=str(e))

    yield

    # Shutdown
    await stop_scheduler()
    log.info("background_scheduler_stopped")

    await close_dolt_client()
    log.info("dolt_client_disconnected")
    log.info("ralph_server_stopped")
```

#### 3. Add croniter to dependencies
**File**: `pyproject.toml` (update dependencies)

Add to the `[project]` dependencies list:
```toml
"croniter>=2.0.0",
```

#### 4. Update Package Init
**File**: `src/ralph/background/__init__.py` (final)

```python
"""Background task system for Ralph."""

from ralph.background.executor import BackgroundExecutor
from ralph.background.models import (
    BackgroundTask,
    CronTrigger,
    IdleTrigger,
    RunStatus,
    TaskRun,
    TriggerType,
    UserActivity,
    UserRunResult,
)
from ralph.background.registry import TaskRegistry, get_registry
from ralph.background.scheduler import BackgroundScheduler, get_scheduler, stop_scheduler

__all__ = [
    "BackgroundExecutor",
    "BackgroundScheduler",
    "BackgroundTask",
    "CronTrigger",
    "IdleTrigger",
    "RunStatus",
    "TaskRegistry",
    "TaskRun",
    "TriggerType",
    "UserActivity",
    "UserRunResult",
    "get_registry",
    "get_scheduler",
    "stop_scheduler",
]
```

### Success Criteria

#### Automated Verification
- [x] Type checking passes: `uv run basedpyright src/ralph/`
- [x] Linting passes: `uv run ruff check src/ralph/`
- [x] Dependencies install: `uv sync`
- [ ] Server starts: `uv run ralph-server` (check logs for scheduler startup)

#### Manual Verification
- [ ] Scheduler starts with server and loads tasks from database
- [ ] Cron tasks fire at scheduled times
- [ ] Idle tasks fire after user goes idle

---

## Phase 6: HTTP API & User Activity Tracking

### Overview
Add HTTP endpoints for task management, manual triggers, and run history. Also hook into chat endpoint to track user activity.

### Changes Required

#### 1. Background Tasks API Router
**File**: `src/ralph/api/background.py` (new file)

```python
"""HTTP API for background task management."""

from __future__ import annotations

from datetime import datetime
from typing import Any

import structlog
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

from ralph.background import (
    BackgroundTask,
    CronTrigger,
    IdleTrigger,
    RunStatus,
    TriggerType,
    get_registry,
)
from ralph.background.scheduler import get_scheduler, BackgroundScheduler
from ralph.background.executor import BackgroundExecutor
from ralph.dolt import get_dolt_client

log = structlog.get_logger()

router = APIRouter(prefix="/background", tags=["background"])


# Request/Response Models


class CronTriggerRequest(BaseModel):
    """Cron trigger configuration."""

    type: str = "cron"
    schedule: str  # Cron expression


class IdleTriggerRequest(BaseModel):
    """Idle trigger configuration."""

    type: str = "idle"
    idle_minutes: int
    cooldown_minutes: int = 60


class CreateTaskRequest(BaseModel):
    """Request to create a background task."""

    name: str
    system_prompt: str
    tools: list[str]
    memory_blocks: list[str]
    trigger: CronTriggerRequest | IdleTriggerRequest
    user_ids: list[str]
    batch_size: int = 5
    max_turns: int = 10
    enabled: bool = True


class TaskResponse(BaseModel):
    """Background task response."""

    name: str
    system_prompt: str
    tools: list[str]
    memory_blocks: list[str]
    trigger_type: str
    trigger_config: dict[str, Any]
    user_ids: list[str]
    batch_size: int
    max_turns: int
    enabled: bool


class UserRunResultResponse(BaseModel):
    """Result for a single user in a task run."""

    user_id: str
    status: str
    started_at: datetime
    completed_at: datetime | None
    turns_used: int
    error: str | None
    proposals_created: int


class TaskRunResponse(BaseModel):
    """Task run response."""

    id: str
    task_name: str
    trigger_type: str
    status: str
    started_at: datetime
    completed_at: datetime | None
    user_results: list[UserRunResultResponse]
    error: str | None


class RunTaskResponse(BaseModel):
    """Response from manually triggering a task."""

    run_id: str
    message: str


# Helper functions


def task_to_response(task: BackgroundTask) -> TaskResponse:
    """Convert BackgroundTask to response model."""
    if isinstance(task.trigger, CronTrigger):
        trigger_type = "cron"
        trigger_config = {"schedule": task.trigger.schedule}
    else:
        trigger_type = "idle"
        trigger_config = {
            "idle_minutes": task.trigger.idle_minutes,
            "cooldown_minutes": task.trigger.cooldown_minutes,
        }

    return TaskResponse(
        name=task.name,
        system_prompt=task.system_prompt,
        tools=task.tools,
        memory_blocks=task.memory_blocks,
        trigger_type=trigger_type,
        trigger_config=trigger_config,
        user_ids=task.user_ids,
        batch_size=task.batch_size,
        max_turns=task.max_turns,
        enabled=task.enabled,
    )


# Endpoints


@router.get("/tasks", response_model=list[TaskResponse])
async def list_tasks() -> list[TaskResponse]:
    """List all registered background tasks."""
    registry = get_registry()
    return [task_to_response(t) for t in registry.list_all()]


@router.get("/tasks/{name}", response_model=TaskResponse)
async def get_task(name: str) -> TaskResponse:
    """Get a background task by name."""
    registry = get_registry()
    task = registry.get(name)
    if not task:
        raise HTTPException(status_code=404, detail=f"Task '{name}' not found")
    return task_to_response(task)


@router.post("/tasks", response_model=TaskResponse)
async def create_task(request: CreateTaskRequest) -> TaskResponse:
    """Create or update a background task."""
    # Convert trigger
    if request.trigger.type == "cron":
        trigger = CronTrigger(schedule=request.trigger.schedule)  # type: ignore[union-attr]
    else:
        trigger = IdleTrigger(
            idle_minutes=request.trigger.idle_minutes,  # type: ignore[union-attr]
            cooldown_minutes=request.trigger.cooldown_minutes,  # type: ignore[union-attr]
        )

    task = BackgroundTask(
        name=request.name,
        system_prompt=request.system_prompt,
        tools=request.tools,
        memory_blocks=request.memory_blocks,
        trigger=trigger,
        user_ids=request.user_ids,
        batch_size=request.batch_size,
        max_turns=request.max_turns,
        enabled=request.enabled,
    )

    registry = get_registry()
    await registry.register(task)

    log.info("task_created_via_api", name=task.name)
    return task_to_response(task)


@router.delete("/tasks/{name}")
async def delete_task(name: str) -> dict[str, bool]:
    """Delete a background task."""
    registry = get_registry()
    deleted = await registry.unregister(name)
    if not deleted:
        raise HTTPException(status_code=404, detail=f"Task '{name}' not found")
    return {"deleted": True}


@router.post("/tasks/{name}/enable")
async def enable_task(name: str) -> TaskResponse:
    """Enable a background task."""
    registry = get_registry()
    if not await registry.set_enabled(name, True):
        raise HTTPException(status_code=404, detail=f"Task '{name}' not found")
    task = registry.get(name)
    return task_to_response(task)  # type: ignore[arg-type]


@router.post("/tasks/{name}/disable")
async def disable_task(name: str) -> TaskResponse:
    """Disable a background task."""
    registry = get_registry()
    if not await registry.set_enabled(name, False):
        raise HTTPException(status_code=404, detail=f"Task '{name}' not found")
    task = registry.get(name)
    return task_to_response(task)  # type: ignore[arg-type]


@router.post("/tasks/{name}/run", response_model=RunTaskResponse)
async def run_task(name: str) -> RunTaskResponse:
    """Manually trigger a background task to run now."""
    registry = get_registry()
    task = registry.get(name)
    if not task:
        raise HTTPException(status_code=404, detail=f"Task '{name}' not found")

    dolt = await get_dolt_client()
    executor = BackgroundExecutor(dolt)

    # Run synchronously for manual triggers (caller waits for completion)
    run = await executor.execute_task(task, TriggerType.CRON)

    return RunTaskResponse(
        run_id=run.id,
        message=f"Task '{name}' completed with status: {run.status.value}",
    )


@router.get("/tasks/{name}/runs", response_model=list[TaskRunResponse])
async def list_task_runs(name: str, limit: int = 50) -> list[TaskRunResponse]:
    """List execution history for a task."""
    dolt = await get_dolt_client()
    runs = await dolt.list_task_runs(task_name=name, limit=limit)
    return [
        TaskRunResponse(
            id=r.id,
            task_name=r.task_name,
            trigger_type=r.trigger_type.value,
            status=r.status.value,
            started_at=r.started_at,
            completed_at=r.completed_at,
            user_results=[
                UserRunResultResponse(
                    user_id=ur.user_id,
                    status=ur.status.value,
                    started_at=ur.started_at,
                    completed_at=ur.completed_at,
                    turns_used=ur.turns_used,
                    error=ur.error,
                    proposals_created=ur.proposals_created,
                )
                for ur in r.user_results
            ],
            error=r.error,
        )
        for r in runs
    ]


@router.get("/runs/{run_id}", response_model=TaskRunResponse)
async def get_task_run(run_id: str) -> TaskRunResponse:
    """Get details of a specific task run."""
    dolt = await get_dolt_client()
    run = await dolt.get_task_run(run_id)
    if not run:
        raise HTTPException(status_code=404, detail=f"Run '{run_id}' not found")
    return TaskRunResponse(
        id=run.id,
        task_name=run.task_name,
        trigger_type=run.trigger_type.value,
        status=run.status.value,
        started_at=run.started_at,
        completed_at=run.completed_at,
        user_results=[
            UserRunResultResponse(
                user_id=ur.user_id,
                status=ur.status.value,
                started_at=ur.started_at,
                completed_at=ur.completed_at,
                turns_used=ur.turns_used,
                error=ur.error,
                proposals_created=ur.proposals_created,
            )
            for ur in run.user_results
        ],
        error=run.error,
    )
```

#### 2. Update server.py to include router and track activity
**File**: `src/ralph/server.py` (updates)

Add import:
```python
from ralph.api.background import router as background_router
```

Add router after other routers:
```python
# Include the background tasks API router
app.include_router(background_router)
```

Update the `generate()` function in `chat_stream` to track user activity after successful response. Add this near the end of the try block, before `yield {"event": "message", "data": json.dumps({"type": "done"})}`:

```python
            # Track user activity for idle triggers
            try:
                await dolt.update_user_activity(request.user_id, datetime.utcnow())
            except Exception as e:
                log.warning("activity_tracking_failed", user_id=request.user_id, error=str(e))
```

Add import at top:
```python
from datetime import datetime
```

### Success Criteria

#### Automated Verification
- [x] Type checking passes: `make check-agent`
- [x] Linting passes: `uv run ruff check src/ralph/`
- [ ] Server starts: `uv run ralph-server`

#### Manual Verification
- [ ] `GET /background/tasks` returns empty list initially
- [ ] `POST /background/tasks` creates a task
- [ ] `POST /background/tasks/{name}/run` executes task and returns results
- [ ] `GET /background/tasks/{name}/runs` shows execution history
- [ ] Chat messages update user_activity table (check via SQL)

---

## Testing Strategy

### Unit Tests
- Task registry: register/unregister/list operations
- Scheduler trigger logic: cron parsing, idle detection
- Executor batch processing: concurrent user handling

### Integration Tests
- End-to-end task creation and execution via API
- Scheduler firing at correct times
- User activity tracking during chat

### Manual Testing Steps
1. Start Ralph server with Dolt running
2. Create a test task via API:
   ```bash
   curl -X POST http://localhost:8200/background/tasks \
     -H "Content-Type: application/json" \
     -d '{
       "name": "test-progress-tracker",
       "system_prompt": "You are a background agent. Review the student context and report what you see.",
       "tools": ["file_tools"],
       "memory_blocks": ["student", "progress"],
       "trigger": {"type": "cron", "schedule": "*/5 * * * *"},
       "user_ids": ["test-user-1"],
       "batch_size": 5,
       "max_turns": 5
     }'
   ```
3. Manually trigger the task:
   ```bash
   curl -X POST http://localhost:8200/background/tasks/test-progress-tracker/run
   ```
4. Check execution history:
   ```bash
   curl http://localhost:8200/background/tasks/test-progress-tracker/runs
   ```

## Performance Considerations

- **Batch concurrency**: `batch_size` controls parallel user processing; tune based on API rate limits
- **Scheduler interval**: Default 60s check interval balances responsiveness vs overhead
- **Connection pool**: Dolt client uses pool_size=20, sufficient for background + foreground
- **Memory**: In-memory registry is fine for <1000 tasks; Dolt provides persistence

## Future Enhancements (Out of Scope)

- **ARQ/Redis migration**: Replace in-process scheduler with ARQ workers
- **TOML configuration**: Load task definitions from config files
- **Retry logic**: Automatic retry for failed user runs
- **Task dependencies**: Run task B only after task A completes
- **UI dashboard**: Visual task management and monitoring

## References

- Current Ralph architecture: `src/ralph/server.py`
- Dolt client: `src/ralph/dolt.py`
- Memory context building: `src/ralph/memory.py`
- Legacy background system (reference): `src/youlab_server/background/`
- Agno Agent docs: https://docs.agno.com/
