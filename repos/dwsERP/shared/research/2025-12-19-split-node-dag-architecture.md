---
date: 2025-12-19T22:51:17-08:00
researcher: Claude
git_commit: 201f79e68db6ef380d44c9b4d64ee7e34b507206
branch: main
repository: dwsERP
topic: "Split Node and DAG Architecture - Keyboard-less Experience Analysis"
tags: [research, millflow, dag, split-nodes, parallel-branches, edges, merge-nodes, keyboard-less]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
---

# Research: Split Node and DAG Architecture for Keyboard-less Experience

**Date**: 2025-12-19T22:51:17-08:00
**Researcher**: Claude
**Git Commit**: 201f79e68db6ef380d44c9b4d64ee7e34b507206
**Branch**: main
**Repository**: dwsERP

## Research Question

User wants to understand how split nodes are currently implemented to brainstorm improvements for a keyboard-less experience that supports:
1. Splitting nodes (creating parallel branches)
2. Merging nodes (converging parallel branches)
3. Up to 4 nodes on a given level
4. Drawing edges at nodes directly (directed graph, vertically stacked)

## Summary

The current millflow DAG implementation uses an **implicit edge model** where connections are defined through `parents[]` and `children[]` arrays on each node rather than separate edge objects. Parallel branches are created via `splitNode()` which duplicates downstream connections, and merged via `joinNodes()` which updates parent references. The system supports unlimited parallel branches but has visual constraints around 4-5 nodes due to minimum width requirements.

**Key limitation for keyboard-less UX**: All manipulation is keyboard-driven (o, O, s, J, d d). There is no:
- Context menu for node operations
- Touch/click interface for creating edges
- Drag-and-drop edge creation
- Visual edge handles on nodes

## Detailed Findings

### 1. Current Data Model: Implicit Edges

**Location**: `millflow/src/types/index.ts:46-47`

```typescript
interface Node {
  id: string;
  name: string;
  type: NodeType;
  children: string[];  // IDs of downstream nodes
  parents: string[];   // IDs of upstream nodes
  // ... other fields
}
```

Edges are NOT separate objects. Instead:
- Each node stores `children[]` (outgoing edges) and `parents[]` (incoming edges)
- This creates bidirectional references that must stay in sync
- No edge-specific properties (no edge IDs, labels, or styles)

**Contrast with factory-floor**: The factory-floor project uses React Flow with explicit Edge objects that have `id`, `source`, `target`, and style properties.

### 2. Split Node Implementation

**Location**: `millflow/src/store/appStore.ts:678-727`

```typescript
splitNode: (sheetId, nodeId) => {
  // Creates a new node that shares children with the original
  const newNode: Node = {
    id: `n-${Date.now()}`,
    name: 'Parallel Task',
    parents: [nodeId],           // New node is child of split point
    children: [...node.children], // Shares same downstream path
  };

  // Add new node as additional child of split point
  node.children.push(newNode.id);

  // Update downstream nodes to have multiple parents (convergence)
  for (const childId of newNode.children) {
    child.parents.push(newNode.id);
  }
}
```

**Key constraints**:
- **Requires children**: Cannot split a node with empty `children[]` - there's no downstream path to duplicate
- **Immediate convergence**: The parallel branch shares the SAME children as the original path
- **Multiple splits allowed**: Calling `splitNode` repeatedly creates additional parallel branches
- **No explicit merge point creation**: Uses existing children as convergence point

**Visual result**:
```
           ↗ n-003 (original) ↘
n-002 → → n-new (parallel)    → n-006
           ↘ n-new2 (parallel) ↗
```

### 3. Join/Merge Node Implementation

**Location**: `millflow/src/store/appStore.ts:729-758`

```typescript
joinNodes: (sheetId, nodeIds, targetNodeId) => {
  // Multiple source nodes now converge to a single target
  for (node in nodeIds) {
    node.children = [targetNodeId];
  }
  targetNode.parents = [...new Set([...targetNode.parents, ...nodeIds])];
}
```

