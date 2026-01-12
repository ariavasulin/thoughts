# Parallel Orchestrator Workflow Implementation Plan

## Overview

A three-tier context abstraction system for orchestrating 10+ parallel Opus agents via `humanlayer launch`. The user interacts with a single orchestrator agent that manages spawning, monitoring, and aggregating work from parallel research/planning/implementation agents.

**Core Philosophy**: Ralph Wiggum style - minimal user interaction (3 turns per PR), heavy subagent delegation, Linear as the source of truth, and fresh context windows for each phase.

## Architecture: Three Layers of Context Abstraction

```
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 1: Orchestrator (You - the interactive session)              │
│ - Receives PRD/spec dump from user                                 │
│ - Proposes task breakdown → user confirms (Turn 1)                 │
│ - Manages 10+ parallel agents via humanlayer launch                │
│ - Aggregates results → user confirms PRs (Turn 2-3)                │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 2: Worker Agents (10+ parallel humanlayer launch sessions)   │
│ - Each gets one task from the breakdown                            │
│ - Runs /research_codebase with bypassed permissions (15 min)       │
│ - Creates Linear ticket with research attached                     │
│ - Then runs /create_plan with bypassed permissions                 │
│ - Finally runs /create_worktree + /implement_plan + PR             │
│ - Reports back via Linear ticket updates                           │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 3: Subagents (spawned by Layer 2 workers)                    │
│ - codebase-locator, codebase-analyzer, codebase-pattern-finder     │
│ - thoughts-locator, thoughts-analyzer                              │
│ - linear-ticket-reader, linear-searcher                            │
│ - Task() calls within /research_codebase and /create_plan          │
└─────────────────────────────────────────────────────────────────────┘
```

## User Flow

```
User: /orchestrate <PRD/spec dump>

Orchestrator:
  1. Analyzes spec, proposes N tasks with titles/descriptions
  2. "I've identified these 12 tasks. Confirm to proceed?"

User: "yes" or provides adjustments (Turn 1)

Orchestrator:
  3. Spawns 10+ parallel humanlayer launch sessions
  4. Each session: research → Linear ticket → plan → worktree → PR
  5. Monitors via Linear ticket updates
  6. "All 12 PRs ready for review. Summary: [links]"

User: Reviews PR list, requests any changes (Turn 2)

Orchestrator:
  7. Addresses feedback, updates docs
  8. "Documentation updated. All PRs ready for merge."

User: Approves (Turn 3)
```

## Phase 1: Create Core Scripts

### 1.1 Create `hack/create_worktree.sh`

**File**: `hack/create_worktree.sh`

```bash
#!/usr/bin/env bash
# Create a git worktree for isolated agent work
# Usage: ./hack/create_worktree.sh <branch-name> [base-branch]

set -euo pipefail

BRANCH_NAME="${1:?Branch name required}"
BASE_BRANCH="${2:-main}"
WORKTREE_DIR=".trees/${BRANCH_NAME}"

echo "Creating worktree for branch: ${BRANCH_NAME}"

# Create the worktree
git worktree add -b "${BRANCH_NAME}" "${WORKTREE_DIR}" "${BASE_BRANCH}"

# Copy .claude directory
if [ -d ".claude" ]; then
    cp -r .claude "${WORKTREE_DIR}/"
fi

# Initialize thoughts directory link (synced separately)
mkdir -p "${WORKTREE_DIR}/thoughts"

echo "✓ Worktree created at ${WORKTREE_DIR}"
echo "  Branch: ${BRANCH_NAME}"
echo "  Base: ${BASE_BRANCH}"
```

### 1.2 Create `hack/cleanup_worktree.sh`

**File**: `hack/cleanup_worktree.sh`

```bash
#!/usr/bin/env bash
# Clean up a git worktree after work is complete
# Usage: ./hack/cleanup_worktree.sh <branch-name>

set -euo pipefail

BRANCH_NAME="${1:?Branch name required}"
WORKTREE_DIR=".trees/${BRANCH_NAME}"

if [ ! -d "${WORKTREE_DIR}" ]; then
    echo "Worktree not found: ${WORKTREE_DIR}"
    exit 1
fi

echo "Cleaning up worktree: ${BRANCH_NAME}"

# Remove the worktree
git worktree remove "${WORKTREE_DIR}" --force

echo "✓ Worktree removed"
```

