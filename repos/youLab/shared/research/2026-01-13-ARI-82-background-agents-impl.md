---
date: 2026-01-13T16:54:49+07:00
researcher: ARI
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "Background Agents Implementation - Current State and Hidden Models Gap"
tags: [research, codebase, background-agents, openwebui, hidden-models, ARI-82]
status: complete
last_updated: 2026-01-13
last_updated_by: ARI
---

# Research: Background Agents Implementation

**Date**: 2026-01-13T16:54:49+07:00
**Researcher**: ARI
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab
**Linear Ticket**: ARI-82

## Research Question

Understand background agents implementation and how they should become hidden models. Focus on:
1. Background agent runner and TOML config
2. Thread management
3. How background agents are triggered (time-based, after x messages)
4. Current agent thread storage pattern
5. OpenWebUI integration for background agent visibility

Gap analysis: New spec says background agents = hidden models with threads persisted but users can't create new chats.

## Summary

The YouLab codebase has a well-structured background agent system with:
- **Configuration**: Full TOML-based schema for triggers (`schedule`, `after_messages`, `on_idle`)
- **Execution**: `BackgroundAgentRunner` that queries Honcho and enriches memory blocks
- **Storage**: Git-backed thread metadata at `{user_dir}/agent_threads/{agent_name}/{chat_id}.json`
- **UI**: AgentsTab.svelte showing agents with clickable thread history

**Critical Gap**: Triggers are NOT implemented - only manual execution works. The spec envisions background agents as "hidden models" where threads are visible but users can't start new conversations. Current implementation has no concept of hidden models.

## Detailed Findings

### 1. Background Agent Runner

**Location**: `src/youlab_server/background/runner.py`

The `BackgroundAgentRunner` class executes background agents based on configuration:

```python
class BackgroundAgentRunner:
    def __init__(self, letta_client, honcho_client):
        self.letta = letta_client
        self.honcho = honcho_client
        self.enricher = MemoryEnricher(letta_client)
```

**Execution Flow** (`runner.py:67-133`):
1. Validates agent is enabled
2. Gets target users via `_get_target_users()` (matches agent naming pattern `youlab_{user_id}_{agent_type}`)
3. Processes users in batches (`batch_size` from config)
4. For each user, executes dialectic queries
5. Enriches memory blocks with insights

**Dialectic Query Execution** (`runner.py:176-227`):
```python
async def _execute_query(self, config, query, user_id, result, agent_id):
    # Query Honcho dialectic
    response = await self.honcho.query_dialectic(
        user_id=user_id,
        question=query.question,
        session_scope=scope,
        recent_limit=query.recent_limit,
    )

    # Apply enrichment to target block
    self.enricher.enrich(
        agent_id=target_agent_id,
        block=query.target_block,
        field=query.target_field,
        content=response.insight,
        strategy=strategy,
    )
```

### 2. TOML Configuration Schema

**Location**: `src/youlab_server/curriculum/schema.py:99-141`

#### TaskConfig (v2 format)

```python
class TaskConfig(BaseModel):
    # Scheduling
    schedule: str | None = None      # Cron expression
    manual: bool = True              # Allow manual triggers
    on_idle: bool = False            # Idle-based trigger
    idle_threshold_minutes: int = 30
    idle_cooldown_minutes: int = 60

    # Scope
    agent_types: list[str] = ["tutor"]
    user_filter: str = "all"
    batch_size: int = 50

    # Queries
    queries: list[QueryConfig] = []

    # Full agent capabilities
    system: str | None = None
    tools: list[str] = []
```

#### Live Configuration Example

**File**: `config/courses/college-essay/course.toml:114-136`

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
system = """You are a curriculum grader..."""
queries = [
    { target = "journey.grader_notes", question = "...", scope = "all", merge = "replace" },
    { target = "journey.blockers", question = "...", scope = "all", merge = "replace" },
    { target = "student.insights", question = "...", scope = "all", merge = "append" },
    { target = "engagement_strategy.approach", question = "...", scope = "all", merge = "llm_diff" }
]
```

### 3. Trigger Implementation Status

| Trigger Type | Config Exists | Implementation |
|--------------|---------------|----------------|
| `manual` | Yes | **Working** via HTTP endpoint |
| `schedule` (cron) | Yes | **Not implemented** - no scheduler |
| `on_idle` | Yes | **Not implemented** - no idle detection |
| `after_messages` | Yes | **Not implemented** - no message counting |

**Manual Trigger Endpoint** (`src/youlab_server/server/background.py:96-138`):
```python
@router.post("/{agent_id}/run")
async def run_background_agent(agent_id: str, request: RunRequest):
    result = await runner.run_agent(config=config, user_ids=request.user_ids)
    return {...}
