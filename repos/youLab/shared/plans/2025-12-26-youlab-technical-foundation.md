# YouLab Technical Foundation Implementation Plan

## Overview

Build the complete technical infrastructure for YouLab's AI tutoring platform. This establishes the foundation by which courses are delivered through personalized AI agents with long-term memory and theory-of-mind capabilities.

**Target**: Robust, modular system ready for prompt tuning and course iteration—not a half-baked prototype.

## Current State Analysis

### What Exists
- LettaStarter framework with BaseAgent, memory blocks, rotation strategies
- OpenWebUI Pipe (single shared agent, no user identity)
- Memory serialization (PersonaBlock/HumanBlock)
- Observability infrastructure (Langfuse, structlog)
- Basic test suite

### What's Missing
- Per-student agent routing
- User identity extraction from OpenWebUI
- Honcho integration (message persistence, dialectic)
- Curriculum parser and hot-reload
- Thread context management
- Background/sleep agent process
- Student onboarding flow

### Key Architectural Decisions Made
| Decision | Choice |
|----------|--------|
| Agent model | One Letta agent per student |
| Honcho model | One peer + one continuous session per student |
| Thread handling | Visual in OpenWebUI; Letta sees continuous conversation |
| Pipe architecture | Thin pipe → LettaStarter HTTP service |
| Thread→context mapping | Parse chat title once, cache |
| Curriculum state | Core memory blocks (not external DB) |
| Background trigger | Idle timeout + manual endpoint |
| Curriculum hot-reload | Polling (check file mtimes) |

## Desired End State

A working system where:
1. Students log into OpenWebUI, each routed to their personal Letta agent
2. All messages persisted to Honcho for long-term ToM modeling
3. Agent context adapts based on which chat/module student is in
4. Curriculum defined in markdown, hot-reloadable without redeploy
5. Background process periodically enriches agent memory from Honcho insights
6. New students smoothly onboarded with initial setup flow

### Verification
- Create test student account in OpenWebUI
- Send messages across multiple chats (different modules)
- Verify Letta agent maintains continuity but adapts context
- Verify Honcho receives all messages
- Query Honcho dialectic and see insights reflected in agent behavior
- Modify curriculum markdown, verify agent picks up changes
- Trigger background process, verify memory updates

## What We're NOT Doing

- Full user management UI (admin dashboard, roles, permissions)
- Multi-facilitator support (just single facilitator for pilot)
- Course marketplace or multi-course support (one course for now)
- Production deployment infrastructure (staying local for pilot)
- Automated testing of pedagogical effectiveness
- Mobile-specific optimizations

---

## Phase 1: LettaStarter HTTP Service

### Overview
Convert LettaStarter from a library to an HTTP service. The Pipe will call this service rather than importing LettaStarter directly. This keeps the Pipe thin and all logic server-side.

### Open Questions to Resolve Before Implementation
- **API Framework**: FastAPI (recommended) or Flask?
- **Port/host configuration**: Environment variables?
- **Authentication**: API key between Pipe and service, or trust localhost?

### Changes Required

#### 1. Create HTTP Service Entry Point
**File**: `src/letta_starter/server.py` (new)

Core endpoints:
```
POST /chat
  Body: { user_id, message, chat_id, chat_title? }
  Returns: { response, agent_id }

GET /health
  Returns: { status, letta_connected, honcho_connected }

GET /student/{user_id}/status
  Returns: { agent_exists, current_context, progress }

POST /admin/reload-curriculum
  Triggers curriculum reload

POST /background/run
  Body: { user_id? }
  Triggers background processing
```

#### 2. Add Dependencies
**File**: `pyproject.toml`

Add FastAPI, uvicorn to dependencies.

#### 3. Update CLI
**File**: `src/letta_starter/main.py`

Add `--serve` mode that starts HTTP server instead of interactive CLI.

### Success Criteria

#### Automated Verification
- [ ] `uv run letta-starter --serve` starts HTTP server
- [ ] `curl localhost:8000/health` returns 200 with status
- [ ] `uv run pytest tests/test_server.py` passes
- [ ] `uv run ruff check src/` passes
- [ ] `uv run mypy src/` passes

