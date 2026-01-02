---
date: 2025-12-19T18:10:44-08:00
researcher: ariasulin
git_commit: 190897b666552a36a1f04bdf1c3561193eb8b63b
branch: main
repository: dwsERP
topic: "Sidebar opens to wrong node after node creation"
tags: [research, codebase, millflow, node-creation, sidebar, bug]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Sidebar Opens to Wrong Node After Node Creation

**Date**: 2025-12-19T18:10:44-08:00
**Researcher**: ariasulin
**Git Commit**: 190897b666552a36a1f04bdf1c3561193eb8b63b
**Branch**: main
**Repository**: dwsERP

## Research Question

When creating a new node in the millflow node view and launching the right sidebar, the sidebar displays the wrong node - it "jumps a couple" nodes instead of showing the newly created node.

## Summary

The bug is caused by an index mismatch in the `openSelected` function. When a new node is created, the `selectedIndex` is calculated using linearized (BFS traversal) order, but when the sidebar is opened via `openSelected`, it looks up the node using raw array index (`sheet.nodes[selectedIndex]`). Since new nodes are pushed to the end of the `sheet.nodes` array but appear at different positions in the linearized order, this causes the sidebar to display a different node than expected.

## Detailed Findings

### Node Creation Flow

When a user creates a new node:

1. **Keyboard trigger** (`millflow/src/hooks/useKeyboard.ts:269-275`):
   - Pressing `o` calls `setNodeCreationMode('after')`
   - Opens the SmartNodeCreator modal

2. **Node insertion** (`millflow/src/store/appStore.ts:512-594`):
   - New node ID is generated: `n-${Date.now()}` (line 515)
   - DAG relationships are wired (parents/children updated)
   - **New node is pushed to the END of the array** (line 578: `nodes.push(newNode)`)
   - `selectedIndex` is calculated from **linearized (BFS) order** (lines 583-585)
   - State is updated with:
     - `selectedNodeId: newNodeId` (correct)
     - `selectedIndex: newIndex` (from linearized order)

### Sidebar Opening Flow

When the user presses Enter to open the sidebar:

1. **Keyboard handler** (`millflow/src/hooks/useKeyboard.ts:222-230`):
   - Calls `openSelected()` if sidebar is not already open

2. **openSelected function** (`millflow/src/store/appStore.ts:289-296`):
   ```typescript
   } else if (view === 'sheet' && selectedSheetId) {
     const sheet = allSheets.find(s => s.id === selectedSheetId);
     if (sheet) {
       const node = sheet.nodes[selectedIndex];  // <-- BUG HERE
       if (node) {
         set({ selectedNodeId: node.id, showNodeDetail: true });
       }
     }
   }
   ```

### The Bug

The bug is at `millflow/src/store/appStore.ts:292`:

```typescript
const node = sheet.nodes[selectedIndex];
```

**Problem**: `selectedIndex` is calculated from linearized (BFS) order, but this line uses it as a raw array index into `sheet.nodes`.

**Example scenario**:
- Existing nodes in `sheet.nodes`: [A, B, C, D, E] (indices 0-4)
- Linearized order might be: [A, B, C, D, E] (same order for simple DAGs)
- User creates new node F after B
- F is **pushed to end**: `sheet.nodes` = [A, B, C, D, E, F] (F at index 5)
- Linearized order: [A, B, **F**, C, D, E] (F at linearized index 2)
- `selectedIndex` = 2 (linearized position)
- `selectedNodeId` = F's ID (correct)
- User presses Enter to open sidebar
- `openSelected` does `sheet.nodes[2]` which returns **C** (not F)
- Sidebar displays **C** instead of **F**

### Relevant Code References

- `millflow/src/store/appStore.ts:289-296` - The buggy `openSelected` logic for sheet view
- `millflow/src/store/appStore.ts:512-594` - `insertNode` function showing correct linearized index calculation
- `millflow/src/store/appStore.ts:578` - New node pushed to end of array
- `millflow/src/store/appStore.ts:582-585` - Correct linearized index calculation
- `millflow/src/lib/dag.ts:7-53` - `linearizeNodes` function (BFS traversal)
- `millflow/src/lib/dag.ts:67-111` - `buildLevels` function (used by DAGRenderer)
- `millflow/src/hooks/useKeyboard.ts:222-230` - Enter key handler calling `openSelected`

### Data Flow Visualization

```
Node Creation:
1. insertNode() → nodes.push(newNode) → node at end of array (index N)
2. linearizeNodes() → finds node at linearized position (index M)
3. set({ selectedNodeId: newId, selectedIndex: M })

Sidebar Opening:
1. openSelected() → sheet.nodes[selectedIndex]
2. Uses M as raw array index (wrong!)
3. Gets node at array position M (not the linearized position M)
4. Overwrites selectedNodeId with wrong node's ID
```

## Architecture Documentation

The millflow application uses two ordering systems:

1. **Raw array order** (`sheet.nodes[]`): Nodes in the order they were added to the sheet. New nodes are appended to the end.

2. **Linearized order** (`linearizeNodes(sheet.nodes)`): BFS traversal starting from root nodes (nodes with no parents). This represents the visual top-to-bottom order in the DAG display.

The `selectedIndex` state variable tracks position in **linearized order** for keyboard navigation (j/k to move up/down visually). However, several places in the code incorrectly use this index to access the raw array.

## Open Questions

1. Are there other places in the codebase that make the same mistake of using `selectedIndex` as a raw array index instead of linearized order?

2. Should the codebase be refactored to maintain consistent indexing, or should each access point linearize the nodes first?

3. Alternatively, should `selectedNodeId` be used directly instead of looking up by index when the node ID is already known?
