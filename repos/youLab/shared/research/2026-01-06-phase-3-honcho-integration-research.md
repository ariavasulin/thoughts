---
date: 2026-01-06T11:17:50+07:00
researcher: ariasulin
git_commit: 29783e20ce6d230469eee40148d80b068d2fd021
branch: main
repository: YouLab
topic: "Phase 3: Honcho Integration Research"
tags: [research, honcho, integration, theory-of-mind, memory, personalization]
status: complete
last_updated: 2026-01-06
last_updated_by: ariasulin
---

# Research: Phase 3 Honcho Integration

**Date**: 2026-01-06T11:17:50+07:00
**Researcher**: ariasulin
**Git Commit**: 29783e20ce6d230469eee40148d80b068d2fd021
**Branch**: main
**Repository**: YouLab

## Research Question

Comprehensive research for Phase 3: Honcho Integration. Document current codebase state, Honcho SDK capabilities, and identify integration points between LettaStarter and Honcho for implementing theory-of-mind capabilities.

## Summary

This research documents the current LettaStarter architecture and provides comprehensive Honcho SDK details for Phase 3 integration. The codebase has a well-structured HTTP service with per-user agent management, memory blocks with rotation strategies, and streaming chat support. Honcho integration will add message persistence and theory-of-mind modeling through the Dialectic API.

**Key Integration Points Identified:**
1. `/chat` endpoint in `server/main.py` - primary location for message persistence
2. `AgentManager` - parallel HonchoClient for user/session management
3. Memory blocks - store Honcho insights in HumanBlock extensions
4. Pipeline metadata - user_id, chat_id, chat_title available for Honcho

---

## Detailed Findings

### 1. Current Codebase Architecture

#### 1.1 HTTP Service Structure

**FastAPI Application** (`src/letta_starter/server/main.py`)
- Application created at line 50 with title "LettaStarter Service"
- Lifespan handler at lines 29-48 manages startup/shutdown
- Creates `AgentManager` at line 34, stores in `app.state.agent_manager`
- Initializes `StrategyManager` singleton at line 35
- Strategy router mounted at `/strategy` prefix (line 58)

**Entry Point** (`src/letta_starter/server/cli.py`)
- `uv run letta-server` runs uvicorn on `127.0.0.1:8100`
- Module: `letta_starter.server.main:app`

**Settings** (`src/letta_starter/config/settings.py`)
- `ServiceSettings` class with env prefix `YOULAB_SERVICE_`
- Letta connection: `letta_base_url` default `http://localhost:8283`
- Langfuse tracing: configurable via `langfuse_enabled`, keys, host

#### 1.2 Agent Management

**AgentManager** (`src/letta_starter/server/agents.py:15-304`)

- **Naming convention**: `youlab_{user_id}_{agent_type}` (line 38)
- **Cache**: `_cache[(user_id, agent_type)] -> agent_id` (line 22)
- **Metadata stored**: `youlab_user_id`, `youlab_agent_type` (lines 40-45)

**Agent Creation** (lines 80-127):
```python
agent = self.client.agents.create(
    name=agent_name,
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-ada-002",
    memory_blocks=[
        {"label": "persona", "value": template.persona.to_memory_string()},
        {"label": "human", "value": human_block.to_memory_string()},
    ],
    metadata=metadata,
)
```

**Agent Templates** (`src/letta_starter/agents/templates.py`)
- `AgentTemplate` schema: type_id, display_name, description, persona, human
- `TUTOR_TEMPLATE` (lines 19-47): Pre-configured for college essay coaching
- Global registry: `templates = AgentTemplateRegistry()` (line 76)

#### 1.3 Chat Endpoint Flow

**Non-Streaming** (`/chat` at lines 151-206):
1. Verify agent exists via `get_agent_info()` (lines 157-162)
2. Create Langfuse trace context (lines 166-171)
3. Send message via `manager.send_message()` (line 182)
4. Return `ChatResponse` with response and agent_id

**Streaming** (`/chat/stream` at lines 209-234):
1. Verify agent exists (lines 215-220)
2. Return `StreamingResponse` with SSE generator
3. `stream_message()` calls Letta SDK streaming (lines 186-196)

**SSE Events Generated** (lines 201-249):
- `status`: Thinking/tool use indicators
- `message`: Assistant response content
- `done`: Completion signal
- `error`: Error messages

#### 1.4 Memory System

**Memory Blocks** (`src/letta_starter/memory/blocks.py`)

**PersonaBlock** (lines 28-132):
- Fields: name, role, capabilities, tone, verbosity, constraints, expertise
- Serializes to structured format with `[IDENTITY]`, `[CAPABILITIES]`, `[STYLE]`, etc.
- Default max_chars: 1500

