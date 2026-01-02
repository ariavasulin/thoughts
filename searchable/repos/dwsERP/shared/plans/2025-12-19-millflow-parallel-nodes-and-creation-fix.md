# MillFlow Parallel Node Display & Node Creation Fix

## Overview

Fix two issues in the MillFlow DAG view:
1. **Parallel nodes look cramped** - Multiple nodes at the same level are squished together with truncated text
2. **Node creation not working** - The 'o'/'O' keyboard shortcuts to insert nodes aren't functioning

## Current State Analysis

### Parallel Node Display (`ParallelBranch.tsx`)

Current implementation uses `flex gap-4` with `flex-1` on each node wrapper:
```tsx
<div className="flex gap-4 pt-4">
  {nodes.map((node) => (
    <div key={node.id} className="flex-1 relative">
```

This forces equal width distribution (e.g., 3 nodes = ~33% each), causing excessive truncation even when horizontal space is available.

### Node Creation

The system IS fully implemented:
- Keyboard bindings: `useKeyboard.ts:239-253` ('o' = after, 'O' = before)
- UI component: `NodeCreator.tsx` with name input + type picker
- Store action: `insertNode` at `appStore.ts:426-502`
- View integration: `SheetView.tsx:157-165`

The keyboard binding requires `selectedNodeId` to be set:
```typescript
if (view === 'sheet' && selectedSheetId && selectedNodeId) {
  setNodeCreationMode('after');
}
```

Recent fixes (handoff `2025-12-19_15-41-02`) sync `selectedNodeId` during navigation, but we should verify this works end-to-end.

## Desired End State

After implementation:
1. Parallel nodes are **centered** horizontally with reasonable widths
2. Nodes only truncate when truly necessary (more horizontal room utilized)
3. No horizontal scrolling
4. Node creation works via 'o' (insert after) and 'O' (insert before) keys

### Verification:
- `cd millflow && npm run build` passes
- `cd millflow && npm run dev` - visual inspection shows centered parallel nodes
- Pressing 'o' on a selected node opens the NodeCreator overlay

## What We're NOT Doing

- Horizontal scrolling for parallel nodes
- Changing the DAG linearization algorithm
- Adding drag-and-drop node creation
- Backend integration

---

## Phase 1: Fix Parallel Node Layout

### Overview
Center parallel nodes and give them more horizontal room before truncating.

### Changes Required:

#### 1. Update ParallelBranch Component
**File**: `millflow/src/components/dag/ParallelBranch.tsx`

Change the flex container from equal-width distribution to centered layout with constrained node widths.

**Current (line 34-46):**
```tsx
<div className="flex gap-4 pt-4">
  {nodes.map((node) => (
    <div key={node.id} className="flex-1 relative">
      {/* Vertical connector */}
      <div className="absolute -top-4 left-1/2 -translate-x-1/2 w-px h-4 bg-gruvbox-bg-3" />
      <DAGNode
        node={node}
        isSelected={selectedNodeId === node.id}
        onClick={() => onSelectNode(node.id)}
        compact
      />
    </div>
  ))}
</div>
```

**New:**
```tsx
<div className="flex justify-center gap-4 pt-4">
  {nodes.map((node) => (
    <div key={node.id} className="relative w-56 min-w-0">
      {/* Vertical connector */}
      <div className="absolute -top-4 left-1/2 -translate-x-1/2 w-px h-4 bg-gruvbox-bg-3" />
      <DAGNode
        node={node}
        isSelected={selectedNodeId === node.id}
        onClick={() => onSelectNode(node.id)}
        compact
      />
    </div>
  ))}
</div>
```

**Changes:**
- Add `justify-center` to center the node group
- Replace `flex-1` with `w-56` (224px) for consistent node width
- Add `min-w-0` to allow shrinking below content size if needed (prevents overflow)

#### 2. Update Split/Join Lines Width
**File**: `millflow/src/components/dag/ParallelBranch.tsx`

The split/join indicator lines need to span the actual width of the parallel nodes, not a fixed 80%. Calculate width based on node count.

**Current (lines 31, 50):**
```tsx
<div className="absolute top-0 left-1/2 -translate-x-1/2 w-4/5 h-px bg-gruvbox-bg-3" />
...
<div className="absolute bottom-0 left-1/2 -translate-x-1/2 w-4/5 h-px bg-gruvbox-bg-3" />
```

