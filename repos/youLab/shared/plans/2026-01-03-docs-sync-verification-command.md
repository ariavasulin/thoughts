# Docs Sync Verification Command Implementation Plan

## Overview

Create a slash command `/verify_docs` that compares `/docs` and `CLAUDE.md` against recent codebase changes to ensure 100% up-to-date accuracy. The command spawns parallel agents to analyze documentation vs implementation, then produces a discrepancy report.

## Current State Analysis

### Documentation Structure
- **`/docs`**: Docsify-powered site with 20+ markdown files
  - `README.md` - Overview with architecture diagram, status table, tech stack
  - `HTTP-Service.md` - Endpoints, request/response examples, streaming details
  - `Memory-System.md`, `Agent-System.md` - Core system documentation
  - `Changelog.md` - Version history with migration notes
  - `_sidebar.md` - Navigation structure
- **`CLAUDE.md`**: Project structure, commands, key files, roadmap

### Key Discovery: No Automated Sync Tracking
- Docs and code evolve independently
- Last docs update: commit `560cdbc` ("docs: update documentation for Phase 1 HTTP service")
- Subsequent code changes may have introduced undocumented features or changed behavior

### Git History Pattern
```
70314e1 feat: implement streaming responses with Letta v0.16.1 compatibility
9edb6a0 refactor: extract strategy agent into standalone package
0b5608d feat: add agent-optimized check and verify commands
560cdbc docs: update documentation for Phase 1 HTTP service  <-- last doc update
```

## Desired End State

A `/verify_docs` slash command that:
1. Identifies the last documentation update commit
2. Analyzes all code changes since that commit
3. Cross-references doc content with actual implementation
4. Produces a structured discrepancy report
5. Optionally generates fix suggestions

### Verification
- Running `/verify_docs` produces a markdown report listing any discrepancies
- Report includes specific file:line references for both docs and code
- Command can be run periodically to catch documentation drift

## What We're NOT Doing

- Automatically updating docs (human review required)
- Modifying git history or commit messages
- Creating a CI/CD integration (future scope)
- Validating code correctness (only doc accuracy)

## Implementation Approach

Create a new slash command that:
1. Uses git to find the last doc update and subsequent changes
2. Spawns parallel agents to analyze specific doc-to-code mappings
3. Synthesizes findings into a discrepancy report
4. Writes report to `thoughts/shared/research/` for review

## Phase 1: Create the Slash Command

### Overview
Create `.claude/commands/verify_docs.md` with the command logic.

### Changes Required:

#### 1. Command File
**File**: `.claude/commands/verify_docs.md`

```markdown
---
description: Verify documentation accuracy against recent codebase changes
model: opus
---

# Verify Documentation Sync

You are tasked with verifying that `/docs` and `CLAUDE.md` are 100% accurate compared to the current codebase state. You will identify discrepancies between documentation and implementation.

## Initial Setup

When this command is invoked, respond with:
```
I'll verify documentation accuracy against the codebase. Starting analysis...
```

Then immediately proceed with the verification steps.

## Steps

### Step 1: Identify Documentation Baseline

Run these git commands to understand the timeline:

1. **Find last docs update**:
   ```bash
   git log -1 --format="%H %s" -- docs/ CLAUDE.md
   ```

2. **List code changes since docs update**:
   ```bash
   git log --oneline <last_docs_commit>..HEAD -- 'src/' 'tests/' '*.py' 'pyproject.toml' 'Makefile'
   ```

3. **Get list of changed files**:
   ```bash
   git diff --name-only <last_docs_commit>..HEAD -- 'src/' 'tests/'
   ```

### Step 2: Create Doc-to-Code Mapping

Build a mapping of which docs cover which code areas:

| Doc File | Code Areas Covered |
|----------|-------------------|
| `HTTP-Service.md` | `server/main.py`, `server/agents.py`, `server/schemas.py` |
| `Memory-System.md` | `memory/blocks.py`, `memory/strategies.py`, `memory/manager.py` |
| `Agent-System.md` | `agents/base.py`, `agents/default.py`, `agents/templates.py` |
| `Pipeline.md` | `pipelines/letta_pipe.py` |
| `Strategy-Agent.md` | `agents/strategy.py`, `server/strategy.py` |
| `Configuration.md` | `config/settings.py` |
| `API.md`, `Schemas.md` | `server/schemas.py`, endpoint implementations |
| `CLAUDE.md` | Overall structure, commands, key files list |

### Step 3: Spawn Parallel Verification Agents

Launch parallel Task agents for each doc-code pair that has changes:

**For each changed code area**, spawn a verification agent:

```
Task: Verify docs/<doc_name>.md accuracy

Compare the documentation claims in docs/<doc_name>.md against the actual implementation in:
- <list of relevant source files>

Check for:
1. **API Accuracy**: Do documented endpoints, parameters, and responses match implementation?
2. **Code Examples**: Do code snippets work with current implementation?
3. **Feature Completeness**: Are all implemented features documented?
4. **Removed Features**: Are any documented features no longer implemented?
5. **Configuration**: Do documented config options match actual settings?