### Success Criteria - Phase 1:

#### Automated Verification:
- [ ] `./hack/create_worktree.sh test-branch` creates `.trees/test-branch/`
- [ ] `./hack/cleanup_worktree.sh test-branch` removes the worktree

#### Manual Verification:
- [ ] Worktree contains `.claude/` directory
- [ ] Git branch is properly created

---

## Phase 2: Create Worker Command

### 2.1 Create `/worker` command

**File**: `.claude/commands/worker.md`

This command is what each parallel agent runs. It's designed for minimal interaction - it just executes the full workflow.

```markdown
---
description: Execute full research-plan-implement workflow for a single task (used by orchestrator)
model: opus
---

# Worker Agent

You are a worker agent spawned by the orchestrator. Execute the full workflow with MINIMAL interaction.

## Input Format

You receive a task specification:
```
TASK_ID: <unique-id>
TITLE: <task title>
DESCRIPTION: <detailed description from PRD>
SCOPE: <what's in/out of scope>
DEPENDENCIES: <other tasks this depends on, if any>
```

## Execution Flow

### Step 1: Research (use subagents generously)

1. **Spawn 5+ parallel research subagents immediately**:
   - codebase-locator: "Find all files related to [TITLE]"
   - codebase-analyzer: "Analyze how [relevant system] currently works"
   - codebase-pattern-finder: "Find similar implementations we can model after"
   - thoughts-locator: "Find any existing research/plans about [topic]"
   - linear-searcher: "Find related tickets for [topic]"

2. **Wait for all subagents, synthesize findings**

3. **Write research to** `thoughts/shared/research/YYYY-MM-DD-TASK_ID.md`

### Step 2: Create Linear Ticket

1. **Create ticket** using Linear MCP:
   ```
   Title: [TITLE]
   Description: [From PRD + research summary]
   Labels: ["orchestrated"]
   Project: YouLab
   ```

2. **Attach research document** via comment with GitHub permalink

3. **Update ticket status** to "plan in progress"

### Step 3: Create Implementation Plan

1. **Spawn parallel planning subagents**:
   - codebase-analyzer: Deep dive on specific implementation areas
   - codebase-pattern-finder: Find patterns to follow

2. **Write plan to** `thoughts/shared/plans/YYYY-MM-DD-TASK_ID.md`

3. **Attach plan to Linear ticket** via comment

4. **Update ticket status** to "ready for dev"

### Step 4: Implement

1. **Create worktree**: `./hack/create_worktree.sh TASK_ID`

2. **Implement following the plan** - spawn subagents for complex sections

3. **Run verification**: `make verify-agent`

4. **Create commit** (no Claude attribution)

5. **Create PR** using gh CLI

6. **Comment PR link on Linear ticket**

7. **Update ticket status** to "code review"

### Step 5: Report Completion

Print summary:
```
✅ TASK_ID Complete

Linear: https://linear.app/[workspace]/issue/YOU-XXX
PR: https://github.com/[repo]/pull/XXX
Research: thoughts/shared/research/YYYY-MM-DD-TASK_ID.md
Plan: thoughts/shared/plans/YYYY-MM-DD-TASK_ID.md
```

## Critical Rules

- **USE SUBAGENTS LIBERALLY** - spawn 5+ parallel agents for every research phase
- **NEVER ask questions** - make reasonable decisions and document assumptions
- **Linear is source of truth** - update ticket at every stage
- **Fresh context** - don't carry bloat between phases
- **Fail fast** - if blocked, update Linear with blocker and exit
```

### Success Criteria - Phase 2:

#### Automated Verification:
- [ ] Command file exists at `.claude/commands/worker.md`
- [ ] YAML frontmatter is valid

#### Manual Verification:
- [ ] Worker can be invoked with task specification
- [ ] Creates Linear ticket
- [ ] Spawns subagents as specified

