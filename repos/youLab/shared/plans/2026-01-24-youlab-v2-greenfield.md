# YouLab v2: Greenfield Architecture Plan

## Overview

This document captures the architectural direction for YouLab v2 - a greenfield rebuild replacing Letta with a cleaner, more portable architecture. The primary goal is a **web-based tutoring platform using OpenWebUI** that can run the college essay course. The architecture is designed with future extensibility in mind (multi-interface support, API access, etc.) but those are not MVP requirements.

**This is a living document.** Implementation details and design choices are intentionally left open where not explicitly decided.

---

## Core Ethos

These principles guide all architectural decisions:

1. **User Data Ownership**: Users own their context. Not as a moat, but as their property.
2. **Portability**: Users can export, self-host, or migrate their data.
3. **Open Source**: Architecture should work for self-hosted deployments.
4. **Local-First Capable**: Can run entirely locally (future, not MVP).
5. **Proactive Context**: Background agents work for users even when they're not chatting to diff and pr their memory blocks.
Users have complete control over their triggers, prompts, memory blocks, permissions.

---

## Agreed Architecture

### High-Level View

```
┌─────────────────────────────────────────────────────────────────┐
│                        OpenWebUI                                │
│                  (Frontend + Thread UI)                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      YouLab Service                             │
│                      (FastAPI + Agno)                           │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Chat Agent (Agno)                      │   │
│   │   Tools: edit_memory_block, query_honcho, advance_lesson│   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Background Agent Manager                   │   │
│   │   Scheduled + triggered tasks, proactive analysis       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
  ┌───────────┐     ┌───────────┐     ┌───────────┐
  │   Dolt    │     │  Honcho   │     │  Claude   │
  │  (MySQL)  │     │           │     │   API     │
  └───────────┘     └───────────┘     └───────────┘

  Knowledge         Conversation        LLM
  Layer             Layer               Inference
```

### Data Separation (Agreed)

| Data Type | Store | Rationale |
|-----------|-------|-----------|
| Memory blocks | Dolt | Version controlled, diffable, user-editable |
| Pending diffs | Dolt | Tied to block approval workflow |
| User preferences | Dolt | Stored as memory blocks; user-controlled settings |
| Notes/artifacts | Dolt | User-created content (future) |
| **Messages** | **Honcho** | Append-only, needs dialectic/theory-of-mind |
| **Sessions** | **Honcho** | Conversation structure for dialectic |

**Key Decision**: Messages don't need version control. Honcho's dialectic capabilities are valuable and non-trivial to rebuild. Both Dolt and Honcho are open source and self-hostable.

---

## What's Replacing What

| Current (Letta-based) | New (Agno + Dolt + Honcho) |
|-----------------------|----------------------------|
| Letta server + PostgreSQL | Removed entirely |
| Letta client SDK | Removed |
| Git repos per user | Dolt (single database) |
| Block sync layer (git → Letta) | Removed (Dolt is sole truth for blocks) |
| Sandbox tool duplication | Removed (in-process tools) |
| Global state patterns | RunContext session state |
| Complex streaming layers | Single Agno streaming layer |

---

## Components

### 1. YouLab Service (FastAPI)

The main backend service. Responsibilities:
- HTTP endpoints for chat, blocks, diffs, export
- Agno agent creation and streaming
- Background agent scheduling and execution
- Coordination between Dolt and Honcho

**Status**: Architecture agreed. Implementation details TBD.

### 2. Dolt Storage Layer

Replaces Git-backed storage with MySQL-compatible Dolt.

**Why Dolt over DoltgreSQL**: Dolt (MySQL) is more mature. We don't need pgvector since Honcho handles message semantics.

**Schema**: TBD - will include at minimum:
- `blocks` table (user_id, label, content, metadata)
- `pending_diffs` table (approval workflow)
- Possibly, task queue tables

**Storage class interface**: TBD - likely similar to existing `GitUserStorage` interface.

### 3. Honcho Integration

Continues to handle conversation storage and dialectic queries.

**Key points**:
- Source of truth for messages/sessions
- Provides `query_honcho` tool functionality
- Fire-and-forget persistence from chat flow
- Open source, self-hostable

**Sync model**: Messages written to Honcho after each exchange. Dolt does NOT store messages.

### 4. Agno Agent Layer

Stateless agents created per-request.

**Agreed patterns**:
- `@tool` decorator for tool definitions
- `RunContext` for dependency injection (no global state)
- Session state carries user_id, course_id, etc.
- Streaming via `arun_response_stream()`

**Tools to implement**:
- `edit_memory_block` - proposes diffs to Dolt
- `query_honcho` - dialectic queries
- `advance_lesson` - curriculum navigation

**Tool implementation details**: TBD

### 5. Background Agent Manager

