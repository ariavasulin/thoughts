# Background Agent Implementation Research
**Ticket**: ARI-85 - Create proof-of-concept single-module course with background agent
**Date**: 2026-01-16
**Researched by**: Claude (via /research_codebase)

## Executive Summary

Background agents in YouLab are automated processes that run periodically to enrich student agents' memory blocks with insights derived from conversation history. They use Honcho's dialectic query system to analyze past conversations and apply targeted memory updates via the MemoryEnricher service.

**Key Findings:**
- Background agents are configured in TOML course files using `[[task]]` sections (v2 schema)
- Trigger options: `on_idle`, `schedule` (cron), `manual`, and `after_messages` (not yet implemented)
- No direct tool access - they execute pre-defined dialectic queries that update memory blocks
- Memory updates use three merge strategies: `append`, `replace`, and `llm_diff`

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│ Background Agent Flow                                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Trigger Event                                            │
│     ├─ Manual (API endpoint: POST /background/{id}/run)     │
│     ├─ on_idle (threshold + cooldown)                       │
│     ├─ schedule (cron expression)                            │
│     └─ after_messages (NOT IMPLEMENTED)                     │
│                                                              │
│  2. BackgroundAgentRunner.run_agent()                       │
│     ├─ Identify target users (from agent_types)             │
│     ├─ Process users in batches                             │
│     └─ For each user, execute all queries                   │
│                                                              │
│  3. Query Execution (_execute_query)                        │
│     ├─ HonchoClient.query_dialectic()                       │
│     │  └─ Analyzes conversation history                     │
│     │                                                        │
│     └─ MemoryEnricher.enrich()                              │
│        ├─ Apply merge strategy                              │
│        └─ Write audit entry to archival memory              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Implementation Details

### 1. Configuration Schema

**Location**: `src/youlab_server/curriculum/schema.py`

#### V2 Schema (Current - `[[task]]`)

```toml
[[task]]
name = "progression-grader"
on_idle = true
idle_threshold_minutes = 5
idle_cooldown_minutes = 30
manual = true
agent_types = ["tutor", "college-essay"]
user_filter = "all"
batch_size = 50
system = """Background agent system prompt (optional)"""

# Simple queries (executed sequentially)
queries = [
    {
        target = "journey.grader_notes",
        question = "Based on the recent conversation, assess progress...",
        scope = "all",
        merge = "replace"
    },
    {
        target = "student.insights",
        question = "What new insights emerged?",
        scope = "recent",
        recent_limit = 10,
        merge = "append"
    }
]

# Future: Full agent capabilities
# tools = ["send_message", "edit_memory_block"]
```

**Key Fields:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | - | Task identifier (optional in schema, set by TOML key) |
| `on_idle` | bool | false | Trigger when user is idle |
| `idle_threshold_minutes` | int | 30 | Minutes before considered idle |
| `idle_cooldown_minutes` | int | 60 | Minutes before re-triggering |
| `schedule` | string? | null | Cron expression (e.g., "0 3 * * *") |
| `manual` | bool | true | Can be triggered via API |
| `agent_types` | list[str] | ["tutor"] | Target agent types to process |
| `user_filter` | string | "all" | User filtering (currently only "all") |
| `batch_size` | int | 50 | Users per batch |
| `queries` | QueryConfig[] | [] | Dialectic queries to execute |
| `system` | string? | null | System prompt (future use) |
| `tools` | list[str] | [] | Tool access (future use) |

#### V1 Schema (Deprecated - `[background.*]`)

```toml
[background.progression-grader]
enabled = true
agent_types = ["tutor"]
batch_size = 50

[background.progression-grader.triggers]
manual = true
after_messages = 3  # NOT IMPLEMENTED YET
idle.enabled = true
idle.threshold_minutes = 5
idle.cooldown_minutes = 30

[[background.progression-grader.queries]]
id = "assess_progress"
question = "Has the student met the lesson objectives?"
session_scope = "all"
recent_limit = 5
target_block = "journey"
target_field = "grader_notes"
merge_strategy = "replace"
```

### 2. Trigger Mechanisms

**Location**: `src/youlab_server/curriculum/schema.py:111-118`

```python
class Triggers(BaseModel):
    schedule: str | None = None          # Cron expression
    manual: bool = True                   # API-triggered
    after_messages: int | None = None     # NOT IMPLEMENTED
    idle: IdleTrigger = Field(default_factory=IdleTrigger)
```

#### Implemented Triggers

1. **Manual Trigger** (`manual = true`)
   - Endpoint: `POST /background/{agent_id}/run`
   - Can specify `user_ids` in request body
   - Returns execution report (users processed, enrichments applied, errors)