---

## Phase 3: Create Orchestrator Command

### 3.1 Create `/orchestrate` command

**File**: `.claude/commands/orchestrate.md`

This is the main entry point - what the user invokes.

```markdown
---
description: Orchestrate 10+ parallel agents to implement a large PRD/spec
model: opus
---

# Orchestrator

You manage the parallel execution of a large PRD/spec by spawning 10+ worker agents.

## Initial Response

When invoked with a spec:

1. **Read the spec completely** - understand full scope

2. **Spawn research subagents** to understand current codebase state:
   - codebase-locator: "Find all areas that would be affected by [spec overview]"
   - thoughts-locator: "Find any existing research/plans related to [spec topics]"

3. **Propose task breakdown**:
   ```
   I've analyzed your spec and identified {N} discrete tasks that can be implemented in parallel:

   | # | Task | Scope | Estimated Impact |
   |---|------|-------|------------------|
   | 1 | [Title] | [Scope] | [Files/areas affected] |
   | 2 | [Title] | [Scope] | [Files/areas affected] |
   ...

   Dependencies:
   - Task 3 depends on Task 1 (will run sequentially)
   - All others can run in parallel

   Shall I proceed with this breakdown? You can:
   - Approve as-is
   - Merge/split tasks
   - Adjust scope
   - Add/remove tasks
   ```

4. **Wait for user confirmation** (this is Turn 1)

## After Confirmation

### Step 1: Create Master Linear Ticket

1. **Create parent ticket** for the entire spec:
   ```
   Title: [Spec Title] - Orchestrated Implementation
   Description: [Spec summary]
   Labels: ["orchestrated", "epic"]
   ```

2. **Note the ticket ID** for child tickets

### Step 2: Launch Parallel Workers

For each task (respecting dependencies):

1. **Create task specification**:
   ```
   TASK_ID: task-{N}-{kebab-title}
   TITLE: {title}
   DESCRIPTION: {from spec}
   SCOPE: {in/out}
   DEPENDENCIES: {if any}
   PARENT_TICKET: {master ticket ID}
   ```

2. **Launch worker**:
   ```bash
   humanlayer launch --model opus \
     --dangerously-skip-permissions \
     --dangerously-skip-permissions-timeout 15m \
     --title "Worker: {title}" \
     -w .trees/task-{N} \
     "/worker {task specification}"
   ```

3. **Run up to 10 workers in parallel** - wait for batch completion before next batch

### Step 3: Monitor Progress

1. **Check Linear** for ticket status updates every 2 minutes
   - Use linear MCP to query child tickets of master

2. **Track completion**:
   ```
   Progress: 7/12 tasks complete

   ✅ Task 1: [Title] - PR #123
   ✅ Task 2: [Title] - PR #124
   🔄 Task 3: [Title] - implementing...
   ⏳ Task 4: [Title] - waiting for Task 3
   ...
   ```

3. **Handle failures**: If a worker fails, note the blocker and continue

### Step 4: Documentation Update

After all workers complete:

1. **Launch documentation agent**:
   ```bash
   humanlayer launch --model opus \
     --dangerously-skip-permissions \
     --dangerously-skip-permissions-timeout 15m \
     --title "Documentation Update" \
     "/research_codebase Update documentation for [spec changes]"
   ```

### Step 5: Present Summary

```
All {N} tasks complete!

## PRs Ready for Review

| Task | PR | Status |
|------|-----|--------|
| [Title] | #123 | Ready |
| [Title] | #124 | Ready |
...

## Linear Tickets

Parent: https://linear.app/.../YOU-XXX
Children: [links]

## Next Steps

1. Review PRs (CodeRabbit will auto-review)
2. Let me know if any changes needed
3. Approve for merge

Any adjustments needed? (Turn 2)
```

## After PR Review

1. **Address any feedback** - relaunch workers for specific tasks if needed

2. **Final confirmation**:
   ```
   All PRs updated based on feedback.

   Ready to merge? (Turn 3)
   ```

## Critical Rules

- **SPAWN SUBAGENTS LIBERALLY** in research phases
- **LINEAR IS SOURCE OF TRUTH** - all progress tracked there
- **3 TURNS MAX** - breakdown → review → approve
- **PARALLEL EXECUTION** - up to 10 workers at once
- **FRESH CONTEXT** - workers run in separate sessions
```

