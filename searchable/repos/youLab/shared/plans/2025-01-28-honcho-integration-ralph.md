# Honcho Integration for Ralph Architecture

## Overview

Integrate Honcho into the Ralph (Agno + Dolt) architecture to enable:
1. **Fire-and-forget message persistence** - Messages stored in Honcho for dialectic queries
2. **Dialectic query tool** - Agent can ask Honcho for insights about students

## Current State Analysis

### What Exists

**HonchoClient** (`src/ralph/honcho.py:44-133`):
- `persist_message(user_id, chat_id, message, is_user)` - Async persistence with peer/session hierarchy
- `query_dialectic(user_id, question)` - Returns `DialecticResponse` with insights
- `persist_message_fire_and_forget()` - Fire-and-forget wrapper at module level
- Proper peer naming: `student_{user_id}` for user messages, `tutor` for assistant messages
- Session naming: `chat_{chat_id}`

**Server Flow** (`src/ralph/server.py:137-259`):
- Per-request agent creation with workspace-scoped tools
- SSE streaming of agent responses
- Memory context loaded from Dolt and injected into agent instructions

### What's Missing

1. **Message persistence not called** - Server never calls `persist_message_fire_and_forget()`
2. **No Agno-compatible dialectic tool** - Existing `QueryHonchoTool` is OpenHands-based
3. **Response accumulation** - Need to capture full assistant response during streaming

### Key Discoveries

- Agno has native `RunContext` injection: tools can accept `run_context: RunContext` parameter
- `RunContext` provides `user_id`, `session_id`, and `dependencies` dict
- Agent can pass `user_id` and `dependencies` to `arun()` method
- Toolkits inherit from `agno.tools.Toolkit` and register tools via constructor

## Desired End State

After implementation:

1. Every user message is persisted to Honcho (fire-and-forget) before agent processing
2. Every assistant response is persisted to Honcho (fire-and-forget) after streaming completes
3. Agent has access to `query_honcho` tool that uses `RunContext.user_id` to query insights
4. No changes to pipe.py - all changes in server.py and new toolkit

### Verification:

```bash
# Check Honcho has messages (via Honcho CLI or API)
# After a chat, messages should appear under student_{user_id} peer

# Test dialectic tool via chat
User: "What do you know about me?"
Agent should use query_honcho tool and return insights
```

## What We're NOT Doing

- Not modifying the pipe.py (OpenWebUI client)
- Not changing the Dolt memory block system
- Not adding Honcho persistence for tool calls (only user/assistant messages)
- Not blocking on Honcho failures (fire-and-forget with warning logs)
- Not removing the legacy OpenHands-based `QueryHonchoTool` (can coexist)

## Implementation Approach

Use Agno's native `RunContext` for passing user context to tools. Create a new `HonchoTools` toolkit that registers a `query_student` function accepting `run_context: RunContext`. Pass `user_id` and `dependencies` when calling `agent.arun()`.

For message persistence, accumulate response chunks during streaming and persist both user message (before) and assistant message (after).

---

## Phase 1: Fire-and-Forget Message Persistence

### Overview

Add message persistence calls to the chat endpoint. User messages are persisted before agent processing. Assistant responses are accumulated during streaming and persisted after completion.

### Changes Required:

#### 1. Update server.py imports

**File**: `src/ralph/server.py`
**Changes**: Add import for fire-and-forget persistence

```python
# Add to imports section (around line 12)
from ralph.honcho import persist_message_fire_and_forget
```

#### 2. Persist user message before agent processing

**File**: `src/ralph/server.py`
**Changes**: Add persistence call before agent.arun()

After line 236 (after prompt is constructed), before the streaming generator:

```python
    # Persist user message to Honcho (fire-and-forget)
    last_user_message = request.messages[-1].content if request.messages else ""
    persist_message_fire_and_forget(
        user_id=request.user_id,
        chat_id=request.chat_id,
        message=last_user_message,
        is_user=True,
    )
```

#### 3. Accumulate and persist assistant response

**File**: `src/ralph/server.py`
**Changes**: Accumulate chunks during streaming, persist after completion

Modify the streaming generator (lines 238-258):

