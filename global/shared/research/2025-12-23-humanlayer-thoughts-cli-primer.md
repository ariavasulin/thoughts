---
date: 2025-12-23T13:05:07-08:00
researcher: ariasulin
git_commit: 02f3bb203604061e80bd32858583ba6c7191d606
branch: main
repository: humanlayer
topic: "HumanLayer CLI and Thoughts Functionality Primer for WSL Setup"
tags: [research, thoughts, cli, wsl, setup, collaboration]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: HumanLayer CLI and Thoughts Functionality Primer for WSL Setup

**Date**: 2025-12-23T13:05:07-08:00
**Researcher**: ariasulin
**Git Commit**: 02f3bb203604061e80bd32858583ba6c7191d606
**Branch**: main
**Repository**: humanlayer

## Research Question
How to set up and use HumanLayer CLI with the thoughts functionality for sharing between users, specifically for someone using WSL on Windows with Claude Code.

## Summary

The HumanLayer thoughts system is a developer notes management system that keeps architecture decisions, TODOs, and development thoughts separate from code repositories while making them accessible to AI coding assistants like Claude Code. Two users can share thoughts by pointing to the same thoughts git repository (either via shared filesystem or remote git repo). The CLI tool `humanlayer` provides all necessary commands for initialization, syncing, and management.

---

## Quick Start for Your Roommate on WSL

### Step 1: Install Prerequisites on WSL

```bash
# Ensure Node.js is installed (v16+)
node --version

# If not installed, install via nvm (recommended for WSL)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install --lts

# Ensure git is configured
git config --global user.name "YourName"
git config --global user.email "your@email.com"
```

### Step 2: Install HumanLayer CLI

```bash
# Option A: Use npx (no installation required)
npx humanlayer thoughts --help

# Option B: Install globally
npm install -g humanlayer
```

### Step 3: Set Up Shared Thoughts Repository

**If sharing via git remote (recommended for different machines):**

```bash
# Clone the shared thoughts repo (you'll need to share this URL)
git clone git@github.com:your-org/thoughts.git ~/thoughts

# Or create a new one and push to remote
mkdir ~/thoughts && cd ~/thoughts
git init
git remote add origin git@github.com:your-org/thoughts.git
```

### Step 4: Initialize Thoughts in Your Code Repository

```bash
cd /path/to/your-code-repo
humanlayer thoughts init
```

The init wizard will:
1. Ask for your username (use something unique, e.g., "roommate-name")
2. Set up the thoughts directory structure
3. Create symlinks to `~/thoughts`
4. Install git hooks for auto-sync

### Step 5: Verify Setup

```bash
humanlayer thoughts status
```

---

## How Thoughts Work

### Directory Structure After Init

In your code repository:
```
your-project/
├── thoughts/
│   ├── yourname/       # Your personal thoughts (symlink to ~/thoughts/repos/project/yourname)
│   ├── shared/         # Team thoughts (symlink to ~/thoughts/repos/project/shared)
│   ├── global/         # Cross-repo thoughts (symlink to ~/thoughts/global)
│   │   ├── yourname/   # Your cross-repo notes
│   │   └── shared/     # Team cross-repo notes
│   ├── searchable/     # Hard links for AI search (auto-generated)
│   └── CLAUDE.md       # Auto-generated context for AI
```

In your central thoughts repository (`~/thoughts`):
```
~/thoughts/
├── repos/
│   └── your-project/
│       ├── yourname/
│       ├── roommate/   # Auto-discovered when they sync
│       └── shared/
└── global/
    ├── yourname/
    ├── roommate/
    └── shared/
```

### User Directories

- Each person gets their own directory (e.g., `thoughts/alice/`, `thoughts/bob/`)
- The `shared/` directory is for team collaboration
- The `global/` directory is for cross-repository notes

---

## Essential Commands

### `humanlayer thoughts init`
Initialize thoughts for the current code repository.

```bash
humanlayer thoughts init
humanlayer thoughts init --force  # Reconfigure even if set up
```

### `humanlayer thoughts sync`
Manually sync thoughts to the thoughts repository. Usually automatic via git hooks.

```bash
humanlayer thoughts sync
humanlayer thoughts sync -m "Updated architecture notes"
```

### `humanlayer thoughts status`
Check setup status and see any uncommitted changes.

```bash
humanlayer thoughts status
```

### `humanlayer thoughts config`
View or edit configuration.

```bash
humanlayer thoughts config          # View config
humanlayer thoughts config --edit   # Open in $EDITOR
humanlayer thoughts config --json   # Output as JSON
```

---

## Sharing Thoughts Between Two Users

### Option A: Shared Git Remote (Different Machines/Users)

