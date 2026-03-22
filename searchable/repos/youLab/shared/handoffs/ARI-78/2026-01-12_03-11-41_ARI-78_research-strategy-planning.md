---
date: 2026-01-12T03:11:41Z
researcher: Claude
git_commit: 5921af254188eca4151966cebff1e202da4db885
branch: main
repository: YouLab
topic: "YouLab Product Spec Research Strategy"
tags: [product-spec, research, openwebui, memory-system, background-agents]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: ARI-78 Read Research & Strategize Next Steps

## Task(s)

| Task | Status |
|------|--------|
| Create comprehensive product spec | Completed |
| Create Linear ticket ARI-78 | Completed |
| Launch 11 research agents | Completed |
| Review research findings & plan implementation | **Next Step** |

The product spec for YouLab's major UI/UX overhaul is complete. 11 research agents (Opus) were launched to investigate technical questions. The next session should:
1. Read all research comments on ARI-78
2. Synthesize findings into actionable implementation phases
3. Create detailed implementation plans for each phase

## Critical References

1. **Product Spec**: `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Complete spec with UI wireframes, architecture decisions, and open questions
2. **Linear Ticket**: [ARI-78](https://linear.app/ariav/issue/ARI-78/youlab-product-spec-menu-redesign-memory-system-background-agents) - All research comments will be attached here

## Recent changes

- `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Created comprehensive product spec

## Learnings

1. **OpenWebUI Sidebar Structure**: Sidebar uses collapsible `Folder` components with localStorage persistence. Models section uses `PinnedModelList`. Key files:
   - `OpenWebUI/open-webui/src/lib/components/layout/Sidebar.svelte`
   - `OpenWebUI/open-webui/src/lib/components/common/Folder.svelte`

2. **Current Memory System**: Memory blocks are TOML-defined in `config/courses/*/course.toml`, dynamically generated as Pydantic models. No version control exists currently - only archival audit trail.

3. **Background Agents**: Currently use dialectic queries via Honcho, apply to memory with append/replace/llm_diff strategies. `LLM_DIFF` is stubbed out.

4. **Key Architecture Decisions**:
   - 1:1 Honcho sessions ↔ OpenWebUI threads
   - Same Letta agent persists across threads (1 per user+course)
   - Memory blocks are unified per-user, not per-agent
   - Git-backed storage (abstracted from users)

## Artifacts

- `thoughts/shared/plans/2025-01-12-youlab-product-spec.md` - Full product specification
- Linear ticket ARI-78 (ID: `719d97dd-ab58-4068-86ec-d8c9405acf9f`) - Research comments attached

## Action Items & Next Steps

1. **Read ARI-78 Comments**: All 11 research agents will comment their findings on the ticket. Read all comments to understand:
   - OpenWebUI sidebar components (reusable for Modules, You sections)
   - OpenWebUI editor components (for memory block editing)
   - Git integration options (pygit2 vs GitPython vs subprocess)
   - Background agent diff workflow design
   - Chat thread & module system mapping
   - iA Writer styling & agent-speaks-first options
   - Letta unified memory architecture
   - OpenWebUI database schema extensions
   - Real-time update patterns (WebSocket/SSE)
   - YouLab pipe integration points
   - User onboarding flow

2. **Synthesize Research**: Identify:
   - Which OpenWebUI components to reuse vs build
   - Recommended git approach
   - Database schema extensions needed
   - API endpoints to create

3. **Create Implementation Plans**: Break spec into phases with detailed plans:
   - Phase 1: UI Foundation (rename, menu restructure)
   - Phase 2: Modules System
   - Phase 3: Memory Architecture (git-backed TOML)
   - Phase 4: Background Agent Diffs
   - Phase 5: "You" Section
   - Phase 6: Polish (styling, agent-first)

## Other Notes

### Research Agent IDs (for monitoring)
```
# Initial 6
b0d1e68 - OpenWebUI Sidebar Components
b06ed59 - OpenWebUI Editor Components
b811b30 - Git Integration for Python
b1de9a2 - Background Agent Diff Workflow
b50906c - Chat Thread & Module System
b348c5a - iA Writer Styling & Agent-First

# Additional 5
b3f3a31 - Letta Unified Memory Architecture
bde32a2 - OpenWebUI Database Schema
b623f91 - OpenWebUI Real-time Updates
b0b98b6 - YouLab Pipe Integration
b2961d6 - User Onboarding Flow
```

### Key Open Questions (from spec)
1. Module progression triggers
2. Diff approval workflow (immediate vs batched)
3. Schema customization (fixed vs user-extendable)
4. Multi-course memory writes
5. Rollback granularity
6. Agent-speaks-first timing

### Useful Commands
```bash
# Check research agent status
humanlayer tasks

# View specific agent output
cat /tmp/claude/tasks/<id>.output

# Read Linear ticket comments
# Use mcp__linear__list_comments with issueId: 719d97dd-ab58-4068-86ec-d8c9405acf9f
```
