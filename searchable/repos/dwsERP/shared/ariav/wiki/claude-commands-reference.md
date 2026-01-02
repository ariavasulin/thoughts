# Claude Code Commands Reference

A comprehensive reference for all slash commands in the dwsERP repository.

---

## Workflow & Ticket Management

### `/linear`
Manage Linear tickets - create, update, comment, and follow workflow patterns.

- Creates tickets from thoughts documents
- Adds comments with proper formatting and GitHub links
- Searches for tickets by query, team, or project
- Updates ticket status through the team workflow
- Auto-applies labels (hld, wui, meta) based on content

**Workflow states**: Triage → Spec Needed → Research Needed → Ready for Plan → Plan in Progress → Ready for Dev → In Dev → Done

---

### `/ralph_research`
Research the highest priority Linear ticket needing investigation.

- Fetches top 10 priority items in "research needed" status
- Selects highest priority SMALL/XS issue
- Conducts codebase research and web searches as needed
- Documents findings in `thoughts/shared/research/`
- Moves ticket to "research in review" when complete

---

### `/ralph_plan`
Create implementation plan for highest priority Linear ticket ready for spec.

- Fetches top 10 priority items in "ready for spec" status
- Selects highest priority SMALL/XS issue
- Creates detailed implementation plan
- Syncs plan and attaches to Linear ticket
- Moves ticket to "plan in review" status

---

### `/ralph_impl`
Implement highest priority small Linear ticket with worktree setup.

- Fetches top 10 priority items in "ready for dev" status
- Selects highest priority SMALL/XS issue
- Sets up git worktree for isolated development
- Launches implementation session with Opus model
- Auto-commits and creates PR upon completion

**Model**: Sonnet

---

### `/oneshot`
Research ticket and launch planning session.

- Runs `/ralph_research` for the given ticket
- Launches new session with `/oneshot_plan` to create the implementation plan

---

### `/oneshot_plan`
Execute ralph plan and implementation for a ticket in sequence.

- Calls `/ralph_plan` with the given ticket number
- Calls `/ralph_impl` with the given ticket number

---

## Research & Documentation

### `/research_codebase`
Document codebase as-is with thoughts directory for historical context.

- Spawns parallel sub-agents for efficient exploration
- Documents what exists WITHOUT suggesting improvements
- Produces research document in `thoughts/shared/research/`
- Includes GitHub permalinks and file:line references
- Uses thoughts/ for historical context

**Model**: Opus

---

### `/research_codebase_nt`
Document codebase as-is without evaluation or recommendations (no thoughts directory).

- Same as `/research_codebase` but without thoughts/ directory integration
- Pure documentation of current state
- No recommendations or critiques

**Model**: Opus

---

### `/research_codebase_generic`
Research codebase comprehensively using parallel sub-agents.

- Spawns multiple specialized agents in parallel
- Synthesizes findings into research document
- Includes both codebase and thoughts/ findings
- Generates YAML frontmatter with metadata

**Model**: Opus

---

## Planning Commands

### `/create_plan`
Create detailed implementation plans through interactive research and iteration.

- Reads tickets and research documents fully
- Spawns parallel research tasks
- Presents findings and design options interactively
- Gets user buy-in at each major step
- Produces plan in `thoughts/shared/plans/` with phases, success criteria
- Separates automated vs manual verification

**Model**: Opus

---

### `/create_plan_nt`
Create implementation plans with thorough research (no thoughts directory).

- Same as `/create_plan` but without thoughts/ directory references

**Model**: Opus

---

### `/create_plan_generic`
Create detailed implementation plans with thorough research and iteration.

- General-purpose planning command
- Full interactive workflow with research phases

**Model**: Opus

---

### `/iterate_plan`
Iterate on existing implementation plans with thorough research and updates.

- Reads and understands current plan structure
- Researches if changes require new technical understanding
- Makes surgical edits rather than wholesale rewrites
- Syncs updated plan to thoughts/ directory

**Usage**: `/iterate_plan path/to/plan.md - add phase for error handling`

**Model**: Opus

---

### `/iterate_plan_nt`
Iterate on existing implementation plans (no thoughts directory sync).

- Same as `/iterate_plan` but without thoughts/ directory syncing

**Model**: Opus

---

### `/validate_plan`
Validate implementation against plan, verify success criteria, identify issues.

- Discovers what was implemented through git and codebase analysis
- Runs automated verification commands
- Generates validation report with pass/fail status
- Identifies deviations from plan
- Lists manual testing requirements

---

## Implementation Commands

### `/implement_plan`
Implement technical plans from thoughts/shared/plans with verification.

- Reads plan completely and checks for existing checkmarks
- Implements each phase fully before moving to next
- Updates checkboxes in plan as sections complete
- Runs success criteria checks after each phase
- Pauses for human verification of manual testing steps

---

### `/create_worktree`
Create worktree and launch implementation session for a plan.

- Sets up git worktree with Linear branch name
- Launches implementation session with Opus model
- Auto-commits and creates PR upon completion

---

## Git & PR Commands

