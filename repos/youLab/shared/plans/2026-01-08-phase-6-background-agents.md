# Phase 6: Background Agents Implementation Plan

## Overview

Implement background agent infrastructure enabling Letta agents to query Honcho's dialectic API for student insights and update memory blocks. This creates a foundation for both real-time agent-driven memory enrichment and scheduled background processing.

## Current State Analysis

### What Exists

- **Honcho Integration (Phase 3)**: Fire-and-forget message persistence via `HonchoClient`
- **Memory Management**: `PersonaBlock`/`HumanBlock` schemas with `MemoryManager` for updates
- **Agent Templates**: `AgentTemplateRegistry` with tool registration capability
- **No dialectic queries**: Only message storage, no insight retrieval

### Key Files

- `src/letta_starter/honcho/client.py` - HonchoClient (message persistence only)
- `src/letta_starter/memory/blocks.py` - PersonaBlock, HumanBlock schemas
- `src/letta_starter/memory/manager.py` - MemoryManager orchestration
- `src/letta_starter/agents/templates.py` - AgentTemplate with tools list
- `src/letta_starter/server/main.py` - FastAPI app

## Desired End State

1. **Two agent-callable tools** registered with tutor agents:
   - `query_honcho`: Query Honcho dialectic for student insights
   - `edit_memory_block`: Update memory block fields with configurable merge strategies

2. **Background worker infrastructure**:
   - YAML-driven configuration (one file per course)
   - Scheduled, idle-based, and manual triggers
   - Batch processing across users

3. **Audit trail** aligned with Letta patterns:
   - Agent tool calls: logged in Letta message history (automatic)
   - Background edits: logged to target agent's archival memory

### Verification

```bash
# Agent can call tools
curl -X POST localhost:8100/chat -d '{"message": "Check my learning style insights"}'
# → Agent calls query_honcho, optionally edit_memory_block

# Manual background trigger
curl -X POST localhost:8100/background/insight-harvester/run
# → Returns execution summary

# Config reload
curl -X POST localhost:8100/config/reload
# → Reloads YAML without restart
```

## What We're NOT Doing

- Full YAML schema for curriculum/lessons (Phase 5)
- Complex multi-agent orchestration
- Real-time idle detection (use scheduled + manual for now)
- Temporal/Celery integration (APScheduler sufficient for MVP)

---

## Implementation Approach

**Priority Order**: Agent tools first (critical path), then background infrastructure.

```
Phase 1: Dialectic Query Service     ─┐
Phase 2: Agent Tools                  ├─► Agent can query & edit
Phase 3: Memory Enricher             ─┘
Phase 4: Background Worker + YAML    ─► Scheduled processing
Phase 5: HTTP Endpoints + Reload     ─► Management API
```

---

## Phase 1: Dialectic Query Service

### Overview

Wrap Honcho's `peer.chat()` method with session scoping and structured responses.

### Changes Required

#### 1. Extend HonchoClient

**File**: `src/letta_starter/honcho/client.py`

**Changes**: Add dialectic query method

```python
from dataclasses import dataclass
from enum import Enum

class SessionScope(str, Enum):
    """Scope for dialectic queries."""
    ALL = "all"           # All sessions for this user
    RECENT = "recent"     # Last N sessions
    CURRENT = "current"   # Current/active session only
    SPECIFIC = "specific" # Explicit session ID

@dataclass
class DialecticResponse:
    """Structured response from Honcho dialectic."""
    insight: str
    session_scope: SessionScope
    query: str


class HonchoClient:
    # ... existing code ...

    async def query_dialectic(
        self,
        user_id: str,
        question: str,
        session_scope: SessionScope = SessionScope.ALL,
        session_id: str | None = None,
        recent_limit: int = 5,
    ) -> DialecticResponse | None:
        """
        Query Honcho dialectic for insights about a student.

        Args:
            user_id: Student identifier
            question: Natural language question
            session_scope: Which sessions to include
            session_id: Specific session ID (when scope=SPECIFIC)
            recent_limit: Number of recent sessions (when scope=RECENT)

        Returns:
            DialecticResponse with insight, or None if unavailable
        """
        if self.client is None:
            return None

        try:
            peer = self.client.peer(self._get_student_peer_id(user_id))

            # Build scope parameter
            scope_param = None
            if session_scope == SessionScope.SPECIFIC and session_id:
                scope_param = self._get_session_id(session_id)
            elif session_scope == SessionScope.CURRENT:
                # Current session must be passed explicitly
                scope_param = self._get_session_id(session_id) if session_id else None
            # ALL and RECENT handled by Honcho's default behavior

            response = peer.chat(question, session_id=scope_param)

            log.info(
                "honcho_dialectic_queried",
                user_id=user_id,
                question_preview=question[:50],
                session_scope=session_scope.value,
            )

            return DialecticResponse(
                insight=response,
                session_scope=session_scope,
                query=question,
            )
        except Exception as e:
            log.warning(
                "honcho_dialectic_failed",
                error=str(e),
                user_id=user_id,
            )
            return None
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Lint passes: `make lint-fix`

#### Manual Verification:
- [ ] `query_dialectic()` returns insight from Honcho demo environment
- [ ] Session scoping filters results appropriately

---

## Phase 2: Agent Tools

### Overview

Create two Letta-compatible tools that agents can call during conversations.

### Changes Required

#### 1. Create Tools Module

**File**: `src/letta_starter/tools/__init__.py` (new)

```python
"""Agent-callable tools for memory enrichment."""

