# Assignment Constraint System Implementation Plan

## Overview

A complete constraint-based worker assignment system for MillFlow. PMs define **who can do what** using boolean logic (AND/OR), worker groups, and keyboard-driven UI. The optimizer will consume these constraints to produce schedules. This is the foundation for treating project management as DAG orchestration.

## Current State Analysis

### What Exists
- Simple `owner: string` field on nodes (free-text input)
- Hardcoded `OWNERS` map in `mockShopData.ts` mapping stations to name strings
- No formal Worker or Group data model
- Zustand installed but unused (all state via React hooks)
- Basic keyboard shortcuts in App.tsx (Enter, Escape, o, b)
- No command palette, fuzzy search, or mode system

### Key Discoveries
- `OperationNodeData.owner` is optional string (`factory-floor/src/components/nodes/OperationNode.tsx:12`)
- Keyboard handler guards inputs/modals (`App.tsx:382-389`)
- No reusable UI components - all inline Tailwind
- Node editing via `NodeEditModal.tsx` with `handleNodeSave` callback

## Desired End State

1. **Workers** are first-class entities with id, name, initials, color, and group memberships
2. **Groups** are named collections of workers (e.g., "CNC Team", "Senior Ops", "Night Shift")
3. **Assignment Constraints** are boolean expressions on nodes specifying who can do the work
4. **Keyboard-driven UI** with contextual hint bar showing available actions per mode
5. **Workers modal** (Cmd+K or `w`) for managing workers and groups with full CRUD
6. All existing functionality preserved and enhanced

### Verification
- Pressing `a` on a selected node opens the assignment builder
- Typing filters workers/groups with fuzzy search
- `&` adds AND, `|` adds OR, `()` groups expressions
- Constraints display as visual chips on nodes
- Pressing `Cmd+K` or `w` opens workers management modal
- All actions have keyboard shortcuts shown in hint bar

## What We're NOT Doing

- Backend persistence (stays frontend-only with mock data)
- Actual optimizer/scheduler integration (just defining constraints)
- Worker availability/scheduling (that's optimizer's job)
- Skills/certifications system (future enhancement)
- Time-based constraints (shift schedules, etc.)

## Implementation Approach

Build in layers: data model → display → keyboard system → builder UI → management UI. Each phase is independently testable and builds on the previous.

---

## Phase 1: Zustand Store + Data Model

### Overview
Establish the foundational data types and global state management. Replace React hooks prop-drilling with Zustand for workers, groups, and UI mode state.

### Changes Required

#### 1. Create Store with Types
**File**: `factory-floor/src/store/appStore.ts` (new file)

```typescript
import { create } from 'zustand';

// === WORKER & GROUP TYPES ===

export interface Worker {
  id: string;
  name: string;
  initials: string;
  color: string; // gruvbox color key: 'red' | 'green' | 'yellow' | 'blue' | 'purple' | 'aqua' | 'orange'
  groupIds: string[];
}

export interface WorkerGroup {
  id: string;
  name: string;
  color: string;
  description?: string;
}

// === ASSIGNMENT CONSTRAINT TYPE ===

export type AssignmentConstraint =
  | { type: 'worker'; workerId: string }
  | { type: 'anyFrom'; groupId: string }
  | { type: 'and'; operands: AssignmentConstraint[] }
  | { type: 'or'; operands: AssignmentConstraint[] };

// === UI MODE ===

export type UIMode = 'normal' | 'assign' | 'blocker' | 'operation' | 'edge' | 'workers';

// === STORE ===

interface AppState {
  // Workers & Groups
  workers: Worker[];
  groups: WorkerGroup[];

  // UI State
  activeMode: UIMode;

  // Worker Actions
  addWorker: (worker: Worker) => void;
  updateWorker: (id: string, updates: Partial<Worker>) => void;
  deleteWorker: (id: string) => void;

  // Group Actions
  addGroup: (group: WorkerGroup) => void;
  updateGroup: (id: string, updates: Partial<WorkerGroup>) => void;
  deleteGroup: (id: string) => void;

  // UI Actions
  setActiveMode: (mode: UIMode) => void;

  // Helpers
  getWorkerById: (id: string) => Worker | undefined;
  getGroupById: (id: string) => WorkerGroup | undefined;
  getWorkersInGroup: (groupId: string) => Worker[];
}

export const useAppStore = create<AppState>((set, get) => ({
  workers: [],
  groups: [],
  activeMode: 'normal',

  addWorker: (worker) => set((state) => ({
    workers: [...state.workers, worker]
  })),

  updateWorker: (id, updates) => set((state) => ({
    workers: state.workers.map((w) =>
      w.id === id ? { ...w, ...updates } : w
    ),
  })),

  deleteWorker: (id) => set((state) => ({
    workers: state.workers.filter((w) => w.id !== id),
  })),

  addGroup: (group) => set((state) => ({
    groups: [...state.groups, group]
  })),

  updateGroup: (id, updates) => set((state) => ({
    groups: state.groups.map((g) =>
      g.id === id ? { ...g, ...updates } : g
    ),
  })),

  deleteGroup: (id) => set((state) => ({
    groups: state.groups.filter((g) => g.id !== id),
    // Also remove group from all workers
    workers: state.workers.map((w) => ({
      ...w,
      groupIds: w.groupIds.filter((gid) => gid !== id),
    })),
  })),

  setActiveMode: (mode) => set({ activeMode: mode }),

  getWorkerById: (id) => get().workers.find((w) => w.id === id),
  getGroupById: (id) => get().groups.find((g) => g.id === id),
  getWorkersInGroup: (groupId) => get().workers.filter((w) => w.groupIds.includes(groupId)),
}));
```

#### 2. Create Mock Workers & Groups Data
**File**: `factory-floor/src/data/workers.ts` (new file)

```typescript
import { Worker, WorkerGroup } from '../store/appStore';

export const MOCK_GROUPS: WorkerGroup[] = [
  { id: 'grp-cnc', name: 'CNC Team', color: 'red', description: 'CNC operators and programmers' },
  { id: 'grp-veneer', name: 'Veneer Team', color: 'green', description: 'Veneer and laminate specialists' },
  { id: 'grp-bench', name: 'Bench Team', color: 'blue', description: 'Assembly and bench work' },
  { id: 'grp-finish', name: 'Finishing Team', color: 'purple', description: 'Finishing and coating' },
  { id: 'grp-senior', name: 'Senior Ops', color: 'orange', description: 'Senior operators (any station)' },
  { id: 'grp-field', name: 'Field Team', color: 'aqua', description: 'Delivery and installation' },
];

export const MOCK_WORKERS: Worker[] = [
  // CNC Team
  { id: 'w-charlie', name: 'Charlie R.', initials: 'CR', color: 'red', groupIds: ['grp-cnc', 'grp-senior'] },
  { id: 'w-dave', name: 'Dave K.', initials: 'DK', color: 'red', groupIds: ['grp-cnc'] },

  // Veneer Team
  { id: 'w-eve', name: 'Eve W.', initials: 'EW', color: 'green', groupIds: ['grp-veneer', 'grp-senior'] },
  { id: 'w-vera', name: 'Vera N.', initials: 'VN', color: 'green', groupIds: ['grp-veneer'] },

  // Bench Team
  { id: 'w-grace', name: 'Grace H.', initials: 'GH', color: 'blue', groupIds: ['grp-bench'] },
  { id: 'w-heidi', name: 'Heidi P.', initials: 'HP', color: 'blue', groupIds: ['grp-bench'] },
  { id: 'w-sam', name: 'Sam O.', initials: 'SO', color: 'blue', groupIds: ['grp-bench', 'grp-senior'] },

  // Finishing Team
  { id: 'w-finn', name: 'Finn S.', initials: 'FS', color: 'purple', groupIds: ['grp-finish'] },
  { id: 'w-nora', name: 'Nora F.', initials: 'NF', color: 'purple', groupIds: ['grp-finish'] },

  // Field Team
  { id: 'w-dan', name: 'Dan D.', initials: 'DD', color: 'aqua', groupIds: ['grp-field'] },
  { id: 'w-ian', name: 'Ian F.', initials: 'IF', color: 'aqua', groupIds: ['grp-field', 'grp-senior'] },
  { id: 'w-rita', name: 'Rita T.', initials: 'RT', color: 'aqua', groupIds: ['grp-field'] },
];
```

#### 3. Initialize Store with Mock Data
**File**: `factory-floor/src/App.tsx`
**Changes**: Add store initialization on mount

