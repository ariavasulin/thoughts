---
date: 2025-12-20T01:52:38-08:00
researcher: claude
git_commit: 201f79e68db6ef380d44c9b4d64ee7e34b507206
branch: main
repository: dwsERP
topic: "Branch Region Duplicate Node and Navigation Bug"
tags: [research, codebase, dag, navigation, millflow, bug-analysis]
status: complete
last_updated: 2025-12-20
last_updated_by: claude
---

# Research: Branch Region Duplicate Node and Navigation Bug

**Date**: 2025-12-20T01:52:38-08:00
**Researcher**: claude
**Git Commit**: 201f79e68db6ef380d44c9b4d64ee7e34b507206
**Branch**: main
**Repository**: dwsERP

## Research Question

Investigate the bugs identified in the Branch Region Renderer implementation:
1. Duplicate node rendering - "Finishing" node appears twice
2. j/k navigation gets stuck, doesn't traverse correctly

## Summary

The bugs are caused by a single root issue in `buildRegions()` at `millflow/src/lib/dag.ts:294-296`. When a `BranchRegion` is created, the convergence node is stored in `region.convergenceNode` but is NOT added to the `processed` set. Subsequently, `processNode(region.convergenceNode.id)` is called, which creates a **second** `SingleRegion` for the same node. This causes:
1. The node to be rendered twice (once in `BranchRegionView` and once as a standalone `SingleRegion`)
2. The node to appear twice in `linearizeRegions()` output, breaking j/k navigation

## Detailed Findings

### The Bug Location: `buildRegions()` in dag.ts:282-306

```typescript
function processNode(nodeId: string): void {
  if (processed.has(nodeId)) return;  // LINE 283: Guard check

  const node = nodeMap.get(nodeId);
  if (!node) return;

  if (node.children.length > 1) {
    const region = buildBranchRegion(node, nodeMap, processed);  // Builds region
    regions.push(region);

    // BUG: convergenceNode is NOT in `processed` set here
    if (region.convergenceNode) {
      processNode(region.convergenceNode.id);  // LINE 295: Calls processNode
      // ^ This creates a SingleRegion because convergenceNode isn't processed
    }
  } else {
    processed.add(nodeId);  // LINE 299: Only adds to processed here
    regions.push({ type: 'single', node });
    // ...
  }
}
```

### How `buildBranchRegion()` Works (dag.ts:224-263)

1. Marks `splitNode` as processed (line 229)
2. Finds convergence point using `findConvergencePoint()`
3. Extracts each branch:
   - Iterates through children of split node
   - For each branch, follows single-child paths until reaching convergence
   - Marks each branch node as processed (line 244)
   - **Stops at convergenceId WITHOUT marking it as processed**

The convergence node is intentionally excluded from branches (line 240: `while currentId !== convergenceId`) but is never added to the `processed` set.

### DAG Structure in Mock Data (sheet-012 "Reception Desk")

```
n-001 (CNC)
   ↓
n-002 (Assembly)  ← SPLIT POINT (3 children)
   ↓ ↓ ↓
n-003  n-004  n-005  ← BRANCHES (Hardware, Finish approval, Edge banding)
   ↓    ↓      ↓
      n-006       ← CONVERGENCE POINT (Finishing) - all 3 point here
         ↓
      n-007 (QC)
         ↓
      n-008 (Delivery)
         ↓
      n-009 (Install)
```

### What Happens Currently

1. `processNode('n-001')` → SingleRegion for n-001
2. `processNode('n-002')` → BranchRegion with:
   - splitNode: n-002
   - convergenceNode: n-006
   - branches: [[n-003], [n-004], [n-005]]
3. After BranchRegion, calls `processNode('n-006')` at line 295
4. n-006 is NOT in `processed`, so it creates **another** SingleRegion for n-006
5. `processNode('n-007')` → SingleRegion for n-007
6. ...and so on

### Resulting Regions Array

```
[
  SingleRegion(n-001),
  BranchRegion(splitNode: n-002, convergenceNode: n-006, branches: [...]),
  SingleRegion(n-006),  // ← DUPLICATE!
  SingleRegion(n-007),
  SingleRegion(n-008),
  SingleRegion(n-009),
]
```

### How Rendering Is Affected

In `DAGRenderer.tsx:26-51`, the `regions` array is mapped directly:
- BranchRegion renders n-006 via `BranchRegionView.tsx:96-108`
- The duplicate SingleRegion also renders n-006 via `DAGNode`

### How Navigation Is Affected

`linearizeRegions()` in dag.ts:320-342:
- For BranchRegion: adds splitNode, branch nodes, then convergenceNode (line 336-338)
- For SingleRegion: adds the node (line 325)

Result: `flatNodes` contains n-006 twice, causing j/k to oscillate between two entries pointing to the same node.

## Code References

- `millflow/src/lib/dag.ts:282-306` - `processNode()` function with the bug
- `millflow/src/lib/dag.ts:224-263` - `buildBranchRegion()` that doesn't mark convergence as processed
- `millflow/src/lib/dag.ts:320-342` - `linearizeRegions()` that produces duplicate entries
- `millflow/src/components/dag/DAGRenderer.tsx:26-51` - Renderer that iterates regions
- `millflow/src/components/dag/BranchRegionView.tsx:96-108` - Convergence node rendering in branch view
- `millflow/src/store/appStore.ts:198-221` - `navigateDown()` using linearized nodes
- `millflow/src/data/mockData.ts:118-260` - sheet-012 DAG with the split/convergence pattern

## Architecture Documentation

### Region Model (dag.ts:138-168)

- `SingleRegion`: A node that is not part of a parallel split
- `BranchRegion`: Contains a split node, optional convergence node, and array of branches
- `Branch`: A linear path of nodes between split and convergence

### Navigation Model (appStore.ts:180-293)

- j/k (navigateUp/Down): Uses `linearizeRegions()` for flat traversal
- h/l (navigateLeft/Right): Finds current branch in BranchRegion, moves to adjacent branch

## Historical Context (from thoughts/)

- `thoughts/shared/handoffs/general/2025-12-20_01-50-34_millflow-branch-region-bugs.md` - Original bug identification with suggested fixes
- `thoughts/shared/plans/2025-12-20-millflow-branch-region-renderer.md` - Implementation plan for the branch region feature

## Related Research

None found.

## Open Questions

None - the bug root cause is fully identified.