from letta_starter.tools.dialectic import query_honcho
from letta_starter.tools.memory import edit_memory_block

__all__ = ["query_honcho", "edit_memory_block"]
```

#### 2. Dialectic Query Tool

**File**: `src/letta_starter/tools/dialectic.py` (new)

```python
"""Honcho dialectic query tool for Letta agents."""

from __future__ import annotations

from typing import TYPE_CHECKING

import structlog

if TYPE_CHECKING:
    from letta_starter.honcho.client import HonchoClient

log = structlog.get_logger()

# Global reference set by service initialization
_honcho_client: HonchoClient | None = None
_user_context: dict[str, str] = {}  # agent_id -> user_id mapping


def set_honcho_client(client: HonchoClient | None) -> None:
    """Set the global Honcho client for tool use."""
    global _honcho_client
    _honcho_client = client


def set_user_context(agent_id: str, user_id: str) -> None:
    """Set user context for an agent (called before chat)."""
    _user_context[agent_id] = user_id


def query_honcho(
    question: str,
    session_scope: str = "all",
    agent_state: dict | None = None,  # Letta injects this
) -> str:
    """
    Query Honcho for insights about the current student.

    Use this tool to understand:
    - Student learning patterns and preferences
    - Communication style that works best
    - Historical context from past conversations
    - Engagement patterns and motivations

    Args:
        question: Natural language question about the student
                  (e.g., "What learning style works best for this student?")
        session_scope: Which conversations to include:
                      - "all": All sessions (default)
                      - "recent": Last few sessions
                      - "current": This conversation only

    Returns:
        Honcho's insight about the student based on conversation history.
        Returns error message if query fails.
    """
    if _honcho_client is None:
        return "Honcho is not available. Proceeding without external insights."

    # Get user_id from agent context
    agent_id = agent_state.get("agent_id") if agent_state else None
    user_id = _user_context.get(agent_id) if agent_id else None

    if not user_id:
        return "Unable to identify current student. Cannot query Honcho."

    import asyncio
    from letta_starter.honcho.client import SessionScope

    try:
        scope = SessionScope(session_scope)
    except ValueError:
        scope = SessionScope.ALL

    # Run async query in sync context
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(
        _honcho_client.query_dialectic(
            user_id=user_id,
            question=question,
            session_scope=scope,
        )
    )

    if result is None:
        return "Failed to query Honcho. The service may be temporarily unavailable."

    return result.insight
```

#### 3. Memory Edit Tool

**File**: `src/letta_starter/tools/memory.py` (new)

```python
"""Memory block editing tool for Letta agents."""

from __future__ import annotations

from datetime import datetime
from enum import Enum
from typing import TYPE_CHECKING, Any

import structlog

if TYPE_CHECKING:
    from letta import Letta

log = structlog.get_logger()

# Global references
_letta_client: Letta | None = None

# Protected fields that require explicit override
PROTECTED_FIELDS = {"persona.name", "persona.role"}


class MergeStrategy(str, Enum):
    """Strategy for merging new content with existing."""
    APPEND = "append"       # Add to existing list/content
    REPLACE = "replace"     # Overwrite existing content
    LLM_DIFF = "llm_diff"   # Use LLM to intelligently merge


def set_letta_client(client: Letta) -> None:
    """Set the global Letta client for tool use."""
    global _letta_client
    _letta_client = client


