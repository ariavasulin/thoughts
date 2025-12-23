---
date: 2025-12-19T17:14:13-08:00
researcher: ariasulin
git_commit: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
branch: main
repository: dwsERP
topic: "Millflow Data Manipulation and State Management - Current Implementation Status"
tags: [research, millflow, zustand, data-manipulation, nodes, blockers, forms]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Millflow Data Manipulation and State Management - Current Implementation Status

**Date**: 2025-12-19T17:14:13-08:00
**Researcher**: ariasulin
**Git Commit**: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
**Branch**: main
**Repository**: dwsERP

## Research Question

Document the current state of data manipulation functionality in Millflow - specifically what's wired up vs what's not for: adding nodes, adding blockers, removing things, editing nodes, and updating data. The goal is to understand what needs to be connected so local data becomes fully modifiable.

## Summary

Millflow has a well-structured Zustand store with comprehensive node manipulation capabilities. **Most create operations are fully wired** (adding nodes, blockers, notes, templates). However, **update and delete operations for supporting entities are incomplete**. The application persists all changes to an in-memory Zustand store with full undo/redo history, but data resets on page refresh (no localStorage/backend persistence).

### What's Fully Wired

| Operation | Store Method | UI Trigger |
|-----------|--------------|------------|
| Insert node (before/after) | `insertNode()` | SmartNodeCreator modal |
| Insert template sequence | `insertTemplate()` | TemplatePicker modal |
| Split into parallel branch | `splitNode()` | `s` keyboard shortcut |
| Delete node | `deleteNode()` | `dd` keyboard shortcut |
| Update node (generic) | `updateNode()` | DueDatePicker form |
| Assign node | `assignNode()` | AssigneePicker form |
| Add blocker | `addBlocker()` | SmartBlockerForm |
| Add note | `addNote()` | NoteForm |
| Toggle status | `toggleNodeStatus()` | Spacebar |
| Undo/Redo | `undo()`, `redo()` | `u`, `Ctrl+r` keys |

### What's NOT Wired

| Missing Operation | Entity | Notes |
|-------------------|--------|-------|
| Edit blocker | Blocker | No form, no store method |
| Delete blocker | Blocker | No button, no store method |
| Resolve blocker | Blocker | No mechanism to mark resolved |
| Edit note | Note | No form, no store method |
| Delete note | Note | No button, no store method |
| Create job | Job | Read-only from mock data |
| Create sheet | Sheet | Read-only from mock data |
| Edit sheet metadata | Sheet | No form, no store method |
| Create/edit person | Person | Read-only from mock data |
| Join parallel nodes | Node | `joinNodes()` exists but no UI |
| Clear blocked status | Node | Auto-set but not auto-cleared |

---

## Detailed Findings

### 1. Zustand Store Architecture (`millflow/src/store/appStore.ts`)

The store manages all application state with an undo/redo history system.

**State Structure:**
- **Navigation**: `view`, `selectedJobId`, `selectedSheetId`, `selectedNodeId`, `selectedIndex`
- **UI State**: `showCommandPalette`, `showNodeDetail`, `showHelpModal`, `nodeCreationMode`, `activeActionForm`, `showTemplatePicker`
- **History**: `history: HistoryEntry[]`, `historyIndex: number` (max 50 entries)
- **Data**: `jobs`, `sheets`, `people`, `dashboardState` (all from mock data)

**Immutable Update Pattern:**
All mutations create new arrays via `.map()`, never mutate in place:
```typescript
const newSheets = state.sheets.map(sheet => {
  if (sheet.id !== sheetId) return sheet;
  // ... mutation logic
  return { ...sheet, nodes };
});
```

**History Tracking:**
Every data mutation calls `pushHistory(description)` before changes, enabling full undo/redo.

### 2. Node Operations - Fully Wired

#### Node Creation (`appStore.ts:489-571`)

**`insertNode(sheetId, targetNodeId, position, nodeData)`**
- Creates node with unique ID: `n-${Date.now()}`
- Rewires parent/child relationships based on `position` ('before' or 'after')
- Updates `selectedNodeId` to focus new node
- UI: `SmartNodeCreator` component (`millflow/src/components/dag/SmartNodeCreator.tsx`)
- Trigger: `o` (after) or `O` (before) keyboard shortcuts

**Graph Rewiring Logic:**
- **After target**: New node inherits target's children, target points to new node
- **Before target**: New node inherits target's parents, target points from new node
- All affected nodes' parent/child arrays updated atomically

