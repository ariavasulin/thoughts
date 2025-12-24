---
date: 2025-12-23T16:47:19-08:00
researcher: ariasulin
git_commit: 2660d75535156917eaa5c961be1e74917e090158
branch: main
repository: dwsERP
topic: "Options for breaking up appStore.ts"
tags: [research, zustand, state-management, refactoring, appStore]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: Options for Breaking Up appStore.ts

**Date**: 2025-12-23T16:47:19-08:00
**Researcher**: ariasulin
**Git Commit**: 2660d75535156917eaa5c961be1e74917e090158
**Branch**: main
**Repository**: dwsERP

## Research Question

The appStore.ts file is 1566 lines. What are the options for breaking it up, simplifying it, etc.?

## Summary

The store has grown organically through multiple feature phases and now contains **7 distinct functional domains** with **24 mutation actions** that all follow the same `pushHistory()` → mutate pattern. There are three viable approaches to reduce complexity:

1. **Zustand Slice Pattern** - Split into domain-specific slice files, combine into single store
2. **Extract Helper Functions** - Keep single store, extract repeated patterns into utilities
3. **Hybrid Approach** - Extract helpers first, then slice if still unwieldy

The slice pattern is the "official" Zustand recommendation for large stores. Given that all 21 consuming files import `useAppStore` uniformly and actions need cross-slice access (e.g., node operations calling `pushHistory()`), **Option 1 (Slice Pattern)** is the cleanest long-term solution.

## Current Structure Analysis

### File Statistics
- **Total lines**: 1566
- **Interface definition**: Lines 27-148 (~120 lines)
- **Store implementation**: Lines 150-1566 (~1400 lines)
- **Consuming files**: 21 files across views, components, and hooks

### Functional Domains Identified

| Domain | Lines | Actions | Description |
|--------|-------|---------|-------------|
| Navigation | ~200 | 14 | View switching, up/down/left/right, openSelected, goBack, goHome, etc. |
| UI State | ~30 | 8 | Toggles for modals, sidebar width, pending key sequences |
| History | ~70 | 5 | pushHistory, undo, redo, canUndo, canRedo |
| Node Operations | ~550 | 16 | CRUD for nodes, blockers, notes, status toggling |
| Marks | ~40 | 4 | Vim-style node bookmarks |
| Multi-Select | ~100 | 5 | Selection mode, batch operations |
| Sheet Management | ~70 | 4 | create/update/archive/restore sheets |

### Code Duplication Patterns

**Pattern 1: History + Sheet Mutation** (24 occurrences)
```typescript
get().pushHistory('description');
return set(state => {
  const newSheets = state.sheets.map(sheet => {
    if (sheet.id !== sheetId) return sheet;
    // mutation logic
    return { ...sheet, nodes: modifiedNodes };
  });
  return { sheets: newSheets };
});
```

**Pattern 2: DAG Navigation Setup** (4 occurrences)
```typescript
const { view, selectedNodeId, selectedSheetId, sheets } = get();
if (view !== 'sheet' || !selectedSheetId || !selectedNodeId) return;
const sheet = sheets.find(s => s.id === selectedSheetId);
if (!sheet) return;
const regions = buildRegions(sheet.nodes);
```

**Pattern 3: Node Update by ID** (15 occurrences)
```typescript
nodes: sheet.nodes.map(node =>
  node.id === nodeId ? { ...node, ...updates } : node
)
```

**Pattern 4: Parent/Child Graph Reconnection** (8 occurrences)
```typescript
for (const childId of newNode.children) {
  const child = nodes.find(n => n.id === childId);
  if (child) {
    child.parents = child.parents.map(p => p === oldId ? newId : p);
  }
}
```

## Options

### Option 1: Zustand Slice Pattern

Split the store into domain-specific slice files that combine into one bounded store.

**File Structure:**
```
millflow/src/store/
├── appStore.ts           # Combined store (imports + spreads slices)
├── slices/
│   ├── navigationSlice.ts
│   ├── uiSlice.ts
│   ├── historySlice.ts
│   ├── nodeSlice.ts
│   ├── markSlice.ts
│   ├── multiSelectSlice.ts
│   └── sheetSlice.ts
└── types.ts              # Shared AppState interface
```

**Example Slice (historySlice.ts):**
```typescript
import { StateCreator } from 'zustand';
import type { AppState } from '../types';

export interface HistorySlice {
  history: HistoryEntry[];
  historyIndex: number;
  pushHistory: (description: string) => void;
  undo: () => void;
  redo: () => void;
  canUndo: () => boolean;
  canRedo: () => boolean;
}

export const createHistorySlice: StateCreator<
  AppState,
  [],
  [],
  HistorySlice
> = (set, get) => ({
  history: [],
  historyIndex: -1,
  pushHistory: (description) => set(state => { /* ... */ }),
  undo: () => set(state => { /* ... */ }),
  redo: () => set(state => { /* ... */ }),
  canUndo: () => get().historyIndex >= 0,
  canRedo: () => get().historyIndex < get().history.length - 1,
});
```

