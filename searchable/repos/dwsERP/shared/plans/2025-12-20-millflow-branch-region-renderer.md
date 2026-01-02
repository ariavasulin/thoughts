# Branch Region DAG Renderer Implementation Plan

## Overview

Refactor the millflow DAG rendering system to properly handle asymmetric parallel branches. Currently, the BFS level-based approach breaks down when branches have different depths (e.g., Milling → Veneer vs CNC alone). This plan introduces a "branch region" model that groups nodes between split and convergence points, rendering each branch as a vertical column.

## Current State Analysis

### What Works
- `splitNode` (s) - Creates parallel branches correctly
- `insertNode` (o/O) - Inserts nodes in linear chains
- `joinNodes` (J) - Converges selected nodes to a target
- `deleteNode` (d d) - Removes nodes, bridges gaps
- Marks for navigation (m/') - Useful for jumping around

### What's Broken
- `moveNodeUp/Down` (M k/j) - Tries to change DAG levels, creates invalid structures
- `moveNodeAfter/Before` (>/< marks) - Detaches nodes from children incorrectly
- `connectNodes/disconnectNodes` (c/C/x) - Arbitrary edge manipulation doesn't fit workflow model
- Level-based rendering - Asymmetric branches render incorrectly (nodes float to center)

### Key Files
- `millflow/src/lib/dag.ts` - BFS-based level building (to be refactored)
- `millflow/src/components/dag/DAGRenderer.tsx` - Level-based renderer (to be refactored)
- `millflow/src/components/dag/ParallelBranch.tsx` - Horizontal parallel nodes (to be replaced)
- `millflow/src/store/appStore.ts` - Contains broken move/connect operations
- `millflow/src/hooks/useKeyboard.ts` - Keyboard bindings for broken operations

## Desired End State

After implementation:
1. Asymmetric branches render correctly as vertical columns
2. Navigation (j/k/h/l) works intuitively within branch regions
3. `M k`/`M j` swaps nodes within a branch (reorder workflow steps)
4. Broken operations removed, clean command set
5. Visual style preserved (gruvbox colors, spacing, connector lines)

### Verification
- `cd millflow && npm run build` passes
- `cd millflow && npm run lint` passes
- Asymmetric DAG renders with branches as columns
- j/k moves within branch, h/l moves between branches
- M k/M j swaps node with predecessor/successor in same branch

## What We're NOT Doing

- Arbitrary edge creation (not a meaningful workflow operation)
- Cross-branch node moves (use split/delete/insert instead)
- Nested branch region rendering (keep it simple for v1)
- Drag-and-drop (keyboard-first focus)

---

## Phase 1: Branch Region Detection Algorithm

### Overview
Add algorithm to detect split→convergence regions and group nodes into branches.

### Changes Required:

#### 1. Add Branch Region Types
**File**: `millflow/src/lib/dag.ts`

Add after existing types:
```typescript
/**
 * A branch within a split region - a linear path of nodes.
 */
export interface Branch {
  id: string;
  nodes: Node[];  // Ordered from split to convergence
}

/**
 * A region between a split point and convergence point.
 */
export interface BranchRegion {
  type: 'split';
  splitNode: Node;
  convergenceNode: Node | null;  // null if branches never reconverge
  branches: Branch[];
}

/**
 * A single node not in a parallel region.
 */
export interface SingleRegion {
  type: 'single';
  node: Node;
}

export type DAGRegion = BranchRegion | SingleRegion;
```

#### 2. Implement Region Detection
**File**: `millflow/src/lib/dag.ts`

Add new function:
```typescript
/**
 * Build DAG regions for rendering - identifies split/convergence patterns.
 */
export function buildRegions(nodes: Node[]): DAGRegion[] {
  if (nodes.length === 0) return [];

  const nodeMap = new Map(nodes.map(n => [n.id, n]));
  const regions: DAGRegion[] = [];
  const processed = new Set<string>();

  // Find root nodes
  let roots = nodes.filter(n => n.parents.length === 0);
  if (roots.length === 0 && nodes.length > 0) {
    roots = [nodes[0]];
  }

  function processNode(nodeId: string): void {
    if (processed.has(nodeId)) return;

    const node = nodeMap.get(nodeId);
    if (!node) return;

    // Check if this is a split point (multiple children)
    if (node.children.length > 1) {
      const region = buildBranchRegion(node, nodeMap, processed);
      regions.push(region);

      // Continue from convergence point if exists
      if (region.convergenceNode) {
        processNode(region.convergenceNode.id);
      }
    } else {
      // Single node
      processed.add(nodeId);
      regions.push({ type: 'single', node });

      // Continue to children
      for (const childId of node.children) {
        processNode(childId);
      }
    }
  }

  for (const root of roots) {
    processNode(root.id);
  }

  return regions;
}

/**
 * Build a branch region starting from a split node.
 */
function buildBranchRegion(
  splitNode: Node,
  nodeMap: Map<string, Node>,
  processed: Set<string>
): BranchRegion {
  processed.add(splitNode.id);

  // Find convergence point using multi-path traversal
  const convergenceId = findConvergencePoint(splitNode, nodeMap);
  const convergenceNode = convergenceId ? nodeMap.get(convergenceId) || null : null;

  // Extract each branch
  const branches: Branch[] = splitNode.children.map((childId, index) => {
    const branchNodes: Node[] = [];
    let currentId: string | null = childId;

    while (currentId && currentId !== convergenceId) {
      const node = nodeMap.get(currentId);
      if (!node || processed.has(currentId)) break;

      processed.add(currentId);
      branchNodes.push(node);

      // Follow single-child path (stop at splits or convergence)
      if (node.children.length === 1) {
        currentId = node.children[0];
      } else {
        break;  // Stop at nested split or leaf
      }
    }

    return { id: `branch-${index}`, nodes: branchNodes };
  });

  return {
    type: 'split',
    splitNode,
    convergenceNode,
    branches,
  };
}

/**
 * Find where branches from a split point reconverge.
 */
function findConvergencePoint(
  splitNode: Node,
  nodeMap: Map<string, Node>
): string | null {
  const childIds = splitNode.children;
  if (childIds.length < 2) return null;

  // Track which branches have visited each node
  const visitedBy = new Map<string, Set<number>>();
  const queue: Array<{ nodeId: string; branchIdx: number }> = [];

  // Initialize with each child as a separate branch
  childIds.forEach((childId, idx) => {
    queue.push({ nodeId: childId, branchIdx: idx });
    visitedBy.set(childId, new Set([idx]));
  });

  while (queue.length > 0) {
    const { nodeId, branchIdx } = queue.shift()!;
    const node = nodeMap.get(nodeId);
    if (!node) continue;

    // Mark this branch as visiting this node
    if (!visitedBy.has(nodeId)) {
      visitedBy.set(nodeId, new Set());
    }
    visitedBy.get(nodeId)!.add(branchIdx);

    // Check if all branches have reached this node
    if (visitedBy.get(nodeId)!.size === childIds.length) {
      return nodeId;  // Found convergence
    }

    // Continue to children
    for (const childId of node.children) {
      if (!visitedBy.has(childId) || !visitedBy.get(childId)!.has(branchIdx)) {
        queue.push({ nodeId: childId, branchIdx });
        if (!visitedBy.has(childId)) {
          visitedBy.set(childId, new Set());
        }
        visitedBy.get(childId)!.add(branchIdx);
      }
    }
  }

  return null;  // No convergence found
}
```

#### 3. Add Linearization for Regions
**File**: `millflow/src/lib/dag.ts`

Add function for keyboard navigation order:
```typescript
/**
 * Linearize regions into a flat node list for navigation.
 * Order: split node, then each branch left-to-right depth-first, then convergence.
 */
export function linearizeRegions(regions: DAGRegion[]): Node[] {
  const result: Node[] = [];

  for (const region of regions) {
    if (region.type === 'single') {
      result.push(region.node);
    } else {
      // Split node first
      result.push(region.splitNode);

      // Each branch depth-first
      for (const branch of region.branches) {
        result.push(...branch.nodes);
      }

      // Convergence node last (if exists)
      if (region.convergenceNode) {
        result.push(region.convergenceNode);
      }
    }
  }

  return result;
}
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`
- [x] No TypeScript errors in dag.ts

#### Manual Verification:
- [ ] `buildRegions` correctly identifies split/convergence for test DAGs
- [ ] Asymmetric branches are captured with correct node counts per branch

---

## Phase 2: Branch Region Renderer

### Overview
Create new renderer component that displays branches as vertical columns.

### Changes Required:

#### 1. Create BranchRegion Component
**File**: `millflow/src/components/dag/BranchRegionView.tsx` (new file)

```typescript
import { memo } from 'react';
import type { BranchRegion } from '@/lib/dag';
import DAGNode from './DAGNode';

interface BranchRegionViewProps {
  region: BranchRegion;
  selectedNodeId: string | null;
  onSelectNode: (nodeId: string) => void;
}

export const BranchRegionView = memo(function BranchRegionView({
  region,
  selectedNodeId,
  onSelectNode,
}: BranchRegionViewProps) {
  const { splitNode, convergenceNode, branches } = region;

  // Calculate max branch depth for consistent heights
  const maxDepth = Math.max(...branches.map(b => b.nodes.length));

  // Container width based on branch count
  const containerWidth = branches.length === 2 ? 'w-[600px]' :
                         branches.length === 3 ? 'w-[800px]' : 'w-[1000px]';

  return (
    <div className="space-y-3">
      {/* Split node */}
      <div className="max-w-2xl mx-auto">
        <DAGNode
          node={splitNode}
          isSelected={selectedNodeId === splitNode.id}
          onClick={() => onSelectNode(splitNode.id)}
        />
      </div>

      {/* Branch columns */}
      <div className="flex justify-center">
        <div className={`relative ${containerWidth} max-w-[calc(100vw-4rem)]`}>
          {/* Horizontal split line */}
          <div className="absolute top-0 left-0 right-0 h-px bg-gruvbox-bg-3" />

          {/* Branch columns */}
          <div className="flex gap-4 pt-4">
            {branches.map((branch) => (
              <div key={branch.id} className="flex-1 min-w-48 space-y-3">
                {/* Vertical connector from split line */}
                <div className="flex justify-center -mt-4 mb-3">
                  <div className="w-px h-4 bg-gruvbox-bg-3" />
                </div>

                {/* Branch nodes stacked vertically */}
                {branch.nodes.map((node, idx) => (
                  <div key={node.id}>
                    {idx > 0 && (
                      <div className="flex justify-center mb-3">
                        <div className="w-px h-4 bg-gruvbox-bg-3" />
                      </div>
                    )}
                    <DAGNode
                      node={node}
                      isSelected={selectedNodeId === node.id}
                      onClick={() => onSelectNode(node.id)}
                      compact
                    />
                  </div>
                ))}

                {/* Connector to convergence (if branch is shorter than max) */}
                {convergenceNode && branch.nodes.length < maxDepth && (
                  <div className="flex justify-center">
                    <div
                      className="w-px bg-gruvbox-bg-3"
                      style={{ height: `${(maxDepth - branch.nodes.length) * 4}rem` }}
                    />
                  </div>
                )}

                {/* Bottom connector to convergence line */}
                {convergenceNode && (
                  <div className="flex justify-center">
                    <div className="w-px h-4 bg-gruvbox-bg-3" />
                  </div>
                )}
              </div>
            ))}
          </div>

          {/* Horizontal convergence line */}
          {convergenceNode && (
            <div className="absolute bottom-0 left-0 right-0 h-px bg-gruvbox-bg-3" />
          )}
        </div>
      </div>

      {/* Convergence node */}
      {convergenceNode && (
        <>
          <div className="flex justify-center">
            <div className="w-px h-4 bg-gruvbox-bg-3" />
          </div>
          <div className="max-w-2xl mx-auto">
            <DAGNode
              node={convergenceNode}
              isSelected={selectedNodeId === convergenceNode.id}
              onClick={() => onSelectNode(convergenceNode.id)}
            />
          </div>
        </>
      )}
    </div>
  );
});

export default BranchRegionView;
```

#### 2. Update DAGRenderer to Use Regions
**File**: `millflow/src/components/dag/DAGRenderer.tsx`

Replace content:
```typescript
import { memo, useMemo } from 'react';
import type { Sheet } from '@/types';
import { buildRegions, linearizeRegions } from '@/lib/dag';
import DAGNode from './DAGNode';
import BranchRegionView from './BranchRegionView';

interface DAGRendererProps {
  sheet: Sheet;
  selectedNodeId: string | null;
  selectedIndex: number;
  onSelectNode: (nodeId: string) => void;
}

export const DAGRenderer = memo(function DAGRenderer({
  sheet,
  selectedNodeId,
  onSelectNode,
}: DAGRendererProps) {
  const regions = useMemo(() => buildRegions(sheet.nodes), [sheet.nodes]);

  const flatNodes = useMemo(() => linearizeRegions(regions), [regions]);
  const effectiveSelectedId = selectedNodeId || flatNodes[0]?.id;

  return (
    <div className="space-y-3">
      {regions.map((region, idx) => (
        <div key={idx}>
          {/* Connector from previous region */}
          {idx > 0 && (
            <div className="flex justify-center mb-3">
              <div className="w-px h-4 bg-gruvbox-bg-3" />
            </div>
          )}

          {region.type === 'single' ? (
            <div className="max-w-2xl mx-auto">
              <DAGNode
                node={region.node}
                isSelected={effectiveSelectedId === region.node.id}
                onClick={() => onSelectNode(region.node.id)}
              />
            </div>
          ) : (
            <BranchRegionView
              region={region}
              selectedNodeId={effectiveSelectedId}
              onSelectNode={onSelectNode}
            />
          )}
        </div>
      ))}
    </div>
  );
});

export default DAGRenderer;
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Asymmetric branches render as columns with correct depths
- [ ] Connector lines are straight and properly aligned
- [ ] Visual style matches current (gruvbox colors, spacing)
- [ ] Selection highlighting works on all nodes

---

## Phase 3: Updated Keyboard Navigation

### Overview
Update navigation to work with branch-region structure.

### Changes Required:

#### 1. Update Navigation Actions
**File**: `millflow/src/store/appStore.ts`

Update `navigateDown` and `navigateUp` to use linearized regions:
```typescript
navigateDown: () => {
  const { view, selectedNodeId, selectedSheetId, sheets, selectedIndex } = get();

  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    const sheet = sheets.find(s => s.id === selectedSheetId);
    if (!sheet) return;

    const regions = buildRegions(sheet.nodes);
    const flatNodes = linearizeRegions(regions);
    const currentIdx = flatNodes.findIndex(n => n.id === selectedNodeId);

    if (currentIdx < flatNodes.length - 1) {
      const nextNode = flatNodes[currentIdx + 1];
      set({
        selectedNodeId: nextNode.id,
        selectedIndex: currentIdx + 1,
      });
    }
    return;
  }

  // Fallback for other views...
  set(state => ({ selectedIndex: state.selectedIndex + 1 }));
},

