## Summary

OpenWebUI Pipes receive a rich `body` dict containing the chat completion request payload (messages, model, stream options), plus several dependency-injected special arguments including `__user__` (user info with id, email, name, role), `__metadata__` (containing chat_id, message_id, session_id, user_id, files, features, variables, and model info), and `__request__` (FastAPI Request object). While chat_id is exposed directly, organizational metadata like folder IDs, tags, and chat title are **not** passed to Pipes by default—Pipes would need to query internal OpenWebUI models directly to access this information.

---

## Detailed Findings

### 1. Pipe Function Signature and Body Parameter Structure

**Function Signature (v0.5+):**

```python
from pydantic import BaseModel, Field
from fastapi import Request
from typing import Optional, Dict

class Pipe:
    def __init__(self):
        pass
    
    async def pipe(
        self,
        body: dict,
        __user__: dict = None,
        __metadata__: dict = None,
        __request__: Request = None,
        __event_emitter__: Callable = None,
    ) -> str:
        # Your logic here
        return "response"
```

**Body Parameter Structure:**

```python
body = {
    "stream": True,
    "model": "model-id-string",
    "messages": [
        {
            "role": "user",
            "content": "What is in this picture?"
            # or content can be a list for multimodal:
            # "content": [{"type": "text", "text": "..."}, {"type": "image_url", "image_url": {...}}]
        },
        {
            "role": "assistant", 
            "content": "Previous response..."
        }
    ],
    "features": {
        "image_generation": False,
        "code_interpreter": False,
        "web_search": False
    },
    "stream_options": {
        "include_usage": True
    },
    "metadata": {...},  # Same as __metadata__
    "files": [...]      # Same as __files__ if present
}
```