**Combined Store (appStore.ts):**
```typescript
import { create } from 'zustand';
import { createNavigationSlice } from './slices/navigationSlice';
import { createHistorySlice } from './slices/historySlice';
// ... other slices

export const useAppStore = create<AppState>()((...a) => ({
  ...createNavigationSlice(...a),
  ...createHistorySlice(...a),
  ...createNodeSlice(...a),
  // ... other slices
}));
```

**Pros:**
- Official Zustand pattern with good documentation
- Clean separation of concerns
- Each slice is independently testable
- Cross-slice actions work via `get()` (already used)
- No changes to consuming components

**Cons:**
- TypeScript boilerplate (StateCreator types)
- 7-8 new files to manage
- Need to carefully define which slice "owns" shared state like `sheets`

**Effort:** Medium (2-3 hours)

### Option 2: Extract Helper Functions

Keep single store file but extract repeated patterns into utility functions.

**New Utilities (store/helpers.ts):**
```typescript
// Sheet mutation helper
export const mutateSheet = (
  sheets: Sheet[],
  sheetId: string,
  mutator: (sheet: Sheet) => Sheet
): Sheet[] => sheets.map(s => s.id === sheetId ? mutator(s) : s);

// Node update helper
export const updateNodeInSheet = (
  sheet: Sheet,
  nodeId: string,
  updates: Partial<Node>
): Sheet => ({
  ...sheet,
  nodes: sheet.nodes.map(n => n.id === nodeId ? { ...n, ...updates } : n)
});

// Graph reconnection helper
export const reconnectParentChild = (
  nodes: Node[],
  deletedId: string,
  deletedNode: Node
): Node[] => { /* ... */ };
```

**Refactored Action:**
```typescript
// Before: 20 lines
assignNode: (sheetId, nodeId, personIds) => {
  get().pushHistory('Assign node');
  return set(state => ({
    sheets: mutateSheet(state.sheets, sheetId, sheet =>
      updateNodeInSheet(sheet, nodeId, { assignees: personIds })
    )
  }));
}
```

**Pros:**
- Minimal structural change
- Reduces appStore.ts by ~300-400 lines
- Helpers are reusable across actions
- No TypeScript complexity

**Cons:**
- File still large (~1100 lines)
- Doesn't address separation of concerns
- Harder to test individual domains

**Effort:** Low (1-2 hours)

### Option 3: Hybrid Approach

1. First extract helpers (Option 2)
2. Then apply slice pattern to logical groupings (Option 1)

This yields smaller, cleaner slices because the boilerplate is already extracted.

**Pros:**
- Best of both approaches
- Each slice is small and focused
- Helpers prevent code duplication across slices

**Cons:**
- Highest total effort
- May be over-engineering for current needs

**Effort:** Medium-High (3-4 hours)

## Recommendation

**Start with Option 2 (Extract Helpers)** as immediate cleanup, then evaluate if slices are still needed.

Rationale:
- The store works fine functionally - it's just long
- Helper extraction provides immediate line count reduction
- If the store continues to grow (e.g., time estimates, scheduling), then apply slices
- Avoids TypeScript complexity for marginal benefit

However, if architectural cleanliness is prioritized over effort, **Option 1 (Slices)** is the proper long-term solution.

## Code References

- `millflow/src/store/appStore.ts:1-1566` - Current store implementation
- `millflow/src/lib/dag.ts` - DAG utilities heavily used by navigation actions
- `millflow/src/lib/queue.ts` - Queue calculation used by navigation

## Consumers (21 files)

**Views:** DashboardView, DeliveriesView, DepartmentsView, JobView, JobsView, QueueView, ScheduleView, SheetView, TimelineView

**Components:** Layout, ShortcutBar, CommandPalette, HelpModal, DAGNode, SmartNodeCreator, NodeDetailPanel, NodeDetailContent, SmartBlockerForm, AssigneePicker, DashboardSection

**Hooks:** useKeyboard

## Historical Context (from thoughts/)

- `thoughts/shared/research/2025-12-19-millflow-data-manipulation-state.md` - Comprehensive state management analysis
- Multiple handoff documents show the store grew through phases: initial CRUD → undo/redo → marks → multi-select → sheet management

## Related Resources

- [Zustand Slice Pattern (Official)](https://zustand.docs.pmnd.rs/guides/slices-pattern)
- [Zustand Advanced TypeScript](https://zustand.docs.pmnd.rs/guides/advanced-typescript)
- [GitHub Discussion: Multiple stores vs slices](https://github.com/pmndrs/zustand/discussions/2496)
