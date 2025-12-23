---
date: 2025-12-21T00:33:48-08:00
researcher: ariasulin
git_commit: 56c02369e91bfd15c379d9b7df2524340d80e7a2
branch: main
repository: dwsERP
topic: "Node Behavior and Keyboard Actions for Insert/Cut/Paste"
tags: [research, codebase, nodes, keyboard, dag, millflow]
status: complete
last_updated: 2025-12-21
last_updated_by: ariasulin
---

# Research: Node Behavior and Keyboard Actions for Insert/Cut/Paste

**Date**: 2025-12-21T00:33:48-08:00
**Researcher**: ariasulin
**Git Commit**: 56c02369e91bfd15c379d9b7df2524340d80e7a2
**Branch**: main
**Repository**: dwsERP

## Research Question

Research node behavior for planned changes:
- O/o node before/after - Node directly attached to parent and attaches exactly to child below
- Nodes may ONLY have two edges: parent edge and child edge
- S - make node right of focused node with exact parent and child
- x - cuts a node
- P/p - paste before/after
- s, p - paste right

## Summary

The current implementation supports unlimited parent/child edges per node (no two-edge constraint). Insert before/after already exists via `o k`/`o j` sequences. Sibling insertion exists via `o l`/`o h`. **NO clipboard/cut/paste functionality exists** - this must be implemented from scratch. The planned changes involve:
1. Remapping 'O'/'o' to single-key before/after insert
2. Adding 'S' for sibling creation
3. Implementing clipboard state and 'x' for cut
4. Implementing 'P'/'p'/'s' for paste operations

## Detailed Findings

### 1. Node Data Structure & Edge Model

**Location**: `millflow/src/types/index.ts:32-49`

```typescript
export interface Node {
  id: string;
  name: string;
  // ... other properties ...
  children: string[];  // Unbounded array
  parents: string[];   // Unbounded array
}
```

**Current Edge Constraints**:
- **NONE** - Both `children` and `parents` are unbounded arrays
- A node can have 0 to N parents (root nodes have 0)
- A node can have 0 to N children (leaf nodes have 0)
- Nodes with `children.length > 1` create split points (parallel branches)
- Nodes with `parents.length > 1` are convergence points

**Planned Change Impact**:
To enforce "only two edges" (one parent, one child), you would need to:
1. Update types to enforce single parent/child (or add validation)
2. Modify sibling creation logic - currently siblings share parents AND children
3. Rethink the split/reconverge model entirely

### 2. Current Keyboard Mappings

**Location**: `millflow/src/hooks/useKeyboard.ts`

#### Currently Mapped Keys

| Key/Sequence | Action | Location |
|--------------|--------|----------|
| `o j` / `o ArrowDown` | Insert node AFTER focused | lines 245-253 |
| `o k` / `o ArrowUp` | Insert node BEFORE focused | lines 255-263 |
| `o l` / `o ArrowRight` | Insert sibling to RIGHT | lines 265-273 |
| `o h` / `o ArrowLeft` | Insert sibling to LEFT | lines 275-283 |
| `d d` | Delete node(s) | lines 187-197 |
| `M k` / `M ArrowUp` | Swap node up in branch | lines 226-233 |
| `M j` / `M ArrowDown` | Swap node down in branch | lines 235-243 |

#### NOT Currently Mapped (Available)

| Key | Current Status |
|-----|----------------|
| `O` (uppercase) | Not mapped |
| `o` (single key) | Only starts sequence, no single-key action |
| `x` | Not mapped |
| `p` | Not mapped |
| `P` | Not mapped |
| `s` | Not mapped |
| `S` | Not mapped |

### 3. Node Insertion Implementation

**Location**: `millflow/src/store/appStore.ts:549-690`

#### `insertNode(sheetId, targetNodeId, position, nodeData)`

**Position: 'before'** (lines 594-612):
- New node's `children = [targetNodeId]`
- New node's `parents = target.parents` (inherits target's parents)
- Target's `parents = [newNode.id]`
- All original parents update their children to point to new node

**Position: 'after'** (lines 575-593):
- New node's `parents = [targetNodeId]`
- New node's `children = target.children` (inherits target's children)
- Target's `children = [newNode.id]`
- All original children update their parents to point to new node

**Position: 'left'/'right' (Sibling)** (lines 614-671):
- If target has no parents, falls back to 'after' behavior
- Otherwise creates parallel branch:
  - New node's `parents = target.parents` (same parents)
  - New node's `children = target.children` (same children)
  - All shared parents add new node to their children array
  - All shared children add new node to their parents array
