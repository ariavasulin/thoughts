---
date: 2025-12-23T16:14:33-08:00
researcher: ariasulin
git_commit: 2660d75535156917eaa5c961be1e74917e090158
branch: main
repository: dwsERP
topic: "ARI-41: Additional Phases (Field Measurements, Drafting, Installation)"
tags: [research, codebase, departments, phases, operations]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: ARI-41 - Additional Phases (Field Measurements, Drafting, Installation)

**Date**: 2025-12-23T16:14:33-08:00
**Researcher**: ariasulin
**Git Commit**: 2660d75535156917eaa5c961be1e74917e090158
**Branch**: main
**Repository**: dwsERP

## Research Question

What phases/departments currently exist in the MillFlow system and how are new ones defined? Focus on understanding where departments are defined, how colors are assigned, and whether there's a concept of pre/post production operations.

## Summary

The MillFlow system defines departments in three separate locations with overlapping but distinct purposes:

1. **Master Department List** (`departments.ts`) - 12 departments with keyboard shortcuts and operation mappings
2. **Timeline Departments** (`timelineData.ts`) - 9 departments for Gantt-style timeline visualization
3. **Schedule Departments** (`scheduleData.ts`) - 7 departments for the schedule grid view

**Key Finding**: "Drafting" exists as a department and operation, but "Field Measurements" does not exist anywhere in the current system. Installation exists as `fieldInstall` department.

## Detailed Findings

### 1. Department Definitions - Master List

**File**: `millflow/src/data/departments.ts:5-92`

The `DEPARTMENTS` array defines 12 departments, split into two categories:

**Priority Queues (9)** - Actual work centers that generate prioritized work lists:
| ID | Name | Shortcut | Operations |
|---|---|---|---|
| drafting | Drafting | r | drafting |
| cnc | CNC | c | cnc |
| milling | Milling | m | milling |
| edgeBanding | Edge Banding | e | edge-banding, edgebanding |
| veneer | Veneer | v | veneer |
| finishing | Finishing | f | finishing |
| bench | Bench | b | bench |
| delivery | Delivery | d | delivery |
| fieldInstall | Field Install | i | field-install, fieldinstall |

**Status Lists (3)** - Milestone tracking, not priority queues:
| ID | Name | Shortcut | Operations |
|---|---|---|---|
| staging | Staging | s | staging |
| punchList | Punch List | p | punch-list, punchlist |
| outsourced | Outsourced | o | outsourced |

### 2. DepartmentId Type Definition

**File**: `millflow/src/types/index.ts:8-20`

The `DepartmentId` type is a union of all valid department IDs:
```typescript
export type DepartmentId =
  | 'drafting'
  | 'cnc'
  | 'veneer'
  | 'milling'
  | 'edgeBanding'
  | 'finishing'
  | 'bench'
  | 'outsourced'
  | 'staging'
  | 'delivery'
  | 'fieldInstall'
  | 'punchList';
```

### 3. Operations List

**File**: `millflow/src/data/operations.ts:11-43`

Operations are suggestions for node creation, categorized by `NodeType`:

| Type | Operations |
|---|---|
| shop | cnc, edge-banding, veneer, milling, finishing, bench, drafting |
| external | outsourced |
| material | order-material, receive-material |
| approval | client-approval, shop-drawing |
| qc | qc-check, final-inspection |
| delivery | staging, delivery |
| install | field-install, punch-list |

### 4. Timeline Department Colors

**File**: `millflow/src/data/timelineData.ts:61-71`

Colors are assigned via `TIMELINE_DEPT_COLORS` record:
```typescript
export const TIMELINE_DEPT_COLORS: Record<string, string> = {
  release: 'bg-gruvbox-fg-4',
  veneer: 'bg-gruvbox-purple-bright',
  mill: 'bg-gruvbox-blue-bright',
  cnc: 'bg-gruvbox-aqua-bright',
  bench: 'bg-gruvbox-yellow-bright',
  finish: 'bg-gruvbox-orange-bright',
  assembly: 'bg-gruvbox-green-bright',
  storage: 'bg-gruvbox-fg-4',
  delivery: 'bg-gruvbox-red-bright',
};
```

Note: Timeline uses different department names (mill vs milling, finish vs finishing, assembly vs bench).

### 5. Schedule View Departments

**File**: `millflow/src/data/scheduleData.ts:4-12`

A separate, simpler list for the schedule grid:
```typescript
export const SCHEDULE_DEPARTMENTS: ScheduleDepartment[] = [
  { id: 'veneer', name: 'Veneer' },
  { id: 'mill', name: 'Mill' },
  { id: 'cnc', name: 'CNC' },
  { id: 'bench', name: 'Bench' },
  { id: 'finish', name: 'Finish' },
  { id: 'assembly', name: 'Assembly' },
  { id: 'delivery', name: 'Loading' },
];
```

### 6. Pre/Post Production Concept

**Finding**: There is no explicit pre-production vs post-production distinction in the current system.

The `NodeType` enum (`types/index.ts:4`) provides implicit categorization:
- **Pre-production phases**: Not explicitly modeled
- **Shop floor**: `'shop'` type
- **Field/Site**: `'install'` type

Current NodeTypes:
```typescript
export type NodeType = 'shop' | 'external' | 'material' | 'approval' | 'qc' | 'delivery' | 'install';
```

### 7. Missing Phases for ARI-41

Phases mentioned in ticket that don't exist:
- **Field Measurements** - Not in departments, operations, or types
- **Drafting** - Already exists (department ID: `drafting`)
- **Installation** - Already exists (department ID: `fieldInstall`)

## Code References

- `millflow/src/data/departments.ts:5-92` - Master department list
- `millflow/src/types/index.ts:8-20` - DepartmentId type
- `millflow/src/types/index.ts:4` - NodeType enum
- `millflow/src/types/index.ts:100-106` - Department interface
- `millflow/src/data/operations.ts:11-43` - Operation suggestions
- `millflow/src/data/timelineData.ts:61-71` - Timeline colors
- `millflow/src/data/scheduleData.ts:4-12` - Schedule departments

## Architecture Documentation

### Adding a New Department

To add a new department (e.g., "Field Measurements"), modifications are needed in:

1. **`types/index.ts`** - Add to `DepartmentId` union type
2. **`data/departments.ts`** - Add to `DEPARTMENTS` array with:
   - `id`: camelCase identifier
   - `name`: Display name
   - `shortcut`: Single letter for keyboard nav
   - `operations`: Array of operation names
   - `isPriorityQueue`: true/false
3. **`data/operations.ts`** - Add to `OPERATION_SUGGESTIONS` with:
   - `id`: kebab-case identifier
   - `name`: Display name
   - `type`: NodeType (likely need new type for pre-production)
   - `keywords`: Fuzzy search terms
4. **`data/timelineData.ts`** - Add to `TIMELINE_DEPT_COLORS` if shown on timeline
5. **`data/scheduleData.ts`** - Add to `SCHEDULE_DEPARTMENTS` if shown in schedule view

### NodeType Considerations

For "Field Measurements" as a pre-production phase, the current `NodeType` options are limiting. Options:
- Use existing `'shop'` type (semantically incorrect)
- Add new type like `'pre-production'` or `'field'` to NodeType union
- Use `'external'` type if done by surveyors

## Related Research

None found.

## Open Questions

1. Should "Field Measurements" be a priority queue or status list?
2. Should a new `NodeType` be introduced for pre-production phases?
3. Do timeline and schedule views need to show pre-production phases?
4. What color should new phases use on the timeline?
