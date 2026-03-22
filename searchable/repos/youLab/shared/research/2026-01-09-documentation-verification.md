---
date: 2026-01-09T09:42:58+07:00
researcher: ariasulin
git_commit: 3d8dbffc4b38f61f2e099982610a355a7360926a
branch: main
repository: YouLab
topic: "Documentation Accuracy Verification"
tags: [research, documentation, verification, docs]
status: complete
last_updated: 2026-01-09
last_updated_by: ariasulin
---

# Research: Documentation Accuracy Verification

**Date**: 2026-01-09T09:42:58+07:00
**Researcher**: ariasulin
**Git Commit**: 3d8dbffc4b38f61f2e099982610a355a7360926a
**Branch**: main
**Repository**: YouLab

## Research Question

Analyze the documentation structure in /docs and CLAUDE.md, compare it directly to the codebase to identify any areas for improvement. Prioritize finding incorrect information, but areas of clarification or confusion are also important.

## Summary

The documentation is largely accurate with **4 critical issues**, **5 moderate issues**, and several minor improvements needed. The most significant problem is that **docs/README.md shows Phases 3-6 as "Not Started" when they are fully implemented**. Additionally, there's a **schema duplication issue** between `background/schema.py` and `curriculum/schema.py` that causes incompatibility.

## Critical Issues

### 1. README.md Phase Status Completely Outdated

**Location**: `docs/README.md:36-44`

The "Current Status" table is severely out of date:

| Phase | README Says | Actual Status |
|-------|-------------|---------------|
| Phase 3: Honcho | Not Started | **Complete** |
| Phase 4: Thread Context | Not Started | **Complete** |
| Phase 5: Curriculum | Not Started | **Complete** |
| Phase 6: Background Worker | Not Started | **Complete** |
| Phase 7: Onboarding | Not Started | Not Started (correct) |

**Evidence**: All implementations exist and are integrated:
- HonchoClient: `src/letta_starter/honcho/client.py`
- Chat title extraction: `src/letta_starter/pipelines/letta_pipe.py:57-73`
- Curriculum system: `src/letta_starter/curriculum/` (complete package)
- Background agents: `src/letta_starter/background/runner.py`

### 2. Two Incompatible Schema Systems for Background Agents

**Critical**: The `background/schema.py` and `curriculum/schema.py` define **incompatible schemas** for the same concepts.

**OLD Schema** (`background/schema.py`):
```toml
id = "college-essay"
[[background_agents]]
id = "insight-harvester"
name = "Student Insight Harvester"  # Required field
```

**NEW Schema** (`curriculum/schema.py`):
```toml
[course]
id = "college-essay"
[background.insight-harvester]
enabled = true
# No 'name' field
```

**Impact**:
- The actual TOML file (`config/courses/college-essay/course.toml`) uses the NEW schema
- The server's background loader (`background/schema.py:102-118`) expects flat `*.toml` files, not subdirectories
- The server will load **ZERO** background agents because the loader can't find the config

### 3. Architecture.md Missing Curriculum Directory

**Location**: `docs/Architecture.md:218-269`

The project structure diagram is missing the `curriculum/` directory entirely:

**Missing**:
```
src/letta_starter/
├── curriculum/          # ← NOT DOCUMENTED
│   ├── __init__.py
│   ├── schema.py
│   ├── loader.py
│   └── blocks.py
```

### 4. Architecture.md Wrong Config Path

**Location**: `docs/Architecture.md:268`

**Documented**:
```
config/
└── courses/
    └── college-essay.toml
```

**Actual**:
```
config/
└── courses/
    └── college-essay/
        ├── course.toml
        └── modules/
            ├── 01-self-discovery.toml
            ├── 02-topic-development.toml
            └── 03-drafting.toml
```

## Moderate Issues

### 5. HTTP-Service.md Missing Curriculum Endpoints

**Location**: `docs/HTTP-Service.md`

Five curriculum endpoints are not documented:
- `GET /curriculum/courses` - List all courses
- `GET /curriculum/courses/{course_id}` - Get course details
- `GET /curriculum/courses/{course_id}/full` - Get full config as JSON
- `GET /curriculum/courses/{course_id}/modules` - Get modules with lessons
- `POST /curriculum/reload` - Hot-reload configurations

**Implementation**: `src/letta_starter/server/curriculum.py:75-173`

### 6. HTTP-Service.md Missing AgentManager Method

**Undocumented**: `create_agent_from_curriculum()` method at `src/letta_starter/server/agents.py:130-245`