#### Manual Verification
- [ ] Can POST to /chat endpoint and receive response
- [ ] Server logs show structured output
- [ ] Server survives Letta being temporarily unavailable

---

## Phase 2: User Identity & Agent Routing

### Overview
Update the Pipe to extract user identity from OpenWebUI and route to per-student Letta agents. Create agents on first interaction.

### Open Questions to Resolve Before Implementation
- **Agent naming convention**: `student_{user_id}` or `youlab_{user_id}`?
- **What to do if agent creation fails**: Retry? Fallback? Error to user?

### Changes Required

#### 1. Update Pipe to Extract User Info
**File**: `src/letta_starter/pipelines/letta_pipe.py`

Update `pipe()` signature to accept special arguments:
```python
def pipe(
    self,
    body: dict,
    __user__: dict = None,
    __metadata__: dict = None,
) -> str:
    user_id = __user__["id"]
    chat_id = __metadata__.get("chat_id")
    # Forward to LettaStarter service
```

#### 2. Implement Agent Registry in Service
**File**: `src/letta_starter/server.py`

- On `/chat` request, look up or create agent for user_id
- Cache agent_id → user_id mapping
- Create new agent with initial memory blocks if first interaction

#### 3. Define Initial Memory Block Templates
**File**: `src/letta_starter/memory/templates.py` (new)

Templates for new student agents:
- Initial persona (course tutor identity)
- Initial human block (empty student profile)

### Open Questions: Memory Block Schema

**To be finalized before implementation:**

PersonaBlock fields:
```
[IDENTITY] - Agent name and role
[CAPABILITIES] - What the agent can do
[STYLE] - Communication style (may be updated by Honcho)
[MODULE] - Current module context
[LESSON] - Current lesson context
[OBJECTIVE] - Current lesson objective
```

HumanBlock fields:
```
[USER] - Student name and basic info
[PROGRESS] - Module/lesson completion status
[STRENGTHS] - Clifton strengths (when available)
[TASK] - Current task/activity
[CONTEXT] - Recent context notes
[HONCHO_INSIGHTS] - Insights from dialectic (updated by background process)
```

### Success Criteria

#### Automated Verification
- [ ] Pipe correctly extracts `__user__["id"]` and `__metadata__["chat_id"]`
- [ ] Service creates new agent on first request from user
- [ ] Service reuses existing agent on subsequent requests
- [ ] `uv run pytest tests/test_routing.py` passes

#### Manual Verification
- [ ] Two different OpenWebUI users get different Letta agents
- [ ] Same user across sessions maintains agent continuity
- [ ] Agent memory persists between interactions

---

## Phase 3: Honcho Integration

### Overview
Integrate Honcho for message persistence and theory-of-mind capabilities. Every message flows through Honcho. Dialectic queries inform agent behavior.

### Prerequisites
- Plastic Labs hosted Honcho account
- API key generated
- Workspace created

### Open Questions to Resolve Before Implementation
- **Workspace naming**: `youlab` or `youlab-pilot`?
- **Peer naming**: Same as Letta agent name?
- **What metadata to attach to messages**: chat_id, module, lesson?

### Changes Required

#### 1. Add Honcho Client
**File**: `src/letta_starter/honcho/client.py` (new)

```python
from honcho import Honcho

class HonchoClient:
    def __init__(self, workspace_id: str, api_key: str):
        self.honcho = Honcho(
            workspace_id=workspace_id,
            api_key=api_key,
            environment="production"
        )

    def get_or_create_peer(self, user_id: str) -> Peer:
        return self.honcho.peer(f"student_{user_id}")

    def get_or_create_session(self, user_id: str) -> Session:
        session = self.honcho.session(f"student_{user_id}")
        # Add student and tutor peers
        return session

    def add_message(self, user_id: str, content: str, role: str, metadata: dict):
        session = self.get_or_create_session(user_id)
        peer = self.get_or_create_peer(user_id) if role == "user" else self.honcho.peer("tutor")
        session.add_messages([peer.message(content, metadata=metadata)])

    def query_dialectic(self, user_id: str, question: str) -> str:
        peer = self.get_or_create_peer(user_id)
        return peer.chat(question)

    def get_working_rep(self, user_id: str) -> str:
        peer = self.get_or_create_peer(user_id)
        return peer.working_rep()
```

