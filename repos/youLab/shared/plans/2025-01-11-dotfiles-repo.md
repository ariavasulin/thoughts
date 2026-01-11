# Dotfiles Repository Implementation Plan

## Overview

Create a `dotfiles` repository to organize and backup Claude Code configuration, system configs, and automation scripts scattered across the filesystem. Uses one-way sync (local → repo) with automatic secret detection via gitleaks.

## Current State Analysis

**Files to sync are scattered across:**
- `~/.config/` - 18 app configs (ghostty, aerospace, nvim, karabiner, etc.)
- `~/.claude/` - Global Claude config (commands, agents, CLAUDE.md)
- `~/git/brain/hack/` - 9 canonical scripts (symlinked to other projects)
- Various project CLAUDE.md files

**Key discoveries:**
- `~/.config/raycast/` is 374MB - must exclude (extensions/cache bloat)
- `~/.config/opencode/` is 65MB - likely caches, exclude
- `~/.config/gh/hosts.yml` contains OAuth tokens - must exclude
- `hack/` folder is already centralized in `~/git/brain/hack/`

## Desired End State

A GitHub repository at `~/Git/dotfiles/` that:
1. Mirrors all config files in an organized structure
2. Syncs with a single `./sync.sh` command
3. Blocks commits containing secrets via gitleaks pre-commit hook
4. Can be cloned on new machines for reference (not auto-install)

**Verification:**
- `./sync.sh` runs without errors
- `gitleaks protect --staged` passes before each commit
- All expected files appear in correct locations in repo
- No secrets in git history

## What We're NOT Doing

- Reverse sync (repo → local) - this is backup only
- Auto-installation scripts for new machines
- Encryption of sensitive files (we just exclude them)
- Syncing runtime/cache data

## Implementation Approach

Three phases: repo scaffolding, sync tooling, secret detection.

---

## Phase 1: Repository Structure

### Overview
Create the dotfiles repo with organized directory structure and manifest file.

### Changes Required:

#### 1. Initialize repository
```bash
mkdir -p ~/Git/dotfiles
cd ~/Git/dotfiles
git init
```

#### 2. Create directory structure
```bash
mkdir -p config claude/global claude/commands claude/agents hack projects
```

#### 3. Create manifest.toml
**File**: `~/Git/dotfiles/manifest.toml`

```toml
# Dotfiles Sync Manifest
# Format: [section] with source = "local path" entries
# Destination is derived from section name

[config]
# ~/.config/* → config/*
source = "~/.config"
include = [
    "aerospace",
    "broot",
    "configstore",
    "gh/config.yml",      # Only config, not hosts.yml (has tokens)
    "ghostty",
    "git",
    "humanlayer",
    "iterm2",
    "karabiner",
    "lf",
    "nvim",
    "uv",
    "zed",
]
exclude = [
    "raycast",            # 374MB, too large
    "opencode",           # 65MB caches
    "qBittorrent",        # Not useful to backup
    "instagram-secretary", # Likely has API keys
    "zotero-mcp",         # MCP server, not config
    "gh/hosts.yml",       # Contains OAuth tokens
    "**/cache",
    "**/Cache",
    "**/*.log",
]

[claude.global]
source = "~/.claude"
include = [
    "CLAUDE.md",
    "settings.json",
]

[claude.commands]
source = "~/.claude/commands"
include = ["*.md"]

[claude.agents]
source = "~/.claude/agents"
include = ["*.md", "*.json", "*.toml"]

[hack]
source = "~/git/brain/hack"
include = ["*.sh"]

[projects]
# Manual entries - project CLAUDE.md files to collect
# These get renamed to {project}.CLAUDE.md
entries = [
    { source = "~/Git/YouLab/CLAUDE.md", dest = "youlab.CLAUDE.md" },
    # Add more as needed
]
```

#### 4. Create .gitignore
**File**: `~/Git/dotfiles/.gitignore`

```gitignore
# OS files
.DS_Store
Thumbs.db

# Secrets (belt and suspenders with gitleaks)
*.pem
*.key
**/hosts.yml
**/*secret*
**/*token*
**/*credential*

# Caches
**/cache/
**/Cache/
**/*.log
**/node_modules/

# Large files
*.db
*.sqlite
```

#### 5. Create README.md
**File**: `~/Git/dotfiles/README.md`

```markdown
# Dotfiles

Personal configuration files for Claude Code, system apps, and automation scripts.

## Structure

```
config/          # ~/.config/ apps (ghostty, nvim, karabiner, etc.)
claude/
  global/        # ~/.claude/CLAUDE.md, settings.json
  commands/      # Slash commands
  agents/        # Agent configurations
