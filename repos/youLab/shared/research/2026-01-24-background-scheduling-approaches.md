---
date: "2026-01-24T12:00:00-08:00"
researcher: ARI
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Background Agent Scheduling for YouLab v2"
tags: [research, background-agents, scheduling, apscheduler, celery, redis, toml-config]
status: complete
last_updated: "2026-01-24"
last_updated_by: ARI
---

# Research: Background Agent Scheduling for YouLab v2

**Date**: 2026-01-24T12:00:00-08:00
**Researcher**: ARI
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Evaluate scheduling approaches for YouLab v2 background agents. Analyze the current implementation, assess options (TOML-configured, Dolt-driven queue, APScheduler, Redis/Celery), and consider trigger types, persistence, observability, and self-hosting requirements.

## Summary

The current YouLab background agent system has **trigger schema definitions but no automated execution**. Background tasks only run via manual HTTP trigger (`POST /background/{task_name}/run`). The schema supports idle triggers, cron schedules, and after_messages triggers—none of which are implemented.

For YouLab v2 with <1000 users, the recommended approach is **APScheduler 3.x + PostgreSQL/SQLite** for scheduled triggers, with optional **Dramatiq/Taskiq + Redis** for heavy background work. This provides robust scheduling without the operational complexity of Celery.

## Detailed Findings

### 1. Current Implementation State

#### Entry Points and Flow

- **HTTP Endpoint**: `src/youlab_server/server/background.py:107` - `POST /background/{agent_id}/run`
- **Runner**: `src/youlab_server/background/runner.py:67` - `BackgroundAgentRunner` class
- **Factory**: `src/youlab_server/background/factory.py:22` - `BackgroundAgentFactory` for Letta agents

#### What Works Today

| Feature | Status | Location |
|---------|--------|----------|
| Manual HTTP trigger | ✅ Implemented | `background.py:107` |
| Config loading (v1 + v2) | ✅ Implemented | `background.py:131-151` |
| Agent-based execution | ✅ Implemented | `runner.py:251-309` |
| Query-based execution (legacy) | ✅ Implemented | `runner.py:310-371` |
| PendingDiff creation | ✅ Implemented | `runner.py:342-366` |
| Batch processing | ✅ Implemented | `runner.py:201-218` |

#### What's NOT Implemented (Schema Only)

| Trigger Type | Schema Location | Status |
|--------------|----------------|--------|
| `on_idle` | `schema.py:185` | Schema only, no scheduler |
| `idle_threshold_minutes` | `schema.py:186` | Schema only |
| `idle_cooldown_minutes` | `schema.py:187` | Schema only |
| `schedule` (cron) | `schema.py:183` | Schema only |
| `after_messages` | `schema.py:119` (v1 only) | Schema only, deprecated in v2 |

#### Current Architecture Diagram

```
Manual HTTP Request
        ↓
POST /background/{task_name}/run
        ↓
BackgroundAgentRunner.run_task()
        ↓
    ┌───────────────────────────────┐
    │ For each user in batch:       │
    │   ├─ Get memory blocks        │
    │   ├─ Create Letta agent       │
    │   ├─ Send instruction         │
    │   └─ Agent calls tools:       │
    │       ├─ query_honcho         │
    │       └─ edit_memory_block    │
    │           ↓                   │
    │       PendingDiff created     │
    └───────────────────────────────┘
```

### 2. TOML Configuration Schema (Current)

The existing TOML schema (`config/courses/*/course.toml`) defines task configuration:

```toml
[[task]]
name = "progression-grader"
on_idle = true                    # NOT IMPLEMENTED
idle_threshold_minutes = 5        # NOT IMPLEMENTED
idle_cooldown_minutes = 30        # NOT IMPLEMENTED
schedule = "0 */6 * * *"          # NOT IMPLEMENTED (cron syntax)
manual = true                     # IMPLEMENTED
agent_types = ["tutor", "college-essay"]
user_filter = "all"
batch_size = 50
system = "You are a curriculum grader..."
tools = ["query_honcho", "edit_memory_block"]
queries = [...]  # Legacy format, deprecated
```

#### Task Schema (`src/youlab_server/curriculum/schema.py:172-202`)

```python
class TaskConfig(BaseModel):
    name: str = ""
    schedule: str | None = None           # Cron syntax
    manual: bool = True
    on_idle: bool = False
    idle_threshold_minutes: int = 30
    idle_cooldown_minutes: int = 60
    agent_types: list[str] = Field(default_factory=lambda: ["tutor"])
    user_filter: str = "all"
    batch_size: int = 50
    queries: list[QueryConfig] = Field(default_factory=list)  # Deprecated
    system: str | None = None              # Agent system prompt
    tools: list[str] = Field(default_factory=list)
```

### 3. Scheduling Approach Analysis

#### Option A: TOML-Configured + APScheduler (Recommended)

