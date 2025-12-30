# Letta Archival Memory Research Report
## Comprehensive Guide for RAG Implementation

---

## Summary

Letta's archival memory is a **semantically searchable vector database** that serves as an agent's external knowledge repository—enabling storage and retrieval of large amounts of information outside the immediate context window. Unlike traditional RAG systems that passively retrieve documents, Letta agents **actively manage** their archival memory through built-in tools (`archival_memory_insert`, `archival_memory_search`), deciding what to store and when to retrieve it. The system supports both **agent-initiated operations** (tool calls during conversations) and **developer-initiated operations** (SDK/API calls for programmatic document loading).

**Current Version Context:** This documentation applies to the current Letta platform (evolved from the MemGPT research project). The SDK is distributed as `letta-client` (Python) and `@letta-ai/letta-client` (TypeScript).

---

## 1. Detailed API Reference

### 1.1 Core SDK Operations (Python)

```python
from letta_client import Letta

# Initialize client
client = Letta(api_key="LETTA_API_KEY")
# OR for self-hosted:
# client = Letta(base_url="http://localhost:8283")
```

### 1.2 Complete Archival Memory Operations

#### **Insert Passage**
```python
# Insert a single memory/passage into archival memory
client.agents.passages.insert(
    agent_id=agent.id,
    content="The Voight-Kampff test requires a minimum of 20 cross-referenced questions",
    tags=["technical", "testing", "protocol"]
)
```

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent_id` | string | Yes | The agent's unique identifier |
| `content` | string | Yes | Text content to store |
| `tags` | list[str] | No | Tags for filtering and organization |
| `created_at` | datetime | No | Backdate the memory (SDK only) |

#### **Search Passages**
```python
# Search archival memory semantically
results = client.agents.passages.search(
    agent_id=agent.id,
    query="testing procedures",
    tags=["protocol"],  # Optional: filter by tags
    page=0              # Pagination (0-indexed)
)
```

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent_id` | string | Yes | The agent's unique identifier |
| `query` | string | Yes | Search query (semantic + keyword) |
| `tags` | list[str] | No | Filter results by tags |
| `page` | int | No | Page number (default: 0) |
| `start_datetime` | datetime | No | Filter by creation time (start) |
| `end_datetime` | datetime | No | Filter by creation time (end) |

**Search Response Format:**
```python
# Each result contains:
{
    "content": "The stored text",
    "tags": ["associated", "tags"],
    "timestamp": "2025-01-15T10:30:00Z",
    "rrf_score": 0.85,      # Reciprocal Rank Fusion score
    "vector_rank": 2,        # Semantic search rank
    "fts_rank": 1            # Full-text search rank
}
```

#### **List All Passages**
```python
# List all archival memories with pagination
passages = client.agents.passages.list(
    agent_id=agent.id,
    limit=100,              # Max results to return
    ascending=True          # Sort order by timestamp
)
```

#### **Get Specific Passage**
```python
# Retrieve a specific passage by ID
passage = client.agents.passages.get(
    agent_id=agent.id,
    passage_id=passage_id
)
```

#### **Delete Passage**
```python
# Delete a specific passage
client.agents.passages.delete(
    agent_id=agent.id,
    passage_id=passage_id
)
```

### 1.3 REST API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/v1/agents/{agent_id}/archival` | Insert archival memory |
| `GET` | `/v1/agents/{agent_id}/archival` | List passages |
| `GET` | `/v1/agents/{agent_id}/archival/{passage_id}` | Get specific passage |
| `DELETE` | `/v1/agents/{agent_id}/archival/{passage_id}` | Delete passage |
| `POST` | `/v1/passages/search` | Search across passages |

---

## 2. Document Loading & Insertion

### 2.1 Two Approaches for Loading Documents

**Approach A: Direct Passage Insertion (Programmatic)**
- Best for: Pre-processed text, facts, structured data
- You handle: Chunking, preprocessing
- Agent access: Immediate via `archival_memory_search`