def edit_memory_block(
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    agent_state: dict | None = None,
) -> str:
    """
    Update a field in your memory blocks.

    Use this tool to:
    - Record learned facts about the student
    - Update context notes with new information
    - Adjust your communication style based on insights

    Args:
        block: Which memory block to edit:
               - "human": Student context (facts, preferences, notes)
               - "persona": Your behavior (constraints, style)
        field: Which field to update:
               - human: "context_notes", "facts", "preferences"
               - persona: "constraints", "expertise"
        content: The content to add or replace
        strategy: How to merge with existing content:
                 - "append": Add to existing (default, safe)
                 - "replace": Overwrite existing
                 - "llm_diff": Intelligently merge old and new

    Returns:
        Confirmation of the update or error message.
    """
    if _letta_client is None:
        return "Memory system unavailable. Update not applied."

    agent_id = agent_state.get("agent_id") if agent_state else None
    if not agent_id:
        return "Unable to identify agent. Update not applied."

    # Check protected fields
    field_key = f"{block}.{field}"
    if field_key in PROTECTED_FIELDS:
        log.warning(
            "protected_field_edit_attempted",
            agent_id=agent_id,
            field=field_key,
        )
        return f"Cannot edit protected field '{field_key}'. This field requires manual configuration."

    try:
        strategy_enum = MergeStrategy(strategy)
    except ValueError:
        strategy_enum = MergeStrategy.APPEND

    try:
        result = _apply_memory_edit(
            agent_id=agent_id,
            block=block,
            field=field,
            content=content,
            strategy=strategy_enum,
        )

        log.info(
            "memory_block_edited",
            agent_id=agent_id,
            block=block,
            field=field,
            strategy=strategy,
            content_preview=content[:50],
            source="agent_tool",
        )

        return result

    except Exception as e:
        log.error(
            "memory_edit_failed",
            agent_id=agent_id,
            block=block,
            field=field,
            error=str(e),
        )
        return f"Failed to update memory: {e}"


def _apply_memory_edit(
    agent_id: str,
    block: str,
    field: str,
    content: str,
    strategy: MergeStrategy,
) -> str:
    """Apply the memory edit using MemoryManager."""
    from letta_starter.memory.blocks import HumanBlock, PersonaBlock
    from letta_starter.memory.manager import MemoryManager

    manager = MemoryManager(
        client=_letta_client,
        agent_id=agent_id,
    )

    if block == "human":
        human = manager.get_human_block()
        _update_human_field(human, field, content, strategy)
        manager.update_human(human)
        return f"Updated human.{field} via {strategy.value}"

    elif block == "persona":
        persona = manager.get_persona_block()
        _update_persona_field(persona, field, content, strategy)
        manager.update_persona(persona)
        return f"Updated persona.{field} via {strategy.value}"

    else:
        return f"Unknown block '{block}'. Use 'human' or 'persona'."


def _update_human_field(
    human: HumanBlock,
    field: str,
    content: str,
    strategy: MergeStrategy,
) -> None:
    """Update a field on the human block."""
    if field == "context_notes":
        if strategy == MergeStrategy.REPLACE:
            human.context_notes = [content]
        elif strategy == MergeStrategy.APPEND:
            human.add_context_note(content)
        elif strategy == MergeStrategy.LLM_DIFF:
            merged = _llm_merge(human.context_notes, content)
            human.context_notes = [merged]

    elif field == "facts":
        if strategy == MergeStrategy.REPLACE:
            human.facts = [content]
        elif strategy == MergeStrategy.APPEND:
            human.add_fact(content)
        elif strategy == MergeStrategy.LLM_DIFF:
            merged = _llm_merge(human.facts, content)
            human.facts = [merged]

    elif field == "preferences":
        if strategy == MergeStrategy.REPLACE:
            human.preferences = [content]
        elif strategy == MergeStrategy.APPEND:
            human.add_preference(content)
        elif strategy == MergeStrategy.LLM_DIFF:
            merged = _llm_merge(human.preferences, content)
            human.preferences = [merged]
    else:
        raise ValueError(f"Unknown human field: {field}")


def _update_persona_field(
    persona: PersonaBlock,
    field: str,
    content: str,
    strategy: MergeStrategy,
) -> None:
    """Update a field on the persona block."""
    if field == "constraints":
        if strategy == MergeStrategy.REPLACE:
            persona.constraints = [content]
        elif strategy == MergeStrategy.APPEND:
            if content not in persona.constraints:
                persona.constraints.append(content)
        elif strategy == MergeStrategy.LLM_DIFF:
            merged = _llm_merge(persona.constraints, content)
            persona.constraints = [merged]

    elif field == "expertise":
        if strategy == MergeStrategy.REPLACE:
            persona.expertise = [content]
        elif strategy == MergeStrategy.APPEND:
            if content not in persona.expertise:
                persona.expertise.append(content)
        elif strategy == MergeStrategy.LLM_DIFF:
            merged = _llm_merge(persona.expertise, content)
            persona.expertise = [merged]
    else:
        raise ValueError(f"Unknown persona field: {field}")


def _llm_merge(existing: list[str] | str, new_content: str) -> str:
    """Use LLM to intelligently merge existing and new content."""
    # TODO: Implement LLM-based merging
    # For now, fall back to append behavior
    if isinstance(existing, list):
        existing_str = "; ".join(existing)
    else:
        existing_str = existing

    return f"{existing_str}; {new_content}"
```

#### 4. Register Tools with Agent Template

**File**: `src/letta_starter/agents/templates.py`

**Changes**: Add tools to TUTOR_TEMPLATE

```python
from letta_starter.tools import query_honcho, edit_memory_block