```typescript
import { useEffect } from 'react';
import { useAppStore } from './store/appStore';
import { MOCK_WORKERS, MOCK_GROUPS } from './data/workers';

// Inside App component, add:
useEffect(() => {
  const { workers, groups, addWorker, addGroup } = useAppStore.getState();

  // Only initialize if empty (prevents re-init on hot reload)
  if (workers.length === 0) {
    MOCK_GROUPS.forEach(addGroup);
    MOCK_WORKERS.forEach(addWorker);
  }
}, []);
```

#### 4. Update OperationNodeData Type
**File**: `factory-floor/src/components/nodes/OperationNode.tsx`
**Changes**: Add constraint field alongside owner (backward compatible)

```typescript
import { AssignmentConstraint } from '../../store/appStore';

export type OperationNodeData = {
  label: string;
  type: 'CNC' | 'EdgeBanding' | 'Assembly' | 'Veneer' | 'Generic';
  notes?: string;
  duration?: number;
  owner?: string; // Keep for backward compatibility
  assignmentConstraint?: AssignmentConstraint; // New field
  blockers?: string[];
  gridPos?: { c: number; r: number };
};
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd factory-floor && npm run build`
- [ ] App loads without console errors: `npm run dev` and check browser console
- [ ] Store initializes with mock data: Add `console.log(useAppStore.getState())` temporarily

#### Manual Verification:
- [ ] App still functions normally (no regressions)
- [ ] React DevTools shows Zustand store with workers/groups

---

## Phase 2: Keyboard Mode System + HintBar

### Overview
Create the contextual keyboard mode system and hint bar component. This establishes the UX pattern that all keyboard-driven features will use.

### Changes Required

#### 1. Create HintBar Component
**File**: `factory-floor/src/components/keyboard/HintBar.tsx` (new file)

```typescript
import { memo } from 'react';
import { UIMode, useAppStore } from '../../store/appStore';

interface HintItem {
  key: string;
  label: string;
}

const MODE_HINTS: Record<UIMode, HintItem[]> = {
  normal: [
    { key: 'j/k', label: 'navigate' },
    { key: 'Enter', label: 'edit' },
    { key: 'a', label: 'assign' },
    { key: 'b', label: 'blocker' },
    { key: 'o', label: 'new op' },
    { key: '⌘K', label: 'workers' },
  ],
  assign: [
    { key: 'type', label: 'search' },
    { key: '&', label: 'AND' },
    { key: '|', label: 'OR' },
    { key: '( )', label: 'group' },
    { key: 'Tab', label: 'toggle' },
    { key: 'Enter', label: 'confirm' },
    { key: 'Esc', label: 'cancel' },
  ],
  blocker: [
    { key: 'type', label: 'reason' },
    { key: 'Tab', label: 'category' },
    { key: 'Enter', label: 'add' },
    { key: 'Esc', label: 'cancel' },
  ],
  operation: [
    { key: 'type', label: 'name' },
    { key: 'Tab', label: 'type' },
    { key: 'Enter', label: 'create' },
    { key: 'Esc', label: 'cancel' },
  ],
  edge: [
    { key: 'click', label: 'target node' },
    { key: 'Esc', label: 'cancel' },
  ],
  workers: [
    { key: 'j/k', label: 'navigate' },
    { key: 'n', label: 'new worker' },
    { key: 'g', label: 'new group' },
    { key: 'Enter', label: 'edit' },
    { key: 'd', label: 'delete' },
    { key: 'Esc', label: 'close' },
  ],
};

const MODE_LABELS: Record<UIMode, string> = {
  normal: 'NORMAL',
  assign: 'ASSIGN',
  blocker: 'BLOCKER',
  operation: 'NEW OP',
  edge: 'EDGE',
  workers: 'WORKERS',
};

const MODE_COLORS: Record<UIMode, string> = {
  normal: 'text-gruvbox-fg-4',
  assign: 'text-gruvbox-aqua',
  blocker: 'text-gruvbox-red',
  operation: 'text-gruvbox-green',
  edge: 'text-gruvbox-yellow',
  workers: 'text-gruvbox-purple',
};

export const HintBar = memo(() => {
  const activeMode = useAppStore((s) => s.activeMode);
  const hints = MODE_HINTS[activeMode];
  const label = MODE_LABELS[activeMode];
  const color = MODE_COLORS[activeMode];

  return (
    <div className="fixed bottom-0 left-0 right-0 h-8 bg-gruvbox-bg-hard border-t border-gruvbox-fg-4/20 flex items-center px-4 gap-4 z-50">
      {/* Mode indicator */}
      <div className={`font-bold text-xs ${color}`}>
        {label}
      </div>

      <div className="w-px h-4 bg-gruvbox-fg-4/30" />

      {/* Hints */}
      <div className="flex items-center gap-3">
        {hints.map((hint) => (
          <div key={hint.key} className="flex items-center gap-1.5 text-xs">
            <kbd className="px-1.5 py-0.5 bg-gruvbox-bg-soft rounded border border-gruvbox-fg-4/30 font-mono text-gruvbox-fg-1">
              {hint.key}
            </kbd>
            <span className="text-gruvbox-fg-4">{hint.label}</span>
          </div>
        ))}
      </div>
    </div>
  );
});

HintBar.displayName = 'HintBar';
```

#### 2. Create Keyboard Hook
**File**: `factory-floor/src/hooks/useKeyboardMode.ts` (new file)

```typescript
import { useEffect, useCallback } from 'react';
import { UIMode, useAppStore } from '../store/appStore';

interface UseKeyboardModeOptions {
  onModeChange?: (mode: UIMode) => void;
}

export function useKeyboardMode(options: UseKeyboardModeOptions = {}) {
  const { activeMode, setActiveMode } = useAppStore();

  const enterMode = useCallback((mode: UIMode) => {
    setActiveMode(mode);
    options.onModeChange?.(mode);
  }, [setActiveMode, options]);

  const exitMode = useCallback(() => {
    setActiveMode('normal');
    options.onModeChange?.('normal');
  }, [setActiveMode, options]);

  // Global escape handler to exit any mode
  useEffect(() => {
    const handleEscape = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && activeMode !== 'normal') {
        e.preventDefault();
        exitMode();
      }
    };

    window.addEventListener('keydown', handleEscape);
    return () => window.removeEventListener('keydown', handleEscape);
  }, [activeMode, exitMode]);

  return {
    activeMode,
    enterMode,
    exitMode,
    isInMode: (mode: UIMode) => activeMode === mode,
  };
}
```

#### 3. Integrate HintBar into App
**File**: `factory-floor/src/App.tsx`
**Changes**: Add HintBar component and adjust layout for bottom bar

```typescript
import { HintBar } from './components/keyboard/HintBar';

// In the App component's return, wrap content and add HintBar:
return (
  <div className="h-screen w-screen bg-gruvbox-bg-hard text-gruvbox-fg font-sans selection:bg-gruvbox-aqua selection:text-gruvbox-bg-hard flex flex-col">
    {/* Header */}
    <header className="h-12 ...">
      {/* existing header */}
    </header>

    {/* Main Content - adjust for hint bar */}
    <div className="flex-1 overflow-hidden relative pb-8">
      {currentView === 'editor' ? <Editor /> : <Dashboard />}
    </div>

    {/* Hint Bar */}
    <HintBar />
  </div>
);
```

#### 4. Update Keyboard Handler to Use Modes
**File**: `factory-floor/src/App.tsx` (in Editor component)
**Changes**: Integrate mode system with existing shortcuts

```typescript
import { useKeyboardMode } from '../hooks/useKeyboardMode';
import { useAppStore } from '../store/appStore';

// In Editor component:
const { activeMode, enterMode, exitMode } = useKeyboardMode();

// Update the keyboard handler:
useEffect(() => {
  const handleKeyDown = (event: KeyboardEvent) => {
    // Ignore if typing in an input
    if (
      event.target instanceof HTMLInputElement ||
      event.target instanceof HTMLTextAreaElement
    ) {
      return;
    }

    // Mode-specific handling
    if (activeMode !== 'normal') {
      // Let mode-specific handlers deal with it
      return;
    }

    // Normal mode shortcuts
    if (event.key === 'Enter' && selectedNodes.length === 1 && !editingNodeId) {
      event.preventDefault();
      setEditingNodeId(selectedNodes[0]);
      return;
    }

    if (event.key === 'Escape') {
      event.preventDefault();
      setSelectedNodes([]);
      return;
    }

    // 'a' - enter assign mode
    if (event.key === 'a' && selectedNodes.length === 1) {
      event.preventDefault();
      enterMode('assign');
      return;
    }

    // Cmd+K or 'w' - enter workers mode (open modal)
    if ((event.key === 'k' && (event.metaKey || event.ctrlKey)) || event.key === 'w') {
      event.preventDefault();
      enterMode('workers');
      return;
    }

    // Existing 'o' and 'b' shortcuts...
    if (event.key === 'o' && selectedNodes.length === 1) {
      // existing code
    }

    if (event.key === 'b' && selectedNodes.length === 1) {
      event.preventDefault();
      enterMode('blocker');
      setEditingNodeId(selectedNodes[0]);
      return;
    }
  };

  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, [activeMode, selectedNodes, editingNodeId, enterMode]);
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `npm run build`
- [ ] No console errors on load

#### Manual Verification:
- [ ] HintBar visible at bottom of screen
- [ ] Shows "NORMAL" mode by default with correct hints
- [ ] Pressing 'a' on selected node changes hint bar to "ASSIGN" mode
- [ ] Pressing 'w' changes to "WORKERS" mode
- [ ] Pressing Escape returns to "NORMAL" mode
- [ ] Hint bar colors match mode (aqua for assign, red for blocker, etc.)

---

## Phase 3: Constraint Display (ExpressionRenderer)

### Overview
Create components to visually display assignment constraints on nodes as chips/pills. Read-only display before we build the editor.

### Changes Required

#### 1. Create Expression Renderer Component
**File**: `factory-floor/src/components/assignment/ExpressionRenderer.tsx` (new file)

```typescript
import { memo } from 'react';
import { AssignmentConstraint, useAppStore } from '../../store/appStore';
import clsx from 'clsx';

