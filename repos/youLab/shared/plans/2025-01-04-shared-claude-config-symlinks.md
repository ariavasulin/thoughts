# Shared .claude and hack Configuration via Symlinks

You ## Overview

Symlink `.claude/` contents and `hack/` from `~/git/brain/` to all repos.

## Desired End State

```
~/git/brain/.claude/           # Source of truth (git tracked)
├── commands/
├── agents/
├── settings.json
└── WriteClaudeMD.md

~/.claude/                     # User-level (symlinks)
├── commands → brain
├── agents → brain
├── settings.json → brain
└── (Claude Code internal files untouched)

~/git/{repo}/                  # Project-level (symlinks)
├── .claude/
│   ├── commands → brain
│   └── agents → brain
└── hack → brain
```

---

## Phase 1: Update clinit in ~/.zshrc

Replace the old `clinit` function:

```bash
# Claude config symlinks - source: ~/git/brain
clinit() {
    local brain="$HOME/git/brain"
    local force=false
    [[ "$1" == "-f" ]] && force=true

    [[ ! -d "$brain" ]] && echo "brain not found" && return 1
    [[ "$(pwd)" == "$brain" ]] && echo "already in brain" && return 1

    mkdir -p .claude

    for item in commands agents; do
        local src="$brain/.claude/$item" dst=".claude/$item"
        if [[ -L "$dst" ]]; then
            echo "✓ $dst"
        elif [[ -e "$dst" ]]; then
            $force && mv "$dst" "${dst}.bak" && ln -s "$src" "$dst" && echo "↻ $dst" || echo "✗ $dst exists (-f to force)"
        else
            ln -s "$src" "$dst" && echo "+ $dst"
        fi
    done

    # hack directory
    if [[ -L "hack" ]]; then
        echo "✓ hack"
    elif [[ -e "hack" ]]; then
        $force && mv "hack" "hack.bak" && ln -s "$brain/hack" "hack" && echo "↻ hack" || echo "✗ hack exists (-f to force)"
    else
        ln -s "$brain/hack" "hack" && echo "+ hack"
    fi
}
```

---

## Phase 2: Set up ~/.claude symlinks (one-time)

Run manually:

```bash
ln -sf ~/git/brain/.claude/commands ~/.claude/commands
ln -sf ~/git/brain/.claude/agents ~/.claude/agents
ln -sf ~/git/brain/.claude/settings.json ~/.claude/settings.json
```

---

## Phase 3: Migrate existing repos (one-time)

Run manually:

```bash
for repo in ~/git/*/; do
    [[ "$(basename "$repo")" =~ ^(brain|hack|thoughts)$ ]] && continue
    echo "→ $repo"
    (cd "$repo" && clinit -f)
done
```

---

## Usage

For new repos going forward:
```bash
cd ~/git/new-project
clinit        # create symlinks
clinit -f     # force (backs up existing to .bak)
```