navigateUp: () => {
  const { view, selectedNodeId, selectedSheetId, sheets } = get();

  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    const sheet = sheets.find(s => s.id === selectedSheetId);
    if (!sheet) return;

    const regions = buildRegions(sheet.nodes);
    const flatNodes = linearizeRegions(regions);
    const currentIdx = flatNodes.findIndex(n => n.id === selectedNodeId);

    if (currentIdx > 0) {
      const prevNode = flatNodes[currentIdx - 1];
      set({
        selectedNodeId: prevNode.id,
        selectedIndex: currentIdx - 1,
      });
    }
    return;
  }

  // Fallback for other views...
  set(state => ({ selectedIndex: Math.max(0, state.selectedIndex - 1) }));
},
```

#### 2. Update Left/Right Navigation for Branches
**File**: `millflow/src/store/appStore.ts`

Update `navigateLeft` and `navigateRight`:
```typescript
navigateLeft: () => {
  const { view, selectedNodeId, selectedSheetId, sheets } = get();
  if (view !== 'sheet' || !selectedSheetId || !selectedNodeId) return;

  const sheet = sheets.find(s => s.id === selectedSheetId);
  if (!sheet) return;

  const regions = buildRegions(sheet.nodes);

  // Find which branch region contains the current node
  for (const region of regions) {
    if (region.type !== 'split') continue;

    for (let branchIdx = 0; branchIdx < region.branches.length; branchIdx++) {
      const branch = region.branches[branchIdx];
      const nodeIdx = branch.nodes.findIndex(n => n.id === selectedNodeId);

      if (nodeIdx !== -1 && branchIdx > 0) {
        // Move to same position in left branch (or closest)
        const leftBranch = region.branches[branchIdx - 1];
        const targetIdx = Math.min(nodeIdx, leftBranch.nodes.length - 1);
        if (targetIdx >= 0) {
          const targetNode = leftBranch.nodes[targetIdx];
          const flatNodes = linearizeRegions(regions);
          const flatIdx = flatNodes.findIndex(n => n.id === targetNode.id);
          set({
            selectedNodeId: targetNode.id,
            selectedIndex: flatIdx,
          });
        }
        return;
      }
    }
  }
},