**Current limitations**:
- Requires multi-select mode (`v` key) to select nodes to join
- Then `J` key joins them to the currently selected node
- No visual UI for drawing join connections

### 4. Level Detection for Parallel Rendering

**Location**: `millflow/src/lib/dag.ts:67-111`

The `buildLevels()` function uses BFS traversal to determine which nodes are parallel:

```typescript
// Nodes are "parallel" when they share the same BFS depth level
if (unvisited.length > 1) {
  result.push({ type: 'parallel', nodes: unvisited });
} else {
  result.push({ type: 'single', nodes: unvisited });
}

// Critical constraint: node only appears after ALL parents visited
const allParentsVisited = child.parents.every(pid => visited.has(pid));
```

**How parallel is determined**:
- NOT by explicit marking - purely by graph topology
- Nodes at the same "distance" from root AND with all parents visited = parallel
- Multiple children of the same parent = parallel at next level
- Multiple parents of the same child = that child appears after all parents (convergence)

### 5. Parallel Branch Rendering

**Location**: `millflow/src/components/dag/ParallelBranch.tsx`

```typescript
// Dynamic container width based on node count
const containerWidthClass = nodes.length === 2 ? 'w-[600px]'
  : nodes.length === 3 ? 'w-[800px]'
  : 'w-[1000px]';

// Nodes distributed evenly with minimum width
<div className="flex gap-4 pt-4">
  {nodes.map((node) => (
    <div className="relative flex-1 min-w-48"> {/* 192px min */}
```

**Maximum parallel nodes**:
- No hard code limit
- **Practical limit ~4-5**: With `min-w-48` (192px) and 1000px max container, 5 nodes = 192px each
- 6+ nodes would either overflow or violate minimum width

### 6. Current Keyboard Shortcuts for Node Manipulation

**Location**: `millflow/src/hooks/useKeyboard.ts`

| Key | Action | Constraint |
|-----|--------|------------|
| `o` | Insert node after | Requires `selectedNodeId` |
| `O` | Insert node before | Requires `selectedNodeId` |
| `s` | Split node (parallel branch) | Requires node with children |
| `v` | Toggle multi-select mode | - |
| `Space` | Toggle node selection (in multi-select) | - |
| `J` | Join selected nodes to current | Requires multi-select with nodes |
| `d d` | Delete node(s) | Vim-style double-key |
| `t` | Open template picker | - |
| `u` | Undo | - |
| `Ctrl+r` | Redo | - |

### 7. What's Missing for Keyboard-less Experience

**No context menus**: Right-click doesn't trigger any node operations.

**No touch/click edge creation**: Unlike React Flow (used in factory-floor), there are no:
- Source/target handles on nodes
- `onConnect` callback for drag-creating edges
- Edge manipulation UI

**No visual edge handles**: The DAGRenderer renders only vertical connector lines between levels, not interactive edge elements.

**Location for potential integration**: `millflow/src/components/dag/DAGNode.tsx` - would need to add Handle components similar to `factory-floor/src/components/nodes/OperationNode.tsx`.

## Code References

### Core Data Model
- `millflow/src/types/index.ts:32-56` - Node interface with children/parents arrays

### Split/Join Implementation
- `millflow/src/store/appStore.ts:678-727` - `splitNode()` function
- `millflow/src/store/appStore.ts:729-758` - `joinNodes()` function
- `millflow/src/store/appStore.ts:514-596` - `insertNode()` for adding nodes

### Level/Parallel Detection
- `millflow/src/lib/dag.ts:67-111` - `buildLevels()` BFS traversal
- `millflow/src/lib/dag.ts:7-53` - `linearizeNodes()` for navigation order
- `millflow/src/lib/dag.ts:117-125` - `findNodePosition()` for level/index lookup

### Rendering
- `millflow/src/components/dag/DAGRenderer.tsx:18-95` - Main DAG renderer
- `millflow/src/components/dag/ParallelBranch.tsx:11-55` - Parallel node layout
- `millflow/src/components/dag/DAGNode.tsx:15-130` - Individual node component