hack/            # Shared automation scripts
projects/        # Project-specific CLAUDE.md files
```

## Usage

### Sync local files to repo
```bash
./sync.sh
```

### Commit changes
```bash
git add -A && git commit -m "sync: $(date +%Y-%m-%d)"
```

## Secret Detection

Pre-commit hook runs gitleaks automatically. To check manually:
```bash
gitleaks protect --staged
```
```

### Success Criteria:

#### Automated Verification:
- [ ] Directory structure exists: `ls ~/Git/dotfiles/`
- [ ] Git repo initialized: `git -C ~/Git/dotfiles status`
- [ ] manifest.toml is valid TOML: `cat ~/Git/dotfiles/manifest.toml | python3 -c "import sys,tomllib; tomllib.load(sys.stdin.buffer)"`

#### Manual Verification:
- [ ] Manifest includes all desired config sources
- [ ] Exclusions cover known secrets and large files

---

## Phase 2: Sync Script

### Overview
Create a bash script that reads manifest.toml and syncs files into the repo.

### Changes Required:

#### 1. Create sync.sh
**File**: `~/Git/dotfiles/sync.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m'

log() { echo -e "${GREEN}✓${NC} $1"; }
warn() { echo -e "${YELLOW}!${NC} $1"; }
error() { echo -e "${RED}✗${NC} $1" >&2; }

# Expand ~ in paths
expand_path() {
    echo "${1/#\~/$HOME}"
}

# Sync a directory with rsync
sync_dir() {
    local src="$1"
    local dest="$2"
    shift 2
    local excludes=("$@")

    src=$(expand_path "$src")

    if [[ ! -d "$src" ]]; then
        warn "Source not found: $src"
        return 0
    fi

    mkdir -p "$dest"

    local rsync_args=(-av --delete)
    for excl in "${excludes[@]}"; do
        rsync_args+=(--exclude="$excl")
    done

    rsync "${rsync_args[@]}" "$src/" "$dest/" > /dev/null
    log "Synced $src → $dest"
}

# Sync specific files
sync_files() {
    local src_dir="$1"
    local dest_dir="$2"
    shift 2
    local files=("$@")

    src_dir=$(expand_path "$src_dir")
    mkdir -p "$dest_dir"

    for f in "${files[@]}"; do
        if [[ -e "$src_dir/$f" ]]; then
            # Handle nested paths
            local dest_path="$dest_dir/$f"
            mkdir -p "$(dirname "$dest_path")"
            cp -R "$src_dir/$f" "$dest_path"
        fi
    done
    log "Synced files from $src_dir → $dest_dir"
}

# Sync single file with rename
sync_file() {
    local src="$1"
    local dest="$2"

    src=$(expand_path "$src")

    if [[ ! -f "$src" ]]; then
        warn "File not found: $src"
        return 0
    fi

    mkdir -p "$(dirname "$dest")"
    cp "$src" "$dest"
    log "Synced $src → $dest"
}

echo "Syncing dotfiles..."
echo

# === Config files ===
sync_files ~/.config config \
    "aerospace" \
    "broot" \
    "configstore" \
    "gh/config.yml" \
    "ghostty" \
    "git" \
    "humanlayer" \
    "iterm2" \
    "karabiner" \
    "lf" \
    "nvim" \
    "uv" \
    "zed"

# === Claude global ===
sync_files ~/.claude claude/global \
    "CLAUDE.md" \
    "settings.json"

# === Claude commands ===
sync_dir ~/.claude/commands claude/commands

# === Claude agents ===
sync_dir ~/.claude/agents claude/agents

# === Hack scripts ===
sync_dir ~/git/brain/hack hack

# === Project CLAUDE.md files ===
sync_file ~/Git/YouLab/CLAUDE.md projects/youlab.CLAUDE.md
# Add more projects here as needed

echo
echo "Sync complete!"
echo

# Show what changed
if command -v git &> /dev/null && [[ -d .git ]]; then
    changed=$(git status --porcelain | wc -l | tr -d ' ')
    if [[ "$changed" -gt 0 ]]; then
        echo "Changed files:"
        git status --short
    else
        echo "No changes detected."
    fi
fi
```

#### 2. Make executable
```bash
chmod +x ~/Git/dotfiles/sync.sh
```

### Success Criteria:

#### Automated Verification:
- [ ] Script is executable: `test -x ~/Git/dotfiles/sync.sh`
- [ ] Script runs without errors: `~/Git/dotfiles/sync.sh`
- [ ] Files appear in correct locations: `ls ~/Git/dotfiles/config/ghostty`
- [ ] Claude commands synced: `ls ~/Git/dotfiles/claude/commands/*.md | wc -l` (should be 31)

