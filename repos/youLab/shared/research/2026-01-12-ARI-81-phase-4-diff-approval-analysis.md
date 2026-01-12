---
date: 2026-01-12T16:48:30+07:00
researcher: ariasulin
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "ARI-81 Phase 4 Diff Approval UI - Root Cause Analysis"
tags: [research, codebase, memory-blocks, diff-approval, phase-c, ARI-81]
status: complete
last_updated: 2026-01-12
last_updated_by: ariasulin
---

# Research: ARI-81 Phase 4 Diff Approval UI - Root Cause Analysis

**Date**: 2026-01-12T16:48:30+07:00
**Researcher**: ariasulin
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab

## Research Question

Phase 4 of ARI-81 (Diff Approval UI) is encountering cascading errors where every fix attempt breaks something else. Analyze the root causes of the documented issues and determine whether to refactor or proceed with targeted fixes.

## Summary

After comprehensive analysis of today's changes across ARI-79, ARI-80, and ARI-81, **the core problem is an architectural mismatch in how diffs store and apply changes**. The `propose_edit()` function stores the entire file content as `current_value`, but `approve_diff()` treats `current_value` as a snippet to replace. This fundamental disconnect causes all four Phase 4 issues.

**Recommendation**: A targeted refactor of the diff creation/approval flow is needed before proceeding with UI fixes.

## Detailed Findings

### Today's Sprint Summary

| Sprint | Status | What Was Built |
|--------|--------|----------------|
| ARI-79 | Complete | YouLab branding, iA Writer theme, Modules sidebar, youlabMode |
| ARI-80 | Complete | Git storage, TOML↔MD conversion, UserBlockManager, CRUD API, "You" page |
| ARI-81 | Phase 4 Blocked | Phases 1-3 complete, Phase 4 (Diff Approval) has 4 blocking issues |

### Issue 4.4: Diff View Not Displaying Green/Red Lines

**Root Cause**: Data structure mismatch between how diffs are created and how the frontend expects them.

**Code Path**:
1. `propose_edit()` at `storage/blocks.py:279` stores:
   ```python
   current = self.storage.read_block(block_label) or ""  # ENTIRE file
   diff = PendingDiff.create(
       current_value=current,      # ENTIRE file content
       proposed_value=proposed_value,  # Just the new value
   )
   ```

2. `list_pending_diffs()` at `storage/blocks.py:386-387` returns:
   ```python
   "oldValue": d.current_value,  # ENTIRE file
   "newValue": d.proposed_value,  # Just the new line
   ```

3. `buildDiffView()` in `BlockDetailModal.svelte:73-78` tries to match:
   ```typescript
   const lineIndex = lines.findIndex(
       (line) => line.trim() === diff.oldValue?.trim()
   );
   ```
   This fails because `oldValue` is the entire file, not a single line.

**Why It Fails**: The frontend expects `oldValue` to be the specific text being replaced (a line or snippet), but the backend provides the entire file content.

### Issue 4.5: Diff View Shows TOML Instead of Markdown

**Root Cause**: Design decision conflict between storage format and display format.

**Code Path**:
1. Storage layer uses TOML (`storage/blocks.py:56-58`, `100-119`)
2. Diffs are created against TOML content (`storage/blocks.py:279`)
3. `BlockDetailModal.svelte:135` stores: `tomlContent = block.contentToml`
4. When `showDiffView` is true (lines 267-320), it displays `tomlContent`
5. When editing manually, it shows `markdownContent`

**Design Decision Needed**: The system stores TOML but users expect to see Markdown. Options:
- A) Create diffs against Markdown representation (requires conversion)
- B) Show TOML diffs with a "View as Markdown" option
- C) Convert TOML diffs to Markdown on display (complex, lossy)

### Issue 4.6: Manual Edits Do Not Persist

**Root Cause**: Round-trip data loss in TOML↔Markdown conversion.

**Code Path**:
1. User edits `markdownContent` in textarea
2. `save()` calls `updateBlock()` with `format='markdown'`
3. Backend calls `update_block_from_markdown()` at `storage/blocks.py:67-98`
4. `markdown_to_toml()` converts at `convert.py:74-133`
5. Converted TOML may differ from original

**Data Loss Scenarios in `convert.py`**:
| TOML Type | Markdown Output | Round-trip Result |
|-----------|-----------------|-------------------|
| Boolean `true` | "Yes" | String "Yes" |
| Number `42` | "42" | String "42" |
| Nested dict | `str(value)` | Flattened string |
| Mixed list/text | List items only | Text lost |

**Example**:
```toml
# Original TOML
active = true
score = 95
```
```markdown
# Converted to Markdown
## Active
Yes

## Score
95
```
```toml
# Round-trip back to TOML
active = "Yes"
score = "95"
```

The data loss causes the re-fetched content to differ from what was saved, making it appear that edits don't persist.

### Issue 4.7: Approve Diff Deletes Entire Block

**Root Cause**: The `approve_diff()` function's replace logic assumes `current_value` is a snippet, but it's actually the entire file.

**Code Path**:
1. `approve_diff()` at `storage/blocks.py:325-330`:
   ```python
   if diff.current_value and diff.current_value in current_content:
       new_content = current_content.replace(
           diff.current_value, diff.proposed_value, 1
       )
   ```