#### 2. Add Honcho Config
**File**: `.env.example`

Add:
```
HONCHO_WORKSPACE_ID=youlab
HONCHO_API_KEY=
HONCHO_ENVIRONMENT=production
```

#### 3. Integrate into Message Flow
**File**: `src/letta_starter/server.py`

In `/chat` endpoint:
1. Persist user message to Honcho
2. Send to Letta
3. Persist agent response to Honcho

#### 4. Add Dependencies
**File**: `pyproject.toml`

Add `honcho-ai` to dependencies.

### Success Criteria

#### Automated Verification
- [ ] `uv run pytest tests/test_honcho.py` passes
- [ ] Messages appear in Honcho dashboard after conversation
- [ ] Dialectic queries return meaningful responses (after sufficient messages)

#### Manual Verification
- [ ] Send several messages, verify they appear in Honcho
- [ ] Query dialectic about student, see synthesized insights
- [ ] Honcho connection failure doesn't crash the service (graceful degradation)

---

## Phase 4: Thread Context Management

### Overview
Parse OpenWebUI chat titles to determine module/lesson context. Update Letta memory blocks accordingly. Handle the "student went back to old chat" scenario.

### Open Questions to Resolve Before Implementation
- **Chat title format**: `Module 1 / Lesson 2` or `M1L2` or freeform?
- **Fallback when title doesn't match pattern**: Infer from progress? Ask student?
- **Cache invalidation**: When does cached title→context mapping expire?

### Changes Required

#### 1. Title Parser
**File**: `src/letta_starter/context/parser.py` (new)

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

def parse_chat_title(title: str) -> ThreadContext:
    # Parse patterns like "Module 1 / Lesson 2" or "Side Chat"
    # Return structured context
    ...
```

#### 2. Context Cache
**File**: `src/letta_starter/context/cache.py` (new)

- Map chat_id → ThreadContext
- Query OpenWebUI for title on first encounter (using internal Chats model)
- Cache result

#### 3. Memory Block Updater
**File**: `src/letta_starter/context/updater.py` (new)

- Given ThreadContext, update agent's memory blocks
- Add `[CURRENT_THREAD]` note when navigating to old thread
- Load module/lesson specific instructions from curriculum

#### 4. Integrate into Chat Flow
**File**: `src/letta_starter/server.py`

Before sending to Letta:
1. Parse/lookup thread context
2. Update memory blocks if context changed
3. Proceed with message

### Success Criteria

#### Automated Verification
- [ ] Parser correctly extracts module/lesson from various title formats
- [ ] Cache returns consistent results for same chat_id
- [ ] Memory blocks updated when context changes
- [ ] `uv run pytest tests/test_context.py` passes

#### Manual Verification
- [ ] Create chat titled "Module 1 / Lesson 2", verify agent knows context
- [ ] Navigate to different chat, verify agent context switches
- [ ] Navigate back to old chat, verify agent acknowledges "returning"

---

## Phase 5: Curriculum Parser & Hot-Reload

### Overview
Load course definitions from markdown files. Parse into in-memory structures. Detect file changes and reload automatically.

### Open Questions to Resolve Before Implementation

**Curriculum format** (needs finalization):

```markdown
# courses/college-essay/course.md
---
name: College Essay Mastery
version: 1.0
modules:
  - module-1-discovery
  - module-2-story-mining
---

# courses/college-essay/module-1-discovery.md
---
name: Self-Discovery
lessons:
  - strengths-assessment
  - processing-results
memory_blocks:
  - clifton_strengths
---

## Lesson: strengths-assessment
trigger: module_start
objectives:
  - Complete Clifton StrengthsFinder
  - Initial reaction conversation

### Agent Instructions
[Instructions for agent behavior during this lesson]

