# Letta Message History & Archival System Research

## Summary

Letta (formerly MemGPT) handles long conversation histories through an **automatic eviction and recursive summarization system** inspired by virtual memory management in operating systems. When the context window fills, old messages are evicted to external storage ("recall memory") and recursively summarized into a rolling summary that's always present in context. Messages are **never deleted**—they remain searchable via tool calls. The archival memory system provides semantic search over agent-curated facts stored in vector databases. Developers have full programmatic control via SDK endpoints for compacting, resetting, listing messages, and importing/exporting agent state through the `.af` (Agent File) format.

---

## 1. Long Conversation History Management

### Automatic Truncation & Eviction

**Key Mechanism:** When the message buffer (FIFO queue) reaches the configured context window limit, Letta automatically:
1. Evicts approximately 70% of older messages to recall storage (external database)
2. Creates/updates a **recursive summary** of evicted messages
3. Places the summary as the first message in the new context window
4. Keeps the most recent messages intact for conversation continuity

**Configurable Parameters:**
- `context_window` - Maximum tokens for the LLM (can be artificially limited below model max)
- `max_message_buffer_length` - Maximum messages before compaction triggers
- `min_message_buffer_length` - Minimum messages to preserve (for continuity)
- `keep_last_n_messages` - Number of recent messages to always keep (env: `letta_summarizer_keep_last_n_messages`)

**Eviction Process:**
```
1. Queue manager detects context at ~70% capacity → appends memory warning message
2. At 100% capacity → flush/eviction triggered
3. Older messages summarized into existing rolling summary (recursive)
4. Evicted messages persisted to recall storage (PostgreSQL/SQLite)
5. Agent retains access via conversation_search and conversation_search_date tools
```

### Context Window Limits

Letta allows you to set a **maximum context window** below the model's native limit:
- Improves performance/stability
- Reduces cost/latency  
- Set via "Advanced" agent settings in ADE or API

```python
# Setting context window limit via SDK
agent = client.agents.update(
    agent_id=agent.id,
    context_window=16000  # Limit to 16k even if model supports 200k
)
```

### What Happens When History Exceeds Context

1. **Memory warning** added to context at ~70% capacity
2. **Automatic compaction** triggers at capacity
3. Rolling summary recreated incorporating new evicted messages
4. Old messages moved to **recall memory** (searchable database)
5. Agent can retrieve via `conversation_search(query)` or `conversation_search_date(start, end)`

**The summary is recursive:** Each compaction summarizes previous summary + newly evicted messages, so older conversations have progressively less influence.

---

## 2. Archival Memory System

### Structure & Organization

**Archival memory** is a semantically searchable database separate from conversation history:

| Aspect | Description |
|--------|-------------|
| **Storage** | Vector database (TurboPuffer on Cloud, pgvector self-hosted, SQLite desktop) |
| **Unit** | "Passages" - individual memory entries |
| **Search** | Semantic similarity via embeddings |
| **Organization** | Tags (agent-managed categories) |
| **Mutability** | Agent-immutable (agents can insert, but can't easily delete—developers can via SDK) |

**Key Characteristics:**
- **Unlimited storage** - No practical size limits
- **Semantic search** - Finds information by meaning, not exact keywords
- **Tagged organization** - Agents can categorize memories with tags
- **Always stats in context** - Agent sees metadata about archival (total count, tags) but not content

### How Agents Interact

**Agent Tools (autonomous):**
- `archival_memory_insert(content, tags)` - Store new information
- `archival_memory_search(query, tags, page)` - Query for relevant memories

```python
# What the agent does (agent tool call)
archival_memory_insert(
    content="User prefers Python for data science projects",
    tags=["preferences", "programming", "data-science"]
)

results = archival_memory_search(
    query="programming language preferences",
    tags=["preferences"],  # Optional filter
    page=0
)
```

**Developer SDK (programmatic):**

```python
# Insert via SDK
client.agents.passages.insert(
    agent_id=agent.id,
    content="The user's API key expires on 2025-06-01",
    tags=["credentials", "important"]
)

# Search via SDK
results = client.agents.passages.search(
    agent_id=agent.id,
    query="API credentials",
    tags=["credentials"],
    page=0
)

# List all passages (paginated)
passages = client.agents.passages.list(
    agent_id=agent.id,
    limit=50,
    search="optional text search",
    ascending=True  # oldest first
)

# Delete a passage
client.agents.passages.delete(agent_id, memory_id)

# Update a passage
client.agents.passages.modify(agent_id, memory_id, content="updated content")
```

### Archival vs Conversation Search (Recall Memory)

| Feature | Archival Memory | Recall Memory (Conversation Search) |
|---------|-----------------|-------------------------------------|
| **Purpose** | Intentional storage of facts/knowledge | Historical retrieval of actual messages |
| **Source** | Agent-curated insertions | Automatic logging of all messages |
| **Search Type** | Semantic (vector similarity) | Full-text + semantic + date range |
| **Curation** | Agent decides what to store | Everything stored automatically |
| **Use Case** | Reference material, learned facts | "What did we discuss last week?" |

---

## 3. Message History API

### Listing Messages

```python
# List message history (paginated)
messages = client.agents.messages.list(
    agent_id=agent.id,
    limit=100,                    # Max messages to return
    before="message-id",          # Cursor for pagination
    after="message-id",           # Cursor for pagination  
    order="desc",                 # 'asc' or 'desc'
    include_errors=False          # Include error messages (debugging)
)
```

**Response includes:**
- `UserMessage` - Messages from users
- `AssistantMessage` - Agent responses
- `ToolCallMessage` - Tool invocation requests
- `ToolReturnMessage` - Tool execution results
- `ReasoningMessage` - Agent's internal chain-of-thought
- `SystemMessage` - Internal system context (not streamed)

### Compacting Message History

```python
# Manually trigger compaction (summarization)
response = client.agents.messages.compact(
    agent_id=agent.id,
    # Optional compaction settings
)
```

**Compaction settings (`compaction_settings` on agent):**
```python
compaction_settings = {
    "model": "openai/gpt-4o-mini",  # Model for summarization
    "clip_chars": 2000,              # Max summary length
    "mode": "sliding_window",        # Summarization technique
    # "keep_percentage": 0.3,        # % of context to keep (sliding window mode)
}
```

### Resetting Message History

```python
# Clear all messages while preserving memory blocks
agent_state = client.agents.messages.reset(
    agent_id=agent.id,
    add_default_initial_messages=True  # Re-add initial system messages
)
```

**What reset preserves:**
- ✅ Core memory blocks (persona, human)
- ✅ Archival memory (passages)
- ✅ Recall memory storage (can still be searched)
- ✅ Agent configuration/tools

**What reset clears:**
- ❌ In-context message history
- ❌ Rolling summary

### Export/Import Conversation State

**Agent File (.af) Format:**
Letta uses the `.af` format for serializing complete agent state:

```python
# Export agent (including conversation history)
with open("agent_backup.af", "wb") as f:
    agent_file = client.agents.export(agent_id)
    f.write(agent_file)

# Import agent
with open("agent_backup.af", "rb") as f:
    agent_state = client.agents.import_file(
        file=f,
        strip_messages=False  # Set True to import without message history
    )
```

**Export via REST API:**
```bash
curl -X GET "https://api.letta.com/v1/agents/{agent_id}/export" \
  -H "Authorization: Bearer $LETTA_API_KEY" \
  -o agent.af
```

**Import with options:**
```python
agent_state = client.agents.import_file(
    file=agent_file_bytes,
    strip_messages=True,      # Remove messages from imported agent
    # Other options available
)
```

---

## 4. Multi-Conversation Patterns

### Core Philosophy: No Threads/Sessions

**Letta explicitly does NOT have threads or sessions.** Each agent has a single perpetual message history:

> "Letta doesn't have the concept of threads or sessions. Instead, there are only stateful agents with a single perpetual message history."

### Recommended Patterns for Multiple Conversations

**Pattern 1: Agent Per User (Recommended)**
```python
# Create separate agents for each user
def get_or_create_user_agent(user_id: str):
    # Try to find existing agent
    agents = client.agents.list(query=f"user_{user_id}")
    if agents:
        return agents[0]
    
    # Create new agent from template
    return client.agents.create(
        name=f"assistant_for_{user_id}",
        model="openai/gpt-4.1",
        memory_blocks=[
            {"label": "human", "value": f"User ID: {user_id}"},
            {"label": "persona", "value": "I am a helpful assistant."}
        ]
    )
```

**Pattern 2: Agent Templates for Fresh Conversations**
```python
# Create template once
template = client.templates.create(
    name="customer_support_v1",
    agent_config={...}
)

# Spawn new agents from template for each "conversation"
new_conversation = client.agents.create_from_template(
    template_id=template.id,
    name=f"conversation_{session_id}"
)
```

**Pattern 3: Shared Memory Blocks**
```python
# Create shared block for organizational knowledge
org_block = client.blocks.create(
    label="organization",
    value="Company: Acme Corp\nPolicies: ...",
    limit=2000
)

# Attach to multiple agents
agent1 = client.agents.create(block_ids=[org_block.id], ...)
agent2 = client.agents.create(block_ids=[org_block.id], ...)
# Updates to org_block visible to both agents
```

### Scaling Long-Running Agents

**Managing Thousands of Messages:**

1. **Automatic context management** - Letta handles eviction/summarization automatically
2. **Configure aggressive compaction** for latency-sensitive apps:
   ```python
   agent = client.agents.update(
       agent_id=agent.id,
       context_window=16000,  # Smaller window = more frequent compaction
       max_message_buffer_length=50,  # Limit in-context messages
   )
   ```

3. **Use recall memory for search** - Agent can search all historical messages:
   ```python
   # Agent tool call
   conversation_search(query="what was the project deadline?")
   conversation_search_date(start_date="2025-01-01", end_date="2025-03-01")
   ```

4. **Periodic archival summarization** - Have agents periodically summarize key facts to archival:
   ```python
   # Prompt agent to consolidate learnings
   client.agents.messages.create(
       agent_id=agent.id,
       input="Please review our recent conversations and update your archival memory with any important facts you've learned."
   )
   ```

5. **Database backend** - Use PostgreSQL for production (not SQLite):
   ```yaml
   # docker-compose.yml
   services:
     letta:
       environment:
         - LETTA_PG_URI=postgresql://user:pass@db:5432/letta
   ```

### Best Practices for Persistent Agents

1. **Memory block hygiene** - Keep blocks concise; agent should actively manage them
2. **Tag archival memories** - Use consistent tagging for organized retrieval
3. **Monitor context window** - Use ADE context viewer to optimize
4. **Implement compaction strategy** - Configure based on your latency/cost requirements
5. **Use identities for multi-user** - Associate agents with user identities via `identities` API
6. **Export regularly** - Back up important agents to `.af` files
7. **Disable memory when appropriate** - Set `memory_enabled=False` for stateless use cases

---

## Key Documentation Links

| Resource | URL |
|----------|-----|
| **Memory Overview** | https://docs.letta.com/guides/agents/memory/ |
| **Memory Blocks** | https://docs.letta.com/guides/agents/memory-blocks/ |
| **Archival Memory** | https://docs.letta.com/guides/agents/archival-memory/ |
| **Archival Search** | https://docs.letta.com/guides/agents/archival-search/ |
| **Context Engineering** | https://docs.letta.com/guides/agents/context-engineering/ |
| **Building Stateful Agents** | https://docs.letta.com/guides/agents/overview/ |
| **Message Types** | https://docs.letta.com/guides/agents/message-types/ |
| **Agent File (.af)** | https://docs.letta.com/guides/agents/agent-file/ |
| **Multi-Agent Systems** | https://docs.letta.com/guides/agents/multi-agent/ |
| **API Reference - Messages** | https://docs.letta.com/api/resources/agents/subresources/messages/ |
| **GitHub Repository** | https://github.com/letta-ai/letta |
| **Agent File Repo** | https://github.com/letta-ai/agent-file |

---

## Limitations & Caveats

1. **No native thread/session concept** - Must use agent-per-user or templates pattern
2. **Recursive summary has recency bias** - Older conversations have less influence over time
3. **Compaction is lossy** - Some detail lost during summarization
4. **Embedding model changes are destructive** - Export archival before changing embedding model
5. **Last-write-wins** - Concurrent block modifications may cause data loss
6. **Rate limits** - Storage operations are rate-limited; batch related data
7. **No built-in message pruning API** - Can reset or import with `strip_messages`, but no selective deletion
8. **Recall memory search limitations** - Only returns top results; may miss relevant older messages

---

## Code Examples

### Complete Agent Lifecycle

```python
from letta_client import Letta
import os

client = Letta(api_key=os.getenv("LETTA_API_KEY"))

# 1. Create agent
agent = client.agents.create(
    model="openai/gpt-4.1",
    embedding="openai/text-embedding-3-small",
    memory_blocks=[
        {"label": "human", "value": "Name: Unknown"},
        {"label": "persona", "value": "I am a helpful assistant."}
    ],
    context_window=32000,  # Limit context
)

# 2. Send messages
response = client.agents.messages.create(
    agent_id=agent.id,
    input="Hi! My name is Sarah and I work in data science."
)

# 3. Check message count
messages = client.agents.messages.list(agent_id=agent.id, limit=100)
print(f"Total messages: {len(messages)}")

# 4. Manually compact if needed
client.agents.messages.compact(agent_id=agent.id)

# 5. Search conversation history (via agent tool)
response = client.agents.messages.create(
    agent_id=agent.id,
    input="What was my name again? Search your conversation history if needed."
)

# 6. Export agent state
agent_file = client.agents.export(agent.id)
with open("sarah_agent.af", "wb") as f:
    f.write(agent_file)

# 7. Reset messages but keep memory
client.agents.messages.reset(
    agent_id=agent.id,
    add_default_initial_messages=True
)

# 8. Verify memory blocks preserved
agent_state = client.agents.retrieve(agent.id)
for block in agent_state.blocks:
    print(f"{block.label}: {block.value[:100]}...")
```

### Archival Memory Management

```python
# Insert passages programmatically
client.agents.passages.insert(
    agent_id=agent.id,
    content="Project Alpha deadline: June 15, 2025",
    tags=["project", "deadline", "alpha"]
)

# List all passages
all_passages = client.agents.passages.list(
    agent_id=agent.id,
    limit=250  # Max 250
)

# Search passages
results = client.agents.passages.search(
    agent_id=agent.id,
    query="project deadlines",
    tags=["deadline"]
)

# Delete old passages
for passage in all_passages:
    if "deprecated" in passage.tags:
        client.agents.passages.delete(agent.id, passage.id)
```

### Multi-User Pattern

```python
def handle_user_message(user_id: str, message: str):
    # Get or create agent for this user
    agents = client.agents.list(
        query=f"user_agent_{user_id}"
    )
    
    if agents:
        agent = agents[0]
    else:
        agent = client.agents.create(
            name=f"user_agent_{user_id}",
            model="openai/gpt-4.1",
            memory_blocks=[
                {"label": "human", "value": f"User ID: {user_id}"},
                {"label": "persona", "value": "I am a personal assistant."}
            ]
        )
    
    # Send message
    response = client.agents.messages.create(
        agent_id=agent.id,
        input=message
    )
    
    # Extract assistant response
    for msg in response.messages:
        if msg.message_type == "assistant_message":
            return msg.content
    
    return "No response generated"
```