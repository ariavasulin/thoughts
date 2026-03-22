---
date: 2026-01-08T11:13:47+07:00
researcher: ariasulin
git_commit: 864e1dcf90c15a7988ba7cc667c00680ee6edbf2
branch: main
repository: YouLab
topic: "Status of hack/.claude sync plan across repos"
tags: [research, configuration, symlinks, tooling, brain-repo]
status: complete
last_updated: 2026-01-08
last_updated_by: ariasulin
---

# Research: Status of hack/.claude Sync Plan

**Date**: 2026-01-08T11:13:47+07:00
**Researcher**: ariasulin
**Git Commit**: 864e1dcf90c15a7988ba7cc667c00680ee6edbf2
**Branch**: main
**Repository**: YouLab

## Research Question
What is the state of the plan for syncing /hack and .claude across repos? Why does YouLab only have hack.bak and no hack?

## Summary

The plan exists at `thoughts/shared/plans/2025-01-04-shared-claude-config-symlinks.md` and is **partially implemented**:

| Component | Status | Notes |
|-----------|--------|-------|
| `~/.claude/` as source of truth | **Complete** | 29 commands, 6 agents |
| `~/git/brain/.claude/` symlinks | **Complete** | Points to ~/.claude/* |
| `~/git/brain/hack/` | **Complete** | 22 scripts present |
| YouLab `hack` symlink | **Not created** | Only `hack.bak/` exists |
| YouLab `.claude/` cleanup | **Partial** | Backups made, but dir still exists |

**Bottom line**: The brain repo is fully configured as the central source of truth. YouLab's directories were backed up but the `clinit` command was never run to create the symlink.

## Detailed Findings

### The Plan Architecture

From `thoughts/shared/plans/2025-01-04-shared-claude-config-symlinks.md`:

```
~/.claude/                     # Source of truth (REAL directories)
├── commands/                  # Claude Code reads from here
├── agents/
└── settings.json

~/git/brain/.claude/           # Git tracking (SYMLINKS to ~/.claude/)
├── commands → ~/.claude/commands
├── agents → ~/.claude/agents
└── settings.json → ~/.claude/settings.json

~/git/brain/hack/              # Source of truth (REAL directory)

~/git/{repos}/                 # Project repos
└── hack → ~/git/brain/hack    # Symlink only
                               # No .claude/ needed - user-level applies
```

### Current State: brain repo

**`~/git/brain/.claude/`** - Fully configured:
```
agents -> /Users/ariasulin/.claude/agents
commands -> /Users/ariasulin/.claude/commands
settings.json -> /Users/ariasulin/.claude/settings.json
```

**`~/git/brain/hack/`** - Contains all shared scripts (22 files):
- check-agent.sh, verify-agent.sh, test-agent.sh
- create_worktree.sh, cleanup_worktree.sh
- spec_metadata.sh, docs-diff.sh
- linear/ subdirectory with Linear CLI tools
- Icon generation scripts
- And more

### Current State: ~/.claude/

Source of truth with real files:
- **29 command files** in `~/.claude/commands/`
- **6 agent files** in `~/.claude/agents/`
- settings.json

### Current State: YouLab repo

**Problem**: Backup was made but symlink was never created.

Current directory structure:
```
/Users/ariasulin/Git/YouLab/
├── .claude/
│   ├── agents.bak/           # 6 backed-up agent files
│   ├── commands.bak/         # 28 backed-up command files
│   ├── WriteClaudeMD.md
│   └── settings.json
├── hack.bak/                 # 24 backed-up hack files
└── hack                      # DOES NOT EXIST
```

Git status shows deletions staged for `.claude/agents/`, `.claude/commands/`, and `hack/` but the symlink was never created.

### The clinit Command

The plan defines a `clinit` shell function to set up repos:

```bash
clinit() {
    local brain="$HOME/git/brain"
    [[ ! -d "$brain" ]] && echo "brain not found" && return 1
    [[ "$(pwd)" == "$brain" ]] && echo "already in brain" && return 1

    if [[ -L "hack" ]]; then
        echo "✓ hack"
    elif [[ -e "hack" ]]; then
        [[ "$1" == "-f" ]] && mv "hack" "hack.bak" && ln -s "$brain/hack" "hack" && echo "↻ hack" || echo "✗ hack exists (-f to force)"
    else
        ln -s "$brain/hack" "hack" && echo "+ hack"
    fi
}
```

## Code References

- `thoughts/shared/plans/2025-01-04-shared-claude-config-symlinks.md` - The complete plan
- `~/git/brain/.claude/` - Symlinks to ~/.claude/ (verified working)
- `~/git/brain/hack/` - Source of truth for shared scripts

## What Needs to Happen

To complete the plan in YouLab:

1. **Create the hack symlink**:
   ```bash
   cd ~/Git/YouLab
   ln -s ~/git/brain/hack hack
   ```

2. **Clean up .claude/** (optional - user-level config already works):
   - The `.bak` directories can stay or be deleted
   - The `settings.json` and `WriteClaudeMD.md` might be repo-specific overrides

3. **Add clinit to shell profile**:
   - Add the function to `~/.zshrc` or `~/.bashrc` for future repos

## Historical Context

The plan was created on 2025-01-04 (dated in filename) after discovering issues documented in `thoughts/shared/plans/2025-12-26-ai-dev-workflow-improvements.md`:
- 5 command files had hardcoded wrong repo paths
- 17 command files referenced non-existent Makefile targets
- The solution was to centralize configuration to avoid per-repo drift

## Open Questions

1. Was `clinit` added to shell profile? If not, it should be.
2. Should the `.bak` files be deleted now that the source of truth is established?
3. Are there other repos in `/git/` that need the symlink created?
