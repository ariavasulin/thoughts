---
date: 2026-01-16T14:40:07-08:00
researcher: ariasulin
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Letta to Agno Migration Research"
tags: [research, migration, agno, letta, tools, honcho, streaming, rag]
status: complete
last_updated: 2026-01-16
last_updated_by: ariasulin
---

# Research: Letta to Agno Migration

**Date**: 2026-01-16T14:40:07-08:00
**Researcher**: ariasulin
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

How to migrate YouLab from Letta to Agno framework, focusing on:
1. Adapting existing tools (edit_memory_block, query_honcho, advance_lesson)
2. Integrating Honcho for message persistence
3. Using Git-backed blocks as Agno knowledge/memory
4. Streaming integration with OpenWebUI pipeline
5. Model provider flexibility (Claude, OpenAI, etc.)
6. Agentic RAG setup for future use

## Summary

Migration from Letta to Agno is feasible and offers significant benefits: simpler tool definitions (plain Python functions), native multi-provider support, built-in agentic RAG, and better streaming control. The key migration challenges are:

1. **Tools**: Direct mapping - Agno uses decorated Python functions with automatic JSON schema generation
2. **Honcho Integration**: Wrap existing `HonchoClient` methods as Agno tools with `RunContext` for state
3. **Git-backed Blocks**: Load as Agno knowledge base or inject via `session_state` / custom retriever
4. **Streaming**: Use Agno's `run_response_stream()` with SSE conversion layer
5. **Model Providers**: Native support for Claude, OpenAI, Groq, Ollama, and 20+ providers
6. **RAG**: Built-in agentic RAG with 15+ vector DB integrations

## Detailed Findings

### 1. Current YouLab Architecture (Letta)

#### Tool System (`src/youlab_server/tools/`)

YouLab has three main tools registered with Letta:

**edit_memory_block** (`tools/memory.py:33-114`)
```python
def edit_memory_block(
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
    agent_state: dict[str, Any] | None = None,  # Injected by Letta
) -> str
```
- Creates `PendingDiff` for user approval
- Uses global `_letta_client` set at startup
- Extracts `agent_id`, `user_id` from `agent_state`

**query_honcho** (`tools/dialectic.py:39-103`)
```python
def query_honcho(
    question: str,
    session_scope: str = "all",
    agent_state: dict[str, Any] | None = None,
) -> str
```
- Queries conversation history via Honcho dialectic
- Uses global `_honcho_client` and `_user_context` mapping
- Returns insight string

**advance_lesson** (`tools/curriculum.py:27-129`)
```python
def advance_lesson(
    reason: str,
    agent_state: dict[str, Any] | None = None,
) -> str
```
- Updates journey block and returns next lesson opening
- Uses global `_letta_client` and `_agent_course_map`

**Sandbox Pattern**: Duplicate HTTP-based versions exist in `tools/sandbox.py` for Docker compatibility. They POST to YouLab server endpoints instead of importing modules directly.

#### Honcho Integration (`src/youlab_server/honcho/`)

- **Client**: `HonchoClient` class with lazy SDK loading
- **Workspace**: Single `"youlab"` workspace
- **Peers**: `student_{user_id}` per student, shared `tutor` peer
- **Sessions**: `chat_{chat_id}` per conversation
- **Persistence**: Fire-and-forget via `create_persist_task()`
- **Queries**: `query_dialectic()` for theory-of-mind modeling

#### Memory Blocks (`src/youlab_server/storage/`)

- **Schema**: TOML-defined in `config/courses/*/course.toml`
- **Storage**: Git-backed markdown files with YAML frontmatter
- **Models**: Dynamic Pydantic classes generated at runtime
- **Diffs**: `PendingDiff` system for agent-proposed changes
- **Sync**: `UserBlockManager` syncs git → Letta blocks

#### Streaming Pipeline (`src/youlab_server/pipelines/letta_pipe.py`)

```
OpenWebUI → Pipe.pipe() → POST /chat/stream → AgentManager.stream_message() → Letta SDK
```
- Uses `httpx-sse` for SSE client
- Transforms Letta chunks to OpenWebUI events
- Persists messages to Honcho during streaming

---

### 2. Agno Framework Overview