### Keyboard Controls
- `millflow/src/hooks/useKeyboard.ts:285-291` - Split shortcut (`s`)
- `millflow/src/hooks/useKeyboard.ts:301-311` - Join shortcut (`J`)
- `millflow/src/hooks/useKeyboard.ts:269-283` - Insert shortcuts (`o`, `O`)

### Contrast: React Flow in factory-floor
- `factory-floor/src/App.tsx:269` - `onConnect` handler for edge creation
- `factory-floor/src/components/nodes/OperationNode.tsx` - Handle components for connections

## Architecture Documentation

### Current DAG Data Flow

```
Sheet.nodes[] (with children/parents arrays)
    ↓
buildLevels() - BFS traversal determines parallel vs sequential
    ↓
RenderLevel[] - { type: 'single' | 'parallel', nodes: Node[] }
    ↓
DAGRenderer maps levels to components:
  - type === 'parallel' → <ParallelBranch nodes={...}>
  - type === 'single'   → <DAGNode node={...}>
    ↓
Visual: Vertical stack with connector lines
```

### Split Operation Data Flow

```
User presses 's' on node n-002 (which has children: ['n-006'])
    ↓
splitNode(sheetId, 'n-002') called
    ↓
New node created: { id: 'n-new', parents: ['n-002'], children: ['n-006'] }
    ↓
n-002.children updated: ['n-006', 'n-new']
    ↓
n-006.parents updated: ['n-002', 'n-new']
    ↓
buildLevels() now sees n-002 has two children → next level is 'parallel'
    ↓
ParallelBranch renders both n-006 and n-new side by side
```

### Join Operation Data Flow

```
User enables multi-select (v), selects n-003 and n-004, navigates to n-006
    ↓
User presses 'J' to join
    ↓
joinNodes(sheetId, ['n-003', 'n-004'], 'n-006')
    ↓
n-003.children = ['n-006'], n-004.children = ['n-006']
n-006.parents = [...existing, 'n-003', 'n-004']
    ↓
Graph now converges multiple paths to n-006
```

## Historical Context (from thoughts/)

### Related Plans
- `thoughts/shared/plans/2025-12-19-millflow-parallel-nodes-and-creation-fix.md` - Fixed parallel node display, added `justify-center` and `w-56` sizing
- `thoughts/shared/plans/2025-12-18-visual-dag-editor.md` - Original DAG editor design
- `thoughts/shared/plans/2025-12-19-millflow-full-data-manipulation.md` - Full data manipulation implementation

### Related Research
- `thoughts/shared/research/2025-12-19-millflow-dag-rendering-node-creation.md` - DAG rendering pipeline documentation
- `thoughts/shared/research/2025-12-19-parallel-node-sizing.md` - Parallel node sizing constraints
- `thoughts/shared/research/2025-12-19-node-positioning-architecture.md` - Node positioning system

## Open Questions

1. **Should edges become explicit objects?**
   - Current implicit model (children/parents arrays) works for simple DAGs
   - Explicit edges would enable: edge labels, edge styling, edge-level operations
   - Would require significant refactor of all node manipulation functions

2. **React Flow integration for millflow?**
   - factory-floor already uses @xyflow/react with full edge manipulation
   - Could potentially share infrastructure for edge creation UI
   - Trade-off: custom DAGRenderer gives more control over layout

3. **Touch/mobile considerations?**
   - No current touch gesture handling
   - Would need long-press for context menu, drag for edge creation
   - ParallelBranch's min-width constraints may need adjustment for small screens

4. **How to create edges without keyboard?**
   - Option A: Add Handle components to DAGNode, use React Flow's onConnect
   - Option B: Add context menu with "Connect to..." option that enters selection mode
   - Option C: Add floating action buttons on node hover
   - Option D: Tap source node, then tap target node (two-tap pattern)
