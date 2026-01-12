---
date: 2026-01-12T10:32:21+07:00
researcher: Claude Code
git_commit: 4c099b8eefca5a1c178847faf034af8047936438
branch: chore/coderabbit-setup
repository: YouLab
topic: "OpenWebUI Editor Components for Memory Block Editing"
tags: [research, codebase, openwebui, tiptap, editors, memory-blocks]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude Code
---

# Research: OpenWebUI Editor Components for Memory Block Editing

**Date**: 2026-01-12T10:32:21+07:00
**Researcher**: Claude Code
**Git Commit**: 4c099b8eefca5a1c178847faf034af8047936438
**Branch**: chore/coderabbit-setup
**Repository**: YouLab

## Research Question

What markdown/rich-text editor does OpenWebUI use, where are editor components located, and how can we reuse them for the "You" section memory block editing?

## Summary

OpenWebUI uses **TipTap 3.x** (built on ProseMirror) as its primary rich-text editor and **CodeMirror 6** for code editing. The main reusable component is `RichTextInput.svelte` which provides a highly configurable TipTap wrapper with support for markdown bidirectional conversion, collaboration, and various editing features. For the "You" section memory block editing, we can reuse this component with simplified configuration (no collaboration, basic formatting only).

## Detailed Findings

### 1. Editor Libraries Installed

#### TipTap v3 Ecosystem (Primary Rich-Text Editor)
| Package | Version | Purpose |
|---------|---------|---------|
| @tiptap/core | 3.0.7 | Core editor framework |
| @tiptap/starter-kit | 3.0.7 | Basic editing features bundle |
| @tiptap/pm | 3.0.7 | ProseMirror integration |
| @tiptap/extensions | 3.0.7 | Extension utilities |
| @tiptap/suggestion | 3.4.2 | Mention/autocomplete support |

#### TipTap v3 Extensions
- `@tiptap/extension-code-block-lowlight` - Syntax highlighting
- `@tiptap/extension-highlight` - Text highlighting
- `@tiptap/extension-image` - Image insertion
- `@tiptap/extension-link` - Link support
- `@tiptap/extension-list` - Advanced lists
- `@tiptap/extension-mention` - @mentions
- `@tiptap/extension-table` - Table editing
- `@tiptap/extension-typography` - Smart typography

#### CodeMirror 6 (Code Editing)
- `codemirror` v6.0.1 - Core editor
- `@codemirror/lang-javascript` - JavaScript support
- `@codemirror/lang-python` - Python support
- `@codemirror/theme-one-dark` - Dark theme

#### Supporting Libraries
- `marked` v9.1.0 - Markdown → HTML
- `turndown` v7.2.0 - HTML → Markdown

### 2. Editor Component Locations

#### Primary Rich Text Editor
```
OpenWebUI/open-webui/src/lib/components/common/
├── RichTextInput.svelte           # Main TipTap wrapper (USE THIS)
├── RichTextInput/
│   ├── FormattingButtons.svelte   # Toolbar buttons
│   ├── AutoCompletion.js          # AI completion extension
│   ├── Collaboration.ts           # Real-time collab
│   ├── commands.ts                # Slash commands
│   ├── suggestions.ts             # Mention suggestions
│   ├── listDragHandlePlugin.js    # List drag handles
│   └── Image/                     # Custom image handling
├── CodeEditor.svelte              # CodeMirror wrapper
├── CodeEditorModal.svelte         # Code editor in modal
└── Textarea.svelte                # Basic textarea
```

#### Usage Examples
```
OpenWebUI/open-webui/src/lib/components/
├── notes/NoteEditor.svelte        # Full-featured editor
├── chat/MessageInput.svelte       # Chat input
├── chat/Messages/UserMessage.svelte  # Message editing
└── workspace/Prompts/PromptEditor.svelte
```

### 3. RichTextInput Props and Configuration

**Location**: `OpenWebUI/open-webui/src/lib/components/common/RichTextInput.svelte`