**Sources:** [Official Pipe Documentation](https://docs.openwebui.com/features/plugin/functions/pipe/), [Special Arguments Guide](https://docs.openwebui.com/tutorials/tips/special_arguments/)

---

### 2. User ID - Format and Location

**Yes, user_id is included in multiple places:**

```python
# Via __user__ dict:
__user__ = {
    "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",  # UUID format
    "email": "user@example.com",
    "name": "Patrick",
    "role": "user",  # or "admin"
    "valves": "<UserValve instance if defined>"
}

# Via __metadata__:
__metadata__["user_id"]  # Same UUID
```

**Access Pattern:**
```python
async def pipe(self, body: dict, __user__: dict, __metadata__: dict):
    user_id = __user__["id"]
    # or
    user_id = __metadata__.get("user_id")
```

**Source:** [GitHub Discussion #6999](https://github.com/open-webui/open-webui/discussions/6999)

---

### 3. Chat/Thread ID or Title

**Chat ID: ✅ Yes, included**

```python
# Via __metadata__ (recommended):
chat_id = __metadata__.get("chat_id")  # UUID string

# Also available as shorthand reserved argument:
async def pipe(self, body: dict, __chat_id__: str):
    # __chat_id__ is auto-injected
```

**Chat Title: ❌ Not directly passed**

Chat title is **not** included in the body or metadata passed to Pipes. To access chat title, you would need to query OpenWebUI's internal Chats model:

```python
from open_webui.models.chats import Chats

chat = Chats.get_chat_by_id(chat_id)
title = chat.title if chat else None
```

**Source:** [Pipeline Enriched with Metadata](https://openwebui.com/f/codysandahl/pipeline_enriched_with_metadata)

---

### 4. Metadata About Folders or Workspaces

**❌ Not exposed to Pipes**

Folder/workspace metadata is **not included** in the data passed to Pipes. OpenWebUI organizes chats using:
- **Folders** (project-like workspaces with system prompts and knowledge bases)
- **Tags** (labels for conversations)

Neither folder_id nor folder metadata appears in `__metadata__` or `body`. This is a UI/organization layer that doesn't propagate to the Pipe execution context.

**Source:** [Organizing Conversations Documentation](https://docs.openwebui.com/features/chat-features/conversation-organization/)

---

### 5. Can Pipes Access Chat Metadata?

**What's Available Directly:**

| Metadata | Available | Location |
|----------|-----------|----------|
| chat_id | ✅ Yes | `__metadata__["chat_id"]` |
| message_id | ✅ Yes | `__metadata__["message_id"]` |
| session_id | ✅ Yes | `__metadata__["session_id"]` |
| Previous messages | ✅ Yes | `body["messages"]` or `__messages__` |
| Chat title | ❌ No | Must query internal API |
| Creation date | ❌ No | Must query internal API |
| Tags | ❌ No | Must query internal API |
| Folder ID | ❌ No | Not exposed |

**Full __metadata__ Structure:**

```python
__metadata__ = {
    "user_id": "uuid-string",
    "chat_id": "uuid-string",
    "message_id": "uuid-string",
    "session_id": "session-string",
    "tool_ids": None,  # or list of strings
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
        "{{CURRENT_DATETIME}}": "2025-02-02 XX:XX:XX",
        "{{CURRENT_DATE}}": "2025-02-02",
        "{{CURRENT_TIME}}": "XX:XX:XX",
        "{{CURRENT_WEEKDAY}}": "Monday",
        "{{CURRENT_TIMEZONE}}": "Europe/Berlin",
        "{{USER_LANGUAGE}}": "en-US"
    },
    "model": {...},  # Model configuration dict
    "direct": False,
    "function_calling": "native",
    "type": "user_response",
    "interface": "open-webui"
}
```

**To Access Additional Chat Metadata (title, created_at, tags):**

```python
from open_webui.models.chats import Chats

async def pipe(self, body: dict, __metadata__: dict):
    chat_id = __metadata__.get("chat_id")
    chat = Chats.get_chat_by_id(chat_id)
    
    if chat:
        title = chat.title
        created_at = chat.created_at
        # Note: tag access may require additional queries
```

---

### 6. Configuring OpenWebUI to Pass Custom Metadata

**No built-in configuration option** exists to pass custom metadata to Pipes. However, there are workarounds:

**Option A: Use a Filter Function as middleware**

```python
class Filter:
    async def inlet(self, body: dict, user: dict) -> dict:
        # Enrich body with custom metadata before it reaches Pipe
        body["custom_metadata"] = {
            "my_key": "my_value"
        }
        return body
```

**Option B: Use Valves for static configuration**

```python
class Pipe:
    class Valves(BaseModel):
        CUSTOM_CONFIG: str = Field(default="", description="Custom config")
```

**Limitation:** OpenWebUI strips non-standard fields from the body before sending to external pipelines to maintain OpenAI API compatibility. Custom metadata added via filters may be removed before reaching external pipeline servers.

**Source:** [GitHub Discussion #9804](https://github.com/open-webui/open-webui/discussions/9804)

---

### 7. Chat Organization - Folders, Workspaces, Tags

**How OpenWebUI Organizes Chats:**

| Feature | Description | Exposed to Pipes? |
|---------|-------------|-------------------|
| **Folders** | Project-like workspaces; can have system prompts and knowledge bases attached | ❌ No |
| **Tags** | Labels/keywords on conversations; supports auto-tagging | ❌ No |
| **Pinned Chats** | Priority conversations | ❌ No |

**Internal Database Schema (from source):**
- Chats are stored with `id`, `user_id`, `title`, `created_at`, `updated_at`, `visibility`
- Folder association exists but is not exposed via the Pipe interface
- Tags are stored separately and linked to chats

**To Access Organization Data Programmatically:**

```python
# This is speculative based on internal model patterns
from open_webui.models.chats import Chats
from open_webui.models.tags import Tags  # if exists

# Get chat with potential folder info
chat = Chats.get_chat_by_id(chat_id)
# Access folder_id if available in schema
```

---

## Code Examples with Sources

### Basic Pipe with All Reserved Arguments

```python
"""
title: Complete Pipe Example
version: 0.1.0
"""
from pydantic import BaseModel, Field
from fastapi import Request
from typing import Optional, Dict, Callable, Awaitable
from open_webui.models.users import Users
from open_webui.utils.chat import generate_chat_completion

class Pipe:
    class Valves(BaseModel):
        API_KEY: str = Field(default="", description="Your API key")
    
    def __init__(self):
        self.valves = self.Valves()
    
    async def pipe(
        self,
        body: dict,
        __user__: Optional[dict] = None,
        __metadata__: Optional[dict] = None,
        __request__: Request = None,
        __event_emitter__: Callable[[dict], Awaitable[None]] = None,
    ) -> str:
        # Extract user info
        user_id = __user__["id"] if __user__ else "anonymous"
        user_name = __user__.get("name", "Unknown")
        
        # Extract chat context
        chat_id = __metadata__.get("chat_id") if __metadata__ else None
        message_id = __metadata__.get("message_id") if __metadata__ else None
        session_id = __metadata__.get("session_id") if __metadata__ else None
        
        # Get messages from body
        messages = body.get("messages", [])
        
        # Get user object for internal calls
        user = Users.get_user_by_id(user_id)
        
        # Emit status
        if __event_emitter__:
            await __event_emitter__({
                "type": "status",
                "data": {"description": "Processing...", "done": False}
            })
        
        # Your logic here
        body["model"] = "llama3.2:latest"
        return await generate_chat_completion(__request__, body, user)
```

**Source:** [Pipe Function Documentation](https://docs.openwebui.com/features/plugin/functions/pipe/)

### Extracting Metadata (Real Community Example)

```python
"""
Source: https://openwebui.com/f/codysandahl/pipeline_enriched_with_metadata
"""
class Metadata(BaseModel):
    user: str
    user_id: str
    chat_id: str
    message_id: str
    session_id: str

def _extract_metadata(
    self,
    body: dict,
    __metadata__: Optional[Dict] = None,
    __user__: Optional[Dict] = None
) -> Metadata:
    """Extract and validate metadata from OpenWebUI request."""
    return Metadata(
        user=__user__.get("name", "Unknown") if __user__ else "Unknown",
        user_id=__metadata__.get("user_id", "") if __metadata__ else "",
        chat_id=__metadata__.get("chat_id", "") if __metadata__ else "",
        message_id=__metadata__.get("message_id", "") if __metadata__ else "",
        session_id=__metadata__.get("session_id", "") if __metadata__ else "",
    )
```

**Source:** [Medium Article - LangGraph Integration](https://pessini.medium.com/from-open-webui-to-langgraph-building-a-human-in-the-loop-pipe-for-real-time-ai-control-26561cca9f9c)

---

## Key Documentation Links

| Resource | URL |
|----------|-----|
| **Official Pipe Functions Guide** | https://docs.openwebui.com/features/plugin/functions/pipe/ |
| **Special Arguments Reference** | https://docs.openwebui.com/tutorials/tips/special_arguments/ |
| **Migration Guide (0.4 → 0.5)** | https://docs.openwebui.com/features/plugin/migration/ |
| **Functions Overview** | https://docs.openwebui.com/features/plugin/functions/ |
| **Community Functions Registry** | https://openwebui.com/functions |
| **GitHub: open-webui/open-webui** | https://github.com/open-webui/open-webui |
| **GitHub: open-webui/docs** | https://github.com/open-webui/docs |

---

## Version-Specific Notes

| Version | Key Changes |
|---------|-------------|
| **v0.5+** | `__request__: Request` parameter is **mandatory** for internal function calls; unified router architecture |
| **v0.5.1+** | `chat_id` moved from `body.get("chat_id")` to `body.get("metadata")["chat_id"]` |
| **v0.4.x** | Separate app architecture; different import paths (`open_webui.apps.ollama`) |

**Critical:** Always check for both locations when accessing `chat_id` for compatibility:
```python
chat_id = body.get("chat_id") or body.get("metadata", {}).get("chat_id")
```