TUTOR_TEMPLATE = AgentTemplate(
    name="tutor",
    # ... existing fields ...
    tools=[
        query_honcho,
        edit_memory_block,
    ],
)
```

#### 5. Initialize Tools in Server Lifespan

**File**: `src/letta_starter/server/main.py`

**Changes**: Set global clients during startup

```python
from letta_starter.tools.dialectic import set_honcho_client, set_user_context
from letta_starter.tools.memory import set_letta_client

@asynccontextmanager
async def lifespan(app: FastAPI):
    # ... existing initialization ...

    # Initialize tool globals
    set_letta_client(agent_manager.client)
    if app.state.honcho_client:
        set_honcho_client(app.state.honcho_client)

    yield
```

#### 6. Set User Context Before Chat

**File**: `src/letta_starter/server/main.py`

**Changes**: In chat endpoints, set user context before sending message

```python
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    # ... existing code ...

    # Set user context for tools
    set_user_context(agent_id=agent_id, user_id=request.user_id)

    # ... proceed with chat ...
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests for tools pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Tools register correctly with Letta agent

#### Manual Verification:
- [ ] Agent can call `query_honcho` and receive insight
- [ ] Agent can call `edit_memory_block` and memory updates
- [ ] Protected fields are blocked with appropriate message
- [ ] Tool calls appear in agent's message history

**Implementation Note**: After Phase 2, pause for manual testing of the two tools before proceeding.

---

## Phase 3: Memory Enricher Service

### Overview

Create a service that handles memory edits from external sources (background workers) with proper audit trailing.

### Changes Required

#### 1. Create Memory Enricher

**File**: `src/letta_starter/memory/enricher.py` (new)

```python
"""Memory enrichment service for external updates."""

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import TYPE_CHECKING

import structlog

if TYPE_CHECKING:
    from letta import Letta

from letta_starter.memory.blocks import HumanBlock, PersonaBlock
from letta_starter.memory.manager import MemoryManager

log = structlog.get_logger()


class MergeStrategy(str, Enum):
    """Strategy for merging content."""
    APPEND = "append"
    REPLACE = "replace"
    LLM_DIFF = "llm_diff"


@dataclass
class EnrichmentResult:
    """Result of a memory enrichment operation."""
    success: bool
    block: str
    field: str
    strategy: MergeStrategy
    message: str
    audit_entry_id: str | None = None


class MemoryEnricher:
    """
    Service for external memory enrichment with audit trailing.

    Unlike agent tools (which auto-log to message history),
    this service handles edits from background workers and
    writes audit entries to the target agent's archival memory.
    """

    def __init__(self, client: Letta) -> None:
        self.client = client
        self.logger = log.bind(component="memory_enricher")

    def enrich(
        self,
        agent_id: str,
        block: str,
        field: str,
        content: str,
        strategy: MergeStrategy = MergeStrategy.APPEND,
        source: str = "background_worker",
        source_query: str | None = None,
    ) -> EnrichmentResult:
        """
        Enrich an agent's memory with external content.

        Args:
            agent_id: Target agent to enrich
            block: Memory block ("human" or "persona")
            field: Field to update
            content: Content to add/replace
            strategy: Merge strategy
            source: Source of the enrichment
            source_query: Optional query that produced this content

        Returns:
            EnrichmentResult with success status and details
        """
        manager = MemoryManager(client=self.client, agent_id=agent_id)

        try:
            # Apply the edit
            if block == "human":
                self._enrich_human(manager, field, content, strategy)
            elif block == "persona":
                self._enrich_persona(manager, field, content, strategy)
            else:
                return EnrichmentResult(
                    success=False,
                    block=block,
                    field=field,
                    strategy=strategy,
                    message=f"Unknown block: {block}",
                )

            # Write audit entry to target agent's archival
            audit_id = self._write_audit_entry(
                agent_id=agent_id,
                block=block,
                field=field,
                content=content,
                strategy=strategy,
                source=source,
                source_query=source_query,
            )

            self.logger.info(
                "memory_enriched",
                agent_id=agent_id,
                block=block,
                field=field,
                strategy=strategy.value,
                source=source,
            )

            return EnrichmentResult(
                success=True,
                block=block,
                field=field,
                strategy=strategy,
                message=f"Enriched {block}.{field} via {strategy.value}",
                audit_entry_id=audit_id,
            )

        except Exception as e:
            self.logger.error(
                "enrichment_failed",
                agent_id=agent_id,
                block=block,
                field=field,
                error=str(e),
            )
            return EnrichmentResult(
                success=False,
                block=block,
                field=field,
                strategy=strategy,
                message=f"Enrichment failed: {e}",
            )

    def _enrich_human(
        self,
        manager: MemoryManager,
        field: str,
        content: str,
        strategy: MergeStrategy,
    ) -> None:
        """Apply enrichment to human block."""
        human = manager.get_human_block()

        if field == "context_notes":
            if strategy == MergeStrategy.REPLACE:
                human.context_notes = [content]
            else:  # APPEND or LLM_DIFF (TODO: implement diff)
                human.add_context_note(content)
        elif field == "facts":
            if strategy == MergeStrategy.REPLACE:
                human.facts = [content]
            else:
                human.add_fact(content)
        elif field == "preferences":
            if strategy == MergeStrategy.REPLACE:
                human.preferences = [content]
            else:
                human.add_preference(content)
        else:
            raise ValueError(f"Unknown human field: {field}")

        manager.update_human(human)

    def _enrich_persona(
        self,
        manager: MemoryManager,
        field: str,
        content: str,
        strategy: MergeStrategy,
    ) -> None:
        """Apply enrichment to persona block."""
        persona = manager.get_persona_block()

        if field == "constraints":
            if strategy == MergeStrategy.REPLACE:
                persona.constraints = [content]
            else:
                if content not in persona.constraints:
                    persona.constraints.append(content)
        elif field == "expertise":
            if strategy == MergeStrategy.REPLACE:
                persona.expertise = [content]
            else:
                if content not in persona.expertise:
                    persona.expertise.append(content)
        else:
            raise ValueError(f"Unknown persona field: {field}")

        manager.update_persona(persona)

    def _write_audit_entry(
        self,
        agent_id: str,
        block: str,
        field: str,
        content: str,
        strategy: MergeStrategy,
        source: str,
        source_query: str | None,
    ) -> str | None:
        """Write audit entry to agent's archival memory."""
        timestamp = datetime.now().isoformat()

        entry_parts = [
            f"[MEMORY_EDIT {timestamp}]",
            f"Source: {source}",
            f"Block: {block}",
            f"Field: {field}",
            f"Strategy: {strategy.value}",
        ]

        if source_query:
            entry_parts.append(f"Query: {source_query[:100]}")

        entry_parts.append(f"Content: {content[:200]}{'...' if len(content) > 200 else ''}")

        audit_entry = "\n".join(entry_parts)

        try:
            self.client.insert_archival_memory(
                agent_id=agent_id,
                memory=audit_entry,
            )
            return timestamp  # Use timestamp as pseudo-ID
        except Exception as e:
            self.logger.warning(
                "audit_entry_failed",
                agent_id=agent_id,
                error=str(e),
            )
            return None
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] `MemoryEnricher.enrich()` updates target agent's memory
- [ ] Audit entry appears in agent's archival memory
- [ ] Audit entry is searchable via archival search

---

## Phase 4: Background Worker + YAML Configuration

### Overview

Create YAML-driven background agent configuration and execution engine.

### Changes Required

#### 1. YAML Schema Definition

**File**: `src/letta_starter/background/schema.py` (new)

```python
"""YAML configuration schema for background agents."""