2. **Idle Trigger** (`on_idle = true` in v2, `idle.enabled = true` in v1)
   - `threshold_minutes`: Wait time before considering user idle
   - `cooldown_minutes`: Wait time before re-triggering
   - Requires external scheduler to check idle status

3. **Schedule Trigger** (`schedule = "cron expression"`)
   - Standard cron format
   - Requires external scheduler (e.g., celery, APScheduler)

#### NOT Implemented

- **`after_messages`**: Trigger after N messages
  - Defined in schema but **not implemented in runner**
  - Would require message count tracking per user/agent
  - **This is what you need for "run every 3 messages"**

### 3. Background Agent Runner

**Location**: `src/youlab_server/background/runner.py`

**Key Methods:**

#### `run_agent(config, user_ids, agent_id)`
Main entry point - orchestrates the entire run.

```python
async def run_agent(
    config: BackgroundAgentConfig,
    user_ids: list[str] | None = None,  # None = all users
    agent_id: str = "unknown",
) -> RunResult
```

**Process:**
1. Check if enabled
2. Verify Honcho client available
3. Get target users (from `user_ids` or discover from `agent_types`)
4. Process users in batches
5. For each user, execute all queries
6. Return execution report

#### `_get_target_users(config)`
Discovers users by finding agents matching naming pattern.

**Agent Naming Convention**: `youlab_{user_id}_{agent_type}`

Example: `youlab_alice123_tutor`

#### `_execute_query(config, query, user_id, result, agent_id)`
Executes a single dialectic query and applies enrichment.

**Steps:**
1. Query Honcho dialectic with question
2. Find target agent by user_id + agent_type
3. Apply enrichment via MemoryEnricher
4. Record success/error in result

### 4. Dialectic Queries

**Location**: `src/youlab_server/curriculum/schema.py:120-130`

```python
class DialecticQuery(BaseModel):
    id: str                                    # Query identifier
    question: str                              # Question to ask
    session_scope: SessionScope = SessionScope.ALL
    recent_limit: int = 5                      # For RECENT scope
    target_block: str                          # Memory block to update
    target_field: str                          # Field within block
    merge_strategy: MergeStrategy = MergeStrategy.APPEND
```

**Session Scopes:**
- `ALL`: Query entire conversation history
- `RECENT`: Query last N messages (use `recent_limit`)
- `CURRENT`: Query current session only
- `SPECIFIC`: Query specific session ID

**Merge Strategies:**
- `APPEND`: Add to existing content (lists or concatenate strings)
- `REPLACE`: Overwrite existing content
- `LLM_DIFF`: Use LLM to merge intelligently (NOT IMPLEMENTED - falls back to append)

### 5. Memory Enrichment

**Location**: `src/youlab_server/memory/enricher.py`

Background agents **do not have direct tool access**. Instead, they use the MemoryEnricher service to apply updates.

```python
class MemoryEnricher:
    def enrich(
        agent_id: str,
        block: str,          # "human", "persona", or custom block
        field: str,          # Field name within block
        content: str,        # Insight from dialectic query
        strategy: MergeStrategy = APPEND,
        source: str = "background_worker",
        source_query: str | None = None,
    ) -> EnrichmentResult
```

**Process:**
1. Get current block from agent memory
2. Apply merge strategy to update field
3. Write updated block back to agent
4. **Write audit entry to archival memory**

**Audit Entry Format:**
```
[MEMORY_EDIT 2026-01-16T10:30:00]
Source: background:progression-grader
Block: journey
Field: grader_notes
Strategy: replace
Query: Based on the recent conversation, assess...
Content: Student has demonstrated understanding of...
```

### 6. Server Integration

**Location**: `src/youlab_server/server/background.py`

**Endpoints:**

1. `GET /background/agents`
   - Lists all configured background agents across all courses
   - Returns: agent_id, course_id, enabled status, trigger config, query count

2. `POST /background/{agent_id}/run`
   - Manually trigger a background agent
   - Body: `{ "user_ids": ["user1", "user2"] }` (optional)
   - Returns: execution report

3. `POST /background/config/reload`
   - Reloads TOML configuration files
   - Returns: courses loaded, course IDs

## Example Configuration

**Working Example**: `config/courses/college-essay/course.toml:114-136`

