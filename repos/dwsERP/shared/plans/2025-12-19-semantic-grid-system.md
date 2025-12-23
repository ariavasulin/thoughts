# Semantic Grid System Implementation Plan

## Overview

We will implement a **Semantic Grid Abstraction System** for the `factory-floor` application. This system will map the continuous, arbitrary pixel coordinate system of React Flow to a discrete, integer-based grid `(row, column)`. This abstraction will allow both humans and AI agents to reasoning about layout in logical terms (e.g., "Place CNC at column 2, row 1") rather than pixel values.

## Current State Analysis

-   **Coordinates**: Nodes use raw floating-point pixel positions (e.g., `x: 124.5, y: 300.2`).
-   **Layout**: `dagre` calculates positions based on hardcoded constants ($256 \times 100$) and returns pixel values.
-   **Storage**: Coordinates are stored directly in the `Node` state.
-   **Grid**: Only a visual grid exists (`<Background />`), with no logical enforcement.

## Desired End State

-   **Semantic Coordinates**: Nodes can be described and positioned using integer Grid Coordinates `(col, row)`.
-   **Origin Centric**: The grid origin $(0,0)$ is conceptually at the center of the canvas.
-   **Snapping**: Dragged and dropped nodes automatically snap to the nearest valid grid cell.
-   **AI-Readable**: The state can be exported as a simple markdown table or JSON grid, making it trivial for LLMs to generate or modify layouts.

## What We're NOT Doing

-   We are **not** removing React Flow's internal coordinate system (it still needs pixels to render).
-   We are **not** rewriting the entire renderer; we are adding a translation layer.

## Implementation Approach

1.  **Define the Grid**: Create a dedicated configuration for cell dimensions and origin logic.
2.  **Math Layer**: Implement a bidirectional translation service (`toPixel` / `toGrid`).
3.  **Integration**: Hook this service into `App.tsx` (drag, drop, auto-layout).
4.  **Debugging**: Add visual indicators to verify the grid logic instancy.

---

## Phase 1: Core Grid Logic & Math

### Overview
Establish the mathematical foundation for the grid system in a standalone utility module.

### Changes Required:

#### 1. Create Grid Utility
**File**: `src/lib/gridSystem.ts`
**Description**:
-   Define constants: `CELL_WIDTH = 300` (allows for $256px$ node + gap), `CELL_HEIGHT = 160`.
-   Implement `GridPosition` interface `{ c: number, r: number }`.
-   Implement `gridToPixel(pos: GridPosition): { x, y }`.
-   Implement `pixelToGrid(x, y): GridPosition`.
-   Implement `snapToGrid(x, y): { x, y }` (helper for UI).

```typescript
export const CELL_WIDTH = 320; // 256px node + 64px gap
export const CELL_HEIGHT = 180; // ~100px node + gap

export const getGridPosition = (x: number, y: number) => {
  return {
    c: Math.round(x / CELL_WIDTH),
    r: Math.round(y / CELL_HEIGHT)
  };
};

export const getPixelPosition = (c: number, r: number) => {
  return {
    x: c * CELL_WIDTH,
    y: r * CELL_HEIGHT
  };
};
```

#### 2. Unit Tests
**File**: `src/lib/gridSystem.test.ts`
-   Verify round-trip layout (Grid -> Pixel -> Grid).
-   Verify snapping behavior for edge cases.

### Success Criteria:
-   [ ] `src/lib/gridSystem.ts` exists and exports core functions.
-   [ ] Unit tests pass for coordinate conversion.

---

## Phase 2: Editor Integration

### Overview
Apply the grid logic to the React Flow editor to enforce the grid system during user interaction.

### Changes Required:

#### 1. Update Node Type Definition
**File**: `src/components/nodes/OperationNode.tsx`
-   Extend `OperationNodeData` to optionally include `gridPos?: { c: number, r: number }` for debugging/persistence.

#### 2. Implement Snapping in Editor
**File**: `src/App.tsx`
-   **Drag & Drop**: In `onDrop`, use `snapToGrid` on the calculated position before creating the node.
-   **Node Dragging**: Use `onNodeDragStop` event to snap nodes to the grid when the user releases them.
    ```typescript
    const onNodeDragStop = (_: any, node: Node) => {
      const snapped = snapToGrid(node.position.x, node.position.y);
      // Update node position
    };
    ```

#### 3. Tokenize Auto-Layout
**File**: `src/App.tsx`
-   Update `getLayoutedElements` to:
    1.  Run `dagre` as before.
    2.  Map the results through `pixelToGrid` to find the nearest integer slot.
    3.  Map *back* via `gridToPixel` to set the final position.
    -   *Benefit*: This ensures auto-layout also respects the rigid grid structure.

### Success Criteria:
#### Automated:
-   [ ] App compiles without type errors.

#### Manual:
-   [ ] Dragging a node from the sidebar places it perfectly in a grid cell.
-   [ ] Moving a node snaps it to the grid on release.
-   [ ] "Tidy Up" (Auto-layout) results in a clean, grid-aligned graph.

---

## Phase 3: AI & Debug Tools

### Overview
Create tools to make this grid system visible to humans and readable by AI.

### Changes Required:

#### 1. Visual Debug Toggle
**File**: `src/App.tsx` / `OperationNode.tsx`
-   Add a "Show Grid" toggle in the UI.
-   When enabled, nodes display their `(c, r)` coordinates in a badge (e.g., `[0, 2]`).

#### 2. Grid Serialization
**File**: `src/lib/gridExport.ts`
-   Create `exportAsGridList(nodes)`: Returns a simple text representation.
    ```
    (0,0) [CNC] "Main"
    (1,0) [Assembly] "Table Leg"
    ```
-   Useful for pasting into LLM prompts.

### Success Criteria:
-   [ ] Can visually see coordinates on nodes.
-   [ ] Can generate a text-based grid map.

---

## Migration Notes
-   Existing mock data in `mockShopData.ts` simply uses pixels. We can run a one-time migration or just let them snap when touched.
-   No database migration needed (client-side only state).