### `/commit`
Create git commits with user approval and no Claude attribution.

- Reviews conversation history and git diff
- Plans logical commit groupings
- Presents plan to user for approval
- Creates commits with imperative mood messages
- **No co-author or Claude attribution**

---

### `/ci_commit`
Create git commits for session changes with clear, atomic messages.

- Reviews what changed during session
- Groups related changes together
- Executes commits without stopping for feedback
- Never commits thoughts/ directory

---

### `/describe_pr`
Generate comprehensive PR descriptions following repository templates.

- Reads PR description template from thoughts/shared/pr_description.md
- Gathers full PR diff and commit history
- Runs verification commands and marks checkboxes
- Saves description to thoughts/shared/prs/
- Updates PR via `gh pr edit`

---

### `/describe_pr_nt`
Generate comprehensive PR descriptions (no thoughts directory).

- Uses embedded template instead of thoughts/ file
- Saves description to /tmp/{repo_name}/prs/
- Otherwise same workflow as `/describe_pr`

---

### `/ci_describe_pr`
Generate comprehensive PR descriptions following repository templates (CI context).

- Same as `/describe_pr` with CI-appropriate defaults

---

## Session Management

### `/create_handoff`
Create handoff document for transferring work to another session.

- Creates document in `thoughts/shared/handoffs/ENG-XXXX/`
- Includes tasks, recent changes, learnings, artifacts
- Lists action items and next steps
- Syncs to thoughts repository

---

### `/resume_handoff`
Resume work from handoff document with context analysis and validation.

- Accepts handoff path or ticket number (e.g., ENG-XXXX)
- Reads handoff and all linked plans/research documents
- Spawns research tasks to verify current state
- Presents analysis and recommended actions
- Creates todo list from action items

**Usage**:
- `/resume_handoff path/to/handoff.md`
- `/resume_handoff ENG-XXXX` (uses most recent handoff for ticket)

---

## Collaboration Commands

### `/local_review`
Set up worktree for reviewing colleague's branch.

- Adds colleague's fork as remote
- Creates worktree at `~/wt/humanlayer/SHORT_NAME`
- Copies Claude settings and runs setup
- Initializes thoughts directory

**Usage**: `/local_review gh_username:branchName`

---

### `/founder_mode`
Create Linear ticket and PR for experimental features after implementation.

- For features developed without prior ticketing
- Creates Linear ticket from implemented work
- Cherry-picks commit to new branch
- Creates PR and generates description

---

## Debugging

### `/debug`
Debug issues by investigating logs, database state, and git history.

- Investigates without editing files
- Spawns parallel tasks for logs, database, and git
- Produces debug report with evidence and root cause
- Suggests next steps for resolution

**Key locations investigated**:
- Logs: `~/.humanlayer/logs/`
- Database: `~/.humanlayer/daemon-{BRANCH_NAME}.db`
- Git state and recent changes

---

## Quick Reference Table

| Command | Model | Primary Use |
|---------|-------|-------------|
| `/linear` | default | Ticket management |
| `/ralph_research` | default | Auto-research tickets |
| `/ralph_plan` | default | Auto-plan tickets |
| `/ralph_impl` | sonnet | Auto-implement tickets |
| `/oneshot` | default | Research + plan launch |
| `/oneshot_plan` | default | Plan + implement |
| `/research_codebase` | opus | Document codebase |
| `/research_codebase_nt` | opus | Document codebase (no thoughts) |
| `/research_codebase_generic` | opus | Generic codebase research |
| `/create_plan` | opus | Interactive planning |
| `/create_plan_nt` | opus | Planning (no thoughts) |
| `/create_plan_generic` | opus | Generic planning |
| `/iterate_plan` | opus | Update existing plans |
| `/iterate_plan_nt` | opus | Update plans (no thoughts) |
| `/validate_plan` | default | Verify implementation |
| `/implement_plan` | default | Execute plans |
| `/create_worktree` | default | Worktree + implementation |
| `/commit` | default | User-approved commits |
| `/ci_commit` | default | CI commits |
| `/describe_pr` | default | PR descriptions |
| `/describe_pr_nt` | default | PR descriptions (no thoughts) |
| `/ci_describe_pr` | default | CI PR descriptions |
| `/create_handoff` | default | Session handoff |
| `/resume_handoff` | default | Resume from handoff |
| `/local_review` | default | Review colleague's branch |
| `/founder_mode` | default | Retroactive ticketing |
| `/debug` | default | Issue investigation |

---

## Command Naming Conventions

- **`_nt` suffix**: "No Thoughts" - skips thoughts/ directory integration
- **`ci_` prefix**: CI-optimized (no user interaction, auto-proceeds)
- **`ralph_` prefix**: Automated workflow for Linear tickets (research → plan → implement)
- **`create_` prefix**: Creates new artifacts (plans, handoffs, worktrees)
- **`describe_` prefix**: Generates descriptions (PRs)
- **`iterate_` prefix**: Updates existing artifacts
- **`resume_` prefix**: Continues from previous state
