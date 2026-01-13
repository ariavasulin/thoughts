# ARI-82: Memory Blocks as Notes Architecture - Implementation Plan

## Overview

This plan implements a unified architecture where memory blocks are treated as a special type of note in OpenWebUI. Based on comprehensive research (8 documents in `thoughts/shared/research/2026-01-13-ARI-82-*.md`), this plan prioritizes enhancing the existing BlockDetailModal over adapting Notes components, leveraging git-backed versioning as the more robust solution.

## Current State Analysis

### What Exists (ARI-80)

| Component | Implementation | Location |
|-----------|----------------|----------|
| Git Storage | Per-user repos with full version history | `src/youlab_server/storage/git.py:41-360` |
| Block Manager | TOML↔MD conversion, Letta sync, pending diffs | `src/youlab_server/storage/blocks.py:21-395` |
| Diff Approval | PendingDiffStore with approve/reject workflow | `src/youlab_server/storage/diffs.py:19-144` |
| Server API | Full CRUD + history + diff endpoints | `src/youlab_server/server/blocks.py:109-274` |
| Frontend | BlockCard + BlockDetailModal with diff view | `OpenWebUI/open-webui/src/lib/components/you/` |
| Background Agents | Runner + TOML config, manual triggers only | `src/youlab_server/background/runner.py` |

### Key Research Findings

1. **Notes Components Not Suitable** (research: `openwebui-notes-architecture.md`)
   - Notes uses TipTap rich editor + WebSocket collaboration
   - BlockDetailModal already has better architecture for structured data
   - Recommendation: Enhance existing components, adopt only autosave pattern

2. **Hidden Models Work** (research: `openwebui-models-chat-folders.md`)
   - `meta.hidden=true` filtering exists and works
   - `meta.type="background_agent"` already filtered from tag groups
   - Gap: No auto-folder creation or auto-chat assignment

3. **Freeform Mode Missing** (research: `toml-md-conversion.md`)
   - Current converter assumes multi-field structure
   - Single `body` field produces awkward `## Body` section
   - Need passthrough mode for freeform content

4. **Letta Sync Unidirectional** (research: `letta-memory-integration.md`)
   - Git → Letta only via `_sync_block_to_letta()`
   - No Letta → Git sync (not needed for this spec)
   - Block naming: `youlab_user_{user_id}_{label}`

## Desired End State

After implementation:

1. **Memory Blocks** default to freeform (single body field) unless schema specified in course config
2. **User Storage** follows new schema: `.data/{user-id}/memory-blocks/`, `notes/`, `files/`, `courses/`
3. **Modules** persist as visible models with `base_model_id = youlab-pipe`
4. **Background Agents** persist as hidden models (`meta.hidden=true`)
5. **Chat Folders** auto-created and auto-assigned per model
6. **Autosave** enabled with 200ms debounce like Notes
7. **Undo/Redo** buttons in BlockDetailModal navigating git history

### Verification

```bash
# User storage structure
ls .data/test-user-id/
# Expected: config.toml, memory-blocks/, notes/, files/, courses/

# Module appears as model
curl -s $OPENWEBUI_URL/api/models | jq '.[] | select(.base_model_id == "youlab")'
# Expected: visible modules with YouLab as base

# Background agent hidden
curl -s $OPENWEBUI_URL/api/models | jq '.[] | select(.meta.hidden == true)'
# Expected: background agent models with hidden=true

# Autosave working
# Edit block in UI, wait 200ms, check git log
git -C .data/test-user-id log --oneline -1
# Expected: "Autosave {block_label}"
```

## What We're NOT Doing

1. **Not replacing BlockDetailModal with NoteEditor** - BlockDetailModal is better suited
2. **Not implementing real-time collaboration** - Not needed for memory blocks
3. **Not implementing bidirectional Letta sync** - Git remains source of truth
4. **Not implementing rich text editing** - Keep simple Textarea for now
5. **Not implementing background agent triggers** - Deferred to future ticket
6. **Not migrating to Markdown-first storage** - Keep TOML source with freeform mode

## Implementation Approach

