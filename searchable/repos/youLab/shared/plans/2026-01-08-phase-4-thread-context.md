# Phase 4: Thread Context Implementation Plan

## Overview

Add chat title renaming capability to the Pipe. Title extraction already works - this phase just adds the ability to programmatically set/update titles.

## Current State Analysis

**Already implemented:**
- `_get_chat_title(chat_id)` extracts title from OpenWebUI DB (`letta_pipe.py:57-71`)
- Title passed to HTTP service in `/chat/stream` request (`letta_pipe.py:170`)
- Title stored in Honcho as message metadata
- Title logged in Langfuse traces

**Missing:**
- Ability to rename/set chat titles programmatically

## Desired End State

The Pipe can both read AND write chat titles, enabling:
- Auto-naming sessions based on content
- Programmatic title updates from the HTTP service
- Future curriculum-based naming (Phase 5)

### Verification

- [ ] `_set_chat_title(chat_id, title)` successfully updates title in OpenWebUI DB
- [ ] Updated title persists and is returned by `_get_chat_title()`
- [ ] Tests pass: `uv run pytest tests/test_pipe.py -k chat_title`

## What We're NOT Doing

- Parsing titles into module/lesson structure (deferred - may not be needed)
- Memory block updates based on context (deferred)
- Context caching in HTTP service (not needed - Pipe provides title)
- Exposing title via HTTP API endpoints (can add later if needed)

## Implementation Approach

Minimal change: Add one method to the Pipe that wraps OpenWebUI's `Chats.update_chat_title_by_id()`.

---

## Phase 4.1: Add Title Rename Method

### Overview

Add `_set_chat_title()` method to the Pipe.

### Changes Required

#### 1. Add Method to Pipe
**File**: `src/letta_starter/pipelines/letta_pipe.py`

Add after `_get_chat_title()` (after line 71):

```python
def _set_chat_title(self, chat_id: str | None, title: str) -> bool:
    """Set chat title in OpenWebUI database.

    Args:
        chat_id: The chat ID to update
        title: The new title to set

    Returns:
        True if successful, False otherwise
    """
    if not chat_id or chat_id.startswith("local:"):
        return False
    try:
        from open_webui.models.chats import Chats

        result = Chats.update_chat_title_by_id(chat_id, title)
        return result is not None
    except ImportError:
        return False
    except Exception as e:
        if self.valves.ENABLE_LOGGING:
            print(f"Failed to set chat title: {e}")
        return False
```

### Success Criteria

#### Automated Verification
- [ ] Linting passes: `make lint-fix`
- [ ] Type checking passes: `make check-agent`
- [ ] Existing tests pass: `make test-agent`
- [ ] New tests pass: `uv run pytest tests/test_pipe.py -k set_chat_title -v`

#### Manual Verification
- [ ] In OpenWebUI, send a message to create a chat
- [ ] Verify title can be updated programmatically (via test or debug endpoint)

---

## Phase 4.2: Add Tests

### Overview

Add unit tests for the new `_set_chat_title()` method.

### Changes Required

#### 1. Add Tests
**File**: `tests/test_pipe.py`

Add to `TestGetChatTitle` class or create new `TestSetChatTitle` class:

```python
class TestSetChatTitle:
    """Tests for _set_chat_title method."""

    def test_no_chat_id(self) -> None:
        """Returns False when no chat_id provided."""
        pipe = Pipe()
        assert pipe._set_chat_title(None, "New Title") is False

    def test_local_chat_id(self) -> None:
        """Returns False for local: prefixed IDs."""
        pipe = Pipe()
        assert pipe._set_chat_title("local:123", "New Title") is False

    def test_openwebui_import_error(self, monkeypatch: pytest.MonkeyPatch) -> None:
        """Returns False when OpenWebUI not available."""
        pipe = Pipe()

        def mock_import(*args: Any, **kwargs: Any) -> None:
            raise ImportError("No module named 'open_webui'")

        monkeypatch.setattr("builtins.__import__", mock_import)
        assert pipe._set_chat_title("chat-123", "New Title") is False

    def test_successful_update(self, monkeypatch: pytest.MonkeyPatch) -> None:
        """Returns True when update succeeds."""
        pipe = Pipe()

        class MockChats:
            @staticmethod
            def update_chat_title_by_id(chat_id: str, title: str) -> object:
                return object()  # Non-None = success

        mock_module = type("Module", (), {"Chats": MockChats})()
        monkeypatch.setitem(sys.modules, "open_webui.models.chats", mock_module)

        assert pipe._set_chat_title("chat-123", "New Title") is True

    def test_update_returns_none(self, monkeypatch: pytest.MonkeyPatch) -> None:
        """Returns False when update returns None (chat not found)."""
        pipe = Pipe()

        class MockChats:
            @staticmethod
            def update_chat_title_by_id(chat_id: str, title: str) -> None:
                return None

        mock_module = type("Module", (), {"Chats": MockChats})()
        monkeypatch.setitem(sys.modules, "open_webui.models.chats", mock_module)

        assert pipe._set_chat_title("chat-123", "New Title") is False
```

### Success Criteria

#### Automated Verification
- [ ] All tests pass: `make verify-agent`
- [ ] New tests specifically pass: `uv run pytest tests/test_pipe.py::TestSetChatTitle -v`

---

## Testing Strategy

### Unit Tests
- No chat_id → returns False
- Local chat_id → returns False
- ImportError (no OpenWebUI) → returns False
- Successful update → returns True
- Update returns None → returns False

### Integration Tests
- None needed for this minimal scope

### Manual Testing Steps
1. Start OpenWebUI and HTTP service
2. Create a new chat
3. Call `_set_chat_title()` with new title
4. Verify title updated in OpenWebUI UI

---

## Design Decisions Made

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Location | Pipe method | Direct DB access, mirrors `_get_chat_title()` |
| Return type | `bool` | Simple success/failure, no need for rich errors |
| Error handling | Silent with logging | Consistent with `_get_chat_title()` pattern |
| HTTP endpoint | Not added | Can add later if service needs to call it |

---

## References

- Current implementation: `src/letta_starter/pipelines/letta_pipe.py:57-71`
- OpenWebUI API: `Chats.update_chat_title_by_id()` in `open_webui/models/chats.py:325-333`
- Master plan: `thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md`