2. If `current_value` equals the entire file:
   - `diff.current_value in current_content` returns `True` (entire file matches itself)
   - `replace(entire_file, proposed_value)` replaces everything with just the new value
   - Result: entire block content replaced with a single line

**Why "Fixed" Still Fails**: The fix added validation at lines 331-337 that raises an error if `current_value` isn't found. But when `current_value` IS the entire file, it IS found - and the destructive replace happens anyway.

## Architecture Documentation

### Current Data Flow

```
[Agent calls edit_memory_block tool]
         ↓
[propose_edit() creates PendingDiff]
    • current_value = ENTIRE file content
    • proposed_value = new line/content
    • operation = "replace" | "append" | "full_replace"
         ↓
[Diff stored as JSON in pending_diffs/]
         ↓
[Frontend fetches via GET /blocks/{label}/diffs]
    • Returns oldValue = current_value (entire file)
    • Returns newValue = proposed_value (snippet)
         ↓
[buildDiffView() tries to match oldValue to lines]
    • FAILS because oldValue is entire file
         ↓
[User clicks Approve]
         ↓
[approve_diff() in storage/blocks.py]
    • "replace": current_content.replace(current_value, proposed_value)
    • If current_value == entire file → replaces everything
```

### Expected Data Flow

```
[Agent calls edit_memory_block tool]
         ↓
[propose_edit() creates PendingDiff]
    • current_value = SPECIFIC line/snippet being replaced
    • proposed_value = new line/content to insert
    • operation = "replace" | "append" | "full_replace"
    • (optional) line_number for precise placement
         ↓
[Frontend fetches diffs]
    • oldValue = specific line being replaced
    • newValue = replacement content
         ↓
[buildDiffView() matches oldValue to specific line]
    • SUCCESS - can highlight exact change location
         ↓
[approve_diff() applies change]
    • "replace": finds specific snippet and replaces just that
```

### Key Files

| File | Purpose | Line Numbers |
|------|---------|--------------|
| `storage/blocks.py` | UserBlockManager - CRUD + diff operations | `propose_edit:263-302`, `approve_diff:304-366` |
| `storage/diffs.py` | PendingDiff dataclass + store | `PendingDiff:19-38` |
| `server/blocks.py` | REST API endpoints | `approve_diff:246-259` |
| `storage/convert.py` | TOML↔MD conversion | `toml_to_markdown:12-71`, `markdown_to_toml:74-133` |
| `BlockDetailModal.svelte` | Frontend diff view + editing | `buildDiffView:58-116`, `save:151-165` |

## Historical Context (from thoughts/)

From today's research documents:

- `thoughts/shared/research/2026-01-12-ARI-78-toml-markdown-roundtrip.md` - Documents known conversion limitations
- `thoughts/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md` - Phase 4 `propose_edit` design shows intent was for snippet replacement
- `thoughts/shared/plans/2026-01-12-ARI-81-phase-c-demo-mvp.md` - Documents all 4 blocking issues

## Analysis: Refactor vs. Proceed

### Option A: Targeted Refactor (Recommended)

**Scope**: Fix the diff creation to store snippets, not full files.

**Changes Required**:
1. **`propose_edit()` in `storage/blocks.py`**:
   - Change `current_value` to store only the specific content being replaced
   - Add `line_number` or `field` to precisely identify what's being changed
   - Keep `full_replace` operation for intentional full replacements

2. **`approve_diff()` in `storage/blocks.py`**:
   - Already correctly handles snippet replacement
   - Validation logic is correct once `current_value` is a snippet

3. **Frontend `buildDiffView()`**:
   - Already designed to match snippets to lines
   - Will work once backend sends correct `oldValue`

**Estimated Impact**: ~50 lines of backend code, no frontend changes needed.

### Option B: Work Around Issues

**Approach**: Keep current architecture, add workarounds.

**Required Workarounds**:
1. Issue 4.4: Compute diff client-side by comparing TOML versions
2. Issue 4.5: Convert TOML to Markdown for display (lossy)
3. Issue 4.6: Skip markdown conversion, force TOML editing
4. Issue 4.7: Always use `full_replace`, show full file diffs

**Problems**:
- Compounds technical debt
- Each workaround adds complexity
- User experience suffers (TOML editing, confusing diffs)

### Recommendation

**Proceed with Option A (Targeted Refactor)** because:
1. Root cause is clear and isolated to `propose_edit()`
2. Fix is small (~50 lines) vs. workarounds (~200+ lines)
3. Frontend is already designed correctly
4. Fixes all 4 issues simultaneously
5. No downstream breaking changes (diffs are ephemeral)

## Open Questions

1. **How should agents specify what to replace?**
   - Option: Require agents to provide `old_content` (what to find) and `new_content` (replacement)
   - Option: Require `field` name and let system extract current field value

2. **Should markdown editing be preserved?**
   - The conversion is lossy for complex TOML
   - Consider: TOML-only editing, or simplified Markdown subset

3. **What about existing pending diffs?**
   - Current diffs have `current_value` = entire file
   - Could: auto-expire them, or show warning "legacy diff - please reject and recreate"

## Related Research

- `thoughts/shared/plans/2026-01-12-ARI-81-phase-c-demo-mvp.md` - Original plan with issue documentation
- `thoughts/shared/plans/2026-01-12-ARI-80-memory-system-mvp.md` - Memory system design
- `thoughts/shared/research/2026-01-12-ARI-78-toml-markdown-roundtrip.md` - Conversion analysis