**New - Add dynamic width calculation:**
```tsx
// Calculate line width: (node_count * node_width) + ((node_count - 1) * gap)
// With w-56 (224px) and gap-4 (16px): 2 nodes = 464px, 3 nodes = 704px
const lineWidthClass = nodes.length === 2 ? 'w-[464px]' : nodes.length === 3 ? 'w-[704px]' : 'w-[944px]';

// In JSX:
<div className={`absolute top-0 left-1/2 -translate-x-1/2 ${lineWidthClass} max-w-full h-px bg-gruvbox-bg-3`} />
...
<div className={`absolute bottom-0 left-1/2 -translate-x-1/2 ${lineWidthClass} max-w-full h-px bg-gruvbox-bg-3`} />
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`
- [x] No TypeScript errors

#### Manual Verification:
- [x] Parallel nodes appear centered in the DAG view
- [x] Node names have more room before truncating (compact mode now shows only name + icons)
- [x] Split/join lines span the width of the parallel node group
- [x] No horizontal scrolling introduced
- [x] Single nodes (non-parallel) still render full-width

---

## Phase 2: Verify and Fix Node Creation

### Overview
Verify node creation works end-to-end. If issues are found, fix them.

### Verification Steps:

1. Run `cd millflow && npm run dev`
2. Navigate to a sheet (Dashboard → Job → Sheet)
3. Press `j` or `k` to navigate between nodes
4. Press `o` to insert a node after the selected node
5. Expected: NodeCreator overlay appears with name input

### Potential Issues to Check:

#### Issue A: `selectedNodeId` not set when entering sheet view
**Check**: In `appStore.ts:194-201`, verify `firstNodeId` is correctly obtained:
```typescript
const fullSheet = state.sheets.find(s => s.id === sheet.id);
const firstNodeId = fullSheet?.nodes[0]?.id || null;
```

If `fullSheet` is undefined, `firstNodeId` will be null and node creation won't work.

#### Issue B: NodeCreator not rendering
**Check**: In `SheetView.tsx:157`, the condition requires both:
```tsx
{nodeCreationMode && selectedNodeId && (
```

If either is falsy, the overlay won't show.

### Changes Required (if verification fails):

#### Fallback Fix: Ensure selectedNodeId is set on first render
**File**: `millflow/src/views/SheetView.tsx`

Add an effect to ensure `selectedNodeId` is set when entering sheet view:

```tsx
// Add after existing useEffect (around line 42-48)
useEffect(() => {
  // Ensure selectedNodeId is set when sheet view loads
  if (sheet && !selectedNodeId && sheet.nodes.length > 0) {
    selectNode(sheet.nodes[0].id);
  }
}, [sheet, selectedNodeId, selectNode]);
```

This ensures even if the navigation path didn't set `selectedNodeId`, it gets set on mount.

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [x] Navigate to sheet view via keyboard (Enter from job list)
- [x] Press 'o' - NodeCreator overlay appears
- [x] Type a name and press Enter - new node is inserted after selected node
- [x] Press 'O' (shift+o) - NodeCreator appears for "insert before"
- [x] Press Escape - NodeCreator closes without creating node
- [x] Navigate with j/k, then press 'o' - works on any selected node

---

## Testing Strategy

### Manual Testing Steps:
1. **Parallel node display**:
   - Navigate to sheet-012 (Reception Desk) which has 3 parallel nodes at one level
   - Verify nodes are centered and have readable text
   - Resize browser window to ensure no horizontal scroll appears

2. **Node creation flow**:
   - From dashboard, press Enter to go to a job
   - Press j/k to select a sheet, press Enter
   - In sheet view, press j/k to navigate nodes
   - Press 'o' to open NodeCreator
   - Type "Test Node", press Enter
   - Verify new node appears after the previously selected node
   - Press 'u' to undo - node should disappear

3. **Edge cases**:
   - Try 'o' on the last node in the DAG
   - Try 'O' on the first node in the DAG
   - Try creating multiple nodes in sequence

## References

- Research document: `thoughts/shared/research/2025-12-19-millflow-dag-rendering-node-creation.md`
- Keyboard fixes handoff: `thoughts/shared/handoffs/general/2025-12-19_15-41-02_millflow-keyboard-fixes.md`
- Implementation plan: `thoughts/shared/plans/2025-12-19-millflow-remaining-features.md`
