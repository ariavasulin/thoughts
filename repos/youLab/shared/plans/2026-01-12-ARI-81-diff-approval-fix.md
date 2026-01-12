# ARI-81 Phase 4 Fix: Snippet-Based Diff Approval

## Overview

Fix the diff approval system so that diffs store and display specific snippets instead of entire file contents. This resolves all 4 blocking issues documented in the Phase C plan.

## Current State Analysis

**Root Cause**: `propose_edit()` at `storage/blocks.py:279` stores the entire file content as `current_value`:
```python
current = self.storage.read_block(block_label) or ""  # ENTIRE file
```

This causes 4 cascading failures:

| Issue | Symptom | Root Cause |
|-------|---------|------------|
| 4.4 | Diff view not displaying | Frontend tries to match `oldValue` to lines, but it's the entire file |
| 4.5 | Shows TOML instead of Markdown | Less critical - format display issue |
| 4.6 | Manual edits don't persist | TOML↔MD conversion loss (separate issue) |
| 4.7 | Approve deletes entire block | `replace(entire_file, snippet)` replaces everything |

**Key Insight**: The agent tool `edit_memory_block` doesn't provide what to replace - it only provides what to add. For "replace" operations, the system needs to know the old content being replaced.

## Desired End State

After this fix:
1. Agent provides `old_content` parameter for replace operations
2. `propose_edit()` stores the snippet (not entire file) as `current_value`
3. Frontend `buildDiffView()` matches `oldValue` to specific lines (already works)
4. `approve_diff()` replaces just the snippet (already works)
5. Diff view shows red/green lines correctly

### How to Verify
- Create a diff with operation="replace" via agent tool
- Open block in UI → see green/red diff highlighting
- Click Approve → only the specific line changes, not entire file
- Click Reject → diff is removed, content unchanged

## What We're NOT Doing

- **Markdown editing fix** (Issue 4.6) - TOML↔MD conversion is a separate concern
- **LLM diff operation** - Future enhancement, not needed for MVP
- **Multi-line diff support** - Single line replacement is sufficient for now
- **Line number tracking** - Using content matching instead

## Implementation Approach

Two-phase approach:
1. **Phase 1**: Update backend to accept and store `old_content` parameter
2. **Phase 2**: Verify frontend works without changes (it should)

---

## Phase 1: Backend Snippet Support

### Overview
Modify the agent tool and storage layer to pass through `old_content` as `current_value` instead of reading the entire file.

### Changes Required:

#### 1.1 Update `edit_memory_block` Tool Signature
**File**: `src/youlab_server/tools/memory.py`
**Changes**: Add `old_content` parameter for replace operations

```python
def edit_memory_block(
    block: str,
    field: str,
    content: str,
    strategy: str = "append",
    reasoning: str = "",
    old_content: str | None = None,  # NEW: For replace operations
    agent_state: dict[str, Any] | None = None,
) -> str:
    """
    Propose an update to a memory block.

    This creates a pending diff that the user must approve.
    The change is NOT applied immediately.

    Args:
        block: The block label (e.g., "student", "journey")
        field: The field being updated
        content: The new content to add or replace with
        strategy: How to apply the change:
            - "append": Add content to end of block
            - "replace": Replace old_content with content (requires old_content)
            - "full_replace": Replace entire block with content
        reasoning: Explanation for why this change is being proposed
        old_content: For "replace" strategy, the exact content being replaced.
                     Must match a line in the current block.
        agent_state: Internal state passed by Letta

    Returns:
        Confirmation message with diff ID
    """
```

#### 1.2 Update Tool Call to `propose_edit()`
**File**: `src/youlab_server/tools/memory.py`
**Changes**: Pass `old_content` to `propose_edit()`

Around line 77, update the call:

```python
diff = manager.propose_edit(
    agent_id=agent_id,
    block_label=block,
    field=field,
    operation=strategy,
    proposed_value=content,
    reasoning=reasoning or "No reasoning provided",
    old_content=old_content,  # NEW: Pass through old_content
)
```

#### 1.3 Update `propose_edit()` to Use `old_content`
**File**: `src/youlab_server/storage/blocks.py`
**Changes**: Accept `old_content` parameter and use it as `current_value`

