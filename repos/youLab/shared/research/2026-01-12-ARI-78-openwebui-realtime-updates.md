---
date: 2026-01-12T03:13:31Z
researcher: Claude (Opus 4.5)
git_commit: 5921af254188eca4151966cebff1e202da4db885
branch: main
repository: YouLab
topic: "OpenWebUI Real-Time Update Patterns"
tags: [research, codebase, websocket, socket-io, sse, notifications, badges, real-time]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude (Opus 4.5)
linear_ticket: ARI-78
---

# Research: OpenWebUI Real-Time Update Patterns

**Date**: 2026-01-12T03:13:31Z
**Researcher**: Claude (Opus 4.5)
**Git Commit**: 5921af254188eca4151966cebff1e202da4db885
**Branch**: main
**Repository**: YouLab

## Research Question

How does OpenWebUI handle real-time updates? Specifically:
1. Does OpenWebUI use WebSockets? Where and how?
2. How does chat streaming work (SSE vs WebSocket)?
3. How are notifications/badges currently implemented?
4. What real-time update patterns exist in the codebase?
5. How to push updates from backend to frontend?

## Summary

OpenWebUI uses a **dual-transport architecture** for real-time communication:

1. **Socket.IO** (python-socketio) for bidirectional real-time events - channels, typing indicators, presence, notifications
2. **Server-Sent Events (SSE)** for unidirectional chat streaming - LLM response streaming

The system supports horizontal scaling via Redis-backed Socket.IO with distributed state. Notifications use svelte-sonner toasts with native browser notification support. Badge counts are managed through Svelte stores with real-time Socket.IO updates.

## Detailed Findings

### 1. WebSocket/Socket.IO Implementation

#### Backend Setup
**File**: `OpenWebUI/open-webui/backend/open_webui/socket/main.py`

Socket.IO server initialization (lines 62-97):
```python
sio = socketio.AsyncServer(
    cors_allowed_origins=SOCKETIO_CORS_ORIGINS,
    async_mode="asgi",
    transports=(["websocket"] if ENABLE_WEBSOCKET_SUPPORT else ["polling"]),
    allow_upgrades=ENABLE_WEBSOCKET_SUPPORT,
    always_connect=True,
)
```

**Redis Mode** (lines 62-85): When `WEBSOCKET_MANAGER == "redis"`, creates `AsyncRedisManager` for multi-instance deployments.

**ASGI Mounting** (lines 217-220):
```python
app = socketio.ASGIApp(
    sio,
    socketio_path="/ws/socket.io",
)
```
Mounted at `/ws` in `main.py:1372`.

#### Connection Lifecycle
- **Connect** (lines 300-314): Validates JWT token, stores user in `SESSION_POOL[sid]`, joins user to personal room `user:{user_id}`
- **User Join** (lines 316-349): Re-authenticates, joins all user's channels `channel:{channel_id}`
- **Disconnect** (lines 682-691): Removes from `SESSION_POOL`, cleans up collaborative documents
- **Heartbeat** (lines 352-356): Client sends every 30 seconds, updates `last_active` timestamp

#### Room-Based Communication
Room naming patterns:
- `user:{user_id}` - Personal user room (notifications, chat updates)
- `channel:{channel_id}` - Group/DM chat room
- `note:{note_id}` - Note viewing room
- `doc_{document_id}` - Collaborative editing room

Helper functions (lines 236-283):
- `emit_to_users(event, data, user_ids)` - Emit to specific users
- `enter_room_for_users(room, user_ids)` - Batch room joining
- `get_user_ids_from_room(room)` - List users in room

### 2. Chat Streaming (SSE)

#### Transport
Chat responses use **SSE** (Server-Sent Events), not WebSocket.

**File**: `OpenWebUI/open-webui/backend/open_webui/routers/openai.py` (lines 942-952)
```python
if "text/event-stream" in r.headers.get("Content-Type", ""):
    streaming = True
    return StreamingResponse(
        stream_chunks_handler(r.content),
        status_code=r.status,
        headers=dict(r.headers),
    )
```

#### SSE Format
**File**: `OpenWebUI/open-webui/backend/open_webui/utils/response.py` (lines 102-128)
```
data: {"id": "...", "choices": [{"delta": {"content": "text"}}]}\n\n
data: [DONE]\n\n
```

#### Event Emitter (for status/metadata during streaming)
**File**: `OpenWebUI/open-webui/backend/open_webui/socket/main.py` (lines 693-810)

`get_event_emitter()` creates a closure that:
1. Emits events to `user:{user_id}` room via Socket.IO
2. Updates database with streaming content (status, message, replace, files, sources)

Event types:
- `status` - Progress indicators ("Creating image", "Done")
- `message` - Incremental content append
- `replace` - Full content replacement
- `files` - Attached files/images
- `embeds` - Embedded content
- `source`/`citation` - Reference sources

