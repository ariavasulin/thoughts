---
date: 2025-12-19T23:49:11Z
researcher: Claude
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "MillFlow DAG Rendering and Node Creation Systems"
tags: [research, millflow, dag, node-creation, parallel-branches, rendering]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
---

# Research: MillFlow DAG Rendering and Node Creation Systems

**Date**: 2025-12-19T23:49:11Z
**Researcher**: Claude
**Git Commit**: b44e10ada59603fff73defcdafb24f122525e52c
**Branch**: main
**Repository**: dwsERP

## Research Question

User observed:
1. Parallel nodes (multiple nodes at same DAG level) appear "messy and weird" with truncated names
2. Adding new nodes via keyboard doesn't work despite UI mockup existing

## Summary

### Parallel Node Rendering

The MillFlow DAG uses a three-tier rendering system:
- **DAGRenderer** linearizes the DAG into levels, detecting parallel branches when `nodes.length > 1`
- **ParallelBranch** renders multiple nodes horizontally using flexbox with `flex-1` (equal width distribution)
- **DAGNode** renders individual cards with `truncate` class for text overflow

When 3 parallel nodes exist, each gets approximately 1/3 of container width minus 16px gaps. This causes text truncation on longer names.

### Node Creation

The node creation system **is fully implemented**:
- Keyboard bindings: `o` (insert after), `O` (insert before) at `useKeyboard.ts:239-253`
- UI component: `NodeCreator.tsx` with name input and type picker
- Store actions: `insertNode` with complete graph manipulation logic
- View integration: `SheetView.tsx:157-165` renders NodeCreator overlay

**Critical requirement**: `selectedNodeId` must be set for the `o`/`O` keys to work. The keyboard binding guards check:
```typescript
if (view === 'sheet' && selectedSheetId && selectedNodeId) {
  setNodeCreationMode('after');
}
```

## Detailed Findings

### Parallel Node Layout (`millflow/src/components/dag/ParallelBranch.tsx`)

**Container structure** (line 34):
```tsx
<div className="flex gap-4 pt-4">
```
- `flex`: Horizontal flexbox layout
- `gap-4`: 16px spacing between nodes
- `pt-4`: 16px top padding for split connector line

**Node wrappers** (line 36):
```tsx
<div key={node.id} className="flex-1 relative">
```
- `flex-1`: Each node gets equal fraction of remaining width
- With 3 nodes and 16px gaps: each node width = `(container - 32px) / 3`

**Compact mode** (lines 39-44):
- Parallel nodes render with `compact={true}` prop
- Reduces padding from `px-4 py-3` to `px-3 py-2`
- Hides secondary content (waiting-on text, suggestions)

### Text Truncation (`millflow/src/components/dag/DAGNode.tsx`)

**Truncation classes** (lines 49, 60-65):
```tsx
<div className="flex items-center gap-3 min-w-0">
  ...
  <span className="font-medium truncate">
    {node.name}
  </span>
```
- `min-w-0`: Allows flex item to shrink below content size
- `truncate`: Applies `overflow: hidden`, `text-overflow: ellipsis`, `white-space: nowrap`

**Right-side metadata** (line 69):
```tsx
<div className="flex items-center gap-3 shrink-0 text-sm">
```
- `shrink-0`: Right side doesn't shrink, forcing left side to truncate

### Node Creation Flow

**1. Keyboard trigger** (`millflow/src/hooks/useKeyboard.ts:239-253`):
```typescript
case 'o':
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    setNodeCreationMode('after');
  }
  break;
```

**2. State management** (`millflow/src/store/appStore.ts`):
- `nodeCreationMode: 'before' | 'after' | null` (line 24)
- `setNodeCreationMode` action (line 296)
- `insertNode` action with full graph manipulation (lines 426-502)