The implementation follows a bottom-up approach:
1. Backend storage changes first (directory structure, freeform conversion)
2. API updates to support new patterns
3. Frontend enhancements (autosave, undo/redo)
4. OpenWebUI integrations (models, folders)

---

## Phase 1: User Storage Schema Migration

### Overview

Migrate user storage from `blocks/` to new schema with `memory-blocks/`, `notes/`, `files/`, and `courses/` directories. Maintain backward compatibility during transition.

### Changes Required:

#### 1. Update GitUserStorage

**File**: `src/youlab_server/storage/git.py`

Add new directory properties and migration logic:

```python
# After line 57, add new properties:
@property
def memory_blocks_dir(self) -> Path:
    """Memory blocks directory (new schema)."""
    return self.user_dir / "memory-blocks"

@property
def notes_dir(self) -> Path:
    """User notes directory."""
    return self.user_dir / "notes"

@property
def files_dir(self) -> Path:
    """User files directory."""
    return self.user_dir / "files"

@property
def courses_dir(self) -> Path:
    """Course symlinks directory."""
    return self.user_dir / "courses"

# Update init() around line 93-105:
def init(self, migrate_blocks: bool = True) -> None:
    """Initialize user storage directories and git repo."""
    # Create new directory structure
    self.memory_blocks_dir.mkdir(parents=True, exist_ok=True)
    self.notes_dir.mkdir(exist_ok=True)
    self.files_dir.mkdir(exist_ok=True)
    (self.files_dir / "private").mkdir(exist_ok=True)
    (self.files_dir / "shared").mkdir(exist_ok=True)
    (self.files_dir / "archive").mkdir(exist_ok=True)
    self.courses_dir.mkdir(exist_ok=True)

    # Migrate from blocks/ to memory-blocks/ if needed
    if migrate_blocks and self.blocks_dir.exists():
        self._migrate_blocks()

    # ... rest of init

def _migrate_blocks(self) -> None:
    """Migrate blocks/ to memory-blocks/ for existing users."""
    old_dir = self.user_dir / "blocks"
    if not old_dir.exists():
        return

    for toml_file in old_dir.glob("*.toml"):
        dest = self.memory_blocks_dir / toml_file.name
        if not dest.exists():
            toml_file.rename(dest)

    # Keep empty blocks/ for backward compat, or remove if empty
    if not any(old_dir.iterdir()):
        old_dir.rmdir()
```

#### 2. Update blocks_dir property

**File**: `src/youlab_server/storage/git.py`

Update the `blocks_dir` property to prefer new location:

```python
@property
def blocks_dir(self) -> Path:
    """Memory blocks directory (prefers new schema)."""
    new_dir = self.user_dir / "memory-blocks"
    if new_dir.exists():
        return new_dir
    # Fall back to legacy location
    return self.user_dir / "blocks"
```

#### 3. Add config.toml support

**File**: `src/youlab_server/storage/git.py`

Add user config file handling:

```python
@property
def config_file(self) -> Path:
    """User configuration file."""
    return self.user_dir / "config.toml"

def get_user_config(self) -> dict[str, Any]:
    """Read user configuration."""
    if not self.config_file.exists():
        return {}
    return tomllib.loads(self.config_file.read_text())

def update_user_config(self, config: dict[str, Any]) -> None:
    """Update user configuration."""
    import tomli_w
    self.config_file.write_text(tomli_w.dumps(config))
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] New user init creates correct structure:
  ```python
  # In test
  storage = GitUserStorage(Path(".data/test-user"))
  storage.init()
  assert (storage.user_dir / "memory-blocks").exists()
  assert (storage.user_dir / "notes").exists()
  assert (storage.user_dir / "files" / "private").exists()
  ```
- [ ] Existing user migration works: blocks/ → memory-blocks/

#### Manual Verification:
- [ ] Create new user via API, verify directory structure
- [ ] Migrate existing test user, verify blocks accessible

---

## Phase 2: Freeform Block Support

### Overview

Add freeform mode to TOML↔MD conversion. Freeform blocks have a single `body` field that passes through without section wrapping.

### Changes Required:

#### 1. Update TOML→Markdown conversion

**File**: `src/youlab_server/storage/convert.py`

Replace `toml_to_markdown` function (lines 12-71):

```python
def toml_to_markdown(toml_content: str, label: str, schema: dict | None = None) -> str:
    """
    Convert TOML content to Markdown for editing.

    If the block has a single 'body' key (or schema specifies freeform),
    pass through without section wrapping.
    """
    # Frontmatter header
    lines = [
        "---",
        f"block: {label}",
        "---",
        "",
    ]

    try:
        data = tomllib.loads(toml_content)
    except Exception:
        # Invalid TOML - show raw with error flag
        lines.insert(2, "error: invalid_toml")
        lines.append("```toml")
        lines.append(toml_content)
        lines.append("```")
        return "\n".join(lines)

    # Check if freeform: single 'body' key or schema says freeform
    is_freeform = (
        len(data) == 1 and "body" in data
    ) or (schema and schema.get("freeform", False))

    if is_freeform and "body" in data:
        # Freeform mode: pass through body directly
        lines.append(data["body"])
    else:
        # Multi-field mode: each key becomes a section
        for key, value in data.items():
            title = key.replace("_", " ").title()
            lines.append(f"## {title}")
            lines.append("")

            if value is None:
                lines.append("*(not set)*")
            elif isinstance(value, bool):
                lines.append("Yes" if value else "No")
            elif isinstance(value, list):
                for item in value:
                    lines.append(f"- {item}")
            else:
                lines.append(str(value))

            lines.append("")

    return "\n".join(lines).rstrip() + "\n"
```

#### 2. Update Markdown→TOML conversion

**File**: `src/youlab_server/storage/convert.py`

Update `markdown_to_toml` function (lines 74-133):

```python
def markdown_to_toml(markdown_content: str, schema: dict | None = None) -> tuple[str, dict[str, Any]]:
    """
    Convert Markdown back to TOML.

    Returns (toml_string, metadata_dict).
    """
    lines = markdown_content.split("\n")

    # Extract frontmatter
    metadata = {}
    content_start = 0
    if lines and lines[0].strip() == "---":
        for i, line in enumerate(lines[1:], start=1):
            if line.strip() == "---":
                content_start = i + 1
                break
            if ":" in line:
                key, _, value = line.partition(":")
                metadata[key.strip()] = value.strip()

    content_lines = lines[content_start:]

    # Check if schema specifies freeform
    is_freeform = schema and schema.get("freeform", False)

    # Detect if content has sections (## headings)
    has_sections = any(line.strip().startswith("## ") for line in content_lines)

    if is_freeform or not has_sections:
        # Freeform mode: wrap all content in body field
        body = "\n".join(content_lines).strip()
        return f'body = """\n{body}\n"""', metadata

    # Multi-field mode: parse sections
    data = {}
    current_key = None
    current_lines = []

    for line in content_lines:
        if line.startswith("## "):
            # Save previous section
            if current_key:
                data[current_key] = _finalize_section(current_lines)

            # Start new section
            title = line[3:].strip()
            current_key = _title_to_key(title)
            current_lines = []
        elif current_key is not None:
            current_lines.append(line)

    # Save last section
    if current_key:
        data[current_key] = _finalize_section(current_lines)

    # Convert to TOML
    toml_lines = []
    for key, value in data.items():
        if value is None or value == "":
            continue
        elif isinstance(value, list):
            toml_lines.append(f"{key} = {value!r}")
        elif "\n" in str(value):
            toml_lines.append(f'{key} = """\n{value}\n"""')
        else:
            toml_lines.append(f'{key} = "{value}"')

    return "\n".join(toml_lines), metadata
```

#### 3. Update UserBlockManager to pass schema

**File**: `src/youlab_server/storage/blocks.py`

Add schema parameter to conversion methods:

```python
def get_block_markdown(self, label: str, schema: dict | None = None) -> str | None:
    """Get block content as Markdown for editing."""
    toml_content = self.storage.read_block(label)
    if toml_content is None:
        return None
    return toml_to_markdown(toml_content, label, schema)