navigateRight: () => {
  const { view, selectedNodeId, selectedSheetId, sheets } = get();
  if (view !== 'sheet' || !selectedSheetId || !selectedNodeId) return;

  const sheet = sheets.find(s => s.id === selectedSheetId);
  if (!sheet) return;

  const regions = buildRegions(sheet.nodes);

  // Find which branch region contains the current node
  for (const region of regions) {
    if (region.type !== 'split') continue;

    for (let branchIdx = 0; branchIdx < region.branches.length; branchIdx++) {
      const branch = region.branches[branchIdx];
      const nodeIdx = branch.nodes.findIndex(n => n.id === selectedNodeId);

      if (nodeIdx !== -1 && branchIdx < region.branches.length - 1) {
        // Move to same position in right branch (or closest)
        const rightBranch = region.branches[branchIdx + 1];
        const targetIdx = Math.min(nodeIdx, rightBranch.nodes.length - 1);
        if (targetIdx >= 0) {
          const targetNode = rightBranch.nodes[targetIdx];
          const flatNodes = linearizeRegions(regions);
          const flatIdx = flatNodes.findIndex(n => n.id === targetNode.id);
          set({
            selectedNodeId: targetNode.id,
            selectedIndex: flatIdx,
          });
        }
        return;
      }
    }
  }
},
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] j/k navigates through linearized order (depth-first through branches)
- [ ] h/l moves between branches at same position
- [ ] Navigation wraps correctly at boundaries
- [ ] Selected node highlight follows navigation