#### Key Props
```typescript
export let id = '';                    // Element ID
export let className = '';             // CSS classes
export let placeholder = '';           // Placeholder text
export let value = '';                 // Initial content (markdown)
export let html = '';                  // Initial content (HTML)
export let json = false;               // Use JSON mode
export let editable = true;            // Allow editing
export let richText = true;            // Rich text vs plain text mode
export let messageInput = false;       // Chat input mode
export let showFormattingToolbar = true; // Show bubble menu
export let dragHandle = false;         // List drag handles
export let link = true;                // Link support
export let image = false;              // Image support
export let fileHandler = false;        // File upload
export let suggestions = null;         // Mention suggestions
export let autocomplete = false;       // AI completion
export let collaboration = false;      // Real-time collab
export let socket = null;              // Socket.IO instance
export let user = null;                // User for collab
export let documentId = '';            // Collab document ID
export let files = [];                 // Attached files
```

#### Events
```typescript
export let onChange = ({ html, json, md }) => {};  // Content changed
export let onBlur = () => {};                       // Editor blurred
export let onFocus = () => {};                      // Editor focused
```

#### Output Formats
The editor provides three synchronized formats:
- `html` - TipTap HTML output
- `json` - ProseMirror JSON structure
- `md` - Markdown via Turndown conversion

### 4. Recommended Configuration for Memory Block Editing

For the "You" section memory blocks, use a simplified configuration:

```svelte
<script>
import RichTextInput from '$lib/components/common/RichTextInput.svelte';

let content = ''; // Markdown content
let editor;

function handleChange({ md }) {
  content = md;
  // Debounce and save to backend
}
</script>

<RichTextInput
  bind:editor
  id="memory-block-{blockId}"
  className="min-h-[100px] border rounded-lg p-2"
  value={content}
  placeholder="Describe yourself..."
  richText={true}
  showFormattingToolbar={true}
  link={false}
  image={false}
  dragHandle={false}
  collaboration={false}
  autocomplete={false}
  onChange={handleChange}
/>
```

### 5. Inline Editing Patterns in OpenWebUI

#### Pattern A: Always-Visible Input (Note Titles)
**Found in**: `NoteEditor.svelte:909-930`

```svelte
<script>
let ignoreBlur = false;
let titleInputFocused = false;

const changeDebounceHandler = () => {
  if (debounceTimeout) clearTimeout(debounceTimeout);
  debounceTimeout = setTimeout(async () => {
    await updateNoteById(localStorage.token, id, { title: note.title });
  }, 200);
};
</script>

<input
  class="w-full text-2xl font-medium bg-transparent outline-hidden"
  bind:value={note.title}
  on:focus={() => titleInputFocused = true}
  on:blur={(e) => {
    if (ignoreBlur) { ignoreBlur = false; return; }
    titleInputFocused = false;
    changeDebounceHandler();
  }}
/>
```

**Use case**: Primary editing interaction, always editable

#### Pattern B: Double-Click to Edit (Chat Titles, Folders)
**Found in**: `ChatItem.svelte:353-431`, `RecursiveFolder.svelte:552-581`

```svelte
<script>
let confirmEdit = false;
let chatTitle = '';

const renameHandler = async () => {
  chatTitle = title;
  confirmEdit = true;
  await tick();
  const input = document.getElementById(`chat-title-input-${id}`);
  input?.focus();
  input?.select();
};
</script>

{#if confirmEdit}
  <input
    id="chat-title-input-{id}"
    bind:value={chatTitle}
    class="bg-transparent outline-hidden"
    on:keydown={(e) => {
      if (e.key === 'Enter') { /* save */ }
      if (e.key === 'Escape') { confirmEdit = false; }
    }}
    on:blur={async () => {
      if (chatTitle !== title) await editChatTitle(id, chatTitle);
      confirmEdit = false;
    }}
  />
{:else}
  <div on:dblclick={renameHandler}>{title}</div>
{/if}
```

**Use case**: Occasional editing, space-efficient

#### Pattern C: Textarea with Buttons (Message Editing)
**Found in**: `UserMessage.svelte:219-342`