from __future__ import annotations

from enum import Enum
from pathlib import Path
from typing import Any

import yaml
from pydantic import BaseModel, Field


class SessionScope(str, Enum):
    ALL = "all"
    RECENT = "recent"
    CURRENT = "current"
    SPECIFIC = "specific"


class MergeStrategy(str, Enum):
    APPEND = "append"
    REPLACE = "replace"
    LLM_DIFF = "llm_diff"


class IdleTrigger(BaseModel):
    """Idle-based trigger configuration."""
    enabled: bool = False
    threshold_minutes: int = 30
    cooldown_minutes: int = 60


class Triggers(BaseModel):
    """Trigger configuration for background agent."""
    schedule: str | None = None  # Cron expression
    idle: IdleTrigger = Field(default_factory=IdleTrigger)
    manual: bool = True


class DialecticQuery(BaseModel):
    """Single dialectic query configuration."""
    id: str
    question: str
    session_scope: SessionScope = SessionScope.ALL
    recent_limit: int = 5
    target_block: str  # "human" or "persona"
    target_field: str  # "context_notes", "facts", etc.
    merge_strategy: MergeStrategy = MergeStrategy.APPEND


class BackgroundAgentConfig(BaseModel):
    """Configuration for a single background agent."""
    id: str
    name: str
    enabled: bool = True
    triggers: Triggers = Field(default_factory=Triggers)
    agent_types: list[str] = Field(default_factory=lambda: ["tutor"])
    user_filter: str = "all"  # "all" or specific user_ids
    batch_size: int = 50
    queries: list[DialecticQuery] = Field(default_factory=list)


class CourseConfig(BaseModel):
    """Course-level configuration including background agents."""
    id: str
    name: str
    background_agents: list[BackgroundAgentConfig] = Field(default_factory=list)


def load_course_config(path: Path) -> CourseConfig:
    """Load course configuration from YAML file."""
    with open(path) as f:
        data = yaml.safe_load(f)
    return CourseConfig(**data)


def load_all_course_configs(directory: Path) -> dict[str, CourseConfig]:
    """Load all course configs from a directory."""
    configs = {}
    for yaml_file in directory.glob("*.yaml"):
        config = load_course_config(yaml_file)
        configs[config.id] = config
    return configs
