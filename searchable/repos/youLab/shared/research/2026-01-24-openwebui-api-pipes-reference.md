---
date: 2026-01-25T02:31:42Z
researcher: Claude Code
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "OpenWebUI API and Pipe System Reference for YouLab v2"
tags: [research, openwebui, api, pipes, streaming, authentication]
status: complete
last_updated: 2026-01-24
last_updated_by: Claude Code
---

# Research: OpenWebUI API and Pipe System Reference for YouLab v2

**Date**: 2026-01-25T02:31:42Z
**Researcher**: Claude Code
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

Comprehensive reference documentation for OpenWebUI API and Pipe system to support YouLab v2 development. Covers Chat CRUD, model management, user/auth endpoints, completion finalization, Pipe architecture/lifecycle, model registration (base vs derived), SSE streaming, user context extraction, and API key management.

## Summary

OpenWebUI provides a comprehensive plugin architecture through its Pipe/Pipeline system, allowing custom "models" that appear in the chat interface. The system is built on FastAPI with SQLAlchemy ORM, supporting SQLite and PostgreSQL. Key integration points for YouLab include:

1. **Pipe System**: Custom Python classes that implement the `pipe()` method receive chat requests with rich context (`__user__`, `__metadata__`, `__event_emitter__`)
2. **Chat API**: Full CRUD operations at `/api/chats/*` with support for folders, tags, sharing, and archiving
3. **Model Registration**: Base models (actual LLM endpoints) vs derived models (virtual models proxying to base with custom params)
4. **SSE Streaming**: Real-time streaming via Server-Sent Events with event types: `status`, `message`, `done`, `error`
5. **Authentication**: JWT tokens with API key support, user context passed via `X-OpenWebUI-User-*` headers

## Detailed Findings

---

## 1. Chat CRUD API (`/api/chats/*`)

**Router**: `OpenWebUI/open-webui/backend/open_webui/routers/chats.py`
**Models**: `OpenWebUI/open-webui/backend/open_webui/models/chats.py`

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` or `/list` | List user's chats (paginated, 60/page) |
| GET | `/search` | Search chats by text/tags/folders |
| GET | `/{id}` | Get chat by ID |
| POST | `/new` | Create new chat |
| POST | `/{id}` | Update chat |
| DELETE | `/{id}` | Delete chat |
| POST | `/{id}/pin` | Toggle pin status |
| POST | `/{id}/archive` | Toggle archive status |
| POST | `/{id}/folder` | Assign to folder |
| POST | `/{id}/share` | Create share link |
| DELETE | `/{id}/share` | Remove share link |
| GET | `/share/{share_id}` | Get shared chat |
| POST | `/{id}/clone` | Clone chat |
| GET | `/{id}/tags` | Get chat tags |
| POST | `/{id}/tags` | Add tag |
| DELETE | `/{id}/tags` | Remove tag |
| POST | `/{id}/messages/{message_id}` | Update message content |

### Chat Data Model (`models/chats.py:35-65`)

```python
class Chat(Base):
    __tablename__ = "chat"

    id: String (PK)
    user_id: String
    title: Text
    chat: JSON  # Contains history.messages map and history.currentId
    created_at: BigInteger
    updated_at: BigInteger
    share_id: Text (unique, nullable)
    archived: Boolean
    pinned: Boolean
    meta: JSON  # Contains tags array
    folder_id: Text (nullable)
```

### Chat JSON Structure

```python
chat = {
    "title": str,
    "history": {
        "messages": {
            "<message_id>": {
                "content": str,
                "role": str,
                "timestamp": int,
                "parentId": str,
                "model": str,
                "files": list,
                "statusHistory": list,
            }
        },
        "currentId": str  # Points to current message in thread
    }
}
```

### Key Operations

**Create Chat** (`chats.py:259-268`):
- Generates UUID for chat ID
- Extracts title from `form_data.chat["title"]` or defaults to "New Chat"
- Returns `ChatResponse` with all fields

**Update Message** (`chats.py:594-641`):
- Upserts message into `chat.history.messages`
- Updates `history.currentId` to point to message
- Emits WebSocket event for real-time updates

**Sharing** (`chats.py:890-944`):
- Creates a copy of chat with `user_id="shared-{chat_id}"`
- Sets `share_id` on original chat for lookup
- Accessible via `/share/{share_id}` endpoint

---

## 2. Model Management API

**Router**: `OpenWebUI/open-webui/backend/open_webui/routers/models.py`
**Models**: `OpenWebUI/open-webui/backend/open_webui/models/models.py`

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/list` | Paginated model list with search |
| GET | `/base` | Get base models only (admin) |
| GET | `/tags` | Extract unique tags from models |
| POST | `/create` | Create new model |
| GET | `/export` | Export models for backup |
| POST | `/import` | Import models |
| POST | `/sync` | Sync models (admin) |
| GET | `/model?id=` | Get model by ID |
| GET | `/model/profile/image?id=` | Get model icon |
| POST | `/model/toggle` | Toggle active state |
| POST | `/model/update` | Update model |
| POST | `/model/delete` | Delete model |
| DELETE | `/delete/all` | Delete all models (admin) |