1. **User 1** creates and pushes a thoughts repo:
   ```bash
   cd ~/thoughts
   git remote add origin git@github.com:you/thoughts-shared.git
   git push -u origin main
   ```

2. **User 2** clones the same repo:
   ```bash
   git clone git@github.com:you/thoughts-shared.git ~/thoughts
   ```

3. Both users initialize in the same code repo:
   ```bash
   cd /shared/code/repo
   humanlayer thoughts init  # Each person uses their own username
   ```

4. When either syncs, the other gets their thoughts on next pull:
   - Auto-sync happens on git commits (via post-commit hook)
   - Manual sync: `humanlayer thoughts sync`

### How User Discovery Works

When User 1 syncs, User 2's thoughts directory appears automatically:
- The `updateSymlinksForNewUsers()` function scans for new user directories
- Symlinks are auto-created in `thoughts/username/` for each discovered user
- No manual configuration needed!

---

## Configuration File

Location: `~/.config/humanlayer/humanlayer.json`

```json
{
  "thoughts": {
    "thoughtsRepo": "~/thoughts",
    "reposDir": "repos",
    "globalDir": "global",
    "user": "yourname",
    "repoMappings": {
      "/home/user/code/project": "project"
    }
  }
}
```

Key fields:
- `thoughtsRepo`: Path to your central thoughts git repository
- `user`: Your username (appears as directory name)
- `repoMappings`: Maps code repo paths to thoughts directory names

---

## WSL-Specific Considerations

The thoughts system works the same on WSL as native Linux. Key notes:

1. **Use WSL paths, not Windows paths**
   - Good: `/home/username/thoughts`
   - Bad: `/mnt/c/Users/...` (avoid cross-filesystem symlinks)

2. **Git SSH setup in WSL**
   ```bash
   # Generate SSH key if needed
   ssh-keygen -t ed25519 -C "your@email.com"

   # Add to ssh-agent
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519

   # Add public key to GitHub/GitLab
   cat ~/.ssh/id_ed25519.pub
   ```

3. **Line endings** - Configure git for Linux-style:
   ```bash
   git config --global core.autocrlf input
   ```

4. **File permissions** - WSL typically handles Unix permissions correctly

---

## Using with Claude Code

Once thoughts are set up:

1. **Claude can read your thoughts** via the `thoughts/` directory
2. **The `searchable/` directory** contains hard links for search tools
3. **CLAUDE.md** provides auto-generated context about structure

### What to Put Where

**Personal repo thoughts** (`thoughts/yourname/`):
- Architecture decisions
- TODO lists and planning
- Debugging notes
- Design trade-offs

**Shared repo thoughts** (`thoughts/shared/`):
- Team conventions
- Project documentation
- Shared decisions

**Global thoughts** (`thoughts/global/yourname/`):
- Cross-project patterns
- Personal learning notes
- Company-wide standards

---

## Automatic Syncing

Git hooks handle syncing automatically:

1. **Pre-commit hook**: Prevents `thoughts/` from being committed to code repo
2. **Post-commit hook**: Syncs thoughts changes after each code commit

The hooks are installed during `humanlayer thoughts init`.

---

## Troubleshooting

### "Thoughts not configured"
```bash
humanlayer thoughts init
```

### "Not in a git repository"
```bash
git init
humanlayer thoughts init
```

### Sync not working
```bash
# Check hooks are installed
ls -la .git/hooks/

# Manual sync
humanlayer thoughts sync

# Check status
humanlayer thoughts status
```

### Can't see other user's thoughts
Ensure both users have synced and the remote repo is shared:
```bash
# Pull latest from remote
cd ~/thoughts
git pull

# Then sync in your code repo
cd /path/to/code
humanlayer thoughts sync
```

---

## Profile Support (Advanced)

For separate thoughts repos per context (e.g., work vs personal):

```bash
# Create a profile
humanlayer thoughts profile create work --repo ~/thoughts-work

# Initialize with profile
humanlayer thoughts init --profile work

# List profiles
humanlayer thoughts profile list
```

---

## Code References

- CLI commands: `hlyr/src/commands/thoughts.ts:12-84`
- Sync implementation: `hlyr/src/commands/thoughts/sync.ts:32-99`
- Init implementation: `hlyr/src/commands/thoughts/init.ts:319-728`
- Config resolution: `hlyr/src/thoughtsConfig.ts:178-222`
- User discovery: `hlyr/src/thoughtsConfig.ts:361-422`
- Documentation: `hlyr/THOUGHTS.md`

## Related Documentation

- Full THOUGHTS.md: `hlyr/THOUGHTS.md`
- README commands: `hlyr/README.md:188-256`