**Approach B: Folders/Data Sources (File Upload)**
- Best for: PDFs, docs, markdown files
- Letta handles: Chunking, embedding automatically
- Agent access: Via filesystem tools (`open_file`, `search_file`, `grep_file`)

### 2.2 Direct Passage Insertion Pattern

```python
from letta_client import Letta

client = Letta(api_key="LETTA_API_KEY")

# Create agent with embedding model
agent = client.agents.create(
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-3-small",
    memory_blocks=[
        {"label": "human", "value": "User information here"},
        {"label": "persona", "value": "Agent persona here"}
    ]
)

# Load multiple documents/facts
documents = [
    {"content": "Fact 1 about the topic", "tags": ["category1"]},
    {"content": "Fact 2 about the topic", "tags": ["category1", "important"]},
    {"content": "Reference material section 3", "tags": ["reference"]},
]

for doc in documents:
    client.agents.passages.insert(
        agent_id=agent.id,
        content=doc["content"],
        tags=doc.get("tags", [])
    )

print(f"Loaded {len(documents)} passages into agent archival memory")
```

### 2.3 File-Based Loading (Folders API)

```python
import time
from letta_client import Letta

client = Letta(api_key="LETTA_API_KEY")

# Step 1: Create a folder with embedding config
folder = client.folders.create(
    name="my_knowledge_base",
    embedding="openai/text-embedding-3-small"
)

# Step 2: Upload file (async processing)
with open("document.pdf", "rb") as f:
    upload_job = client.folders.files.upload(
        folder_id=folder.id,
        file=f
    )

# Step 3: Wait for processing to complete
while True:
    job = client.jobs.retrieve(upload_job.id)
    if job.status == "completed":
        break
    elif job.status == "failed":
        raise ValueError(f"Job failed: {job.metadata}")
    print(f"Processing status: {job.status}")
    time.sleep(1)

# Step 4: Attach folder to agent
client.agents.folders.attach(
    agent_id=agent.id,
    folder_id=folder.id
)

# Verify files and passages
files = client.folders.files.list(folder_id=folder.id)
passages = client.folders.passages.list(folder_id=folder.id)
print(f"Files: {len(files)}, Passages: {len(passages)}")
```

**Supported File Formats:**
- PDF documents
- Text files (.txt)
- Markdown files (.md)
- Word documents (.docx)
- HTML files

---

## 3. Search & Retrieval Mechanisms

### 3.1 Hybrid Search System

Letta uses **hybrid search** combining:
1. **Semantic Search** - Vector similarity using embeddings
2. **Full-Text Search** - Keyword matching
3. **Reciprocal Rank Fusion (RRF)** - Combines rankings from both methods

### 3.2 Agent-Side Tools (During Conversations)

```python
# What agents call autonomously:

# INSERT - Store new information
archival_memory_insert(
    content="User prefers Python for data science projects",
    tags=["preferences", "programming", "user_info"]
)

# SEARCH - Retrieve relevant information
results = archival_memory_search(
    query="user programming preferences",
    tags=["preferences"],  # Optional filter
    page=0                  # Pagination
)
```

### 3.3 Query Types That Work Well

```python
# Natural language questions (best performance)
archival_memory_search(query="How does the authentication system work?")

# Keywords
archival_memory_search(query="API rate limits")

# Concept-based (semantic understanding)
archival_memory_search(query="artificial memories")
# Will find: "experimental replicant with implanted memories"

# Time-filtered queries
archival_memory_search(
    query="meeting notes",
    start_datetime="2025-09-29T00:00:00",
    end_datetime="2025-09-30T23:59:59"
)
```

### 3.4 Tag-Based Organization

```python
# Common tag patterns:
tags = [
    "user_info", "professional", "personal_history",
    "documentation", "technical", "reference",
    "conversation", "milestone", "event",
    "company_policy", "procedure", "guideline"
]

# Insert with multiple tags
client.agents.passages.insert(
    agent_id=agent.id,
    content="Nexus-6 replicants have a four-year lifespan",
    tags=["technical", "replicant", "nexus-6"]
)

# Search with tag filter
results = client.agents.passages.search(
    agent_id=agent.id,
    query="how long do replicants live",
    tags=["technical"]
)
```

