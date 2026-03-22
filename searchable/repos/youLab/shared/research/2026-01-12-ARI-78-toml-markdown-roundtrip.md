---
date: 2026-01-12T03:32:51Z
researcher: Claude Code
git_commit: 4c099b8eefca5a1c178847faf034af8047936438
branch: chore/coderabbit-setup
repository: YouLab
topic: "TOML to Markdown Round-Trip for Memory Blocks"
tags: [research, memory-system, toml, markdown, serialization, ARI-78]
status: complete
last_updated: 2026-01-12
last_updated_by: Claude Code
---

# Research: TOML to Markdown Round-Trip for Memory Blocks

**Date**: 2026-01-12T03:32:51Z
**Researcher**: Claude Code
**Git Commit**: 4c099b8eefca5a1c178847faf034af8047936438
**Branch**: chore/coderabbit-setup
**Repository**: YouLab
**Linear Ticket**: ARI-78

## Research Question

The product spec says "users edit markdown, it compiles to TOML". Investigate:
1. Current TOML handling in YouLab
2. How memory blocks are currently serialized/deserialized
3. Simplest approach for storing markdown content in TOML
4. Existing TOML↔display conversion patterns
5. Should we use raw markdown in a content field or something more structured?

## Summary

The product spec describes a **new pattern** that differs from the current system. The current codebase uses TOML for agent configuration with complex field schemas, while the spec calls for simple markdown content storage with metadata. The simplest approach is to store markdown directly in a `content` field using TOML's triple-quoted multiline strings, with minimal metadata fields alongside it.

**Key insight**: The spec's memory system is fundamentally simpler than the current curriculum/block system. It's just "store markdown, add metadata, version control."

## Detailed Findings

### 1. Current TOML Handling

**Parser**: Python's built-in `tomllib` module (`curriculum/loader.py:137-138`)

**Multiline String Pattern**: Triple-quoted strings (`"""..."""`) are used extensively:

```toml
system = """You are YouLab Essay Coach...

## YOUR PHILOSOPHY
- Never write essays for students
- Go deep rather than wide
"""
```

**File Structure**:
- `config/courses/{course}/course.toml` - Course configuration
- `config/courses/{course}/modules/*.toml` - Module definitions

**Two Schema Versions**:
- v1: `[blocks.name.fields]` nested sections
- v2: `[block.name]` with `field.x = {...}` dotted keys

### 2. Current Memory Block Serialization

The current system is **not designed for user-facing memory**. It serializes blocks to a YAML-like format for LLM context:

**DynamicBlock.to_memory_string()** (`curriculum/blocks.py:51-70`):
```
field_name: value
field_name:
- item1
- item2
```

**DynamicBlock.from_memory_string()** (`curriculum/blocks.py:72-116`):
- State machine parser for the above format
- Best-effort parsing, missing fields use defaults

**Key Classes**:
- `BlockSchema` (`curriculum/schema.py:208-215`): Configuration schema with label, description, shared flag, and field definitions
- `FieldSchema` (`curriculum/schema.py:197-206`): Individual field with type, default, options, max, description
- `DynamicBlock` (`curriculum/blocks.py:48-117`): Runtime Pydantic models generated from schemas
- `create_block_model()` (`curriculum/blocks.py:119-168`): Generates Pydantic classes dynamically

**Current flow**: TOML → BlockSchema → DynamicBlock class → instance → memory string → Letta

### 3. What the Spec Actually Needs

The spec describes a **different pattern** (`thoughts/shared/plans/2025-01-12-youlab-product-spec.md:317-333`):

```toml
# memory/operating-manual.toml
content = """
This student responds best to direct feedback...
"""

last_modified = "2025-01-12T10:30:00Z"
modified_by = "insight-synthesizer"
```

**Display**: TOML content field rendered as markdown
**Edit**: User edits markdown, saved back to content field
**Diff**: Show changes to content field in diff format

This is **simpler** than the current BlockSchema system because:
1. No complex field types (just string content)
2. No field schemas or validation
3. Pure markdown round-trip
4. Metadata is separate from content

### 4. Simplest Approach for Markdown in TOML

**Recommended pattern** (matches spec):

```toml
# memory/operating-manual.toml

content = """
This student responds best to direct feedback and specific examples.

## Key Behaviors
- Prefers concrete over abstract
- Likes to understand "why" before "how"
- Works best with structured guidance
"""

[metadata]
last_modified = "2025-01-12T10:30:00Z"
modified_by = "insight-synthesizer"
version = 3
```