Agno (formerly Phidata) is a lightweight Python framework for multi-modal AI agents.

**Key Features**:
- Plain Python functions as tools (automatic JSON schema generation)
- Built-in agentic RAG with 15+ vector DB integrations
- Native multi-provider support (Claude, OpenAI, Groq, Ollama, etc.)
- Session state management via `RunContext`
- Streaming with `run_response_stream()`

**Installation**:
```bash
pip install agno
```

---

### 3. Tool Migration

#### Agno Tool Definition

Agno converts Python functions to tools automatically:

```python
from agno.tools import tool
from agno.run import RunContext

@tool(
    name="edit_memory_block",
    description="Propose changes to a memory block field",
    show_result=True,
)
def edit_memory_block(
    run_context: RunContext,  # Injected by Agno
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
) -> str:
    """Propose changes to a user's memory block.

    Args:
        block: Memory block label (e.g., "student", "journey")
        field: Specific field to update within the block
        content: Content to add or replace
        strategy: Merge strategy - "append", "replace", or "full_replace"
        reasoning: Explanation for the change

    Returns:
        Confirmation message with diff ID for user review.
    """
    # Get user context from session state
    user_id = run_context.session_state.get("user_id")
    agent_id = run_context.session_state.get("agent_id")

    # Import and use existing logic
    from youlab_server.storage import get_storage_manager
    from youlab_server.storage.blocks import UserBlockManager

    storage_manager = get_storage_manager()
    user_storage = storage_manager.get_user_storage(user_id)
    manager = UserBlockManager(user_id, user_storage)

    diff = manager.propose_edit(
        agent_id=agent_id,
        block_label=block,
        field=field,
        proposed_value=content,
        operation=strategy,
        reasoning=reasoning,
    )

    return f"Proposed change to {block}.{field} (ID: {diff.id[:8]}). User will review."
```

#### query_honcho Migration

```python
from agno.tools import tool
from agno.run import RunContext

@tool(name="query_honcho", description="Query conversation history for student insights")
def query_honcho(
    run_context: RunContext,
    question: str,
    session_scope: str = "all",
) -> str:
    """Query conversation history to understand the student better.

    Args:
        question: Natural language question about the student
        session_scope: Which sessions to query - "all", "recent", or "current"

    Returns:
        Insight about the student based on conversation history.
    """
    import asyncio
    from youlab_server.honcho.client import HonchoClient, SessionScope

    user_id = run_context.session_state.get("user_id")
    honcho_client = run_context.session_state.get("honcho_client")

    if not honcho_client:
        return "Honcho not available for conversation queries."

    scope = SessionScope(session_scope) if session_scope in SessionScope.__members__.values() else SessionScope.ALL

    # Run async query
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(
        honcho_client.query_dialectic(user_id, question, scope)
    )

    return result.insight if result else "Unable to query conversation history."
```

#### advance_lesson Migration

```python
from agno.tools import tool
from agno.run import RunContext

@tool(name="advance_lesson", description="Advance student to next curriculum lesson")
def advance_lesson(
    run_context: RunContext,
    reason: str,
) -> str:
    """Request advancement to the next lesson in the curriculum.

    Args:
        reason: Explanation for why the student is ready to advance

    Returns:
        Next lesson opening message or error.
    """
    from youlab_server.curriculum import get_curriculum_registry
    from youlab_server.storage.blocks import UserBlockManager

    user_id = run_context.session_state.get("user_id")
    course_id = run_context.session_state.get("course_id")

    # Reuse existing curriculum advancement logic
    registry = get_curriculum_registry()
    course = registry.get_course(course_id)

    # ... (existing _do_advance_lesson logic)

    return f"Advanced to next lesson. {opening_message}"
```

#### Tool Registration

```python
from agno.agent import Agent
from agno.models.anthropic import Claude

agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    tools=[edit_memory_block, query_honcho, advance_lesson],
    session_state={
        "user_id": user_id,
        "agent_id": agent_id,
        "course_id": course_id,
        "honcho_client": honcho_client,
    },
)
```

---

### 4. Honcho Integration with Agno

#### Option A: Session State Injection

Pass `HonchoClient` via session state (shown above).

