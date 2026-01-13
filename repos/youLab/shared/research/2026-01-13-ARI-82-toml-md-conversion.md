---
date: 2026-01-13T17:12:41Z
researcher: Claude Code
git_commit: 7e5649b56d37976fb00dd567b1f1ee344cace890
branch: main
repository: YouLab
topic: "TOML to Markdown Conversion Robustness and Gap Analysis"
tags: [research, memory-system, toml, markdown, conversion, ARI-82]
status: complete
last_updated: 2026-01-13
last_updated_by: Claude Code
---

# Research: TOML ↔ Markdown Conversion Robustness and Gap Analysis

**Date**: 2026-01-13T17:12:41Z
**Researcher**: Claude Code
**Git Commit**: 7e5649b56d37976fb00dd567b1f1ee344cace890
**Branch**: main
**Repository**: YouLab
**Linear Ticket**: ARI-82

## Research Question

Research TOML ↔ Markdown conversion for memory blocks. Focus on:
1. Current converter implementation
2. Round-trip robustness (TOML → MD → TOML)
3. How structured fields map to markdown sections
4. Edge cases: lists, multi-line strings, nested data
5. Gap analysis: New spec requires memory blocks to default to freeform (single body field) unless schema specified

## Summary

The current converter in `src/youlab_server/storage/convert.py` is designed exclusively for **multi-field structured blocks**. It maps each TOML top-level key to a Markdown `## Section` header. The conversion is **not suitable for freeform content** because:

1. **Single-field blocks produce awkward output**: A block with only `body = "content"` becomes `## Body\n\ncontent...` instead of clean markdown
2. **Round-trip is lossy for some types**: Booleans become strings, numbers become strings, nested data is flattened
3. **No freeform mode exists**: Every TOML key gets a section header; there's no "passthrough" for pure content

**Key gap**: The new spec requires freeform (single body field) as the default, but current conversion assumes multi-field structure.

## Detailed Findings

### 1. Current Converter Implementation

**Location**: `src/youlab_server/storage/convert.py`

#### toml_to_markdown (lines 12-71)

Converts TOML to Markdown with frontmatter:

```markdown
---
block: student
---

## Profile
Background, CliftonStrengths, aspirations...

## Insights
Key observations from conversation...
```

**Logic**:
1. Parse TOML with `tomllib.loads()`
2. Add frontmatter with block label
3. For each top-level key:
   - Convert `snake_case` → `Title Case` for header
   - Add `## {Title}` section
   - Lists → bullet points (`- item`)
   - Booleans → "Yes"/"No"
   - None → `*(not set)*`
   - Everything else → `str(value)`

#### markdown_to_toml (lines 74-133)

Parses Markdown back to TOML:

```python
def markdown_to_toml(markdown_content: str) -> tuple[str, dict[str, Any]]:
```

**Logic**:
1. Extract frontmatter metadata between `---` delimiters
2. Parse `## Section` headers as keys
3. Detect bullet lists by `- ` prefix
4. Convert `Title Case` → `snake_case` for keys
5. Filter `*(not set)*` markers
6. Return `(toml_string, metadata_dict)`

### 2. Round-Trip Robustness

**Tested in**: `tests/test_storage/test_convert.py`

#### What Works (Lossless)

| Type | TOML Input | Markdown | Back to TOML |
|------|------------|----------|--------------|
| Strings | `name = "Alice"` | `## Name\n\nAlice` | `name = "Alice"` |
| Lists | `strengths = ["A", "B"]` | `## Strengths\n\n- A\n- B` | `strengths = ["A", "B"]` |
| Multi-line strings | `bio = "Line 1\n\nLine 2"` | Preserved in section | Preserved |
| Snake_case keys | `engagement_strategy` | `## Engagement Strategy` | `engagement_strategy` |

#### What's Lossy