def update_block_from_markdown(
    self,
    label: str,
    markdown: str,
    message: str | None = None,
    sync_to_letta: bool = True,
    schema: dict | None = None,
) -> str:
    """Update block from Markdown content (user edit)."""
    toml_content, _ = markdown_to_toml(markdown, schema)
    # ... rest unchanged
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Freeform round-trip test:
  ```python
  # In test
  toml_in = 'body = """\nHello world\n"""'
  md = toml_to_markdown(toml_in, "test")
  toml_out, _ = markdown_to_toml(md)
  assert "Hello world" in toml_out
  assert "## Body" not in md  # No section wrapper
  ```
- [ ] Multi-field still works:
  ```python
  toml_in = 'name = "Alice"\nrole = "Student"'
  md = toml_to_markdown(toml_in, "test")
  assert "## Name" in md
  assert "## Role" in md
  ```

#### Manual Verification:
- [ ] Create freeform block via API, verify MD has no section headers
- [ ] Edit freeform block in UI, verify body preserved

---

## Phase 3: Autosave and Undo/Redo

### Overview

Add autosave with 200ms debounce (like Notes) and undo/redo navigation through git history.

### Changes Required:

#### 1. Add autosave endpoint

**File**: `src/youlab_server/server/blocks.py`

Add autosave endpoint after line 177:

```python
@router.post("/{label}/autosave")
async def autosave_block(
    user_id: str,
    label: str,
    request: BlockUpdateRequest,
    storage: Annotated[GitUserStorageManager, Depends(get_storage)],
    letta: Annotated[Letta | None, Depends(get_letta_client)],
) -> BlockUpdateResponse:
    """
    Autosave block content with minimal commit message.

    Same as update but with "Autosave" prefix for commit message.
    """
    user_storage = storage.get(user_id)
    manager = UserBlockManager(user_id, user_storage, letta)

    try:
        if request.format == "markdown":
            commit_sha = manager.update_block_from_markdown(
                label=label,
                markdown=request.content,
                message=f"Autosave {label}",
                sync_to_letta=True,
            )
        else:
            commit_sha = manager.update_block_from_toml(
                label=label,
                toml_content=request.content,
                message=f"Autosave {label}",
                author="autosave",
                sync_to_letta=True,
            )

        return BlockUpdateResponse(success=True, commit_sha=commit_sha)
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

#### 2. Add autosave to BlockDetailModal

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add autosave logic (after line 34):

```typescript
let autosaveTimeout: ReturnType<typeof setTimeout> | null = null;
let lastSavedContent = '';
let autosaveStatus: 'idle' | 'saving' | 'saved' | 'error' = 'idle';

function scheduleAutosave() {
    if (autosaveTimeout) {
        clearTimeout(autosaveTimeout);
    }

    autosaveTimeout = setTimeout(async () => {
        if (markdownContent !== lastSavedContent && markdownContent.trim()) {
            autosaveStatus = 'saving';
            try {
                await autosaveBlock($user.id, label, markdownContent, localStorage.token);
                lastSavedContent = markdownContent;
                autosaveStatus = 'saved';
                // Reset to idle after 2 seconds
                setTimeout(() => { autosaveStatus = 'idle'; }, 2000);
            } catch (e) {
                autosaveStatus = 'error';
                console.error('Autosave failed:', e);
            }
        }
    }, 200);
}