#### Option B: Custom Toolkit

```python
from agno.tools import Toolkit
from youlab_server.honcho.client import HonchoClient, SessionScope

class HonchoToolkit(Toolkit):
    def __init__(self, honcho_client: HonchoClient, user_id: str):
        self.honcho = honcho_client
        self.user_id = user_id
        super().__init__(name="honcho", tools=[self.query_dialectic])

    def query_dialectic(self, question: str, session_scope: str = "all") -> str:
        """Query conversation history for insights about the student.

        Args:
            question: Natural language question about the student
            session_scope: Scope of sessions to query

        Returns:
            Insight from conversation history.
        """
        import asyncio
        scope = SessionScope(session_scope) if session_scope in ["all", "recent", "current"] else SessionScope.ALL

        loop = asyncio.get_event_loop()
        result = loop.run_until_complete(
            self.honcho.query_dialectic(self.user_id, question, scope)
        )
        return result.insight if result else "No insight available."

# Usage
agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    tools=[HonchoToolkit(honcho_client, user_id)],
)
```

#### Message Persistence

Keep existing fire-and-forget pattern, called from streaming wrapper:

```python
async def stream_with_persistence(agent, message, user_id, chat_id, honcho_client):
    # Persist user message
    honcho_client.create_persist_task(user_id, chat_id, message, is_user=True)

    full_response = []
    async for chunk in agent.run_response_stream(message):
        full_response.append(chunk.content)
        yield chunk

    # Persist agent response
    response_text = "".join(full_response)
    honcho_client.create_persist_task(user_id, chat_id, response_text, is_user=False)
```

---

### 5. Git-Backed Blocks as Agno Knowledge/Memory

#### Option A: Load Blocks into Session State

```python
from youlab_server.storage.git import GitUserStorage

def load_user_blocks(user_id: str, course_id: str) -> dict[str, str]:
    """Load all user blocks as a dict for session state."""
    storage = GitUserStorage(user_id)
    blocks = {}
    for label in storage.list_blocks():
        blocks[label] = storage.read_block_body(label)
    return blocks

# Create agent with blocks in session state
agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    session_state={
        "user_id": user_id,
        "memory_blocks": load_user_blocks(user_id, course_id),
    },
    instructions=[
        "You have access to memory blocks about the student in session_state['memory_blocks'].",
        "Use this context to personalize your responses.",
    ],
)
```

#### Option B: Custom Knowledge Retriever

```python
from typing import Optional
from agno.agent import Agent
from youlab_server.storage.git import GitUserStorage

def git_block_retriever(
    agent: Agent,
    query: str,
    num_documents: Optional[int] = None,
    **kwargs
) -> Optional[list[dict]]:
    """Retrieve relevant blocks from git storage based on query."""
    user_id = agent.session_state.get("user_id")
    storage = GitUserStorage(user_id)

    results = []
    for label in storage.list_blocks():
        body = storage.read_block_body(label)
        metadata = storage.read_block_metadata(label)

        # Simple keyword matching (could use embeddings for better matching)
        if any(keyword in body.lower() for keyword in query.lower().split()):
            results.append({
                "content": body,
                "meta_data": {
                    "block_label": label,
                    "schema": metadata.get("schema"),
                    "updated_at": metadata.get("updated_at"),
                }
            })

    return results[:num_documents] if num_documents else results

agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    knowledge_retriever=git_block_retriever,
    search_knowledge=True,
)
```

#### Option C: Inject Blocks into System Prompt

```python
def build_system_prompt(base_prompt: str, blocks: dict[str, str]) -> str:
    """Build system prompt with embedded memory blocks."""
    block_context = "\n\n".join([
        f"## {label.title()} Block\n{content}"
        for label, content in blocks.items()
    ])

    return f"""{base_prompt}

# Memory Blocks (Student Context)

{block_context}
"""

agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    system_prompt=build_system_prompt(base_prompt, user_blocks),
)
```

---

### 6. Streaming Integration with OpenWebUI

#### Agno Streaming

```python
from agno.agent import Agent
from agno.models.anthropic import Claude

agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    tools=[...],
    stream=True,  # Enable streaming
)

# Async streaming
async for chunk in agent.arun_response_stream("Hello"):
    print(chunk.content, end="", flush=True)
```