**3. UI rendering** (`millflow/src/views/SheetView.tsx:157-165`):
```tsx
{nodeCreationMode && selectedNodeId && (
  <div className="fixed inset-0 z-30 flex items-center justify-center bg-black/40">
    <NodeCreator
      position={nodeCreationMode}
      onSave={handleNodeCreate}
      onCancel={handleNodeCreateCancel}
    />
  </div>
)}
```

**4. NodeCreator component** (`millflow/src/components/dag/NodeCreator.tsx`):
- Name input (auto-focused on mount)
- Type selector (triggered by `/` key)
- Keyboard handling: Enter to save, Escape to cancel
- Available types: shop, external, material, approval, qc, delivery, install

**5. Graph insertion** (`millflow/src/store/appStore.ts:426-502`):
- Pushes history before mutation (line 427)
- Creates node with unique ID: `n-${Date.now()}` (line 429)
- Updates parent/child relationships bidirectionally (lines 452-490)
- Clears creation mode and selects new node (lines 498-500)

### Selection State Requirements

For node creation to work, `selectedNodeId` must be set. This happens via:

1. **j/k navigation** (`appStore.ts:131-162`): Now syncs `selectedNodeId` with `selectedIndex`
2. **Entering sheet view** (`appStore.ts:194-202`): Sets `firstNodeId` when opening a sheet
3. **Clicking a node** (`SheetView.tsx:58-65`): `handleSelectNode` calls `selectNode(nodeId)`

## Code References

### Parallel Rendering
- `millflow/src/components/dag/DAGRenderer.tsx:26-98` - Level detection algorithm
- `millflow/src/components/dag/DAGRenderer.tsx:124-136` - Conditional parallel/single rendering
- `millflow/src/components/dag/ParallelBranch.tsx:28-53` - Horizontal layout structure
- `millflow/src/components/dag/DAGNode.tsx:34-46` - Node sizing and layout
- `millflow/src/components/dag/DAGNode.tsx:60-65` - Text truncation

### Node Creation
- `millflow/src/hooks/useKeyboard.ts:239-253` - o/O keyboard bindings
- `millflow/src/hooks/useKeyboard.ts:94-101` - Creation mode keyboard guards
- `millflow/src/store/appStore.ts:24,106,296` - nodeCreationMode state
- `millflow/src/store/appStore.ts:426-502` - insertNode action
- `millflow/src/views/SheetView.tsx:72-80` - Creation handlers
- `millflow/src/views/SheetView.tsx:157-165` - NodeCreator overlay rendering
- `millflow/src/components/dag/NodeCreator.tsx` - Creation UI component

## Architecture Documentation

### DAG Rendering Pipeline

```
Sheet.nodes[]
    ↓
DAGRenderer.linearizeDAG() → RenderLevel[]
    ↓
For each level:
  - single → <DAGNode>
  - parallel → <ParallelBranch nodes={level.nodes}>
                    ↓
              <div className="flex flex-1">
                    ↓
              N × <DAGNode compact={true}>
```

### Node Creation State Machine

```
Initial: nodeCreationMode = null
    ↓ Press 'o' (if selectedNodeId set)
Creating: nodeCreationMode = 'after'
    ↓ Press Enter with name
Inserting: insertNode() → updates graph
    ↓ nodeCreationMode = null, select new node
Complete: nodeCreationMode = null
```

## Related Research

- Keyboard shortcuts fix handoff: `thoughts/shared/handoffs/general/2025-12-19_15-41-02_millflow-keyboard-fixes.md`
- Full implementation plan: `thoughts/shared/plans/2025-12-19-millflow-remaining-features.md`

## Open Questions

1. **Why might node creation "not work" for the user?**
   - If `selectedNodeId` is null (node not selected), the 'o' key does nothing
   - The user may need to navigate with j/k or click a node first to ensure selection

2. **How to improve parallel node display?**
   - Current behavior: equal-width distribution causes cramping with many nodes
   - Potential approaches: min-width constraints, horizontal scrolling, or responsive breakpoints