interface ExpressionRendererProps {
  constraint: AssignmentConstraint;
  compact?: boolean; // For inline node display
}

const GRUVBOX_COLORS: Record<string, string> = {
  red: 'bg-gruvbox-red/20 text-gruvbox-red border-gruvbox-red/30',
  green: 'bg-gruvbox-green/20 text-gruvbox-green border-gruvbox-green/30',
  yellow: 'bg-gruvbox-yellow/20 text-gruvbox-yellow border-gruvbox-yellow/30',
  blue: 'bg-gruvbox-blue/20 text-gruvbox-blue border-gruvbox-blue/30',
  purple: 'bg-gruvbox-purple/20 text-gruvbox-purple border-gruvbox-purple/30',
  aqua: 'bg-gruvbox-aqua/20 text-gruvbox-aqua border-gruvbox-aqua/30',
  orange: 'bg-gruvbox-orange/20 text-gruvbox-orange border-gruvbox-orange/30',
};

export const ExpressionRenderer = memo(({ constraint, compact = false }: ExpressionRendererProps) => {
  const getWorkerById = useAppStore((s) => s.getWorkerById);
  const getGroupById = useAppStore((s) => s.getGroupById);

  const renderConstraint = (c: AssignmentConstraint, depth = 0): React.ReactNode => {
    switch (c.type) {
      case 'worker': {
        const worker = getWorkerById(c.workerId);
        if (!worker) return <span className="text-gruvbox-red">Unknown</span>;

        const colorClasses = GRUVBOX_COLORS[worker.color] || GRUVBOX_COLORS.aqua;

        return (
          <span className={clsx(
            'inline-flex items-center gap-1 px-1.5 py-0.5 rounded border text-xs font-medium',
            colorClasses
          )}>
            {compact ? worker.initials : worker.name}
          </span>
        );
      }

      case 'anyFrom': {
        const group = getGroupById(c.groupId);
        if (!group) return <span className="text-gruvbox-red">Unknown Group</span>;

        const colorClasses = GRUVBOX_COLORS[group.color] || GRUVBOX_COLORS.aqua;

        return (
          <span className={clsx(
            'inline-flex items-center gap-1 px-1.5 py-0.5 rounded border text-xs font-medium',
            colorClasses
          )}>
            <span className="opacity-60">any</span>
            {group.name}
          </span>
        );
      }

      case 'and': {
        return (
          <span className="inline-flex items-center gap-1 flex-wrap">
            {c.operands.map((op, i) => (
              <span key={i} className="inline-flex items-center gap-1">
                {renderConstraint(op, depth + 1)}
                {i < c.operands.length - 1 && (
                  <span className="text-gruvbox-fg-4 text-xs font-bold">&</span>
                )}
              </span>
            ))}
          </span>
        );
      }

      case 'or': {
        const needsParens = depth > 0;
        return (
          <span className="inline-flex items-center gap-1 flex-wrap">
            {needsParens && <span className="text-gruvbox-fg-4">(</span>}
            {c.operands.map((op, i) => (
              <span key={i} className="inline-flex items-center gap-1">
                {renderConstraint(op, depth + 1)}
                {i < c.operands.length - 1 && (
                  <span className="text-gruvbox-fg-4 text-xs font-bold">|</span>
                )}
              </span>
            ))}
            {needsParens && <span className="text-gruvbox-fg-4">)</span>}
          </span>
        );
      }
    }
  };

  return (
    <div className="inline-flex items-center gap-1 flex-wrap">
      {renderConstraint(constraint)}
    </div>
  );
});

ExpressionRenderer.displayName = 'ExpressionRenderer';
```

#### 2. Create AssignmentChip for Node Display
**File**: `factory-floor/src/components/assignment/AssignmentChip.tsx` (new file)

```typescript
import { memo } from 'react';
import { AssignmentConstraint } from '../../store/appStore';
import { ExpressionRenderer } from './ExpressionRenderer';

interface AssignmentChipProps {
  constraint: AssignmentConstraint;
  onClick?: () => void;
}

export const AssignmentChip = memo(({ constraint, onClick }: AssignmentChipProps) => {
  return (
    <button
      onClick={onClick}
      className="flex items-center gap-1 text-xs hover:bg-gruvbox-bg-soft/50 rounded px-1 py-0.5 transition-colors"
    >
      <ExpressionRenderer constraint={constraint} compact />
    </button>
  );
});

AssignmentChip.displayName = 'AssignmentChip';
```

#### 3. Create Barrel Export
**File**: `factory-floor/src/components/assignment/index.ts` (new file)

```typescript
export { ExpressionRenderer } from './ExpressionRenderer';
export { AssignmentChip } from './AssignmentChip';
```

#### 4. Update OperationNode to Display Constraints
**File**: `factory-floor/src/components/nodes/OperationNode.tsx`
**Changes**: Replace owner display with constraint display

```typescript
import { AssignmentChip } from '../assignment';

// In the component, replace the owner display section:

{/* Assignment Constraint (new) */}
{data.assignmentConstraint && (
  <div className="mb-1">
    <AssignmentChip constraint={data.assignmentConstraint} />
  </div>
)}

{/* Owner (legacy fallback) */}
{!data.assignmentConstraint && data.owner && (
  <div className="text-xs text-gruvbox-aqua mb-1 flex items-center gap-1">
    <span className="opacity-60">Owner:</span> {data.owner}
  </div>
)}
```

#### 5. Add Test Constraint to Mock Data
**File**: `factory-floor/src/data/mockShopData.ts`
**Changes**: Add sample constraints to some workflow nodes for testing

```typescript
// In generateWorkflowForSheet or similar, add test constraints to some nodes:
// Example constraint: CNC Team OR (Charlie AND Dave)
const testConstraint: AssignmentConstraint = {
  type: 'or',
  operands: [
    { type: 'anyFrom', groupId: 'grp-cnc' },
    {
      type: 'and',
      operands: [
        { type: 'worker', workerId: 'w-charlie' },
        { type: 'worker', workerId: 'w-dave' },
      ]
    }
  ]
};
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `npm run build`
- [ ] No console errors

#### Manual Verification:
- [ ] Nodes with constraints show colored chips
- [ ] Worker chips show initials (compact) or full name
- [ ] Group chips show "any [Group Name]"
- [ ] AND shows `&` between operands
- [ ] OR shows `|` between operands
- [ ] Nested expressions render correctly with parens
- [ ] Legacy `owner` field still displays for nodes without constraints

---

## Phase 4: Assignment Builder UI

### Overview
The main event: a keyboard-driven constraint expression builder. Opens when pressing 'a' on a node, allows building complex boolean expressions through typing and keyboard shortcuts.

### Changes Required

#### 1. Create Fuzzy Search Utility
**File**: `factory-floor/src/lib/fuzzySearch.ts` (new file)

