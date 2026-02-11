# Chat Message Injection Endpoint

## Overview

Add a general-purpose Ralph API endpoint for injecting messages into OpenWebUI chats. This requires a small patch to the OpenWebUI fork to allow admin-level chat creation on behalf of other users, plus a new Ralph endpoint that wraps the OpenWebUI chat API.

## Current State Analysis

- Ralph already has `RALPH_OPENWEBUI_URL` and `RALPH_OPENWEBUI_API_KEY` configured in docker-compose (`http://openwebui:8080` internally)
- `config.py` already defines `openwebui_url` and `openwebui_api_key` settings
- OpenWebUI's `POST /api/chats/new` only creates chats for the authenticated user — no `user_id` override
- OpenWebUI's `POST /api/chats/{id}` does full-replacement updates, so append = GET + modify + POST
- The existing `src/ralph/api/` pattern uses FastAPI routers with Pydantic models

### Key Discoveries:
- OpenWebUI chat JSON stores messages in `history.messages` as a UUID-keyed dict with `parentId`/`childrenIds` tree structure, plus a `currentId` pointer
- The `ChatForm` model is just `{chat: dict, folder_id: Optional[str]}` — no `user_id` field
- OpenWebUI already has admin-only endpoints gated by `get_admin_user` (e.g., `GET /api/chats/list/user/{user_id}`) — good precedent for our patch
- The `insert_new_chat` method already takes `user_id` as its first arg, so no DB layer changes needed

## Desired End State

Ralph exposes `POST /chats/send` that can:
1. **Create a new chat** for any user with a pre-populated message (e.g., welcome message)
2. **Append a message** to an existing chat (e.g., background task nudge)

Both operations go through the OpenWebUI API (not direct DB access), keeping Ralph decoupled from OpenWebUI's internals.

### Verification:
```bash
# Create a new chat with a welcome message for a user
curl -X POST http://localhost:8200/chats/send \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "ed6d1437-7b38-47a4-bd49-670267f0a7ce",
    "role": "assistant",
    "content": "Welcome to YouLab!",
    "title": "Welcome",
    "archived": true
  }'
# Returns: {"chat_id": "...", "message_id": "..."}

# Append to existing chat
curl -X POST http://localhost:8200/chats/send \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "ed6d1437-7b38-47a4-bd49-670267f0a7ce",
    "chat_id": "existing-chat-id",
    "role": "assistant",
    "content": "Hey, checking in..."
  }'
# Returns: {"chat_id": "...", "message_id": "..."}
```

## What We're NOT Doing

