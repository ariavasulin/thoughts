---
date: 2025-12-19T16:59:11-08:00
researcher: ariasulin
git_commit: 96bdd2df9b027434f43a051b614a8e705e96f1ce
branch: main
repository: dwsERP
topic: "Parallel Node Sizing and Layout in DAG Renderer"
tags: [research, codebase, millflow, dag, parallel-nodes, layout, sizing]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Parallel Node Sizing and Layout in DAG Renderer

**Date**: 2025-12-19T16:59:11-08:00
**Researcher**: ariasulin
**Git Commit**: 96bdd2df9b027434f43a051b614a8e705e96f1ce
**Branch**: main
**Repository**: dwsERP

## Research Question
How are parallel nodes currently sized and laid out in the millflow DAG renderer? The user wants parallel nodes (2+ on same layer) to expand and take up more space rather than being constrained to the same fixed width as single nodes.

## Summary

Parallel nodes in millflow are currently rendered with a **fixed width of 224px (`w-56`)** per node, regardless of how many nodes exist in the parallel branch. The containing view constrains the DAG to `max-w-2xl` (672px), but parallel branches can exceed this with calculated connector line widths.

### Key Sizing Constants
- **Single node**: Expands to `w-full` of container (up to 672px)
- **Parallel node container**: Fixed `w-56` (224px) per node
- **Gap between parallel nodes**: `gap-4` (16px)
- **Connector line widths**: 464px (2 nodes), 704px (3 nodes), 944px (4 nodes)

## Detailed Findings

### ParallelBranch Component - Fixed Node Width

**Location**: `millflow/src/components/dag/ParallelBranch.tsx:38-50`

```tsx
<div className="flex justify-center gap-4 pt-4">
  {nodes.map((node) => (
    <div key={node.id} className="relative w-56 min-w-0">
      {/* ... */}
      <DAGNode node={node} compact />
    </div>
  ))}
</div>
```

Each parallel node is wrapped in a container with `w-56` (224px fixed width). This is the same regardless of whether there are 2, 3, or 4 parallel nodes.

### Connector Line Width Calculation

**Location**: `millflow/src/components/dag/ParallelBranch.tsx:28-30`

```tsx
// Calculate line width: (node_count * node_width) + ((node_count - 1) * gap)
// With w-56 (224px) and gap-4 (16px): 2 nodes = 464px, 3 nodes = 704px, 4 nodes = 944px
const lineWidthClass = nodes.length === 2 ? 'w-[464px]' : nodes.length === 3 ? 'w-[704px]' : 'w-[944px]';
```

The horizontal connector lines DO scale based on node count, but the individual node containers remain fixed at 224px.

### View Container Constraint

**Location**: `millflow/src/views/SheetView.tsx:168-177`

```tsx
<div className="flex-1 overflow-auto p-6">
  <div className="max-w-2xl mx-auto">
    <DAGRenderer ... />
  </div>
</div>
```

The DAG is constrained to `max-w-2xl` (672px max width), but parallel branches can exceed this (they render outside the constraint with horizontal overflow).

### DAGNode Sizing - Width Difference

**Location**: `millflow/src/components/dag/DAGNode.tsx:34-45`

```tsx
<button className={cn('w-full text-left transition-all duration-100', ...)}>
```

DAGNode itself uses `w-full`, but in parallel context it's constrained by its parent container (`w-56` in ParallelBranch).

### Compact Mode in Parallel

**Location**: `millflow/src/components/dag/ParallelBranch.tsx:47`

Parallel nodes receive `compact` prop, which reduces padding and hides some detail info to fit in the smaller space.

## Code References

- `millflow/src/components/dag/ParallelBranch.tsx:38-40` - Fixed `w-56` container for each parallel node
- `millflow/src/components/dag/ParallelBranch.tsx:28-30` - Connector line width calculation
- `millflow/src/components/dag/DAGRenderer.tsx:127-132` - ParallelBranch vs single DAGNode rendering decision
- `millflow/src/components/dag/DAGNode.tsx:34` - DAGNode uses `w-full`
- `millflow/src/views/SheetView.tsx:169` - Container uses `max-w-2xl`

## Architecture Documentation

### Current Layout Flow

1. **SheetView** provides container with `max-w-2xl` (672px)
2. **DAGRenderer** builds levels using BFS, identifies parallel branches
3. For parallel branches, uses **ParallelBranch** component
4. **ParallelBranch** wraps each node in `w-56` (224px) fixed container
5. **DAGNode** fills its container with `w-full`

### Parallel Detection Logic

**Location**: `millflow/src/components/dag/DAGRenderer.tsx:52-71`

When multiple unvisited nodes exist at the same level, they're grouped into a `RenderLevel` with `type: 'parallel'`.

## Related Files

| File | Purpose |
|------|---------|
| `millflow/src/components/dag/ParallelBranch.tsx` | Parallel node container with fixed sizing |
| `millflow/src/components/dag/DAGRenderer.tsx` | Level detection and rendering orchestration |
| `millflow/src/components/dag/DAGNode.tsx` | Individual node rendering |
| `millflow/src/views/SheetView.tsx` | View container with max-width constraint |
| `millflow/src/lib/dag.ts` | BFS linearization utilities |

## Open Questions

1. Should expanded parallel nodes have a maximum width limit?
2. How should the layout handle 5+ parallel nodes (currently only calculates up to 4)?
3. Should the `max-w-2xl` constraint on SheetView be removed/increased for parallel branches?