**Why this works**:
1. **No escaping needed**: TOML multiline strings preserve markdown formatting perfectly
2. **Clean separation**: Content vs metadata in separate sections
3. **Git-friendly**: TOML diffs cleanly for version control
4. **Simple parsing**: `toml.load()` → `data["content"]` → display as markdown
5. **Simple saving**: `data["content"] = markdown_text` → `toml.dump()`

### 5. Existing Conversion Patterns (None for User Display)

The current system has **no markdown rendering or display conversion**. Block content is:
- Serialized for LLM context (never displayed to users)
- Stored internally in Letta agent memory
- Never rendered to HTML or rich markdown

**What exists**:
- `to_memory_string()` / `from_memory_string()` for YAML-like format
- No Jinja, mustache, or template libraries
- No markdown processing libraries

### 6. Recommendation: Raw Markdown in Content Field

**Use raw markdown** rather than structured fields because:

1. **Spec alignment**: The spec explicitly shows `content = """..."""` pattern
2. **User editing**: Users edit as markdown, not structured forms
3. **Flexibility**: Markdown can contain any formatting, lists, headers
4. **Simplicity**: No schema validation needed, just content + metadata
5. **Diff-friendly**: Full-text diffs are meaningful for markdown
6. **Existing patterns**: System prompts already use this exact approach

**When to use structured fields instead**:
- Journey tracking (module_id, lesson_id, status) - needs validation
- Discrete options (tone: warm/professional/friendly) - needs constraints
- Numeric values - needs type checking

For user memory blocks (Operating Manual, Values, Personality), **raw markdown is ideal**.

## Code References

| File | Lines | Description |
|------|-------|-------------|
| `src/youlab_server/curriculum/schema.py` | 197-215 | BlockSchema and FieldSchema definitions |
| `src/youlab_server/curriculum/blocks.py` | 48-117 | DynamicBlock base class |
| `src/youlab_server/curriculum/blocks.py` | 119-168 | create_block_model() function |
| `src/youlab_server/curriculum/loader.py` | 136-138 | TOML loading via tomllib |
| `src/youlab_server/curriculum/loader.py` | 294-323 | v2 block schema parsing |
| `config/courses/college-essay/course.toml` | 15-72 | Example multiline TOML strings |

## Architecture Documentation

### Current System (Curriculum/Agent Blocks)
```
TOML Config → BlockSchema → DynamicBlock class → instance → memory string → Letta
```

### New System (User Memory - Per Spec)
```
TOML File → content field → Markdown Display
           ↑
Markdown Edit → content field → TOML File → Git Commit
```

### Proposed Implementation

```python
import tomllib
import tomli_w  # For writing TOML
from pathlib import Path
from datetime import datetime

def load_memory_block(path: Path) -> dict:
    """Load memory block from TOML file."""
    with path.open("rb") as f:
        return tomllib.load(f)

def display_block(path: Path) -> str:
    """Get markdown content for display."""
    data = load_memory_block(path)
    return data.get("content", "")

def save_block(path: Path, markdown: str, modified_by: str) -> None:
    """Save edited markdown back to TOML file."""
    # Load existing to preserve structure
    if path.exists():
        data = load_memory_block(path)
    else:
        data = {}

    data["content"] = markdown
    data.setdefault("metadata", {})
    data["metadata"]["last_modified"] = datetime.utcnow().isoformat() + "Z"
    data["metadata"]["modified_by"] = modified_by
    data["metadata"]["version"] = data["metadata"].get("version", 0) + 1

    with path.open("wb") as f:
        tomli_w.dump(data, f)
```

## Historical Context

The current system evolved from:
1. Legacy hardcoded `PersonaBlock`/`HumanBlock` classes (`memory/blocks.py`)
2. TOML-based `BlockSchema` system for agent configuration
3. Dynamic Pydantic model generation for runtime validation

The spec's user memory system is **intentionally simpler** - it's about storing free-form insights about the user, not structured agent configuration.

## Open Questions

1. **Schema definition**: Where is the default schema (Operating Manual, Values, etc.) stored?
   - Current answer: `config/schema.toml` per spec, but doesn't exist yet

2. **Block inheritance**: Can users customize the schema? Add new blocks?
   - Spec leaves this as "open question"

3. **Diff format**: How are markdown diffs rendered in the approval UI?
   - Could use unified diff format with syntax highlighting

4. **Multi-agent writes**: If multiple agents propose changes, how are conflicts resolved?
   - Spec leaves this as "open question"

5. **tomli_w dependency**: Need to add for TOML writing (tomllib is read-only)