---

## Phase 4: Swap Operation (Reorder Within Branch)

### Overview
Replace broken move operations with a simple swap that reorders nodes within a branch.

### Changes Required:

#### 1. Add Swap Action
**File**: `millflow/src/store/appStore.ts`

Replace `moveNodeUp/Down` with:
```typescript
// Replace moveNodeUp/Down with swapNodeUp/Down
swapNodeUp: (sheetId, nodeId) => {
  const sheet = get().sheets.find(s => s.id === sheetId);
  if (!sheet) return;

  const regions = buildRegions(sheet.nodes);

  // Find the branch containing this node
  for (const region of regions) {
    if (region.type !== 'split') continue;

    for (const branch of region.branches) {
      const nodeIdx = branch.nodes.findIndex(n => n.id === nodeId);
      if (nodeIdx > 0) {
        // Found it, swap with previous
        const prevNode = branch.nodes[nodeIdx - 1];
        get().pushHistory('Swap node up');

        set(state => ({
          sheets: state.sheets.map(s => {
            if (s.id !== sheetId) return s;

            return {
              ...s,
              nodes: s.nodes.map(n => {
                if (n.id === nodeId) {
                  // Current node takes prev's position
                  return { ...n, parents: [...prevNode.parents], children: [prevNode.id] };
                }
                if (n.id === prevNode.id) {
                  // Prev node takes current's position
                  return { ...n, parents: [nodeId], children: [...n.children.filter(c => c !== nodeId)] };
                }
                // Update other nodes that referenced these
                return {
                  ...n,
                  parents: n.parents.map(p => p === prevNode.id ? nodeId : p === nodeId ? prevNode.id : p),
                  children: n.children.map(c => c === prevNode.id ? nodeId : c === nodeId ? prevNode.id : c),
                };
              }),
            };
          }),
        }));
        return;
      }
    }
  }
},

swapNodeDown: (sheetId, nodeId) => {
  const sheet = get().sheets.find(s => s.id === sheetId);
  if (!sheet) return;

  const regions = buildRegions(sheet.nodes);

  // Find the branch containing this node
  for (const region of regions) {
    if (region.type !== 'split') continue;

    for (const branch of region.branches) {
      const nodeIdx = branch.nodes.findIndex(n => n.id === nodeId);
      if (nodeIdx !== -1 && nodeIdx < branch.nodes.length - 1) {
        // Found it, swap with next
        const nextNode = branch.nodes[nodeIdx + 1];
        get().pushHistory('Swap node down');

        set(state => ({
          sheets: state.sheets.map(s => {
            if (s.id !== sheetId) return s;

            return {
              ...s,
              nodes: s.nodes.map(n => {
                if (n.id === nodeId) {
                  // Current node takes next's position
                  return { ...n, parents: [nextNode.id], children: [...nextNode.children] };
                }
                if (n.id === nextNode.id) {
                  // Next node takes current's position
                  return { ...n, parents: [...n.parents.filter(p => p !== nodeId)], children: [nodeId] };
                }
                // Update other nodes that referenced these
                return {
                  ...n,
                  parents: n.parents.map(p => p === nextNode.id ? nodeId : p === nodeId ? nextNode.id : p),
                  children: n.children.map(c => c === nextNode.id ? nodeId : c === nodeId ? nextNode.id : c),
                };
              }),
            };
          }),
        }));
        return;
      }
    }
  }
},
```