// Trigger autosave on content change
$: if (editMode && markdownContent) {
    scheduleAutosave();
}
```

Add autosave status indicator in the UI (update line 270 area):

```svelte
{#if editMode}
    <div class="flex items-center gap-2 text-xs text-gray-400">
        {#if autosaveStatus === 'saving'}
            <span>Saving...</span>
        {:else if autosaveStatus === 'saved'}
            <span class="text-green-500">Saved</span>
        {:else if autosaveStatus === 'error'}
            <span class="text-red-500">Save failed</span>
        {/if}
    </div>
{/if}
```

#### 3. Add undo/redo navigation

**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`

Add navigation state and handlers (after autosave code):

```typescript
let historyIndex = 0;  // 0 = current, 1 = one back, etc.

async function undo() {
    if (historyIndex < $blockHistory.length - 1) {
        historyIndex++;
        const version = $blockHistory[historyIndex];
        const content = await getBlockVersion($user.id, label, version.sha, localStorage.token);
        markdownContent = toml_to_markdown(content);  // Need client-side conversion or API change
    }
}

async function redo() {
    if (historyIndex > 0) {
        historyIndex--;
        if (historyIndex === 0) {
            // Back to current
            await loadBlock();
        } else {
            const version = $blockHistory[historyIndex];
            const content = await getBlockVersion($user.id, label, version.sha, localStorage.token);
            markdownContent = toml_to_markdown(content);
        }
    }
}

$: canUndo = historyIndex < $blockHistory.length - 1;
$: canRedo = historyIndex > 0;
```

Add undo/redo buttons to header (update line 237 area):

```svelte
<div class="flex gap-2">
    {#if editMode}
        <button
            class="p-1.5 rounded hover:bg-gray-100 dark:hover:bg-gray-800 disabled:opacity-30"
            disabled={!canUndo}
            on:click={undo}
            title={$i18n.t('Undo')}
        >
            <ArrowUturnLeft className="size-4" />
        </button>
        <button
            class="p-1.5 rounded hover:bg-gray-100 dark:hover:bg-gray-800 disabled:opacity-30"
            disabled={!canRedo}
            on:click={redo}
            title={$i18n.t('Redo')}
        >
            <ArrowUturnRight className="size-4" />
        </button>
    {/if}
    <!-- existing buttons -->
</div>
```

#### 4. Add autosave API function

**File**: `OpenWebUI/open-webui/src/lib/apis/memory/index.ts`

Add autosave function:

```typescript
export async function autosaveBlock(
    userId: string,
    label: string,
    content: string,
    token: string
): Promise<{ success: boolean; commit_sha: string }> {
    const response = await fetch(
        `${YOULAB_API_BASE_URL}/users/${userId}/blocks/${label}/autosave`,
        {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                Authorization: `Bearer ${token}`,
            },
            body: JSON.stringify({
                content,
                format: 'markdown',
            }),
        }
    );

    if (!response.ok) {
        throw new Error(`Autosave failed: ${response.statusText}`);
    }

    return response.json();
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Autosave endpoint works:
  ```bash
  curl -X POST $YOULAB_URL/users/test/blocks/student/autosave \
    -H "Content-Type: application/json" \
    -d '{"content": "# Test\n\nContent", "format": "markdown"}'
  # Expected: {"success": true, "commit_sha": "..."}
  ```

#### Manual Verification:
- [ ] Edit block in UI, wait 200ms, verify autosave status shows "Saved"
- [ ] Check git log shows "Autosave student" commit
- [ ] Click undo button, verify previous content loads
- [ ] Click redo button, verify returns to latest content

---

## Phase 4: Models and Folders Integration

### Overview

Create modules as visible OpenWebUI models with `base_model_id = youlab`. Create background agents as hidden models. Auto-create folders and auto-assign chats.

### Changes Required:

#### 1. Add YouLab base model creation

**File**: `src/youlab_server/server/main.py`

Add base model creation in lifespan (after curriculum init, around line 99):

```python
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    # ... existing init

    # Ensure YouLab base model exists in OpenWebUI
    from youlab_server.server.sync.openwebui_client import OpenWebUIClient
    openwebui = OpenWebUIClient(settings.openwebui_api_url, settings.openwebui_api_key)

    await openwebui.ensure_base_model(
        model_id="youlab",
        name="YouLab",
        description="YouLab tutoring platform",
        pipe=True,  # This is a pipe model
    )

    yield
    # cleanup...
```

#### 2. Add model sync to OpenWebUIClient

**File**: `src/youlab_server/server/sync/openwebui_client.py`

Add model management methods:

```python
async def ensure_base_model(
    self,
    model_id: str,
    name: str,
    description: str,
    pipe: bool = False,
) -> dict:
    """Ensure base model exists in OpenWebUI."""
    models = await self._request("GET", "/api/models")

    existing = next((m for m in models if m["id"] == model_id), None)
    if existing:
        return existing

    return await self._request(
        "POST",
        "/api/models",
        json={
            "id": model_id,
            "name": name,
            "params": {"system": ""},
            "meta": {
                "description": description,
                "hidden": False,
            },
            "is_active": True,
        }
    )

async def create_module_model(
    self,
    module_id: str,
    module_name: str,
    system_prompt: str,
    base_model_id: str = "youlab",
) -> dict:
    """Create a module as a visible model."""
    return await self._request(
        "POST",
        "/api/models",
        json={
            "id": f"module-{module_id}",
            "name": module_name,
            "base_model_id": base_model_id,
            "params": {"system": system_prompt},
            "meta": {
                "description": f"YouLab module: {module_name}",
                "hidden": False,
                "type": "module",
            },
            "is_active": True,
        }
    )

async def create_background_agent_model(
    self,
    agent_id: str,
    agent_name: str,
    system_prompt: str,
    base_model_id: str = "youlab",
) -> dict:
    """Create a background agent as a hidden model."""
    return await self._request(
        "POST",
        "/api/models",
        json={
            "id": f"agent-{agent_id}",
            "name": agent_name,
            "base_model_id": base_model_id,
            "params": {"system": system_prompt},
            "meta": {
                "description": f"Background agent: {agent_name}",
                "hidden": True,
                "type": "background_agent",
            },
            "is_active": True,
        }
    )

async def ensure_model_folder(
    self,
    model_id: str,
    folder_name: str,
    user_id: str,
) -> str:
    """Ensure a folder exists for a model, return folder_id."""
    # Check existing folders
    folders = await self._request("GET", f"/api/folders?user_id={user_id}")

    # Look for folder with model_id in meta
    existing = next(
        (f for f in folders if f.get("meta", {}).get("model_id") == model_id),
        None
    )
    if existing:
        return existing["id"]

    # Create new folder
    result = await self._request(
        "POST",
        "/api/folders",
        json={
            "name": folder_name,
            "user_id": user_id,
            "meta": {"model_id": model_id},
        }
    )
    return result["id"]

async def create_chat_in_folder(
    self,
    title: str,
    model_id: str,
    folder_id: str,
    user_id: str,
) -> dict:
    """Create a chat in a specific folder."""
    return await self._request(
        "POST",
        "/api/chats/new",
        json={
            "title": title,
            "model": model_id,
            "folder_id": folder_id,
            "chat": {"messages": []},
        }
    )
```

#### 3. Create models on course enrollment

**File**: `src/youlab_server/server/agents.py`

Update `create_agent_from_curriculum` to sync models (around line 220):

```python
async def create_agent_from_curriculum(
    self,
    user_id: str,
    course_id: str,
    openwebui_client: OpenWebUIClient | None = None,
) -> str:
    """Create agent and sync module models to OpenWebUI."""
    # ... existing agent creation logic

    if openwebui_client:
        # Create module models
        for module in course_config.modules:
            await openwebui_client.create_module_model(
                module_id=module.id,
                module_name=module.name,
                system_prompt=module.agent.system or course_config.agent.system,
            )

            # Create folder for module
            await openwebui_client.ensure_model_folder(
                model_id=f"module-{module.id}",
                folder_name=module.name,
                user_id=user_id,
            )

        # Create background agent models
        for task in course_config.tasks:
            await openwebui_client.create_background_agent_model(
                agent_id=task.name,
                agent_name=task.name.replace("-", " ").title(),
                system_prompt=task.system or "",
            )

    return agent_id
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Module model created:
  ```bash
  curl -s $OPENWEBUI_URL/api/models | jq '.[] | select(.id | startswith("module-"))'
  # Expected: visible models with base_model_id = youlab
  ```
- [ ] Background agent hidden:
  ```bash
  curl -s $OPENWEBUI_URL/api/models | jq '.[] | select(.meta.hidden == true)'
  # Expected: background agent models
  ```

#### Manual Verification:
- [ ] Module appears in model selector dropdown
- [ ] Background agent does NOT appear in model selector
- [ ] Folder auto-created for module
- [ ] New chat with module auto-assigned to folder

---

## Phase 5: Notes Storage (Future)

### Overview

Add user notes storage alongside memory blocks. Notes are freeform Markdown files stored in git.

**Note**: This phase is lower priority and can be deferred. The current focus is on memory blocks.

### Changes Required:

#### 1. Add notes CRUD to GitUserStorage

**File**: `src/youlab_server/storage/git.py`

```python
def list_notes(self) -> list[str]:
    """List all note IDs."""
    return [f.stem for f in self.notes_dir.glob("*.md")]

def read_note(self, note_id: str) -> str | None:
    """Read note content."""
    path = self.notes_dir / f"{note_id}.md"
    if not path.exists():
        return None
    return path.read_text()

def write_note(
    self,
    note_id: str,
    content: str,
    message: str | None = None,
) -> str:
    """Write note and commit."""
    path = self.notes_dir / f"{note_id}.md"
    path.write_text(content)

    self.repo.index.add([str(path.relative_to(self.user_dir))])
    commit = self.repo.index.commit(message or f"Update note {note_id}")
    return commit.hexsha

def delete_note(self, note_id: str) -> None:
    """Delete note."""
    path = self.notes_dir / f"{note_id}.md"
    if path.exists():
        path.unlink()
        self.repo.index.remove([str(path.relative_to(self.user_dir))])
        self.repo.index.commit(f"Delete note {note_id}")
```

#### 2. Add notes API endpoints

**File**: `src/youlab_server/server/notes.py` (new file)

```python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

router = APIRouter(prefix="/users/{user_id}/notes", tags=["notes"])

class NoteCreateRequest(BaseModel):
    content: str
    title: str | None = None

class NoteResponse(BaseModel):
    id: str
    content: str
    title: str | None
    created_at: str
    updated_at: str

@router.get("/")
async def list_notes(user_id: str, storage: ...) -> list[NoteResponse]:
    """List all notes for user."""
    ...

@router.post("/")
async def create_note(user_id: str, request: NoteCreateRequest, storage: ...) -> NoteResponse:
    """Create a new note."""
    ...

@router.get("/{note_id}")
async def get_note(user_id: str, note_id: str, storage: ...) -> NoteResponse:
    """Get note by ID."""
    ...

@router.put("/{note_id}")
async def update_note(user_id: str, note_id: str, request: NoteCreateRequest, storage: ...) -> NoteResponse:
    """Update note content."""
    ...

@router.delete("/{note_id}")
async def delete_note(user_id: str, note_id: str, storage: ...) -> dict:
    """Delete note."""
    ...
```

### Success Criteria:

#### Automated Verification:
- [ ] Notes CRUD operations work
- [ ] Notes stored in `.data/{user_id}/notes/`
- [ ] Git versioning for notes

#### Manual Verification:
- [ ] Create/read/update/delete notes via API
- [ ] Notes visible in git history

---

## Testing Strategy

### Unit Tests

Add to `tests/test_storage/`:

1. **test_convert_freeform.py** - Freeform conversion round-trip
2. **test_git_migration.py** - blocks/ → memory-blocks/ migration
3. **test_user_config.py** - User config.toml handling

### Integration Tests

Add to `tests/test_server/`:

1. **test_autosave.py** - Autosave endpoint behavior
2. **test_model_sync.py** - Module/agent model creation in OpenWebUI

### Manual Testing Steps

1. Create new user, verify directory structure
2. Create freeform block, verify MD conversion
3. Edit block in UI, verify autosave
4. Use undo/redo, verify navigation
5. Enroll in course, verify module models appear
6. Verify background agents are hidden

## Performance Considerations

1. **Autosave debounce**: 200ms prevents excessive commits
2. **Model sync**: Only on enrollment, not on every request
3. **Folder lookup**: Cache folder_id per model in session

## Migration Notes

### Existing Users

- `blocks/` → `memory-blocks/` migration happens on first access
- Old directory kept if non-empty for backward compatibility
- No data loss - files are moved, not copied

### OpenWebUI Models

- Models created on enrollment, not retroactively
- Existing users need re-enrollment or manual model creation

## References

- Research documents: `thoughts/shared/research/2026-01-13-ARI-82-*.md`
- Linear ticket: [ARI-82](https://linear.app/ariav/issue/ARI-82)
- Related: ARI-79 (UI+Modules), ARI-80 (Memory MVP), ARI-81 (Demo-Ready)
- OpenWebUI Notes: `OpenWebUI/open-webui/src/lib/components/notes/`
- Current blocks: `src/youlab_server/storage/blocks.py`
