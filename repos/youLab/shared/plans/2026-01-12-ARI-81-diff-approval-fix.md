# ARI-81 Phase 4 Fix: Diff Approval & TOML Conversion

## Overview

Fix the diff approval system and TOML↔Markdown conversion to resolve all blocking issues from Phase C. Two distinct root causes require fixes:

1. **Issues 4.4 & 4.7**: `propose_edit()` stores entire file as `current_value` instead of snippets
2. **Issue 4.6**: `toml_to_markdown()` corrupts nested TOML tables during conversion

## Current State Analysis

### Root Cause 1: Full-File Diffs (Issues 4.4, 4.7)

`propose_edit()` at `storage/blocks.py:279` stores the entire file content:
```python
current = self.storage.read_block(block_label) or ""  # ENTIRE file
```

**Impact:**
- **4.4**: Frontend `buildDiffView()` uses `findIndex(line => line.trim() === oldValue.trim())` - can't match entire file to a single line
- **4.7**: `current_content.replace(entire_file, snippet)` replaces everything with just the snippet

### Root Cause 2: Nested Table Corruption (Issue 4.6)

`toml_to_markdown()` at `storage/convert.py:67` doesn't handle nested tables:
```python
else:
    lines.append(str(value))  # DICT becomes "{'key': 'value'}" string!
```

**Reproduced with actual data:**
```toml
# Original
[personal]
name = "Alex Chen"
grade = "12th"
```
→ Markdown: `## Personal\n{'name': 'Alex Chen', 'grade': '12th'}`
→ Round-trip TOML: `personal = "{'name': 'Alex Chen', 'grade': '12th'}"`

**The entire nested structure is flattened into a string!**

### Issue Summary

| Issue | Symptom | Root Cause | Confidence |
|-------|---------|------------|------------|
| 4.4 | Diff view not displaying | Full file stored as `current_value` | 95% |
| 4.5 | Shows TOML instead of MD | By design - diffs are TOML-based | 100% (not a bug) |
| 4.6 | Manual edits corrupt data | Nested tables → `str(dict)` → corruption | **100% (reproduced)** |
| 4.7 | Approve deletes block | `replace(entire_file, snippet)` | 95% |

## Desired End State

After these fixes:
1. Diffs store snippets for replace operations, empty for append
2. Frontend displays red/green diff highlighting correctly
3. Approve/reject modifies only targeted content
4. Manual markdown editing preserves nested TOML structure
5. Round-trip conversion is lossless for structure (types may still convert)

### How to Verify
- Create diff via agent → see green/red highlighting in UI
- Approve → only targeted line changes
- Edit block manually → nested structure preserved after save
- Type changes (bool→string) are acceptable; structure corruption is not

## What We're NOT Doing

- **Issue 4.5** - Diffs display as TOML by design. Converting to Markdown would be lossy.
- **Type preservation** - `true` → `"Yes"` → `"Yes"` is acceptable for MVP
- **LLM diff operation** - Future enhancement
- **Multi-line diff support** - Single line replacement sufficient for now

## Implementation Approach

Four phases:
1. **Phase 1**: Fix TOML→Markdown conversion for nested tables
2. **Phase 2**: Fix Markdown→TOML conversion for nested tables
3. **Phase 3**: Fix `propose_edit()` to store snippets
4. **Phase 4**: End-to-end verification

---

## Phase 1: Fix TOML→Markdown Conversion

### Overview
Update `toml_to_markdown()` to properly format nested TOML tables instead of using `str(dict)`.

### Changes Required:

#### 1.1 Update `toml_to_markdown()` for Nested Tables
**File**: `src/youlab_server/storage/convert.py`
**Changes**: Handle dict values by flattening nested keys with proper formatting

```python
def toml_to_markdown(toml_content: str, block_label: str) -> str:
    """
    Convert TOML block to Markdown for user editing.

    Format:
        ---
        block: student
        ---

        ## Personal

        ### Name
        Alex Chen

        ### Grade
        12th

        ## Simple Field
        Just a string value
    """
    try:
        data = tomllib.loads(toml_content)
    except tomllib.TOMLDecodeError:
        # If invalid TOML, return as-is in code block
        return (
            f"---\nblock: {block_label}\nerror: invalid_toml\n---\n\n```toml\n{toml_content}\n```"
        )

    lines = [
        "---",
        f"block: {block_label}",
        "---",
        "",
    ]

    def format_value(value: Any, indent: int = 0) -> list[str]:
        """Format a value for markdown output."""
        result = []
        if isinstance(value, dict):
            for k, v in value.items():
                sub_title = k.replace("_", " ").title()
                result.append(f"{'#' * (3 + indent)} {sub_title}")
                result.append("")
                result.extend(format_value(v, indent + 1))
        elif isinstance(value, list):
            result.extend(f"- {item}" for item in value)
        elif isinstance(value, bool):
            result.append("Yes" if value else "No")
        elif value is None:
            result.append("*(not set)*")
        elif isinstance(value, str):
            result.append(value)
        else:
            result.append(str(value))
        result.append("")
        return result

    # Convert each top-level field to a section
    for key, value in data.items():
        title = key.replace("_", " ").title()
        lines.append(f"## {title}")
        lines.append("")
        lines.extend(format_value(value))

    return "\n".join(lines)
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`

