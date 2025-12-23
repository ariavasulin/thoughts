# Parallel Node Expand on Select Implementation Plan

## Overview

When navigating to a parallel row and selecting a node with J/K or H/L, the selected node expands to show full details while unselected siblings collapse to small icon-only indicators. The row maintains its dimensions, with the expanded node taking the available space.

## Current State Analysis

**ParallelBranch.tsx** renders all parallel nodes in compact mode side-by-side:
- Container width: 600px (2 nodes), 800px (3 nodes), 1000px (4+ nodes)
- All nodes use `flex-1` to share space equally
- All nodes receive `compact` prop, hiding most details

**DAGNode.tsx** compact mode:
- Shows: type icon, node name, blocker indicator
- Hides: status icon, assignees, dates, waiting-on text, suggestions
- Smaller padding: `px-3 py-2`

**Selection state** is already tracked and passed through:
- `selectedNodeId` passed to ParallelBranch
- `isSelected` prop passed to each DAGNode
- H/L navigation already works for moving between parallel nodes

## Desired End State

When a parallel node is selected:
1. Selected node renders at full width (non-compact) with all details
2. Unselected nodes render as small icon-only indicators
3. All nodes stay on the same horizontal row
4. Container maintains consistent dimensions
5. H/L navigation swaps which node is expanded
6. Smooth 100ms transitions between states

**Visual example (3 nodes, middle selected):**
```
[icon] ────────────[ Full Node Card ]──────────── [icon]
```

## What We're NOT Doing

- No vertical stacking or reflow
- No tooltips on collapsed indicators
- No animation of connector lines
- No changes to J/K (vertical) navigation logic
- No changes to single-node rendering

## Implementation Approach

1. Create a `CollapsedNodeIndicator` component for icon-only display
2. Modify `ParallelBranch` to conditionally render expanded vs collapsed nodes
3. Add smooth transitions using existing `transition-all duration-100` pattern

---

## Phase 1: Create CollapsedNodeIndicator Component

### Overview
A minimal icon-only button that represents a collapsed parallel node.

### Changes Required

#### 1. New Component
**File**: `millflow/src/components/dag/CollapsedNodeIndicator.tsx`

```tsx
import { memo } from 'react';
import type { Node } from '@/types';
import { nodeTypeIcons } from '@/lib/icons';
import { cn, statusBorderColors } from '@/lib/styles';
import { Lock } from 'lucide-react';

interface CollapsedNodeIndicatorProps {
  node: Node;
  onClick: () => void;
}

export const CollapsedNodeIndicator = memo(function CollapsedNodeIndicator({
  node,
  onClick,
}: CollapsedNodeIndicatorProps) {
  const TypeIcon = nodeTypeIcons[node.type];
  const borderColor = statusBorderColors[node.status];
  const isBlocked = node.status === 'blocked';
  const isDone = node.status === 'done';
  const isReady = node.status === 'ready';

  return (
    <button
      onClick={onClick}
      className={cn(
        'flex items-center justify-center',
        'w-10 h-10 shrink-0',
        'bg-gruvbox-bg-soft border rounded',
        borderColor,
        'border-l-3',
        'border-gruvbox-bg-2 hover:bg-gruvbox-bg-1 hover:border-gruvbox-bg-3',
        'transition-all duration-100'
      )}
      title={node.name}
    >
      {isBlocked ? (
        <Lock size={16} className="text-gruvbox-red-bright" />
      ) : (
        <TypeIcon
          size={16}
          className={cn(
            isDone ? 'text-gruvbox-green-bright' :
            isBlocked ? 'text-gruvbox-red-bright' :
            isReady ? 'text-gruvbox-blue-bright' :
            'text-gruvbox-fg-4'
          )}
        />
      )}
    </button>
  );
});

export default CollapsedNodeIndicator;
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd millflow && npm run build`
- [ ] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Component file exists at correct path
- [ ] (Will test integration in Phase 2)

---

## Phase 2: Modify ParallelBranch Layout

### Overview
Update ParallelBranch to render the selected node expanded and siblings as collapsed indicators.

### Changes Required

#### 1. Update ParallelBranch Component
**File**: `millflow/src/components/dag/ParallelBranch.tsx`

**Changes**:
- Import `CollapsedNodeIndicator`
- Check if any node in the branch is selected
- If selected: render mixed layout (expanded + collapsed indicators)
- If not selected: keep current compact behavior

