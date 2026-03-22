---
date: 2026-01-30T17:22:35-08:00
researcher: Claude
git_commit: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "OpenWebUI Chat Creation API - Can admins create chats for other users?"
tags: [research, openwebui, api, chat-creation, phase-3, module-thread-binding]
status: complete
last_updated: 2026-01-30
last_updated_by: Claude
---

# Research: OpenWebUI Chat Creation API

**Date**: 2026-01-30T17:22:35-08:00
**Researcher**: Claude
**Git Commit**: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

Can we create OpenWebUI chat threads on behalf of users using an admin API key? Or do we need to modify OpenWebUI?

This is critical for Phase 3 of the module-thread binding implementation plan, where Ralph needs to create module threads for users when they sign up via webhook.

## Summary

**Answer: No, OpenWebUI does not support creating chats for other users via admin API.**

The `/api/chats/new` endpoint strictly creates chats for the authenticated user only. The `user_id` is extracted from the JWT token or API key and cannot be overridden via request parameters. There is no admin impersonation capability.

**Recommended Solution: Direct database insertion**

Since YouLab already has access to the Dolt database and OpenWebUI uses a simple `chat` table schema, the most practical approach is to insert chat records directly into the database, bypassing the API. This avoids modifying the OpenWebUI codebase and works with the existing infrastructure.

## Detailed Findings

### 1. Chat Creation Endpoint Analysis

**Location**: `OpenWebUI/open-webui/backend/open_webui/routers/chats.py:259-268`

```python
@router.post("/new", response_model=Optional[ChatResponse])
async def create_new_chat(form_data: ChatForm, user=Depends(get_verified_user)):
    try:
        chat = Chats.insert_new_chat(user.id, form_data)
        return ChatResponse(**chat.model_dump())
    except Exception as e:
        log.exception(e)
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST, detail=ERROR_MESSAGES.DEFAULT()
        )
```

**Key observations**:
- Uses `get_verified_user` dependency injection
- `user.id` is extracted from authentication context automatically
- No parameter exists to override user_id
- Calls `Chats.insert_new_chat(user.id, form_data)` with authenticated user's ID

### 2. Request Schema

**Location**: `OpenWebUI/open-webui/backend/open_webui/models/chats.py:124-127`

```python
class ChatForm(BaseModel):
    chat: dict
    folder_id: Optional[str] = None
```

The `ChatForm` schema only accepts:
- `chat`: Dictionary containing chat data (title, messages, history)
- `folder_id`: Optional folder assignment

**No `user_id` field exists in the request schema.**

### 3. Authentication Flow

**Location**: `OpenWebUI/open-webui/backend/open_webui/utils/auth.py:269-365`

Authentication supports two methods:
1. **JWT Token**: Decoded from `Authorization: Bearer {token}` header or cookie
2. **API Key**: Format `sk-{32_hex_chars}`, looked up via `Users.get_user_by_api_key()`

Both methods bind the request to a specific user. There is no mechanism to specify a different `user_id`.

### 4. Admin Capabilities

Admins have **read-only** access to other users' chats:
- `GET /chats/list/user/{user_id}` - View other users' chats
- `GET /chats/all/db` - Export all chats (when `ENABLE_ADMIN_EXPORT=True`)
- `DELETE /chats/{id}` - Delete any chat by ID

Admins **cannot**:
- Create chats for other users
- Impersonate users
- Override user_id in any request

### 5. Database Schema

**Location**: `OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-65`

```python
class Chat(Base):
    __tablename__ = "chat"

    id = Column(String, primary_key=True, unique=True)
    user_id = Column(String)  # Owner of chat
    title = Column(Text)
    chat = Column(JSON)  # Full chat data blob

    created_at = Column(BigInteger)
    updated_at = Column(BigInteger)

    share_id = Column(Text, unique=True, nullable=True)
    archived = Column(Boolean, default=False)
    pinned = Column(Boolean, default=False, nullable=True)

    meta = Column(JSON, server_default="{}")
    folder_id = Column(Text, nullable=True)
```

