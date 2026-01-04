# Shared .claude and hack Configuration via Symlinks

## Overview

Implement a symlink-based approach to share Claude config across:
1. `~/git/brain/.claude/` - Source of truth (version controlled)
2. `~/.claude/` - User-level config (symlinks to brain)
3. `~/git/{repos}/.claude/` - Project-level config (symlinks to brain)

Plus `hack/` directory shared across all repos.

## Current State Analysis

- 13 repos have `.claude/` and `hack/` directories as full copies
- `clinit()` in `.zshrc` copies from `~/git/humanlayer` (deprecated)
- `~/git/brain/` already contains the canonical versions
- `~/.claude/` has Claude Code internal files mixed with config

### Key Discoveries:
- Claude Code has a known bug (#764) with symlinking the entire `~/.claude/` directory
- Symlinking individual files/subdirectories inside `.claude/` works correctly
- `settings.json` at user-level applies to all projects (no per-repo needed)

## Desired End State

```
~/git/brain/.claude/           # Source of truth (git tracked)
├── commands/                  # Shared slash commands
├── agents/                    # Shared custom agents
├── settings.json              # Shared settings (permissions, env)
└── WriteClaudeMD.md           # Shared reference doc

~/.claude/                     # User-level
├── commands      → ~/git/brain/.claude/commands
├── agents        → ~/git/brain/.claude/agents
├── settings.json → ~/git/brain/.claude/settings.json
├── WriteClaudeMD.md → ~/git/brain/.claude/WriteClaudeMD.md
├── debug/                     # Claude Code internal (untouched)
├── projects/                  # Claude Code internal (untouched)
├── todos/                     # Claude Code internal (untouched)
└── ...                        # Other Claude Code files

~/git/{repo}/.claude/          # Project-level
├── commands      → ~/git/brain/.claude/commands
├── agents        → ~/git/brain/.claude/agents
└── WriteClaudeMD.md → ~/git/brain/.claude/WriteClaudeMD.md
                               # No settings.json (inherits from ~/.claude/)

~/git/{repo}/
└── hack          → ~/git/brain/hack
```

### Verification:
- `ls -la ~/.claude/` shows symlinks for commands, agents, settings.json
- `ls -la .claude/` in any repo shows symlinks to brain
- `ls -la hack` shows symlink to brain
- Changes in brain appear everywhere instantly
- `git status` in brain tracks all shared config changes

## What We're NOT Doing

- Not symlinking the entire `~/.claude/` directory (bug #764)
- Not creating per-repo `settings.json` (user-level is sufficient)
- Not touching Claude Code internal files (debug, todos, projects, etc.)
- Not adding hack to PATH (keeping `./hack/` prefix for prompts)

## Implementation Approach

Two-part setup:
1. `csetup` - One-time setup for user-level `~/.claude/` symlinks
2. `clinit` - Per-repo setup for project-level symlinks

---

## Phase 1: Create Shell Functions

### Overview
Add `csetup` and updated `clinit` functions to `.zshrc`.

### Changes Required:

#### 1. Update ~/.zshrc
**File**: `~/.zshrc`
**Changes**: Replace `clinit()` and add `csetup()`, `cstatus()`

```bash
# ============================================================================
# Claude Config Management
# Source of truth: ~/git/brain/.claude and ~/git/brain/hack
# ============================================================================

CLAUDE_BRAIN_DIR="$HOME/git/brain"

# Helper function to create symlinks
_claude_link() {
    local src="$1"
    local dst="$2"
    local force="$3"

    if [[ -L "$dst" ]]; then
        local current=$(readlink "$dst")
        if [[ "$current" == "$src" ]]; then
            echo "  ✓ $dst"
            return 0
        else
            echo "  ↻ $dst (updating from $current)"
            rm "$dst"
        fi
    elif [[ -e "$dst" ]]; then
        if [[ "$force" == "true" ]]; then
            echo "  ⟳ $dst → ${dst}.bak"
            mv "$dst" "${dst}.bak"
        else
            echo "  ✗ $dst exists (use -f to force)"
            return 1
        fi
    else
        echo "  + $dst"
    fi

    ln -s "$src" "$dst"
}

# One-time setup: symlink user-level ~/.claude/ config to brain
csetup() {
    local force=false
    [[ "$1" == "-f" || "$1" == "--force" ]] && force=true

    if [[ ! -d "$CLAUDE_BRAIN_DIR" ]]; then
        echo "Error: brain not found at $CLAUDE_BRAIN_DIR"
        return 1
    fi

    echo "Setting up ~/.claude/ symlinks to brain..."

    _claude_link "$CLAUDE_BRAIN_DIR/.claude/commands" "$HOME/.claude/commands" "$force"
    _claude_link "$CLAUDE_BRAIN_DIR/.claude/agents" "$HOME/.claude/agents" "$force"
    _claude_link "$CLAUDE_BRAIN_DIR/.claude/settings.json" "$HOME/.claude/settings.json" "$force"
    _claude_link "$CLAUDE_BRAIN_DIR/.claude/WriteClaudeMD.md" "$HOME/.claude/WriteClaudeMD.md" "$force"

    echo "Done! User-level config linked."
}

# Per-repo setup: symlink project .claude/ and hack/ to brain
clinit() {
    local force=false
    [[ "$1" == "-f" || "$1" == "--force" ]] && force=true

    if [[ ! -d "$CLAUDE_BRAIN_DIR" ]]; then
        echo "Error: brain not found at $CLAUDE_BRAIN_DIR"
        return 1
    fi

    # Don't run in brain itself
    if [[ "$(pwd)" == "$CLAUDE_BRAIN_DIR" ]]; then
        echo "Error: Already in brain directory"
        return 1
    fi

    echo "Setting up Claude config symlinks..."

    # Create .claude directory if needed
    mkdir -p .claude

    # Symlink .claude contents
    _claude_link "$CLAUDE_BRAIN_DIR/.claude/commands" ".claude/commands" "$force"
    _claude_link "$CLAUDE_BRAIN_DIR/.claude/agents" ".claude/agents" "$force"
    _claude_link "$CLAUDE_BRAIN_DIR/.claude/WriteClaudeMD.md" ".claude/WriteClaudeMD.md" "$force"

    # Symlink hack directory
    _claude_link "$CLAUDE_BRAIN_DIR/hack" "hack" "$force"

    echo "Done!"
}

# Status check across all repos
cstatus() {
    echo "Claude config status"
    echo "Source: $CLAUDE_BRAIN_DIR"
    echo ""

    # Check user-level
    echo "~/.claude/:"
    for item in commands agents settings.json WriteClaudeMD.md; do
        if [[ -L "$HOME/.claude/$item" ]]; then
            printf "  ✓ %s\n" "$item"
        elif [[ -e "$HOME/.claude/$item" ]]; then
            printf "  C %s (copy)\n" "$item"
        else
            printf "  ✗ %s (missing)\n" "$item"
        fi
    done

    echo ""
    echo "Repos:"

    for repo in ~/git/*/; do
        local name=$(basename "$repo")
        [[ "$name" == "brain" || "$name" == "hack" || "$name" == "thoughts" ]] && continue

        local cmds="" agents="" hack=""

        [[ -L "${repo}.claude/commands" ]] && cmds="✓" || { [[ -d "${repo}.claude/commands" ]] && cmds="C" || cmds="✗"; }
        [[ -L "${repo}.claude/agents" ]] && agents="✓" || { [[ -d "${repo}.claude/agents" ]] && agents="C" || agents="✗"; }
        [[ -L "${repo}hack" ]] && hack="✓" || { [[ -d "${repo}hack" ]] && hack="C" || hack="✗"; }

        printf "  %-25s cmd:%s  agt:%s  hack:%s\n" "$name" "$cmds" "$agents" "$hack"
    done

    echo ""
    echo "Legend: ✓=symlink  C=copy  ✗=missing"
}

# Migrate all repos at once
cmigrate() {
    echo "Migrating all repos to symlinks..."
    echo ""

    for repo in ~/git/*/; do
        local name=$(basename "$repo")
        [[ "$name" == "brain" || "$name" == "hack" || "$name" == "thoughts" ]] && continue

        echo "→ $name"
        (cd "$repo" && clinit -f)
        echo ""
    done

    echo "Migration complete! Run 'cstatus' to verify."
}
```

### Success Criteria:

#### Automated Verification:
- [ ] `source ~/.zshrc` completes without errors
- [ ] `type csetup` shows function definition
- [ ] `type clinit` shows function definition
- [ ] `type cstatus` shows function definition
- [ ] `type cmigrate` shows function definition

#### Manual Verification:
- [ ] `cstatus` runs and shows current state
- [ ] Functions display helpful output

**Pause here for confirmation before proceeding to Phase 2.**

---

## Phase 2: Set Up User-Level Symlinks

### Overview
Run `csetup` to symlink `~/.claude/` config to brain.

### Steps:
```bash
# Back up existing and create symlinks
csetup -f
```

### Success Criteria:

#### Automated Verification:
- [ ] `ls -la ~/.claude/commands` shows symlink to brain
- [ ] `ls -la ~/.claude/agents` shows symlink to brain
- [ ] `ls -la ~/.claude/settings.json` shows symlink to brain

#### Manual Verification:
- [ ] Claude Code still works (test a slash command)
- [ ] Backups created at `~/.claude/*.bak` if needed

**Pause here for confirmation before proceeding to Phase 3.**

---

## Phase 3: Migrate All Repos

### Overview
Run `cmigrate` to convert all repos from copies to symlinks.

### Steps:
```bash
# Migrate all repos
cmigrate

# Verify
cstatus
```

### Success Criteria:

#### Automated Verification:
- [ ] `cstatus` shows all repos with ✓ for commands, agents, hack

#### Manual Verification:
- [ ] Open Claude Code in any repo, verify slash commands work
- [ ] Create a test command in brain, verify it appears in other repos
- [ ] Backups created at `.claude/*.bak` and `hack.bak` if needed

---

## Phase 4: Clean Up (Optional)

### Overview
Remove backup directories after verifying everything works.

### Steps:
```bash
# In ~/.claude/
rm -rf ~/.claude/commands.bak ~/.claude/agents.bak

# In each repo (or run this loop)
for repo in ~/git/*/; do
    rm -rf "${repo}.claude/commands.bak" "${repo}.claude/agents.bak" "${repo}hack.bak" 2>/dev/null
done
```

---

## Testing Strategy

### Manual Testing Steps:
1. After Phase 1: Run `cstatus` to see current state
2. After Phase 2: Test Claude Code with a slash command
3. After Phase 3:
   - Add new command to `~/git/brain/.claude/commands/test.md`
   - Verify it appears in any repo with `/test`
   - Delete test command
4. Verify `git status` in brain shows changes when you edit commands

---

## Rollback Plan

If issues arise:
```bash
# Remove symlinks
rm ~/.claude/commands ~/.claude/agents ~/.claude/settings.json
rm .claude/commands .claude/agents hack

# Restore backups
mv ~/.claude/commands.bak ~/.claude/commands
mv .claude/commands.bak .claude/commands
# etc.
```

---

## Git Considerations

**For brain repo**: All changes to shared config are tracked normally.

**For other repos**: The symlinks themselves are committed to git. This is fine for personal repos. If you ever share a repo:
- Collaborators without brain setup will see broken symlinks
- They can run `clinit` after setting up their own brain, or
- Copy the directories manually from somewhere

To gitignore symlinks instead (not recommended):
```bash
echo ".claude/commands" >> .gitignore
echo ".claude/agents" >> .gitignore
echo "hack" >> .gitignore
```
