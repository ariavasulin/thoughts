# YouLab Product Specification

> Vision: A personalized AI tutoring platform that feels like Steve Jobs made it. Simple, opinionated, and focused.

## Core Philosophy

- **Text-based everything**: Anything an LLM sees, configures, or edits lives as text/code. This enables version control, visibility, and flexibility.
- **Good defaults, purposeful configurability**: We make opinionated choices but allow meaningful customization.
- **Claude Code-inspired agents**: Background agents are autonomous but require human approval. They diff and PR your own memory.
- **HumanLayer thoughts-inspired**: Version-controlled workspace with notes, files, and configurations.
- **Unified memory architecture**: Memory blocks are about the user, not per-course. Users see themselves as a unified tapestry.

---

## Architecture Overview

```
User's Sandbox (per-user VPS folder)
├── config/                    # User configurations (TOML)
│   ├── schema.toml           # Memory block schema
│   └── background-agents/    # Background agent configs
├── memory/                   # Memory blocks (TOML → pretty markdown display)
│   ├── operating-manual.toml
│   ├── values.toml
│   ├── personality.toml
│   └── ...
├── notes/                    # Active memory artifacts (markdown)
├── files/                    # Archival storage (markdown, uploads)
└── .git/                     # Version control (hidden from user)

Agents:
├── Fast Agents (Letta)       # User-facing course/module tutors
└── Slow Agents (Letta)       # Background agents (memory editors)
```

---

## 1. Menu Bar Redesign

### 1.1 Remove/Hide
- [ ] Remove **Workspace** from menu bar
- [ ] Remove **Code Interpreter** visibility
- [ ] Remove **Controls** visibility
- [ ] Disable **Arena Models** feature

### 1.2 Rename
- [ ] Rename app to **YouLab** in UI (logo, title, branding)
- [ ] Rename "Models" header to **"Chat"**
- [ ] Rename private folders to **"Private"**
- [ ] Rename shared folders to **"Shared"**

### 1.3 New Menu Bar Structure

```
┌─────────────────────────────┐
│  YouLab Logo                │
├─────────────────────────────┤
│  + New Chat                 │
│  🔍 Search                  │
├─────────────────────────────┤
│  📚 Modules (collapsible)   │  ← Replaces "Models"
│    └─ [active modules list] │
├─────────────────────────────┤
│  💬 Chat (collapsible)      │  ← Chat threads
│    └─ [thread list]         │
├─────────────────────────────┤
│  📁 Channels                │  ← Keep existing
├─────────────────────────────┤
│  👤 You                     │  ← NEW: Memory/Profile
├─────────────────────────────┤
│  📂 Files                   │  ← Simplified Knowledge
├─────────────────────────────┤
│  📋 Models                  │  ← Module catalog
└─────────────────────────────┘
```

### 1.4 Modules Section (replaces Models in sidebar)

