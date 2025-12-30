---
date: 2025-12-30T00:47:39-08:00
researcher: ariasulin
git_commit: ea3995572138e025e64c01f1f6e5c5a0012c8005
branch: main
repository: YouLab
topic: "Strategy RAG Agent Integration with HTTP Service"
tags: [research, rag, archival-memory, letta, http-service, strategy-agent]
status: complete
last_updated: 2025-12-30
last_updated_by: ariasulin
---

# Research: Strategy RAG Agent Integration with HTTP Service

**Date**: 2025-12-30T00:47:39-08:00
**Researcher**: ariasulin
**Git Commit**: ea3995572138e025e64c01f1f6e5c5a0012c8005
**Branch**: main
**Repository**: YouLab

## Research Question

How to create a simple RAG (Retrieval Augmented Generation) agent integrated with the existing HTTP service that can store YouLab project documentation and answer strategic questions about the platform?

## Summary

The existing codebase already has archival memory infrastructure (`insert_archival_memory`, `get_archival_memory`) integrated into the MemoryManager. Adding a strategy RAG agent requires: (1) creating a new agent template with RAG-aware persona, (2) adding HTTP endpoints for document upload and strategy queries, and (3) optionally updating to the newer Letta SDK passages API for better chunking/tagging support. The implementation can follow existing patterns exactly, requiring ~200 lines of code across 4 files.

## Detailed Findings

### 1. Current Archival Memory Infrastructure

The codebase already supports archival memory operations through the MemoryManager class.

**Existing API Usage** (`src/letta_starter/memory/manager.py`):

| Method | Location | Purpose |
|--------|----------|---------|
| `insert_archival_memory()` | Lines 234, 263 | Insert text into vector store |
| `get_archival_memory()` | Line 284 | Search archival with query |

**Current Insert Pattern** (`memory/manager.py:234-236`):
```python
self.client.insert_archival_memory(
    agent_id=self.agent_id,
    memory=archival_entry,  # String with timestamp header
)
```

**Current Search Pattern** (`memory/manager.py:284-288`):
```python
results = self.client.get_archival_memory(
    agent_id=self.agent_id,
    query=query,
    limit=limit,
)
return [r.text for r in results] if results else []
```

**Public API**: `BaseAgent.search_memory(query, limit)` at `agents/base.py:201-213` exposes archival search.

### 2. New Letta SDK (letta_client) Passages API

The newer Letta SDK provides enhanced archival operations with tagging and hybrid search.

**Insert with Tags**:
```python
from letta_client import Letta
client = Letta(base_url="http://localhost:8283")

client.agents.passages.insert(
    agent_id=agent.id,
    content="Document content here",
    tags=["architecture", "phase1"],  # For filtering
)
```

**Search with Filters**:
```python
results = client.agents.passages.search(
    agent_id=agent.id,
    query="What is the architecture?",
    tags=["architecture"],  # Optional filter
    limit=5,
)
```

**Key Differences from Current API**:
- Supports tags for categorization
- Content field instead of memory field
- Hybrid search (semantic + keyword) built-in
- Automatic chunking with configurable overlap

### 3. HTTP Service Architecture

The service follows consistent patterns that a strategy endpoint should match.

**Endpoint Pattern** (`server/main.py`):
```python
@app.post("/endpoint", response_model=ResponseSchema, status_code=201)
async def create_something(
    request: RequestSchema,
    manager: AgentManager = Depends(get_agent_manager),
) -> ResponseSchema:
    try:
        result = manager.operation(...)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    except Exception as e:
        log.exception("operation_failed")
        raise HTTPException(status_code=503, detail="Service unavailable")
    return ResponseSchema(**result)
```

**Schema Pattern** (`server/schemas.py`):
```python
class RequestSchema(BaseModel):
    field: str = Field(..., description="Required field")
    optional_field: str | None = Field(default=None, description="Optional")
```

**Error Handling**:
- `ValueError` (validation) → 400 Bad Request
- `Exception` (infrastructure) → 503 Service Unavailable
- Not found → 404 Not Found

### 4. AgentManager Extension Points

**Option A: Add Strategy Template** (Recommended)

Create `STRATEGY_TEMPLATE` in `agents/templates.py`:
```python
STRATEGY_TEMPLATE = AgentTemplate(
    type_id="strategy",
    display_name="Strategy Advisor",
    description="RAG-enabled project strategy advisor",
    persona=PersonaBlock(
        name="YouLab Strategy Advisor",
        role="Project strategy and architecture advisor",
        capabilities=[
            "Search project documentation",
            "Answer architecture questions",
            "Explain design decisions",
        ],
        constraints=[
            "Always search archival memory before answering",
            "Cite sources when referencing documentation",
        ],
    ),
    human=HumanBlock(),
)
templates.register(STRATEGY_TEMPLATE)
```

Then create agents with `agent_type="strategy"`:
```python
manager.create_agent(user_id="shared", agent_type="strategy")
```

**Option B: Dedicated StrategyManager Class**