**HumanBlock** (lines 134-274):
- Fields: name, role, current_task, session_state, preferences, context_notes, facts
- Serializes to `[USER]`, `[TASK]`, `[STATE]`, `[PREFS]`, `[FACTS]`, `[CONTEXT]`
- Helper methods: `add_context_note()`, `add_preference()`, `add_fact()`, `set_task()`, `clear_task()`

**Memory Strategies** (`src/letta_starter/memory/strategies.py`)

| Strategy | Threshold | Behavior |
|----------|-----------|----------|
| AggressiveRotation | 0.7 | Truncates to recent content |
| PreservativeRotation | 0.9 | Preserves structure, priority lines |
| AdaptiveRotation (default) | 0.8 | Learns from rotation patterns |

**MemoryManager** (`src/letta_starter/memory/manager.py`)
- Orchestrates block reading, updating, rotation, archival
- `update_human()` checks rotation and persists to Letta (lines 140-174)
- `_rotate_human_to_archival()` creates timestamped archival entries (lines 225-250)
- `search_archival()` queries archival memory (lines 271-292)

#### 1.5 Pipeline Integration

**OpenWebUI Pipe** (`src/letta_starter/pipelines/letta_pipe.py`)

**Data Available from OpenWebUI**:
- `__user__["id"]`: User identifier string
- `__user__["name"]`: User display name
- `__metadata__["chat_id"]`: Conversation identifier
- `chat.title`: Via `Chats.get_chat_by_id()` internal API

**Pipe → HTTP Service Flow**:
1. Extract user context (`user_id`, `user_name`, `chat_id`, `chat_title`)
2. Ensure agent exists via `GET /agents?user_id=X` then `POST /agents` if needed
3. Stream to `POST /chat/stream` with full context
4. Translate SSE events to OpenWebUI format

---

### 2. Honcho SDK Capabilities

#### 2.1 Installation & Versions

```bash
pip install honcho-ai
```

**Latest Versions (2025)**:
- v2.3.2 (September 2025): 10x faster, peer cards API
- v1.0.0 (June 2025): First stable release

**Requirements**: Python >= 3.9

#### 2.2 Client Initialization

```python
from honcho import Honcho

# Production setup
honcho = Honcho(
    workspace_id="youlab",
    api_key=os.environ["HONCHO_API_KEY"],
    environment="production",
    timeout=30.0,
    max_retries=2,
)
```

**Environment Variables**:
- `HONCHO_API_KEY`: Required for production
- `HONCHO_BASE_URL`: Default `https://demo.honcho.dev` (demo only)

**Async Client**:
```python
from honcho import AsyncHoncho

async_client = AsyncHoncho(
    workspace_id="youlab",
    api_key=os.environ["HONCHO_API_KEY"],
)
```

#### 2.3 Data Model

```
Workspace
├── Peers (users, AI agents)
├── Sessions (conversations)
│   ├── Messages
│   └── Metamessages
└── Collections
    └── Documents
```

#### 2.4 Core Operations

**Peers (Users/Agents)**:
```python
# Lazy creation - no API call until first use
alice = honcho.peer("student_123")
tutor = honcho.peer("tutor")

# With configuration
alice = honcho.peer(
    "student_123",
    config={"role": "student"},
    metadata={"course": "college-essay"}
)
```

**Sessions**:
```python
# One continuous session per student (per plan)
session = honcho.session(f"student_{user_id}")
session.add_peers([alice, tutor])
```

**Messages**:
```python
session.add_messages([
    alice.message(user_input, metadata={"chat_id": chat_id, "module": "1"}),
    tutor.message(response, metadata={"chat_id": chat_id})
])
```

#### 2.5 Dialectic API (Theory-of-Mind Queries)

**Purpose**: Natural language queries about users for personalization.

```python
# Query Honcho about the student
learning_style = alice.chat("What learning style works best for this student?")
engagement = alice.chat("How engaged is this student? What motivates them?")
```

**Latency**: Involves LLM inference (Claude 3.5 Sonnet). Use for quality-critical personalization.

**Example Response**:
> "Alice prefers visual explanations with concrete examples. She's more engaged when topics connect to her personal interests..."

#### 2.6 Working Representation (Low-Latency Alternative)

```python
# Get pre-computed insights (no LLM call - fast)
rep = alice.working_rep()

# With semantic search
rep = alice.working_rep(search_query="learning preferences", search_top_k=10)

# From session context
rep = session.working_rep("alice", target="bob")
```

**Use For**: Real-time context injection where dialectic latency is unacceptable.

#### 2.7 Context Retrieval