The schema is simple and can be directly inserted into.

### 6. Community Status

GitHub Discussion #12114 proposes admin API key management for user impersonation, but this is **not implemented yet**. The discussion mentions:
- `POST /api/v1/users/{user_id}/api_key` - Generate API key for a user
- `GET /api/v1/users/{user_id}/api_key` - Retrieve existing API key

This would allow getting a user's API key and acting as them, but the feature doesn't exist.

## Proposed Solutions

### Option 1: Direct Database Insertion (Recommended)

**Approach**: Insert chat records directly into OpenWebUI's database, bypassing the API.

**Implementation in `src/ralph/openwebui.py`**:

```python
import uuid
import time
import json
from ralph.config import settings

class OpenWebUIClient:
    """Client for OpenWebUI operations via direct DB access."""

    def __init__(self, db_connection):
        self.db = db_connection  # OpenWebUI's SQLite or PostgreSQL

    async def create_chat_for_user(
        self,
        user_id: str,
        title: str,
        first_message: str | None = None,
    ) -> str:
        """Create a new chat owned by the specified user."""
        chat_id = str(uuid.uuid4())
        now = int(time.time())

        # Build chat JSON structure matching OpenWebUI format
        chat_data = {
            "title": title,
            "models": ["ralph-wiggum"],
            "messages": [],
            "history": {
                "messages": {},
                "currentId": None
            },
            "tags": [],
            "params": {},
        }

        # Add first message if provided (assistant message)
        if first_message:
            msg_id = str(uuid.uuid4())
            chat_data["messages"] = [
                {
                    "id": msg_id,
                    "parentId": None,
                    "childrenIds": [],
                    "role": "assistant",
                    "content": first_message,
                    "timestamp": now,
                }
            ]
            chat_data["history"]["messages"][msg_id] = {
                "id": msg_id,
                "parentId": None,
                "childrenIds": [],
                "role": "assistant",
                "content": first_message,
            }
            chat_data["history"]["currentId"] = msg_id

        # Insert into database
        await self.db.execute(
            """
            INSERT INTO chat (id, user_id, title, chat, created_at, updated_at, archived)
            VALUES (?, ?, ?, ?, ?, ?, ?)
            """,
            (chat_id, user_id, title, json.dumps(chat_data), now, now, True)
        )

        return chat_id
```

**Pros**:
- No OpenWebUI modifications needed
- Works immediately
- Full control over chat structure
- Can set `archived=True` directly (hiding from Chats sidebar)

**Cons**:
- Requires access to OpenWebUI's database
- Must stay in sync with OpenWebUI schema changes
- Bypasses any OpenWebUI business logic

**Database access options**:
1. **Shared PostgreSQL**: If OpenWebUI uses PostgreSQL, Ralph can connect directly
2. **SQLite file access**: If OpenWebUI uses SQLite, mount the volume in Ralph's container
3. **HTTP proxy**: Create a simple endpoint in OpenWebUI that accepts admin requests

### Option 2: OpenWebUI Modification (Minimal)

**Approach**: Add a `user_id` parameter to the `/api/chats/new` endpoint for admin users.

**Modification in `OpenWebUI/open-webui/backend/open_webui/routers/chats.py`**:

```python
class ChatFormAdmin(ChatForm):
    """Extended form for admin chat creation."""
    user_id: str | None = None  # Optional user_id override for admins

@router.post("/new", response_model=Optional[ChatResponse])
async def create_new_chat(form_data: ChatFormAdmin, user=Depends(get_verified_user)):
    try:
        # Allow admins to create chats for other users
        target_user_id = user.id
        if form_data.user_id and user.role == "admin":
            target_user_id = form_data.user_id

        chat = Chats.insert_new_chat(target_user_id, form_data)
        return ChatResponse(**chat.model_dump())
    except Exception as e:
        log.exception(e)
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST, detail=ERROR_MESSAGES.DEFAULT()
        )
```

