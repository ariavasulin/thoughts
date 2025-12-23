---
date: 2025-12-19T03:19:00-08:00
researcher: Antigravity
git_commit: b44e10a
branch: main
repository: dwsERP
topic: "Current Node Positioning & Layout Architecture"
tags: [research, codebase, layout, react-flow, dagre]
status: complete
last_updated: 2025-12-19
last_updated_by: Antigravity
---

# Research: Current Node Positioning & Layout Architecture

**Date**: 2025-12-19T03:19:00-08:00
**Researcher**: Antigravity
**Git Commit**: b44e10a
**Branch**: main
**Repository**: dwsERP

## Research Question
"Trying to find a way to have a good abstraction system such that I can exactly encode where the nodes are on the screen. Like, I need a grid system that is readable by AI..."

## Summary
This research documents the current implementation of node positioning, drag-and-drop logic, and automatic layout in the `factory-floor` application. Currently, the system relies on raw pixel coordinates managed by React Flow and `dagre`. There is no semantic grid abstraction; nodes are positioned using absolute pixel values (`x`, `y`) derived either from manual drop events or the Dagre layout engine.

## Detailed Findings

### 1. Coordinate System & Dimensions
The application uses a raw pixel coordinate system standard to React Flow.

-   **Node Dimensions**: Fixed at **256px** width and approx **100px** height.
    -   **Visuals**: defined in `src/components/nodes/OperationNode.tsx:60` via Tailwind class `w-64` (256px).
    -   **Layout Logic**: defined in `src/App.tsx:82-83` as constants `nodeWidth = 256` and `nodeHeight = 100`.
-   **Grid**: A visual-only grid is rendered via `<Background gap={24} />` in `App.tsx`, but this does not enforce logical snapping.

### 2. Layout Engine (Dagre)
Auto-layout is handled by `dagre` in `src/App.tsx`.

-   **Logic**: `getLayoutedElements` (`src/App.tsx:80`)
-   **Configuration**:
    -   `rankdir: 'LR'` (Left-to-Right).
    -   `dagre` calculates center points.
-   **Coordinate Transformation**:
    The system converts Dagre's center-based coordinates to React Flow's top-left coordinates:
    ```typescript
    // src/App.tsx:98
    x: nodeWithPosition.x - nodeWidth / 2,
    y: nodeWithPosition.y - nodeHeight / 2,
    ```

### 3. Drag and Drop Logic
Manual positioning uses screen-to-flow conversion.

-   **Logic**: `onDrop` (`src/App.tsx:212`)
-   **Calculation**:
    ```typescript
    // src/App.tsx:230
    const position = _reactFlowInstance.screenToFlowPosition({
        x: event.clientX,
        y: event.clientY,
    });
    ```
-   **Result**: This produces distinct floating-point coordinates (e.g., `x: 124.5, y: 300.2`) rather than integer grid coordinates.

### 4. Data Structures
Nodes are typed via `OperationNodeType` exporting a standard React Flow Node interface.

-   **Definition**: `src/components/nodes/OperationNode.tsx:17`
    ```typescript
    export type OperationNodeType = Node<OperationNodeData>;
    ```
-   **Node Data**:
    ```typescript
    type OperationNodeData = {
        label: string;
        type: 'CNC' | ...;
        // ...
    }
    ```
-   **Storage**: Nodes are stored in local state (`nodes` atom from `useNodesState`) inside `App.tsx`.

## Code References
-   `src/App.tsx:80-112`: `getLayoutedElements` - Core auto-layout logic.
-   `src/App.tsx:212-246`: `onDrop` - Manual positioning logic.
-   `src/components/nodes/OperationNode.tsx:60`: Node width definition (`w-64`).

## Architecture Documentation
The current architecture creates a tight coupling between the visual layer (React Flow) and the logical layer (Data). Positions are stored directly as UI coordinates. A "Semantic Grid" system would need to sit between these layers, translating `(row, col)` instructions into the `(x: px, y: px)` format that React Flow expects.

## Open Questions
-   How should variable height nodes (like `StationTableNode`) be handled in a rigid grid system?
-   Should the grid system replace Dagre entirely, or map Dagre's output to the nearest grid points?
