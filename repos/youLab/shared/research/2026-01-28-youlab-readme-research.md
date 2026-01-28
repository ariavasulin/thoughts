---
date: 2026-01-28T10:30:00-08:00
researcher: Claude
git_commit: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
branch: ralph/ralph-wiggum-mvp
repository: YouLab
topic: "README Research for Post-Refactor Documentation"
tags: [research, codebase, readme, architecture, ralph, documentation]
status: complete
last_updated: 2026-01-28
last_updated_by: Claude
---

# Research: README for Post-Refactor YouLab Documentation

**Date**: 2026-01-28T10:30:00-08:00
**Researcher**: Claude
**Git Commit**: 9dae3bb66cf73bac3ea5425398a84cf9c3863138
**Branch**: ralph/ralph-wiggum-mvp
**Repository**: YouLab

## Research Question

Create a comprehensive understanding of YouLab to write a new README that explains the project to both a senior SWE and a non-technical person (like the user's mother). The README needs to answer "why does this exist?" for audiences who have no context on Letta, Honcho, Dolt, or any of the underlying technology.

## Summary

YouLab is an AI tutoring platform with a radical premise: **you should control what AI systems "know" about you**. Unlike black-box AI products where personalization happens invisibly, YouLab makes AI context transparent, version-controlled, and user-approved.

The project has two implementations:
1. **Legacy Stack** (`src/youlab_server/`) - Letta-based, being phased out
2. **Ralph** (`src/ralph/`) - New Agno-based implementation, actively developed

The first use case is college essay coaching, but the platform is designed to support any curriculum.

## Detailed Findings

### 1. The Core Vision: Who Controls Your AI's Context?

**Source**: `/Users/ariasulin/Git/YouLab/docs/README-MANIFESTO.md`

The manifesto articulates three principles:

**Transparency**
- Most AI systems personalize invisibly - you can't see what they "know" about you
- YouLab makes context visible: memory blocks are readable markdown files
- Every change is a git commit with a reason you can read
- Agents propose changes via "pending diffs" - you approve or reject

**Portability**
- Context stored in token-space (text), not weight-space (model internals)
- Export everything at any time (JSON, markdown, git clone)
- Upgrade models without losing your context
- Self-host the entire system

**Authority**
- Agents can *propose* changes to memory
- Only *you* can approve them
- No silent updates, no hidden personalization

### 2. Architecture Overview

**Current Architecture (Ralph)**:
```
OpenWebUI → Ralph Pipe → Ralph Server (FastAPI) → Agno Agent → OpenRouter API
                                                        ↓
                                                 FileTools + ShellTools
                                                        ↓
                                                 User Workspace
```

**Key Components**:

| Component | What It Is | Role |
|-----------|-----------|------|
| OpenWebUI | Self-hosted chat UI (like ChatGPT) | Frontend interface students use |
| Ralph Pipe | OpenWebUI plugin | Bridges chat UI to YouLab backend |
| Agno | Python AI agent framework | Powers the tutor with tools |
| Dolt | MySQL + git versioning | Stores memory blocks with full history |
| Honcho | Theory-of-mind service | Answers questions about student learning patterns |

### 3. What Each Technology Does

**OpenWebUI**
- Open-source, self-hosted chat interface (similar to ChatGPT)
- Provides user authentication, chat history, and extensibility via "pipes"
- Located at: `/Users/ariasulin/Git/YouLab/OpenWebUI/open-webui/` (nested git repo)

**Agno**
- Python library for building AI agents with tools
- Provides `FileTools` (read/write files) and `ShellTools` (run commands)
- Each tool is scoped to a workspace directory for security
- Used at: `/Users/ariasulin/Git/YouLab/src/ralph/server.py:16-19`

**Dolt**
- MySQL-compatible database with git-like versioning
- Every change creates a commit with full history
- Supports branches for isolated proposals
- Enables "time travel" queries to see historical state
- Used at: `/Users/ariasulin/Git/YouLab/src/ralph/dolt.py`

**Honcho**
- External service for "theory of mind" queries
- Ask natural language questions about a student's learning patterns
- Example: "What topics has this student struggled with?" → synthesized insight
- Used at: `/Users/ariasulin/Git/YouLab/src/ralph/honcho.py`

### 4. Memory Block System

Memory blocks are structured markdown files that store what the AI "knows" about a student.

**Structure** (`src/ralph/dolt.py:32-42`):
- `user_id`: Which student owns this block
- `label`: Block identifier (e.g., "student", "goals", "journey")
- `title`: Human-readable name
- `body`: Markdown content (the actual data)
- `schema_ref`: Optional reference to structured schema
- `updated_at`: Last modification timestamp

**Agent Editing Workflow**:
1. Agent reads a memory block via `read_memory_block` tool
2. Agent proposes a surgical edit via `propose_memory_edit` (Claude Code-inspired)
3. Proposal creates a Dolt branch: `agent/{user_id}/{block_label}`
4. User reviews diff in UI
5. User approves (merge to main) or rejects (delete branch)

**Code References**:
- DoltClient: `/Users/ariasulin/Git/YouLab/src/ralph/dolt.py:68-558`
- MemoryBlockTools: `/Users/ariasulin/Git/YouLab/src/ralph/tools/memory_blocks.py:57-274`
- API endpoints: `/Users/ariasulin/Git/YouLab/src/ralph/api/blocks.py`

### 5. Honcho Integration

**What Honcho Provides**:
- Message persistence across sessions
- "Dialectic queries" - natural language questions about conversation history

**How It Works** (`src/ralph/honcho.py`):
1. Messages are persisted fire-and-forget after each exchange (lines 147-166)
2. Agent can call `query_student` tool to ask about the student (lines 34-90 in `honcho_tools.py`)
3. Honcho synthesizes insights from conversation history
4. Insights inform tutoring strategy

**Example Tool Usage**:
```python
# Agent calls this tool:
query_student(question="What is their learning style?")
# Returns: "Student engages best with concrete examples and Socratic questioning"
```

### 6. The "Claude Code-like" Experience

The Ralph implementation provides a "Claude Code-like" experience:

**Persistent Workspace**: Each user has a workspace directory that persists across chats
- Shared mode: `RALPH_AGENT_WORKSPACE` points to a codebase
- Per-user mode: `RALPH_USER_DATA_DIR/{user_id}/workspace`

**Tool Access**: Agent has `FileTools` and `ShellTools` scoped to workspace
- Can read/write files
- Can execute shell commands
- Security: Tools cannot escape the workspace directory

**Project Instructions**: If workspace contains `CLAUDE.md`, it's injected into agent instructions

**Code Reference**: `/Users/ariasulin/Git/YouLab/src/ralph/server.py:46-64`

### 7. Use Case: College Essay Coaching

**First Course** (from `thoughts/shared/youlab-project-context.md`):
- Platform is course-agnostic; college essay coaching is the first course
- Students upload CliftonStrengths results
- AI tutor guides self-discovery and essay writing

**Module Structure** (from project-context):
1. Self-Discovery: Strengths assessment, time inventory, pattern recognition
2. Story Mining: (planned)
3. Drafting: (planned)

**Memory Blocks for Course**:
- `student`: Profile, insights, preferences
- `journey`: Current module, completed lessons
- `clifton_strengths`: Top 5 strengths, reactions

### 8. Why Education?

From the manifesto (`docs/README-MANIFESTO.md:89-99`):

> A tutor who "personalizes" your learning but won't tell you what they think they know about you? That's surveillance, not teaching.
>
> A system that "adapts" to your learning style but you can't see or correct its model? That's manipulation, not education.

The best human tutors are transparent. They say "I notice you learn better with examples" and you can correct them. AI tutoring should work the same way.

### 9. Current State

**Working Today**:
- Ralph server with Agno agent (`src/ralph/server.py`)
- Memory blocks in Dolt with version history
- Proposal/approval workflow for agent edits
- Honcho message persistence and queries
- OpenWebUI integration via pipe

**Legacy (Being Phased Out)**:
- Letta-based agents in `src/youlab_server/`
- TOML-based curriculum configs
- Background agents

## README Structure Recommendation

Based on this research, here's a recommended structure for the new README:

### For Non-Technical Readers (Mom-Friendly)

**Section 1: The Problem**
- When you use AI assistants, they "learn" about you
- But you can't see what they learned
- You can't correct mistakes
- You can't take your "profile" elsewhere

**Section 2: Our Solution**
- YouLab is an AI tutor that shows you what it thinks it knows
- You review and approve any changes to its understanding
- It's like having a tutor who keeps notes you can read

**Section 3: How It Works (Simple)**
- You chat with an AI tutor
- The tutor takes notes about you (memory blocks)
- You can see the notes anytime
- When the tutor wants to update its notes, it asks your permission first

### For Technical Readers (Senior SWE)

**Section 4: Architecture**
- Chat UI (OpenWebUI) → Backend (FastAPI) → Agent (Agno) → LLM (OpenRouter)
- Memory: Dolt (MySQL + git) for version-controlled context
- Theory of Mind: Honcho for dialectic queries

**Section 5: Key Technical Decisions**
- Token-space not weight-space: Context stored as text, portable across models
- Branch-per-proposal: Dolt branches isolate agent edits until approved
- Workspace scoping: File/shell tools can't escape user directory
- Fire-and-forget persistence: Messages saved async, don't block responses

**Section 6: Getting Started**
- Quick start commands
- Environment variables
- Docker setup for Dolt

**Section 7: Current Limitations**
- Legacy Letta stack being phased out
- Background agents not yet migrated
- Only one course curriculum exists

## Code References

| Component | File | Key Lines |
|-----------|------|-----------|
| Manifesto | `/docs/README-MANIFESTO.md` | Full file |
| Ralph Server | `/src/ralph/server.py` | 163-346 (chat endpoint) |
| Ralph Pipe | `/src/ralph/pipe.py` | 45-130 (main pipe) |
| DoltClient | `/src/ralph/dolt.py` | 68-558 (all operations) |
| Memory Tools | `/src/ralph/tools/memory_blocks.py` | 163-273 (propose_edit) |
| Honcho Client | `/src/ralph/honcho.py` | 87-132 (persist + query) |
| Config | `/src/ralph/config.py` | Full file |

## Historical Context

**From thoughts/shared/youlab-project-context.md**:
- Built by Berkeley senior (transferred from CC, former carpenter)
- Working with Joey Gonzalez (Letta/MemGPT creator) at Berkeley
- Mom is college counselor (potential pilot facilitator)
- Christmas 2024 build sprint, January 2025 student testing goal

**Evolution**:
- Started with Letta (MemGPT-style self-editing memory)
- Added Honcho for theory-of-mind layer
- Now migrating to Agno + Dolt ("Ralph") for simpler, more transparent system

## Related Research

- `/thoughts/shared/research/2026-01-08-youlab-memory-philosophy.md` - Tri-layer cognitive architecture
- `/thoughts/shared/plans/2026-01-26-ralph-wiggum-mvp.md` - Ralph MVP planning
- `/thoughts/shared/plans/2025-12-26-youlab-technical-foundation.md` - Original technical foundation

## Open Questions

1. **Curriculum Migration**: How will TOML course configs migrate to the new Ralph architecture?
2. **Background Agents**: Will the background agent pattern be reimplemented in Ralph?
3. **Multi-Course**: What's the plan for supporting courses beyond college essay coaching?
4. **Self-Hosting**: What's the minimal self-hosting setup for end users?
