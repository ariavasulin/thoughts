# Parallel Orchestrator Workflow - IMPLEMENTED

## Status: Complete

**Files Created:**
- `~/.claude/commands/orchestrate.md` - Main entry point
- `~/.claude/commands/worker.md` - Autonomous worker
- `hack/create_worktree.sh` - Create isolated worktrees
- `hack/cleanup_worktree.sh` - Clean up worktrees

## Overview

A three-tier context abstraction system for orchestrating 10+ parallel Opus agents via `humanlayer launch`. Minimal user interaction (3 turns), heavy subagent delegation, Linear as source of truth.

**Core Philosophy**: Ralph Wiggum style - agent does the work, human provides judgment. Fresh context windows for each phase.

## Architecture: Three Layers of Context Abstraction

```
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 1: /orchestrate (Interactive session with user)              │
│ - Receives PRD/spec dump                                           │
│ - Proposes task breakdown → user confirms (Turn 1)                 │
│ - Monitors via Linear → presents PRs (Turn 2)                      │
│ - Handles feedback → final approval (Turn 3)                       │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 2: /worker (10+ parallel humanlayer launch sessions)         │
│ - Each runs: research → ticket → plan → implement → PR             │
│ - 15-min permission bypass via --dangerously-skip-permissions      │
│ - Reports via Linear ticket updates                                │
│ - NO user interaction - fully autonomous                           │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 3: Subagents (Task() calls within workers)                   │
│ - codebase-locator, codebase-analyzer, codebase-pattern-finder     │
│ - thoughts-locator, thoughts-analyzer                              │
│ - linear-ticket-reader, linear-searcher                            │
│ - 5+ parallel agents per research phase                            │
└─────────────────────────────────────────────────────────────────────┘
```

## User Flow (3 Turns Max)

```
User: /orchestrate <dumps PRD/spec>

[Orchestrator spawns 5+ research subagents]

Orchestrator: "I've identified 12 tasks:
| # | Task | Scope | Size |
|---|------|-------|------|
| 1 | Auth system | login, register | M |
| 2 | Dashboard | metrics, charts | L |
...
Confirm?" (Turn 1)

User: "yes, but merge 3 and 4"

[Orchestrator creates Linear epic, launches 11 workers in parallel]
[Each worker: research → ticket → plan → worktree → PR]
[Workers update Linear as they progress]

Orchestrator: "All 11 PRs ready:
| Task | PR | CodeRabbit |
|------|-----|------------|
| Auth | #123 | Approved |
...
Changes needed?" (Turn 2)

User: "Fix comments on #124"

[Orchestrator comments on PR]

Orchestrator: "Fixed. Ready to merge?" (Turn 3)

User: "yes"
```

## Usage

```bash
/orchestrate

## Feature: College Essay Platform MVP

### Requirements
1. User authentication with email/password
2. Course enrollment system
3. Lesson delivery with AI tutoring
4. Progress tracking dashboard
5. Essay submission and feedback

### Technical Constraints
- Python backend with FastAPI
- SQLite for persistence
- OpenWebUI for chat frontend
```

## Key Design Decisions

### Why 3 Turns Max?
- Turn 1: Scope/breakdown decisions (human judgment)
- Turn 2: PR review decisions (human judgment)
- Turn 3: Merge approval (human judgment)
- Everything else is agent work

### Why Linear as Source of Truth?
- Workers run in isolated sessions with no shared memory
- Linear tickets persist state across all agents
- User can monitor progress passively
- Research/plans attached as comments

### Why 15-Minute Permission Bypass?
```bash
humanlayer launch --model opus \
  --dangerously-skip-permissions \
  --dangerously-skip-permissions-timeout 15 \
  "/worker ..."
```
- Research + planning takes significant time
- Workers complete full workflow uninterrupted

### Why 5+ Subagents Per Phase?
- Parallel research maximizes efficiency
- Fresh context per subagent prevents bloat
- Each agent type is specialized

## Leverages Existing Commands

The orchestrator builds on:
- `/research_codebase` - Research patterns
- `/create_plan` - Plan structure
- `/implement_plan` - Implementation flow
- `/commit` - Commit patterns
- `/describe_pr` - PR creation
- `/create_worktree` - Worktree setup
- Linear MCP tools - Ticket management

## Success Metrics

- **Turn efficiency**: ≤3 turns for spec → PRs
- **Parallel utilization**: 10+ workers simultaneously
- **Linear integration**: 100% of work tracked
- **PR quality**: CodeRabbit auto-reviews all PRs

## References

- Ralph Wiggum technique: https://ghuntley.com/ralph/
- HumanLayer context engineering: https://github.com/humanlayer/advanced-context-engineering-for-coding-agents
- Brief History of Ralph: https://www.humanlayer.dev/blog/brief-history-of-ralph