### Model Schema (`models/models.py:53-103`)

```python
class Model(Base):
    __tablename__ = "model"

    id = Column(Text, primary_key=True)
    user_id = Column(Text)
    base_model_id = Column(Text, nullable=True)  # Key field!
    name = Column(Text)
    params = Column(JSON)  # Model parameters
    meta = Column(JSON)    # Metadata (profile_image_url, description, capabilities, tags)
    access_control = Column(JSON, nullable=True)
    is_active = Column(Boolean, default=True)
    updated_at = Column(BigInteger)
    created_at = Column(BigInteger)
```

### Base vs Derived Models

| Type | `base_model_id` | Behavior |
|------|-----------------|----------|
| **Base Model** | `None` | Actual LLM endpoint, handles requests directly |
| **Derived Model** | Set to base model ID | Virtual model, proxies to base with custom params/meta |

**Base model query** (`models.py:204-209`):
```python
db.query(Model).filter(Model.base_model_id == None).all()
```

**Derived model query** (`models.py:184`):
```python
db.query(Model).filter(Model.base_model_id != None).all()
```

### Access Control System

```python
access_control = {
    # None = public (all users)
    # {} = private (owner only)
    # Custom structure:
    "read": {
        "group_ids": ["group_id1", "group_id2"],
        "user_ids": ["user_id1", "user_id2"]
    },
    "write": {
        "group_ids": ["group_id1"],
        "user_ids": ["user_id1"]
    }
}
```

---

## 3. User/Auth API

**Router**: `OpenWebUI/open-webui/backend/open_webui/routers/auths.py`
**User Router**: `OpenWebUI/open-webui/backend/open_webui/routers/users.py`

### Authentication Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/signin` | Email/password signin |
| POST | `/signup` | User registration |
| POST | `/ldap` | LDAP authentication |
| GET | `/signout` | Sign out |
| POST | `/update/profile` | Update user profile |
| POST | `/update/password` | Update password |
| POST | `/api_key` | Generate API key |
| GET | `/api_key` | Get user's API key |
| DELETE | `/api_key` | Delete API key |

### User Management Endpoints (Admin)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List users (paginated) |
| GET | `/all` | Get all users |
| GET | `/search` | Search users |
| GET | `/{user_id}` | Get user by ID |
| POST | `/{user_id}/update` | Update user |
| DELETE | `/{user_id}` | Delete user |

### Authentication Flow

1. **Signin** (`auths.py:510-634`):
   - Rate limiting: 5 requests/3 minutes per email
   - Password truncated to 72 bytes (bcrypt limit)
   - JWT token created with configurable expiration
   - Token stored in httponly cookie

2. **Token Validation** (`auths.py:104-159`):
   - Token from Authorization header or cookie
   - API keys start with `sk-` prefix
   - Returns `SessionUserResponse` with user data + permissions

### User Context Headers

When `ENABLE_FORWARD_USER_INFO_HEADERS=true`, these headers are added to pipeline requests (`utils/headers.py:4-11`):

```
X-OpenWebUI-User-Name: {url_encoded_name}
X-OpenWebUI-User-Id: {user_uuid}
X-OpenWebUI-User-Email: {email}
X-OpenWebUI-User-Role: {admin|user|pending}
X-OpenWebUI-Chat-Id: {chat_uuid}  # If available
```

### Role System

| Role | Access Level |
|------|--------------|
| `admin` | Full system access |
| `user` | Standard access, feature-based permissions |
| `pending` | Restricted, awaiting approval |

---

## 4. Pipe Architecture & Lifecycle

**Core Files**:
- `OpenWebUI/open-webui/backend/open_webui/functions.py` - Pipe execution engine
- `OpenWebUI/open-webui/backend/open_webui/utils/plugin.py` - Module loading
- `OpenWebUI/open-webui/backend/open_webui/routers/pipelines.py` - Admin endpoints

