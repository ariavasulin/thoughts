# Beads Integration Plan

**Created**: 2026-01-28
**Author**: Claude
**Status**: Draft
**Ticket**: N/A (Infrastructure improvement)

## Overview

Integrate Beads (bd) - a Git-backed issue tracker for AI coding agents - into the YouLab development workflow. Beads solves "agent amnesia" by persisting tasks as JSONL files that survive session compaction and restarts.

## Goals

1. Install and configure Beads for the YouLab repository
2. Set up Claude Code hooks for automatic context injection and sync
3. Update documentation (CLAUDE.md) with task management instructions
4. Update worktree scripts to propagate beads configuration
5. Document relationship with existing systems (Linear, thoughts/, TodoWrite)

## Non-Goals

- Replacing Linear for project-level planning (beads is for implementation tasks)
- Migrating existing inline TODOs to beads issues
- Multi-worktree synchronization (each worktree maintains independent state)

## Open Questions from Research

1. **Multi-worktree coordination**: Should each worktree have its own `.beads/` or share via symlink?
   - **Decision**: Each worktree gets its own `.beads/` - tasks are implementation-specific

2. **Linear ticket linking**: Can/should beads issues link to Linear tickets?
   - **Decision**: Add Linear ticket ID to beads issue description when relevant, but no formal integration

---

## Phase 1: Installation & Initialization

### Changes

1. **Install beads CLI** (user action - not scriptable)
   ```bash
   brew tap steveyegge/beads && brew install bd
   ```

2. **Initialize beads in YouLab**
   ```bash
   cd /Users/ariasulin/Git/YouLab
   bd init
   ```

3. **Set up Claude Code hooks**
   ```bash
   bd setup claude
   bd setup claude --check  # Verify
   ```

4. **Add `.beads/` to `.gitignore` exceptions** (so issues.jsonl is tracked)
   - The `bd init` command handles this automatically

### Success Criteria

- [ ] `bd --version` returns version info
- [ ] `.beads/` directory exists with `issues.jsonl`
- [ ] `bd setup claude --check` passes
- [ ] `bd ready` runs without error

### Manual Verification

- [ ] Confirm `.beads/issues.jsonl` is git-tracked (not ignored)
- [ ] Test creating an issue: `bd create "Test issue" --description "Testing beads"`
- [ ] Test closing: `bd close <id> --reason "Test complete"`
- [ ] Test sync: `bd sync`

---

## Phase 2: Documentation Updates

### Changes

1. **Update project CLAUDE.md** - Add task management section

   Add after the "Commands" section:
   ```markdown
   ## Task Management

   Use beads (`bd` command) for persistent task tracking across sessions:
   - `bd ready` - See unblocked tasks
   - `bd start <id>` - Claim a task before working
   - `bd close <id> --reason "what you did"` - Complete tasks
   - `bd sync` - Save to git (CRITICAL before session ends)

   **Always run `bd sync` before ending any session.**

   Relationship to other systems:
   - **Linear**: Project-level planning and tracking
   - **beads**: Implementation-level task persistence
   - **Inline TODOs**: Quick code markers for single-session work
   ```

2. **Update universal CLAUDE.md** - Modify task approach section

   Replace "Task Approach (Inline TODOs)" with:
   ```markdown
   ## Task Management

   **For persistent task tracking**, use beads (`bd` command). Tasks survive session compaction.
   - Run `bd ready` to see available work
   - Run `bd start <id>` before working on a task
   - Run `bd close <id> --reason "description"` when done
   - Run `bd sync` before ending sessions (CRITICAL)

   **For quick code markers**, embed inline TODOs:
   ```python
   # TODO(feature): validate input before processing
   ```

   Use beads for multi-session work; inline TODOs for single-session notes.
   ```

### Success Criteria

- [ ] Project CLAUDE.md has Task Management section
- [ ] Universal CLAUDE.md has updated task management guidance
- [ ] No duplicate/conflicting instructions about TodoWrite

### Manual Verification

- [ ] Read through both CLAUDE.md files to verify clarity
- [ ] Verify instructions don't conflict with existing workflow

---

## Phase 3: Worktree Script Updates

### Changes

1. **Update `hack/create_worktree.sh`**

   After copying `.claude` directory (around line 94), add beads initialization:
   ```bash
   # Initialize beads in worktree
   if command -v bd &> /dev/null; then
       echo "🔮 Initializing beads..."
       cd "$WORKTREE_PATH"
       if bd init --quiet > /dev/null 2>&1; then
           echo "✅ Beads initialized!"
       else
           echo "⚠️  Could not initialize beads. Run 'bd init' manually if needed."
       fi
       cd - > /dev/null
   fi
   ```

2. **Update `hack/cleanup_worktree.sh`** (if it exists)

   Add a note about syncing beads before cleanup:
   ```bash
   echo "💡 Reminder: Run 'bd sync' in the worktree before cleanup to preserve task state."
   ```

### Success Criteria

- [ ] `hack/create_worktree.sh` includes beads initialization
- [ ] New worktrees get their own `.beads/` directory
- [ ] Cleanup script reminds about `bd sync`

### Manual Verification

- [ ] Create a test worktree and verify beads is initialized
- [ ] Verify worktree beads state is independent from main repo

---

## Phase 4: Configuration & Best Practices

### Changes

1. **Create `.beads/config.yaml`** with project-specific settings:
   ```yaml
   # YouLab beads configuration
   flush-debounce: 15s
   create:
     require-description: true
   validation:
     on-create: warn
     on-sync: none
   git:
     author: "beads-bot <beads@youlab.dev>"
     no-gpg-sign: true
   ```

2. **Add beads workflow to team documentation** (if docs/ has workflow guides)
   - Link to beads docs for detailed usage
   - Document YouLab-specific conventions

### Success Criteria

- [ ] `.beads/config.yaml` exists with project settings
- [ ] Config requires descriptions on issue creation

### Manual Verification

- [ ] Create issue without description - should warn
- [ ] Verify flush-debounce is respected

---

## Rollback Plan

If beads causes issues:

1. Remove Claude Code hooks:
   ```bash
   bd setup claude --remove  # Or manually edit Claude settings
   ```

2. Remove `.beads/` directory:
   ```bash
   rm -rf .beads/
   ```

3. Revert CLAUDE.md changes via git

4. Revert worktree script changes via git

---

## Implementation Notes

### TodoWrite Tool Coexistence

The built-in TodoWrite tool is session-scoped and useful for immediate task breakdown. Beads provides cross-session persistence. They complement each other:

- **TodoWrite**: Break down current work into steps, track within-session progress
- **Beads**: Persist work across sessions, survive compaction, share with team

No changes needed to TodoWrite usage - it remains valuable for session-local planning.

### Session Workflow

Recommended workflow for Claude sessions:

1. **Start**: `bd prime` runs automatically (via hook), shows context
2. **Check tasks**: `bd ready` to see available work
3. **Claim task**: `bd start <id>` before working
4. **Work**: Use TodoWrite for session-local breakdown if needed
5. **Complete**: `bd close <id> --reason "what you did"`
6. **End**: `bd sync` runs automatically (via PreCompact hook)

### Avoiding Common Issues

- Never use `bd edit` (interactive editor doesn't work with agents)
- Always include `--reason` when closing issues
- Run `bd doctor` if things seem broken
- Keep issues manageable (start cleaning at ~200)