**Pros**:
- Clean API interface
- Works with existing OpenWebUI infrastructure
- Easy to understand and maintain

**Cons**:
- Requires maintaining a fork of OpenWebUI
- Must merge upstream changes carefully
- May conflict with future OpenWebUI updates

### Option 3: Trusted Header Authentication

**Approach**: Configure OpenWebUI to trust headers from Ralph's reverse proxy.

**Configuration**:
```bash
WEBUI_AUTH_TRUSTED_EMAIL_HEADER=X-User-Email
```

**Flow**:
1. Ralph receives webhook with user signup
2. Ralph makes API call to OpenWebUI with header `X-User-Email: user@example.com`
3. OpenWebUI authenticates request as that user
4. Chat created for the correct user

**Pros**:
- No code changes to OpenWebUI
- Uses existing OpenWebUI feature

**Cons**:
- Security risk if misconfigured
- OpenWebUI must only be accessible via trusted proxy
- Adds infrastructure complexity

### Option 4: Wait for Upstream Feature

GitHub Discussion #12114 proposes the exact feature needed. If accepted, OpenWebUI would add:
- `POST /api/v1/users/{user_id}/api_key` - Generate API key for a user
- Admin could get user's API key and use it to create chats

**Status**: Under discussion, not implemented.

## Recommendation

**Use Option 1 (Direct Database Insertion)** for the following reasons:

1. **Immediate**: Works now without waiting for upstream
2. **Simple**: No OpenWebUI modifications to maintain
3. **Aligned**: YouLab already manages Dolt database directly
4. **Flexible**: Full control over chat structure and metadata
5. **Safe**: Doesn't affect OpenWebUI's internal operation

**Implementation path**:
1. Determine OpenWebUI's database type (PostgreSQL recommended for production)
2. Add database connection configuration to Ralph
3. Implement `OpenWebUIClient.create_chat_for_user()` as shown above
4. Test with a new user signup flow

If OpenWebUI is using SQLite (default), consider migrating it to PostgreSQL for better multi-service access.

## Code References

- `OpenWebUI/open-webui/backend/open_webui/routers/chats.py:259-268` - Chat creation endpoint
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:124-127` - ChatForm schema
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:35-65` - Chat database model
- `OpenWebUI/open-webui/backend/open_webui/utils/auth.py:269-365` - Authentication flow
- `OpenWebUI/open-webui/backend/open_webui/models/chats.py:241-264` - Database insertion logic

## Historical Context

From the implementation plan at `thoughts/shared/plans/2026-01-30-module-thread-binding.md`:
- Phase 3 requires creating module threads when users sign up
- Threads should be archived (hidden from Chats) but visible in Module sidebar
- First message content comes from course TOML configuration
- Ralph already has Dolt database access patterns established

## Related Research

- `thoughts/shared/research/2026-01-30-welcome-course-implementation-gap-analysis.md` - Gap analysis for module-thread binding
- `thoughts/shared/plans/2026-01-30-module-thread-binding.md` - Full implementation plan

## Open Questions

1. **Database type**: What database backend does the OpenWebUI production deployment use? (SQLite vs PostgreSQL)
2. **Schema versioning**: How to handle OpenWebUI schema changes during upgrades?
3. **Transaction boundaries**: Should chat creation and Ralph enrollment be in a single transaction?

## External Sources

- [Backend-Controlled, UI-Compatible API Flow | Open WebUI](https://docs.openwebui.com/tutorials/integrations/backend-controlled-ui-compatible-flow/)
- [GitHub Discussion #12114 - API Key Management for User Impersonation](https://github.com/open-webui/open-webui/discussions/12114)
- [SSO (Trusted Header Authentication) | Open WebUI](https://docs.openwebui.com/features/auth/sso/)
- [Reassigning OpenWebUI Chat Users](https://orionrobots.co.uk/2024/10/12/reassigning-openwebui-chat-users.html)