- TOML config loading / course system
- Block template initialization
- First message content (that's a separate task that will *use* this endpoint)
- Background task integration
- Direct SQLite/DB access to OpenWebUI
- Any OpenWebUI frontend changes

## Implementation Approach

Two changes: a small patch to the OpenWebUI fork (allow admin `user_id` on chat creation), then a new Ralph API router that wraps OpenWebUI's chat API.

## Phase 1: OpenWebUI Fork Patch

### Overview
Allow admins to create chats on behalf of other users by adding an optional `user_id` field to `ChatForm`.

### Changes Required:

#### 1. ChatForm model
**File**: `backend/open_webui/models/chats.py` (in the open-webui fork)
**Changes**: Add optional `user_id` field to `ChatForm`

```python
class ChatForm(BaseModel):
    chat: dict
    folder_id: Optional[str] = None
    user_id: Optional[str] = None  # Admin-only: create chat as this user
```

#### 2. Chat creation endpoint
**File**: `backend/open_webui/routers/chats.py` (in the open-webui fork)
**Changes**: Use `form_data.user_id` when caller is admin

```python
@router.post("/new", response_model=Optional[ChatResponse])
async def create_new_chat(form_data: ChatForm, user=Depends(get_verified_user)):
    try:
        # Allow admins to create chats on behalf of other users
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

### Success Criteria:

#### Automated Verification:
- [ ] OpenWebUI builds successfully: `docker compose -f docker-compose.prod.yml build openwebui`
- [ ] Existing tests pass (if any exist in the fork)

#### Manual Verification:
- [ ] Admin API key can create a chat for another user:
  ```bash
  curl -X POST http://localhost:3000/api/chats/new \
    -H "Authorization: Bearer sk-ADMIN_KEY" \
    -H "Content-Type: application/json" \
    -d '{"chat": {"title": "Test", "messages": []}, "user_id": "TARGET_USER_ID"}'
  ```
- [ ] Non-admin requests with `user_id` silently ignore it (uses their own ID)
- [ ] Chat appears in the target user's chat list

**Implementation Note**: This change is in the separate open-webui fork repo. Commit and push there before proceeding to Phase 2.

---

## Phase 2: Ralph Chat Injection Endpoint

### Overview
New Ralph API router at `src/ralph/api/chats.py` with a `POST /chats/send` endpoint. It wraps OpenWebUI's chat API to create/append messages.

### Changes Required:

#### 1. OpenWebUI HTTP client
**File**: `src/ralph/openwebui_client.py` (new)
**Changes**: Thin async httpx client for OpenWebUI's chat API

```python
"""OpenWebUI API client for chat operations."""

from __future__ import annotations

import uuid
import time
from typing import Any

import httpx
import structlog

from ralph.config import get_settings

log = structlog.get_logger()


class OpenWebUIClient:
    """Async client for OpenWebUI chat API."""

    def __init__(self) -> None:
        settings = get_settings()
        self._base_url = settings.openwebui_url
        self._api_key = settings.openwebui_api_key

    def _headers(self) -> dict[str, str]:
        return {
            "Authorization": f"Bearer {self._api_key}",
            "Content-Type": "application/json",
        }

    async def create_chat(
        self,
        user_id: str,
        title: str,
        role: str,
        content: str,
        model: str = "ralph-wiggum",
        archived: bool = False,
    ) -> dict[str, str]:
        """Create a new chat with an initial message. Returns {chat_id, message_id}."""
        msg_id = str(uuid.uuid4())
        now = int(time.time())

        chat_data: dict[str, Any] = {
            "title": title,
            "models": [model],
            "tags": [],
            "history": {
                "messages": {
                    msg_id: {
                        "id": msg_id,
                        "parentId": None,
                        "childrenIds": [],
                        "role": role,
                        "content": content,
                        "timestamp": now,
                    }
                },
                "currentId": msg_id,
            },
        }

        async with httpx.AsyncClient(timeout=30.0) as client:
            resp = await client.post(
                f"{self._base_url}/api/chats/new",
                headers=self._headers(),
                json={
                    "chat": chat_data,
                    "user_id": user_id,
                },
            )
            resp.raise_for_status()
            result = resp.json()
            chat_id = result["id"]

        # Archive if requested (separate call since ChatForm doesn't have archived)
        if archived:
            async with httpx.AsyncClient(timeout=30.0) as client:
                await client.post(
                    f"{self._base_url}/api/chats/{chat_id}/archive",
                    headers=self._headers(),
                )

        log.info("openwebui_chat_created", chat_id=chat_id, user_id=user_id)
        return {"chat_id": chat_id, "message_id": msg_id}

    async def append_message(
        self,
        chat_id: str,
        role: str,
        content: str,
    ) -> dict[str, str]:
        """Append a message to an existing chat. Returns {chat_id, message_id}."""
        async with httpx.AsyncClient(timeout=30.0) as client:
            # GET current chat
            resp = await client.get(
                f"{self._base_url}/api/chats/{chat_id}",
                headers=self._headers(),
            )
            resp.raise_for_status()
            chat_obj = resp.json()

        chat_data = chat_obj["chat"]
        history = chat_data.get("history", {"messages": {}, "currentId": None})
        current_id = history.get("currentId")

        # Build new message
        msg_id = str(uuid.uuid4())
        now = int(time.time())
        new_msg = {
            "id": msg_id,
            "parentId": current_id,
            "childrenIds": [],
            "role": role,
            "content": content,
            "timestamp": now,
        }

        # Link parent -> child
        if current_id and current_id in history["messages"]:
            history["messages"][current_id]["childrenIds"].append(msg_id)

        # Add message and update pointer
        history["messages"][msg_id] = new_msg
        history["currentId"] = msg_id
        chat_data["history"] = history

        # POST updated chat
        async with httpx.AsyncClient(timeout=30.0) as client:
            resp = await client.post(
                f"{self._base_url}/api/chats/{chat_id}",
                headers=self._headers(),
                json={"chat": chat_data},
            )
            resp.raise_for_status()

        log.info("openwebui_message_appended", chat_id=chat_id, message_id=msg_id)
        return {"chat_id": chat_id, "message_id": msg_id}
```

#### 2. API router
**File**: `src/ralph/api/chats.py` (new)
**Changes**: FastAPI router with the `POST /chats/send` endpoint

```python
"""Chat message injection API."""

from __future__ import annotations

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

from ralph.openwebui_client import OpenWebUIClient

router = APIRouter(prefix="/chats", tags=["chats"])


class SendMessageRequest(BaseModel):
    """Request to send a message to a chat."""

    user_id: str
    chat_id: str | None = None  # If omitted, creates new chat
    role: str = "assistant"
    content: str
    title: str = "New Chat"  # Only used when creating
    archived: bool = False  # Only used when creating


class SendMessageResponse(BaseModel):
    """Response from sending a message."""

    chat_id: str
    message_id: str
    created: bool  # True if new chat was created


@router.post("/send", response_model=SendMessageResponse)
async def send_message(request: SendMessageRequest) -> SendMessageResponse:
    """Send a message to a chat. Creates the chat if chat_id is not provided."""
    client = OpenWebUIClient()

    try:
        if request.chat_id:
            result = await client.append_message(
                chat_id=request.chat_id,
                role=request.role,
                content=request.content,
            )
            return SendMessageResponse(**result, created=False)
        else:
            result = await client.create_chat(
                user_id=request.user_id,
                title=request.title,
                role=request.role,
                content=request.content,
                archived=request.archived,
            )
            return SendMessageResponse(**result, created=True)
    except Exception as e:
        raise HTTPException(status_code=502, detail=f"OpenWebUI API error: {e}") from e
```

#### 3. Register the router
**File**: `src/ralph/server.py`
**Changes**: Import and include the new chats router

```python
# Add import
from ralph.api.chats import router as chats_router

# Add after existing router includes
app.include_router(chats_router)
```

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `uv run ruff check src/ralph/`
- [ ] Type checking passes: `uv run basedpyright src/ralph/`
- [ ] Full verify: `make verify-agent`

#### Manual Verification:
- [ ] Create new chat for a user (archived):
  ```bash
  curl -X POST http://localhost:8200/chats/send \
    -H "Content-Type: application/json" \
    -d '{
      "user_id": "ed6d1437-7b38-47a4-bd49-670267f0a7ce",
      "role": "assistant",
      "content": "Welcome to YouLab!",
      "title": "Welcome",
      "archived": true
    }'
  ```
  Verify: response has `chat_id`, `message_id`, `created: true`
- [ ] Chat appears in user's archived chats in OpenWebUI
- [ ] Append message to existing chat:
  ```bash
  curl -X POST http://localhost:8200/chats/send \
    -H "Content-Type: application/json" \
    -d '{
      "user_id": "ed6d1437-7b38-47a4-bd49-670267f0a7ce",
      "chat_id": "CHAT_ID_FROM_ABOVE",
      "role": "assistant",
      "content": "Following up..."
    }'
  ```
  Verify: message appears in the chat when opened in OpenWebUI
- [ ] Error handling: invalid chat_id returns 502, missing OpenWebUI returns clear error

**Implementation Note**: After completing this phase, verify that the full flow works end-to-end: create chat via Ralph → see it in OpenWebUI → open it → message is there.

---

## Testing Strategy

### Manual Testing Steps:
1. Start local stack: `docker compose -f docker-compose.prod.yml up -d`
2. Create archived welcome chat for test user via `POST /chats/send`
3. Log in as test user in OpenWebUI, check archived chats
4. Open the chat — verify message renders correctly
5. Append a second message via `POST /chats/send` with `chat_id`
6. Refresh chat in OpenWebUI — verify both messages appear in order
7. Test error cases: bad chat_id, missing OpenWebUI connection

## References

- OpenWebUI Chat API docs: https://github.com/open-webui/open-webui/discussions/16402
- OpenWebUI fork: https://github.com/ariavasulin/open-webui
- Chat message structure: `history.messages` is UUID-keyed dict with `parentId`/`childrenIds` tree
- Existing Ralph API pattern: `src/ralph/api/blocks.py`