Return findings in this format:
## <Doc File> Verification

### Accurate
- [item] - verified at [file:line]

### Discrepancies Found
- [issue]: [doc says X] vs [code does Y] at [file:line]

### Missing Documentation
- [feature] implemented at [file:line] - not documented
```

### Step 4: Verify CLAUDE.md Specifically

Spawn a dedicated agent for CLAUDE.md verification:

```
Verify CLAUDE.md accuracy:

1. **Project Structure**: Does the documented structure match `ls -la src/letta_starter/`?
2. **Commands**: Do all documented commands work? Run `make --help` and compare.
3. **Key Files**: Do all listed files exist and have correct descriptions?
4. **Roadmap**: Do completed/pending phases match actual state?
```

### Step 5: Synthesize Discrepancy Report

After all agents complete, create a structured report:

**File**: `thoughts/shared/research/YYYY-MM-DD-docs-verification.md`

```markdown
---
date: [ISO timestamp]
researcher: [name]
git_commit: [current HEAD]
docs_baseline: [last docs commit]
changes_analyzed: [count of commits since baseline]
status: complete
---

# Documentation Verification Report

**Analysis Period**: [last_docs_commit]..HEAD
**Commits Analyzed**: [N]
**Files Changed**: [list]

## Summary

| Category | Count |
|----------|-------|
| Docs Verified Accurate | X |
| Minor Discrepancies | Y |
| Major Discrepancies | Z |
| Missing Documentation | W |

## Detailed Findings

### docs/HTTP-Service.md
**Status**: [Accurate / Needs Update]

#### Discrepancies
- [specific issue with file:line references]

#### Missing
- [undocumented feature with file:line]

### docs/Agent-System.md
...

### CLAUDE.md
...

## Recommended Actions

### Priority 1 (Major Discrepancies)
- [ ] [specific fix needed]

### Priority 2 (Minor Updates)
- [ ] [specific update needed]

### Priority 3 (Enhancements)
- [ ] [optional improvements]
```

### Step 6: Sync and Present

1. Run `humanlayer thoughts sync`
2. Present summary to user with report location
3. Ask if they want to proceed with fixes

## Important Notes

- All agents are read-only (no modifications)
- Focus on factual accuracy, not style
- Include specific file:line references for both docs and code
- Prioritize discrepancies by impact (API changes > internal details)
- Don't flag documentation style issues unless they cause confusion
```

### Success Criteria:

#### Automated Verification:
- [ ] Command file exists at `.claude/commands/verify_docs.md`
- [ ] Lint passes: `make lint-fix`
- [ ] Command appears in available commands list

#### Manual Verification:
- [ ] Run `/verify_docs` and verify it produces a coherent discrepancy report
- [ ] Report correctly identifies at least one known discrepancy (if any exist)
- [ ] Report is written to the correct location in thoughts/shared/research/

---

## Phase 2: Add Supporting Shell Script (Optional)

### Overview
Create a helper script to quickly get doc-vs-code diff stats.

### Changes Required:

#### 1. Helper Script
**File**: `hack/docs-diff.sh`

```bash
#!/bin/bash
# Get documentation change analysis
# Usage: ./hack/docs-diff.sh

set -euo pipefail

# Find last docs commit
LAST_DOCS_COMMIT=$(git log -1 --format="%H" -- docs/ CLAUDE.md 2>/dev/null || echo "")

if [ -z "$LAST_DOCS_COMMIT" ]; then
    echo "No documentation commits found"
    exit 1
fi

echo "=== Documentation Sync Analysis ==="
echo ""
echo "Last docs update: $(git log -1 --format='%h %s (%ar)' $LAST_DOCS_COMMIT)"
echo ""
echo "=== Code Changes Since Docs Update ==="
git log --oneline "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' '*.py' 'pyproject.toml' 'Makefile' 2>/dev/null || echo "No changes"
echo ""
echo "=== Changed Files ==="
git diff --name-only "$LAST_DOCS_COMMIT"..HEAD -- 'src/' 'tests/' 2>/dev/null || echo "None"
```

### Success Criteria:

#### Automated Verification:
- [ ] Script is executable: `chmod +x hack/docs-diff.sh`
- [ ] Script runs without error: `./hack/docs-diff.sh`

#### Manual Verification:
- [ ] Output correctly shows last docs commit
- [ ] Output lists code changes since that commit

---

## Testing Strategy

### Unit Tests:
- N/A (this is a slash command, not code)

### Integration Tests:
- N/A

### Manual Testing Steps:
1. Run `/verify_docs` in a Claude session
2. Verify it spawns parallel agents for each doc area
3. Verify final report is produced in correct format
4. Verify report is synced to thoughts directory
5. Introduce a deliberate discrepancy (e.g., change a function name in code) and verify the command catches it

## Performance Considerations

- Parallel agents maximize efficiency
- Large docs may need chunked analysis
- Consider caching git analysis results if run frequently

## References

- Existing command: `.claude/commands/research_codebase.md`
- Docs structure: `docs/_sidebar.md`
- CLAUDE.md location: `/CLAUDE.md`