#### Template Insertion (`appStore.ts:573-651`)

**`insertTemplate(sheetId, afterNodeId, nodes)`**
- Creates multiple nodes with sequential IDs
- Wires nodes in sequence maintaining parent/child chain
- Connects to existing graph at insertion point
- UI: `TemplatePicker` component (`millflow/src/components/dag/TemplatePicker.tsx`)
- Trigger: `t` keyboard shortcut

#### Node Deletion (`appStore.ts:735-768`)

**`deleteNode(sheetId, nodeId)`**
- Reconnects parents directly to children, bypassing deleted node
- Handles multiple parents/children correctly
- Clears `selectedNodeId` after deletion
- Trigger: `dd` (vim-style double-tap)

#### Node Updates (`appStore.ts:770-800`)

**`updateNode(sheetId, nodeId, updates)`** - Generic partial update
**`assignNode(sheetId, nodeId, personIds)`** - Sets assignees array

Both fully connected to UI forms.

#### Status Toggle (`appStore.ts:854-875`)

**`toggleNodeStatus(sheetId, nodeId)`**
- Cycles: `not_started → ready → in_progress → done`
- Blocked nodes cannot be toggled (must resolve blockers first)
- Trigger: Spacebar

#### Parallel Branches (`appStore.ts:653-702`)

**`splitNode(sheetId, nodeId)`**
- Creates new parallel node with same parents/children
- Enables divergent workflows
- Trigger: `s` keyboard shortcut

**`joinNodes(sheetId, nodeIds, targetNodeId)`** - EXISTS but NO UI TRIGGER

### 3. Blocker Operations - Partially Wired

#### Adding Blockers - WIRED (`appStore.ts:802-827`)

**`addBlocker(sheetId, nodeId, blocker)`**
- Creates blocker with ID `b-${Date.now()}`
- Auto-sets node status to `'blocked'`
- UI: `SmartBlockerForm` with natural language parsing
- Features: Template shortcuts (`/material`, `/approval`), date parsing, owner fuzzy matching

#### Editing Blockers - NOT WIRED
- No `updateBlocker()` store method
- `NodeDetailPanel` displays blockers read-only (lines 157-169)

#### Deleting Blockers - NOT WIRED
- No `removeBlocker()` store method
- No delete UI in any component

#### Resolving Blockers - NOT WIRED
- No mechanism to set `resolved_at`
- Blocked status persists even after theoretical resolution
- Node stays blocked indefinitely unless manually status-toggled (which is blocked for blocked nodes)

### 4. Note Operations - Partially Wired

#### Adding Notes - WIRED (`appStore.ts:829-852`)

**`addNote(sheetId, nodeId, { text })`**
- Auto-populates: author='p-009' (Jake Reeves), created_at=today
- UI: `NoteForm` component
- Trigger: `n` keyboard shortcut

#### Editing/Deleting Notes - NOT WIRED
- No `updateNote()` or `removeNote()` methods
- No edit/delete UI

### 5. Form Components - Connection Status

| Component | Location | onSave Handler | Store Method | Status |
|-----------|----------|----------------|--------------|--------|
| SmartNodeCreator | `components/dag/SmartNodeCreator.tsx` | `handleNodeCreate` | `insertNode()` | WIRED |
| TemplatePicker | `components/dag/TemplatePicker.tsx` | `handleTemplateSelect` | `insertTemplate()` | WIRED |
| AssigneePicker | `components/node/AssigneePicker.tsx` | `handleAssignSave` | `assignNode()` | WIRED |
| NoteForm | `components/node/NoteForm.tsx` | `handleNoteSave` | `addNote()` | WIRED |
| SmartBlockerForm | `components/node/SmartBlockerForm.tsx` | `handleBlockerSave` | `addBlocker()` | WIRED |
| DueDatePicker | `components/node/DueDatePicker.tsx` | `handleDueDateSave` | `updateNode()` | WIRED |
| NodeDetailPanel | `components/node/NodeDetailPanel.tsx` | N/A | N/A | READ-ONLY |

All forms in `SheetView.tsx` (lines 189-226) are fully connected via handlers (lines 109-139).

### 6. Entity CRUD Summary

#### Nodes (Full CRUD)
- **Create**: `addNode`, `insertNode`, `insertTemplate`, `splitNode`
- **Read**: Via `sheet.nodes` array
- **Update**: `updateNode`, `assignNode`, `toggleNodeStatus`
- **Delete**: `deleteNode`