**How it would work:**
1. Parse TOML `schedule` and `on_idle` fields during config loading
2. APScheduler runs in-process with FastAPI server
3. Scheduler creates jobs based on parsed TOML configs
4. Jobs call existing `BackgroundAgentRunner.run_task()` method

**Implementation Requirements:**
- APScheduler 3.x with PostgreSQL/SQLite JobStore
- Activity tracker for idle detection (last_message_at per user)
- Cooldown state storage (last_run_at per user per task)

**Pros:**
- Pure Python, no external broker required
- Jobs survive restarts with persistent JobStore
- Native cron expression support
- Low operational complexity

**Cons:**
- Idle detection requires custom activity tracking
- No built-in distributed execution (single server)
- APScheduler 4.0 still in pre-release

**Estimated Effort:** 2-3 days

#### Option B: Dolt-Driven Queue

**How it would work:**
1. Tasks stored as rows in Dolt versioned database
2. Worker process polls queue table for pending tasks
3. Task execution creates git-like commits in Dolt
4. Built-in audit trail through Dolt history

**Pros:**
- Built-in versioning and audit trail
- SQL interface for task management
- Branch-based testing (try tasks on branch before merge)

**Cons:**
- Adds new infrastructure dependency (Dolt)
- Not designed as a job queue (polling inefficiency)
- Limited ecosystem compared to Redis-based solutions
- Team unfamiliar with Dolt

**Estimated Effort:** 5-7 days

#### Option C: In-Memory (APScheduler Without Persistence)

**How it would work:**
- APScheduler with MemoryJobStore
- Jobs lost on restart, recreated from TOML on startup

**Pros:**
- Zero infrastructure dependencies
- Simplest implementation

**Cons:**
- Jobs lost on restart (must reload from TOML)
- No distributed execution
- No durability guarantees
- Not suitable for production

**Estimated Effort:** 1 day

#### Option D: External Queue (Redis + Dramatiq/Celery)

**How it would work:**
1. APScheduler creates scheduled tasks → pushes to Redis queue
2. Dramatiq/Celery worker processes tasks
3. Worker isolation from web server

**Redis + Dramatiq (Recommended if going this route):**
- Modern, simpler than Celery
- 4s for 20K jobs (benchmark)
- Built-in retries, rate limiting

**Redis + Celery:**
- Battle-tested, most features
- 12s for 20K jobs (benchmark)
- Complex setup for small teams

**Pros:**
- Horizontal scaling (multiple workers)
- Task isolation from web process
- Mature retry/failure handling
- Redis likely already in stack for caching

**Cons:**
- Added infrastructure (Redis)
- More moving parts
- Overkill for <1000 users with simple tasks

**Estimated Effort:** 3-5 days (Dramatiq), 5-7 days (Celery)

### 4. Trigger Type Implementation

#### Cron/Schedule Triggers

**APScheduler Implementation:**
```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

scheduler = AsyncIOScheduler()

# Parse TOML schedule field
for task in course.tasks:
    if task.schedule:
        scheduler.add_job(
            runner.run_task,
            CronTrigger.from_crontab(task.schedule),
            args=[task],
            id=f"{course.id}:{task.name}",
        )
```

#### Idle Triggers

**Implementation Requirements:**
1. **Activity Tracking**: Store `last_message_at` per user
   - Update on each chat message
   - Storage: Redis, PostgreSQL, or user storage files

2. **Idle Check Job**: APScheduler interval job (e.g., every 1 minute)
   ```python
   async def check_idle_triggers():
       for task in get_idle_tasks():
           for user_id in get_users():
               if is_idle(user_id, task.idle_threshold_minutes):
                   if not in_cooldown(user_id, task.name, task.idle_cooldown_minutes):
                       await runner.run_task(task, [user_id])
                       record_run(user_id, task.name)
   ```

3. **Cooldown Storage**: Track `last_run_at` per user per task

#### After-Messages Triggers (Deprecated in v2)

Would require:
- Message counter per user per agent
- Hook in chat handler to increment counter
- Trigger when threshold reached

### 5. Persistence Requirements

| Approach | Job Persistence | State Storage | Observability |
|----------|----------------|---------------|---------------|
| APScheduler + PostgreSQL | ✅ SQLAlchemy JobStore | PostgreSQL table | Logs + manual metrics |
| APScheduler + SQLite | ✅ SQLite file | SQLite table | Logs only |
| Redis + Dramatiq | ✅ Redis streams | Redis keys | Redis monitoring |
| Dolt Queue | ✅ Dolt tables | Dolt versioned | Git-like history |

**State to Persist:**
- `last_run_at` per user per task (for cooldown)
- `last_message_at` per user (for idle detection)
- Job execution history (optional, for debugging)

### 6. Observability Considerations

**Current State:**
- Structured logging via `structlog` (runner.py:90, factory.py:41)
- Log events: `background_task_started`, `background_task_completed`, `pending_diff_created`
- Errors collected in `RunResult.errors` list
- No external metrics or tracing