- Order in parent's children array determines left/right position

### 4. Sibling Behavior (Current)

When creating a sibling with 'left'/'right':
- **Creates a split point at the parent**
- **Creates a convergence point at the child**
- The new node shares BOTH parents AND children with target
- This creates a parallel branch in the DAG

Example: If node B has parent A and child C:
```
Before:  A → B → C

After (add sibling S to right of B):
         ┌→ B →┐
    A →──┤     ├──→ C
         └→ S →┘
```

Both B and S have `parents: ['A']` and `children: ['C']`.

### 5. Clipboard/Cut/Paste: NOT IMPLEMENTED

**Finding**: No clipboard functionality exists in the codebase.

**What needs to be implemented for cut/paste**:

1. **Clipboard State** (add to appStore.ts):
```typescript
interface AppState {
  clipboard: {
    node: Node | null;
    operation: 'cut' | 'copy' | null;
  };
}
```

2. **Cut Action** (`cutNode`):
   - Store node data in clipboard
   - Remove node from DAG (similar to deleteNode)
   - Reconnect parent/child relationships
   - Set operation to 'cut'

3. **Paste Actions**:
   - `pasteNodeBefore`: Insert clipboard node before target
   - `pasteNodeAfter`: Insert clipboard node after target
   - `pasteNodeRight`: Insert clipboard node as sibling to right
   - Clear clipboard after paste (for cut) or keep (for copy)

### 6. Delete Node Implementation (Reference for Cut)

**Location**: `millflow/src/store/appStore.ts:859-900`

The `deleteNode` function shows how to cleanly remove a node:
1. Clear any marks pointing to node
2. Filter node from nodes array
3. For each former parent: remove node from children, add node's children
4. For each former child: remove node from parents, add node's parents

This reconnection logic should be reused for cut operation.

### 7. DAG Layout & Navigation

**Location**: `millflow/src/lib/dag.ts`

**`buildRegions()`** (lines 269-317):
- Identifies split points (`node.children.length > 1`)
- Groups parallel branches into `BranchRegion` structures
- Used by navigation to determine left/right movement

**Left/Right Navigation** (`appStore.ts:223-293`):
- Finds current node's position within its branch
- Moves to same vertical position in adjacent branch
- Uses `region.branches[branchIdx + 1]` for right navigation

## Code References

- `millflow/src/types/index.ts:32-49` - Node interface with parents/children arrays
- `millflow/src/hooks/useKeyboard.ts:245-283` - Current o-key sequences for insertion
- `millflow/src/store/appStore.ts:549-690` - insertNode implementation
- `millflow/src/store/appStore.ts:772-826` - splitNode (sibling creation)
- `millflow/src/store/appStore.ts:859-900` - deleteNode (reference for cut)
- `millflow/src/lib/dag.ts:269-317` - buildRegions for layout

## Implementation Notes for Planned Changes

### Recommended Keyboard Mapping

| Key | Action | Store Function |
|-----|--------|----------------|
| `O` | Insert node BEFORE focused | `insertNode(..., 'before', ...)` |
| `o` | Insert node AFTER focused | `insertNode(..., 'after', ...)` |
| `S` | Insert sibling RIGHT of focused | `insertNode(..., 'right', ...)` |
| `x` | Cut focused node | NEW: `cutNode(sheetId, nodeId)` |
| `P` | Paste BEFORE focused | NEW: `pasteNode(sheetId, targetId, 'before')` |
| `p` | Paste AFTER focused | NEW: `pasteNode(sheetId, targetId, 'after')` |
| `s` | Paste RIGHT of focused | NEW: `pasteNode(sheetId, targetId, 'right')` |

### Two-Edge Constraint Consideration

If you want to enforce "only two edges" (one parent, one child):
1. This changes the fundamental DAG model from multi-parent/multi-child to single-parent/single-child
2. Sibling creation would need different semantics
3. Split/convergence regions wouldn't exist in current form
4. Consider if you want a strict linear chain or still allow some branching

The current sibling model creates nodes with **same parents AND same children**, which inherently creates multiple edges.

## Open Questions

1. For the two-edge constraint: Should sibling nodes share the same single parent and single child, or should they branch differently?
2. For paste right (`s`): Should this behave exactly like `S` (create sibling) but with clipboard data?
3. Should cut preserve the node's children/parents configuration, or paste as a "fresh" node?