```typescript
export interface SearchResult<T> {
  item: T;
  score: number;
  matches: [number, number][]; // Start/end indices of matching chars
}

export function fuzzySearch<T>(
  items: T[],
  query: string,
  getSearchText: (item: T) => string
): SearchResult<T>[] {
  if (!query) {
    return items.map((item) => ({ item, score: 0, matches: [] }));
  }

  const queryLower = query.toLowerCase();
  const results: SearchResult<T>[] = [];

  for (const item of items) {
    const text = getSearchText(item).toLowerCase();
    const matches: [number, number][] = [];
    let score = 0;
    let queryIdx = 0;
    let lastMatchIdx = -1;

    for (let i = 0; i < text.length && queryIdx < queryLower.length; i++) {
      if (text[i] === queryLower[queryIdx]) {
        const matchStart = i;
        // Find consecutive matches
        while (i < text.length && queryIdx < queryLower.length && text[i] === queryLower[queryIdx]) {
          i++;
          queryIdx++;
        }
        matches.push([matchStart, i]);

        // Bonus for consecutive matches and early matches
        const consecutiveBonus = (i - matchStart) * 10;
        const positionBonus = Math.max(0, 20 - matchStart);
        const gapPenalty = lastMatchIdx >= 0 ? Math.min(10, matchStart - lastMatchIdx - 1) : 0;

        score += consecutiveBonus + positionBonus - gapPenalty;
        lastMatchIdx = i - 1;
        i--; // Adjust for loop increment
      }
    }

    // Only include if all query chars were found
    if (queryIdx === queryLower.length) {
      results.push({ item, score, matches });
    }
  }

  return results.sort((a, b) => b.score - a.score);
}
```

#### 2. Create Assignment Builder Component
**File**: `factory-floor/src/components/assignment/AssignmentBuilder.tsx` (new file)

```typescript
import { useState, useEffect, useCallback, useRef, memo } from 'react';
import { AssignmentConstraint, useAppStore, Worker, WorkerGroup } from '../../store/appStore';
import { fuzzySearch } from '../../lib/fuzzySearch';
import { ExpressionRenderer } from './ExpressionRenderer';
import clsx from 'clsx';

interface AssignmentBuilderProps {
  initialConstraint?: AssignmentConstraint;
  onConfirm: (constraint: AssignmentConstraint | undefined) => void;
  onCancel: () => void;
}

type SearchItem =
  | { type: 'worker'; worker: Worker }
  | { type: 'group'; group: WorkerGroup };

type ExpressionToken =
  | { type: 'constraint'; constraint: AssignmentConstraint }
  | { type: 'operator'; operator: '&' | '|' }
  | { type: 'paren'; paren: '(' | ')' };

export const AssignmentBuilder = memo(({ initialConstraint, onConfirm, onCancel }: AssignmentBuilderProps) => {
  const workers = useAppStore((s) => s.workers);
  const groups = useAppStore((s) => s.groups);

  const [searchQuery, setSearchQuery] = useState('');
  const [tokens, setTokens] = useState<ExpressionToken[]>(() => {
    if (initialConstraint) {
      return [{ type: 'constraint', constraint: initialConstraint }];
    }
    return [];
  });
  const [selectedIndex, setSelectedIndex] = useState(0);
  const [showResults, setShowResults] = useState(true);

  const inputRef = useRef<HTMLInputElement>(null);

  // Build searchable items
  const searchItems: SearchItem[] = [
    ...workers.map((w) => ({ type: 'worker' as const, worker: w })),
    ...groups.map((g) => ({ type: 'group' as const, group: g })),
  ];

  // Fuzzy search
  const searchResults = fuzzySearch(
    searchItems,
    searchQuery,
    (item) => item.type === 'worker' ? item.worker.name : item.group.name
  );

  // Focus input on mount
  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  // Build constraint from tokens
  const buildConstraint = useCallback((): AssignmentConstraint | undefined => {
    const constraints = tokens.filter((t) => t.type === 'constraint') as { type: 'constraint'; constraint: AssignmentConstraint }[];

    if (constraints.length === 0) return undefined;
    if (constraints.length === 1) return constraints[0].constraint;

    // Check for operators
    const hasAnd = tokens.some((t) => t.type === 'operator' && t.operator === '&');
    const hasOr = tokens.some((t) => t.type === 'operator' && t.operator === '|');

    if (hasAnd && !hasOr) {
      return { type: 'and', operands: constraints.map((c) => c.constraint) };
    }
    if (hasOr && !hasAnd) {
      return { type: 'or', operands: constraints.map((c) => c.constraint) };
    }

    // Mixed - OR has lower precedence, so group ANDs first
    // Simple implementation: just use OR for now
    return { type: 'or', operands: constraints.map((c) => c.constraint) };
  }, [tokens]);

  // Add selection to tokens
  const addSelection = useCallback((item: SearchItem) => {
    const constraint: AssignmentConstraint = item.type === 'worker'
      ? { type: 'worker', workerId: item.worker.id }
      : { type: 'anyFrom', groupId: item.group.id };

    setTokens((prev) => [...prev, { type: 'constraint', constraint }]);
    setSearchQuery('');
    setSelectedIndex(0);
    inputRef.current?.focus();
  }, []);

  // Add operator
  const addOperator = useCallback((op: '&' | '|') => {
    // Only add if last token is a constraint
    const lastToken = tokens[tokens.length - 1];
    if (lastToken?.type === 'constraint') {
      setTokens((prev) => [...prev, { type: 'operator', operator: op }]);
    }
    inputRef.current?.focus();
  }, [tokens]);

  // Remove last token
  const removeLastToken = useCallback(() => {
    setTokens((prev) => prev.slice(0, -1));
    inputRef.current?.focus();
  }, []);

  // Keyboard handling
  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setSelectedIndex((i) => Math.min(i + 1, searchResults.length - 1));
        break;

      case 'ArrowUp':
        e.preventDefault();
        setSelectedIndex((i) => Math.max(i - 1, 0));
        break;

      case 'Enter':
        e.preventDefault();
        if (searchResults.length > 0 && searchQuery) {
          addSelection(searchResults[selectedIndex].item);
        } else if (tokens.length > 0) {
          onConfirm(buildConstraint());
        }
        break;

      case 'Escape':
        e.preventDefault();
        onCancel();
        break;

      case 'Backspace':
        if (searchQuery === '' && tokens.length > 0) {
          e.preventDefault();
          removeLastToken();
        }
        break;

      case '&':
        if (searchQuery === '') {
          e.preventDefault();
          addOperator('&');
        }
        break;

      case '|':
        if (searchQuery === '') {
          e.preventDefault();
          addOperator('|');
        }
        break;
    }
  }, [searchResults, selectedIndex, searchQuery, tokens, addSelection, addOperator, removeLastToken, buildConstraint, onConfirm, onCancel]);

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50" onClick={onCancel}>
      <div
        className="bg-gruvbox-bg border border-gruvbox-aqua rounded-lg shadow-xl w-full max-w-lg"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="px-4 py-2 border-b border-gruvbox-fg-4/20 flex items-center gap-2">
          <span className="text-xs font-bold text-gruvbox-aqua">ASSIGN</span>
          <span className="text-xs text-gruvbox-fg-4">Build constraint expression</span>
        </div>

        {/* Expression Builder */}
        <div className="p-4">
          {/* Current Expression */}
          <div className="flex flex-wrap items-center gap-1 min-h-[32px] mb-3 p-2 bg-gruvbox-bg-soft rounded border border-gruvbox-fg-4/20">
            {tokens.map((token, i) => (
              <span key={i} className="inline-flex items-center">
                {token.type === 'constraint' && (
                  <ExpressionRenderer constraint={token.constraint} />
                )}
                {token.type === 'operator' && (
                  <span className="px-1.5 py-0.5 text-xs font-bold text-gruvbox-fg-4">
                    {token.operator === '&' ? 'AND' : 'OR'}
                  </span>
                )}
                {token.type === 'paren' && (
                  <span className="text-gruvbox-fg-4">{token.paren}</span>
                )}
              </span>
            ))}

            {/* Input */}
            <input
              ref={inputRef}
              type="text"
              value={searchQuery}
              onChange={(e) => {
                setSearchQuery(e.target.value);
                setSelectedIndex(0);
                setShowResults(true);
              }}
              onKeyDown={handleKeyDown}
              placeholder={tokens.length === 0 ? "Type to search workers or groups..." : ""}
              className="flex-1 min-w-[100px] bg-transparent border-none outline-none text-gruvbox-fg placeholder:text-gruvbox-fg-4/50"
            />
          </div>

          {/* Search Results */}
          {showResults && searchQuery && (
            <div className="border border-gruvbox-fg-4/20 rounded bg-gruvbox-bg-soft max-h-48 overflow-y-auto">
              {searchResults.length === 0 ? (
                <div className="p-3 text-xs text-gruvbox-fg-4 text-center">
                  No matches found
                </div>
              ) : (
                searchResults.slice(0, 10).map((result, i) => {
                  const item = result.item;
                  const isSelected = i === selectedIndex;
                  const isWorker = item.type === 'worker';

                  return (
                    <button
                      key={isWorker ? item.worker.id : item.group.id}
                      onClick={() => addSelection(item)}
                      className={clsx(
                        'w-full text-left px-3 py-2 flex items-center gap-2 text-sm',
                        isSelected ? 'bg-gruvbox-aqua/20' : 'hover:bg-gruvbox-bg-hard'
                      )}
                    >
                      {isWorker ? (
                        <>
                          <span className={`w-6 h-6 rounded-full bg-gruvbox-${item.worker.color}/20 text-gruvbox-${item.worker.color} flex items-center justify-center text-xs font-bold`}>
                            {item.worker.initials}
                          </span>
                          <span className="text-gruvbox-fg">{item.worker.name}</span>
                          <span className="text-xs text-gruvbox-fg-4 ml-auto">worker</span>
                        </>
                      ) : (
                        <>
                          <span className={`w-6 h-6 rounded bg-gruvbox-${item.group.color}/20 text-gruvbox-${item.group.color} flex items-center justify-center text-xs font-bold`}>
                            G
                          </span>
                          <span className="text-gruvbox-fg">{item.group.name}</span>
                          <span className="text-xs text-gruvbox-fg-4 ml-auto">group</span>
                        </>
                      )}
                    </button>
                  );
                })
              )}
            </div>
          )}
        </div>

        {/* Footer Hints */}
        <div className="px-4 py-2 border-t border-gruvbox-fg-4/20 flex items-center gap-4 text-xs text-gruvbox-fg-4">
          <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">Enter</kbd> select/confirm</span>
          <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">&</kbd> AND</span>
          <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">|</kbd> OR</span>
          <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">Backspace</kbd> remove</span>
          <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">Esc</kbd> cancel</span>
        </div>
      </div>
    </div>
  );
});

AssignmentBuilder.displayName = 'AssignmentBuilder';
```

