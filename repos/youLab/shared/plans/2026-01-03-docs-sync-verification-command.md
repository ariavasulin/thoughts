# Docs Sync Verification Command Implementation Plan

## Overview

Create a slash command `/verify_docs` that leverages `/research_codebase` to compare `/docs` and `CLAUDE.md` against recent codebase changes, producing a research document identifying discrepancies. User explicitly decides whether to update docs after review.

## Current State Analysis

### Documentation Structure
- **`/docs`**: Docsify-powered site with 20+ markdown files
- **`CLAUDE.md`**: Project structure, commands, key files, roadmap

### Key Pattern: research_codebase Already Does Most of This
The existing `/research_codebase` command:
- Spawns parallel agents to analyze code
- Produces research documents in `thoughts/shared/research/`
- Uses the documentarian pattern (describe what IS, not what SHOULD BE)

We just need a wrapper that:
1. Runs a helper script to get the git diff context
2. Invokes research_codebase with docs-specific focus
3. Formats output as a discrepancy report

## Desired End State

A `/verify_docs` command that:
1. Runs `hack/docs-diff.sh` to get git context (agent-friendly backpressure)
2. Calls `/research_codebase` with docs verification focus
3. Produces a research doc in `thoughts/shared/research/YYYY-MM-DD-docs-verification.md`

### Verification
- Running `/verify_docs` produces a research document
- User reviews and decides whether to update docs
- No automatic modifications

## What We're NOT Doing

- Automatically updating docs (human review required)
- Reinventing research_codebase logic
- Adding CI/CD integration (future scope)

## Phase 1: Create Agent-Friendly Hack Script

### Overview
Create `hack/docs-diff.sh` with context-efficient backpressure pattern.

### Changes Required:

**File**: `hack/docs-diff.sh`

```bash
#!/bin/bash
# Documentation sync analysis - agent-friendly output
# Usage: ./hack/docs-diff.sh [--verbose]
#
# Default: Compact output for agents (swallow success details)
# --verbose: Full output for human review

set -euo pipefail

VERBOSE="${1:-}"

# Find last docs commit
LAST_DOCS_COMMIT=$(git log -1 --format="%H" -- docs/ CLAUDE.md 2>/dev/null || echo "")

if [ -z "$LAST_DOCS_COMMIT" ]; then
    echo "error: No documentation commits found"
    exit 1
fi

LAST_DOCS_SHORT=$(git log -1 --format="%h" "$LAST_DOCS_COMMIT")
LAST_DOCS_MSG=$(git log -1 --format="%s" "$LAST_DOCS_COMMIT")
LAST_DOCS_DATE=$(git log -1 --format="%ar" "$LAST_DOCS_COMMIT")

# Count changes since docs update
CODE_COMMITS=$(git rev-list --count "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' '*.py' 'pyproject.toml' 'Makefile' 2>/dev/null || echo "0")
CHANGED_FILES=$(git diff --name-only "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' 2>/dev/null | wc -l | tr -d ' ')

# Agent-friendly compact output
if [ "$VERBOSE" != "--verbose" ]; then
    if [ "$CODE_COMMITS" = "0" ]; then
        echo "✓ Docs in sync (no code changes since $LAST_DOCS_SHORT)"
        exit 0
    fi

    echo "docs_baseline: $LAST_DOCS_SHORT"
    echo "docs_date: $LAST_DOCS_DATE"
    echo "code_commits_since: $CODE_COMMITS"
    echo "files_changed: $CHANGED_FILES"
    echo ""
    echo "changed_files:"
    git diff --name-only "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' 2>/dev/null || true
    exit 0
fi

# Verbose output for humans
echo "=== Documentation Sync Analysis ==="
echo ""
echo "Last docs update: $LAST_DOCS_SHORT $LAST_DOCS_MSG ($LAST_DOCS_DATE)"
echo ""

if [ "$CODE_COMMITS" = "0" ]; then
    echo "✓ No code changes since last documentation update"
    exit 0
fi

echo "=== Code Changes Since Docs Update ($CODE_COMMITS commits) ==="
git log --oneline "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' '*.py' 'pyproject.toml' 'Makefile' 2>/dev/null || echo "None"
echo ""
echo "=== Changed Files ($CHANGED_FILES files) ==="
git diff --name-only "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' 2>/dev/null || echo "None"
```

### Success Criteria:

#### Automated Verification:
- [ ] Script exists and is executable
- [ ] `./hack/docs-diff.sh` runs without error
- [ ] `./hack/docs-diff.sh --verbose` shows detailed output