```svelte
{#if edit}
  <div class="bg-gray-50 rounded-3xl px-5 py-3">
    <textarea
      bind:value={editedContent}
      on:input={(e) => {
        e.target.style.height = `${e.target.scrollHeight}px`;
      }}
      on:keydown={(e) => {
        if (e.key === 'Escape') cancelEdit();
        if ((e.metaKey || e.ctrlKey) && e.key === 'Enter') confirmEdit();
      }}
    />
    <div class="flex justify-between">
      <button on:click={() => confirmEdit(false)}>Save</button>
      <button on:click={cancelEdit}>Cancel</button>
      <button on:click={() => confirmEdit(true)}>Send</button>
    </div>
  </div>
{:else}
  <Markdown content={message.content} />
{/if}
```

**Use case**: Multi-line content, explicit save/cancel

### 6. Recommendations for "You" Section Implementation

Based on the codebase patterns, here's the recommended approach:

#### For Memory Block Editing
1. **Use `RichTextInput`** with simplified config (no collab, no AI, basic formatting)
2. **Pattern B (toggle edit)** for block titles
3. **Pattern A (always visible) or RichTextInput** for block content
4. **Auto-save with debounce** (200ms) on blur

#### Minimal Implementation
```svelte
<script>
import RichTextInput from '$lib/components/common/RichTextInput.svelte';

export let block; // { id, title, content }
let editing = false;
let saveTimeout;

async function saveContent(md) {
  clearTimeout(saveTimeout);
  saveTimeout = setTimeout(async () => {
    await updateMemoryBlock(block.id, { content: md });
  }, 500);
}
</script>

<div class="memory-block border rounded-lg p-4">
  <h3 class="font-semibold mb-2">{block.title}</h3>

  <RichTextInput
    id="block-{block.id}"
    value={block.content}
    richText={true}
    showFormattingToolbar={true}
    link={false}
    image={false}
    onChange={({ md }) => saveContent(md)}
  />
</div>
```

## Code References

### Primary Components
- `OpenWebUI/open-webui/src/lib/components/common/RichTextInput.svelte` - Main editor
- `OpenWebUI/open-webui/src/lib/components/common/RichTextInput/FormattingButtons.svelte` - Toolbar
- `OpenWebUI/open-webui/src/lib/components/common/CodeEditor.svelte` - Code editing

### Usage Examples
- `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte:1150-1167` - Full RichTextInput usage
- `OpenWebUI/open-webui/src/lib/components/chat/MessageInput.svelte:1276-1287` - Simplified usage
- `OpenWebUI/open-webui/src/lib/components/layout/Sidebar/ChatItem.svelte:353-431` - Inline title editing
- `OpenWebUI/open-webui/src/lib/components/notes/NoteEditor.svelte:909-930` - Focus/blur pattern

### TipTap Configuration
- `OpenWebUI/open-webui/src/lib/components/common/RichTextInput.svelte:683-775` - Editor initialization
- `OpenWebUI/open-webui/package.json:67-84` - TipTap dependencies

## Architecture Documentation

### Editor Data Flow
```
User Input
    ↓
TipTap Editor (ProseMirror state)
    ↓
onTransaction callback
    ↓
├── editor.getHTML() → htmlValue
├── editor.getJSON() → jsonValue
└── turndownService.turndown(html) → mdValue
    ↓
onChange({ html, json, md })
    ↓
Parent component saves to backend
```

### Extension Loading Pattern
TipTap extensions are conditionally loaded based on props:
```javascript
extensions: [
  StarterKit,
  ...(richText ? [Typography, TableKit, ListKit] : []),
  ...(image ? [Image] : []),
  ...(collaboration ? [provider.getEditorExtension()] : [])
]
```

## Open Questions

1. **Diff visualization**: How to show memory block changes (additions/deletions)?
2. **Version history UI**: Modal or inline for viewing past versions?
3. **Conflict resolution**: What happens if user edits while background agent proposes changes?
4. **Block reordering**: Should users be able to reorder memory blocks?
