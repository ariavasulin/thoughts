---
date: 2026-02-10T12:00:00-08:00
researcher: ARI
git_commit: cf5dcf7a2195108b0ea2823343599628bd12ed85
branch: main
repository: YouLab
topic: "OpenWebUI Artifact System: Sideloading HTML/PDF without Chat Rendering"
tags: [research, codebase, openwebui, artifacts, pipe, events, sideloading]
status: complete
last_updated: 2026-02-10
last_updated_by: ARI
---

# Research: OpenWebUI Artifact Sideloading

**Date**: 2026-02-10
**Researcher**: ARI
**Git Commit**: cf5dcf7a
**Branch**: main
**Repository**: YouLab

## Research Question

How does OpenWebUI's artifact system work, and can we programmatically force an artifact (PDF/HTML/JS) to display in the side panel without rendering the HTML code block in the chat message itself?

## Summary

OpenWebUI has **two independent rendering paths** for interactive content, both of which can be exploited for sideloading:

1. **Artifacts Panel** (side panel) - Triggered by `html`/`svg` code blocks in messages. Controlled by Svelte stores `showArtifacts`, `artifactContents`, `artifactCode`.
2. **Embeds System** (inline iframes) - Triggered by the `embeds` event type from pipes. Rendered via `FullHeightIframe` component below the message.

The **cleanest hack** for sideloading is **Approach B: the `embeds` event** — it displays HTML in an iframe below the message without requiring any code block in the chat text. For the artifact *side panel* specifically, the content must exist as a code block in message history, but the `replace` event can be used to hide it after rendering.

## Detailed Findings

### How the Artifact System Works (Current Flow)

#### Detection Pipeline

1. Agent emits markdown containing an ` ```html ` code block
2. `Chat.svelte:830-884` reactively calls `getContents()` on every history change
3. `getCodeBlockContents()` (`utils/index.ts:1622-1680`) regex-matches all fenced code blocks
4. HTML/CSS/JS blocks are assembled into full HTML documents
5. `artifactContents.set(contents)` updates the store
6. `ContentRenderer.svelte:179-191` detects `html`/`svg` lang and sets `showArtifacts.set(true)`
7. `Artifacts.svelte` subscribes to the store and renders in a sandboxed iframe

#### Key Stores (`src/lib/stores/index.ts:93-97`)

```typescript
export const showArtifacts = writable(false);      // Panel visibility
export const artifactCode = writable(null);         // Code to highlight/jump to
export const artifactContents = writable(null);     // Array of {type, content}
```

#### Detection Conditions (`ContentRenderer.svelte:182-187`)

- `$settings?.detectArtifacts !== false` (default: enabled)
- Language is `html`, `svg`, or `xml` containing `<svg`
- Not on mobile (`!$mobile`)
- Active chat exists (`$chatId`)

### Sideloading Approaches

#### Approach A: `replace` Event (Artifact Panel Hack)

**Mechanism**: Emit the HTML code block first (triggering artifact detection), then immediately `replace` the message content to remove the code block from the visible chat.

**In Ralph pipe** (`src/ralph/pipe.py`):

```python
# Step 1: Emit HTML block to trigger artifact detection
await __event_emitter__({
    "type": "message",
    "data": {"content": "```html\n<html>...</html>\n```"}
})

