---
date: 2026-01-24T18:35:55Z
researcher: Claude
git_commit: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
branch: main
repository: YouLab
topic: "Beads Setup and Integration for YouLab"
tags: [research, beads, task-management, claude-code, context-persistence]
status: complete
last_updated: 2026-01-24
last_updated_by: Claude
---

# Research: Beads Setup and Integration for YouLab

**Date**: 2026-01-24T18:35:55Z
**Researcher**: Claude
**Git Commit**: 2611ba2e2938a3a1528fab03b0600ce6b8f14a33
**Branch**: main
**Repository**: YouLab

## Research Question

How to set up beads in the YouLab codebase, including what context to add to CLAUDE.md, configuration options, and default configs for inspiration.

## Summary

**Beads** is a Git-backed issue tracker by Steve Yegge designed for AI coding agents. It solves "agent amnesia" by persisting tasks as JSONL files in `.beads/` directory, surviving session compactions and restarts. The integration with Claude Code is minimal—a single `bd setup claude` command installs hooks, and CLAUDE.md needs just a brief mention since agents naturally discover and use beads.

## Detailed Findings

### What Beads Is

- **CLI tool**: `bd` command for task management
- **Storage**: `.beads/issues.jsonl` (git-tracked) + local SQLite cache
- **Purpose**: Persistent memory across Claude sessions, survives context compaction
- **Features**: Dependency tracking, blocking relationships, epics, priority levels (0-4)
- **Repository**: [github.com/steveyegge/beads](https://github.com/steveyegge/beads) (12.6k stars)

### Installation Options

```bash
# Homebrew (recommended for macOS)
brew tap steveyegge/beads
brew install bd

# npm/bun
npm install -g @beads/bd
bun install -g --trust @beads/bd

# Go
go install github.com/steveyegge/beads/cmd/bd@latest

# Quick install script
curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash
```

### Claude Code Integration

**Quick setup** (recommended):
```bash
bd setup claude
```

This installs two hooks automatically:
- **SessionStart**: Runs `bd prime` to inject ~1-2k tokens of context
- **PreCompact**: Runs `bd sync` to persist work before compaction

**Verify installation**:
```bash
bd setup claude --check
```

**Manual hook configuration** (if needed):
Add to Claude Code settings:
```json
{
  "hooks": {
    "SessionStart": ["bd prime"],
    "PreCompact": ["bd sync"]
  }
}
```

### Directory Structure After Init

```
.beads/
├── beads.db           # SQLite database (git-ignored)
├── issues.jsonl       # Issue data (git-tracked)
├── metadata.json      # Database metadata (git-tracked)
├── config.yaml        # Project configuration (git-tracked)
├── interactions.jsonl # Agent audit log (git-tracked)
└── .gitignore
```

### Initialization Modes

```bash
bd init                    # Standard setup
bd init --quiet            # Non-interactive (for agents)
bd init --stealth          # Local-only, no repo pollution
bd init --contributor      # Fork workflow for OSS
bd init --team             # Multi-agent coordination
```

### Configuration Files

**Project config** (`.beads/config.yaml`):
```yaml
flush-debounce: 15s
create:
  require-description: true
validation:
  on-create: warn
  on-sync: none
git:
  author: "beads-bot <beads@example.com>"
  no-gpg-sign: true
sync:
  mode: git-portable
  export_on: push
  import_on: pull
conflict:
  strategy: newest
```

**User config** (`~/.config/bd/config.yaml`):
```yaml
json: true
no-daemon: false
flush-debounce: 10s
auto-start-daemon: true
```

### Essential Commands

| Command | Purpose |
|---------|---------|
| `bd ready` | Show tasks with no blockers |
| `bd start <id>` | Mark task in progress |
| `bd close <id>` | Complete a task |
| `bd comment <id>` | Add notes for future sessions |
| `bd sync` | Persist changes to git |
| `bd prime` | Inject context for Claude |
| `bd doctor` | Diagnose and fix issues |
| `bd cleanup` | Clean up closed issues |

### CLAUDE.md Integration

Beads is designed so agents discover and use it naturally. Minimal prompting needed.

**Option 1: Brief mention** (recommended)
```markdown
## Task Management
Use beads (bd command) for task tracking. Run `bd quickstart` for setup instructions.
```

**Option 2: Detailed instructions**
```markdown
## Task Management with Beads

Use Beads (bd command) for persistent task tracking across sessions:
- `bd ready` - See available tasks with no blockers
- `bd start <id>` - Mark task in progress before working
- `bd close <id> --reason "description"` - Complete tasks
- `bd comment <id> "notes"` - Add context for future sessions
- `bd sync` - Persist changes to git (CRITICAL before ending session)

Always run `bd sync` before ending any session to preserve progress.
```

### Best Practices

1. **Session workflow**: `bd ready` → `bd start <id>` → work → `bd close <id>` → `bd sync`
2. **Write notes for future agents**: Explain as if to someone with zero context
3. **Run `bd doctor` daily**: Diagnoses and fixes common issues
4. **Keep issues manageable**: Start cleaning at ~200 issues, rarely exceed 500
5. **Use `--json` flag**: For programmatic parsing in automation
6. **Never use `bd edit`**: Interactive editor doesn't work with agents

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `BD_ACTOR` | Actor name for audit trails |
| `BD_NO_DAEMON` | Bypass daemon mode |
| `BD_NO_AUTO_FLUSH` | Disable auto JSONL export |
| `BEADS_FLUSH_DEBOUNCE` | Auto-flush delay |

### Known Issues

- Some users report issues with Opus 4.5 + Claude Code (incorrect syntax generation)
- Keep an eye on [github.com/steveyegge/beads/issues](https://github.com/steveyegge/beads/issues)

## YouLab-Specific Recommendations

### Changes to CLAUDE.md

Add to `CLAUDE.md` (project-level):

```markdown
## Task Management

Use beads (`bd` command) for persistent task tracking across sessions:
- `bd ready` - See unblocked tasks
- `bd start <id>` - Claim a task before working
- `bd close <id> --reason "what you did"` - Complete tasks
- `bd sync` - Save to git (CRITICAL before session ends)

**Always run `bd sync` before ending any session.**
```

### Changes to Universal CLAUDE.md

Replace or supplement the "Task Approach (Inline TODOs)" section in `~/.claude/CLAUDE.md`:

```markdown
## Task Management

**For persistent task tracking**, use beads (`bd` command). Tasks survive session compaction.
- Run `bd ready` to see available work
- Run `bd sync` before ending sessions

**For quick code markers**, embed inline TODOs:
```python
# TODO(feature): validate input before processing
```

Use beads for multi-session work; inline TODOs for single-session notes.
```

### Setup Commands

```bash
# Install beads
brew tap steveyegge/beads && brew install bd

# Initialize in YouLab
cd /Users/ariasulin/Git/YouLab
bd init

# Set up Claude Code integration
bd setup claude

# Verify
bd setup claude --check
```

### Relationship to Existing Systems

| Current System | Beads Relationship |
|----------------|-------------------|
| `thoughts/` directory | Complementary - thoughts for research/docs, beads for tasks |
| Inline TODOs | Use beads for cross-session work, TODOs for quick markers |
| TodoWrite tool | Beads replaces session-scoped tracking with persistent tracking |
| Linear tickets | Beads for implementation tasks, Linear for planning/tracking |

## Code References

- Project CLAUDE.md: `/Users/ariasulin/Git/YouLab/CLAUDE.md`
- Universal CLAUDE.md: `/Users/ariasulin/.claude/CLAUDE.md`

## External References

- [Beads GitHub Repository](https://github.com/steveyegge/beads)
- [Claude Code Integration Docs](https://steveyegge.github.io/beads/integrations/claude-code)
- [Configuration Reference](https://github.com/steveyegge/beads/blob/main/docs/CONFIG.md)
- [Installation Guide](https://github.com/steveyegge/beads/blob/main/docs/INSTALLING.md)

## Open Questions

1. **HumanLayer integration**: How does beads interact with `humanlayer launch` sessions in worktrees?
2. **Multi-worktree coordination**: Should each worktree have its own `.beads/` or share via symlink?
3. **Linear ticket linking**: Can/should beads issues link to Linear tickets for traceability?