### Completion Criteria
- Articulated one resonant strength
- Named one thing assessment missed
- Minimum 3 turns
```

**Questions:**
- How are triggers evaluated? Simple strings vs expressions?
- How is completion judged? LLM-judged vs explicit vs turn count?
- How do memory block references resolve at runtime?

### Changes Required

#### 1. Curriculum Data Models
**File**: `src/letta_starter/curriculum/models.py` (new)

```python
@dataclass
class Lesson:
    id: str
    name: str
    trigger: str
    objectives: list[str]
    agent_instructions: str
    completion_criteria: list[str]

@dataclass
class Module:
    id: str
    name: str
    lessons: list[Lesson]
    memory_blocks: list[str]

@dataclass
class Course:
    name: str
    version: str
    modules: list[Module]
```

#### 2. Markdown Parser
**File**: `src/letta_starter/curriculum/parser.py` (new)

- Parse YAML frontmatter
- Parse lesson sections (## Lesson: name)
- Extract agent instructions and completion criteria
- Return Course object

#### 3. Curriculum Loader
**File**: `src/letta_starter/curriculum/loader.py` (new)

- Load course from directory
- Watch for file changes (polling)
- Reload on change
- Expose `get_current_course()`, `get_module(id)`, `get_lesson(id)`

#### 4. Integration Point
**File**: `src/letta_starter/context/updater.py`

When updating context, pull agent instructions from curriculum loader.

### Success Criteria

#### Automated Verification
- [ ] Parser correctly loads sample course markdown
- [ ] Lesson objects have all expected fields
- [ ] File modification triggers reload
- [ ] `uv run pytest tests/test_curriculum.py` passes

#### Manual Verification
- [ ] Create courses/ directory with sample course
- [ ] Start service, verify course loaded
- [ ] Modify markdown file, verify service detects change
- [ ] Agent behavior reflects loaded curriculum

---

## Phase 6: Background/Sleep Agent Process

### Overview
Implement background process that queries Honcho dialectic and updates Letta memory blocks. Runs on idle timeout or manual trigger.

### Open Questions to Resolve Before Implementation
- **Idle timeout duration**: 10 minutes? 30 minutes?
- **What dialectic queries to make**: Learning style? Engagement? Personality?
- **Which memory blocks to update**: Just HONCHO_INSIGHTS? Also STYLE in persona?
- **Progress summarization**: How to detect lesson completion?

### Changes Required

#### 1. Background Worker
**File**: `src/letta_starter/background/worker.py` (new)

```python
class BackgroundWorker:
    def __init__(self, honcho_client, letta_client):
        ...

    def process_student(self, user_id: str):
        # 1. Query dialectic for insights
        insights = self.honcho.query_dialectic(
            user_id,
            "How is this student engaging? What's their learning style?"
        )

        # 2. Update Letta memory blocks
        self.update_honcho_insights(user_id, insights)

        # 3. Optionally update persona style
        style_suggestion = self.honcho.query_dialectic(
            user_id,
            "How should I adjust my communication style for this student?"
        )
        self.update_persona_style(user_id, style_suggestion)

        # 4. Update progress if needed
        self.update_progress_summary(user_id)