```python
# Get context with token limit
context = session.get_context(
    tokens=3000,
    summary=True,
    peer_target="student_123",
    peer_perspective="tutor"
)

# Convert to OpenAI format
messages = context.to_openai(assistant=tutor)
```

#### 2.8 Peer Cards (v2.3.0+)

Summarized peer information for quick access:
```python
card = client.workspaces.peers.get_card(
    workspace_id="youlab",
    peer_id="student_123"
)
# Returns: name, location, age, occupation, interests, likes/dislikes
```

#### 2.9 Error Handling

```python
import honcho

try:
    response = alice.chat("query")
except honcho.APIConnectionError:
    # Network issues (auto-retried)
    pass
except honcho.RateLimitError:
    # 429 - back off
    pass
except honcho.APIStatusError as e:
    # 4xx/5xx responses
    print(f"Status: {e.status_code}")
```

**Auto-retried errors**: Connection errors, 408, 409, 429, 500+

---

### 3. Integration Points

#### 3.1 Configuration Updates

**New Environment Variables** (`.env.example`):
```bash
HONCHO_WORKSPACE_ID=youlab
HONCHO_API_KEY=
HONCHO_ENVIRONMENT=production
```

**Settings Extension** (`config/settings.py`):
```python
# Add to ServiceSettings
honcho_workspace_id: str = Field(default="youlab")
honcho_api_key: str | None = Field(default=None)
honcho_environment: str = Field(default="production")
honcho_enabled: bool = Field(default=True)
```

#### 3.2 HonchoClient Module

**New File**: `src/letta_starter/honcho/client.py`

```python
from honcho import Honcho

class HonchoClient:
    """Manages Honcho peers and sessions for YouLab users."""

    def __init__(self, workspace_id: str, api_key: str, environment: str = "production"):
        self.honcho = Honcho(
            workspace_id=workspace_id,
            api_key=api_key,
            environment=environment,
        )
        self._tutor_peer = None

    @property
    def tutor(self):
        """Shared tutor peer for all conversations."""
        if self._tutor_peer is None:
            self._tutor_peer = self.honcho.peer("tutor")
        return self._tutor_peer

    def get_student_peer(self, user_id: str):
        """Get or create peer for a student."""
        return self.honcho.peer(f"student_{user_id}")

    def get_student_session(self, user_id: str):
        """Get continuous session for a student."""
        session = self.honcho.session(f"student_{user_id}")
        return session

    def persist_message(self, user_id: str, content: str, role: str, metadata: dict):
        """Persist a message to Honcho."""
        session = self.get_student_session(user_id)
        peer = self.get_student_peer(user_id) if role == "user" else self.tutor
        session.add_messages([peer.message(content, metadata=metadata)])

    def query_dialectic(self, user_id: str, question: str) -> str:
        """Query Honcho about a student."""
        peer = self.get_student_peer(user_id)
        return peer.chat(question)

    def get_working_rep(self, user_id: str, search_query: str | None = None) -> str:
        """Get low-latency working representation."""
        peer = self.get_student_peer(user_id)
        return peer.working_rep(search_query=search_query)
```

#### 3.3 Message Flow Integration

**Modified File**: `src/letta_starter/server/main.py`

In `/chat` endpoint (around line 182):
```python
# After getting agent_id and before sending to Letta
user_id = agent_info.get("user_id")

# 1. Persist user message to Honcho
if honcho_client:
    honcho_client.persist_message(
        user_id=user_id,
        content=message,
        role="user",
        metadata={"chat_id": chat_id, "chat_title": chat_title}
    )

# 2. Send to Letta (existing code)
response = await manager.send_message(agent_id, message)

# 3. Persist agent response to Honcho
if honcho_client:
    honcho_client.persist_message(
        user_id=user_id,
        content=response,
        role="assistant",
        metadata={"chat_id": chat_id, "chat_title": chat_title}
    )
```

#### 3.4 Memory Block Extensions

**Extended HumanBlock fields** (add to `blocks.py`):
```python
class HumanBlock(BaseModel):
    # ... existing fields ...

    # Phase 3: Honcho integration
    honcho_insights: str | None = Field(
        default=None,
        description="Latest insights from Honcho dialectic"
    )
    learning_style: str | None = Field(
        default=None,
        description="Student's preferred learning style"
    )
    engagement_patterns: str | None = Field(
        default=None,
        description="Observed engagement patterns"
    )
```

**Serialization update** (add to `to_memory_string()`):
```python
# Add after [CONTEXT] section
if self.honcho_insights:
    lines.append(f"[HONCHO] {self.honcho_insights}")
```

#### 3.5 Background Worker Integration (Phase 6)