#### SSE Conversion Layer

```python
import json
from typing import AsyncIterator

async def agno_to_sse(
    agent: Agent,
    message: str,
) -> AsyncIterator[str]:
    """Convert Agno streaming chunks to SSE format for OpenWebUI."""

    async for chunk in agent.arun_response_stream(message):
        # Handle different chunk types
        if hasattr(chunk, "tool_calls") and chunk.tool_calls:
            for tool_call in chunk.tool_calls:
                yield f"data: {json.dumps({'type': 'status', 'content': f'Using {tool_call.name}...'})}\n\n"

        if hasattr(chunk, "reasoning") and chunk.reasoning:
            yield f"data: {json.dumps({'type': 'status', 'content': 'Thinking...', 'reasoning': chunk.reasoning})}\n\n"

        if chunk.content:
            yield f"data: {json.dumps({'type': 'message', 'content': chunk.content})}\n\n"

    yield f"data: {json.dumps({'type': 'done'})}\n\n"
```

#### Updated Pipeline Endpoint

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/chat/stream")
async def chat_stream(request: StreamChatRequest):
    agent = get_or_create_agent(request.agent_id)

    # Set session state for this request
    agent.session_state.update({
        "user_id": request.user_id,
        "chat_id": request.chat_id,
    })

    async def generate():
        async for sse_event in agno_to_sse(agent, request.message):
            yield sse_event

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

---

### 7. Model Provider Flexibility

Agno supports 20+ model providers natively:

```python
# Anthropic Claude
from agno.models.anthropic import Claude
agent = Agent(model=Claude(id="claude-sonnet-4-20250514"))

# OpenAI
from agno.models.openai import OpenAIChat
agent = Agent(model=OpenAIChat(id="gpt-4o"))

# Groq (fast inference)
from agno.models.groq import Groq
agent = Agent(model=Groq(id="llama-3.3-70b-versatile"))

# Local via Ollama
from agno.models.ollama import Ollama
agent = Agent(model=Ollama(id="llama3.2"))

# Google Gemini
from agno.models.google import Gemini
agent = Agent(model=Gemini(id="gemini-2.0-flash-exp"))

# AWS Bedrock
from agno.models.aws import AwsBedrock
agent = Agent(model=AwsBedrock(id="anthropic.claude-3-sonnet-20240229-v1:0"))
```

#### Configuration Pattern

```python
from agno.models.anthropic import Claude
from agno.models.openai import OpenAIChat

MODEL_PROVIDERS = {
    "claude-sonnet": lambda: Claude(id="claude-sonnet-4-20250514"),
    "claude-opus": lambda: Claude(id="claude-opus-4-20250514"),
    "gpt-4o": lambda: OpenAIChat(id="gpt-4o"),
    "gpt-4o-mini": lambda: OpenAIChat(id="gpt-4o-mini"),
}

def create_agent(model_name: str, **kwargs) -> Agent:
    model_factory = MODEL_PROVIDERS.get(model_name, MODEL_PROVIDERS["claude-sonnet"])
    return Agent(model=model_factory(), **kwargs)
```

---

### 8. Agentic RAG Setup

#### Basic Setup with PgVector

```python
from agno.agent import Agent
from agno.models.anthropic import Claude
from agno.knowledge.pdf_url import PDFUrlKnowledgeBase
from agno.vectordb.pgvector import PgVector, SearchType

# Create knowledge base
knowledge_base = PDFUrlKnowledgeBase(
    urls=["https://example.com/course-materials.pdf"],
    vector_db=PgVector(
        table_name="youlab_knowledge",
        db_url="postgresql+psycopg://ai:ai@localhost:5532/ai",
        search_type=SearchType.hybrid,  # Semantic + keyword
    ),
)

# Load documents (run once)
knowledge_base.load(recreate=False)

# Create agent with knowledge
agent = Agent(
    model=Claude(id="claude-sonnet-4-20250514"),
    knowledge=knowledge_base,
    search_knowledge=True,  # Enables agentic RAG (agent decides when to search)
    instructions=[
        "Search your knowledge base for course materials when relevant.",
        "Cite sources when using knowledge base information.",
    ],
)
```

