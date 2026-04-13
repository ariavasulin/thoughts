# Remove .beads and All Evidence from Repo and Git History

## Overview

Completely purge the `.beads/` issue-tracking system from the repository — working tree, git history, and all file references. The repo previously used `bd` (beads) for issue tracking but has moved on. No trace should remain.

## Current State Analysis

### Tracked .beads files (in git):
- `.beads/.gitignore`
- `.beads/README.md`
- `.beads/config.yaml`
- `.beads/interactions.jsonl`
- `.beads/issues.jsonl`
- `.beads/metadata.json`

### Untracked .beads files (local only):
- `.beads/beads.db`
- `.beads/daemon.lock`
- `.beads/daemon.log`
- `.beads/last-touched`
- `.beads/.local_version`

### References in other files:
- `.gitattributes` — entire file is beads merge driver config (lines 2-3)
- `AGENTS.md` — line 3 references `bd` (beads), lines 7-13 are `bd` commands, line 27 has `bd sync`

### Git history:
- 92 commits total, single branch (main)
- 2 commits touched `.beads/`: `dc00673`, `138a419`
- 1 commit mentions "beads" in message: `dc00673`

### Worktree copy:
- `.trees/live-artifact-push/.beads` — already gitignored, will be cleaned manually

## Desired End State

- No `.beads/` directory on disk
- No `.gitattributes` file (it was entirely beads-related)
- `AGENTS.md` has no beads/bd references
- No `.beads/` files in any git commit in history
- No `.gitattributes` in any git commit in history
- Remote origin force-pushed with clean history

### Verification:
```bash
# No .beads on disk
ls .beads 2>/dev/null && echo "FAIL" || echo "PASS"

# No .beads in git history
git log --all --diff-filter=A -- '.beads/*' | wc -l  # should be 0

# No .gitattributes in history
git log --all --diff-filter=A -- '.gitattributes' | wc -l  # should be 0

# No beads references in tracked files
git grep -i beads | wc -l  # should be 0

# Clean remote
git fetch origin && git diff origin/main..main  # should be empty
```

## What We're NOT Doing

- Not adding `.beads/` to `.gitignore` (purging completely)
- Not preserving any beads issue data (it's disposable)
- Not rewriting commit messages that mention "beads" (only purging files)

## Implementation Approach

Use `git filter-repo` to rewrite history removing `.beads/` and `.gitattributes` from all commits, then clean up remaining references in `AGENTS.md`, purge local files, and force push.

## Phase 1: Clean Up File References

### Overview
Remove beads references from files that will persist (AGENTS.md). Delete .gitattributes.

### Changes Required:

#### 1. AGENTS.md
**File**: `AGENTS.md`
**Changes**: Remove line 3 (bd onboard reference), lines 7-13 (bd commands), line 27 (bd sync). Keep the rest of the session completion workflow intact.

#### 2. .gitattributes
**File**: `.gitattributes`
**Changes**: Delete entirely (all content is beads-related).

### Success Criteria:

#### Automated Verification:
- [ ] `grep -i beads AGENTS.md` returns nothing
- [ ] `test ! -f .gitattributes` passes

---

## Phase 2: Rewrite Git History

### Overview
Use `git filter-repo` to remove `.beads/` and `.gitattributes` from all historical commits.

### Commands:
```bash
git filter-repo --invert-paths --path .beads/ --path .gitattributes --force
```

Note: `git filter-repo` removes the remote after rewriting. We'll re-add it.

```bash
git remote add origin https://github.com/ariavasulin/YouLab.git
```

### Success Criteria:

#### Automated Verification:
- [ ] `git log --all --diff-filter=A -- '.beads/*' | wc -l` returns 0
- [ ] `git log --all --diff-filter=A -- '.gitattributes' | wc -l` returns 0
- [ ] `git log --all --oneline | wc -l` still shows ~92 commits (no commits lost)

---

## Phase 3: Purge Local Files and Force Push

### Overview
Remove .beads/ from disk (including worktree copy) and force push clean history.

### Commands:
```bash
rm -rf .beads/
rm -rf .trees/live-artifact-push/.beads 2>/dev/null
git push --force origin main
```

### Success Criteria:

#### Automated Verification:
- [ ] `ls .beads 2>/dev/null` returns nothing
- [ ] `git grep -i beads` returns nothing (except this plan file in thoughts/, which is gitignored)
- [ ] `git push` succeeds
- [ ] `git status` is clean

## References

- Beads integration plan (historical): `thoughts/shared/plans/beads-integration.md`