The background worker will query Honcho dialectic and update memory blocks:

```python
# In background/worker.py
def process_student(self, user_id: str):
    # Query dialectic for insights
    insights = self.honcho_client.query_dialectic(
        user_id,
        "How is this student engaging? What learning style do they prefer?"
    )

    # Update Letta memory block
    human_block = self.memory_manager.get_human_block()
    human_block.honcho_insights = insights
    self.memory_manager.update_human(human_block)
```

---

### 4. Data Available for Honcho Persistence

| Field | Source | Honcho Destination |
|-------|--------|-------------------|
| `user_id` | `__user__["id"]` | Peer ID: `student_{user_id}` |
| `user_name` | `__user__["name"]` | Peer metadata |
| `chat_id` | `__metadata__["chat_id"]` | Message metadata |
| `chat_title` | `Chats.get_chat_by_id()` | Message metadata |
| `message` | Request body | Message content |
| `response` | Letta response | Message content (tutor peer) |
| `agent_type` | Configuration | Session metadata |

---

### 5. Architecture Decision Summary

From `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md`:

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Honcho model | One peer + one continuous session per student | Maintains conversation continuity |
| Thread handling | Visual in OpenWebUI; Letta sees continuous conversation | Simplified memory model |
| Peer naming | `student_{user_id}` | Matches Letta agent naming pattern |
| Message metadata | chat_id, module, lesson | Enable future thread context queries |

---

## Code References

### Server Architecture
- `src/letta_starter/server/main.py:50` - FastAPI app initialization
- `src/letta_starter/server/main.py:29-48` - Lifespan handler
- `src/letta_starter/server/main.py:151-206` - `/chat` endpoint
- `src/letta_starter/server/agents.py:80-127` - Agent creation

### Memory System
- `src/letta_starter/memory/blocks.py:28-132` - PersonaBlock
- `src/letta_starter/memory/blocks.py:134-274` - HumanBlock
- `src/letta_starter/memory/manager.py:140-174` - Memory update with rotation
- `src/letta_starter/memory/strategies.py:156-231` - AdaptiveRotation

### Pipeline
- `src/letta_starter/pipelines/letta_pipe.py:112-197` - Main pipe handler
- `src/letta_starter/pipelines/letta_pipe.py:127-146` - User context extraction

---

## Historical Context (from thoughts/)

- `thoughts/global/shared/reference/honcho-reference.md` - Comprehensive Honcho technical research (December 2025)
- `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Main technical foundation with Phase 3 spec
- `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` - Phase 1 implementation (foundation for Phase 3)
- `thoughts/shared/research/2025-12-31-phase-2-consolidated-reference.md` - Phase 2+ personalization context

---

## Related Research

- `thoughts/shared/research/2026-01-01-youlab-codebase-documentation.md` - Current codebase state
- `thoughts/shared/research/2026-01-01-technical-foundation-progress-analysis.md` - Implementation progress

---

## Open Questions

### Resolved by This Research

| Question | Resolution |
|----------|------------|
| Honcho SDK version | v2.3.2+ (September 2025), use `honcho-ai` package |
| Client initialization | Use `Honcho(workspace_id, api_key, environment)` |
| Async support | `AsyncHoncho` available for async contexts |
| Low-latency option | `working_rep()` method for real-time use |

### Still Open (from Technical Foundation Plan)

| Question | Notes |
|----------|-------|
| Workspace naming | `youlab` or `youlab-pilot`? |
| Peer naming | `student_{user_id}` seems consistent |
| Message metadata | chat_id, module, lesson suggested |
| Graceful degradation | How to handle Honcho unavailability? |
| Background worker queries | What specific dialectic queries to make? |

---

## Sources

### Official Honcho Documentation
- [Honcho Documentation](https://docs.honcho.dev/)
- [SDK Reference](https://docs.honcho.dev/v2/documentation/reference/sdk)
- [Features Overview](https://docs.honcho.dev/v2/documentation/core-concepts/features)

### GitHub Repositories
- [plastic-labs/honcho](https://github.com/plastic-labs/honcho)
- [plastic-labs/honcho-python-core](https://github.com/plastic-labs/honcho-python-core)

### Release Notes
- [Release Notes 07.17.25](https://blog.plasticlabs.ai/releases/Release-Notes-07.17.25)
- [Release Notes 08.14.25](https://blog.plasticlabs.ai/releases/Release-Notes-08.14.25)

### Blog Posts
- [Introducing Honcho's Dialectic API](https://blog.plasticlabs.ai/blog/Introducing-Honcho's-Dialectic-API)
- [A Simple Honcho Primer](https://blog.plasticlabs.ai/blog/A-Simple-Honcho-Primer)