```

**No Scheduler Infrastructure**:
- No APScheduler, Celery, or similar installed (`pyproject.toml` confirms)
- No cron parsing or scheduled job registration
- No idle timestamp tracking per user
- No message counting logic in chat endpoints

### 4. Agent Thread Storage Pattern

**Location**: `src/youlab_server/server/agents_threads.py`

#### Directory Structure
```
{user_dir}/
  agent_threads/
    {agent_name}/           # e.g., "progression_grader"
      {chat_id}.json        # Thread metadata file
```

#### Thread Metadata Format
```json
{
  "chat_id": "abc123-def456",
  "created_at": "2026-01-13T10:30:00.123456",
  "agent_name": "Progression Grader"
}
```

#### Thread Registration (`agents_threads.py:125-165`)
```python
@router.post("/{agent_name}/threads")
async def register_agent_thread(user_id, agent_name, chat_id, storage):
    agent_dir = user_storage.user_dir / "agent_threads" / agent_name.lower().replace(" ", "_")
    agent_dir.mkdir(parents=True, exist_ok=True)

    thread_file = agent_dir / f"{chat_id}.json"
    thread_file.write_text(json.dumps({
        "chat_id": chat_id,
        "created_at": datetime.now().isoformat(),
        "agent_name": agent_name,
    }))
```

#### OpenWebUI Thread Creation (`agents_threads.py:168-221`)
```python
async def create_agent_thread(user_id, agent_id, agent_name, openwebui_client):
    # Ensure folder exists with metadata
    folder_id = await openwebui_client.ensure_folder(
        folder_name,
        meta={"type": "background_agent", "agentId": agent_id},
    )

    # Archive previous threads
    existing_chats = await openwebui_client.get_chats_by_folder(folder_id)
    for chat in existing_chats:
        if not chat.get("archived"):
            await openwebui_client.archive_chat(chat["id"])

    # Create new chat with date title
    title = f"{agent_name} - {date_str}"
    chat = await openwebui_client.create_chat({"title": title, ...}, folder_id)
    return chat["id"]