```python
    async def generate() -> AsyncGenerator[str, None]:
        yield f"data: {json.dumps({'type': 'status', 'content': 'Thinking...'})}\n\n"

        # Accumulate response for persistence
        response_chunks: list[str] = []

        try:
            async for chunk in await agent.arun(prompt, stream=True):
                if chunk.content:
                    response_chunks.append(chunk.content)
                    yield f"data: {json.dumps({'type': 'message', 'content': chunk.content})}\n\n"
            yield f"data: {json.dumps({'type': 'done'})}\n\n"

            # Persist assistant response to Honcho (fire-and-forget)
            full_response = "".join(response_chunks)
            if full_response:
                persist_message_fire_and_forget(
                    user_id=request.user_id,
                    chat_id=request.chat_id,
                    message=full_response,
                    is_user=False,
                )
        except Exception as e:
            logger.exception("Error during agent execution")
            yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `uv run basedpyright src/ralph/`
- [x] Linting passes: `uv run ruff check src/ralph/`
- [x] Server starts without errors: `uv run ralph-server`

#### Manual Verification:
- [ ] Send a chat message through OpenWebUI
- [ ] Check Honcho has the user message (via Honcho dashboard or API)
- [ ] Check Honcho has the assistant response
- [ ] Verify messages appear under correct peer/session

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that messages are being persisted correctly before proceeding to Phase 2.

---

## Phase 2: Agno-Compatible Honcho Toolkit

### Overview

Create a new `HonchoTools` toolkit that uses Agno's native `RunContext` injection to access `user_id`. This toolkit provides a `query_student` function the agent can call to get insights about the current student.

### Changes Required:

#### 1. Create new Agno toolkit

**File**: `src/ralph/tools/honcho_tools.py` (new file)
**Changes**: Create Agno-compatible toolkit

```python
"""Honcho tools for Agno agents."""

from __future__ import annotations

import asyncio
import logging
from typing import TYPE_CHECKING

from agno.tools import Toolkit

if TYPE_CHECKING:
    from agno.run.response import RunContext

from ralph.honcho import get_honcho

logger = logging.getLogger(__name__)


class HonchoTools(Toolkit):
    """Toolkit for querying Honcho about students.

    This toolkit provides tools for the agent to query Honcho's dialectic
    system for insights about the current student based on their conversation
    history and context.
    """

    def __init__(self, **kwargs):
        """Initialize HonchoTools."""
        tools = [self.query_student]
        super().__init__(name="honcho_tools", tools=tools, **kwargs)

    def query_student(self, run_context: RunContext, question: str) -> str:
        """Query Honcho for insights about the current student.

        Use this tool when you need to understand the student better, recall
        previous interactions, or get context about their learning journey.

        Args:
            question: A natural language question about the student.
                     Examples: "What topics has this student struggled with?",
                     "What is their learning style?", "What were we discussing last time?"

        Returns:
            Insights about the student from Honcho's dialectic system,
            or an error message if the query fails.
        """
        # Get user_id from RunContext (injected by Agno)
        user_id = run_context.user_id
        if not user_id:
            # Fallback: try dependencies
            deps = run_context.dependencies or {}
            user_id = deps.get("user_id")

        if not user_id:
            return "Unable to identify student. No user context available."

        logger.info(f"Querying Honcho for user {user_id}: {question}")

        try:
            # Run async query in sync context (Agno tools are sync by default)
            honcho = get_honcho()

            # Create event loop if needed (tool may be called from sync context)
            try:
                loop = asyncio.get_running_loop()
                # If we're in an async context, use create_task
                future = asyncio.ensure_future(honcho.query_dialectic(user_id, question))
                # This is tricky - we need to await somehow
                # For now, use run_until_complete in a new thread
                import concurrent.futures
                with concurrent.futures.ThreadPoolExecutor() as pool:
                    result = pool.submit(
                        asyncio.run,
                        honcho.query_dialectic(user_id, question)
                    ).result(timeout=30)
            except RuntimeError:
                # No running loop, can use asyncio.run directly
                result = asyncio.run(honcho.query_dialectic(user_id, question))

            if result is None:
                return "No insights available for this student yet."

            return result.insight

        except Exception as e:
            logger.warning(f"Honcho query failed: {e}")
            return f"Unable to retrieve student insights: {str(e)}"
```

#### 2. Update tools __init__.py

**File**: `src/ralph/tools/__init__.py`
**Changes**: Export the new toolkit

```python
"""Ralph tools package."""

from ralph.tools.honcho_tools import HonchoTools

__all__ = ["HonchoTools"]
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `uv run basedpyright src/ralph/`
- [x] Linting passes: `uv run ruff check src/ralph/`
- [x] Import works: `python -c "from ralph.tools import HonchoTools"`

#### Manual Verification:
- [x] N/A - tool not registered with agent yet

**Implementation Note**: This phase creates the toolkit but doesn't register it. Proceed to Phase 3 to integrate with the agent.

