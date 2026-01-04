# Shared .claude and hack Configuration via Symlinks

## Overview

Implement a symlink-based approach to share `.claude/commands/`, `.claude/agents/`, and `hack/` directories across all repositories in `~/git/`. The canonical source of truth lives in `~/git/brain/`, and each repo symlinks to it.

## Current State Analysis

- 13 repos have `.claude/` and `hack/` directories as full copies
- `clinit()` in `.zshrc` copies from `~/git/humanlayer` (deprecated)
- `~/git/brain/` already contains the canonical versions
- Changes made in one repo don't propagate to others

### Key Discoveries:
- Claude Code has a known bug (#764) with symlinking the entire `.claude/` directory
- Symlinking subdirectories inside `.claude/` works correctly
- `.claude/settings.json` contains repo-specific permissions and should remain local

## Desired End State

```
~/git/brain/                          # Source of truth (version controlled)
├── .claude/
│   ├── commands/                     # Shared slash commands
│   ├── agents/                       # Shared custom agents
│   └── WriteClaudeMD.md              # Shared reference doc
└── hack/                             # Shared utility scripts

~/git/{any-repo}/
├── .claude/
│   ├── commands      → ~/git/brain/.claude/commands (symlink)
│   ├── agents        → ~/git/brain/.claude/agents (symlink)
│   ├── WriteClaudeMD.md → ~/git/brain/.claude/WriteClaudeMD.md (symlink)
│   └── settings.json                 # Local, repo-specific
├── hack              → ~/git/brain/hack (symlink)
└── CLAUDE.md                         # Local, repo-specific
```

### Verification:
- `ls -la .claude/` shows symlinks pointing to brain
- `ls -la hack` shows symlink pointing to brain
- Changes in any repo's `.claude/commands/` appear in all repos instantly
- `git status` in brain shows changes to shared config

## What We're NOT Doing

- Not symlinking the entire `.claude/` directory (bug #764)
- Not symlinking `.claude/settings.json` (repo-specific permissions)
- Not symlinking `CLAUDE.md` (repo-specific project instructions)
- Not using git submodules or subtrees (unnecessary complexity)

## Implementation Approach

Create an updated `clinit` function that sets up symlinks instead of copying, plus a migration script for existing repos.

---

## Phase 1: Update clinit in .zshrc

### Overview
Replace the copy-based `clinit` with a symlink-based approach.

### Changes Required:

#### 1. Update ~/.zshrc
**File**: `~/.zshrc`
**Changes**: Replace `clinit()` function

```bash
# Claude Code shared config initialization
# Source: ~/git/brain/.claude and ~/git/brain/hack
clinit() {
    local brain_dir="$HOME/git/brain"
    local force=false

    # Parse arguments
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -f|--force) force=true; shift ;;
            *) echo "Unknown option: $1"; return 1 ;;
        esac
    done

    # Verify brain directory exists
    if [[ ! -d "$brain_dir" ]]; then
        echo "Error: brain directory not found at $brain_dir"
        return 1
    fi

    # Create .claude directory if it doesn't exist
    mkdir -p .claude

    # Function to create symlink (with backup if force)
    _clinit_link() {
        local src="$1"
        local dst="$2"

        if [[ -L "$dst" ]]; then
            # Already a symlink - check if pointing to right place
            local current_target=$(readlink "$dst")
            if [[ "$current_target" == "$src" ]]; then
                echo "  ✓ $dst (already linked)"
                return 0
            else
                echo "  ! $dst points to $current_target, updating..."
                rm "$dst"
            fi
        elif [[ -e "$dst" ]]; then
            if $force; then
                echo "  ! $dst exists, backing up to ${dst}.bak"
                mv "$dst" "${dst}.bak"
            else
                echo "  ✗ $dst exists (use -f to force)"
                return 1
            fi
        fi

        ln -s "$src" "$dst"
        echo "  ✓ $dst → $src"
    }

    echo "Setting up Claude config symlinks..."

    # Symlink .claude subdirectories
    _clinit_link "$brain_dir/.claude/commands" ".claude/commands"
    _clinit_link "$brain_dir/.claude/agents" ".claude/agents"
    _clinit_link "$brain_dir/.claude/WriteClaudeMD.md" ".claude/WriteClaudeMD.md"

    # Symlink hack directory
    _clinit_link "$brain_dir/hack" "hack"

    # Create default settings.json if it doesn't exist
    if [[ ! -f ".claude/settings.json" ]]; then
        echo "  + Creating default .claude/settings.json"
        cat > .claude/settings.json << 'EOF'
{
  "permissions": {
    "allow": [
      "Bash(./hack/spec_metadata.sh)",
      "Bash(hack/spec_metadata.sh)",
      "Bash(bash hack/spec_metadata.sh)",
      "Bash(./hack/claude-research.sh)",
      "Bash(hack/claude-research.sh)"
    ]
  },
  "enableAllProjectMcpServers": false,
  "env": {
    "MAX_THINKING_TOKENS": "32000"
  }
}
EOF
    fi

    echo "Done! Shared config linked from $brain_dir"

    # Cleanup helper function
    unfunction _clinit_link 2>/dev/null
}

# Check symlink status across all repos
cstatus() {
    local brain_dir="$HOME/git/brain"
    echo "Checking Claude config status across ~/git repos..."
    echo ""

    for repo in ~/git/*/; do
        local repo_name=$(basename "$repo")
        [[ "$repo_name" == "brain" ]] && continue
        [[ "$repo_name" == "hack" ]] && continue  # Skip standalone hack if exists

        local status=""

        # Check .claude/commands
        if [[ -L "${repo}.claude/commands" ]]; then
            status+="✓"
        elif [[ -d "${repo}.claude/commands" ]]; then
            status+="C"  # Copy
        else
            status+="✗"
        fi

        # Check .claude/agents
        if [[ -L "${repo}.claude/agents" ]]; then
            status+="✓"
        elif [[ -d "${repo}.claude/agents" ]]; then
            status+="C"
        else
            status+="✗"
        fi

        # Check hack
        if [[ -L "${repo}hack" ]]; then
            status+="✓"
        elif [[ -d "${repo}hack" ]]; then
            status+="C"
        else
            status+="✗"
        fi

        printf "  %-25s [%s] (commands/agents/hack)\n" "$repo_name" "$status"
    done

    echo ""
    echo "Legend: ✓=symlinked, C=copy, ✗=missing"
}
```

### Success Criteria:

#### Automated Verification:
- [ ] `source ~/.zshrc` completes without errors
- [ ] `type clinit` shows the new function definition
- [ ] `type cstatus` shows the status function definition

#### Manual Verification:
- [ ] In a test directory, `clinit` creates proper symlinks
- [ ] `clinit -f` backs up existing directories and creates symlinks
- [ ] `cstatus` shows status of all repos

---

## Phase 2: Migrate Existing Repos

### Overview
Run `clinit -f` in each existing repo to convert copies to symlinks.

### Changes Required:

#### 1. Migration Script (one-time use)
**File**: `~/git/brain/Automations/migrate-claude-config.sh`
**Changes**: Create migration script

```bash
#!/bin/bash
# Migrate all repos from copied .claude/hack to symlinks
# Run once after updating clinit in .zshrc

set -e

BRAIN_DIR="$HOME/git/brain"
GIT_DIR="$HOME/git"

echo "Migrating Claude config to symlinks..."
echo "Source: $BRAIN_DIR"
echo ""

# Repos to migrate
repos=(
    "DWS-Receipts"
    "Dealtrail"
    "Honcho-Proxy"
    "Ingestion Engine"
    "LettaStarter"
    "Personal Website"
    "YouLab"
    "YouLab Website"
    "dwsERP"
    "futureSelf"
    "humanlayer"
    "open-webui"
)

for repo in "${repos[@]}"; do
    repo_path="$GIT_DIR/$repo"

    if [[ ! -d "$repo_path" ]]; then
        echo "⚠ Skipping $repo (not found)"
        continue
    fi

    echo "→ Migrating $repo..."
    cd "$repo_path"

    # Run clinit with force flag
    clinit -f

    echo ""
done

echo "Migration complete!"
echo "Run 'cstatus' to verify all repos are properly linked."
```

### Success Criteria:

#### Automated Verification:
- [ ] Script runs without errors: `bash ~/git/brain/Automations/migrate-claude-config.sh`
- [ ] `cstatus` shows all repos with `✓✓✓`

#### Manual Verification:
- [ ] Backup files (`.bak`) created for previously copied directories
- [ ] Changes in `~/git/brain/.claude/commands/` appear in all repos
- [ ] Claude Code in any repo can access slash commands

---

## Phase 3: Update .gitignore in Repos

### Overview
Ensure symlinked directories are properly handled in git.

### Considerations:

**Option A: Commit symlinks to git**
- Symlinks are committed as symlinks (git tracks them)
- Other collaborators would need same `~/git/brain` structure
- Pros: Explicit in repo
- Cons: Breaks for collaborators without brain setup

**Option B: Gitignore symlinked directories**
- Add to `.gitignore`: `.claude/commands`, `.claude/agents`, `hack`
- Pros: Clean for collaborators
- Cons: They won't get slash commands

**Recommendation**: Since these are personal repos, Option A is fine. The symlinks are committed and will work on your machine. If you share a repo, collaborators can either:
1. Set up their own brain directory
2. Run `clinit` which will detect missing brain and fail gracefully
3. Copy the directories manually

### Changes Required:

No changes needed if you're okay with committing symlinks. If you want to gitignore:

```bash
# Add to each repo's .gitignore
.claude/commands
.claude/agents
.claude/WriteClaudeMD.md
hack
```

---

## Phase 4: Clean Up Old Backups (Optional)

### Overview
After verifying everything works, remove `.bak` backup directories.

```bash
# Run in each repo after verifying symlinks work
rm -rf .claude/commands.bak .claude/agents.bak hack.bak
```

---

## Testing Strategy

### Manual Testing Steps:
1. Source updated `.zshrc`
2. Run `clinit` in a test directory
3. Verify symlinks created correctly
4. Create a new slash command in brain, verify it appears in test repo
5. Run migration script on all repos
6. Run `cstatus` to verify all repos linked
7. Test Claude Code in one repo to ensure commands work

---

## Rollback Plan

If issues arise:
1. Symlinks can be removed: `rm .claude/commands .claude/agents hack`
2. Restore from backups: `mv .claude/commands.bak .claude/commands`
3. Revert `clinit` function in `.zshrc` to original copy-based version

---

## Future Enhancements

1. **Add `cpush`/`cpull` commands** for explicit sync if symlinks cause issues
2. **Add brain to PATH** instead of symlinking hack: `export PATH="$HOME/git/brain/hack:$PATH"`
3. **Create `cinit` for new repos** that also initializes CLAUDE.md from template