```

#### 2. Example Course YAML

**File**: `config/courses/college-essay.yaml` (new)

```yaml
id: college-essay
name: College Essay Coaching

background_agents:
  - id: insight-harvester
    name: Student Insight Harvester
    enabled: true

    triggers:
      schedule: "0 3 * * *"  # 3 AM daily
      idle:
        enabled: false  # Disabled for MVP
        threshold_minutes: 30
        cooldown_minutes: 60
      manual: true

    agent_types:
      - tutor

    user_filter: all
    batch_size: 50

    queries:
      - id: learning_style
        question: "What learning style works best for this student? Do they prefer examples, theory, or hands-on practice?"
        session_scope: all
        target_block: human
        target_field: context_notes
        merge_strategy: append

      - id: engagement_patterns
        question: "How engaged is this student? What topics or activities seem to motivate them?"
        session_scope: recent
        recent_limit: 5
        target_block: human
        target_field: facts
        merge_strategy: append

      - id: communication_style
        question: "How should I adjust my communication style for this student? What tone resonates best?"
        session_scope: all
        target_block: persona
        target_field: constraints
        merge_strategy: llm_diff
```

#### 3. Background Worker Runner

**File**: `src/letta_starter/background/runner.py` (new)

```python
"""Background agent execution engine."""

from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime
from typing import TYPE_CHECKING

import structlog

if TYPE_CHECKING:
    from letta import Letta
    from letta_starter.honcho.client import HonchoClient

from letta_starter.background.schema import (
    BackgroundAgentConfig,
    CourseConfig,
    DialecticQuery,
)
from letta_starter.honcho.client import SessionScope
from letta_starter.memory.enricher import MemoryEnricher, MergeStrategy

log = structlog.get_logger()


@dataclass
class QueryResult:
    """Result of a single query execution."""
    query_id: str
    user_id: str
    agent_id: str
    success: bool
    insight: str | None = None
    error: str | None = None


@dataclass
class RunResult:
    """Result of a background agent run."""
    agent_id: str
    started_at: datetime
    completed_at: datetime | None = None
    users_processed: int = 0
    queries_executed: int = 0
    enrichments_applied: int = 0
    errors: list[str] = field(default_factory=list)


class BackgroundAgentRunner:
    """Executes background agents based on configuration."""

    def __init__(
        self,
        letta_client: Letta,
        honcho_client: HonchoClient | None,
    ) -> None:
        self.letta = letta_client
        self.honcho = honcho_client
        self.enricher = MemoryEnricher(letta_client)
        self.logger = log.bind(component="background_runner")

    async def run_agent(
        self,
        config: BackgroundAgentConfig,
        user_ids: list[str] | None = None,
    ) -> RunResult:
        """
        Execute a background agent for specified users.

        Args:
            config: Background agent configuration
            user_ids: Specific users to process (None = all)

        Returns:
            RunResult with execution details
        """
        result = RunResult(
            agent_id=config.id,
            started_at=datetime.now(),
        )

        if not config.enabled:
            result.errors.append("Agent is disabled")
            result.completed_at = datetime.now()
            return result

        if self.honcho is None:
            result.errors.append("Honcho client not available")
            result.completed_at = datetime.now()
            return result

        # Get users to process
        target_users = user_ids or await self._get_target_users(config)

        self.logger.info(
            "background_agent_started",
            agent_id=config.id,
            user_count=len(target_users),
            query_count=len(config.queries),
        )

        # Process users in batches
        for i in range(0, len(target_users), config.batch_size):
            batch = target_users[i:i + config.batch_size]

            for user_id in batch:
                await self._process_user(
                    config=config,
                    user_id=user_id,
                    result=result,
                )

        result.completed_at = datetime.now()

        self.logger.info(
            "background_agent_completed",
            agent_id=config.id,
            users_processed=result.users_processed,
            queries_executed=result.queries_executed,
            enrichments_applied=result.enrichments_applied,
            errors=len(result.errors),
        )

        return result

    async def _get_target_users(
        self,
        config: BackgroundAgentConfig,
    ) -> list[str]:
        """Get list of users to process based on config."""
        # TODO: Implement user filtering based on config.user_filter
        # For now, get all users with agents of specified types

        users = []
        for agent_type in config.agent_types:
            # List agents matching pattern youlab_{user_id}_{agent_type}
            agents = self.letta.list_agents()
            for agent in agents:
                if agent.name and f"_{agent_type}" in agent.name:
                    # Extract user_id from agent name
                    parts = agent.name.split("_")
                    if len(parts) >= 3:
                        user_id = parts[1]
                        if user_id not in users:
                            users.append(user_id)

        return users

    async def _process_user(
        self,
        config: BackgroundAgentConfig,
        user_id: str,
        result: RunResult,
    ) -> None:
        """Process all queries for a single user."""
        result.users_processed += 1

        for query in config.queries:
            await self._execute_query(
                config=config,
                query=query,
                user_id=user_id,
                result=result,
            )

    async def _execute_query(
        self,
        config: BackgroundAgentConfig,
        query: DialecticQuery,
        user_id: str,
        result: RunResult,
    ) -> None:
        """Execute a single query and apply enrichment."""
        result.queries_executed += 1

        # Query Honcho dialectic
        scope = SessionScope(query.session_scope.value)
        response = await self.honcho.query_dialectic(
            user_id=user_id,
            question=query.question,
            session_scope=scope,
            recent_limit=query.recent_limit,
        )

        if response is None:
            result.errors.append(
                f"Dialectic query failed for user {user_id}: {query.id}"
            )
            return

        # Find target agent
        agent_id = self._get_agent_id(user_id, config.agent_types[0])
        if not agent_id:
            result.errors.append(f"No agent found for user {user_id}")
            return

        # Apply enrichment
        strategy = MergeStrategy(query.merge_strategy.value)
        enrich_result = self.enricher.enrich(
            agent_id=agent_id,
            block=query.target_block,
            field=query.target_field,
            content=response.insight,
            strategy=strategy,
            source=f"background:{config.id}",
            source_query=query.question,
        )

        if enrich_result.success:
            result.enrichments_applied += 1
        else:
            result.errors.append(
                f"Enrichment failed for {user_id}/{query.id}: {enrich_result.message}"
            )

    def _get_agent_id(self, user_id: str, agent_type: str) -> str | None:
        """Look up agent ID for a user."""
        agent_name = f"youlab_{user_id}_{agent_type}"
        agents = self.letta.list_agents()
        for agent in agents:
            if agent.name == agent_name:
                return agent.id
        return None