#### 3. Update Barrel Export
**File**: `factory-floor/src/components/assignment/index.ts`

```typescript
export { ExpressionRenderer } from './ExpressionRenderer';
export { AssignmentChip } from './AssignmentChip';
export { AssignmentBuilder } from './AssignmentBuilder';
```

#### 4. Integrate AssignmentBuilder into Editor
**File**: `factory-floor/src/App.tsx`
**Changes**: Show builder when in assign mode, wire up save

```typescript
import { AssignmentBuilder } from './components/assignment';

// In Editor component, add state for assignment editing:
const [assigningNodeId, setAssigningNodeId] = useState<string | null>(null);

// Update mode change handler:
const { activeMode, enterMode, exitMode } = useKeyboardMode({
  onModeChange: (mode) => {
    if (mode === 'assign' && selectedNodes.length === 1) {
      setAssigningNodeId(selectedNodes[0]);
    } else if (mode === 'normal') {
      setAssigningNodeId(null);
    }
  }
});

// Add handler for constraint save:
const handleAssignmentSave = useCallback((constraint: AssignmentConstraint | undefined) => {
  if (assigningNodeId) {
    setNodes((nds) =>
      nds.map((n) => {
        if (n.id === assigningNodeId) {
          return {
            ...n,
            data: {
              ...n.data,
              assignmentConstraint: constraint,
              owner: undefined, // Clear legacy field
            },
          };
        }
        return n;
      })
    );
  }
  exitMode();
}, [assigningNodeId, setNodes, exitMode]);

// In render, add the builder:
{activeMode === 'assign' && assigningNodeId && (
  <AssignmentBuilder
    initialConstraint={nodes.find(n => n.id === assigningNodeId)?.data.assignmentConstraint}
    onConfirm={handleAssignmentSave}
    onCancel={exitMode}
  />
)}
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `npm run build`
- [ ] No console errors

#### Manual Verification:
- [ ] Pressing 'a' on selected node opens assignment builder
- [ ] Typing filters workers and groups with fuzzy search
- [ ] Arrow keys navigate results, Enter selects
- [ ] Selected items appear as chips in expression area
- [ ] Pressing '&' adds AND operator between items
- [ ] Pressing '|' adds OR operator between items
- [ ] Backspace removes last token
- [ ] Enter with valid expression saves to node
- [ ] Escape cancels and closes builder
- [ ] Saved constraint displays on node as chips
- [ ] Re-opening builder shows existing constraint

---

## Phase 5: Workers Management Modal (Command Palette Style)

### Overview
Full CRUD interface for managing workers and groups. Opens with `Cmd+K` (or `w`), keyboard-navigable, command palette style modal. Allows creating/editing/deleting workers and groups.

### Changes Required

#### 1. Create WorkerCard Component
**File**: `factory-floor/src/components/workers/WorkerCard.tsx` (new file)

```typescript
import { memo } from 'react';
import { Worker, useAppStore } from '../../store/appStore';
import clsx from 'clsx';

interface WorkerCardProps {
  worker: Worker;
  isSelected: boolean;
  onClick: () => void;
}

export const WorkerCard = memo(({ worker, isSelected, onClick }: WorkerCardProps) => {
  const groups = useAppStore((s) => s.groups);
  const workerGroups = groups.filter((g) => worker.groupIds.includes(g.id));

  return (
    <button
      onClick={onClick}
      className={clsx(
        'w-full text-left p-2 rounded flex items-center gap-3 transition-colors',
        isSelected
          ? 'bg-gruvbox-aqua/20 border border-gruvbox-aqua'
          : 'hover:bg-gruvbox-bg-soft border border-transparent'
      )}
    >
      {/* Avatar */}
      <div className={`w-8 h-8 rounded-full bg-gruvbox-${worker.color}/20 text-gruvbox-${worker.color} flex items-center justify-center text-sm font-bold`}>
        {worker.initials}
      </div>

      {/* Info */}
      <div className="flex-1 min-w-0">
        <div className="text-sm font-medium text-gruvbox-fg truncate">
          {worker.name}
        </div>
        {workerGroups.length > 0 && (
          <div className="flex items-center gap-1 mt-0.5">
            {workerGroups.slice(0, 2).map((g) => (
              <span key={g.id} className={`text-[10px] px-1 py-0.5 rounded bg-gruvbox-${g.color}/10 text-gruvbox-${g.color}`}>
                {g.name}
              </span>
            ))}
            {workerGroups.length > 2 && (
              <span className="text-[10px] text-gruvbox-fg-4">+{workerGroups.length - 2}</span>
            )}
          </div>
        )}
      </div>
    </button>
  );
});

WorkerCard.displayName = 'WorkerCard';
```

#### 2. Create GroupCard Component
**File**: `factory-floor/src/components/workers/GroupCard.tsx` (new file)

```typescript
import { memo } from 'react';
import { WorkerGroup, useAppStore } from '../../store/appStore';
import clsx from 'clsx';

interface GroupCardProps {
  group: WorkerGroup;
  isSelected: boolean;
  onClick: () => void;
}

export const GroupCard = memo(({ group, isSelected, onClick }: GroupCardProps) => {
  const getWorkersInGroup = useAppStore((s) => s.getWorkersInGroup);
  const members = getWorkersInGroup(group.id);

  return (
    <button
      onClick={onClick}
      className={clsx(
        'w-full text-left p-2 rounded flex items-center gap-3 transition-colors',
        isSelected
          ? 'bg-gruvbox-purple/20 border border-gruvbox-purple'
          : 'hover:bg-gruvbox-bg-soft border border-transparent'
      )}
    >
      {/* Icon */}
      <div className={`w-8 h-8 rounded bg-gruvbox-${group.color}/20 text-gruvbox-${group.color} flex items-center justify-center text-sm font-bold`}>
        {members.length}
      </div>

      {/* Info */}
      <div className="flex-1 min-w-0">
        <div className="text-sm font-medium text-gruvbox-fg truncate">
          {group.name}
        </div>
        {group.description && (
          <div className="text-[10px] text-gruvbox-fg-4 truncate mt-0.5">
            {group.description}
          </div>
        )}
      </div>
    </button>
  );
});

GroupCard.displayName = 'GroupCard';
```

#### 3. Create WorkerEditor Modal
**File**: `factory-floor/src/components/workers/WorkerEditor.tsx` (new file)

```typescript
import { useState, useEffect, memo } from 'react';
import { Worker, WorkerGroup, useAppStore } from '../../store/appStore';
import { X, Save } from 'lucide-react';
import clsx from 'clsx';

interface WorkerEditorProps {
  worker?: Worker; // undefined for new worker
  onSave: (worker: Worker) => void;
  onClose: () => void;
}

const COLORS = ['red', 'green', 'yellow', 'blue', 'purple', 'aqua', 'orange'];

