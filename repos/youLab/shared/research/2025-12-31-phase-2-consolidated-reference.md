# Phase 2+ Consolidated Reference Document

**Created**: 2025-12-31
**Purpose**: Primary reference for creating Phase 2/3/4 implementation plans
**Status**: Phase 1 complete, Phase 2 largely absorbed into Phase 1

---

## Table of Contents

1. [Current State Analysis](#1-current-state-analysis)
2. [Phase 2: Remaining Work](#2-phase-2-remaining-work)
3. [Phase 3: Honcho Integration](#3-phase-3-honcho-integration)
4. [Phase 4: Thread Context Management](#4-phase-4-thread-context-management)
5. [API References](#5-api-references)
6. [Architectural Decisions](#6-architectural-decisions)
7. [Open Questions](#7-open-questions)

---

## 1. Current State Analysis

### What's Implemented (Phase 1 Complete)

#### HTTP Service (`src/letta_starter/server/`)
| Component | File | Description |
|-----------|------|-------------|
| FastAPI App | `main.py` | Port 8100, endpoints for agents/chat/health |
| AgentManager | `agents.py` | Per-user agent lifecycle with in-memory cache |
| Schemas | `schemas.py` | Pydantic models for API requests/responses |
| Tracing | `tracing.py` | Langfuse integration with user attribution |

#### Pipeline Integration (`src/letta_starter/pipelines/`)
| Component | File | Description |
|-----------|------|-------------|
| LettaPipe | `letta_pipe.py` | OpenWebUI Pipe that forwards to HTTP service |

#### Core Endpoints
```
POST /agents        → Create agent for user
GET  /agents        → List agents (optional user_id filter)
GET  /agents/{id}   → Get agent details
POST /chat          → Send message to agent
GET  /health        → Health check (includes Letta status)
```

### User Identity Flow (Already Working)

```
OpenWebUI injects:  __user__["id"], __user__["name"]
                    __metadata__["chat_id"]

Pipe extracts:      user_id, user_name, chat_id
                    chat_title (via Chats.get_chat_by_id())

Service receives:   CreateAgentRequest or ChatRequest

AgentManager:       Cache lookup → Letta query → Create if needed
```

### Agent Naming Convention
```python
# Pattern: youlab_{user_id}_{agent_type}
def _agent_name(self, user_id: str, agent_type: str) -> str:
    return f"youlab_{user_id}_{agent_type}"

# Metadata stored on Letta agent
metadata = {
    "youlab_user_id": user_id,
    "youlab_agent_type": agent_type,
}
```

### Cache Strategy
- **Type**: `dict[tuple[str, str], str]` mapping `(user_id, agent_type)` → `agent_id`
- **Rebuild**: On startup via `rebuild_cache()` - scans all Letta agents with `youlab_` prefix
- **Lookup**: Check cache first, then query Letta by name if cache miss

---

## 2. Phase 2: Remaining Work

> **Note**: Most Phase 2 work was absorbed into Phase 1. Only minor additions remain.

### 2.1 First-Interaction Detection

**Purpose**: Trigger onboarding flow for new students

**Location**: `src/letta_starter/server/main.py` in `/chat` endpoint

**Proposed Implementation**:
```python
# In /chat endpoint, after agent lookup
async def is_first_interaction(agent_id: str) -> bool:
    # Option A: Check message count in Letta agent
    # Option B: Check for presence of specific memory blocks
    # Option C: Metadata flag on agent
    pass

if await is_first_interaction(agent_id):
    # Trigger onboarding context (Phase 7 will implement handler)
    pass
```

**Decision Needed**: Which detection method to use?

### 2.2 Memory Block Schema Extensions

**Defer to Phases 4/5** — Don't extend upfront.

**Future HumanBlock fields**:
- `[PROGRESS]` — Module/lesson completion status
- `[STRENGTHS]` — Clifton strengths (when available)
- `[HONCHO_INSIGHTS]` — Insights from dialectic (Phase 6)

**Future PersonaBlock fields**:
- `[MODULE]` — Current module context
- `[LESSON]` — Current lesson context
- `[OBJECTIVE]` — Current lesson objective

### 2.3 Success Criteria

- [ ] Phase 1 tests pass (agent creation, routing)
- [ ] First-interaction detection works
- [ ] Two different OpenWebUI users get different Letta agents
- [ ] Same user across sessions maintains agent continuity

---

## 3. Phase 3: Honcho Integration

### 3.1 Overview

Integrate Honcho for message persistence and theory-of-mind capabilities. Every message flows through Honcho. Dialectic queries inform agent behavior.

### 3.2 Honcho Data Model

```
Workspaces
├── Peers (students + tutor)
│   ├── Sessions (continuous per student)
│   ├── Observations (derived facts)
│   └── Peer Cards (structured profiles)
└── Messages (within sessions)
```

**Key Decisions**:
- **One peer per student** in Honcho
- **One continuous session** per student (not per chat/thread)
- **All messages persisted** for long-term ToM modeling

### 3.3 Honcho Client Implementation

**New File**: `src/letta_starter/honcho/client.py`

```python
from honcho import Honcho

class HonchoClient:
    def __init__(self, workspace_id: str, api_key: str):
        self.honcho = Honcho(
            workspace_id=workspace_id,
            api_key=api_key,
            environment="production"
        )
        self._tutor_peer = None

    @property
    def tutor(self):
        if not self._tutor_peer:
            self._tutor_peer = self.honcho.peer("tutor")
        return self._tutor_peer

    def get_or_create_peer(self, user_id: str):
        """Get or create a peer for a student."""
        return self.honcho.peer(f"student_{user_id}")

    def get_or_create_session(self, user_id: str):
        """Get or create a continuous session for a student."""
        session = self.honcho.session(f"student_{user_id}")
        # Ensure peers are added
        student = self.get_or_create_peer(user_id)
        session.add_peers([student, self.tutor])
        return session

    def add_message(
        self,
        user_id: str,
        content: str,
        role: str,
        metadata: dict | None = None
    ):
        """Add a message to the student's session."""
        session = self.get_or_create_session(user_id)
        peer = self.get_or_create_peer(user_id) if role == "user" else self.tutor
        session.add_messages([peer.message(content, metadata=metadata or {})])

    def query_dialectic(self, user_id: str, question: str) -> str:
        """Query Honcho's dialectic API about a student."""
        peer = self.get_or_create_peer(user_id)
        return peer.chat(question)

    def get_working_rep(self, user_id: str) -> str:
        """Get low-latency working representation of student."""
        peer = self.get_or_create_peer(user_id)
        return peer.working_rep()
```

### 3.4 Configuration

**Add to `.env.example`**:
```bash
HONCHO_WORKSPACE_ID=youlab
HONCHO_API_KEY=
HONCHO_ENVIRONMENT=production
```

**Add to `config/settings.py`**:
```python
class HonchoSettings(BaseSettings):
    workspace_id: str = Field(default="youlab", alias="HONCHO_WORKSPACE_ID")
    api_key: str = Field(default="", alias="HONCHO_API_KEY")
    environment: str = Field(default="production", alias="HONCHO_ENVIRONMENT")

    @property
    def enabled(self) -> bool:
        return bool(self.api_key)
```

### 3.5 Integration into Chat Flow

**Modify**: `src/letta_starter/server/main.py`

```python
@app.post("/chat")
async def chat(request: ChatRequest):
    # ... existing agent lookup ...

    # 1. Persist user message to Honcho
    if honcho_client.enabled:
        honcho_client.add_message(
            user_id=user_id,
            content=request.message,
            role="user",
            metadata={"chat_id": request.chat_id, "chat_title": request.chat_title}
        )

    # 2. Send to Letta (existing code)
    response = agent_manager.send_message(request.agent_id, request.message)

    # 3. Persist agent response to Honcho
    if honcho_client.enabled:
        honcho_client.add_message(
            user_id=user_id,
            content=response,
            role="assistant",
            metadata={"chat_id": request.chat_id}
        )

    return ChatResponse(response=response, agent_id=request.agent_id)
```

### 3.6 Dialectic Query Examples

```python
# Learning style
style = peer.chat("What learning style works best for this student?")

# Engagement patterns
engagement = peer.chat("How is this student engaging with the material?")

# Communication preferences
comm = peer.chat("How does this student prefer to receive feedback?")

# Predictions
prediction = peer.chat("How would this student likely respond to constructive criticism?")
```

### 3.7 Latency Considerations

| Method | Latency | Use Case |
|--------|---------|----------|
| `add_messages()` | Low | Every message (async persist) |
| `working_rep()` | Low | Real-time personalization |
| `peer.chat()` (Dialectic) | High (LLM) | Background enrichment, periodic queries |

**Recommendation**: Use `working_rep()` for real-time, `peer.chat()` for Phase 6 background worker.

### 3.8 Dependencies

**Add to `pyproject.toml`**:
```toml
dependencies = [
    # ... existing ...
    "honcho-ai>=0.1.0",
]
```

### 3.9 Success Criteria

- [ ] `uv run pytest tests/test_honcho.py` passes
- [ ] Messages appear in Honcho dashboard after conversation
- [ ] Dialectic queries return meaningful responses (after sufficient messages)
- [ ] Honcho connection failure doesn't crash service (graceful degradation)

---

## 4. Phase 4: Thread Context Management

### 4.1 Overview

Parse OpenWebUI chat titles to determine module/lesson context. Update Letta memory blocks accordingly. Handle "student went back to old chat" scenario.

### 4.2 Chat Title Access

**Already implemented** in `letta_pipe.py`:
```python
from open_webui.models.chats import Chats

chat = Chats.get_chat_by_id(chat_id)
title = chat.title if chat else None
```

**Edge Case**: Temporary chats have IDs starting with `local:` — not in database, skip title lookup.

### 4.3 Title Parser

**New File**: `src/letta_starter/context/parser.py`

```python
import re
from dataclasses import dataclass
from typing import Optional

@dataclass
class ThreadContext:
    module: Optional[str] = None
    lesson: Optional[str] = None
    is_side_conversation: bool = False
    raw_title: str = ""

# Possible title formats to support:
# - "Module 1 / Lesson 2"
# - "M1L2" or "M1 L2"
# - "Self-Discovery / Strengths Assessment"
# - Freeform titles (treated as side conversations)

def parse_chat_title(title: str) -> ThreadContext:
    """Parse chat title into structured context."""
    if not title:
        return ThreadContext(is_side_conversation=True, raw_title="")

    ctx = ThreadContext(raw_title=title)

    # Pattern 1: "Module X / Lesson Y"
    match = re.match(r"Module\s+(\d+)\s*/\s*Lesson\s+(\d+)", title, re.IGNORECASE)
    if match:
        ctx.module = f"module-{match.group(1)}"
        ctx.lesson = f"lesson-{match.group(2)}"
        return ctx

    # Pattern 2: "M1L2" or "M1 L2"
    match = re.match(r"M(\d+)\s*L(\d+)", title, re.IGNORECASE)
    if match:
        ctx.module = f"module-{match.group(1)}"
        ctx.lesson = f"lesson-{match.group(2)}"
        return ctx

    # Pattern 3: Named modules/lessons (from curriculum)
    # TODO: Match against loaded curriculum

    # Default: side conversation
    ctx.is_side_conversation = True
    return ctx
```

### 4.4 Context Cache

**New File**: `src/letta_starter/context/cache.py`

```python
from dataclasses import dataclass
from typing import Optional
import time

@dataclass
class CachedContext:
    context: ThreadContext
    cached_at: float
    ttl: float = 300.0  # 5 minutes

    @property
    def is_expired(self) -> bool:
        return time.time() - self.cached_at > self.ttl

class ContextCache:
    def __init__(self):
        self._cache: dict[str, CachedContext] = {}

    def get(self, chat_id: str) -> Optional[ThreadContext]:
        cached = self._cache.get(chat_id)
        if cached and not cached.is_expired:
            return cached.context
        return None

    def set(self, chat_id: str, context: ThreadContext):
        self._cache[chat_id] = CachedContext(
            context=context,
            cached_at=time.time()
        )

    def invalidate(self, chat_id: str):
        self._cache.pop(chat_id, None)
```

### 4.5 Memory Block Updater

**New File**: `src/letta_starter/context/updater.py`

```python
class ContextUpdater:
    def __init__(self, letta_client, curriculum_loader=None):
        self.letta = letta_client
        self.curriculum = curriculum_loader
        self._last_context: dict[str, ThreadContext] = {}  # agent_id -> last context

    def update_agent_context(
        self,
        agent_id: str,
        new_context: ThreadContext,
        user_id: str
    ) -> bool:
        """Update agent memory blocks if context changed. Returns True if updated."""
        last = self._last_context.get(agent_id)

        if last and last.module == new_context.module and last.lesson == new_context.lesson:
            return False  # No change

        # Context changed - update memory blocks
        updates = {}

        if new_context.module:
            updates["[MODULE]"] = new_context.module
            # Load module-specific instructions from curriculum
            if self.curriculum:
                module_data = self.curriculum.get_module(new_context.module)
                if module_data:
                    updates["module_instructions"] = module_data.agent_instructions

        if new_context.lesson:
            updates["[LESSON]"] = new_context.lesson
            if self.curriculum:
                lesson_data = self.curriculum.get_lesson(new_context.lesson)
                if lesson_data:
                    updates["lesson_instructions"] = lesson_data.agent_instructions
                    updates["[OBJECTIVE]"] = "\n".join(lesson_data.objectives)

        # Detect "returning to old chat"
        if last:
            updates["[CURRENT_THREAD]"] = f"Student returned to: {new_context.raw_title}"

        # Update Letta agent memory blocks
        # TODO: Implement memory block update via Letta API

        self._last_context[agent_id] = new_context
        return True
```

### 4.6 Integration Point

**Modify**: `src/letta_starter/server/main.py`

```python
@app.post("/chat")
async def chat(request: ChatRequest):
    # ... existing agent lookup ...

    # Parse thread context from chat title
    if request.chat_title:
        context = context_cache.get(request.chat_id)
        if not context:
            context = parse_chat_title(request.chat_title)
            context_cache.set(request.chat_id, context)

        # Update agent memory if context changed
        context_updater.update_agent_context(
            agent_id=request.agent_id,
            new_context=context,
            user_id=user_id
        )

    # ... rest of chat flow ...
```

### 4.7 Success Criteria

- [ ] Parser correctly extracts module/lesson from various title formats
- [ ] Cache returns consistent results for same chat_id
- [ ] Memory blocks updated when context changes
- [ ] `uv run pytest tests/test_context.py` passes
- [ ] Create chat titled "Module 1 / Lesson 2", verify agent knows context
- [ ] Navigate to different chat, verify agent context switches

---

## 5. API References

### 5.1 OpenWebUI Pipe Interface

```python
async def pipe(
    self,
    body: dict,
    __user__: dict = None,      # {"id", "email", "name", "role"}
    __metadata__: dict = None,  # {"chat_id", "message_id", "session_id", "user_id", ...}
    __request__: Request = None,
    __event_emitter__: Callable = None,
) -> str:
```

**What's Available Directly**:
| Field | Location | Available |
|-------|----------|-----------|
| user_id | `__user__["id"]` | Yes |
| user_name | `__user__["name"]` | Yes |
| chat_id | `__metadata__["chat_id"]` | Yes |
| message_id | `__metadata__["message_id"]` | Yes |
| chat_title | Must query: `Chats.get_chat_by_id(chat_id).title` | Via DB |
| folder_id | Not exposed | No |

### 5.2 Honcho SDK

```python
from honcho import Honcho

# Initialize
honcho = Honcho(
    workspace_id="youlab",
    api_key="...",
    environment="production"
)

# Peers
student = honcho.peer("student-123")
tutor = honcho.peer("tutor")

# Sessions
session = honcho.session("student-123")
session.add_peers([student, tutor])

# Messages
session.add_messages([
    student.message("Hello", metadata={"chat_id": "..."}),
    tutor.message("Hi there!")
])

# Dialectic (high latency, LLM-based)
insight = student.chat("What learning style works best?")

# Working representation (low latency)
rep = student.working_rep()

# Context for LLM
context = session.get_context(tokens=2000, summary=True)
openai_msgs = context.to_openai(assistant=tutor)
```

### 5.3 Letta Agent API

```python
from letta_client import Letta

client = Letta(base_url="http://localhost:8283")

# Create agent
agent = client.agents.create(
    name="youlab_user123_tutor",
    model="anthropic/claude-sonnet-4-20250514",
    memory_blocks=[
        {"label": "persona", "value": "..."},
        {"label": "human", "value": "..."}
    ],
    metadata={"youlab_user_id": "user123", "youlab_agent_type": "tutor"}
)

# Send message
response = client.agents.messages.send(
    agent_id=agent.id,
    messages=[{"role": "user", "content": "Hello"}]
)

# Update memory block
client.agents.memory.update_block(
    agent_id=agent.id,
    block_label="persona",
    value="Updated persona..."
)

# Archival memory
client.agents.passages.insert(
    agent_id=agent.id,
    content="Important fact to remember",
    tags=["student_info"]
)

results = client.agents.passages.search(
    agent_id=agent.id,
    query="student preferences"
)
```

---

## 6. Architectural Decisions

### 6.1 Resolved Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| API Framework | FastAPI | Modern, async, Pydantic integration |
| Port | 8100 | Separate from Letta (8283) |
| Authentication | Trust localhost | Simpler for pilot, API key designed for later |
| Agent Creation | Explicit `POST /agents` | Clear lifecycle, supports multiple agent types |
| Agent Naming | `youlab_{user_id}_{agent_type}` | Discoverable, supports multiple types per user |
| Chat Title Access | `Chats.get_chat_by_id()` in Pipe | Confirmed available |
| Honcho Model | One peer + one continuous session per student | Simpler than per-chat sessions |
| Thread Handling | Visual in OpenWebUI; Letta sees continuous conversation | Matches Honcho model |

### 6.2 Phase Dependencies

```
Phase 1: HTTP Service ✅ COMPLETE
    ↓
Phase 2: User Identity & Routing (minimal remaining work)
    ↓
    ├── Phase 3: Honcho Integration ←── CAN START NOW
    │       ↓
    │   Phase 6: Background Worker
    │
    └── Phase 4: Thread Context ←── CAN PARALLEL WITH PHASE 3
            ↓
        Phase 5: Curriculum Parser
            ↓
        Phase 7: Onboarding
```

---

## 7. Open Questions

### Phase 3 (Honcho)

| Question | Options | Recommendation |
|----------|---------|----------------|
| Workspace naming | `youlab` vs `youlab-pilot` | `youlab` (simple) |
| Peer naming | Same as Letta agent name? | `student_{user_id}` (simpler) |
| Message metadata | What to attach? | `chat_id`, `chat_title`, `module`, `lesson` |

### Phase 4 (Thread Context)

| Question | Options | Recommendation |
|----------|---------|----------------|
| Chat title format | `Module 1 / Lesson 2` vs `M1L2` vs freeform | Support multiple patterns |
| Fallback for unknown titles | Infer from progress? Ask student? | Treat as side conversation |
| Cache TTL | How long? | 5 minutes (configurable) |

### Phase 5 (Curriculum)

| Question | Status |
|----------|--------|
| Final markdown format | Not finalized |
| Trigger expression syntax | Not decided |
| Completion criteria evaluation | Not decided |

### Phase 6 (Background Worker)

| Question | Status |
|----------|--------|
| Idle timeout duration | 10-30 min? Not decided |
| Specific dialectic queries | Not finalized |
| Progress/completion detection | Not decided |

---

## File Locations Summary

### Existing Files
- `src/letta_starter/server/main.py` — FastAPI app
- `src/letta_starter/server/agents.py` — AgentManager
- `src/letta_starter/server/schemas.py` — Request/response models
- `src/letta_starter/server/tracing.py` — Langfuse integration
- `src/letta_starter/pipelines/letta_pipe.py` — OpenWebUI Pipe
- `src/letta_starter/config/settings.py` — Pydantic settings

### New Files for Phase 3
- `src/letta_starter/honcho/__init__.py`
- `src/letta_starter/honcho/client.py` — Honcho wrapper

### New Files for Phase 4
- `src/letta_starter/context/__init__.py`
- `src/letta_starter/context/parser.py` — Title parser
- `src/letta_starter/context/cache.py` — Context cache
- `src/letta_starter/context/updater.py` — Memory block updater

---

## Reference Documents

| Document | Path |
|----------|------|
| Master Plan | `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` |
| Phase 1 Plan | `thoughts/shared/plans/2025-12-29-phase-1-http-service.md` |
| OpenWebUI Pipes | `thoughts/global/shared/reference/open-web-ui-pipes.md` |
| Honcho API | `thoughts/global/shared/reference/honcho-reference.md` |
| Letta Memory | `thoughts/global/shared/reference/letta-archival-memory.md` |
| Project Context | `thoughts/shared/youlab-project-context.md` |