# Step 2: Replace message content to hide the code block
await __event_emitter__({
    "type": "replace",
    "data": {"content": "Here's your PDF viewer."}
})
```

**Pros**: Uses the real artifact side panel with version navigation
**Cons**: Race condition — `getContents()` re-scans message history reactively, so the replacement may clear `artifactContents`. Needs testing to confirm timing.

#### Approach B: `embeds` Event (Cleanest Hack)

**Mechanism**: Use the `embeds` event type to inject HTML iframes directly below the message, bypassing the artifact panel entirely.

**In Ralph pipe** (`src/ralph/pipe.py`):

```python
await __event_emitter__({
    "type": "embeds",
    "data": {
        "embeds": ["<html><body>Your PDF viewer here</body></html>"]
    }
})
```

**Backend handling** (`socket/main.py`):
- Persists embeds to database via `Chats.upsert_message_to_chat_by_id_and_message_id()`
- Broadcasts to frontend via socket.io

**Frontend rendering** (`ResponseMessage.svelte`):
- Renders each embed in a `FullHeightIframe` component
- Sandbox supports: `allow-scripts`, `allow-downloads`, `allow-forms`, `allow-same-origin`
- Auto-injects Alpine.js and Chart.js

**Pros**: No code block in chat, persists in database, clean separation
**Cons**: Renders *below* the message (not in side panel), different UX from artifacts

#### Approach C: Direct Store Manipulation (Frontend Hack)

**Mechanism**: Modify the OpenWebUI fork to expose a function/API that sets artifact stores directly.

In your OpenWebUI fork, add a custom event listener in `Chat.svelte`:

```typescript
// In Chat.svelte event handler
if (type === 'artifact' || type === 'chat:artifact') {
    artifactContents.update(contents => [
        ...(contents || []),
        { type: 'iframe', content: data.content }
    ]);
    showArtifacts.set(true);
    showControls.set(true);
}
```

Then from the Ralph pipe:

```python
await __event_emitter__({
    "type": "artifact",  # Custom event type
    "data": {"content": "<html>...</html>"}
})
```

**Pros**: Full control, side panel display, no code block in chat, persists properly
**Cons**: Requires OpenWebUI fork modification (you already have a fork)

#### Approach D: Hidden Code Block (CSS Hack)

**Mechanism**: Include the HTML code block in the message but hide it with CSS.

```python
await __event_emitter__({
    "type": "message",
    "data": {"content": '<div style="display:none">\n\n```html\n<html>...</html>\n```\n\n</div>\n\nHere\'s your PDF viewer.'}
})
```

**Pros**: Simple, no fork changes needed
**Cons**: Markdown parsers may not respect the `<div>` wrapper around fenced blocks. Needs testing — the `getCodeBlockContents()` regex scans raw content, so detection should still work even if hidden.

### Event Emitter System Reference

The pipe event emitter (`socket/main.py:693-834`) supports these event types:

| Event Type | Alias | Effect |
|-----------|-------|--------|
| `message` | `chat:message:delta` | Appends to message content |
| `replace` | `chat:message` | Replaces entire message content |
| `embeds` | `chat:message:embeds` | Adds HTML iframes below message |
| `files` | `chat:message:files` | Attaches files to message |
| `status` | — | Updates status indicator |
| `citations`/`source` | — | Adds source citations |
| `notification` | — | Toast notification |
| `confirmation` | — | Dialog with user input |

### Ralph Integration Points

The Ralph pipe (`src/ralph/pipe.py:132-166`) currently handles these events:
- `status` → emits OpenWebUI status
- `message` → emits message delta
- `done` → emits completion status
- `error` → emits error message

To support artifact sideloading, the pipe would need to emit `embeds` or custom events. The server (`src/ralph/server.py`) would need to yield these new event types from the agent.

### Existing LaTeX Tools Implementation

`src/ralph/tools/latex_tools.py:391-397` currently returns artifacts as markdown code blocks:

```python
return f"""I've created your notes on **{title}**...

```html
{html}
```

You can navigate pages..."""
```

This HTML block contains a self-contained PDF.js viewer with base64-embedded PDF data. The artifact panel renders it automatically.

## Code References

- `OpenWebUI/open-webui/src/lib/stores/index.ts:93-97` - Artifact store definitions
- `OpenWebUI/open-webui/src/lib/components/chat/Chat.svelte:830-885` - `getContents()` artifact extraction
- `OpenWebUI/open-webui/src/lib/utils/index.ts:1622-1680` - `getCodeBlockContents()` parser
- `OpenWebUI/open-webui/src/lib/components/chat/Artifacts.svelte` - Artifact panel component
- `OpenWebUI/open-webui/src/lib/components/chat/Messages/ContentRenderer.svelte:179-200` - Auto-detection and preview handlers
- `OpenWebUI/open-webui/src/lib/components/chat/Messages/CodeBlock.svelte:503-512` - Preview button
- `OpenWebUI/open-webui/backend/open_webui/socket/main.py:693-834` - Event emitter implementation
- `src/ralph/pipe.py:132-166` - Ralph pipe event handler
- `src/ralph/server.py:348-366` - Ralph SSE streaming loop
- `src/ralph/tools/latex_tools.py:391-397` - Current artifact delivery

## Architecture Documentation

```
Approach B (embeds):
  Ralph Tool → returns HTML → Server SSE → Pipe emits "embeds" event
                                                    ↓
                               socket.io → Chat.svelte → ResponseMessage → FullHeightIframe

Approach C (fork hack):
  Ralph Tool → returns HTML → Server SSE → Pipe emits custom "artifact" event
                                                    ↓
                               socket.io → Chat.svelte → artifactContents store → Artifacts.svelte

Current flow (code block):
  Ralph Tool → returns ```html block → Server SSE → Pipe emits "message" event
                                                           ↓
                               Chat.svelte → getContents() → artifactContents store → Artifacts.svelte
                                          → ContentRenderer → showArtifacts.set(true)
```

## Historical Context

- `thoughts/shared/research/2026-01-28-latex-pdf-live-sandbox-artifacts.md` - Deep research on artifact system, four implementation approaches (A-D)
- `thoughts/shared/research/2026-02-05-latex-artifacts-persistent-html-rendering.md` - LaTeX PDF implementation status
- `thoughts/shared/plans/2026-01-28-latex-pdf-notes-tool.md` - Complete implementation plan for LaTeX tools

## Related Research

- `thoughts/shared/research/2026-01-24-openwebui-api-pipes-reference.md`
- `thoughts/shared/research/2026-01-12-ARI-78-pipe-pipeline-architecture.md`
- `thoughts/shared/research/2026-01-24-openwebui-youlab-patches.md`

## Recommendation Summary

| Approach | Side Panel? | No Code Block? | Fork Needed? | Complexity |
|----------|------------|----------------|-------------|------------|
| A: replace | Yes | Yes (race risk) | No | Medium |
| B: embeds | No (inline) | Yes | No | Low |
| C: custom event | Yes | Yes | Yes (minor) | Medium |
| D: CSS hide | Yes | Visually yes | No | Low |

**For side panel display without chat rendering**: Approach C (fork hack) is the most reliable. Since YouLab already maintains an OpenWebUI fork, adding a custom `artifact` event handler in `Chat.svelte` is a ~10 line change.

**For quick wins without fork changes**: Approach B (embeds) works immediately from the pipe with zero frontend changes, though content renders inline rather than in the side panel.