export const WorkerEditor = memo(({ worker, onSave, onClose }: WorkerEditorProps) => {
  const groups = useAppStore((s) => s.groups);

  const [name, setName] = useState(worker?.name || '');
  const [initials, setInitials] = useState(worker?.initials || '');
  const [color, setColor] = useState(worker?.color || 'aqua');
  const [selectedGroups, setSelectedGroups] = useState<string[]>(worker?.groupIds || []);

  // Auto-generate initials from name
  useEffect(() => {
    if (!worker && name && !initials) {
      const parts = name.split(' ');
      const auto = parts.map((p) => p[0]?.toUpperCase() || '').join('').slice(0, 2);
      setInitials(auto);
    }
  }, [name, initials, worker]);

  const handleSave = () => {
    if (!name.trim()) return;

    onSave({
      id: worker?.id || `w-${crypto.randomUUID().slice(0, 8)}`,
      name: name.trim(),
      initials: initials.trim() || name.slice(0, 2).toUpperCase(),
      color,
      groupIds: selectedGroups,
    });
  };

  const toggleGroup = (groupId: string) => {
    setSelectedGroups((prev) =>
      prev.includes(groupId)
        ? prev.filter((id) => id !== groupId)
        : [...prev, groupId]
    );
  };

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50" onClick={onClose}>
      <div
        className="bg-gruvbox-bg border border-gruvbox-fg-4/20 rounded-lg shadow-xl w-full max-w-md"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="flex items-center justify-between p-4 border-b border-gruvbox-fg-4/20">
          <h2 className="text-lg font-bold text-gruvbox-fg">
            {worker ? 'Edit Worker' : 'New Worker'}
          </h2>
          <button onClick={onClose} className="text-gruvbox-fg-4 hover:text-gruvbox-fg">
            <X className="w-5 h-5" />
          </button>
        </div>

        {/* Body */}
        <div className="p-4 space-y-4">
          {/* Name */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Name</label>
            <input
              type="text"
              value={name}
              onChange={(e) => setName(e.target.value)}
              placeholder="e.g., Charlie Rodriguez"
              autoFocus
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg focus:outline-none focus:border-gruvbox-aqua"
            />
          </div>

          {/* Initials */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Initials</label>
            <input
              type="text"
              value={initials}
              onChange={(e) => setInitials(e.target.value.toUpperCase().slice(0, 3))}
              placeholder="CR"
              maxLength={3}
              className="w-24 bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg focus:outline-none focus:border-gruvbox-aqua"
            />
          </div>

          {/* Color */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Color</label>
            <div className="flex gap-2">
              {COLORS.map((c) => (
                <button
                  key={c}
                  onClick={() => setColor(c)}
                  className={clsx(
                    `w-8 h-8 rounded-full bg-gruvbox-${c} transition-all`,
                    color === c ? 'ring-2 ring-gruvbox-fg ring-offset-2 ring-offset-gruvbox-bg' : 'opacity-50 hover:opacity-75'
                  )}
                />
              ))}
            </div>
          </div>

          {/* Groups */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Groups</label>
            <div className="flex flex-wrap gap-2">
              {groups.map((g) => (
                <button
                  key={g.id}
                  onClick={() => toggleGroup(g.id)}
                  className={clsx(
                    'px-2 py-1 rounded text-xs font-medium transition-colors',
                    selectedGroups.includes(g.id)
                      ? `bg-gruvbox-${g.color}/30 text-gruvbox-${g.color} border border-gruvbox-${g.color}`
                      : 'bg-gruvbox-bg-hard text-gruvbox-fg-4 border border-gruvbox-fg-4/20 hover:border-gruvbox-fg-4/40'
                  )}
                >
                  {g.name}
                </button>
              ))}
              {groups.length === 0 && (
                <span className="text-xs text-gruvbox-fg-4">No groups yet</span>
              )}
            </div>
          </div>
        </div>

        {/* Footer */}
        <div className="flex justify-end gap-2 p-4 border-t border-gruvbox-fg-4/20">
          <button onClick={onClose} className="px-4 py-2 text-gruvbox-fg-4 hover:text-gruvbox-fg">
            Cancel
          </button>
          <button
            onClick={handleSave}
            disabled={!name.trim()}
            className="px-4 py-2 bg-gruvbox-aqua text-gruvbox-bg-hard rounded font-bold hover:bg-gruvbox-aqua/80 flex items-center gap-2 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <Save className="w-4 h-4" />
            Save
          </button>
        </div>
      </div>
    </div>
  );
});

WorkerEditor.displayName = 'WorkerEditor';
```

#### 4. Create GroupEditor Modal
**File**: `factory-floor/src/components/workers/GroupEditor.tsx` (new file)

```typescript
import { useState, memo } from 'react';
import { WorkerGroup } from '../../store/appStore';
import { X, Save } from 'lucide-react';
import clsx from 'clsx';

interface GroupEditorProps {
  group?: WorkerGroup;
  onSave: (group: WorkerGroup) => void;
  onClose: () => void;
}

const COLORS = ['red', 'green', 'yellow', 'blue', 'purple', 'aqua', 'orange'];

export const GroupEditor = memo(({ group, onSave, onClose }: GroupEditorProps) => {
  const [name, setName] = useState(group?.name || '');
  const [color, setColor] = useState(group?.color || 'purple');
  const [description, setDescription] = useState(group?.description || '');

  const handleSave = () => {
    if (!name.trim()) return;

    onSave({
      id: group?.id || `grp-${crypto.randomUUID().slice(0, 8)}`,
      name: name.trim(),
      color,
      description: description.trim() || undefined,
    });
  };

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50" onClick={onClose}>
      <div
        className="bg-gruvbox-bg border border-gruvbox-fg-4/20 rounded-lg shadow-xl w-full max-w-md"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="flex items-center justify-between p-4 border-b border-gruvbox-fg-4/20">
          <h2 className="text-lg font-bold text-gruvbox-fg">
            {group ? 'Edit Group' : 'New Group'}
          </h2>
          <button onClick={onClose} className="text-gruvbox-fg-4 hover:text-gruvbox-fg">
            <X className="w-5 h-5" />
          </button>
        </div>

        {/* Body */}
        <div className="p-4 space-y-4">
          {/* Name */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Name</label>
            <input
              type="text"
              value={name}
              onChange={(e) => setName(e.target.value)}
              placeholder="e.g., CNC Team"
              autoFocus
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg focus:outline-none focus:border-gruvbox-purple"
            />
          </div>

          {/* Color */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Color</label>
            <div className="flex gap-2">
              {COLORS.map((c) => (
                <button
                  key={c}
                  onClick={() => setColor(c)}
                  className={clsx(
                    `w-8 h-8 rounded bg-gruvbox-${c} transition-all`,
                    color === c ? 'ring-2 ring-gruvbox-fg ring-offset-2 ring-offset-gruvbox-bg' : 'opacity-50 hover:opacity-75'
                  )}
                />
              ))}
            </div>
          </div>

          {/* Description */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Description</label>
            <input
              type="text"
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              placeholder="Optional description..."
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg focus:outline-none focus:border-gruvbox-purple"
            />
          </div>
        </div>

        {/* Footer */}
        <div className="flex justify-end gap-2 p-4 border-t border-gruvbox-fg-4/20">
          <button onClick={onClose} className="px-4 py-2 text-gruvbox-fg-4 hover:text-gruvbox-fg">
            Cancel
          </button>
          <button
            onClick={handleSave}
            disabled={!name.trim()}
            className="px-4 py-2 bg-gruvbox-purple text-gruvbox-bg-hard rounded font-bold hover:bg-gruvbox-purple/80 flex items-center gap-2 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            <Save className="w-4 h-4" />
            Save
          </button>
        </div>
      </div>
    </div>
  );
});

GroupEditor.displayName = 'GroupEditor';
```

#### 5. Create WorkersModal Component (Command Palette Style)
**File**: `factory-floor/src/components/workers/WorkersModal.tsx` (new file)

```typescript
import { useState, useEffect, useCallback, useRef, memo } from 'react';
import { Worker, WorkerGroup, useAppStore } from '../../store/appStore';
import { WorkerCard } from './WorkerCard';
import { GroupCard } from './GroupCard';
import { WorkerEditor } from './WorkerEditor';
import { GroupEditor } from './GroupEditor';
import { Users, Layers, Search } from 'lucide-react';
import { fuzzySearch } from '../../lib/fuzzySearch';
import clsx from 'clsx';

interface WorkersModalProps {
  onClose: () => void;
}

type Tab = 'workers' | 'groups';
type ListItem = { type: 'worker'; worker: Worker } | { type: 'group'; group: WorkerGroup };

export const WorkersModal = memo(({ onClose }: WorkersModalProps) => {
  const workers = useAppStore((s) => s.workers);
  const groups = useAppStore((s) => s.groups);
  const addWorker = useAppStore((s) => s.addWorker);
  const updateWorker = useAppStore((s) => s.updateWorker);
  const deleteWorker = useAppStore((s) => s.deleteWorker);
  const addGroup = useAppStore((s) => s.addGroup);
  const updateGroup = useAppStore((s) => s.updateGroup);
  const deleteGroup = useAppStore((s) => s.deleteGroup);

  const [activeTab, setActiveTab] = useState<Tab>('workers');
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedIndex, setSelectedIndex] = useState(0);
  const [editingWorker, setEditingWorker] = useState<Worker | 'new' | null>(null);
  const [editingGroup, setEditingGroup] = useState<WorkerGroup | 'new' | null>(null);

  const inputRef = useRef<HTMLInputElement>(null);

  // Focus input on mount
  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  // Build filtered list based on tab and search
  const filteredItems: ListItem[] = (() => {
    const items: ListItem[] = activeTab === 'workers'
      ? workers.map((w) => ({ type: 'worker' as const, worker: w }))
      : groups.map((g) => ({ type: 'group' as const, group: g }));

    if (!searchQuery) return items;

    const results = fuzzySearch(items, searchQuery, (item) =>
      item.type === 'worker' ? item.worker.name : item.group.name
    );
    return results.map((r) => r.item);
  })();

  // Reset selection when filter changes
  useEffect(() => {
    setSelectedIndex(0);
  }, [searchQuery, activeTab]);

  // Keyboard navigation
  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    if (editingWorker || editingGroup) return;

    switch (e.key) {
      case 'ArrowDown':
      case 'j': {
        if (e.key === 'j' && searchQuery) break; // Allow typing 'j' in search
        e.preventDefault();
        setSelectedIndex((i) => Math.min(i + 1, filteredItems.length - 1));
        break;
      }

      case 'ArrowUp':
      case 'k': {
        if (e.key === 'k' && searchQuery) break; // Allow typing 'k' in search
        e.preventDefault();
        setSelectedIndex((i) => Math.max(i - 1, 0));
        break;
      }

      case 'Enter': {
        e.preventDefault();
        const selected = filteredItems[selectedIndex];
        if (selected) {
          if (selected.type === 'worker') {
            setEditingWorker(selected.worker);
          } else {
            setEditingGroup(selected.group);
          }
        }
        break;
      }

      case 'n': {
        if (searchQuery) break; // Allow typing 'n' in search
        e.preventDefault();
        setEditingWorker('new');
        break;
      }

      case 'g': {
        if (searchQuery) break; // Allow typing 'g' in search
        e.preventDefault();
        setEditingGroup('new');
        break;
      }

      case 'd': {
        if (searchQuery) break; // Allow typing 'd' in search
        e.preventDefault();
        const selected = filteredItems[selectedIndex];
        if (selected && confirm('Delete this item?')) {
          if (selected.type === 'worker') {
            deleteWorker(selected.worker.id);
          } else {
            deleteGroup(selected.group.id);
          }
        }
        break;
      }

      case 'Tab': {
        e.preventDefault();
        setActiveTab((t) => t === 'workers' ? 'groups' : 'workers');
        setSearchQuery('');
        break;
      }

      case 'Escape': {
        e.preventDefault();
        onClose();
        break;
      }
    }
  }, [searchQuery, filteredItems, selectedIndex, editingWorker, editingGroup, deleteWorker, deleteGroup, onClose]);

  // Handle worker save
  const handleWorkerSave = useCallback((worker: Worker) => {
    if (editingWorker === 'new') {
      addWorker(worker);
    } else {
      updateWorker(worker.id, worker);
    }
    setEditingWorker(null);
    inputRef.current?.focus();
  }, [editingWorker, addWorker, updateWorker]);

  // Handle group save
  const handleGroupSave = useCallback((group: WorkerGroup) => {
    if (editingGroup === 'new') {
      addGroup(group);
    } else {
      updateGroup(group.id, group);
    }
    setEditingGroup(null);
    inputRef.current?.focus();
  }, [editingGroup, addGroup, updateGroup]);

  return (
    <>
      <div className="fixed inset-0 bg-black/50 flex items-start justify-center pt-[15vh] z-50" onClick={onClose}>
        <div
          className="bg-gruvbox-bg border border-gruvbox-purple rounded-lg shadow-2xl w-full max-w-xl overflow-hidden"
          onClick={(e) => e.stopPropagation()}
        >
          {/* Search Header */}
          <div className="flex items-center gap-3 px-4 py-3 border-b border-gruvbox-fg-4/20">
            <Search className="w-4 h-4 text-gruvbox-fg-4" />
            <input
              ref={inputRef}
              type="text"
              value={searchQuery}
              onChange={(e) => setSearchQuery(e.target.value)}
              onKeyDown={handleKeyDown}
              placeholder="Search workers and groups..."
              className="flex-1 bg-transparent border-none outline-none text-gruvbox-fg placeholder:text-gruvbox-fg-4/50"
            />
            <div className="flex items-center gap-1">
              <button
                onClick={() => { setActiveTab('workers'); setSearchQuery(''); }}
                className={clsx(
                  'px-2 py-1 text-xs font-bold rounded flex items-center gap-1 transition-colors',
                  activeTab === 'workers'
                    ? 'bg-gruvbox-aqua/20 text-gruvbox-aqua'
                    : 'text-gruvbox-fg-4 hover:text-gruvbox-fg'
                )}
              >
                <Users className="w-3 h-3" />
                Workers
              </button>
              <button
                onClick={() => { setActiveTab('groups'); setSearchQuery(''); }}
                className={clsx(
                  'px-2 py-1 text-xs font-bold rounded flex items-center gap-1 transition-colors',
                  activeTab === 'groups'
                    ? 'bg-gruvbox-purple/20 text-gruvbox-purple'
                    : 'text-gruvbox-fg-4 hover:text-gruvbox-fg'
                )}
              >
                <Layers className="w-3 h-3" />
                Groups
              </button>
            </div>
          </div>

          {/* List */}
          <div className="max-h-80 overflow-y-auto p-2">
            {filteredItems.length === 0 ? (
              <div className="text-center text-gruvbox-fg-4 text-sm py-8">
                {searchQuery ? (
                  <>No matches found</>
                ) : activeTab === 'workers' ? (
                  <>No workers yet. Press <kbd className="px-1 bg-gruvbox-bg-hard rounded">n</kbd> to add one.</>
                ) : (
                  <>No groups yet. Press <kbd className="px-1 bg-gruvbox-bg-hard rounded">g</kbd> to add one.</>
                )}
              </div>
            ) : (
              <div className="space-y-1">
                {filteredItems.map((item, i) => (
                  item.type === 'worker' ? (
                    <WorkerCard
                      key={item.worker.id}
                      worker={item.worker}
                      isSelected={i === selectedIndex}
                      onClick={() => {
                        setSelectedIndex(i);
                        setEditingWorker(item.worker);
                      }}
                    />
                  ) : (
                    <GroupCard
                      key={item.group.id}
                      group={item.group}
                      isSelected={i === selectedIndex}
                      onClick={() => {
                        setSelectedIndex(i);
                        setEditingGroup(item.group);
                      }}
                    />
                  )
                ))}
              </div>
            )}
          </div>

          {/* Footer hints */}
          <div className="px-4 py-2 border-t border-gruvbox-fg-4/20 flex items-center gap-4 text-xs text-gruvbox-fg-4">
            <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">↑↓</kbd> navigate</span>
            <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">Enter</kbd> edit</span>
            <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">n</kbd> new worker</span>
            <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">g</kbd> new group</span>
            <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">Tab</kbd> switch</span>
            <span><kbd className="px-1 bg-gruvbox-bg-hard rounded">Esc</kbd> close</span>
          </div>
        </div>
      </div>

      {/* Sub-modals */}
      {editingWorker && (
        <WorkerEditor
          worker={editingWorker === 'new' ? undefined : editingWorker}
          onSave={handleWorkerSave}
          onClose={() => { setEditingWorker(null); inputRef.current?.focus(); }}
        />
      )}

      {editingGroup && (
        <GroupEditor
          group={editingGroup === 'new' ? undefined : editingGroup}
          onSave={handleGroupSave}
          onClose={() => { setEditingGroup(null); inputRef.current?.focus(); }}
        />
      )}
    </>
  );
});

WorkersModal.displayName = 'WorkersModal';
```

#### 6. Create Barrel Export
**File**: `factory-floor/src/components/workers/index.ts` (new file)

```typescript
export { WorkerCard } from './WorkerCard';
export { GroupCard } from './GroupCard';
export { WorkerEditor } from './WorkerEditor';
export { GroupEditor } from './GroupEditor';
export { WorkersModal } from './WorkersModal';
```

#### 7. Integrate Workers Modal into App
**File**: `factory-floor/src/App.tsx`
**Changes**: Show modal when in workers mode

```typescript
import { WorkersModal } from './components/workers';

// In Editor component's render, add workers modal (conditionally):
{activeMode === 'workers' && (
  <WorkersModal onClose={exitMode} />
)}
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `npm run build`
- [ ] No console errors

#### Manual Verification:
- [ ] Pressing `Cmd+K` or `w` opens workers modal (command palette style)
- [ ] Modal appears centered near top of screen
- [ ] Search input is focused immediately
- [ ] Typing filters workers/groups with fuzzy search
- [ ] Tab switches between Workers and Groups tabs
- [ ] Arrow keys navigate through list
- [ ] Enter opens editor for selected item
- [ ] 'n' opens new worker form (when search is empty)
- [ ] 'g' opens new group form (when search is empty)
- [ ] 'd' deletes selected item with confirmation (when search is empty)
- [ ] Escape closes modal
- [ ] Worker editor allows setting name, initials, color, groups
- [ ] Group editor allows setting name, color, description
- [ ] Saved workers/groups appear in assignment builder search

---

## Phase 6: Integration & Polish

### Overview
Wire everything together, fix edge cases, ensure all modes work harmoniously, and add finishing touches.

### Changes Required

#### 1. Update NodeEditModal to Use AssignmentBuilder
**File**: `factory-floor/src/components/nodes/NodeEditModal.tsx`
**Changes**: Replace owner text input with button to open assignment builder, show current constraint

```typescript
import { AssignmentConstraint } from '../../store/appStore';
import { ExpressionRenderer } from '../assignment';

// Add to props:
interface NodeEditModalProps {
  // existing props...
  onOpenAssignment?: () => void;
}

// In the form, replace owner input with:
{/* Assignment */}
<div>
  <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Assignment</label>
  {formData.assignmentConstraint ? (
    <div className="flex items-center gap-2">
      <div className="flex-1 p-2 bg-gruvbox-bg-hard rounded border border-gruvbox-fg-4/20">
        <ExpressionRenderer constraint={formData.assignmentConstraint} />
      </div>
      <button
        onClick={onOpenAssignment}
        className="px-3 py-2 text-xs text-gruvbox-aqua hover:bg-gruvbox-aqua/10 rounded"
      >
        Edit
      </button>
    </div>
  ) : (
    <button
      onClick={onOpenAssignment}
      className="w-full text-left p-2 bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded text-gruvbox-fg-4 hover:border-gruvbox-aqua hover:text-gruvbox-aqua transition-colors"
    >
      + Add assignment constraint
    </button>
  )}
</div>
```

#### 2. Add Vim-Style j/k Navigation for Nodes
**File**: `factory-floor/src/App.tsx`
**Changes**: Add j/k navigation through nodes in the graph

```typescript
// In keyboard handler, add j/k navigation:
if ((event.key === 'j' || event.key === 'k') && activeMode === 'normal') {
  event.preventDefault();

  // Get nodes sorted by position (top-left to bottom-right)
  const sortedNodes = [...nodes].sort((a, b) => {
    if (Math.abs(a.position.y - b.position.y) < 50) {
      return a.position.x - b.position.x;
    }
    return a.position.y - b.position.y;
  });

  const currentIdx = sortedNodes.findIndex((n) => n.id === selectedNodes[0]);

  if (event.key === 'j') {
    const nextIdx = currentIdx < 0 ? 0 : Math.min(currentIdx + 1, sortedNodes.length - 1);
    setSelectedNodes([sortedNodes[nextIdx].id]);
  } else {
    const prevIdx = currentIdx < 0 ? 0 : Math.max(currentIdx - 1, 0);
    setSelectedNodes([sortedNodes[prevIdx].id]);
  }
  return;
}
```

#### 3. Add Visual Selection Indicator
**File**: `factory-floor/src/components/nodes/OperationNode.tsx`
**Changes**: Make selected state more prominent

```typescript
// Update the container className to emphasize selection:
className={clsx(
  "w-64 rounded-lg shadow-lg border-2 transition-all duration-200 overflow-hidden bg-gruvbox-bg",
  getTypeStyles(data.type),
  selected && "ring-2 ring-gruvbox-aqua ring-offset-2 ring-offset-gruvbox-bg-hard border-gruvbox-aqua",
  "group"
)}
```

#### 4. Ensure Clean Mode Transitions
**File**: `factory-floor/src/App.tsx`
**Changes**: Handle mode transitions cleanly, prevent conflicts

```typescript
// Update mode change callback to handle all cleanup:
const { activeMode, enterMode, exitMode } = useKeyboardMode({
  onModeChange: (mode) => {
    // Clean up previous mode
    if (mode === 'normal') {
      setAssigningNodeId(null);
      // Any other cleanup
    }

    // Set up new mode
    if (mode === 'assign' && selectedNodes.length === 1) {
      setAssigningNodeId(selectedNodes[0]);
    }
  }
});
```

#### 5. Add Edge Mode for Creating Connections
**File**: `factory-floor/src/App.tsx`
**Changes**: Add 'e' key to enter edge creation mode

```typescript
// In keyboard handler:
if (event.key === 'e' && selectedNodes.length === 1 && activeMode === 'normal') {
  event.preventDefault();
  enterMode('edge');
  // Store source node for edge creation
  setEdgeSourceId(selectedNodes[0]);
  return;
}

// Add click handler for edge mode:
const onNodeClick = useCallback((event: React.MouseEvent, node: Node) => {
  if (activeMode === 'edge' && edgeSourceId && node.id !== edgeSourceId) {
    // Create edge from source to clicked node
    setEdges((eds) => addEdge({
      id: crypto.randomUUID(),
      source: edgeSourceId,
      target: node.id,
      type: 'smoothstep',
      animated: true,
      style: { stroke: '#a89984', strokeWidth: 2 }
    }, eds));
    exitMode();
  } else if (activeMode === 'normal') {
    // Normal selection behavior
    setSelectedNodes([node.id]);
  }
}, [activeMode, edgeSourceId, exitMode, setEdges]);
```

### Success Criteria

#### Automated Verification:
- [ ] TypeScript compiles: `npm run build`
- [ ] Lint passes: `npm run lint`
- [ ] No console errors or warnings

#### Manual Verification:
- [ ] All keyboard shortcuts work as documented
- [ ] j/k navigation moves through nodes logically
- [ ] Selection visual indicator is clear and prominent
- [ ] Mode transitions are clean (no stuck states)
- [ ] 'e' key enters edge mode, clicking target creates edge
- [ ] Assignment builder accessible from NodeEditModal
- [ ] All hint bar modes show correct hints
- [ ] Workers modal integrates with rest of app
- [ ] Cmd+K doesn't conflict with browser shortcuts
- [ ] No regressions in existing functionality

---

## Testing Strategy

### Unit Tests
- Fuzzy search algorithm correctness
- Constraint expression building/parsing
- Store actions (add/update/delete workers/groups)

### Integration Tests
- Mode transitions don't corrupt state
- Constraint saved to node persists across mode changes
- Workers created in modal appear in assignment search

### Manual Testing Steps
1. Open app, verify hint bar shows NORMAL mode
2. Navigate with j/k, verify selection moves
3. Press 'a' on node, verify assignment builder opens
4. Search for worker, select, add AND, add group
5. Confirm, verify constraint displays on node
6. Press `Cmd+K`, verify workers modal opens (command palette style)
7. Search for a worker, verify fuzzy search works
8. Press 'n' (with empty search), create new worker
9. Press 'g', create new group, assign worker to it
10. Close modal, open assignment builder
11. Search for new group, verify it appears
12. Test all keyboard shortcuts work per hint bar

---

## Performance Considerations

- All components use `memo()` to prevent unnecessary re-renders
- Fuzzy search runs synchronously but is fast for < 1000 items
- Zustand updates are efficient (only affected components re-render)
- No major performance concerns for current scale

---

## Migration Notes

- Existing `owner: string` field preserved for backward compatibility
- Nodes with only `owner` field continue to display normally
- New `assignmentConstraint` field takes precedence when present
- No data migration needed - enhancement is additive

---

## References

- Keyboard handling pattern: `factory-floor/src/App.tsx:380-451`
- Modal pattern: `factory-floor/src/components/nodes/NodeEditModal.tsx`
- Node display: `factory-floor/src/components/nodes/OperationNode.tsx`
- Existing owner data: `factory-floor/src/data/mockShopData.ts:110-123`
- Color system: `factory-floor/tailwind.config.js` (gruvbox palette)
