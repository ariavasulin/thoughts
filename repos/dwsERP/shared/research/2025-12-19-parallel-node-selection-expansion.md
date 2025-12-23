---
date: 2025-12-19T23:25:26-08:00
researcher: ariasulin
git_commit: 201f79e68db6ef380d44c9b4d64ee7e34b507206
branch: main
repository: dwsERP
topic: "Parallel node selection and expansion to full width"
tags: [research, codebase, dag, node-rendering, keyboard-navigation, selection]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Parallel Node Selection and Expansion to Full Width

**Date**: 2025-12-19T23:25:26-08:00
**Researcher**: ariasulin
**Git Commit**: 201f79e68db6ef380d44c9b4d64ee7e34b507206
**Branch**: main
**Repository**: dwsERP

## Research Question

When navigating to a row with multiple parallel nodes and selecting a node with J/K, the selected node should expand to full column width and show all information (like a full-sized single node).

## Summary

Currently, parallel nodes are rendered in "compact" mode which:
1. Shares horizontal space equally between nodes via flexbox (`flex-1`)
2. Hides most node details (assignees, dates, status, waiting-on text)
3. Uses smaller padding

Single nodes render in "regular" mode with:
1. Full width (max 672px centered)
2. All node details visible
3. Standard padding

The selection system already tracks `selectedNodeId` and passes `isSelected` to all nodes (including parallel ones), but this prop only affects visual highlighting - not layout or content display.

## Detailed Findings

### Current Parallel Branch Rendering

**ParallelBranch.tsx:28-55** controls the container sizing:
```typescript
const containerWidthClass =
  nodes.length === 2 ? 'w-[600px]' :
  nodes.length === 3 ? 'w-[800px]' :
  'w-[1000px]';
```

Each node gets `flex-1 min-w-48` (equal sharing, 192px minimum).

**ParallelBranch.tsx:44-49** renders nodes with `compact` prop:
```typescript
<DAGNode
  node={node}
  isSelected={selectedNodeId === node.id}
  onClick={() => onSelectNode(node.id)}
  compact  // <-- Always compact for parallel nodes
/>
```

### Compact Mode Differences in DAGNode

**Padding** (`DAGNode.tsx:40`):
- Compact: `px-3 py-2` (12px horizontal, 8px vertical)
- Regular: `px-4 py-3` (16px horizontal, 12px vertical)

**Hidden content in compact mode**:
- Status icon, assignees, dates (`DAGNode.tsx:84` - `!compact` check)
- "Waiting on" indicator (`DAGNode.tsx:131`)
- Suggested assignee text (`DAGNode.tsx:138`)

**Visible in both modes**:
- Selection indicator `>`
- Type icon
- Node name
- Blocker lock icon (if blocked)

### Selection State Flow

1. J/K key press → `useKeyboard.ts:210-220`
2. `navigateDown()/navigateUp()` → `appStore.ts:157-212`
3. Uses `buildLevels()` and `findNodePosition()` for DAG-aware navigation
4. Updates `selectedNodeId` in Zustand store
5. `SheetView` passes `selectedNodeId` to `DAGRenderer`
6. `DAGRenderer` passes to `ParallelBranch` → individual `DAGNode`
7. `isSelected` prop triggers visual highlighting only

### Single Node Rendering (Target State)

**DAGRenderer.tsx:80-86** renders single nodes:
```typescript
<div className="max-w-2xl mx-auto">
  <DAGNode
    node={level.nodes[0]}
    isSelected={effectiveSelectedId === level.nodes[0].id}
    onClick={() => onSelectNode(level.nodes[0].id)}
    // No compact prop - uses full display
  />
</div>
```

## Code References

- `millflow/src/components/dag/ParallelBranch.tsx:28-55` - Parallel container sizing and node rendering
- `millflow/src/components/dag/DAGNode.tsx:40,84,131,138` - Compact mode conditional display
- `millflow/src/components/dag/DAGRenderer.tsx:80-86` - Single node full-width rendering
- `millflow/src/store/appStore.ts:157-212` - Navigation state management
- `millflow/src/hooks/useKeyboard.ts:210-220` - J/K key handlers

## Architecture Documentation

**Current layout model**:
- Levels stack vertically with `space-y-3`
- Single levels: `max-w-2xl mx-auto` (672px max, centered)
- Parallel levels: Fixed container (600/800/1000px), nodes share via `flex-1`

**Selection model**:
- `selectedNodeId` tracked in Zustand
- `isSelected` boolean prop passed to all nodes
- Affects visual styling (ring, background, `>` indicator)
- Does NOT currently affect layout or content display

## Implementation Considerations

To achieve the requested behavior, `ParallelBranch` would need to:
1. Know which node is selected via `selectedNodeId` prop (already passed)
2. Render selected node differently - full width, non-compact
3. Potentially collapse or hide unselected siblings while one is expanded
4. Handle smooth transitions between compact/expanded states

Key decision points:
- Should unselected nodes still be visible when one is expanded?
- Should the expansion animate or be instant?
- Should H/L navigation collapse the current node and expand the new one?

## Open Questions

1. What should happen to non-selected parallel nodes when one expands? Options:
   - Hide completely
   - Show as minimal indicators/chips
   - Keep visible but dimmed
   - Stack vertically below the expanded node

2. Should H/L (left/right) navigation:
   - Collapse current → expand new (swap which is full-width)
   - Just change selection without collapsing

3. Should there be animation for the expand/collapse transition?
