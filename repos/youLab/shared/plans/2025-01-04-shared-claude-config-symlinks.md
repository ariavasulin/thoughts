# Shared .claude and hack Configuration

## Final Architecture

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

## clinit command

```bash
# Symlink hack/ to ~/git/brain/hack (commands/agents come from ~/.claude/)
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

## Usage

```bash
# New repo setup
cd ~/git/new-project
clinit        # creates hack symlink
clinit -f     # force (backs up existing)

# Edit commands/agents
# Just edit ~/.claude/commands/ or ~/.claude/agents/ directly
# Changes tracked in git via ~/git/brain/.claude/ symlinks
```