#### 2. Update Keyboard Handler
**File**: `millflow/src/hooks/useKeyboard.ts`

Replace M k/M j handlers:
```typescript
// M k or M ArrowUp - swap node up in branch
if (sequence === 'M k' || sequence === 'M ArrowUp') {
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    swapNodeUp(selectedSheetId, selectedNodeId);
  }
  return;
}

// M j or M ArrowDown - swap node down in branch
if (sequence === 'M j' || sequence === 'M ArrowDown') {
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    swapNodeDown(selectedSheetId, selectedNodeId);
  }
  return;
}
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] M k swaps node with previous in same branch
- [ ] M j swaps node with next in same branch
- [ ] Swap at branch boundaries does nothing (no-op)
- [ ] Undo reverses swap correctly

---

## Phase 5: Cleanup Broken Operations

### Overview
Remove connect/disconnect/move operations that don't fit the branch-region model.

### Changes Required:

#### 1. Remove From Store Interface
**File**: `millflow/src/store/appStore.ts`

Remove these lines from AppState interface (~lines 108-116):
```typescript
// DELETE THESE:
// Edge Actions (non-destructive)
connectNodes: (fromNodeId: string, toNodeId: string) => void;
disconnectNodes: (nodeId1: string, nodeId2: string) => void;

// Move Actions
moveNodeUp: (sheetId: string, nodeId: string) => void;
moveNodeDown: (sheetId: string, nodeId: string) => void;
moveNodeAfter: (sheetId: string, nodeId: string, targetNodeId: string) => void;
moveNodeBefore: (sheetId: string, nodeId: string, targetNodeId: string) => void;
```

Also remove `pendingMarkAction` from state (keep `nodeMarks` for navigation).

#### 2. Remove Action Implementations
**File**: `millflow/src/store/appStore.ts`

Delete these function implementations:
- `connectNodes` (~lines 1076-1127)
- `disconnectNodes` (~lines 1129-1165)
- `moveNodeUp` (~lines 1168-1208)
- `moveNodeDown` (~lines 1210-1261)
- `moveNodeAfter` (~lines 1263-1309)
- `moveNodeBefore` (~lines 1311-1361)
- `setPendingMarkAction`

#### 3. Remove Keyboard Handlers
**File**: `millflow/src/hooks/useKeyboard.ts`

Remove these handlers:
- `c {a-z}` connect from mark
- `C {a-z}` connect to mark
- `x {a-z}` disconnect
- `> {a-z}` move after mark
- `< {a-z}` move before mark

Remove from sequence starters list (~line 316):
```typescript
// Change from:
if (['g', 'd', 'm', "'", 'x', 'M', '>', '<'].includes(key) ...
// To:
if (['g', 'd', 'm', "'", 'M'].includes(key) ...
```

#### 4. Update Help Modal
**File**: `millflow/src/components/HelpModal.tsx`

Update shortcuts to reflect new behavior:
- Remove "Marks & Connect" section (keep marks in navigation section)
- Update "DAG Editing" section:
  - Change `M k/↑` to "Swap node earlier in branch"
  - Change `M j/↓` to "Swap node later in branch"
  - Remove `> {a-z}` and `< {a-z}`

#### 5. Update ShortcutBar
**File**: `millflow/src/components/ShortcutBar.tsx`

Update sheet shortcuts:
```typescript
const sheetShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'nav' },
  { key: 'h/l', label: 'branches' },
  { key: 'o/O', label: 'insert' },
  { key: 's', label: 'split' },
  { key: 'M', label: 'swap' },  // Changed from 'move'
  { key: 'm', label: 'mark' },
  { key: '?', label: 'help' },
];
```

Remove `markPendingShortcuts` for c/C/x and update M pending to show swap.

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`
- [x] No unused imports or dead code warnings