### Pipe Class Structure

```python
from pydantic import BaseModel, Field
from typing import Callable, Awaitable, Any

class Pipe:
    class Valves(BaseModel):
        """Configuration exposed in admin UI"""
        SERVICE_URL: str = Field(
            default="http://localhost:8100",
            description="Service URL",
        )

    def __init__(self) -> None:
        self.name = "My Pipe"  # Display name in UI
        self.valves = self.Valves()

    async def on_startup(self) -> None:
        """Called when pipe is loaded"""
        pass

    async def on_shutdown(self) -> None:
        """Called when pipe is unloaded"""
        pass

    async def on_valves_updated(self) -> None:
        """Called when admin updates configuration"""
        pass

    async def pipe(
        self,
        body: dict[str, Any],
        __user__: dict[str, Any] | None = None,
        __metadata__: dict[str, Any] | None = None,
        __event_emitter__: Callable[[dict[str, Any]], Awaitable[None]] | None = None,
    ) -> str:
        """Main handler - receives chat requests"""
        messages = body.get("messages", [])
        user_message = messages[-1].get("content", "") if messages else ""

        # Emit status update
        if __event_emitter__:
            await __event_emitter__({
                "type": "status",
                "data": {"description": "Processing...", "done": False}
            })

        # Return response
        return "Hello!"
```

### Pipe Parameters

| Parameter | Type | Contents |
|-----------|------|----------|
| `body` | `dict` | OpenAI-format request: `messages`, `model`, `stream` |
| `__user__` | `dict` | User info: `id`, `email`, `name`, `role`, `valves` (if UserValves defined) |
| `__metadata__` | `dict` | Context: `chat_id`, `session_id`, `message_id`, `files`, `tool_ids`, `variables` |
| `__event_emitter__` | `Callable` | Async function to send real-time updates |
| `__request__` | `Request` | FastAPI Request object |
| `__chat_id__` | `str` | Current chat ID (shorthand) |

### `__metadata__` Full Structure

```python
__metadata__ = {
    "user_id": "uuid-string",
    "chat_id": "uuid-string",
    "message_id": "uuid-string",
    "session_id": "session-string",
    "tool_ids": None | list[str],
    "tool_servers": [],
    "files": [],
    "features": {
        "image_generation": False,
        "code_interpreter": False,
        "web_search": False
    },
    "variables": {
        "{{USER_NAME}}": "username",
        "{{USER_LOCATION}}": "Unknown",
        "{{CURRENT_DATETIME}}": "2026-01-24 12:00:00",
        "{{CURRENT_DATE}}": "2026-01-24",
        "{{CURRENT_TIME}}": "12:00:00",
        "{{CURRENT_WEEKDAY}}": "Friday",
        "{{CURRENT_TIMEZONE}}": "America/Los_Angeles",
        "{{USER_LANGUAGE}}": "en-US"
    },
    "model": {...},
    "direct": False,
    "function_calling": "native",
    "type": "user_response",
    "interface": "open-webui"
}
```

### Pipe Types

| Type | Class Name | Description |
|------|------------|-------------|
| **Pipe** | `Pipe` | Standalone model, appears as single model in UI |
| **Filter** | `Filter` | Middleware, intercepts requests/responses |
| **Manifold** | `Pipe` with `pipes` attribute | Multi-model container, exposes sub-models |

### Filter Functions

```python
class Filter:
    async def inlet(self, body: dict, __user__: dict) -> dict:
        """Process request before reaching model"""
        # Modify body
        return body

    async def outlet(self, body: dict, __user__: dict) -> dict:
        """Process response after model completes"""
        # Modify body
        return body

    async def stream(self, event: dict) -> dict:
        """Process streaming events (v0.5.17+)"""
        return event
```

### Manifold Pattern

```python
class Pipe:
    def __init__(self):
        self.name = "Manifold"
        # Expose multiple sub-models
        self.pipes = [
            {"id": "model1", "name": "Model 1"},
            {"id": "model2", "name": "Model 2"},
        ]
        # OR async function:
        # async def pipes(self): return [...]

    async def pipe(self, body, **kwargs):
        # Model ID will be "manifold.model1" or "manifold.model2"
        model_id = body.get("model")
        # Route to appropriate handler
```

### Module Loading (`plugin.py:117-165`)