```

### Success Criteria

#### Automated Verification:
- [ ] YAML schema validation passes
- [ ] Unit tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] Example YAML loads correctly
- [ ] Runner processes users and applies enrichments
- [ ] Errors are properly logged and reported

---

## Phase 5: HTTP Endpoints + Config Reload

### Overview

Add HTTP endpoints for manual triggers and configuration reload.

### Changes Required

#### 1. Background Router

**File**: `src/letta_starter/server/background.py` (new)

```python
"""Background agent management endpoints."""

from __future__ import annotations

from pathlib import Path
from typing import TYPE_CHECKING

import structlog
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

if TYPE_CHECKING:
    from letta import Letta
    from letta_starter.honcho.client import HonchoClient

from letta_starter.background.runner import BackgroundAgentRunner, RunResult
from letta_starter.background.schema import CourseConfig, load_all_course_configs

log = structlog.get_logger()

router = APIRouter(prefix="/background", tags=["background"])

# Global state
_configs: dict[str, CourseConfig] = {}
_runner: BackgroundAgentRunner | None = None


def initialize_background(
    letta_client: Letta,
    honcho_client: HonchoClient | None,
    config_dir: Path,
) -> None:
    """Initialize background agent system."""
    global _configs, _runner

    _runner = BackgroundAgentRunner(letta_client, honcho_client)
    _configs = load_all_course_configs(config_dir)

    log.info(
        "background_system_initialized",
        courses_loaded=len(_configs),
        course_ids=list(_configs.keys()),
    )


class RunRequest(BaseModel):
    """Request to run a background agent."""
    user_ids: list[str] | None = None  # None = all users


class RunResponse(BaseModel):
    """Response from background agent run."""
    agent_id: str
    started_at: str
    completed_at: str | None
    users_processed: int
    queries_executed: int
    enrichments_applied: int
    error_count: int
    errors: list[str]


class ReloadResponse(BaseModel):
    """Response from config reload."""
    success: bool
    courses_loaded: int
    course_ids: list[str]
    message: str


@router.post("/{agent_id}/run", response_model=RunResponse)
async def run_background_agent(
    agent_id: str,
    request: RunRequest | None = None,
) -> RunResponse:
    """
    Manually trigger a background agent run.

    Args:
        agent_id: ID of the background agent to run
        request: Optional user filtering

    Returns:
        Execution result summary
    """
    if _runner is None:
        raise HTTPException(500, "Background system not initialized")

    # Find agent config across all courses
    agent_config = None
    for course in _configs.values():
        for agent in course.background_agents:
            if agent.id == agent_id:
                agent_config = agent
                break
        if agent_config:
            break

    if agent_config is None:
        raise HTTPException(404, f"Background agent '{agent_id}' not found")

    user_ids = request.user_ids if request else None
    result = await _runner.run_agent(agent_config, user_ids)

    return RunResponse(
        agent_id=result.agent_id,
        started_at=result.started_at.isoformat(),
        completed_at=result.completed_at.isoformat() if result.completed_at else None,
        users_processed=result.users_processed,
        queries_executed=result.queries_executed,
        enrichments_applied=result.enrichments_applied,
        error_count=len(result.errors),
        errors=result.errors[:10],  # Limit errors in response
    )