```tsx
import { memo } from 'react';
import type { Node } from '@/types';
import DAGNode from './DAGNode';
import CollapsedNodeIndicator from './CollapsedNodeIndicator';

interface ParallelBranchProps {
  nodes: Node[];
  selectedNodeId: string | null;
  onSelectNode: (nodeId: string) => void;
}

export const ParallelBranch = memo(function ParallelBranch({
  nodes,
  selectedNodeId,
  onSelectNode,
}: ParallelBranchProps) {
  if (nodes.length === 0) return null;

  if (nodes.length === 1) {
    return (
      <DAGNode
        node={nodes[0]}
        isSelected={selectedNodeId === nodes[0].id}
        onClick={() => onSelectNode(nodes[0].id)}
      />
    );
  }

  // Check if any node in this branch is selected
  const selectedNode = nodes.find(n => n.id === selectedNodeId);
  const hasSelection = selectedNode !== undefined;

  // Container width based on node count
  const containerWidthClass =
    nodes.length === 2 ? 'w-[600px]' :
    nodes.length === 3 ? 'w-[800px]' :
    'w-[1000px]';

  return (
    <div className="relative flex justify-center">
      <div className={`relative ${containerWidthClass} max-w-[calc(100vw-4rem)]`}>
        {/* Split indicator line */}
        <div className="absolute top-0 left-0 right-0 h-px bg-gruvbox-bg-3" />

        {/* Parallel nodes */}
        <div className="flex gap-4 pt-4 items-start">
          {nodes.map((node) => {
            const isThisNodeSelected = node.id === selectedNodeId;

            return (
              <div
                key={node.id}
                className={cn(
                  'relative transition-all duration-100',
                  // If a selection exists in this branch:
                  // - Selected node gets flex-1 (takes remaining space)
                  // - Unselected nodes shrink to indicator size
                  // If no selection: all nodes share equally (current behavior)
                  hasSelection
                    ? isThisNodeSelected
                      ? 'flex-1 min-w-0'
                      : 'shrink-0'
                    : 'flex-1 min-w-48'
                )}
              >
                {/* Vertical connector */}
                <div className="absolute -top-4 left-1/2 -translate-x-1/2 w-px h-4 bg-gruvbox-bg-3" />

                {hasSelection ? (
                  isThisNodeSelected ? (
                    // Expanded: full node without compact mode
                    <DAGNode
                      node={node}
                      isSelected={true}
                      onClick={() => onSelectNode(node.id)}
                    />
                  ) : (
                    // Collapsed: icon-only indicator
                    <CollapsedNodeIndicator
                      node={node}
                      onClick={() => onSelectNode(node.id)}
                    />
                  )
                ) : (
                  // No selection in branch: compact mode for all
                  <DAGNode
                    node={node}
                    isSelected={false}
                    onClick={() => onSelectNode(node.id)}
                    compact
                  />
                )}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
});

export default ParallelBranch;
```

**Note**: Need to add `cn` import at top:
```tsx
import { cn } from '@/lib/styles';
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `cd millflow && npm run build`
- [ ] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Navigate to a sheet with parallel nodes
- [ ] When no node selected, all parallel nodes show in compact mode (current behavior)
- [ ] Press J/K to navigate into the parallel row
- [ ] Selected node expands to full width with all details
- [ ] Unselected nodes collapse to icon-only indicators
- [ ] Press H/L to move selection between parallel nodes
- [ ] Expansion/collapse swaps smoothly to the newly selected node
- [ ] Click on a collapsed indicator to select it (should expand)
- [ ] Transitions are smooth (100ms)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the feature works as expected before proceeding.

---

## Phase 3: Refine Visual Details

### Overview
Fine-tune the visual presentation based on testing.

### Potential Adjustments

#### 1. Connector Line Alignment
The vertical connector may need adjustment for collapsed indicators since they're smaller:

**File**: `millflow/src/components/dag/ParallelBranch.tsx`

If the connector looks off-center on collapsed indicators, we may need to conditionally position it:

```tsx
{/* Vertical connector - centered on element */}
<div className={cn(
  'absolute -top-4 w-px h-4 bg-gruvbox-bg-3',
  hasSelection && !isThisNodeSelected
    ? 'left-5 -translate-x-1/2'  // Center on 40px indicator
    : 'left-1/2 -translate-x-1/2'  // Center on full-width node
)} />
```

#### 2. Indicator Hover State
Consider adding a subtle ring on hover to match node cards:

**File**: `millflow/src/components/dag/CollapsedNodeIndicator.tsx`

```tsx
'hover:ring-1 hover:ring-gruvbox-bg-3'
```

#### 3. Height Consistency
Ensure collapsed indicators match the height of expanded nodes for visual alignment. Current indicator is `h-10` (40px). May need adjustment based on testing:

```tsx
// Match the approximate height of a compact node (py-2 = 8px top + 8px bottom + ~24px content)
'min-h-[40px]'
```

### Success Criteria

#### Automated Verification:
- [ ] Build still passes: `cd millflow && npm run build`

#### Manual Verification:
- [ ] Connector lines align properly with both expanded and collapsed nodes
- [ ] Hover states provide clear feedback
- [ ] Visual rhythm feels balanced across different node counts (2, 3, 4+ parallel)
- [ ] No layout jumps or visual glitches during transitions

---

## Testing Strategy

### Unit Tests
Not adding unit tests for this feature - it's primarily visual/layout changes.

### Manual Testing Steps

1. **Basic expand/collapse**:
   - Navigate to a sheet with 2+ parallel nodes
   - Use J/K to move into the parallel row
   - Verify selected node expands, others collapse
   - Use H/L to change selection within the row
   - Verify expansion swaps correctly

2. **Edge cases**:
   - Test with 2 parallel nodes
   - Test with 3 parallel nodes
   - Test with 4+ parallel nodes
   - Test clicking collapsed indicators
   - Test navigating away (J to next level) and back

3. **Visual quality**:
   - Transitions are smooth, not jarring
   - Connector lines look correct
   - Status colors show properly on collapsed indicators
   - Blocked nodes show lock icon in collapsed state

4. **Keyboard navigation intact**:
   - J/K still work for vertical navigation
   - H/L still work for horizontal navigation
   - Vim-style shortcuts (gg, G, etc.) still function

## Performance Considerations

- Using `memo` on both components to prevent unnecessary re-renders
- Transitions are CSS-only (no JS animation libraries)
- 100ms duration keeps interactions snappy

## References

- Research document: `thoughts/shared/research/2025-12-19-parallel-node-selection-expansion.md`
- Current ParallelBranch: `millflow/src/components/dag/ParallelBranch.tsx`
- Current DAGNode: `millflow/src/components/dag/DAGNode.tsx`
- Transition patterns: `millflow/src/index.css:68-79` (node-card transitions)
- Icon patterns: `millflow/src/lib/icons.ts`