### Success Criteria - Phase 3:

#### Automated Verification:
- [ ] Command file exists at `.claude/commands/orchestrate.md`
- [ ] YAML frontmatter is valid

#### Manual Verification:
- [ ] Can parse a spec and propose breakdown
- [ ] Can launch humanlayer sessions
- [ ] Linear tickets created correctly

---

## Phase 4: Integration and Testing

### 4.1 Update Permission Settings

**File**: `.claude/settings.json`

Add permissions for the new scripts:

```json
{
  "permissions": {
    "allow": [
      "Bash(./hack/spec_metadata.sh)",
      "Bash(hack/spec_metadata.sh)",
      "Bash(bash hack/spec_metadata.sh)",
      "Bash(./hack/claude-research.sh)",
      "Bash(hack/claude-research.sh)",
      "Bash(./hack/create_worktree.sh *)",
      "Bash(./hack/cleanup_worktree.sh *)",
      "Bash(humanlayer launch *)"
    ]
  }
}
```

### 4.2 Create Example Spec Template

**File**: `docs/orchestrator-example-spec.md`

```markdown
# Example Spec for /orchestrate

## Feature: User Dashboard

### Overview
Implement a user dashboard with metrics, settings, and activity feed.

### Requirements

1. **Metrics Display**
   - Show total sessions
   - Show average session length
   - Chart of activity over time

2. **Settings Panel**
   - Theme toggle (light/dark)
   - Notification preferences
   - API key management

3. **Activity Feed**
   - Recent conversations
   - Filterable by date
   - Searchable

### Technical Constraints
- Must use existing auth system
- SQLite for persistence
- Svelte components
```

### Success Criteria - Phase 4:

#### Automated Verification:
- [ ] Settings file parses as valid JSON
- [ ] All scripts are executable: `chmod +x hack/*.sh`

#### Manual Verification:
- [ ] `/orchestrate` command appears in slash command list
- [ ] Can invoke with example spec
- [ ] Workers spawn correctly

---

## Key Design Decisions

### 1. Why Three Layers?

- **Layer 1 (Orchestrator)**: User-facing, handles judgment calls, manages state
- **Layer 2 (Workers)**: Autonomous execution, isolated context per task
- **Layer 3 (Subagents)**: Parallel research, keeps worker context clean

### 2. Why Linear as Source of Truth?

- Workers run in separate sessions with no shared memory
- Linear tickets provide persistent state across all agents
- User can monitor progress without interacting with orchestrator
- Attachments (research/plans) are preserved

### 3. Why 15-Minute Permission Bypass?

- Research + planning can take significant time
- Avoids constant permission prompts during autonomous work
- Workers complete their full workflow without interruption

### 4. Why CodeRabbit for Review?

- Automated first-pass review on all PRs
- Catches obvious issues before human review
- Reduces Turn 2 feedback cycles

### 5. Why Batch Workers (10 at a time)?

- Respects system resources
- Allows dependency ordering
- Easier to monitor and debug

## Rollout Strategy

1. **Phase 1**: Test with single worker on small task
2. **Phase 2**: Test orchestrator with 2-3 parallel workers
3. **Phase 3**: Scale to 10+ workers on real spec
4. **Phase 4**: Iterate on prompts based on results

## Success Metrics

- **Turn efficiency**: ≤3 turns for complete spec implementation
- **Parallel utilization**: 10+ workers running simultaneously
- **Linear integration**: 100% of work tracked in tickets
- **PR quality**: CodeRabbit approval rate > 80%
- **Documentation**: All changes reflected in docs

## References

- Ralph Wiggum technique: https://ghuntley.com/ralph/
- HumanLayer context engineering: https://github.com/humanlayer/advanced-context-engineering-for-coding-agents
- Existing commands: `.trees/file-knowledge-sync/.claude/commands.bak/`