```python
def propose_edit(
    self,
    agent_id: str,
    block_label: str,
    field: str | None,
    operation: str,
    proposed_value: str,
    reasoning: str,
    confidence: str = "medium",
    source_query: str | None = None,
    old_content: str | None = None,  # NEW parameter
) -> PendingDiff:
    """
    Create a pending diff for an agent-proposed edit.

    This does NOT apply the edit - it creates a diff for user approval.

    Args:
        old_content: For "replace" operations, the specific content being replaced.
                     If not provided, reads entire block (legacy behavior for append).
    """
    # Determine current_value based on operation
    if operation == "replace" and old_content:
        # Use provided snippet for replace operations
        current = old_content
    elif operation == "full_replace":
        # For full_replace, store entire file to show complete diff
        current = self.storage.read_block(block_label) or ""
    else:
        # For append, current_value is not used in approval logic
        # Store empty string to indicate "adding new content"
        current = ""

    diff = PendingDiff.create(
        user_id=self.user_id,
        agent_id=agent_id,
        block_label=block_label,
        field=field,
        operation=operation,
        current_value=current,  # Now uses snippet for replace
        proposed_value=proposed_value,
        reasoning=reasoning,
        confidence=confidence,
        source_query=source_query,
    )

    self.diffs.save(diff)
    self.logger.info(
        "diff_proposed",
        diff_id=diff.id,
        block=block_label,
        agent=agent_id,
        operation=operation,
    )

    return diff
```

#### 1.4 Update Validation in `approve_diff()`
**File**: `src/youlab_server/storage/blocks.py`
**Changes**: Improve error message and handle empty `current_value` for append

The existing logic at lines 325-337 should work, but add clarity:

```python
elif diff.operation == "replace":
    if not diff.current_value:
        msg = f"Cannot apply replace diff {diff_id}: no old content specified"
        raise ValueError(msg)
    if diff.current_value not in current_content:
        msg = (
            f"Cannot apply diff {diff_id}: old content not found in block. "
            f"Expected to find: '{diff.current_value[:50]}...'. "
            "The block may have been modified since the diff was created."
        )
        raise ValueError(msg)
    # Replace the specific old value with the new value
    new_content = current_content.replace(
        diff.current_value, diff.proposed_value, 1
    )
```

#### 1.5 Update Agent System Prompt
**File**: Agent configuration in Letta or TOML config
**Changes**: Update tool instructions to explain `old_content` parameter

Add to agent's tool usage instructions:
```
When using edit_memory_block with strategy="replace":
- You MUST provide the old_content parameter with the exact line being replaced
- The old_content must match exactly (including whitespace) to a line in the block
- Example: To change "name = 'Alice'" to "name = 'Bob'":
  - old_content: "name = 'Alice'"
  - content: "name = 'Bob'"
  - strategy: "replace"
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`
- [ ] Tool signature is correct in Letta: `curl ${LETTA_URL}/tools | grep edit_memory_block`

#### Manual Verification:
- [ ] Agent can call `edit_memory_block` with `old_content` parameter
- [ ] Diff JSON file shows `current_value` as snippet (not entire file)
- [ ] Backend logs show correct operation type

**Implementation Note**: After completing this phase, verify by creating a test diff and inspecting the JSON file in `pending_diffs/`.

---

## Phase 2: End-to-End Verification

### Overview
Verify the frontend displays diffs correctly now that backend sends snippets.

### Verification Steps:

#### 2.1 Create Test Diff via API
```bash
# First, ensure a block exists
curl -X POST "${YOULAB_API_BASE_URL}/users/test-user/blocks" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "test_block",
    "content": "name = \"Alice\"\nage = 25\ncity = \"NYC\"",
    "format": "toml"
  }'

# Create a diff with old_content
curl -X POST "${YOULAB_API_BASE_URL}/users/test-user/blocks/test_block/diffs" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "test-agent",
    "field": "name",
    "operation": "replace",
    "old_content": "name = \"Alice\"",
    "proposed_value": "name = \"Bob\"",
    "reasoning": "User prefers Bob"
  }'
```

#### 2.2 Verify Frontend Display
1. Navigate to You > Profile
2. Click on `test_block`
3. Verify:
   - Red line shows: `name = "Alice"` (deletion)
   - Green line shows: `name = "Bob"` (addition)
   - Approve/Reject buttons appear below green line

#### 2.3 Test Approve Flow
1. Click "Approve"
2. Verify:
   - Block content now has `name = "Bob"`
   - Other lines (`age = 25`, `city = "NYC"`) unchanged
   - Diff is removed from pending list
   - Toast shows success

#### 2.4 Test Reject Flow
1. Create another test diff
2. Click "Reject"
3. Verify:
   - Block content unchanged
   - Diff removed from pending list
   - Toast shows success

### Success Criteria:

#### Automated Verification:
- [ ] Frontend builds: `cd OpenWebUI/open-webui && npm run build`
- [ ] API returns correct diff format: `curl ${YOULAB_API_BASE_URL}/users/test-user/blocks/test_block/diffs`

#### Manual Verification:
- [ ] Diff view shows red/green line highlighting
- [ ] Approve changes only the targeted line
- [ ] Reject removes diff without changing content
- [ ] Multiple diffs on same block display correctly
- [ ] Stale diff (content changed) shows appropriate error on approve

---

## Phase 3: Handle Append Operation Display

### Overview
For "append" operations, `current_value` is empty. The frontend should show only a green "addition" line at the end.

### Changes Required:

