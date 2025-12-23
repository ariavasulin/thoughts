---
date: 2025-12-19T18:03:33-08:00
researcher: ariasulin
git_commit: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
branch: main
repository: dwsERP
topic: "DAG Vertical Spacing Between Nodes"
tags: [research, codebase, millflow, dag, visualization]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: DAG Vertical Spacing Between Nodes

**Date**: 2025-12-19T18:03:33-08:00
**Researcher**: ariasulin
**Git Commit**: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
**Branch**: main
**Repository**: dwsERP

## Research Question

Why does the DAG renderer show two vertical connector lines between single nodes but only one line when there are parallel nodes in a layer? How can it be made consistent (always one line)?

## Summary

The inconsistent vertical spacing is caused by **two separate connector lines being rendered for single-type levels**: one "before" line rendered for every level, plus an additional "after" line rendered only for single-type levels. Parallel levels don't render the "after" line, resulting in less space.

## Detailed Findings

### DAGRenderer.tsx - Connector Line Logic

The vertical spacing between nodes is controlled in `DAGRenderer.tsx` by two separate connector line renderers:

**1. "Before" connector (lines 67-71)** - Renders for ALL levels except the first:
```jsx
{levelIdx > 0 && (
  <div className="flex justify-center mb-3">
    <div className="w-px h-4 bg-gruvbox-bg-3" />
  </div>
)}
```

**2. "After" connector (lines 90-94)** - Renders ONLY for single-type levels:
```jsx
{levelIdx < levels.length - 1 && level.type === 'single' && (
  <div className="flex justify-center mt-3">
    <div className="w-px h-4 bg-gruvbox-bg-3" />
  </div>
)}
```

### Why This Causes Inconsistent Spacing

| Transition | Before Line | After Line | Total Lines |
|------------|-------------|------------|-------------|
| Single → Single | Yes (from 2nd) | Yes (from 1st) | **2 lines** |
| Single → Parallel | Yes (from parallel) | Yes (from single) | **2 lines** |
| Parallel → Single | Yes (from single) | No | **1 line** |
| Parallel → Parallel | Yes (from 2nd) | No | **1 line** |

The condition `level.type === 'single'` on line 90 means parallel levels never add an "after" connector, while single levels do.

### ParallelBranch.tsx - Internal Connectors

The `ParallelBranch` component handles its own internal connectors:
- Split indicator line at top (line 36)
- Vertical connectors for each parallel node (line 43)
- Join indicator line at bottom (line 55)

These are separate from the inter-level spacing in DAGRenderer.

## Code References

- `millflow/src/components/dag/DAGRenderer.tsx:67-71` - "Before" connector line (renders for all levels)
- `millflow/src/components/dag/DAGRenderer.tsx:90-94` - "After" connector line (renders only for single levels)
- `millflow/src/components/dag/DAGRenderer.tsx:63` - Container uses `space-y-3` for base spacing
- `millflow/src/components/dag/ParallelBranch.tsx:36` - Split indicator line
- `millflow/src/components/dag/ParallelBranch.tsx:55` - Join indicator line

## Architecture Documentation

The DAG rendering system uses a level-based approach:
1. `buildLevels()` in `@/lib/dag` organizes nodes into sequential levels
2. Each level has a `type` of either `'single'` or `'parallel'`
3. `DAGRenderer` iterates through levels, rendering connectors between them
4. `ParallelBranch` handles multi-node levels with its own connector styling

## Open Questions

None - the cause is clearly identified in the code.