#### Manual Verification:
- [ ] Running `./sync.sh` twice produces "No changes detected"
- [ ] Output is clear and shows what was synced

---

## Phase 3: Secret Detection

### Overview
Set up gitleaks for pre-commit secret scanning.

### Changes Required:

#### 1. Create .gitleaks.toml
**File**: `~/Git/dotfiles/.gitleaks.toml`

```toml
[extend]
useDefault = true

[allowlist]
description = "Global allowlist"
paths = [
    '''\.md$''',  # Allow markdown (but still scan for secrets in them)
]

# Additional rules for common config patterns
[[rules]]
id = "github-oauth-token"
description = "GitHub OAuth Token"
regex = '''oauth_token:\s*["']?[a-zA-Z0-9_-]+["']?'''
keywords = ["oauth_token"]

[[rules]]
id = "api-key-generic"
description = "Generic API Key"
regex = '''(?i)(api[_-]?key|apikey)\s*[:=]\s*["']?[a-zA-Z0-9_-]{20,}["']?'''
keywords = ["api_key", "apikey"]
```

#### 2. Create pre-commit hook
**File**: `~/Git/dotfiles/.githooks/pre-commit`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Check if gitleaks is installed
if ! command -v gitleaks &> /dev/null; then
    echo "Error: gitleaks is not installed"
    echo "Install with: brew install gitleaks"
    exit 1
fi

echo "Running gitleaks..."
if ! gitleaks protect --staged --verbose; then
    echo
    echo "Secrets detected! Commit blocked."
    echo "Review the findings above and remove any secrets."
    exit 1
fi

echo "✓ No secrets detected"
```

#### 3. Configure git to use custom hooks dir
```bash
cd ~/Git/dotfiles
mkdir -p .githooks
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks
```

#### 4. Install gitleaks (if needed)
```bash
brew install gitleaks
```

### Success Criteria:

#### Automated Verification:
- [ ] gitleaks installed: `command -v gitleaks`
- [ ] Pre-commit hook executable: `test -x ~/Git/dotfiles/.githooks/pre-commit`
- [ ] Hook path configured: `git -C ~/Git/dotfiles config core.hooksPath` outputs `.githooks`
- [ ] gitleaks passes on repo: `cd ~/Git/dotfiles && gitleaks detect --source . -v`

#### Manual Verification:
- [ ] Test by creating a file with fake secret, commit should be blocked
- [ ] Verify existing synced files don't trigger false positives

---

## Phase 4: Initial Commit & Push

### Overview
Run initial sync, commit everything, create GitHub repo.

### Changes Required:

#### 1. Run initial sync
```bash
cd ~/Git/dotfiles
./sync.sh
```

#### 2. Verify no secrets
```bash
gitleaks detect --source . -v
```

#### 3. Initial commit
```bash
git add -A
git commit -m "Initial commit: dotfiles collection"
```

#### 4. Create GitHub repo and push
```bash
gh repo create dotfiles --private --source=. --push
```

### Success Criteria:

#### Automated Verification:
- [ ] Repo synced: `git -C ~/Git/dotfiles status` shows clean working tree
- [ ] No secrets in repo: `gitleaks detect --source ~/Git/dotfiles -v`
- [ ] Remote configured: `git -C ~/Git/dotfiles remote -v` shows GitHub

#### Manual Verification:
- [ ] GitHub repo visible at github.com/ariavasulin/dotfiles
- [ ] All expected files present in repo

---

## Testing Strategy

### Pre-commit Hook Test:
1. Create test file with fake secret: `echo "api_key=sk_test_1234567890abcdef" > test.txt`
2. `git add test.txt && git commit -m "test"` - should be blocked
3. `rm test.txt`

### Sync Idempotency Test:
1. Run `./sync.sh` twice
2. Second run should report "No changes detected"

### New Machine Simulation:
1. Clone repo to temp location
2. Verify file structure is correct
3. Verify no sensitive data present

## Maintenance Notes

**Adding new configs:**
1. Edit `sync.sh` to add new sync_files/sync_dir calls
2. Run `./sync.sh && git add -A && git commit -m "add: [config name]"`

**Adding project CLAUDE.md:**
1. Add `sync_file` line to `sync.sh`
2. Run sync and commit

**Periodic sync:**
```bash
cd ~/Git/dotfiles && ./sync.sh && git add -A && git commit -m "sync: $(date +%Y-%m-%d)" && git push
```

## References

- gitleaks: https://github.com/gitleaks/gitleaks
- rsync manual: https://linux.die.net/man/1/rsync