#### Hybrid Knowledge: Git Blocks + Vector DB

```python
from agno.agent import Agent
from agno.knowledge.knowledge import Knowledge
from agno.vectordb.pgvector import PgVector

def create_hybrid_agent(user_id: str, course_id: str) -> Agent:
    """Create agent with both git blocks (memory) and vector DB (knowledge)."""

    # Load user-specific blocks
    user_blocks = load_user_blocks(user_id, course_id)

    # Course knowledge base (shared across users)
    course_kb = PDFUrlKnowledgeBase(
        urls=[f"https://youlab.com/courses/{course_id}/materials.pdf"],
        vector_db=PgVector(
            table_name=f"youlab_{course_id}_knowledge",
            db_url=settings.postgres_url,
        ),
    )

    return Agent(
        model=Claude(id="claude-sonnet-4-20250514"),
        knowledge=course_kb,
        search_knowledge=True,
        session_state={
            "user_id": user_id,
            "memory_blocks": user_blocks,
        },
        instructions=[
            "You have personalized memory about this student in session_state['memory_blocks'].",
            "Search the course knowledge base for curriculum materials.",
        ],
    )
```

---

## Migration Roadmap

### Phase 1: Core Agent Migration
1. Create Agno tool wrappers for `edit_memory_block`, `query_honcho`, `advance_lesson`
2. Implement `AgentManager` using Agno `Agent` class
3. Update streaming endpoint to use `agno_to_sse()` converter
4. Test with existing OpenWebUI pipeline

### Phase 2: Honcho Integration
1. Create `HonchoToolkit` class for Agno
2. Integrate message persistence with streaming wrapper
3. Verify dialectic queries work via session state

### Phase 3: Memory Block Integration
1. Load git blocks into session state
2. Implement custom `git_block_retriever` for search
3. Update `edit_memory_block` tool to work with git storage

### Phase 4: Knowledge & RAG
1. Set up PgVector for course materials
2. Create knowledge bases per course
3. Integrate hybrid memory (blocks) + knowledge (RAG)

### Phase 5: Cleanup
1. Remove Letta dependencies
2. Update OpenWebUI pipe to point to new endpoints
3. Migration complete

---

## Code References

- `src/youlab_server/tools/__init__.py:24-25` - Current tool exports
- `src/youlab_server/tools/memory.py:33-114` - edit_memory_block implementation
- `src/youlab_server/tools/dialectic.py:39-103` - query_honcho implementation
- `src/youlab_server/tools/curriculum.py:27-129` - advance_lesson implementation
- `src/youlab_server/honcho/client.py:240-307` - Dialectic query implementation
- `src/youlab_server/pipelines/letta_pipe.py:138-231` - OpenWebUI streaming pipeline
- `src/youlab_server/server/agents.py:410-439` - Current Letta streaming
- `src/youlab_server/storage/git.py:89-142` - Git-backed storage
- `src/youlab_server/storage/blocks.py:19-40` - UserBlockManager

## External Resources

- [Agno Documentation](https://docs.agno.com)
- [Agno GitHub](https://github.com/agno-agi/agno)
- [Agno Tools Guide](https://docs.agno.com/basics/tools/overview)
- [Agno Knowledge/RAG](https://docs.agno.com/basics/knowledge/agents/overview)
- [Agno Vector DBs](https://docs.agno.com/integrations/vectordb)

## Open Questions

1. **Agent State Persistence**: Agno agents are stateless by default. Need to decide:
   - Recreate agent per request with session state injection?
   - Use Agno's built-in storage (if available)?
   - Keep separate agent registry like current implementation?

2. **Sandbox Compatibility**: Current sandbox tools use HTTP endpoints. With Agno:
   - Does Agno support sandboxed execution?
   - Or can we run Agno tools in-process (simpler)?

3. **Letta Block IDs**: Current system uses Letta block IDs for shared blocks. Need to:
   - Migrate shared block pattern to Agno
   - Or simplify to session-state-only approach

4. **Background Agents**: Current background agents use Letta. Migration approach:
   - Migrate to Agno agents with same tools
   - Or keep as separate scheduled jobs calling tools directly