#### 3.1 Update `buildDiffView()` for Append
**File**: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte`
**Changes**: Handle diffs with empty `oldValue` (append operations)

```typescript
function buildDiffView() {
    if (diffsWithValues.length === 0 || editMode) {
        diffLines = [];
        return;
    }

    const lines = tomlContent.split('\n');
    const result: DiffLine[] = [];

    // Separate append diffs from replace diffs
    const appendDiffs: DiffWithValues[] = [];
    const diffsByLine: Map<number, DiffWithValues> = new Map();

    for (const diff of diffsWithValues) {
        if (diff.operation === 'append' || !diff.oldValue) {
            // Append diffs go at the end
            appendDiffs.push(diff);
        } else if (diff.lineNumber !== undefined) {
            diffsByLine.set(diff.lineNumber, diff);
        } else {
            // Find the line number by matching content
            const lineIdx = lines.findIndex(line => line.trim() === diff.oldValue?.trim());
            if (lineIdx >= 0) {
                diff.lineNumber = lineIdx;
                diffsByLine.set(lineIdx, diff);
            }
        }
    }

    // Build unified view for existing lines
    for (let i = 0; i < lines.length; i++) {
        const diff = diffsByLine.get(i);
        if (diff) {
            // This line has a diff - show deletion then addition
            if (diff.oldValue) {
                result.push({
                    type: 'deletion',
                    content: diff.oldValue,
                    lineNum: i + 1,
                    diffId: diff.id,
                    diff: diff
                });
            }
            if (diff.newValue) {
                result.push({
                    type: 'addition',
                    content: diff.newValue,
                    diffId: diff.id,
                    diff: diff,
                    isLastInDiff: true
                });
            }
        } else {
            // Regular context line
            result.push({
                type: 'context',
                content: lines[i],
                lineNum: i + 1
            });
        }
    }

    // Add append diffs at the end
    for (const diff of appendDiffs) {
        result.push({
            type: 'addition',
            content: diff.newValue || '',
            diffId: diff.id,
            diff: diff,
            isLastInDiff: true
        });
    }

    diffLines = result;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Frontend builds: `cd OpenWebUI/open-webui && npm run build`

#### Manual Verification:
- [ ] Append diffs show as green line at end of content
- [ ] Append diffs have approve/reject buttons
- [ ] Approving append adds content to end of block

---

## Testing Strategy

### Unit Tests
Add tests for the updated `propose_edit()` function:

```python
# tests/test_storage/test_blocks.py

def test_propose_edit_with_old_content(self, manager, user_storage, sample_toml):
    """Propose edit stores old_content as current_value for replace."""
    user_storage.write_block("student", sample_toml, author="system")

    diff = manager.propose_edit(
        agent_id="agent1",
        block_label="student",
        field="name",
        operation="replace",
        proposed_value='name = "Bob"',
        reasoning="User prefers Bob",
        old_content='name = "Alice"',  # Specific snippet
    )

    # current_value should be the snippet, not entire file
    assert diff.current_value == 'name = "Alice"'
    assert diff.proposed_value == 'name = "Bob"'


def test_propose_edit_append_empty_current(self, manager, user_storage, sample_toml):
    """Propose edit with append stores empty current_value."""
    user_storage.write_block("student", sample_toml, author="system")

    diff = manager.propose_edit(
        agent_id="agent1",
        block_label="student",
        field="notes",
        operation="append",
        proposed_value='notes = "New note"',
        reasoning="Adding notes",
    )

    # current_value should be empty for append
    assert diff.current_value == ""
    assert diff.proposed_value == 'notes = "New note"'
```

### Integration Tests
- Create diff via tool call, verify JSON file contents
- Approve diff via API, verify block changes correctly
- Reject diff via API, verify diff removed

### Manual Testing
Follow Phase 2 verification steps

## Migration Notes

### Existing Pending Diffs
Diffs created before this fix have `current_value` = entire file. These will:
- **Display**: May not render correctly (no line match)
- **Approve**: May fail validation (staleness check)
- **Recommend**: Reject old diffs, let agent recreate with new format

### Backwards Compatibility
- The `old_content` parameter is optional in the tool signature
- If not provided with operation="replace", the old behavior applies (reads entire file)
- This allows gradual migration while agents are updated

## Performance Considerations

- No performance impact - just changing what value is stored
- JSON file sizes may be smaller (snippet vs full file)

## References

- Research analysis: `thoughts/shared/research/2026-01-12-ARI-81-phase-4-diff-approval-analysis.md`
- Parent plan: `thoughts/shared/plans/2026-01-12-ARI-81-phase-c-demo-mvp.md`
- Frontend diff view: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte:58-116`
- Backend propose_edit: `src/youlab_server/storage/blocks.py:263-302`
- Backend approve_diff: `src/youlab_server/storage/blocks.py:304-366`

---

*Plan created: 2026-01-12*