---

## Phase 3: Tool Registration with Agent

### Overview

Register the `HonchoTools` toolkit with the Agno agent and pass `user_id` via the agent's `arun()` method so `RunContext` is populated.

### Changes Required:

#### 1. Import HonchoTools in server.py

**File**: `src/ralph/server.py`
**Changes**: Add import for the new toolkit

```python
# Add to imports section
from ralph.tools import HonchoTools
```

#### 2. Register toolkit with agent

**File**: `src/ralph/server.py`
**Changes**: Add HonchoTools to agent's tools list

Modify agent creation (around line 200-211):

```python
    agent = Agent(
        model=OpenRouter(
            id=settings.openrouter_model,
            api_key=settings.openrouter_api_key,
        ),
        tools=[
            strip_agno_fields(ShellTools(base_dir=workspace)),
            strip_agno_fields(FileTools(base_dir=workspace)),
            HonchoTools(),  # Add Honcho query tool
        ],
        instructions=instructions,
        markdown=True,
    )
```

#### 3. Pass user context to arun()

**File**: `src/ralph/server.py`
**Changes**: Pass user_id and dependencies to agent.arun()

Modify the arun() call (around line 245):

```python
            async for chunk in await agent.arun(
                prompt,
                stream=True,
                user_id=request.user_id,
                session_id=request.chat_id,
                dependencies={
                    "user_id": request.user_id,
                    "chat_id": request.chat_id,
                },
            ):
```

#### 4. Update agent instructions to mention the tool

**File**: `src/ralph/server.py`
**Changes**: Add tool guidance to base instructions

Update the base instructions (around line 157-162):

```python
    base_instructions = f"""You are a helpful AI tutor assistant.
Your workspace is: {workspace}
You can read and write files, and execute shell commands within this workspace.

You also have access to a `query_student` tool that lets you ask questions about
the current student's learning history and context. Use this tool when you need
to recall previous interactions, understand their learning style, or get context
about what you've discussed before.

Always be helpful, encouraging, and focused on the student's learning goals."""
```

### Success Criteria:

#### Automated Verification:
- [x] Type checking passes: `uv run basedpyright src/ralph/`
- [x] Linting passes: `uv run ruff check src/ralph/`
- [x] Server starts without errors: `uv run ralph-server`

#### Manual Verification:
- [ ] Send a message asking about previous conversations
- [ ] Agent should use the `query_student` tool
- [ ] Tool should return insights (or "no insights yet" for new users)
- [ ] Verify no errors in server logs

**Implementation Note**: After completing this phase and verifying the tool works end-to-end, the integration is complete.

---

## Testing Strategy

### Unit Tests:

1. **HonchoTools toolkit tests** (`tests/ralph/tools/test_honcho_tools.py`):
   - Test `query_student` with mocked `RunContext`
   - Test fallback to `dependencies` when `user_id` is None
   - Test error handling when Honcho client fails

2. **Message persistence tests**:
   - Test `persist_message_fire_and_forget` is called with correct args
   - Test response accumulation during streaming

### Integration Tests:

1. **End-to-end chat with Honcho**:
   - Start server with Honcho configured
   - Send chat message
   - Verify messages appear in Honcho
   - Send query about student
   - Verify agent uses tool and returns response

### Manual Testing Steps:

1. Start Dolt: `docker compose up -d dolt`
2. Start Ralph server: `uv run ralph-server`
3. Open OpenWebUI and send a test message
4. Check Honcho dashboard/API for persisted messages
5. Ask the agent "What do you know about me?" or similar
6. Verify agent uses the query_student tool
7. Check server logs for any warnings/errors

---

## Performance Considerations

- **Fire-and-forget persistence**: Messages are persisted asynchronously without blocking the response stream
- **Honcho query timeout**: 30-second timeout prevents hanging if Honcho is slow
- **Per-request agent creation**: Agent is created fresh for each request, no state leakage
- **Toolkit instantiation**: `HonchoTools()` is lightweight, no connection pooling needed

---

## Migration Notes

- No database migrations required
- No changes to existing Dolt memory block system
- Existing `QueryHonchoTool` (OpenHands-based) can remain for backwards compatibility
- Honcho environment variables already configured in `config.py`

---

## References

- Current Ralph server: `src/ralph/server.py`
- Existing Honcho client: `src/ralph/honcho.py`
- Agno Toolkit docs: https://docs.agno.com/basics/tools/creating-tools/toolkits
- Agno RunContext: https://docs.agno.com/reference/agents/agent