| Type | TOML Input | Markdown | Back to TOML | Issue |
|------|------------|----------|--------------|-------|
| Booleans | `active = true` | `## Active\n\nYes` | `active = "Yes"` | String not bool |
| Integers | `age = 25` | `## Age\n\n25` | `age = "25"` | String not int |
| Floats | `score = 95.5` | `## Score\n\n95.5` | `score = "95.5"` | String not float |
| None | `field = None` | `*(not set)*` | `field = ""` | Empty string |
| Empty | `field = ""` | (omitted) | (missing) | Lost entirely |

### 3. Field → Section Mapping

The converter applies a **1:1 mapping** between TOML keys and Markdown sections:

```
TOML:                          Markdown:
profile = "..."                ## Profile
                       →       ...
insights = "..."
                               ## Insights
                               ...
```

**Key transformations**:
- `snake_case` → `Title Case` (forward)
- `Title Case` → `snake_case` (reverse)
- Uses `re.sub(r"\s+", "_", title.lower())` for normalization

### 4. Edge Cases

#### Lists

**Works well**:
```toml
# TOML
strengths = ["Creativity", "Communication", "Leadership"]
```
```markdown
## Strengths

- Creativity
- Communication
- Leadership
```

**Not supported**: Numbered lists, nested lists, mixed content lists

#### Multi-line Strings

**Works well** - preserved as paragraphs:
```toml
background = "First paragraph.\n\nSecond paragraph."
```
```markdown
## Background

First paragraph.

Second paragraph.
```

#### Nested Data

**NOT SUPPORTED** - only top-level keys are converted:
```toml
# This structure is NOT handled:
[metadata]
author = "system"
version = 3
```

The nested `[metadata]` table would be serialized as a string representation, not expanded.

#### Invalid TOML

