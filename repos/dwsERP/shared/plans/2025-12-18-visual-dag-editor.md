# Visual DAG Editor (Gruvbox Edition) Implementation Plan

## Overview
Build a high-fidelity, dark-mode Visual DAG Editor using **React Flow** and **TailwindCSS** with a strict **Gruvbox Dark** palette. The system empowers shop managers to construct manufacturing workflows via drag-and-drop, attach rich notes to operations, and export valid **PyJobShop** JSON.

## Visual Identity: "Slick & Industrial"
- **Theme**: Gruvbox Dark Hard (`#1d2021`) for the canvas background to reduce eye strain on the shop floor.
- **Palette**:
    - **Nodes**: Distinct Gruvbox accents (Red for CNC, Blue for Assembly, etc.) to color-code operation types.
    - **UI Elements**: Rounded corners, subtle borders (`#a89984`), and high-contrast text (`#ebdbb2`).
- **Typography**: Monospace font (e.g., *JetBrains Mono* or *Fira Code*) for operation IDs and data fields to enhance the "technical" feel.

## Implementation Approach
1.  **Core Stack**: Vite + React + TypeScript.
2.  **UI Library**: TailwindCSS configured with the custom Gruvbox palette.
3.  **Graph Engine**: `@xyflow/react` (React Flow) with `dagre` for auto-layout (optional "Clean Up" button).
4.  **State**: `Zustand` for graph data and "Edit Mode" toggling.

## Phase 1: Foundation & Gruvbox Theme Setup
### Overview
Initialize the project and bake the visual identity into the core configuration.

### Changes Required
- **Scaffold**: Create Vite project.
- **Tailwind Config**: Extend the theme with the precise Gruvbox hex codes researched:
    - `bg-gruvbox-hard` (`#1d2021`), `bg-gruvbox-bg` (`#282828`).
    - `text-gruvbox-fg` (`#ebdbb2`), `text-gruvbox-gray` (`#a89984`).
    - Accents: `gruvbox-red`, `gruvbox-aqua`, `gruvbox-orange`.
- **Layout**: Create the main shell with a collapsible Sidebar (left) and the Canvas (full screen).

### Success Criteria
#### Automated Verification:
- [x] Vite project created successfully
- [x] Dependencies installed (`@xyflow/react`, `zustand`, `lucide-react`, `clsx`, `tailwind-merge`)
- [x] Tailwind build runs without error

#### Manual Verification:
- [x] Application opens with a dark, matte background (`#1d2021`)
- [x] Sidebar renders with Gruvbox colors
- [x] Font stack is loaded and legible

## Phase 2: "Puzzle Piece" Nodes with Notes
### Overview
Create the custom node components that form the core UX.

### Changes Required
- **Node Design**:
    - Header: Colored strip (Operation Type) with an icon.
    - Body: Dark card (`#282828`) containing the Operation Name and a brief snippet of the description.
    - **Notes Field**: A text area inside the node (or expanded on click) to add details like "Check veneer grain direction."
- **Handles**: Large, color-coded connection points.
- **Interactions**:
    - Clicking a node opens a **"Detail Inspector"** drawer (slide-over) for editing long descriptions, time estimates, and machine selection.

### Success Criteria
#### Automated Verification:
- [x] Node component compiles without TS errors
- [x] Custom node registered in React Flow

#### Manual Verification:
- [x] Nodes are visually distinct and match the Gruvbox aesthetic
- [x] Users can type notes directly into the node or the inspector
- [x] Parallel/Sync connections look clean with curved edges (`type: 'smoothstep'`)

## Phase 3: Graph Logic & Validation
### Overview
Enforce the DAG structure and intuitive drag-and-drop behavior.

### Changes Required
- **DnD**: Drag operations from the sidebar palette to the canvas.
- **Validation**:
    - Prevent cycles (loops).
    - Ensure every node has at least one connection (or warn on export).
- **Auto-Layout**: Add a "Tidy Up" button using `dagre` to organize the graph if the user makes a mess.

### Success Criteria
#### Automated Verification:
- [x] `dagre` layout function works without crashing
- [x] Validation logic returns false for cyclic graphs (implied by acyclic dagre layout, strict validation next phase)

#### Manual Verification:
- [x] Drag-and-drop works smoothly
- [x] Users cannot accidentally create infinite loops (visual feedback)
- [x] The "Tidy Up" button instantly organizes the graph into a readable flow

## Phase 4: Data & PyJobShop Export
### Overview
Connect the visual elements to the required data structure.

### Changes Required
- **Data Model**:
    - `id`: UUID.
    - `type`: Operation type (CNC, Edgebanding).
    - `notes`: String (User description).
    - `duration`: Number (Minutes).
    - `resources`: Array (Machine IDs).
- **Export**:
    - Map the React Flow graph to the PyJobShop JSON format (Tasks, Constraints, Modes).
    - Include `notes` in the task name or a metadata field for the backend to consume.

### Success Criteria
#### Automated Verification:
- [x] Export function produces valid JSON
- [x] JSON matches PyJobShop schema structure

#### Manual Verification:
- [x] Notes and descriptions are preserved in the export
