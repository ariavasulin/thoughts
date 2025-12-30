# Honcho by Plastic Labs - Comprehensive Technical Research

**Research Date:** December 26, 2025  
**Sources:** GitHub Repository, Official Documentation (docs.honcho.dev), Blog Posts, SDK Documentation

---

## Executive Summary

Honcho is an AI-native memory library for building agents with "social cognition" and theory-of-mind capabilities. It provides state-of-the-art long-term memory for AI agents while going beyond simple storage by reasoning about stored data to build rich psychological profiles of users (called "Peers").

**Key Value Proposition:** Agents using Honcho have perfect recall and understand who they are, who they're interacting with, what happened, and when it happened—all without developer configuration.

---

## 1. DIALECTIC API

### What is the Dialectic Endpoint?

The Dialectic API is Honcho's flagship interface for querying insights about peers. It's a natural language endpoint that allows your agent to "chat" with Honcho about any user/peer in your system.

**Endpoint:** `POST /peers/{peer_id}/chat`

> *"This is a regular API endpoint that takes natural language requests to get data about the Peer. This robust design lets us use this single endpoint for all cases where extra personalization or information about the Peer is necessary. A developer's application can treat Honcho as an oracle to the Peer and consult it when necessary."*  
> — GitHub README

### How to Query the Dialectic API

**Python SDK (High-Level):**
```python
from honcho import Honcho

honcho = Honcho()
alice = honcho.peer("alice")

# Basic query about a user's global representation
response = alice.chat("What does this user prefer for learning styles?")

# Query about another peer (what alice knows about bob)
response = alice.chat("What do I know about Bob?", target="bob")

# Query scoped to a specific session
response = alice.chat("What happened in our conversation?", session="session-123")

# Streaming response
response = alice.chat("How should I explain this concept?", stream=True)
```

**Lower-Level SDK (honcho-core):**
```python
from honcho_core import Honcho

client = Honcho(
    api_key=os.environ.get("HONCHO_API_KEY"),
    environment="production"
)

# Using the core client for direct API access
response = client.workspaces.peers.chat(
    workspace_id="my-workspace",
    peer_id="alice",
    query="What learning styles does this user respond to best?"
)
```

### Types of Questions You Can Ask

Based on official documentation, example queries include:

| Query Type | Example Question |
|------------|------------------|
| Learning preferences | "What's the best way to explain technical concepts to this user?" |
| Personality traits | "Is this user more task-oriented or relationship-oriented?" |
| Behavioral patterns | "What time of day is this user most engaged?" |
| Communication preferences | "How does this user prefer to receive feedback?" |
| Values & beliefs | "What are this user's core values based on our conversations?" |
| Predictions | "How would this user likely respond to X?" |

### Response Format

The Dialectic API returns natural language responses synthesized from:
- The peer's **working representation** (derived facts)
- **Peer cards** (structured user profile information)
- **Session context** (if scoped)
- **Semantic search** results over stored observations

**Response Structure:**
```python
# The response is a string containing synthesized insights
response = alice.chat("What do we know about Bob?")
# Returns: "Bob is passionate about cooking, specifically Italian cuisine..."
```

For streaming:
```python
for chunk in alice.chat("Tell me about this user", stream=True):
    print(chunk, end="")
```

### Latency Characteristics

**Dialectic Endpoint:**
- Involves LLM inference (uses Anthropic by default)
- As of release notes (07.24.25): *"Dialectic is ~40% faster + better performance"*
- Suitable for runtime personalization but has LLM latency overhead

**Working Representation (Low-Latency Alternative):**
```python
# For latency-sensitive use cases, use working_rep instead
rep = alice.working_rep(search_query="preferences", search_top_k=10)

# Or get from session context
rep = session.working_rep("alice", target="bob")
```

> *"For low-latency use cases, Honcho provides access to a `get_working_representation` endpoint that returns a static document with insights about a Peer in the context of a particular session. Use this to quickly add context to a prompt without having to wait for an LLM response."*  
> — GitHub README

### Dialectic's Relationship to Message Persistence

The Dialectic API operates on data that has been:
1. **Persisted** via `session.add_messages()`
2. **Processed** by Honcho's background "deriver" workers
3. **Stored** as observations in reserved Collections

**Flow:**
```
Messages → Honcho Storage → Background Deriver → Observations/Facts → Dialectic Query
```

The deriver runs asynchronously, so newly added messages may take a moment before they're reflected in Dialectic responses.

---

## 2. SETUP & AUTHENTICATION

### Setting Up a Honcho App/Project (Hosted Version)