@router.get("/agents")
async def list_background_agents() -> list[dict]:
    """List all configured background agents."""
    agents = []
    for course in _configs.values():
        for agent in course.background_agents:
            agents.append({
                "id": agent.id,
                "name": agent.name,
                "course_id": course.id,
                "enabled": agent.enabled,
                "triggers": {
                    "schedule": agent.triggers.schedule,
                    "idle_enabled": agent.triggers.idle.enabled,
                    "manual": agent.triggers.manual,
                },
                "query_count": len(agent.queries),
            })
    return agents


@router.post("/config/reload", response_model=ReloadResponse)
async def reload_config(config_dir: str | None = None) -> ReloadResponse:
    """
    Reload YAML configuration files.

    Args:
        config_dir: Optional path override

    Returns:
        Reload status
    """
    global _configs

    try:
        path = Path(config_dir) if config_dir else Path("config/courses")
        _configs = load_all_course_configs(path)

        log.info(
            "config_reloaded",
            courses_loaded=len(_configs),
            course_ids=list(_configs.keys()),
        )

        return ReloadResponse(
            success=True,
            courses_loaded=len(_configs),
            course_ids=list(_configs.keys()),
            message="Configuration reloaded successfully",
        )
    except Exception as e:
        log.error("config_reload_failed", error=str(e))
        return ReloadResponse(
            success=False,
            courses_loaded=len(_configs),
            course_ids=list(_configs.keys()),
            message=f"Reload failed: {e}",
        )
```

#### 2. Register Router in Main App

**File**: `src/letta_starter/server/main.py`

**Changes**: Add background router

```python
from letta_starter.server.background import router as background_router, initialize_background

# In lifespan()
initialize_background(
    letta_client=agent_manager.client,
    honcho_client=app.state.honcho_client,
    config_dir=Path("config/courses"),
)

# After app creation
app.include_router(background_router)
```

### Success Criteria

#### Automated Verification:
- [ ] All tests pass: `make verify-agent`
- [ ] OpenAPI schema includes new endpoints

#### Manual Verification:
- [ ] `GET /background/agents` lists configured agents
- [ ] `POST /background/{agent_id}/run` executes and returns results
- [ ] `POST /background/config/reload` reloads YAML without restart
- [ ] Errors are properly returned in response

---

## Testing Strategy

### Unit Tests

**New test files:**
- `tests/test_honcho_dialectic.py` - Dialectic query tests
- `tests/test_tools.py` - Agent tool tests
- `tests/test_memory_enricher.py` - Enricher tests
- `tests/test_background_schema.py` - YAML schema tests
- `tests/test_background_runner.py` - Runner tests

**Key test cases:**
- Dialectic query with different session scopes
- Memory edit with each merge strategy
- Protected field blocking
- YAML validation errors
- Runner error handling
- Audit entry creation

### Integration Tests

- End-to-end: Agent calls `query_honcho` → receives insight
- End-to-end: Agent calls `edit_memory_block` → memory updates
- Background run: Runner processes user → enrichment applied
- Config reload: YAML change → reflected in next run

### Manual Testing Steps

1. Start service with example YAML config
2. Create test user agent via `/agents`
3. Send chat message that triggers `query_honcho` tool
4. Verify insight returned in agent response
5. Send message that triggers `edit_memory_block`
6. Verify memory block updated via `/agents/{id}`
7. Trigger background agent via `/background/insight-harvester/run`
8. Verify enrichments applied and audit entries created
9. Modify YAML, call `/background/config/reload`
10. Verify new config reflected in `/background/agents`

---

## Migration Notes

### New Dependencies

```toml
# pyproject.toml additions
pyyaml = "^6.0"
apscheduler = "^3.10"  # For future scheduled triggers
```

### New Directory Structure

```
config/
  courses/
    college-essay.yaml
src/letta_starter/
  tools/
    __init__.py
    dialectic.py
    memory.py
  background/
    __init__.py
    schema.py
    runner.py
  server/
    background.py  (new router)
```

### Configuration

New environment variables (optional):
```bash
YOULAB_BACKGROUND_CONFIG_DIR=/path/to/config/courses
YOULAB_BACKGROUND_LLM_MODEL=claude-3-haiku  # For llm_diff strategy
```

---

## References

- Roadmap Phase 6: `docs/Roadmap.md:206-222`
- Existing TOML schema: `thoughts/shared/plans/2026-01-07-sleep-agents-toml-schema.toml`
- Honcho SDK: [docs.honcho.dev](https://docs.honcho.dev/v2/documentation/reference/sdk)
- Letta Memory Blocks: [docs.letta.com/guides/agents/memory-blocks](https://docs.letta.com/guides/agents/memory-blocks/)