Handled gracefully with error flag:
```markdown
---
block: student
error: invalid_toml
---

```toml
this is not valid toml {
```
```

### 5. Gap Analysis: Freeform vs Multi-Field

**Current State**: All code assumes multi-field structured blocks.

#### What the Spec Requires

Memory blocks should default to **freeform** (single body field) unless schema specified:

```toml
# Freeform block (default)
content = """
This is free-form markdown content.

## Custom Heading
- User can write any structure
- No enforced sections
"""

[metadata]
last_modified = "2026-01-13T10:00:00Z"
```

#### Current Behavior with Single Field

A single-field block would produce awkward output:

```toml
# Input TOML
content = "User's notes here..."
```

```markdown
# Current output (not desired)
---
block: notes
---

## Content

User's notes here...
```

**Desired output** (freeform):
```markdown
---
block: notes
---

User's notes here...
```

#### What's Missing

1. **Freeform detection**: No way to identify single-body blocks
2. **Passthrough mode**: No conversion that preserves raw content
3. **Schema awareness**: Converter doesn't know block schema
4. **Content field special-case**: No handling for `content` as body

### 6. Usage Patterns in Codebase

**Primary consumers**:
- `UserBlockManager.get_block_markdown()` - `storage/blocks.py:60-65`
- `UserBlockManager.update_block_from_markdown()` - `storage/blocks.py:67-98`
- `GET /users/{user_id}/blocks/{label}` - `server/blocks.py:132-152`
- `PUT /users/{user_id}/blocks/{label}` - `server/blocks.py:155-177`

**Data flow**:
```
Git Storage (TOML) → toml_to_markdown() → API → Frontend (Markdown)
Frontend (Markdown) → markdown_to_toml() → Git Storage (TOML) → Letta
```

**Never used for**:
- Agent-to-agent communication
- Letta internal memory (uses different `_toml_to_memory_string()`)
- Background processing

### 7. Test Coverage

**Unit tests**: `tests/test_storage/test_convert.py` (261 lines)
- Basic string conversion
- List conversion
- Multi-line strings
- Boolean conversion (documents lossiness)
- Snake_case transformation
- Invalid TOML handling
- Round-trip validation

**Integration tests**: `tests/test_storage/test_blocks.py`
- `get_block_markdown()` flow
- `update_block_from_markdown()` flow

**Not tested**:
- Nested TOML tables
- TOML dates/datetimes
- Unicode/emoji in keys
- Empty TOML input
- Blocks with `## ` in content (potential parsing conflict)

## Code References

| File | Lines | Description |
|------|-------|-------------|
| `src/youlab_server/storage/convert.py` | 12-71 | `toml_to_markdown()` function |
| `src/youlab_server/storage/convert.py` | 74-133 | `markdown_to_toml()` function |
| `src/youlab_server/storage/convert.py` | 136-138 | `_title_to_key()` helper |
| `src/youlab_server/storage/convert.py` | 141-147 | `_finalize_section()` helper |
| `src/youlab_server/storage/blocks.py` | 60-65 | `get_block_markdown()` usage |
| `src/youlab_server/storage/blocks.py` | 67-98 | `update_block_from_markdown()` usage |
| `src/youlab_server/storage/blocks.py` | 213-222 | `_toml_to_memory_string()` (Letta format) |
| `tests/test_storage/test_convert.py` | 1-261 | Converter test suite |

## Architecture Documentation

### Current Conversion Flow

```
Multi-field TOML           User-friendly Markdown
┌─────────────────┐       ┌──────────────────────┐
│ name = "Alice"  │       │ ---                  │
│ role = "Student"│  ───► │ block: student       │
│ strengths = [   │       │ ---                  │
│   "Creative",   │       │                      │
│   "Leader"      │       │ ## Name              │
│ ]               │       │ Alice                │
└─────────────────┘       │                      │
                          │ ## Role              │
                          │ Student              │
                          │                      │
                          │ ## Strengths         │
                          │ - Creative           │
                          │ - Leader             │
                          └──────────────────────┘
```

### Proposed Freeform Flow (Not Yet Implemented)

```
Freeform TOML              Raw Markdown Display
┌─────────────────┐       ┌──────────────────────┐
│ content = """   │       │ ---                  │
│ User's notes... │  ───► │ block: notes         │
│ """             │       │ ---                  │
│                 │       │                      │
│ [metadata]      │       │ User's notes...      │
│ version = 3     │       │                      │
└─────────────────┘       └──────────────────────┘
```

## Historical Context

**Prior research**: `thoughts/shared/research/2026-01-12-ARI-78-toml-markdown-roundtrip.md`
- Documented the spec's intent for simple `content` field pattern
- Noted that current system evolved from complex field schemas
- Proposed `content = """..."""` as the canonical pattern for freeform

**Current implementation history**:
- Legacy `PersonaBlock`/`HumanBlock` had hardcoded fields
- TOML-based `BlockSchema` system replaced them
- Converter designed for this multi-field paradigm
- No updates for freeform use case

## Gap Summary

| Aspect | Current State | Required for Freeform |
|--------|---------------|----------------------|
| Detection | No freeform detection | Detect single `content` field |
| Conversion | All keys → sections | Content passthrough |
| Schema | Not consulted | Check if schema defines fields |
| Metadata | In frontmatter | Separate from content |
| Round-trip | Multi-field only | Both modes |

## Open Questions

1. **How should freeform blocks be identified?**
   - By schema (no fields defined)?
   - By convention (only `content` key)?
   - By explicit flag in schema?

2. **What about hybrid blocks?**
   - Some structured fields + freeform body?
   - Example: `content` for body, `status` for state

3. **Metadata handling in freeform mode?**
   - Currently `[metadata]` would become a section
   - Should metadata be in frontmatter only?

4. **Backward compatibility?**
   - Existing multi-field blocks must continue working
   - How to migrate without breaking?

## Related Research

- `thoughts/shared/research/2026-01-12-ARI-78-toml-markdown-roundtrip.md` - Initial spec analysis