**Step 1: Sign Up**
- Go to [app.honcho.dev](https://app.honcho.dev)
- Create an account and join/create an organization
- Each organization gets dedicated infrastructure

**Step 2: Activate Subscription**
- Navigate to Billing page to activate subscription
- Monitor deployment on Instance Status page
- Wait for all systems to show green check marks

**Step 3: Generate API Keys**
- Go to API Keys page
- Create keys scoped to:
  - **Admin level** (full instance access)
  - **Workspace level** (scoped to specific workspace)
  - **Peer/Session level** (granular scoping)

### Authentication Model

**For Hosted Service (api.honcho.dev):**
```python
from honcho import Honcho

# Production with API key
honcho = Honcho(
    workspace_id="my-app-name",
    api_key="your-api-key",        # or use HONCHO_API_KEY env var
    environment="production",       # Points to api.honcho.dev
    base_url="https://api.honcho.dev"  # Optional, defaults based on environment
)
```

**Environment Variables:**
```bash
HONCHO_API_KEY=your-api-key
HONCHO_BASE_URL=https://api.honcho.dev
HONCHO_WORKSPACE_ID=my-app-name
```

**For Demo Server (No Auth):**
```python
# Demo server - no authentication required
honcho = Honcho(environment="demo")  # Points to demo.honcho.dev
```

> *"When you first install the SDKs they will be ready to go, pointing at https://demo.honcho.dev which is a demo server of Honcho. This server has no authentication, no SLA, and should only be used for testing and getting familiar with Honcho."*

**Self-Hosted (JWT Auth):**
```bash
# Generate JWT secret
python scripts/generate_jwt_secret.py

# Set in .env
AUTH_USE_AUTH=true
AUTH_JWT_SECRET=<generated_secret>
```

### Creating/Managing Users and Sessions

**Creating Peers (Users/Agents):**
```python
# Lazy creation - no API call until first use
alice = honcho.peer("alice")
assistant = honcho.peer("assistant")

# Immediate creation with configuration
alice = honcho.peer(
    "alice",
    config={"role": "user", "active": True},
    metadata={"location": "NYC", "tier": "premium"}
)

# List all peers
peers = honcho.get_peers()
```

**Creating Sessions:**
```python
# Lazy creation
session = honcho.session("conversation-1")

# With configuration
session = honcho.session(
    "meeting-1",
    config={"type": "support", "priority": "high"}
)

# Add peers to session
session.add_peers([alice, assistant])

# With observation configuration
from honcho import SessionPeerConfig
session.add_peers([
    (alice, SessionPeerConfig(observe_others=True, observe_me=True))
])

# List sessions
sessions = honcho.get_sessions()
```

### Persisting Messages to Honcho

```python
# Add messages to a session
session.add_messages([
    alice.message("Hey, can you help me with my math homework?"),
    assistant.message("Absolutely! Send me your first problem."),
    alice.message("What's the derivative of x^2?"),
    assistant.message("The derivative of x² is 2x.")
])

# With metadata and custom timestamps
session.add_messages([
    alice.message(
        "I prefer visual explanations",
        metadata={"topic": "preferences", "source": "explicit"},
        created_at="2024-01-15T10:30:00Z"  # Custom timestamp for imports
    )
])

# Upload files to create messages
messages = session.upload_file(
    file=open("document.pdf", "rb"),
    peer="user",
    metadata={"source": "upload"}
)
```

---

## 3. DATA MODEL

### Core Entities

```
Workspaces
├── Peers ←──────────────────┐
│   ├── Sessions             │
│   ├── Collections          │
│   │   └── Documents        │
│   └── Observations         │
│                            │
└── Sessions ←───────────────┤ (many-to-many)
    ├── Peers ───────────────┘
    └── Messages (session-level)
```

### Entity Definitions

| Entity | Description |
|--------|-------------|
| **Workspace** | Top-level container (formerly "Apps"). Isolates data between use cases; provides multi-tenancy. |
| **Peer** | Any participant—human user or AI agent. Unified model enabling multi-participant interactions. |
| **Session** | A conversation/thread between Peers. Supports many-to-many relationships with Peers. |
| **Message** | Atomic communication unit within a Session. Labeled by source Peer. |
| **Collection** | Named group of Documents for vector storage (used internally for Peer representations). |
| **Document** | Vector-embedded data stored in a Collection. |
| **Observation** | Derived facts about a Peer, extracted from Messages by the deriver. |

### Entity Relationships

**Peers ↔ Sessions (Many-to-Many):**
- A Peer can participate in multiple Sessions
- A Session can have multiple Peers
- Each Peer-Session relationship has configurable observation settings

**Observations Model:**
```python
# Self-observations (what Honcho knows about alice)
self_obs = alice.observations
obs_list = self_obs.list()
results = self_obs.query("food preferences")

# Observations of another peer (what alice knows about bob)
bob_obs = alice.observations_of("bob")
bob_obs_list = bob_obs.list()

# Create observations manually (for imports)
alice.observations_of("bob").create([
    {"content": "User prefers dark mode", "session_id": "session-1"},
    {"content": "User works late at night", "session_id": "session-1"}
])
```

### Metadata on Messages

```python
# Add metadata when creating messages
session.add_messages([
    alice.message(
        "Let's discuss the budget",
        metadata={
            "topic": "finance",
            "priority": "high",
            "department": "engineering"
        }
    ),
    assistant.message(
        "I'll prepare the report",
        metadata={
            "action_item": True,
            "due_date": "2024-01-15"
        }
    )
])

# Filter messages by metadata
finance_messages = session.get_messages(
    filters={"metadata": {"topic": "finance"}}
)
action_items = session.get_messages(
    filters={"metadata": {"action_item": True}}
)
```

---

## 4. PYTHON SDK

### Installation

```bash
# Using pip
pip install honcho-ai

# Using uv
uv add honcho-ai

# Using poetry
poetry add honcho-ai
```

**Note:** There are two SDKs:
- `honcho-ai` - High-level ergonomic SDK (recommended)
- `honcho-core` - Lower-level generated SDK via Stainless

### Initializing the Python Client

```python
from honcho import Honcho

# Basic initialization (uses environment variables)
honcho = Honcho(workspace_id="my-app-name")

# Full configuration
honcho = Honcho(
    workspace_id="my-app-name",
    api_key="my-api-key",           # or HONCHO_API_KEY env var
    environment="production",        # "local", "demo", or "production"
    base_url="https://api.honcho.dev",
    timeout=30.0,
    max_retries=3
)

# Using honcho-core (lower level)
from honcho_core import Honcho as HonchoCore

client = HonchoCore(
    api_key=os.environ.get("HONCHO_API_KEY"),
    environment="production"
)
```

### Adding Messages

```python
# Create peers
alice = honcho.peer("alice")
assistant = honcho.peer("assistant")

# Create session
session = honcho.session("conversation-1")

# Add single messages
session.add_messages([alice.message("Hello!")])

# Add batch messages
session.add_messages([
    alice.message("What's the weather?"),
    assistant.message("It's sunny and 75°F!"),
    alice.message("Thanks!"),
    assistant.message("You're welcome!")
])

# With metadata
session.add_messages([
    alice.message(
        "I prefer email communication",
        metadata={"preference_type": "communication"}
    )
])

# Get messages
messages = session.get_messages()
for msg in messages:
    print(f"{msg.peer_id}: {msg.content}")
```

### Querying the Dialectic API from Python

```python
# Simple query
response = alice.chat("What does this user like to do?")
print(response)

# Query with target (what alice knows about bob)
response = alice.chat("What do I know about Bob?", target="bob")

# Query scoped to session
response = alice.chat(
    "What happened in our last conversation?",
    session="session-123"
)

# Streaming response
for chunk in alice.chat("Tell me about this user's preferences", stream=True):
    print(chunk, end="", flush=True)
```

### Complete Integration Example

```python
import openai
from honcho import Honcho

# Initialize clients
honcho = Honcho(
    workspace_id="my-tutoring-app",
    api_key="your-api-key",
    environment="production"
)
openai_client = openai.OpenAI()

# Create peers
student = honcho.peer("student-123")
tutor = honcho.peer("ai-tutor")

# Create or get session
session = honcho.session("tutoring-session-1")
session.add_peers([student, tutor])

def chat_with_personalization(user_input: str) -> str:
    # 1. Add user message to Honcho
    session.add_messages([student.message(user_input)])
    
    # 2. Query Dialectic for personalization insights
    learning_style = student.chat("What learning style works best for this student?")
    
    # 3. Get conversation context
    context = session.get_context(
        tokens=3000,
        summary=True,
        peer_target="student-123",
        peer_perspective="ai-tutor"
    )
    
    # 4. Convert to OpenAI format and add personalization
    messages = context.to_openai(assistant=tutor)
    system_prompt = f"""You are a personalized AI tutor.

Learning style insights: {learning_style}

Adapt your explanations based on these insights about the student."""
    
    messages.insert(0, {"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": user_input})
    
    # 5. Get response from OpenAI
    response = openai_client.chat.completions.create(
        model="gpt-4",
        messages=messages
    )
    ai_response = response.choices[0].message.content
    
    # 6. Store AI response in Honcho
    session.add_messages([tutor.message(ai_response)])
    
    return ai_response

# Example usage
while True:
    user_input = input("Student: ")
    if user_input.lower() in ['quit', 'exit']:
        break
    response = chat_with_personalization(user_input)
    print(f"Tutor: {response}")
```

### Getting Context for LLM Completions

```python
# Basic context retrieval
context = session.get_context()
openai_messages = context.to_openai(assistant=tutor)

# With token limit and summary
context = session.get_context(
    tokens=2000,
    summary=True
)

# With peer representation included
context = session.get_context(
    tokens=2000,
    peer_target="student",           # Get representation for this peer
    peer_perspective="tutor",        # From this peer's perspective
    last_user_message="Help me understand derivatives",  # For semantic search
    limit_to_session=True,           # Limit to session observations only
    search_top_k=10,                 # Number of semantic results
    search_max_distance=0.8,         # Max semantic distance
    include_most_derived=True,       # Include most derived observations
    max_observations=25              # Max observations to include
)

# Context structure
print(context.messages)          # List of Message objects
print(context.summary)           # Summary object (if requested)
print(context.peer_representation)  # String representation (if peer_target set)
print(context.peer_card)         # List of strings (if peer_target set)

# Convert to different formats
openai_msgs = context.to_openai(assistant=tutor)
anthropic_msgs = context.to_anthropic(assistant=tutor)
```

### Working Representation (Low-Latency Alternative)

```python
# Get cached working representation
rep = alice.working_rep()

# With semantic search
rep = alice.working_rep(
    search_query="food preferences",
    search_top_k=10
)

# From session context (what alice knows about bob)
rep = session.working_rep("alice", target="bob")

# Get peer context (representation + peer card in one call)
context = alice.get_context()
print(context.representation)  # Working representation string
print(context.peer_card)       # List of profile facts
```

---

## 5. ADDITIONAL NOTES & CONSIDERATIONS

### Environments

| Environment | URL | Auth Required | Use Case |
|-------------|-----|---------------|----------|
| `demo` | demo.honcho.dev | No | Testing, getting familiar |
| `local` | localhost:8000 | Configurable | Self-hosted development |
| `production` | api.honcho.dev | Yes (API Key) | Production workloads |

### Background Processing

Honcho uses asynchronous workers ("derivers") for expensive operations:
- **Representation updates**: Deriving facts from messages
- **Session summarization**: Creating conversation summaries
- **Peer card generation**: Building structured user profiles

> *"As Messages and Sessions are created for Peers, Honcho will asynchronously reason about peer psychology to derive facts about them and store them in reserved Collections."*

**LLM Providers Used:**
- **Dialectic**: Anthropic (default)
- **Summary/Deriver**: Google Gemini (default)
- **Query Generation**: Groq (default)
- **Embeddings**: OpenAI (optional, if EMBED_MESSAGES=true)

### Theory of Mind Configuration

Control how peers observe each other within sessions:

```python
from honcho import SessionPeerConfig

# Configure observation settings
config = SessionPeerConfig(
    observe_others=False,  # Don't form theory-of-mind of other peers
    observe_me=True        # Allow others to model this peer
)

session.add_peers([(alice, config)])
```

### Limitations & Considerations

1. **Latency**: Dialectic involves LLM inference; use `working_rep` for low-latency needs
2. **Async Processing**: Newly added messages may take time to appear in Dialectic responses
3. **Demo Server**: No SLA, no auth, testing only
4. **Self-Hosting**: Requires Postgres with pgvector, multiple LLM API keys

### Useful Links

| Resource | URL |
|----------|-----|
| Documentation | https://docs.honcho.dev |
| GitHub Repository | https://github.com/plastic-labs/honcho |
| Python SDK (PyPI) | https://pypi.org/project/honcho-ai/ |
| Dashboard | https://app.honcho.dev |
| Discord Community | https://discord.gg/honcho |
| Blog | https://blog.plasticlabs.ai |

---

## Quick Reference Code Snippets

### Minimal Setup
```python
from honcho import Honcho

honcho = Honcho(environment="demo", workspace_id="test-app")
user = honcho.peer("user-1")
assistant = honcho.peer("assistant")
session = honcho.session("session-1")

session.add_messages([
    user.message("Hello!"),
    assistant.message("Hi there! How can I help?")
])
```

### Query Dialectic
```python
insight = user.chat("What communication style does this user prefer?")
```

### Get Context for LLM
```python
context = session.get_context(tokens=2000, summary=True)
messages = context.to_openai(assistant=assistant)
```

### Production Setup
```python
honcho = Honcho(
    workspace_id="production-app",
    api_key=os.environ["HONCHO_API_KEY"],
    environment="production"
)
```