**Gaps:**
- No Langfuse tracing integration (chat has it, background doesn't)
- No Prometheus/StatsD metrics
- No timing metrics (task duration, latency)
- Error storage is ephemeral (response only)

**Recommended Additions:**
1. OpenTelemetry spans around task execution
2. Metrics: `background_tasks_total`, `background_task_duration_seconds`, `background_task_errors_total`
3. Langfuse trace for agent tool calls

### 7. Self-Hosting Considerations

**YouLab Deployment Context:**
- Single-server deployment (initially)
- <1000 users target
- Already using PostgreSQL (likely) or SQLite

**Recommendation:** Avoid introducing Redis initially. APScheduler with PostgreSQL/SQLite JobStore provides sufficient scheduling without new infrastructure.

**When to Add Redis:**
- Task execution takes >30 seconds (worker isolation needed)
- Horizontal scaling required (multiple servers)
- Already using Redis for caching

### 8. Python Task Scheduling 2025-2026 Trends

**From Web Research:**

| Library | 20K Jobs Benchmark | Notes |
|---------|-------------------|-------|
| **Taskiq** | 2.03s | Fastest, async-native, Redis Streams |
| **Huey** | 3.62s | Lightweight, excellent performance |
| **Dramatiq** | 4.12s | Reliable, modern Celery alternative |
| **Celery** | 11.68s | Battle-tested, most features |
| **ARQ** | 35.37s | Async but slower in practice |
| **RQ** | 51.05s | Simplest but slowest |

**Trends:**
- Async-native libraries (Taskiq, Dramatiq) gaining over Celery
- PostgreSQL as broker (Procrastinate, PgQueuer) eliminating Redis
- Simpler alternatives preferred for new projects
- Workflow engines (Temporal.io, Inngest) for AI agent orchestration

## Recommendations

### For YouLab v2 (Immediate)

**Phase 1: APScheduler + File/PostgreSQL (1-2 days)**
```
FastAPI Server
     │
     ├── APScheduler (in-process)
     │       ├── Cron jobs from TOML `schedule`
     │       └── Interval job for idle check
     │
     └── BackgroundAgentRunner (existing)
             └── PendingDiffs + Git storage (existing)
```

**Implementation:**
1. Add `apscheduler>=3.10` dependency
2. Create `src/youlab_server/background/scheduler.py`
3. Initialize scheduler in FastAPI lifespan
4. Parse TOML configs → create APScheduler jobs
5. Store activity/cooldown state in user storage files

**Phase 2: Activity Tracking (1 day)**
1. Update `last_message_at` in chat handler
2. Store in user storage JSON file
3. Implement idle check interval job

**Phase 3: Cooldown Enforcement (0.5 days)**
1. Store `last_run_at` per task in user storage
2. Check cooldown before task execution

### For Scale (Future)

If background tasks become bottleneck:
1. Add Redis for state storage
2. Move to Dramatiq for worker isolation
3. APScheduler remains for scheduling, pushes to Dramatiq queue

## Code References

- `src/youlab_server/background/runner.py:67` - BackgroundAgentRunner class
- `src/youlab_server/background/runner.py:4-15` - TODO comments listing missing trigger features
- `src/youlab_server/background/factory.py:22` - BackgroundAgentFactory class
- `src/youlab_server/curriculum/schema.py:172-202` - TaskConfig with trigger fields
- `src/youlab_server/curriculum/schema.py:179-182` - TODO comment for on_idle
- `src/youlab_server/server/background.py:107` - HTTP trigger endpoint
- `config/courses/college-essay/course.toml:114-136` - Example task configuration
- `config/courses/poc-tutor/course.toml:71-104` - POC task configuration

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-16-ARI-85-poc-course-background-agent.md` - POC implementation plan, documents current state and blockers
- `thoughts/shared/plans/2026-01-08-phase-6-background-agents.md` - Original background agents phase plan
- `thoughts/shared/research/2026-01-13-ARI-82-background-agents-impl.md` - Background agents implementation research

## Related Research

- [APScheduler GitHub](https://github.com/agronholm/apscheduler)
- [APScheduler User Guide](https://apscheduler.readthedocs.io/en/3.x/userguide.html)
- [Dramatiq Motivation](https://dramatiq.io/motivation.html)
- [Python Task Queue Benchmark 2025](https://stevenyue.com/blogs/exploring-python-task-queue-libraries-with-load-test)
- [OpenTelemetry Celery Guide](https://uptrace.dev/guides/opentelemetry-celery)

## Open Questions

1. **Activity Storage Location**: Should `last_message_at` be stored in user storage files, a separate JSON, or PostgreSQL table?
2. **Scheduler Recovery**: How should the scheduler handle jobs missed during server downtime?
3. **Multi-Server Deployment**: If YouLab scales to multiple servers, how should scheduler leader election work?
4. **Observability Priority**: Should Langfuse tracing for background agents be implemented before or after scheduling?