**Behavior:**
- Shows **active modules** the student should work on
- Also shows **always-available agents/courses** (agents that aren't full courses with modules)
- Each module displays with icon and title (like current Models)
- Clicking a module → navigates to last chat with that agent/module
- Two modules from the **same course** CAN point to same Letta agent (different module configs)
- Module titles come from TOML config, not agent names

**Module Visibility Management:**
- Modules become visible when user gains access (enrollment, progression)
- When user progresses through a module:
  - Chat title automatically updates to reflect new module
  - New module appears in the module list
  - Thread stays the same (just renamed)
- Completed modules may be marked or archived (TBD)

**New Chat Flow:**
1. User clicks "New Chat"
2. User selects from available modules (not raw models)
3. Previous chat with that module gets archived
4. New chat becomes the "current" chat for that module
5. Behind the scenes:
   - New Honcho session created (1:1 with OpenWebUI thread)
   - Same Letta agent continues (no new agent)
   - **Current thread messages stay active; previous thread messages → archival**
   - Letta agent keeps full history in archival memory

### 1.5 Files Section (Simplified)

**Behavior:**
- Shows only the Knowledge tab content
- Removes the header component with Models/Prompts/Tools tabs
- Upload behavior: files go to user's Private folder by default
- Cannot upload to Notes (notes are agent-generated)

**Folder Structure:**
```
Files/
├── Private/      # User's private uploads and notes
└── Shared/       # Shared course materials
```

### 1.6 Models Section (Module Catalog)

**Behavior:**
- Repurposes existing Models tab UI
- Shows ALL modules (active + previous + available)
- Acts as a catalog/discovery view
- Can see module details, descriptions, progress

---

## 2. "You" Section - Memory & Profile

> This is the most important and innovative feature. Users see and manage their unified memory profile.

### 2.1 Core Concept

Memory blocks are a **unified architecture of the student** that all courses and modules use. They are NOT per-course or per-agent. Background agents edit these blocks, and users can:
- View their complete profile
- See pending changes (diffs) from background agents
- Approve/reject changes
- Manually edit blocks
- View version history
- Roll back changes

### 2.2 Default Memory Schema

```toml
# schema.toml - Default blocks (1-5 paragraphs each)

[block.operating_manual]
label = "Operating Manual"
description = "How to best engage and work with this student"
# Essentially the engagement strategy

[block.values]
label = "Values"
description = "What matters most to this student"

[block.personality]
label = "Personality"
description = "Communication style, preferences, quirks"

[block.listening_style]
label = "Listening Style"
description = "How this student prefers to receive feedback"

[block.strengths]
label = "Strengths"
description = "Areas where this student excels"

[block.goals]
label = "Goals"
description = "What this student is working toward"
```

### 2.3 UI Structure

**"You" Section has two tabs:**
1. **Profile** (default) - Schema page with all memory blocks
2. **Agents** - Background agents manifest/config

**Tab 1: Profile / Schema Page (primary view):**
```
┌─────────────────────────────────────────────┐
│  You               [Profile] [Agents]       │
├─────────────────────────────────────────────┤
│  ┌──────────────────────────────────────┐   │
│  │ Operating Manual           🔴 2 new  │   │
│  │ ─────────────────────────────────────│   │
│  │ [Pretty markdown render of TOML]     │   │
│  │ ...                           [Edit] │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │ Values                               │   │
│  │ ─────────────────────────────────────│   │
│  │ [Content...]                  [Edit] │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  [More blocks...]                           │
└─────────────────────────────────────────────┘
```

**Tab 2: Agents Page:**
```
┌─────────────────────────────────────────────┐
│  You               [Profile] [Agents]       │
├─────────────────────────────────────────────┤
│  Background Agents                          │
│  ─────────────────────────────────────────  │
│  ┌──────────────────────────────────────┐   │
│  │ 📂 Insight Synthesizer        🔴 1   │   │
│  │    └─ Jan 12, 2025 (latest)          │   │
│  │    └─ Jan 10, 2025                   │   │
│  │    └─ Jan 8, 2025                    │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │ 📂 Goal Tracker                      │   │
│  │    └─ Jan 11, 2025 (latest)          │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Diff Badges:**
- 🔴 Red badge = pending changes to review
- Number indicates count of pending changes
- Click badge on block → **auto-navigates to most recent agent thread with that diff**

**Manual Editing:**
- Each block has an [Edit] button
- Click Edit → opens markdown text editor (repurpose existing OpenWebUI editor)
- Save → creates new version in git
- Uses existing OpenWebUI markdown editor components

**Block Detail View (when clicking into a block, not diff badge):**
```
┌─────────────────────────────────────────────┐
│  ← Back        Operating Manual      [Edit] │
├─────────────────────────────────────────────┤
│  Current Value                              │
│  ─────────────────────────────────────────  │
│  [Full markdown content - editable]         │
│                                             │
├─────────────────────────────────────────────┤
│  Pending Changes                            │
│  ─────────────────────────────────────────  │
│  From: Insight Synthesizer (2h ago)         │
│  ┌───────────────────────────────────────┐  │
│  │ - Old line                            │  │
│  │ + New line                            │  │
│  └───────────────────────────────────────┘  │
│  [Approve] [Reject] [View Thread]           │
│                                             │
├─────────────────────────────────────────────┤
│  History                                    │
│  ─────────────────────────────────────────  │
│  • Jan 12, 2025 - Updated by Insight Agent  │
│  • Jan 10, 2025 - Manual edit               │
│  • Jan 8, 2025 - Initial setup              │
│  [Restore previous version...]              │
└─────────────────────────────────────────────┘
```

### 2.4 Background Agent Threads

**Architecture:**
- Each background agent job = 1 Letta agent (persistent)
- Each job run/trigger = 1 OpenWebUI thread (archived after run)
- User can respond to threads if they want
- Threads show the agent's reasoning + proposed diffs
- **Naming:** `{Agent Name} - {Date}` (e.g., "Insight Synthesizer - Jan 12, 2025")

**Thread Organization:**
- Threads grouped by agent name (like folders in Agents tab)
- Most recent thread at top
- Old threads archived but accessible
- Clicking a diff badge → goes to the most recent thread for that agent

**Thread View:**
```
┌─────────────────────────────────────────────┐
│  Insight Synthesizer - Jan 12, 2025         │
├─────────────────────────────────────────────┤
│  🤖 Agent:                                  │
│  I analyzed your recent conversations and   │
│  noticed you prefer direct feedback. I'd    │
│  like to update your Operating Manual:      │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │ [Diff view]                           │  │
│  │ - Prefers gentle suggestions          │  │
│  │ + Responds best to direct feedback    │  │
│  └───────────────────────────────────────┘  │
│                                             │
│  [Approve] [Reject] [Edit & Approve]        │
│                                             │
├─────────────────────────────────────────────┤
│  You can respond here...                    │
└─────────────────────────────────────────────┘
```

### 2.5 Version Control Implementation

**Behind the scenes:**
- User data stored in git-managed folder on VPS
- TOML files for structured data (memory blocks, config)
- Markdown files for notes/files
- Each approved change = git commit
- Rollback = git revert (abstracted for user)

**User-facing:**
- Simple "History" view with timestamps
- "Restore" button for previous versions
- No git terminology (branches, commits, etc.)
- Clean diff visualization

**Storage Format:**
```
memory/operating-manual.toml:
────────────────────────────
content = """
This student responds best to direct feedback...
"""

last_modified = "2025-01-12T10:30:00Z"
modified_by = "insight-synthesizer"
```

**Display Format:**
- TOML is converted to pretty markdown for display
- User edits markdown, it compiles back to TOML
- This gives us structured data + nice UX

### 2.6 Manual Editing

Users can edit their memory blocks directly:
1. Click "Edit" on any block
2. Edit markdown content
3. Save → creates new version
4. Change tracked in history

---

## 3. Background Agents (Claude Code-Inspired)

### 3.1 Philosophy

Background agents are autonomous Letta agents that:
- Analyze conversation history (via Honcho dialectic)
- Propose changes to memory blocks
- Require human approval (like PRs)
- Are highly autonomous within their scope
- Eventually: Users can create their own

### 3.2 Default Background Agents

```toml
# Example: insight-synthesizer.toml

[agent]
name = "Insight Synthesizer"
description = "Learns about you from conversations"
schedule = "on_idle"
idle_threshold_minutes = 30

[permissions]
blocks = ["operating_manual", "values", "personality"]
# Can propose edits to these blocks

[behavior]
system = """
You are an insight synthesizer. Your job is to learn about the student
from their conversations and propose updates to their profile.

Be thoughtful. Only propose changes you're confident about.
Show your reasoning before proposing changes.
"""
```

### 3.3 Agent Capabilities

**Current Phase:**
- [ ] View user's conversation history (Honcho)
- [ ] Propose diffs to memory blocks
- [ ] Human approval required for all changes
- [ ] Cannot configure agents in UI (TOML only)

**Future Phase:**
- [ ] Users create custom background agents
- [ ] Edit system prompts in UI
- [ ] Configure which blocks/files agents can access
- [ ] Custom triggers and schedules

### 3.4 Diff Format

Agents propose changes using a structured format:

```markdown
## Proposed Change

**Block:** Operating Manual
**Field:** Full content
**Confidence:** High

### Reasoning
Based on the last 5 conversations, I noticed that...

### Diff
```diff
- Prefers gentle, indirect feedback
+ Responds well to direct, specific feedback with examples
```

[The diff would be parsed and displayed in the UI]
```

---

## 4. Notes & Files Architecture

### 4.1 Notes (Active Memory)

- **Purpose:** Active memory artifacts that agents reference
- **Behavior:**
  - Auto-generated or agent-created
  - Regularly reviewed and archived to Files
  - Searchable by agents
  - Markdown format

### 4.2 Files (Archival Memory)

- **Purpose:** Long-term storage, reference materials
- **Behavior:**
  - Uploads go here by default (to Private/)
  - Archived notes land here
  - Course materials in Shared/
  - Searchable by agents

### 4.3 Upload Behavior

- [ ] Uploads → user's Private folder
- [ ] Cannot upload to Notes
- [ ] No upload to Shared (admin only)

---

## 5. Additional Features

### 5.1 Dialectic Module (BACKBURNER)

> Deprioritized for initial release. Implement after core features stable.

- [ ] Expose Honcho's dialectic endpoint as a "module"
- **Purpose:** Users can ask reflective questions about themselves
- **Example queries:**
  - "What patterns do you see in my learning?"
  - "How have I been engaging with the material?"
  - "What should I focus on next?"
- **Implementation:** Special module that routes to dialectic, not a tutor agent

### 5.2 Agent-Speaks-First

- [ ] Research and implement agent-initiated messages
- **Options:**
  - Repurpose suggested prompts as agent greetings
  - Auto-send first message when chat opens
  - Show agent's "thinking" before user types
- **Goal:** More natural tutoring flow where agent welcomes student

### 5.3 iA Writer Styling

- [ ] Research iA Writer font (iA Writer Duo/Mono/Quattro)
- [ ] Research iA Writer color palette
- [ ] Implement in OpenWebUI theme
- **Goal:** Clean, focused reading/writing experience

### 5.4 Tool Sync with Letta

- [ ] Tools registered in Letta available to background agents
- [ ] Course agents can use configured tools
- [ ] Tool visibility/permissions per agent type

---

## 6. Technical Implementation Notes

### 6.1 Per-User Sandbox (VPS)

Each user gets a folder structure on the VPS:

```
/data/users/{user_id}/
├── .git/                     # Hidden version control
├── config/
│   ├── schema.toml          # Memory schema definition
│   └── agents/              # Background agent configs
├── memory/                   # Memory block TOML files
├── notes/                    # Active notes (markdown)
└── files/                    # Archival files
```

### 6.2 TOML ↔ Markdown Display

**Storage:** TOML (structured, version-controllable)
**Display:** Pretty markdown (user-friendly)
**Edit:** Markdown → compile to TOML

```python
# Pseudocode
def display_block(toml_path):
    data = toml.load(toml_path)
    return markdown_render(data['content'])

def save_block(toml_path, markdown_content):
    data = {'content': markdown_content, 'modified': now()}
    toml.dump(data, toml_path)
    git_commit(toml_path, "Update block")
```

### 6.3 Letta Agent Architecture

**Fast Agents (User-facing):**
- One per (user_id, course_id) pair
- Loads module config dynamically
- References shared memory blocks

**Slow Agents (Background):**
- One per background job type
- Runs on triggers (idle, schedule, manual)
- Each run = new OpenWebUI thread
- Proposes diffs, doesn't apply directly

### 6.4 Honcho Session Management

**Key Principle:** 1:1 correspondence between Honcho sessions and OpenWebUI threads.

**Current Chat:**
- One Honcho session per OpenWebUI thread/chat_id
- Messages persisted async (fire-and-forget)
- Only current thread messages are "active"

**New Chat (same module):**
- Archive previous chat in OpenWebUI
- Create new Honcho session
- Same Letta agent continues (1 agent per user+course)
- **Previous thread messages → moved to Letta archival memory**
- Agent's archival memory has full history across all threads
- New thread starts fresh for active context

---

## 7. Open Questions

> To be resolved before detailed implementation planning

### Resolved ✓
- ~~Module-to-agent mapping~~ → Same course modules share Letta agent
- ~~Thread/session behavior~~ → 1:1 Honcho sessions to OpenWebUI threads, previous → archival
- ~~Background agent thread naming~~ → "{Agent Name} - {Date}"
- ~~Version control backend~~ → Git (abstracted from user)
- ~~Memory schema format~~ → Large text blocks (1-5 paragraphs)
- ~~"You" section UI~~ → Two tabs (Profile + Agents), diff badges link to threads

### Still Open

1. **Module progression triggers:** How exactly do we detect when a user progresses? Explicit API call? Agent decision? Time-based?

2. **Diff approval workflow:** Should approved diffs be applied immediately? Batched? Can users "undo" after approval?

3. **Schema customization:** Can users add custom memory blocks? Or fixed schema only?

4. **Multi-course memory:** How do we handle memory blocks that multiple courses want to write to? Locking? Merging?

5. **Rollback granularity:** Roll back individual blocks? All blocks? To specific date?

6. **Agent-speaks-first timing:** On chat open? After delay? Different per module?

7. **Git implementation:** libgit2 bindings vs shell out to git? Per-user bare repos?

8. **OpenWebUI component reuse:** Which existing components map to our needs? (Need research task)

---

## 8. Implementation Phases (High-Level)

> Each phase will have its own detailed research and implementation plan

### Phase 1: UI Foundation
- Rename to YouLab
- Menu bar restructure
- Remove/hide unwanted features
- Simplify Files section

### Phase 2: Modules System
- Modules replacing Models
- Module-to-agent mapping
- Chat organization behavior
- New chat flow

### Phase 3: Memory Architecture
- Unified memory blocks
- TOML storage with git
- Markdown display/edit
- Version history UI

### Phase 4: Background Agent Diffs
- Diff proposal format
- Approval workflow UI
- Background agent threads
- Pending changes badges

### Phase 5: "You" Section
- Schema page UI
- Block detail views
- Manual editing
- History/rollback

### Phase 6: Polish & Extras
- iA Writer styling
- Dialectic module
- Agent-speaks-first
- Tool sync

---

## 9. Success Criteria

- [ ] User can see all their modules in sidebar
- [ ] User can start new chats within modules
- [ ] User can view their unified memory profile
- [ ] User can see pending changes from background agents
- [ ] User can approve/reject memory diffs
- [ ] User can manually edit memory blocks
- [ ] User can view and restore previous versions
- [ ] Background agent threads are visible and interactive
- [ ] Files section shows only Knowledge (simplified)
- [ ] App is branded as YouLab
- [ ] UI feels clean, focused, and opinionated

---

## 10. References

- Claude Code: Autonomous agents with human approval
- HumanLayer thoughts: Git-managed workspace for AI
- iA Writer: Clean, focused writing UX
- OpenWebUI: Base platform we're customizing
- Letta: Agent framework with persistent memory
- Honcho: Conversation persistence and dialectic