---

## 4. Configuration Options

### 4.1 Embedding Models

**Letta Cloud:**
- Fixed to `text-embedding-3-small` (OpenAI)
- Cannot be changed

**Self-Hosted:**
- Configurable at agent creation
- Default: `text-embedding-3-small`
- Pinned to agent (changing requires re-embedding all data)

```python
# Set embedding at agent creation
agent = client.agents.create(
    model="openai/gpt-4o-mini",
    embedding="openai/text-embedding-3-small",  # Or other supported models
    memory_blocks=[...]
)
```

### 4.2 Vector Database Backends

| Environment | Backend | Notes |
|-------------|---------|-------|
| Letta Cloud | TurboPuffer | Extremely fast, scales to 100k+ memories |
| Self-hosted | pgvector (PostgreSQL) | Good performance with proper indexing |
| Letta Desktop | SQLite + vector extensions | Suitable for personal use |

### 4.3 Chunking Configuration

**For Folder/File uploads:**
- Automatic chunking handled by Letta
- Files split into passages with embeddings
- Chunk size is system-managed (not configurable via SDK)

**For Direct Insertion:**
- You control chunking manually
- Recommended: Keep passages atomic (single fact/concept)
- No technical limit on passage size, but smaller = better retrieval

### 4.4 Changing Embedding Models (Self-Hosted)

⚠️ **Warning: Destructive Operation**

```python
# 1. Export all archival memories
passages = client.agents.passages.list(agent_id=agent.id, limit=10000)
exported = [{"content": p.content, "tags": p.tags} for p in passages]

# 2. Delete all archival memories
for p in passages:
    client.agents.passages.delete(agent_id=agent.id, passage_id=p.id)

# 3. Update agent embedding model (requires agent recreation or API call)
# ... update embedding config ...

# 4. Re-insert all memories (they'll be re-embedded)
for item in exported:
    client.agents.passages.insert(
        agent_id=agent.id,
        content=item["content"],
        tags=item["tags"]
    )
```

---

## 5. Common Patterns & Best Practices

### 5.1 Bulk Document Loading Pattern

```python
def load_documents_to_agent(client, agent_id, documents, batch_size=50):
    """
    Load multiple documents into agent's archival memory.
    
    Args:
        client: Letta client instance
        agent_id: Target agent ID
        documents: List of dicts with 'content' and optional 'tags'
        batch_size: Documents to process before status update
    """
    total = len(documents)
    for i, doc in enumerate(documents):
        client.agents.passages.insert(
            agent_id=agent_id,
            content=doc["content"],
            tags=doc.get("tags", [])
        )
        if (i + 1) % batch_size == 0:
            print(f"Loaded {i + 1}/{total} documents")
    
    print(f"Completed loading {total} documents")

# Usage
documents = [
    {"content": "Fact 1...", "tags": ["category"]},
    {"content": "Fact 2...", "tags": ["category"]},
    # ...
]
load_documents_to_agent(client, agent.id, documents)
```

### 5.2 Document Chunking Helper

```python
def chunk_document(text, chunk_size=500, overlap=50):
    """
    Split document into overlapping chunks for better retrieval.
    """
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap
    return chunks

def load_large_document(client, agent_id, text, base_tags=None):
    """Load a large document by chunking it."""
    chunks = chunk_document(text)
    for i, chunk in enumerate(chunks):
        tags = (base_tags or []) + [f"chunk_{i}"]
        client.agents.passages.insert(
            agent_id=agent_id,
            content=chunk,
            tags=tags
        )
    return len(chunks)
```

### 5.3 RAG Agent Creation Pattern