```

#### 2. Idle Detection
**File**: `src/letta_starter/background/scheduler.py` (new)

- Track last message timestamp per user
- Trigger worker after idle timeout
- Prevent duplicate runs

#### 3. Manual Trigger Endpoint
**File**: `src/letta_starter/server.py`

`POST /background/run` endpoint (already defined in Phase 1).

### Success Criteria

#### Automated Verification
- [ ] Worker successfully queries Honcho dialectic
- [ ] Memory blocks updated after worker runs
- [ ] Idle detection triggers worker after timeout
- [ ] `uv run pytest tests/test_background.py` passes

#### Manual Verification
- [ ] Have conversation, wait for idle timeout
- [ ] Verify HONCHO_INSIGHTS block updated
- [ ] Manually trigger via endpoint, verify processing
- [ ] Next conversation reflects updated insights

---

## Phase 7: Student Onboarding

### Overview
Handle new student first-time experience. Create accounts, initialize agents, guide through setup.

### Open Questions to Resolve Before Implementation
- **What data to collect initially**: Name only? Goals? Background?
- **Clifton StrengthsFinder integration**: Link out? Import results?
- **Group/facilitator assignment**: Manual for pilot?
- **Scripted setup vs conversational**: Pre-defined flow vs agent-led?

### Changes Required

#### 1. New Student Detection
**File**: `src/letta_starter/server.py`

In `/chat` endpoint:
- Check if agent exists for user_id
- If not, create with onboarding context
- Set initial lesson to onboarding flow

#### 2. Onboarding Module
**File**: `courses/college-essay/module-0-onboarding.md` (new)

Define onboarding as Module 0 with lessons:
- Introduction and welcome
- Collect basic info (name, goals)
- Explain how the system works
- Transition to Module 1

#### 3. Student Profile Initialization
**File**: `src/letta_starter/onboarding/setup.py` (new)

- Create Letta agent with onboarding persona
- Create Honcho peer
- Initialize empty progress tracking
- Set context to Module 0

### Success Criteria

#### Automated Verification
- [ ] New user_id triggers agent creation
- [ ] Agent starts with onboarding context
- [ ] Progress initialized to Module 0
- [ ] `uv run pytest tests/test_onboarding.py` passes

#### Manual Verification
- [ ] Log in as new user, receive welcome message
- [ ] Complete onboarding flow
- [ ] Verify transition to Module 1
- [ ] Profile data persisted correctly

---

## Phase Ordering & Dependencies

```
Phase 1: HTTP Service (foundation)
    ↓
Phase 2: User Identity & Routing (depends on Phase 1)
    ↓
    ├── Phase 3: Honcho Integration (can parallel with Phase 4)
    │       ↓
    │   Phase 6: Background Worker (depends on Phase 3)
    │
    └── Phase 4: Thread Context (can parallel with Phase 3)
            ↓
        Phase 5: Curriculum Parser (depends on Phase 4 for integration)
            ↓
        Phase 7: Onboarding (depends on curriculum for onboarding module)
```

**Recommended order**: 1 → 2 → 3 & 4 (parallel) → 5 → 6 → 7

---

## Testing Strategy

### Unit Tests
- Memory block serialization/deserialization
- Chat title parsing
- Curriculum markdown parsing
- Context cache behavior

### Integration Tests
- Full message flow: Pipe → Service → Letta → Honcho
- Agent creation and retrieval
- Memory block updates on context change
- Background worker execution

### Manual Testing Steps
1. Create two test users in OpenWebUI
2. Each sends messages in multiple chats
3. Verify isolation (different agents)
4. Verify continuity (same agent across sessions)
5. Check Honcho dashboard for messages
6. Trigger background process, verify updates
7. Modify curriculum, verify hot-reload
8. New user goes through onboarding

---

## Open Questions Summary

These must be resolved before or during implementation:

### Architecture
- [ ] API framework choice (FastAPI recommended)
- [ ] Inter-service authentication (API key vs localhost trust)

### Memory Blocks
- [ ] Exact PersonaBlock schema fields
- [ ] Exact HumanBlock schema fields
- [ ] Size limits per block

### Curriculum
- [ ] Final markdown format specification
- [ ] Trigger expression syntax
- [ ] Completion criteria evaluation method
- [ ] Memory block reference resolution

### Onboarding
- [ ] Initial data collection requirements
- [ ] Clifton StrengthsFinder integration method
- [ ] Group/facilitator assignment process

### Background Processing
- [ ] Idle timeout duration
- [ ] Specific dialectic queries
- [ ] Progress/completion detection logic

---

## References

- Project context: `thoughts/shared/youlab-project-context.md`
- OpenWebUI Pipes: https://docs.openwebui.com/features/plugin/functions/pipe/
- Honcho docs: https://docs.honcho.dev
- Letta docs: https://docs.letta.com
- Current Pipe: `src/letta_starter/pipelines/letta_pipe.py`
- Current memory blocks: `src/letta_starter/memory/blocks.py`