For strategy-specific operations (document upload, RAG search):
```python
class StrategyManager:
    def __init__(self, letta_base_url: str):
        self.letta_base_url = letta_base_url
        self._client = None
        self._agent_id: str | None = None

    def ensure_agent(self) -> str:
        """Get or create the singleton strategy agent."""
        ...

    def upload_document(self, content: str, tags: list[str]) -> None:
        """Upload content to archival memory."""
        ...

    def ask(self, question: str) -> str:
        """Query the strategy agent with RAG."""
        ...
```

### 5. Configuration Requirements

Current `ServiceSettings` (`config/settings.py:106-154`) would need:

```python
# Strategy Agent Configuration
strategy_agent_model: str = Field(
    default="anthropic/claude-sonnet-4-20250514",
    description="Model for strategy agent",
)
strategy_embedding_model: str = Field(
    default="openai/text-embedding-3-small",
    description="Embedding model for RAG",
)
```

Environment variables:
- `YOULAB_SERVICE_STRATEGY_AGENT_MODEL`
- `YOULAB_SERVICE_STRATEGY_EMBEDDING_MODEL`

### 6. SDK Compatibility

**Current codebase uses**: `letta_client.Letta` imported at `server/agents.py:6`

**API methods available**:
- `client.create_agent()` - confirmed working
- `client.list_agents()` - confirmed working
- `client.send_message()` - confirmed working
- `client.insert_archival_memory()` - used in MemoryManager
- `client.get_archival_memory()` - used in MemoryManager

**New passages API**: Requires checking if `client.agents.passages.*` is available in installed version. If not, the older `insert_archival_memory`/`get_archival_memory` API still works for basic RAG.

## Implementation Plan

### Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| `server/strategy.py` | Create | StrategyManager class |
| `server/schemas.py` | Modify | Add strategy request/response schemas |
| `server/main.py` | Modify | Add /strategy endpoints |
| `agents/templates.py` | Modify | Add STRATEGY_TEMPLATE |
| `config/settings.py` | Modify | Add strategy settings (optional) |

### Proposed Endpoints

```
POST /strategy/documents
  - Upload content to strategy agent's archival memory
  - Body: { "content": "...", "tags": ["..."] }
  - Returns: { "success": true, "passage_count": 1 }

POST /strategy/ask
  - Query the strategy agent
  - Body: { "question": "What is YouLab's architecture?" }
  - Returns: { "response": "...", "sources": [...] }

GET /strategy/documents
  - List uploaded documents (optional)
  - Query: ?tag=architecture&limit=10
  - Returns: { "documents": [...] }
```

### Minimal POC (~100 lines)

For fastest validation, create a standalone script that:
1. Creates strategy agent with RAG-aware persona
2. Loads a few YouLab docs into archival
3. Sends a test query

Then integrate into HTTP service once validated.

## Code References

- `src/letta_starter/server/agents.py:6` - Letta client import
- `src/letta_starter/server/agents.py:106-113` - Agent creation pattern
- `src/letta_starter/server/agents.py:158-165` - Message sending pattern
- `src/letta_starter/memory/manager.py:234-236` - Archival insert
- `src/letta_starter/memory/manager.py:284-288` - Archival search
- `src/letta_starter/agents/templates.py:19-47` - TUTOR_TEMPLATE example
- `src/letta_starter/server/schemas.py:11-58` - Schema patterns

## Architecture Documentation

### Current Data Flow

```
POST /agents → AgentManager.create_agent() → client.create_agent()
POST /chat → AgentManager.send_message() → client.send_message()
```

### Proposed Strategy Flow

```
POST /strategy/documents → StrategyManager.upload_document() → client.agents.passages.insert()
POST /strategy/ask → StrategyManager.ask() → client.send_message() (agent searches archival)
```

### Agent Persona Design

The strategy agent's persona should instruct it to search archival memory:

```
You are a YouLab project strategist with comprehensive knowledge stored in your archival memory.

CRITICAL: Before answering ANY question about YouLab:
1. Use archival_memory_search to find relevant documentation
2. Cite the sources you found in your response
3. If no relevant documents found, say so explicitly

Your archival memory contains:
- Architecture documentation
- Roadmap and phase plans
- Design decisions and rationale
- Technical specifications
```

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2025-12-29-phase-1-combined-implementation.md` - Phase 1 HTTP service completed
- `thoughts/global/shared/reference/letta-archival-memory.md` - Letta archival memory API reference
- `thoughts/global/shared/reference/letta-archival-memory-2.md` - Additional Letta passages API details

## Related Research

- Phase 1 implementation plan establishes HTTP service patterns
- Letta archival memory docs confirm passages API availability

## Open Questions

1. **SDK Version**: Does the installed `letta_client` have `client.agents.passages.*` API or only older `insert_archival_memory`?
   - Resolution: Check with `pip show letta-client` and test API availability

2. **Singleton vs Per-User**: Should strategy agent be shared (one for all) or per-user?
   - Recommendation: Start with singleton shared agent for simplicity

3. **Embedding Model**: Does Letta server need OpenAI API key for embeddings, or does it use local embeddings?
   - Resolution: Check Letta server configuration

4. **Document Chunking**: Should we chunk documents before insert, or let Letta handle it?
   - Recommendation: Let Letta handle it (passages API has built-in chunking)