#### Blockers (Create-Only)
- **Create**: `addBlocker`
- **Read**: Via `node.blockers` array
- **Update**: NOT IMPLEMENTED
- **Delete**: NOT IMPLEMENTED

#### Notes (Create-Only)
- **Create**: `addNote`
- **Read**: Via `node.notes` array
- **Update**: NOT IMPLEMENTED
- **Delete**: NOT IMPLEMENTED

#### Jobs (Read-Only)
- All operations: NOT IMPLEMENTED
- Data from: `mockData.jobs`

#### Sheets (Read-Only)
- All operations: NOT IMPLEMENTED (nodes modify sheets indirectly)
- Data from: `mockData.sheets`

#### People (Read-Only)
- All operations: NOT IMPLEMENTED
- Data from: `mockData.people`

#### Dashboard State (Read-Only)
- All operations: NOT IMPLEMENTED
- Data from: `mockData.dashboardState`

### 7. Data Persistence - Current State

**In-Memory Only:**
- All data stored in Zustand state
- Deep clones for history: `JSON.parse(JSON.stringify(state.sheets))`
- **No localStorage** - data resets on page refresh
- **No backend API** - frontend-only prototype

**History System:**
- 50-entry rolling history
- Tracks only `sheets` mutations (jobs, people, dashboard not tracked)
- Undo/redo fully functional via `u` and `Ctrl+r` keys

---

## Code References

### Store
- `millflow/src/store/appStore.ts:489-571` - `insertNode()` implementation
- `millflow/src/store/appStore.ts:735-768` - `deleteNode()` implementation
- `millflow/src/store/appStore.ts:802-827` - `addBlocker()` implementation
- `millflow/src/store/appStore.ts:368-437` - History/undo system

### Forms
- `millflow/src/views/SheetView.tsx:189-226` - Form modal rendering
- `millflow/src/views/SheetView.tsx:109-139` - Form save handlers
- `millflow/src/components/node/SmartBlockerForm.tsx` - Smart blocker parsing
- `millflow/src/components/dag/SmartNodeCreator.tsx` - Node creation UI

### DAG Rendering
- `millflow/src/components/dag/DAGRenderer.tsx:25-47` - Level building
- `millflow/src/lib/dag.ts` - Graph traversal utilities
- `millflow/src/components/dag/ParallelBranch.tsx` - Parallel node layout

### Types
- `millflow/src/types/index.ts:30-47` - Node interface
- `millflow/src/types/index.ts:49-57` - Blocker interface
- `millflow/src/types/index.ts:59-64` - Note interface

### Mock Data
- `millflow/src/data/mockData.ts:3-64` - Job definitions
- `millflow/src/data/mockData.ts:110-602` - Sheet/node definitions
- `millflow/src/data/mockData.ts:94-108` - `createNodes()` helper

---

## Architecture Documentation

### Data Flow Pattern
```
Mock Data → Zustand Store → React Components
                ↓
         Store Actions (with history)
                ↓
         New State → Re-render
```

### Graph Structure
- Nodes use bidirectional `parents[]` and `children[]` arrays
- `createNodes()` helper ensures consistency at initialization
- All mutations manually maintain both arrays

### UI State vs Data State
- UI state (modals, selection) in store but NOT in history
- Data state (sheets, nodes) in store WITH history tracking
- Clear separation enables undo/redo without UI side effects

---

## What's Needed for Full Local Data Manipulation

### High Priority (Core Experience)

1. **Blocker Resolution** - Add `resolveBlocker(sheetId, nodeId, blockerId)` that:
   - Sets `resolved_at` timestamp
   - Clears `blocked` status if no remaining blockers

2. **Blocker Deletion** - Add `removeBlocker()` method + UI delete button

3. **Note Editing/Deletion** - Add `updateNote()`, `removeNote()` + UI

4. **Join Nodes UI** - Connect existing `joinNodes()` to keyboard/UI

### Medium Priority (Better Workflow)

5. **Sheet Creation** - Add `createSheet(jobId, { name, description })` + UI

6. **Sheet Metadata Edit** - Allow editing sheet name/description

7. **localStorage Persistence** - Save `sheets` to localStorage on mutation

### Lower Priority (Full Experience)

8. **Job Creation/Editing**
9. **Person Management**
10. **Dashboard State Editing**

---

## Open Questions

1. Should blockers be resolved individually or should there be a "resolve all" action?
2. When all blockers are resolved, should status auto-change to previous state or to 'ready'?
3. Should note/blocker edits create separate history entries or batch?
4. What should the localStorage key structure be for multi-project support?
