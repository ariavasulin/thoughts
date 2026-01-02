---
date: 2025-12-19T04:13:35-08:00
researcher: Antigravity
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "Auto-Flex Grid System with Dynamic Padding"
tags: [research, grid-system, layout, reactflow, auto-flex]
status: complete
last_updated: 2025-12-19
last_updated_by: Antigravity
---

# Research: Auto-Flex Grid System with Dynamic Padding

**Date**: 2025-12-19T04:13:35-08:00
**Researcher**: Antigravity
**Git Commit**: b44e10ada59603fff73defcdafb24f122525e52c
**Branch**: main
**Repository**: dwsERP

## Research Question

How can the grid system auto-flex to align with each edge of each node, with automatic padding between coordinates? This should work on both Shop Pulse (Dashboard) and Factory Floor views.

## Summary

The current grid system uses **fixed constants** for node dimensions and gaps. To implement auto-flex with dynamic padding, the system needs to:

1. **Measure actual node dimensions** at runtime (nodes vary in height based on content)
2. **Calculate spacing dynamically** based on the largest node in each row/column
3. **Use ReactFlow's built-in layout capabilities** for collision avoidance

## Detailed Findings

### Current Grid System Architecture

**File**: `factory-floor/src/lib/gridSystem.ts`

The current implementation uses static constants:

```typescript
export const NODE_WIDTH = 256;        // Fixed width
export const NODE_HEIGHT = 280;       // Fixed height  
export const GRID_GAP = 80;           // Fixed gap
export const CELL_WIDTH = NODE_WIDTH + GRID_GAP;   // 336px
export const CELL_HEIGHT = NODE_HEIGHT + GRID_GAP; // 360px
```

**Problems with this approach:**
- `StationTableNode` is actually `w-[400px]` (line 19), not 256px
- Node heights vary based on number of sheets (0-300px scroll area)
- Static gaps don't adapt to content

### Node Dimension Discrepancies

| Component | Declared Width | Actual Width | Height Variability |
|-----------|---------------|--------------|-------------------|
| `OperationNode` | 256px (`w-64`) | 256px | ~100-150px |
| `StationTableNode` | 256px (grid constant) | **400px** | 80-400px (varies by content) |

**File**: `factory-floor/src/components/nodes/StationTableNode.tsx:19`
```typescript
className={clsx(
    "w-[400px] rounded-lg shadow-2xl border-4 overflow-hidden...",
```

### Views Using the Grid System

#### 1. Shop Pulse (Dashboard)
**File**: `factory-floor/src/components/dashboard/Dashboard.tsx:17-25`

```typescript
const STATION_POSITIONS = (Object.entries(STATION_GRID_MAP) as [StationId, GridPosition][])
  .map(([id, gridPos]) => {
    const { x, y } = getPixelPosition(gridPos.c, gridPos.r);
    return { id, label: STATION_TO_TYPE[id], x, y };
  });
```

Uses `StationTableNode` (400px wide) but grid assumes 256px nodes.

#### 2. Factory Floor (Editor)
**File**: `factory-floor/src/App.tsx:24-65`

Uses `dagre` for auto-layout with snap-to-grid:
```typescript
const getLayoutedElements = (nodes, edges) => {
    const dagreGraph = new dagre.graphlib.Graph();
    dagreGraph.setGraph({ rankdir: 'LR' }); // Left-to-right
    // ... snaps to grid after dagre positions
}
```

### Existing Auto-Layout via Dagre

**File**: `factory-floor/src/App.tsx:24-65`

The Factory Floor already uses `dagre` for automatic layout:
- Dagre calculates optimal positions
- Results are snapped to the semantic grid

This pattern could be extended to completely drive the grid dynamically.

### ReactFlow FitView Usage

**File**: `factory-floor/src/components/dashboard/Dashboard.tsx:95`
```typescript
<ReactFlow
    fitView
    minZoom={0.1}
    maxZoom={2}
    ...
>
```

**File**: `factory-floor/src/App.tsx:427`
```typescript
fitViewOptions={{ maxZoom: 1, padding: 0.2 }}
```

`fitView` already auto-scales to fit all nodes, but doesn't adjust spacing.

## Code References

- `factory-floor/src/lib/gridSystem.ts:1-15` - Static grid constants
- `factory-floor/src/lib/gridSystem.ts:40-44` - `getPixelPosition()` function
- `factory-floor/src/components/nodes/StationTableNode.tsx:19` - Actual 400px width
- `factory-floor/src/components/dashboard/Dashboard.tsx:17-25` - Dashboard node positioning
- `factory-floor/src/data/mockShopData.ts:144-173` - `STATION_GRID_MAP` grid coordinates
- `factory-floor/src/App.tsx:24-65` - Dagre auto-layout implementation

## Architecture Documentation

### Proposed Solution: Dynamic Grid Calculator

To implement auto-flex with dynamic padding:

#### Approach A: Measure-and-Position (Runtime Calculation)

```typescript
// Proposed new functions for gridSystem.ts

export interface DynamicGridConfig {
  minGap: number;          // Minimum space between nodes (e.g., 40px)
  paddingMultiplier: number; // e.g., 1.2 = 20% extra padding
}

export const calculateDynamicGrid = (
  nodes: Array<{ id: string; gridPos: GridPosition; width: number; height: number }>,
  config: DynamicGridConfig
): Map<string, PixelPosition> => {
  // 1. Group nodes by column and row
  // 2. Find max width per column, max height per row
  // 3. Calculate cumulative offsets
  // 4. Return positioned map
};
```

#### Approach B: Use Dagre for Everything

Let `dagre` handle all positioning with custom node sizes:

```typescript
nodes.forEach((node) => {
  // Use actual measured dimensions instead of constants
  const actualWidth = node.type === 'stationTable' ? 400 : 256;
  const actualHeight = measureNodeHeight(node); // or estimate
  dagreGraph.setNode(node.id, { width: actualWidth, height: actualHeight });
});
```

#### Approach C: CSS Flexbox/Grid in Container

Wrap nodes in a CSS Grid/Flexbox container and let the browser handle spacing:

```css
.node-container {
  display: grid;
  grid-template-columns: repeat(6, auto);
  gap: 40px;
  align-items: start;
}
```

**Limitation**: ReactFlow manages its own canvas, so this approach requires custom rendering.

## Related Research

- `thoughts/shared/plans/2025-12-19-semantic-grid-system.md` - Original grid system plan
- `thoughts/shared/research/2025-12-19-node-positioning-architecture.md` - Node positioning architecture

## Open Questions

1. **Should node dimensions be measured at runtime?** This would require refs and layout effects.
2. **Should dagre completely replace the semantic grid?** It's more flexible but less predictable.
3. **Should Shop Pulse and Factory Floor share the same layout algorithm?** Currently they differ.
4. **How should the grid handle nodes with collapsible content?** (e.g., StationTableNode with scrollable sheets)