```

### 5. OpenWebUI Frontend Integration

#### AgentsTab Component

**Location**: `OpenWebUI/open-webui/src/lib/components/you/AgentsTab.svelte`

Displays background agents in expandable list:
- Agent name with pending diffs badge
- Thread runs sorted by date (newest first)
- Clickable links: `href="/?chat={thread.chatId}"`

```svelte
{#each agents as agent}
  <CollapsibleSection>
    <span>{agent.name}</span>
    {#if agent.pendingDiffs > 0}
      <span class="badge">{agent.pendingDiffs}</span>
    {/if}

    {#each agent.threads as thread}
      <a href="/?chat={thread.chatId}">{thread.displayDate}</a>
    {/each}
  </CollapsibleSection>
{/each}
```

#### Frontend API

**Location**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`

```typescript
export async function getBackgroundAgents(userId: string, token: string): Promise<BackgroundAgent[]> {
    const response = await fetch(`${YOULAB_API_BASE_URL}/users/${userId}/agents`, {...});
    return response.json();
}

interface BackgroundAgent {
    name: string;
    pendingDiffs: number;
    threads: ThreadRun[];
}

interface ThreadRun {
    id: string;
    chatId: string;
    date: string;
    displayDate: string;
}
```

#### Route Integration

**Location**: `OpenWebUI/open-webui/src/routes/(app)/you/+page.svelte`

Tabs: "Profile" (memory blocks) | "Agents" (background agent threads)

### 6. Pending Diffs System

Background agents can propose block edits that require user approval:

**Location**: `src/youlab_server/storage/diffs.py`

```python
@dataclass
class PendingDiff:
    id: str                    # UUID
    agent_id: str              # Proposing agent
    block_label: str           # Target block
    field: str | None          # Specific field
    operation: str             # "append", "replace", "llm_diff"
    current_value: str
    proposed_value: str
    reasoning: str             # Agent explanation
    confidence: str            # "low", "medium", "high"
    status: str                # "pending", "approved", "rejected", "superseded"
```

Flow:
1. Background agent proposes edit via `UserBlockManager.propose_edit()`
2. Diff saved to `{user_dir}/pending_diffs/{diff_id}.json`
3. UI shows diffs per agent with approve/reject buttons
4. Approval applies changes and creates git commit

## Gap Analysis: Hidden Models

### Current State

| Aspect | Current Implementation |
|--------|----------------------|
| Thread visibility | Visible in folders, users can click to view |
| Chat creation | Not explicitly prevented - users could start new chats |
| Folder metadata | `{"type": "background_agent", "agentId": agent_id}` |
| Model selection | No restriction on which models appear in selector |

### Spec Requirements (Hidden Models)

1. **Background agents = hidden models**: Threads persisted but users can't create new chats
2. **Visibility**: Users should see agent history (read-only)
3. **Creation restriction**: No "new chat" button for background agent models

### Gap Areas

| Gap | Description | Impact |
|-----|-------------|--------|
| **No hidden model concept** | OpenWebUI has no mechanism to hide models from selector | Users can potentially start chats with background agents |
| **No read-only mode** | Threads are regular OpenWebUI chats | Users could continue conversations in background agent threads |
| **Folder-based only** | Background agents identified by folder metadata | No model-level restrictions |
| **Triggers not implemented** | Background agents can't auto-run | Manual-only execution limits usefulness |

### Potential Implementation Approaches

1. **OpenWebUI model filtering**: Add `hidden` flag to model config, filter from selector
2. **Chat creation prevention**: Check folder metadata before allowing new messages
3. **Separate UI path**: Background agent threads rendered differently (view-only)
4. **API-level guards**: Block POST to `/chat` for background agent folder chats

## Code References

| Component | File | Lines |
|-----------|------|-------|
| BackgroundAgentRunner | `src/youlab_server/background/runner.py` | 46-237 |
| TaskConfig schema | `src/youlab_server/curriculum/schema.py` | 169-190 |
| Agent threads API | `src/youlab_server/server/agents_threads.py` | 1-222 |
| OpenWebUI client (folders) | `src/youlab_server/server/sync/openwebui_client.py` | 304-375 |
| AgentsTab component | `OpenWebUI/open-webui/src/lib/components/you/AgentsTab.svelte` | 1-118 |
| Frontend API | `OpenWebUI/open-webui/src/lib/apis/memory/index.ts` | Background agents section |
| Manual trigger endpoint | `src/youlab_server/server/background.py` | 96-138 |
| Pending diffs | `src/youlab_server/storage/diffs.py` | 19-143 |

## Architecture Documentation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Background Agent Architecture                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Configuration (TOML)              Execution                         │
│  ┌──────────────────┐             ┌───────────────────────┐         │
│  │ course.toml      │───────────▶│ BackgroundAgentRunner │         │
│  │  [[task]]        │             │   - run_agent()       │         │
│  │    queries=[...] │             │   - process users     │         │
│  │    on_idle=true  │             │   - execute queries   │         │
│  └──────────────────┘             └─────────┬─────────────┘         │
│                                             │                        │
│  ┌──────────────────┐             ┌─────────▼─────────────┐         │
│  │ HonchoClient     │◀────────────│ Dialectic Queries     │         │
│  │  query_dialectic │             │   - session scope     │         │
│  └──────────────────┘             │   - recent limit      │         │
│                                   └─────────┬─────────────┘         │
│                                             │                        │
│  ┌──────────────────┐             ┌─────────▼─────────────┐         │
│  │ MemoryEnricher   │◀────────────│ Enrichment            │         │
│  │  - append        │             │   - target block      │         │
│  │  - replace       │             │   - merge strategy    │         │
│  │  - llm_diff      │             └───────────────────────┘         │
│  └──────────────────┘                                               │
│                                                                      │
│  Storage                          OpenWebUI Integration              │
│  ┌──────────────────┐             ┌───────────────────────┐         │
│  │ agent_threads/   │             │ Folder + Chats        │         │
│  │   {agent}/       │◀───────────▶│   meta.type=          │         │
│  │     {chat}.json  │             │   "background_agent"  │         │
│  └──────────────────┘             └───────────────────────┘         │
│                                                                      │
│  ┌──────────────────┐             ┌───────────────────────┐         │
│  │ pending_diffs/   │             │ AgentsTab.svelte      │         │
│  │   {diff_id}.json │─────────────│   - show threads      │         │
│  └──────────────────┘             │   - pending badges    │         │
│                                   └───────────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

## Open Questions

1. **Hidden model implementation**: What changes are needed in OpenWebUI to support hidden models?
2. **Read-only thread viewing**: How to prevent users from continuing background agent conversations?
3. **Trigger implementation priority**: Which triggers (schedule, idle, after_messages) should be implemented first?
4. **Model filtering API**: Does OpenWebUI support model visibility filtering, or is modification needed?
5. **Background agent orchestration**: How will the scheduler/trigger system be deployed (separate process, FastAPI background task, external scheduler)?