```toml
[[task]]
name = "progression-grader"
on_idle = true
idle_threshold_minutes = 5
idle_cooldown_minutes = 30
manual = true
agent_types = ["tutor", "college-essay"]
user_filter = "all"
batch_size = 50
system = """You are a curriculum grader for the YouLab college essay course.

Your job is to evaluate recent conversation and assess:
1. Has the student met the objectives for their current lesson?
2. What insights have emerged that should be captured?
3. Are there any blockers to progression?

Be specific in your notes - the tutor will read these to understand where to focus."""

queries = [
    {
        target = "journey.grader_notes",
        question = "Based on the recent conversation, assess the student's progress on the current lesson objectives. What has been accomplished? What depth of understanding is demonstrated?",
        scope = "all",
        merge = "replace"
    },
    {
        target = "journey.blockers",
        question = "What specific gaps or missing elements would prevent this student from progressing? Be concrete.",
        scope = "all",
        merge = "replace"
    },
    {
        target = "student.insights",
        question = "What new insights about this student emerged from the recent conversation? Focus on emotional moments, potential essay themes, and learning patterns.",
        scope = "all",
        merge = "append"
    },
    {
        target = "engagement_strategy.approach",
        question = "Based on how this student has engaged, what adjustments should the coach make to their approach?",
        scope = "all",
        merge = "llm_diff"
    }
]
```

## Tool Access (NOT CURRENTLY AVAILABLE)

Background agents currently **do not have direct tool access**. They can only:
1. Execute pre-defined dialectic queries via Honcho
2. Apply memory updates via MemoryEnricher

**Future**: The v2 schema includes `tools` and `system` fields for full agent capabilities:

```toml
[[task]]
system = """You are a background grader agent..."""
tools = ["edit_memory_block", "query_honcho"]  # Future
queries = [...]  # Simple queries for common cases
```

This would allow background agents to:
- Use `edit_memory_block` tool directly
- Use `query_honcho` tool for complex queries
- Run arbitrary LLM reasoning with full tool access

## How to Configure "Every 3 Messages"

**Current Status**: `after_messages` trigger is **NOT IMPLEMENTED**.

**Schema Exists (v1)**:
```toml
[background.my-agent.triggers]
after_messages = 3
```

**But the runner ignores it** - see `src/youlab_server/background/runner.py:67-133`.

### Implementation Requirements

To implement `after_messages` trigger:

1. **Message Count Tracking**
   - Add message counter to agent memory or separate store
   - Increment on each user message
   - Reset counter after background agent runs

2. **Trigger Check**
   - Add middleware to chat endpoint
   - Check if message count >= threshold
   - Queue background agent run

3. **Integration Point**
   - Likely in: `src/youlab_server/server/main.py` or chat handler
   - After processing user message, check trigger conditions
   - Call `BackgroundAgentRunner.run_agent()` if triggered

### Workaround (Current)

Use **idle trigger** with short threshold:

```toml
[[task]]
on_idle = true
idle_threshold_minutes = 1  # Run 1 minute after last message
idle_cooldown_minutes = 5   # Don't re-run for 5 minutes
```

This approximates "run after conversation pauses" but not "every N messages".

## Testing

**Test Files:**
- `tests/test_background_runner.py` - Runner logic
- `tests/test_background_schema.py` - Config schema validation
- `tests/test_server/test_background.py` - API endpoints

**Test Coverage:**
- Disabled agent handling
- Missing Honcho client
- User discovery and batching
- Query execution and enrichment
- Error handling (missing agents, failed queries)
- Multiple queries per run

## Key Takeaways

1. **Background agents are configuration-driven** - no code needed for simple queries
2. **No tool access yet** - only dialectic queries + memory enrichment
3. **Triggers are partially implemented** - `manual`, `idle`, `schedule` work; `after_messages` does not
4. **Memory updates are audited** - all enrichments logged to archival memory
5. **V2 schema is cleaner** - `[[task]]` instead of `[background.*]`
6. **For "every 3 messages"** - need to implement `after_messages` trigger or use idle workaround

## Implementation Recommendations for ARI-85

For a POC single-module course with background agent:

1. **Use v2 schema** (`[[task]]`)
2. **Start with manual trigger** for testing
3. **Use 1-2 simple queries** to validate flow
4. **Target existing blocks** (student, journey, etc.)
5. **Use `append` merge** for safety (easy to inspect)
6. **Test via API**: `POST /background/{agent_id}/run`

**Sample minimal config:**

```toml
[[task]]
name = "insight-harvester"
manual = true
agent_types = ["tutor"]
queries = [
    {
        target = "student.insights",
        question = "What did the student reveal about themselves in this conversation?",
        scope = "recent",
        recent_limit = 5,
        merge = "append"
    }
]
```

Then test:
```bash
curl -X POST http://localhost:8000/background/insight-harvester/run
```

## Related Files

- `src/youlab_server/background/runner.py` - Core execution engine
- `src/youlab_server/curriculum/schema.py:103-141` - Config schema
- `src/youlab_server/memory/enricher.py` - Memory update service
- `src/youlab_server/server/background.py` - API endpoints
- `src/youlab_server/honcho/client.py` - Dialectic query interface
- `config/courses/college-essay/course.toml` - Working example
- `tests/test_background_runner.py` - Test suite