#### Manual Verification:
- [ ] c/C/x keys do nothing (or revert to other behavior)
- [ ] >/< keys do nothing
- [ ] Help modal shows updated shortcuts
- [ ] ShortcutBar shows correct context hints

---

## Phase 6: Delete ParallelBranch Component

### Overview
Remove the now-unused ParallelBranch component.

### Changes Required:

#### 1. Delete File
**File**: `millflow/src/components/dag/ParallelBranch.tsx`

Delete this file entirely.

#### 2. Remove Imports
Ensure no files still import ParallelBranch.

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`

---

## Testing Strategy

### Unit Tests (if added later):
- `buildRegions` correctly identifies split/convergence patterns
- `linearizeRegions` produces correct node order
- `swapNodeUp/Down` maintains DAG integrity

### Manual Testing Steps:
1. Create asymmetric branches: Sheet → split → (Milling → Veneer) and (CNC) → Assembly
2. Verify branches render as columns with different depths
3. Navigate with j/k through linearized order
4. Navigate with h/l between branches
5. Use M k/M j to swap nodes within a branch
6. Verify undo works for all operations
7. Test edge cases: single-node branches, deeply nested

### Edge Cases to Test:
- Single node (no splits)
- All branches same depth
- Branches that never reconverge (leaf branches)
- Node at split point (can't swap up)
- Node at convergence point (can't swap down)

## Performance Considerations

- `buildRegions` is O(n) where n = nodes in sheet
- Called on every navigation, but sheets are small (<100 nodes typically)
- Could memoize in store if performance becomes an issue

## References

- Research from earlier: Branch detection algorithm, visual styling patterns, keyboard navigation analysis
- Current files: `millflow/src/lib/dag.ts`, `millflow/src/components/dag/DAGRenderer.tsx`
- Gruvbox palette: `millflow/tailwind.config.js`