#### Frontend Stream Handling
**File**: `OpenWebUI/open-webui/src/lib/apis/streaming/index.ts` (lines 28-93)
```typescript
export async function createOpenAITextStream(
    responseBody: ReadableStream<Uint8Array>,
    splitLargeDeltas: boolean
): Promise<AsyncGenerator<TextStreamUpdate>> {
    const eventStream = responseBody
        .pipeThrough(new TextDecoderStream())
        .pipeThrough(new EventSourceParserStream())
        .getReader();
    // ... async generator
}
```

### 3. Notification System

#### Toast Notifications
**File**: `OpenWebUI/open-webui/src/lib/components/NotificationToast.svelte`

Uses svelte-sonner library:
- Custom toast component with favicon, title, markdown content
- Notification sound from `/audio/notification.mp3`
- 15-second duration
- Theme-aware (dark/light)
- Drag-detection to differentiate clicks from swipes

**Trigger Pattern** (`+layout.svelte:366-376`):
```javascript
toast.custom(NotificationToast, {
  componentProps: {
    onClick: () => { goto(`/c/${chatId}`); },
    content: messageContent,
    title: messageTitle
  },
  duration: 15000,
  unstyled: true
});
```

#### Browser Notifications
**File**: `OpenWebUI/open-webui/src/routes/+layout.svelte` (lines 357-364, 560-567)

When `$settings?.notificationEnabled` and user is last active tab:
```javascript
new Notification(title, { body, icon })
```

#### Settings
- `notificationSound` - Play sound on notifications
- `notificationSoundAlways` - Always play sound, even for active chats
- `notificationEnabled` - Enable browser notifications
- `playingNotificationSound` - Global flag to prevent overlapping sounds

### 4. Badge/Count Patterns

#### Unread Count Badge
**File**: `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ChannelItem.svelte` (lines 172-181)
```svelte
{#if channel?.unread_count > 0}
  <div class="text-xs py-[1px] px-2 rounded-xl
              bg-gray-100 text-black
              dark:bg-gray-800 dark:text-white
              font-medium">
    {new Intl.NumberFormat($i18n.locale, {
      notation: 'compact',
      compactDisplay: 'short'
    }).format(channel.unread_count)}
  </div>
{/if}
```

#### Unread Count Update Flow
**File**: `OpenWebUI/open-webui/src/routes/+layout.svelte` (lines 524-539)
1. Channel message event received via Socket.IO
2. `channels` store updated: `unread_count: (ch.unread_count ?? 0) + 1`
3. ChannelItem re-renders with badge
4. On click, count reset to 0 and `last_read_at` emitted to server

#### Active Status Indicator
**Pattern** (found in multiple files):
```svelte
<span class="relative flex size-2">
  {#if isActive}
    <span class="animate-ping absolute inline-flex h-full w-full
                 rounded-full bg-green-400 opacity-75"></span>
  {/if}
  <span class="relative inline-flex size-2 rounded-full
               {isActive ? 'bg-green-500' : 'bg-gray-300'}"></span>
</span>
```

Files using this pattern:
- `ChannelItem.svelte:104-119` - DM active status
- `UserMenu.svelte:110-130` - User active status
- `UserStatus.svelte:56` - Message user status

### 5. Backend-to-Frontend Push Patterns

#### Pattern A: Room-Based Broadcast
For channel messages, notifications to groups:
```python
await sio.emit(
    "events:channel",
    event_data,
    to=f"channel:{channel.id}",
)
```

#### Pattern B: User-Targeted Push
For personal notifications, chat updates:
```python
await emit_to_users(
    "events:channel",
    {"data": {"type": "channel:created"}},
    user_ids,
)
```

#### Pattern C: Event Emitter Closure
For chat streaming metadata:
```python
__event_emitter__ = get_event_emitter(request_info)
await __event_emitter__({"type": "status", "data": {"description": "Done", "done": True}})
```

#### Pattern D: SSE Streaming
For LLM response content:
```python
return StreamingResponse(
    stream_chunks_handler(r.content),
    media_type="text/event-stream",
)
```

### 6. Svelte Store Integration

**File**: `OpenWebUI/open-webui/src/lib/stores/index.ts`

Key stores:
- `socket: Writable<null | Socket>` (line 28) - Global socket instance
- `channels: Writable<Channel[]>` - Channel list with unread counts
- `activeUserIds: Writable<null | string[]>` (line 29) - Active user tracking
- `USAGE_POOL: Writable<null | string[]>` (line 30) - Model usage tracking

**Reactive Event Registration** (`+layout.svelte:696-721`):
```javascript
user.subscribe(async (value) => {
    if (value) {
        $socket?.off('events', chatEventHandler);
        $socket?.on('events', chatEventHandler);
        $socket?.off('events:channel', channelEventHandler);
        $socket?.on('events:channel', channelEventHandler);
    } else {
        $socket?.off('events', chatEventHandler);
        $socket?.off('events:channel', channelEventHandler);
    }
});
```

### 7. Collaborative Editing (Yjs)

