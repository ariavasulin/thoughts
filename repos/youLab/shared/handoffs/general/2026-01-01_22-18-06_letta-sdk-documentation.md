---
date: 2026-01-01T22:18:06-08:00
researcher: ariasulin
git_commit: 70314e15449c8c3306b1da483a13e094c5d7f4f2
branch: main
repository: YouLab
topic: "Letta SDK & API Comprehensive Documentation"
tags: [documentation, letta, sdk, api, reference]
status: complete
last_updated: 2026-01-01
last_updated_by: ariasulin
type: documentation
---

# Handoff: Letta SDK & API Documentation

## Task(s)

| Task | Status |
|------|--------|
| Research Letta SDK via parallel agent swarm | Completed |
| Create comprehensive research document | Completed |
| Break research into `/docs` documentation | Completed |
| Update sidebar navigation | Completed |

**Summary**: Created comprehensive Letta SDK/API documentation as an extension of the existing `/docs` directory. This involved spawning 13 parallel research agents to gather up-to-date information about Letta (since it's beyond the knowledge cutoff), then synthesizing findings into structured documentation.

## Critical References

- `thoughts/shared/research/2026-01-01-letta-sdk-api-reference.md` - Master research document (source of truth)
- `docs/_sidebar.md` - Updated navigation with new Letta Reference section

## Recent changes

Created 6 new documentation files in `/docs`:
- `docs/Letta-Concepts.md:1` - Core architecture, agent principles, memory tiers
- `docs/Letta-REST-API.md:1` - Complete endpoint reference
- `docs/Letta-Streaming.md:1` - Message types, SSE format, chunk handling
- `docs/Letta-Archival.md:1` - Vector search, passages, tags, bulk loading
- `docs/Letta-Tools.md:1` - Custom tools, schemas, execution, tool rules
- `docs/Letta-Versions.md:1` - Package versions, breaking changes, migration

Updated navigation:
- `docs/_sidebar.md:22-29` - Added "Letta Reference" section with 7 pages

## Learnings

**Version Clarification**:
- PyPI `letta` package is at **0.10.0** (not 0.16.x from GitHub releases)
- PyPI `letta-client` SDK is at **1.6.2**
- GitHub releases use different versioning than PyPI packages
- Project dependency `letta>=0.6.0` is just a floor; actual installed is latest

**SDK Package Names**:
- Modern SDK: `from letta_client import Letta` (recommended)
- Legacy: `from letta import create_client` (still in codebase)
- Both are installed: `letta` (server+legacy client) and `letta-client` (modern SDK)

**Agent Architecture Evolution**:
- Current: Create agents without `agent_type` parameter
- Legacy `memgpt_agent`, `letta_v1_agent` are deprecated
- Heartbeat system removed in v0.12

**Codebase Integration Points**:
- `src/letta_starter/server/agents.py:28` - Letta client initialization
- `src/letta_starter/server/agents.py:108-117` - Agent creation with memory blocks
- `src/letta_starter/server/agents.py:186-192` - Streaming message API

## Artifacts

Research document:
- `thoughts/shared/research/2026-01-01-letta-sdk-api-reference.md`

Documentation files created:
- `docs/Letta-Concepts.md`
- `docs/Letta-REST-API.md`
- `docs/Letta-Streaming.md`
- `docs/Letta-Archival.md`
- `docs/Letta-Tools.md`
- `docs/Letta-Versions.md`
- `docs/_sidebar.md` (updated)

## Action Items & Next Steps

1. **Consider updating version floor**: Change `letta>=0.6.0` to `letta>=0.10.0` in `pyproject.toml` to reflect actual requirements

2. **Review existing Letta-SDK.md**: The pre-existing `docs/Letta-SDK.md` has some outdated patterns (e.g., `from letta import Letta`). May want to update to match new docs

3. **Test documentation site**: Run docsify locally to verify new pages render correctly:
   ```bash
   cd docs && docsify serve
   ```

4. **Consider embedding model upgrade**: Currently using `text-embedding-ada-002`; docs note `text-embedding-3-small` is recommended

## Other Notes

**Documentation Style**: All new docs follow existing patterns:
- `[[Page|Link Text]]` for internal links
- `---` for section separators
- "Related Pages" sections at bottom
- Docsify-compatible markdown

**Existing Letta Reference**: There's also a global reference document at:
- `thoughts/global/shared/reference/letta-archival-memory.md` - Detailed archival memory guide

**Research Process**: Used 13 parallel sub-agents:
- 4 codebase-analyzer agents for current usage
- 8 web-search-researcher agents for up-to-date Letta docs
- 1 thoughts-locator agent for historical context

The research document in thoughts/ is more comprehensive than the docs/ pages - it includes file:line references and implementation details specific to this codebase.