1. Fetch code from `function` table
2. Replace imports: `from utils` → `from open_webui.utils`
3. Install pip dependencies from frontmatter
4. Create module via `types.ModuleType()`
5. Execute code via `exec()`
6. Instantiate `Pipe`/`Filter`/`Action` class

---

## 5. Model Registration: How Pipes Become Models

**Discovery** (`functions.py:80-155`):

1. Fetches all active `pipe` type functions
2. Loads each module to check for manifold `pipes` attribute
3. Creates OpenAI-compatible model entries
4. Merges with actual LLM models in `/v1/models` response

**Model Entry Format**:
```python
{
    "id": "pipe_id",  # or "manifold_id.sub_id" for manifolds
    "name": "Display Name",
    "object": "model",
    "created": 1234567890,
    "owned_by": "openai",
    "pipe": {"type": "pipe"},
    "has_user_valves": True | False,
}
```

---

## 6. SSE Streaming Implementation

### OpenAI-Compatible Format

```
data: {"choices":[{"delta":{"content":"text"}}]}\n\n
data: {"choices":[{"delta":{"content":"more text"},"finish_reason":"stop"}]}\n\n
data: [DONE]\n\n
```

### YouLab Custom Event Types

| Type | Format | Purpose |
|------|--------|---------|
| `status` | `{"type": "status", "content": "...", "reasoning": "..."}` | Show thinking/processing |
| `message` | `{"type": "message", "content": "..."}` | Append content |
| `done` | `{"type": "done"}` | Signal completion |
| `error` | `{"type": "error", "message": "..."}` | Display error |
| keepalive | `: keepalive\n\n` | Prevent connection timeout |

### Event Emitter Usage

```python
async def pipe(self, body, __event_emitter__, **kwargs):
    # Status update
    await __event_emitter__({
        "type": "status",
        "data": {"description": "Thinking...", "done": False}
    })

    # Content chunk
    await __event_emitter__({
        "type": "message",
        "data": {"content": "Hello "}
    })

    # More content
    await __event_emitter__({
        "type": "message",
        "data": {"content": "world!"}
    })

    # Completion
    await __event_emitter__({
        "type": "status",
        "data": {"description": "Complete", "done": True}
    })

    return ""  # Empty when using event emitter
```

### Backend Streaming Response (`main.py:445-453`)

```python
return StreamingResponse(
    generator(),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "Connection": "keep-alive",
        "X-Accel-Buffering": "no",  # Disable nginx buffering
    },
)
```

### Frontend Consumption (`streaming/index.ts`)

```typescript
const textStream = await createOpenAITextStream(res.body, splitLargeChunks);

for await (const update of textStream) {
    const { value, done, sources, error, usage } = update;

    if (error || done) break;

    mergedResponse.content += value;
}
```

---

## 7. Completion Finalization

### When Chats are Saved

1. After stream completes (`Chat.svelte:2244`)
2. After non-streaming responses
3. After message edits
4. After regeneration

### Save Handler

```typescript
await updateChatById(token, chatId, {
    models: selectedModels,
    history: history,
    messages: createMessagesList(history, history.currentId),
    params: params,
    files: chatFiles
});
```

### Backend Completion Hook

After chat completion, `/chat/completed` endpoint is called:
- Triggers title generation
- Triggers auto-tagging
- Runs background tasks

---

## 8. API Key Management

### Configuration

| Setting | Description |
|---------|-------------|
| `ENABLE_API_KEYS` | Enable API key creation (default: False) |
| `ENABLE_API_KEYS_ENDPOINT_RESTRICTIONS` | Restrict endpoints |
| `API_KEYS_ALLOWED_ENDPOINTS` | Allowed endpoint paths |

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auths/api_key` | Generate new API key |
| GET | `/auths/api_key` | Get current API key |
| DELETE | `/auths/api_key` | Delete API key |

### API Key Format

- Prefix: `sk-` (identifies as API key vs JWT)
- Stored in `api_key` table linked to user
- Validated in `get_current_user()` middleware

### Usage

```bash
curl -H "Authorization: Bearer sk-your-api-key" \
     https://your-openwebui.com/api/v1/models