This is a significant alternative agent creation path that:
- Creates agents based on curriculum configuration instead of templates
- Loads dynamic memory blocks from course TOML files
- Configures tools and tool rules from curriculum

### 7. Honcho.md Wrong Line Number Reference

**Location**: `docs/Honcho.md:180`

**Documented**: Fire-and-forget pattern at lines 220-272
**Actual**: `create_persist_task()` function is at lines **310-363**

### 8. config-schema.md Missing SessionScope.SPECIFIC

**Location**: `docs/config-schema.md:177`

**Documented**: session_scope enum has "all", "recent", "current"
**Actual**: `background/schema.py:21` also has "SPECIFIC"

Note: There's an inconsistency - `curriculum/schema.py:35-40` does NOT include SPECIFIC.

### 9. config-schema.md Missing ModuleConfig.background Field

**Location**: `docs/config-schema.md:278-297`

**Undocumented**: `ModuleConfig` has a `background` field for module-level background agent overrides:
```python
class ModuleConfig(BaseModel):
    # ... other fields ...
    background: dict[str, dict[str, Any]] = Field(default_factory=dict)
```

## Minor Issues

### 10. Agent-System.md Missing tools Field

**Location**: `docs/Agent-System.md`

`AgentTemplate` class has a `tools` field not documented:
```python
class AgentTemplate(BaseModel):
    # ... documented fields ...
    tools: list[Callable[..., str]] = Field(default_factory=list)  # ← Missing
```

`TUTOR_TEMPLATE` includes `tools=[query_honcho, edit_memory_block]`

### 11. Agent-System.md Missing get_all() Method

`AgentTemplateRegistry` has an undocumented `get_all()` method (lines 80-82).

### 12. Agent-Tools.md Missing set_letta_client() Helper

**Location**: `src/letta_starter/tools/memory.py:30`

The `set_letta_client()` helper function follows the same pattern as the documented `set_honcho_client()` but is not mentioned.

### 13. Architecture.md Missing Server Files

**Missing from `server/` section**:
- `server/cli.py` - CLI utilities
- `server/curriculum.py` - Curriculum HTTP endpoints

**Missing from `server/strategy/` expansion**:
- `strategy/__init__.py`
- `strategy/manager.py`
- `strategy/router.py`
- `strategy/schemas.py`

## Verified As Accurate

The following documentation sections were verified as completely accurate:

- **CLAUDE.md**: All 24 key files exist with accurate descriptions
- **Memory-System.md**: All line numbers, class definitions, helper methods, session states, and limits are accurate
- **Agent-Tools.md**: All parameters, session scopes, protected fields, merge strategies, and available fields match
- **Agent-System.md**: 95% accurate (minor omissions noted above)
- **AgentRegistry line numbers**: Exactly correct (160-261)

## Code References

- `docs/README.md:36-44` - Phase status table (outdated)
- `docs/Architecture.md:218-269` - Project structure (incomplete)
- `docs/HTTP-Service.md` - Missing curriculum endpoints
- `src/letta_starter/background/schema.py:102-118` - Old loader expecting flat files
- `src/letta_starter/curriculum/loader.py:43-107` - New loader expecting subdirectories
- `src/letta_starter/server/curriculum.py:75-173` - Undocumented endpoints
- `src/letta_starter/honcho/client.py:310-363` - Fire-and-forget (wrong line ref in docs)

## Recommendations

### Priority 1: Critical Fixes

1. **Update README.md phase status** - Mark Phases 3-6 as Complete
2. **Resolve schema duplication** - Either:
   - Consolidate to one schema file, or
   - Fix the background loader to use curriculum schema, or
   - Document that two systems exist for different purposes

### Priority 2: Moderate Fixes

3. **Add curriculum endpoints to HTTP-Service.md**
4. **Update Architecture.md project structure** to include:
   - `curriculum/` directory
   - Correct `config/courses/` structure
   - Additional server files
5. **Fix Honcho.md line number reference** (220-272 → 310-363)

### Priority 3: Minor Fixes

6. **Add missing fields/methods** to Agent-System.md (tools, get_all)
7. **Add set_letta_client()** to Agent-Tools.md
8. **Document SessionScope.SPECIFIC** or remove from code
9. **Document ModuleConfig.background** field

## Open Questions

1. **Schema consolidation**: Should `background/schema.py` be deprecated in favor of `curriculum/schema.py`?
2. **Roadmap accuracy**: Should docs/Roadmap.md be updated to show Phase 5-6 as Complete instead of Planned?
3. **Background loader**: Is the background system currently broken due to the schema mismatch, or is there a migration path?
