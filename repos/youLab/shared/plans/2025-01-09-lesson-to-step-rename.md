# Lesson → Step Rename Implementation Plan

## Overview

Rename the "Lesson" primitive to "Step" throughout the curriculum system. Steps better reflect the incremental, guided nature of the coaching workflow rather than implying discrete teaching units.

## Current State Analysis

The "Lesson" concept appears in:

| Location | Current | New |
|----------|---------|-----|
| `schema.py` models | `LessonConfig`, `LessonCompletion`, `LessonAgent` | `StepConfig`, `StepCompletion`, `StepAgent` |
| `schema.py` container | `ModuleConfig.lessons` | `ModuleConfig.steps` |
| `loader.py` imports | `LessonConfig`, etc. | `StepConfig`, etc. |
| `loader.py` parsing | `data.get("lessons", [])` | `data.get("steps", [])` |
| `__init__.py` exports | `LessonAgent`, `LessonCompletion`, `LessonConfig` | `StepAgent`, `StepCompletion`, `StepConfig` |
| `server/curriculum.py` | `ModuleSummary.lesson_count` | `ModuleSummary.step_count` |
| TOML module files | `[[lessons]]`, `[lessons.completion]`, `[lessons.agent]` | `[[steps]]`, `[steps.completion]`, `[steps.agent]` |
| `docs/config-schema.md` | "Lesson" references | "Step" references |

## Desired End State

After implementation:
- All code uses `Step*` naming (StepConfig, StepCompletion, StepAgent)
- All TOML files use `[[steps]]` syntax
- API returns `step_count` instead of `lesson_count`
- Documentation reflects the new terminology
- All tests pass
- No backwards compatibility layer (clean break)

### Verification Commands
```bash
# No "Lesson" references in code (except comments explaining the change)
grep -r "Lesson" src/letta_starter/curriculum/ --include="*.py" | grep -v "# "

# No "lessons" in TOML files
grep -r "\[\[lessons\]\]" config/courses/

# All tests pass
make test-agent
```

## What We're NOT Doing

- No backwards compatibility for `[[lessons]]` TOML syntax
- No deprecation warnings (clean break since early development)
- No API versioning (internal API)

## Implementation Approach

This is a straightforward rename across multiple files. We'll proceed layer by layer: schema → loader → exports → API → TOML → docs.

---

## Phase 1: Rename Pydantic Models

### Overview
Rename the three Lesson-related models in schema.py.

### Changes Required:

#### 1. Schema Models
**File**: `src/letta_starter/curriculum/schema.py`
**Changes**: Rename classes and update type references

```python
# Line 200: LessonCompletion → StepCompletion
class StepCompletion(BaseModel):
    """Completion criteria for a step."""
    # ... (fields unchanged)

# Line 209: LessonAgent → StepAgent
class StepAgent(BaseModel):
    """Step-specific agent configuration."""
    # ... (fields unchanged)

# Line 218: LessonConfig → StepConfig
class StepConfig(BaseModel):
    """Configuration for a single step."""
    # ... fields unchanged except:
    completion: StepCompletion = Field(default_factory=StepCompletion)
    agent: StepAgent = Field(default_factory=StepAgent)

# Line 237: ModuleConfig.lessons → ModuleConfig.steps
class ModuleConfig(BaseModel):
    # ...
    steps: list[StepConfig] = Field(default_factory=list)
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`

#### Manual Verification:
- [ ] None required for this phase

---

## Phase 2: Update Loader

### Overview
Update the loader to import new model names and parse `[[steps]]` from TOML.

### Changes Required:

#### 1. Imports
**File**: `src/letta_starter/curriculum/loader.py`
**Changes**: Update import names (lines 25-27)

```python
from letta_starter.curriculum.schema import (
    # ...
    StepAgent,      # was LessonAgent
    StepCompletion, # was LessonCompletion
    StepConfig,     # was LessonConfig
    # ...
)
```

#### 2. Module Loading Logic
**File**: `src/letta_starter/curriculum/loader.py`
**Changes**: Update `_load_modules` method (lines 410-443)

```python
def _load_modules(self, course_dir: Path, module_names: list[str]) -> list[ModuleConfig]:
    """Load module configurations."""
    modules = []
    modules_dir = course_dir / "modules"

    for module_name in module_names:
        module_file = modules_dir / f"{module_name}.toml"
        if not module_file.exists():
            log.warning("module_not_found", module=module_name)
            continue

        data = self._load_toml(module_file)
        module_data = data.get("module", {})

        steps = []  # was: lessons = []
        for step_data in data.get("steps", []):  # was: data.get("lessons", [])
            completion_data = step_data.get("completion", {})
            agent_data = step_data.get("agent", {})

            steps.append(
                StepConfig(  # was: LessonConfig
                    id=step_data.get("id", ""),
                    name=step_data.get("name", ""),
                    order=step_data.get("order", 0),
                    description=step_data.get("description", ""),
                    objectives=step_data.get("objectives", []),
                    completion=StepCompletion(  # was: LessonCompletion
                        required_fields=completion_data.get("required_fields", []),
                        min_turns=completion_data.get("min_turns"),
                        min_list_length=completion_data.get("min_list_length", {}),
                        auto_advance=completion_data.get("auto_advance", False),
                    ),
                    agent=StepAgent(  # was: LessonAgent
                        opening=agent_data.get("opening"),
                        focus=agent_data.get("focus", []),
                        guidance=agent_data.get("guidance", []),
                        persona_overrides=agent_data.get("persona_overrides", {}),
                    ),
                )
            )

        modules.append(
            ModuleConfig(
                id=module_data.get("id", module_name),
                name=module_data.get("name", module_name),
                order=module_data.get("order", 0),
                description=module_data.get("description", ""),
                steps=steps,  # was: lessons=lessons
            )
        )

    return modules
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`