```

---

## Code References

### Backend
- `OpenWebUI/open-webui/backend/open_webui/routers/chats.py` - Chat CRUD
- `OpenWebUI/open-webui/backend/open_webui/routers/models.py` - Model management
- `OpenWebUI/open-webui/backend/open_webui/routers/auths.py` - Authentication
- `OpenWebUI/open-webui/backend/open_webui/routers/users.py` - User management
- `OpenWebUI/open-webui/backend/open_webui/routers/pipelines.py` - Pipeline admin
- `OpenWebUI/open-webui/backend/open_webui/functions.py` - Pipe execution
- `OpenWebUI/open-webui/backend/open_webui/utils/plugin.py` - Module loading
- `OpenWebUI/open-webui/backend/open_webui/utils/headers.py` - User context headers
- `OpenWebUI/open-webui/backend/open_webui/socket/main.py` - Event emitter factory

### YouLab Integration
- `src/youlab_server/pipelines/letta_pipe.py` - Pipe implementation
- `src/youlab_server/server/main.py` - HTTP service endpoints
- `src/youlab_server/server/agents.py` - Agent streaming

### Frontend
- `OpenWebUI/open-webui/src/lib/apis/streaming/index.ts` - Stream consumer
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte` - Chat component
- `OpenWebUI/open-webui/src/lib/apis/chats/index.ts` - Chat API client
- `OpenWebUI/open-webui/src/lib/apis/models/index.ts` - Model API client

---

## Historical Context (from thoughts/)

### Existing Research Documents

| Document | Coverage |
|----------|----------|
| `thoughts/shared/research/2026-01-12-ARI-78-pipe-pipeline-architecture.md` | Letta Pipe architecture, message flow, lifecycle hooks |
| `thoughts/global/shared/reference/open-web-ui-pipes.md` | Pipe function signatures, user/chat ID handling |
| `thoughts/shared/research/ari-85-openwebui-integration-research.md` | User management, authentication flow, pipeline integration |
| `thoughts/shared/research/2025-01-12-ARI-78-openwebui-database-schema.md` | Database schema, ORM layer, migrations |
| `thoughts/shared/research/2026-01-12-ARI-78-openwebui-realtime-updates.md` | WebSocket/Socket.IO, SSE streaming |
| `thoughts/shared/research/2025-01-12-ARI-78-openwebui-model-metadata-extension.md` | Model schema extension |
| `thoughts/shared/research/2025-12-31-openwebui-local-dev-setup.md` | Local development setup |

---

## External Documentation

### Official Docs
- [Pipe Function Documentation](https://docs.openwebui.com/features/plugin/functions/pipe/)
- [Pipelines Framework](https://docs.openwebui.com/features/pipelines/)
- [Functions Overview](https://docs.openwebui.com/features/plugin/functions/)
- [Filter Function Documentation](https://docs.openwebui.com/features/plugin/functions/filter/)
- [API Endpoints](https://docs.openwebui.com/getting-started/api-endpoints/)
- [Environment Configuration](https://docs.openwebui.com/getting-started/env-configuration/)
- [API Keys & Monitoring](https://docs.openwebui.com/getting-started/advanced-topics/monitoring/)

### Community Resources
- [GitHub Repository](https://github.com/open-webui/open-webui)
- [Pipelines Repository](https://github.com/open-webui/pipelines)
- [Community Functions Registry](https://openwebui.com/functions)
- [Complete API Reference Discussion](https://github.com/open-webui/open-webui/discussions/16402)

---

## Open Questions

1. **Webhook callbacks**: Limited documentation on server-to-server webhook integrations
2. **Rate limiting configuration**: No public docs on built-in rate limiting beyond auth endpoints
3. **OpenAPI schema**: Full Swagger docs available only with `ENV=dev` at `/docs`
4. **Pipeline hot-reload**: Exact cache invalidation behavior when updating pipes

---

## Quick Reference: YouLab Pipe Integration Points

### Current Integration (`letta_pipe.py`)

```python
# User identification
user_id = __user__["id"]
user_name = __user__.get("name", "Unknown")

# Chat context
chat_id = __metadata__.get("chat_id")

# Messages
messages = body.get("messages", [])
user_message = messages[-1].get("content", "")

# Streaming via event emitter
await __event_emitter__({"type": "status", "data": {...}})
await __event_emitter__({"type": "message", "data": {"content": "..."}})

# Direct DB access (when needed)
from open_webui.models.chats import Chats
chat = Chats.get_chat_by_id(chat_id)
```

### Key Valves

```python
class Valves(BaseModel):
    LETTA_SERVICE_URL: str = "http://host.docker.internal:8100"
    AGENT_TYPE: str = "poc-tutor"
    ENABLE_LOGGING: bool = True
    ENABLE_THINKING: bool = True
```