#### Manual Verification:
- [ ] `toml_to_markdown()` on nested TOML produces readable Markdown with ### subheadings
- [ ] No `{'key': 'value'}` strings in output

---

## Phase 2: Fix Markdown→TOML Conversion

### Overview
Update `markdown_to_toml()` to reconstruct nested tables from ### subheadings.

### Changes Required:

#### 2.1 Update `markdown_to_toml()` for Nested Structure
**File**: `src/youlab_server/storage/convert.py`
**Changes**: Parse ### headers as nested keys within ## sections

```python
def markdown_to_toml(markdown_content: str) -> tuple[str, dict[str, Any]]:
    """
    Convert Markdown back to TOML.

    Parses ## sections as top-level keys.
    Parses ### subsections as nested table keys.
    """
    lines = markdown_content.strip().split("\n")

    # Parse frontmatter
    metadata: dict[str, Any] = {}
    content_start = 0

    if lines and lines[0].strip() == "---":
        for i, line in enumerate(lines[1:], start=1):
            if line.strip() == "---":
                content_start = i + 1
                break
            if ":" in line:
                key, value = line.split(":", 1)
                metadata[key.strip()] = value.strip()

    # Parse sections
    data: dict[str, Any] = {}
    current_section: str | None = None
    current_subsection: str | None = None
    current_content: list[str] = []
    current_is_list = False

    def save_content():
        """Save accumulated content to the appropriate location."""
        nonlocal current_content, current_is_list
        if current_section is None:
            return

        value = _finalize_section(current_content, current_is_list)

        if current_subsection:
            # Nested under subsection
            if current_section not in data:
                data[current_section] = {}
            if not isinstance(data[current_section], dict):
                # Section was simple value, convert to dict
                data[current_section] = {"_value": data[current_section]}
            data[current_section][current_subsection] = value
        elif current_section in data and isinstance(data[current_section], dict):
            # Already have nested content, add to it
            pass  # Don't overwrite nested structure
        else:
            data[current_section] = value

        current_content = []
        current_is_list = False

    for line in lines[content_start:]:
        if line.startswith("## "):
            # Save previous section
            save_content()
            # Start new top-level section
            title = line[3:].strip()
            current_section = _title_to_key(title)
            current_subsection = None
            current_content = []
            current_is_list = False
        elif line.startswith("### "):
            # Save previous subsection content
            save_content()
            # Start new subsection
            title = line[4:].strip()
            current_subsection = _title_to_key(title)
            current_content = []
            current_is_list = False
        elif current_section is not None:
            stripped = line.strip()
            if stripped.startswith("- "):
                current_is_list = True
                current_content.append(stripped[2:])
            elif stripped and stripped != "*(not set)*":
                current_content.append(line)

    # Save last section
    save_content()

    toml_str = tomli_w.dumps(data)
    return toml_str, metadata
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Round-trip test: nested TOML → MD → TOML preserves structure

#### Manual Verification:
- [ ] Markdown with ### subsections converts to nested TOML tables
- [ ] Editing and saving a block with nested structure preserves nesting

---

## Phase 3: Fix Snippet-Based Diffs

### Overview
Modify `propose_edit()` to store snippets instead of entire files, and update the agent tool to accept `old_content`.

### Changes Required:

#### 3.1 Update `edit_memory_block` Tool Signature
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

#### 3.2 Update Tool Call to `propose_edit()`
**File**: `src/youlab_server/tools/memory.py`
**Changes**: Pass `old_content` to `propose_edit()`

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

#### 3.3 Update `propose_edit()` to Use `old_content`
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

    Args:
        old_content: For "replace" operations, the specific content being replaced.
                     If not provided, reads entire block (legacy behavior).
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
        current = ""

    diff = PendingDiff.create(
        user_id=self.user_id,
        agent_id=agent_id,
        block_label=block_label,
        field=field,
        operation=operation,
        current_value=current,
        proposed_value=proposed_value,
        reasoning=reasoning,
        confidence=confidence,
        source_query=source_query,
    )

    self.diffs.save(diff)
    return diff
```

#### 3.4 Update Validation in `approve_diff()`
**File**: `src/youlab_server/storage/blocks.py`
**Changes**: Improve error messages for replace validation

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
    new_content = current_content.replace(
        diff.current_value, diff.proposed_value, 1
    )
```

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `make test-agent`
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`

#### Manual Verification:
- [ ] Diff JSON shows `current_value` as snippet (not entire file)
- [ ] Frontend displays red/green diff highlighting
- [ ] Approve changes only targeted line

---

## Phase 4: End-to-End Verification

### Overview
Verify all fixes work together in the complete flow.

### Test Cases:

#### 4.1 Diff Display (Issue 4.4)
1. Create block with content: `name = "Alice"\nage = 25`
2. Create diff: `old_content="name = \"Alice\""`, `proposed_value="name = \"Bob\""`
3. Open block in UI
4. **Expected**: Red line `name = "Alice"`, green line `name = "Bob"`

#### 4.2 Approve Diff (Issue 4.7)
1. Click "Approve" on the diff above
2. **Expected**: Block content is `name = "Bob"\nage = 25` (age unchanged)

#### 4.3 Manual Edit Persistence (Issue 4.6)
1. Open a block with nested structure (e.g., `student.toml`)
2. Switch to Edit mode
3. Make a small change (add a word)
4. Click Save
5. Reload the page
6. **Expected**: Change persists AND nested structure is intact

#### 4.4 Round-Trip Test
```bash
uv run python3 -c "
from youlab_server.storage.convert import toml_to_markdown, markdown_to_toml
import tomllib

original = '''
[personal]
name = \"Alex\"
grade = \"12th\"

[academic]
gpa = 3.85
strengths = [\"a\", \"b\"]
'''

md = toml_to_markdown(original, 'test')
toml_back, _ = markdown_to_toml(md)
data_original = tomllib.loads(original)
data_back = tomllib.loads(toml_back)

# Structure should match (types may differ)
assert set(data_original.keys()) == set(data_back.keys())
assert set(data_original['personal'].keys()) == set(data_back['personal'].keys())
print('✓ Round-trip preserves structure')
"
```

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `make test-agent`
- [ ] Frontend builds: `cd OpenWebUI/open-webui && npm run build`
- [ ] Round-trip test passes

#### Manual Verification:
- [ ] Diff view shows red/green highlighting
- [ ] Approve modifies only targeted content
- [ ] Manual edits persist correctly
- [ ] Nested block structure preserved after edit

---

## Testing Strategy

### Unit Tests

```python
# tests/test_storage/test_convert.py

def test_toml_to_markdown_nested_tables():
    """Nested TOML tables convert to ### subsections."""
    toml = '[personal]\nname = "Alex"\nage = 18'
    md = toml_to_markdown(toml, "test")
    assert "## Personal" in md
    assert "### Name" in md
    assert "Alex" in md
    assert "{'name'" not in md  # No dict string!


def test_markdown_to_toml_nested_structure():
    """### subsections convert back to nested tables."""
    md = "---\nblock: test\n---\n\n## Personal\n\n### Name\nAlex\n\n### Age\n18"
    toml, _ = markdown_to_toml(md)
    data = tomllib.loads(toml)
    assert "personal" in data
    assert isinstance(data["personal"], dict)
    assert data["personal"]["name"] == "Alex"


def test_roundtrip_preserves_structure():
    """Round-trip conversion preserves nested structure."""
    original = '[personal]\nname = "Alex"\n\n[academic]\ngpa = 3.85'
    md = toml_to_markdown(original, "test")
    toml_back, _ = markdown_to_toml(md)

    data_orig = tomllib.loads(original)
    data_back = tomllib.loads(toml_back)

    assert set(data_orig.keys()) == set(data_back.keys())
    assert "name" in data_back.get("personal", {})
```

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
        old_content='name = "Alice"',
    )

    assert diff.current_value == 'name = "Alice"'
    assert diff.proposed_value == 'name = "Bob"'


def test_propose_edit_append_empty_current(self, manager, user_storage, sample_toml):
    """Append operation stores empty current_value."""
    user_storage.write_block("student", sample_toml, author="system")

    diff = manager.propose_edit(
        agent_id="agent1",
        block_label="student",
        field="notes",
        operation="append",
        proposed_value='notes = "New note"',
        reasoning="Adding notes",
    )

    assert diff.current_value == ""
```

---

## Migration Notes

### Existing Pending Diffs
Diffs created before this fix may have `current_value` = entire file. These will:
- **Display**: May not render correctly (no line match)
- **Approve**: May fail validation or cause data loss
- **Recommend**: Reject old diffs and let agent recreate

### Existing Block Content
Blocks already corrupted by the conversion bug will need manual repair or restoration from git history.

### Backwards Compatibility
- `old_content` parameter is optional - legacy calls work but may have display issues
- Frontend changes are additive (handle append diffs at end)

---

## Performance Considerations

- No significant performance impact
- Diff JSON files may be smaller (snippets vs full files)
- Conversion adds minimal overhead for nested tables

## References

- Research: `thoughts/shared/research/2026-01-12-ARI-81-phase-4-diff-approval-analysis.md`
- Parent plan: `thoughts/shared/plans/2026-01-12-ARI-81-phase-c-demo-mvp.md`
- Conversion code: `src/youlab_server/storage/convert.py`
- Diff storage: `src/youlab_server/storage/blocks.py:263-366`
- Frontend diff view: `OpenWebUI/open-webui/src/lib/components/you/BlockDetailModal.svelte:58-116`

---

*Plan created: 2026-01-12*
*Updated: 2026-01-12 - Added Issue 4.6 fix (nested table conversion)*