**Backend**: `socket/main.py` (lines 446-679)
- `ydoc:document:join` - Join document room
- `ydoc:document:update` - Broadcast Yjs updates
- `ydoc:document:state` - Send full document state
- `ydoc:awareness:update` - Cursor/selection sync

**Frontend**: `src/lib/components/common/RichTextInput/Collaboration.ts`
- Custom `SocketIOCollaborationProvider` wraps Socket.IO for Yjs
- `SimpleAwareness` class for cursor tracking

## Code References

### Backend
- `OpenWebUI/open-webui/backend/open_webui/socket/main.py` - Socket.IO server, event handlers, event emitter
- `OpenWebUI/open-webui/backend/open_webui/socket/utils.py` - RedisDict, RedisLock, YdocManager
- `OpenWebUI/open-webui/backend/open_webui/routers/openai.py:942-952` - SSE streaming detection
- `OpenWebUI/open-webui/backend/open_webui/routers/channels.py` - Channel event emissions
- `OpenWebUI/open-webui/backend/open_webui/utils/response.py:102-128` - SSE format conversion
- `OpenWebUI/open-webui/backend/open_webui/utils/misc.py:653-719` - Stream chunk handler

### Frontend
- `OpenWebUI/open-webui/src/routes/+layout.svelte` - Socket setup, event handlers, notifications
- `OpenWebUI/open-webui/src/lib/stores/index.ts:28-30` - Socket and state stores
- `OpenWebUI/open-webui/src/lib/apis/streaming/index.ts` - SSE parsing
- `OpenWebUI/open-webui/src/lib/components/NotificationToast.svelte` - Toast component
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ChannelItem.svelte:172-181` - Badge display
- `OpenWebUI/open-webui/src/lib/components/channel/Channel.svelte` - Channel event handling
- `OpenWebUI/open-webui/src/lib/components/common/RichTextInput/Collaboration.ts` - Yjs provider

### Configuration
- `ENABLE_WEBSOCKET_SUPPORT` (env.py:612) - Enable WebSocket transport
- `WEBSOCKET_MANAGER` (env.py:617) - Set to "redis" for distributed mode
- `WEBSOCKET_REDIS_URL` (env.py:637) - Redis connection URL

## Architecture Documentation

### Transport Decision Matrix
| Use Case | Transport | Event Pattern |
|----------|-----------|---------------|
| LLM response streaming | SSE | `StreamingResponse` |
| Chat metadata (status, files) | Socket.IO | `events` to `user:{id}` |
| Channel messages | Socket.IO | `events:channel` to room |
| Typing indicators | Socket.IO | `events:channel` with `type: typing` |
| Presence/heartbeat | Socket.IO | `heartbeat` event |
| Collaborative editing | Socket.IO | `ydoc:*` events |
| Browser notifications | Native API | `new Notification()` |

### Notification Flow
```
Socket Event → channelEventHandler → Check conditions →
  → Toast notification (svelte-sonner)
  → Browser notification (if enabled)
  → Sound (if enabled)
  → Badge update (Svelte store)
```

### Scaling Architecture
```
Client A ──┐
Client B ──┼── Socket.IO Server 1 ──┐
Client C ──┘                        │
                                    ├── Redis Pub/Sub
Client D ──┐                        │
Client E ──┼── Socket.IO Server 2 ──┘
Client F ──┘
```

## Use Cases for ARI-78

### 1. Update diff badge count when background agent proposes change
**Approach**: Emit Socket.IO event to user room
```python
# Backend: After background agent creates diff
await sio.emit(
    "events:memory",
    {"type": "diff:pending", "data": {"count": pending_count, "block_id": block_id}},
    room=f"user:{user_id}",
)
```
```javascript
// Frontend: Listen and update store
$socket?.on('events:memory', (event) => {
    if (event.type === 'diff:pending') {
        pendingDiffs.update(d => ({ ...d, [event.data.block_id]: event.data.count }));
    }
});
```

### 2. Real-time thread updates when agent responds
**Approach**: Use existing `events:channel` pattern or create new `events:agent-thread`
```python
await sio.emit(
    "events:agent-thread",
    {"type": "message:new", "data": {"thread_id": thread_id, "message": message_data}},
    room=f"user:{user_id}",
)
```

### 3. Notification when memory block is modified
**Approach**: Toast + optional browser notification
```javascript
// Frontend handler
if (event.type === 'memory:updated') {
    toast.custom(NotificationToast, {
        componentProps: {
            title: 'Memory Updated',
            content: `${event.data.block_name} was modified`,
            onClick: () => goto('/you/profile')
        },
        duration: 15000
    });
}
```

## Open Questions

1. **Event namespace**: Should background agent events use `events:memory` or extend `events:channel`?
2. **Persistence**: Should pending diff counts be stored in database or memory-only?
3. **Multi-tab sync**: How to handle badge counts across multiple browser tabs?
4. **Redis requirement**: Is Redis-backed Socket.IO required for production, or can local mode work?

## Related Research

- `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Product spec defining requirements