---

## Phase 3: Update Package Exports

### Overview
Update the curriculum package `__init__.py` to export new names.

### Changes Required:

**File**: `src/letta_starter/curriculum/__init__.py`
**Changes**: Update imports (lines 39-41) and `__all__` list (lines 119-121)

```python
# Imports
from letta_starter.curriculum.schema import (
    # ...
    StepAgent,      # was LessonAgent
    StepCompletion, # was LessonCompletion
    StepConfig,     # was LessonConfig
    # ...
)

# __all__ list
__all__ = [
    # ...
    "StepAgent",      # was "LessonAgent"
    "StepCompletion", # was "LessonCompletion"
    "StepConfig",     # was "LessonConfig"
    # ...
]
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`

---

## Phase 4: Update HTTP API

### Overview
Update the curriculum HTTP endpoint response models.

### Changes Required:

**File**: `src/letta_starter/server/curriculum.py`
**Changes**:

1. Rename `ModuleSummary.lesson_count` to `step_count` (line 36)
2. Update usage in `get_course` endpoint (line 94)

```python
class ModuleSummary(BaseModel):
    """Summary of a module."""
    id: str
    name: str
    order: int
    step_count: int  # was: lesson_count

# In get_course endpoint:
modules = [
    ModuleSummary(
        id=m.id,
        name=m.name,
        order=m.order,
        step_count=len(m.steps),  # was: lesson_count=len(m.lessons)
    )
    for m in course.loaded_modules
]
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `make check-agent`
- [ ] Linting passes: `make lint-fix`

---

## Phase 5: Update TOML Module Files

### Overview
Update all module TOML files to use `[[steps]]` syntax.

### Changes Required:

#### 1. Module 01: Self-Discovery
**File**: `config/courses/college-essay/modules/01-self-discovery.toml`
**Changes**: Replace all `[[lessons]]` with `[[steps]]`, `[lessons.completion]` with `[steps.completion]`, `[lessons.agent]` with `[steps.agent]`

#### 2. Module 02: Topic Development
**File**: `config/courses/college-essay/modules/02-topic-development.toml`
**Changes**: Same pattern

#### 3. Module 03: Drafting
**File**: `config/courses/college-essay/modules/03-drafting.toml`
**Changes**: Same pattern

### Success Criteria:

#### Automated Verification:
- [ ] No `[[lessons]]` in TOML files: `grep -r "\[\[lessons\]\]" config/courses/`
- [ ] Course loads successfully: Test via HTTP endpoint or unit test

---

## Phase 6: Update Documentation

### Overview
Update the config schema documentation.

### Changes Required:

**File**: `docs/config-schema.md`
**Changes**:
- Line 249: "Module files define the curriculum structure with lessons." → "...with steps."
- Line 261: `### [[lessons]] - Lesson Configuration` → `### [[steps]] - Step Configuration`
- All field descriptions mentioning "lesson" → "step"
- Line 270: `### [lessons.completion]` → `### [steps.completion]`
- Line 279: `### [lessons.agent]` → `### [steps.agent]`
- Example code blocks updated

### Success Criteria:

#### Automated Verification:
- [ ] No "[[lessons]]" in docs: `grep "\[\[lessons\]\]" docs/config-schema.md`

#### Manual Verification:
- [ ] Documentation renders correctly and is consistent

---

## Phase 7: Run Tests & Final Verification

### Overview
Ensure all tests pass and no "Lesson" references remain (except historical comments).

### Success Criteria:

#### Automated Verification:
- [ ] Full verification: `make verify-agent`
- [ ] No stale references: `grep -r "LessonConfig\|LessonAgent\|LessonCompletion" src/`
- [ ] TOML files clean: `grep -r "\[\[lessons\]\]" config/`

#### Manual Verification:
- [ ] Start server and hit `/curriculum/courses/college-essay` endpoint
- [ ] Verify response shows `step_count` field

---

## Testing Strategy

### Unit Tests
- Existing curriculum loader tests should catch regressions
- No new tests needed (rename only)

### Integration Tests
- Verify course loading via HTTP endpoint

### Manual Testing Steps
1. Start server: `uv run letta-server`
2. Hit endpoint: `curl localhost:8000/curriculum/courses/college-essay`
3. Verify `step_count` appears in response
4. Hit modules endpoint: `curl localhost:8000/curriculum/courses/college-essay/modules`
5. Verify steps array appears in each module

## Migration Notes

This is a clean break - no migration needed. Any external code importing `LessonConfig` etc. will need to update imports.

## References

- Current schema: `src/letta_starter/curriculum/schema.py:200-237`
- TOML parsing: `src/letta_starter/curriculum/loader.py:410-447`
- Documentation: `docs/config-schema.md:249-309`