```python
from letta_client import Letta

def create_rag_agent(client, name, knowledge_base_docs):
    """
    Create an agent with pre-loaded knowledge base.
    """
    # Create agent with archival memory tools enabled by default
    agent = client.agents.create(
        name=name,
        model="openai/gpt-4o-mini",
        embedding="openai/text-embedding-3-small",
        memory_blocks=[
            {
                "label": "human",
                "value": "No information known about the user yet."
            },
            {
                "label": "persona", 
                "value": f"I am a knowledgeable assistant with access to a specialized knowledge base containing {len(knowledge_base_docs)} documents."
            }
        ]
    )
    
    # Load knowledge base
    for doc in knowledge_base_docs:
        client.agents.passages.insert(
            agent_id=agent.id,
            content=doc["content"],
            tags=doc.get("tags", ["knowledge_base"])
        )
    
    return agent

# Usage
docs = [
    {"content": "Technical specification 1...", "tags": ["spec", "v1"]},
    {"content": "Technical specification 2...", "tags": ["spec", "v2"]},
]
agent = create_rag_agent(client, "docs_assistant", docs)
```

### 5.4 Archival Memory vs Memory Blocks Decision Guide

| Use Case | Archival Memory | Memory Blocks |
|----------|----------------|---------------|
| Large document corpus | ✅ | ❌ |
| User preferences | ❌ | ✅ |
| Frequently accessed facts | ❌ | ✅ |
| Historical records | ✅ | ❌ |
| Agent persona | ❌ | ✅ |
| Reference documentation | ✅ | ❌ |
| Current working state | ❌ | ✅ |

**Best Practice:** Use both together—memory blocks hold the "executive summary" while archival memory holds full details.

---

## 6. Key Limitations & Considerations

### 6.1 Important Limitations

1. **Agent Cannot Delete/Modify Archival Memories**
   - Agents can only insert and search
   - Deletion requires SDK/API (developer action)

2. **Embedding Model Lock-in**
   - Changing requires re-embedding all data
   - Plan embedding choice carefully at start

3. **File Processing is Async**
   - Folder uploads create background jobs
   - Must poll for completion status

4. **Search Result Pagination**
   - Results are paginated (default page size varies)
   - Agent may need prompting to iterate through pages

5. **Data Source Re-attachment**
   - New data added to sources doesn't auto-sync
   - Must detach and re-attach to update agent

### 6.2 Performance Considerations

- **Letta Cloud**: Optimized for high-scale (100k+ memories)
- **Self-hosted**: Requires proper PostgreSQL indexing
- **Search is always fast** regardless of archive size

### 6.3 Security Notes

- Archival memory is agent-scoped (not shared by default)
- For shared knowledge bases, use Data Sources attached to multiple agents
- All data persists across sessions and server restarts

---

## 7. Key Resources & Links

### Official Documentation
- **Archival Memory Guide**: https://docs.letta.com/guides/agents/archival-memory/
- **Searching & Querying**: https://docs.letta.com/guides/agents/archival-search/
- **Best Practices**: https://docs.letta.com/guides/agents/archival-best-practices/
- **Data Sources (Legacy)**: https://docs.letta.com/guides/agents/sources/
- **Filesystem/Folders**: https://docs.letta.com/guides/agents/filesystem/
- **Memory Overview**: https://docs.letta.com/guides/agents/memory/
- **API Reference**: https://docs.letta.com/api

### SDK & GitHub
- **Python SDK (letta-client)**: https://github.com/letta-ai/letta-python
- **Main Letta Repository**: https://github.com/letta-ai/letta
- **AI Memory SDK (Experimental)**: https://github.com/letta-ai/ai-memory-sdk

### Research Background
- **MemGPT Paper Concepts**: https://docs.letta.com/concepts/memgpt/
- **Letta Research Background**: https://docs.letta.com/concepts/letta/

### Community
- **Discord**: https://discord.gg/letta
- **Developer Forum**: https://forum.letta.com

---

## Version Information

- **Documentation Date**: December 2024
- **SDK Package**: `letta-client` (Python), `@letta-ai/letta-client` (TypeScript)
- **Default Embedding**: `openai/text-embedding-3-small`
- **Supported Models**: OpenAI, Anthropic, Google Gemini, and more

---

*This research was compiled from official Letta documentation, GitHub repositories, and API references. For the most up-to-date information, always refer to the official documentation at docs.letta.com.*