#### Manual Verification:
- [ ] Compact output is parseable by agents
- [ ] Verbose output is readable by humans

---

## Phase 2: Create the Slash Command

### Overview
Create `.claude/commands/verify_docs.md` that wraps research_codebase.

### Changes Required:

**File**: `.claude/commands/verify_docs.md`

```markdown
---
description: Verify /docs and CLAUDE.md accuracy against recent codebase changes
model: opus
---

# Verify Documentation Accuracy

You are tasked with verifying that `/docs` and `CLAUDE.md` accurately reflect the current codebase. This command uses `/research_codebase` to produce a discrepancy report.

## Step 1: Get Git Context

Run the docs-diff script to understand what changed:

```bash
./hack/docs-diff.sh
```

Parse the output to get:
- `docs_baseline`: Last commit that touched docs
- `code_commits_since`: Number of code commits since
- `files_changed`: List of changed source files

If output shows "✓ Docs in sync", respond:
```
Documentation is in sync with the codebase. No code changes since the last documentation update.
```
And stop.

## Step 2: Map Changes to Documentation

Based on the changed files, identify which docs need verification:

| Changed Path Pattern | Docs to Verify |
|---------------------|----------------|
| `server/` | `docs/HTTP-Service.md`, `docs/API.md`, `docs/Schemas.md` |
| `memory/` | `docs/Memory-System.md` |
| `agents/` | `docs/Agent-System.md`, `docs/Strategy-Agent.md` |
| `pipelines/` | `docs/Pipeline.md` |
| `config/` | `docs/Configuration.md`, `docs/Settings.md` |
| Any `.py` | `CLAUDE.md` (structure, key files) |

## Step 3: Research Each Doc Area

For each relevant doc, invoke research_codebase thinking (via Task agents) to compare:

**Research prompt template**:
```
Compare docs/{doc_file}.md against the current implementation.

Focus on these changed files: {list of changed files}

Document:
1. Claims in the doc that match current code (with file:line refs)
2. Claims that no longer match (discrepancies)
3. New features in code not mentioned in docs

Do NOT suggest fixes. Only document what IS vs what the doc SAYS.
```

Run these in parallel for efficiency.

## Step 4: Generate Verification Report

Write to `thoughts/shared/research/YYYY-MM-DD-docs-verification.md`:

```markdown
---
date: {ISO timestamp}
researcher: claude
git_commit: {current HEAD}
docs_baseline: {last docs commit}
code_commits_since: {count}
status: complete
---

# Documentation Verification Report

**Baseline**: {docs_baseline} ({date})
**Analyzed**: {code_commits_since} commits, {files_changed} files

## Summary

| Doc File | Status | Discrepancies |
|----------|--------|---------------|
| HTTP-Service.md | {Accurate/Needs Update} | {count} |
| ... | ... | ... |

## Discrepancies Found

### docs/HTTP-Service.md

#### Inaccurate Claims
- Line X: Doc says "{quote}" but code at `file:line` does Y

#### Missing Coverage
- Feature at `file:line` not documented

### CLAUDE.md
...

## Files Changed Since Docs Update
{list with brief descriptions}
```

## Step 5: Present and Sync

1. Run `humanlayer thoughts sync`
2. Present summary:
```
Verification complete. Report saved to:
thoughts/shared/research/YYYY-MM-DD-docs-verification.md

Summary:
- X docs verified accurate
- Y docs need updates (Z total discrepancies)

Review the report, then tell me which docs to update.
```

## Important Notes

- This is READ-ONLY - no modifications to docs
- Use parallel agents for efficiency
- Include file:line references for all claims
- Focus on accuracy, not style
```

### Success Criteria:

#### Automated Verification:
- [ ] Command file exists at `.claude/commands/verify_docs.md`
- [ ] Command appears in available commands list

#### Manual Verification:
- [ ] `/verify_docs` runs and produces a research document
- [ ] Report format matches specification
- [ ] Parallel agents are used for efficiency

---

## Testing Strategy

### Manual Testing Steps:
1. Run `./hack/docs-diff.sh` - verify compact output
2. Run `./hack/docs-diff.sh --verbose` - verify human-readable output
3. Run `/verify_docs` in Claude session
4. Verify research doc is created in correct location
5. Verify doc accurately identifies any discrepancies

## References

- Context-efficient backpressure: https://www.humanlayer.dev/blog/context-efficient-backpressure
- 12-factor agents: https://www.humanlayer.dev/blog/12-factor-agents
- Existing command: `.claude/commands/research_codebase.md`
- WriteClaudeMD best practices: `.claude/WriteClaudeMD.md`