**This is a key differentiator.** Proactive agents that:
- Run on schedules (cron-like, built into service)
- Trigger on events (conversation end, diff approved, etc.)
- Analyze conversations via Honcho
- Propose memory updates to Dolt
- Create observable results (logs, metrics)

**Implementation details**: TBD. Options include:
- TOML-configured tasks (like current system)
- Dolt-backed task queue
- Simple in-memory scheduler
- External queue (Redis, etc.)

### 6. OpenWebUI Pipe

Connects OpenWebUI frontend to YouLab service.

**Responsibilities**:
- Extract user_id, course_id, module_id from model name
- Forward messages to `/chat/stream`
- Handle SSE streaming back to UI

**Status**: Architecture agreed. Details TBD.

---

## User Data Portability

### Export Capabilities (MVP)

Users should be able to export their data. Minimum viable:
- Memory blocks (from Dolt)
- Conversation history (from Honcho)
- Block version history (from Dolt)

**Export format**: TBD (JSON, SQLite, Dolt clone, etc.)

### Self-Hosting (Future)

Architecture supports self-hosting:
- Dolt can run locally or as server
- Honcho is self-hostable
- Local LLM option (Ollama) possible but not MVP

---

## Future Extensibility (Not MVP)

These are documented for architectural awareness, not immediate implementation:

### Multi-Interface Support

The architecture should not preclude future interfaces:
- **MCP Server**: Expose context to Claude Code and other MCP clients
- **REST API**: Direct API access for custom integrations
- **Python SDK**: Library for programmatic access
- **Webhooks**: Event-driven integrations

### Local-First Mode

Future option to run entirely locally:
- Embedded Dolt (no server)
- Ollama for LLM inference
- Honcho optional or local

---

## Migration from Current System

### What Stays
- Course TOML configuration (`config/courses/`)
- Curriculum schema system
- OpenWebUI frontend (with pipe updates)
- Honcho integration patterns

### What Gets Deleted
- `server/agents.py` (Letta AgentManager)
- `tools/sandbox.py` (HTTP tool duplication)
- `storage/git.py` (Git-backed storage)
- `background/factory.py` (Letta background factory)
- `pipelines/letta_pipe.py` (Letta streaming)
- All Letta sync code in `storage/blocks.py`

### Data Migration
- Memory blocks: Export from Git, import to Dolt
- Existing Honcho data: Unchanged
- Pending diffs: Migrate format if needed

---

## Open Questions (Require Decision)

These need explicit decisions before implementation:

1. **Dolt schema details**: Exact table structures, indexes, constraints?

2. **Connection management**: Connection pooling strategy for Dolt?

3. **Background task configuration**:
   - TOML-based (like current)?
   - Database-driven?
   - Hardcoded MVP?

4. **Export format**: JSON? SQLite? Both? Dolt clone URL?

5. **Approval workflow**: Keep current PendingDiff flow or simplify?

6. **Module-scoped Honcho sessions**: Keep this pattern from greenfield plan?

7. **Error handling**: Retry strategies, failure modes?

8. **Observability**: Logging, metrics, tracing approach?

---

## Implementation Phases (High-Level)

These phases are directional. Detailed steps TBD per phase.

### Phase 1: Foundation
- Set up Dolt storage layer
- Implement basic block CRUD
- Remove Letta dependencies from storage

### Phase 2: Agno Integration
- Create Agno agent factory
- Migrate tools to RunContext pattern
- Basic chat endpoint working

### Phase 3: Streaming Pipeline
- New OpenWebUI pipe
- SSE streaming from Agno
- Honcho persistence integration

### Phase 4: Background Agents
- Scheduler/trigger infrastructure
- Migrate background task patterns
- Observable task execution

### Phase 5: Polish & Migration
- Data migration tooling
- Export functionality
- Documentation
- Testing

---

## References

### Existing Research
- `thoughts/shared/research/2026-01-16-agno-vs-frameworkless-evaluation.md`
- `thoughts/shared/research/2026-01-16-letta-to-agno-migration.md`
- `thoughts/shared/plans/2026-01-16-letta-to-agno-greenfield.md`

### This Conversation
Key decisions made:
- Dolt for knowledge layer (blocks, diffs, preferences)
- Honcho for conversation layer (messages, dialectic)
- Agno for agent runtime (stateless, RunContext)
- Background agent manager as first-class component
- User data ownership as core ethos
- OpenWebUI + web-based as MVP focus
- Future extensibility (MCP, API) kept in mind but not MVP

### External
- [Agno Documentation](https://docs.agno.com)
- [Dolt Documentation](https://docs.dolthub.com)
- [Honcho Documentation](https://docs.honcho.dev)

---

## Next Steps

1. Review this document and flag any disagreements
2. Make decisions on open questions
3. Detail Phase 1 implementation plan
4. Begin implementation

---

*Last updated: 2026-01-24*
*Status: Draft - awaiting review*
