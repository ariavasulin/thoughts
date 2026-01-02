# Arbitrary DAG Editing with Marks, Connect, and Move

## Overview

Enable power users to build any DAG structure using keyboard-only operations. The current system (`s` for split, `J` for join, `o`/`O` for insert) cannot create arbitrary graphs - splits always share children, joins are destructive, and inserts are linear. This plan adds vim-style marks for node referencing, edge creation/deletion operations, and node repositioning commands.

## Current State Analysis

### Existing Operations and Their Limits

| Operation | Key | Limitation |
|-----------|-----|------------|
| `splitNode` | `s` | Requires children, forces immediate reconvergence |
| `joinNodes` | `J` | Destructive (replaces all children), requires multi-select |
| `insertNode` | `o`/`O` | Always linear (1 parent or 1 child only) |

### Impossible Structures Today
- Divergent branches that never reconverge
- Branches converging at different nodes
- Skip-level edges (A → D, skipping intermediate nodes)
- Asymmetric branch depths
- Arbitrary parent/child connections

### Key Files
- `millflow/src/store/appStore.ts` - State and all node operations
- `millflow/src/hooks/useKeyboard.ts` - Key sequences and handlers
- `millflow/src/components/ShortcutBar.tsx` - Context-aware shortcut display
- `millflow/src/components/HelpModal.tsx` - Full keyboard reference
- `millflow/src/components/dag/DAGNode.tsx` - Node rendering (for mark indicators)

## Desired End State

After implementation, users can:
1. **Mark any node** with `m {a-z}` and jump to it with `' {a-z}`
2. **Create edges** from any node to any other with `c {a-z}` / `C {a-z}`
3. **Remove edges** between nodes with `x {a-z}`
4. **Move nodes** up/down levels with `M k` / `M j`
5. **Relocate nodes** relative to marks with `> {a-z}` / `< {a-z}`
6. **Split leaf nodes** - `s` works even without children (creates sibling)

### New Keyboard Shortcuts Summary

| Key | Action | Description |
|-----|--------|-------------|
| `m {a-z}` | Set mark | Store current node as named mark |
| `' {a-z}` | Jump to mark | Navigate to marked node |
| `c {a-z}` | Connect from mark | Create edge FROM mark TO current |
| `C {a-z}` | Connect to mark | Create edge FROM current TO mark |
| `x {a-z}` | Disconnect | Remove edge between mark and current |
| `M k` | Move up | Move node to earlier level |
| `M j` | Move down | Move node to later level |
| `> {a-z}` | Move after mark | Relocate node after marked node |
| `< {a-z}` | Move before mark | Relocate node before marked node |

### Verification
- `cd millflow && npm run build` passes
- `cd millflow && npm run lint` passes
- All new shortcuts appear in HelpModal under "DAG Editing"
- ShortcutBar shows mark-related shortcuts when in sheet view
- Visual mark indicators appear on marked nodes

## What We're NOT Doing

- Mouse/touch edge creation (keyboard-only focus)
- Drag-and-drop node repositioning
- Edge labels or edge-specific styling
- Cycle detection/prevention (trust the PM to build valid DAGs)
- Cross-sheet edges
- Undo granularity changes (existing history system is sufficient)

---

## Phase 1: State Foundation - Marks and Edge Operations

### Overview
Add marks state to the store and implement core edge manipulation functions.

### Changes Required:

#### 1. Add Marks State to AppStore
**File**: `millflow/src/store/appStore.ts`

Add to `AppState` interface (after line 32):
```typescript
// Marks (vim-style node references)
nodeMarks: Map<string, string>; // mark letter -> nodeId
pendingMarkAction: 'set' | 'jump' | 'connect-from' | 'connect-to' | 'disconnect' | 'move-after' | 'move-before' | null;
```

Add to initial state (after line 135):
```typescript
nodeMarks: new Map(),
pendingMarkAction: null,
```

Add action signatures to interface (after line 102):
```typescript
// Mark Actions
setMark: (mark: string, nodeId: string) => void;
jumpToMark: (mark: string) => void;
clearMark: (mark: string) => void;
clearAllMarks: () => void;
setPendingMarkAction: (action: AppState['pendingMarkAction']) => void;

// Edge Actions (non-destructive)
connectNodes: (fromNodeId: string, toNodeId: string) => void;
disconnectNodes: (nodeId1: string, nodeId2: string) => void;

// Move Actions
moveNodeUp: (sheetId: string, nodeId: string) => void;
moveNodeDown: (sheetId: string, nodeId: string) => void;
moveNodeAfter: (sheetId: string, nodeId: string, targetNodeId: string) => void;
moveNodeBefore: (sheetId: string, nodeId: string, targetNodeId: string) => void;
```

#### 2. Implement Mark Actions
**File**: `millflow/src/store/appStore.ts`

Add after existing actions:
```typescript
// Mark Actions
setMark: (mark, nodeId) => set(state => ({
  nodeMarks: new Map(state.nodeMarks).set(mark, nodeId),
  pendingMarkAction: null,
})),

jumpToMark: (mark) => {
  const { nodeMarks, sheets, selectedSheetId } = get();
  const nodeId = nodeMarks.get(mark);
  if (!nodeId) return;

  // Find which sheet contains this node
  const sheet = sheets.find(s => s.nodes.some(n => n.id === nodeId));
  if (!sheet) return;

  // Navigate to the node
  const linearized = linearizeNodes(sheet.nodes);
  const idx = linearized.findIndex(n => n.id === nodeId);

  set({
    selectedSheetId: sheet.id,
    selectedNodeId: nodeId,
    selectedIndex: idx >= 0 ? idx : 0,
    pendingMarkAction: null,
  });
},

clearMark: (mark) => set(state => {
  const newMarks = new Map(state.nodeMarks);
  newMarks.delete(mark);
  return { nodeMarks: newMarks };
}),

clearAllMarks: () => set({ nodeMarks: new Map() }),

setPendingMarkAction: (action) => set({ pendingMarkAction: action }),
```

#### 3. Implement Edge Operations
**File**: `millflow/src/store/appStore.ts`

Add `connectNodes` - creates edge without destroying existing connections:
```typescript
connectNodes: (fromNodeId, toNodeId) => {
  const { selectedSheetId } = get();
  if (!selectedSheetId) return;

  get().pushHistory('Connect nodes');
  set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== selectedSheetId) return sheet;

      const fromNode = sheet.nodes.find(n => n.id === fromNodeId);
      const toNode = sheet.nodes.find(n => n.id === toNodeId);
      if (!fromNode || !toNode) return sheet;

      // Prevent self-loops
      if (fromNodeId === toNodeId) return sheet;

      // Check if edge already exists
      if (fromNode.children.includes(toNodeId)) return sheet;

      return {
        ...sheet,
        nodes: sheet.nodes.map(node => {
          if (node.id === fromNodeId) {
            return { ...node, children: [...node.children, toNodeId] };
          }
          if (node.id === toNodeId) {
            return { ...node, parents: [...node.parents, fromNodeId] };
          }
          return node;
        }),
      };
    });

    return { sheets: newSheets, pendingMarkAction: null };
  });
},

disconnectNodes: (nodeId1, nodeId2) => {
  const { selectedSheetId } = get();
  if (!selectedSheetId) return;

  get().pushHistory('Disconnect nodes');
  set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== selectedSheetId) return sheet;

      return {
        ...sheet,
        nodes: sheet.nodes.map(node => {
          if (node.id === nodeId1) {
            return {
              ...node,
              children: node.children.filter(id => id !== nodeId2),
              parents: node.parents.filter(id => id !== nodeId2),
            };
          }
          if (node.id === nodeId2) {
            return {
              ...node,
              children: node.children.filter(id => id !== nodeId1),
              parents: node.parents.filter(id => id !== nodeId1),
            };
          }
          return node;
        }),
      };
    });

    return { sheets: newSheets, pendingMarkAction: null };
  });
},
```

#### 4. Implement Move Operations
**File**: `millflow/src/store/appStore.ts`

Add move operations:
```typescript
moveNodeUp: (sheetId, nodeId) => {
  get().pushHistory('Move node up');
  set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;

      const node = sheet.nodes.find(n => n.id === nodeId);
      if (!node || node.parents.length === 0) return sheet;

      // Get grandparents (parents of our parents)
      const grandparents = new Set<string>();
      node.parents.forEach(parentId => {
        const parent = sheet.nodes.find(n => n.id === parentId);
        if (parent) {
          parent.parents.forEach(gp => grandparents.add(gp));
        }
      });

      // Move node: disconnect from parents, connect to grandparents, become sibling of parents
      return {
        ...sheet,
        nodes: sheet.nodes.map(n => {
          if (n.id === nodeId) {
            // Node gets grandparents as new parents
            return { ...n, parents: [...grandparents] };
          }
          if (node.parents.includes(n.id)) {
            // Former parents: remove node from their children
            return { ...n, children: n.children.filter(c => c !== nodeId) };
          }
          if (grandparents.has(n.id)) {
            // Grandparents: add node as child
            return { ...n, children: [...n.children, nodeId] };
          }
          return n;
        }),
      };
    });
    return { sheets: newSheets };
  });
},

moveNodeDown: (sheetId, nodeId) => {
  get().pushHistory('Move node down');
  set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;

      const node = sheet.nodes.find(n => n.id === nodeId);
      if (!node || node.children.length === 0) return sheet;

      // Get grandchildren (children of our children)
      const grandchildren = new Set<string>();
      node.children.forEach(childId => {
        const child = sheet.nodes.find(n => n.id === childId);
        if (child) {
          child.children.forEach(gc => grandchildren.add(gc));
        }
      });

      // Move node: disconnect from children, connect to grandchildren, become sibling of children
      return {
        ...sheet,
        nodes: sheet.nodes.map(n => {
          if (n.id === nodeId) {
            // Node gets grandchildren as new children, current children as parents
            return {
              ...n,
              parents: [...node.children],
              children: [...grandchildren],
            };
          }
          if (node.children.includes(n.id)) {
            // Former children: become parents of node
            return {
              ...n,
              parents: n.parents.filter(p => p !== nodeId),
              children: [...n.children.filter(c => !grandchildren.has(c)), nodeId],
            };
          }
          if (grandchildren.has(n.id)) {
            // Grandchildren: replace child parents with node
            return {
              ...n,
              parents: [...n.parents.filter(p => !node.children.includes(p)), nodeId],
            };
          }
          return n;
        }),
      };
    });
    return { sheets: newSheets };
  });
},

moveNodeAfter: (sheetId, nodeId, targetNodeId) => {
  if (nodeId === targetNodeId) return;

  get().pushHistory('Move node after');
  set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;

      const node = sheet.nodes.find(n => n.id === nodeId);
      const target = sheet.nodes.find(n => n.id === targetNodeId);
      if (!node || !target) return sheet;

      // Disconnect node from current position
      // Connect node's parents directly to node's children (bridge the gap)
      // Place node after target
      return {
        ...sheet,
        nodes: sheet.nodes.map(n => {
          if (n.id === nodeId) {
            // Node: new parent is target, keep original children
            return { ...n, parents: [targetNodeId] };
          }
          if (node.parents.includes(n.id)) {
            // Old parents: replace node with node's children
            return {
              ...n,
              children: [...n.children.filter(c => c !== nodeId), ...node.children],
            };
          }
          if (node.children.includes(n.id)) {
            // Old children: replace node with node's parents
            return {
              ...n,
              parents: [...n.parents.filter(p => p !== nodeId), ...node.parents],
            };
          }
          if (n.id === targetNodeId) {
            // Target: add node as child
            return { ...n, children: [...n.children, nodeId] };
          }
          return n;
        }),
      };
    });
    return { sheets: newSheets, pendingMarkAction: null };
  });
},

moveNodeBefore: (sheetId, nodeId, targetNodeId) => {
  if (nodeId === targetNodeId) return;

  get().pushHistory('Move node before');
  set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;

      const node = sheet.nodes.find(n => n.id === nodeId);
      const target = sheet.nodes.find(n => n.id === targetNodeId);
      if (!node || !target) return sheet;

      return {
        ...sheet,
        nodes: sheet.nodes.map(n => {
          if (n.id === nodeId) {
            // Node: new child is target, keep no children (they get bridged)
            return { ...n, parents: [...target.parents], children: [targetNodeId] };
          }
          if (node.parents.includes(n.id)) {
            // Old parents: replace node with node's children
            return {
              ...n,
              children: [...n.children.filter(c => c !== nodeId), ...node.children],
            };
          }
          if (node.children.includes(n.id)) {
            // Old children: replace node with node's parents
            return {
              ...n,
              parents: [...n.parents.filter(p => p !== nodeId), ...node.parents],
            };
          }
          if (n.id === targetNodeId) {
            // Target: node becomes only parent
            return { ...n, parents: [nodeId] };
          }
          if (target.parents.includes(n.id)) {
            // Target's old parents: replace target with node
            return {
              ...n,
              children: n.children.map(c => c === targetNodeId ? nodeId : c),
            };
          }
          return n;
        }),
      };
    });
    return { sheets: newSheets, pendingMarkAction: null };
  });
},
```

#### 5. Fix splitNode to Work on Leaf Nodes
**File**: `millflow/src/store/appStore.ts`

Update `splitNode` (around line 678) to handle leaf nodes:
```typescript
splitNode: (sheetId, nodeId) => {
  get().pushHistory('Split node');
  return set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;

      const node = sheet.nodes.find(n => n.id === nodeId);
      if (!node) return sheet;

      // For leaf nodes (no children), create a sibling leaf
      // For nodes with children, create parallel path (existing behavior)

      const newNode: Node = {
        id: `n-${Date.now()}`,
        name: 'Parallel Task',
        type: 'shop',
        status: 'not_started',
        assignees: [],
        blockers: [],
        notes: [],
        references: [],
        parents: [...node.parents], // Same parents as original (sibling)
        children: [...node.children], // Same children (may be empty for leaf)
      };

      const nodes = [...sheet.nodes];

      // Update parents to include new node as child
      for (const parentId of node.parents) {
        const parentIdx = nodes.findIndex(n => n.id === parentId);
        if (parentIdx >= 0 && !nodes[parentIdx].children.includes(newNode.id)) {
          nodes[parentIdx] = {
            ...nodes[parentIdx],
            children: [...nodes[parentIdx].children, newNode.id],
          };
        }
      }

      // Update children to include new node as parent (if any)
      for (const childId of newNode.children) {
        const childIdx = nodes.findIndex(n => n.id === childId);
        if (childIdx >= 0 && !nodes[childIdx].parents.includes(newNode.id)) {
          nodes[childIdx] = {
            ...nodes[childIdx],
            parents: [...nodes[childIdx].parents, newNode.id],
          };
        }
      }

      nodes.push(newNode);
      return { ...sheet, nodes };
    });

    return { sheets: newSheets };
  });
},
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`
- [ ] No TypeScript errors in appStore.ts

#### Manual Verification:
- [ ] Can set marks and jump between them
- [ ] `connectNodes` creates edges without destroying existing ones
- [ ] `disconnectNodes` removes only the specified edge
- [ ] Move operations correctly reposition nodes
- [ ] Split works on leaf nodes (creates sibling)
- [ ] Undo works for all new operations

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Keyboard Handler Integration

### Overview
Add key sequence handling for marks, connect, disconnect, and move operations.

### Changes Required:

#### 1. Update useKeyboard Hook
**File**: `millflow/src/hooks/useKeyboard.ts`

Add new imports and state from store (around line 10):
```typescript
const {
  // ... existing imports ...
  nodeMarks,
  pendingMarkAction,
  setPendingMarkAction,
  setMark,
  jumpToMark,
  connectNodes,
  disconnectNodes,
  moveNodeUp,
  moveNodeDown,
  moveNodeAfter,
  moveNodeBefore,
} = useAppStore();
```

Add helper for mark letter validation (after line 55):
```typescript
const isMarkLetter = (key: string): boolean => {
  return /^[a-z]$/.test(key);
};
```

Update sequence handling to include new prefixes (around line 200):
```typescript
// If sequence is incomplete and might match, wait and show pending indicator
if (['g', 'd', 'm', "'", 'c', 'C', 'x', 'M', '>', '<'].includes(key) && prevSequence.length === 0) {
  setPendingKeySequence(key);
  return;
}
```

Add new sequence handlers (after line 197, before "Reset sequence if no match"):
```typescript
// m {a-z} - set mark
if (sequence.length === 2 && prevSequence[0] === 'm' && isMarkLetter(key)) {
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    setMark(key, selectedNodeId);
  }
  return;
}

// ' {a-z} - jump to mark
if (sequence.length === 2 && prevSequence[0] === "'" && isMarkLetter(key)) {
  event.preventDefault();
  resetSequence();
  jumpToMark(key);
  return;
}

// c {a-z} - connect FROM mark TO current
if (sequence.length === 2 && prevSequence[0] === 'c' && isMarkLetter(key) && !shiftKey) {
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    const fromNodeId = nodeMarks.get(key);
    if (fromNodeId) {
      connectNodes(fromNodeId, selectedNodeId);
    }
  }
  return;
}

// C {a-z} - connect FROM current TO mark
if (sequence.length === 2 && prevSequence[0] === 'C' && isMarkLetter(key)) {
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    const toNodeId = nodeMarks.get(key);
    if (toNodeId) {
      connectNodes(selectedNodeId, toNodeId);
    }
  }
  return;
}

// x {a-z} - disconnect edge with mark
if (sequence.length === 2 && prevSequence[0] === 'x' && isMarkLetter(key)) {
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    const markNodeId = nodeMarks.get(key);
    if (markNodeId) {
      disconnectNodes(selectedNodeId, markNodeId);
    }
  }
  return;
}

// M k or M ↑ - move node up
if (sequence === 'M k' || sequence === 'M ArrowUp') {
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    moveNodeUp(selectedSheetId, selectedNodeId);
  }
  return;
}

// M j or M ↓ - move node down
if (sequence === 'M j' || sequence === 'M ArrowDown') {
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    moveNodeDown(selectedSheetId, selectedNodeId);
  }
  return;
}

// > {a-z} - move current node after mark
if (sequence.length === 2 && prevSequence[0] === '>' && isMarkLetter(key)) {
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    const targetNodeId = nodeMarks.get(key);
    if (targetNodeId) {
      moveNodeAfter(selectedSheetId, selectedNodeId, targetNodeId);
    }
  }
  return;
}

// < {a-z} - move current node before mark
if (sequence.length === 2 && prevSequence[0] === '<' && isMarkLetter(key)) {
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    resetSequence();
    const targetNodeId = nodeMarks.get(key);
    if (targetNodeId) {
      moveNodeBefore(selectedSheetId, selectedNodeId, targetNodeId);
    }
  }
  return;
}
```

Update dependencies array (around line 440):
```typescript
], [
  // ... existing deps ...
  nodeMarks,
  pendingMarkAction,
  setPendingMarkAction,
  setMark,
  jumpToMark,
  connectNodes,
  disconnectNodes,
  moveNodeUp,
  moveNodeDown,
  moveNodeAfter,
  moveNodeBefore,
]);
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] `m a` on a node sets mark 'a'
- [ ] `' a` jumps to mark 'a'
- [ ] `c a` creates edge FROM mark 'a' TO current
- [ ] `C a` creates edge FROM current TO mark 'a'
- [ ] `x a` removes edge between current and mark 'a'
- [ ] `M k` moves node up one level
- [ ] `M j` moves node down one level
- [ ] `> a` moves current node after mark 'a'
- [ ] `< a` moves current node before mark 'a'
- [ ] Pending key indicator shows for `m`, `'`, `c`, `C`, `x`, `M`, `>`, `<`

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: UI Updates - ShortcutBar and HelpModal

### Overview
Update the context-aware shortcut bar and help modal to show all new commands.

### Changes Required:

#### 1. Update ShortcutBar
**File**: `millflow/src/components/ShortcutBar.tsx`

Update `sheetShortcuts` array (around line 32):
```typescript
const sheetShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'nav' },
  { key: 'h/l', label: 'branches' },
  { key: 'o/O', label: 'insert' },
  { key: 's', label: 'split' },
  { key: 'm', label: 'mark' },
  { key: 'c/C', label: 'connect' },
  { key: 'M', label: 'move' },
  { key: '?', label: 'help' },
];
```

Add new mode for when pending mark action exists. First, add import for `pendingMarkAction`:
```typescript
const {
  // ... existing ...
  pendingMarkAction,
} = useAppStore();
```

Add new shortcut set for mark pending state (after line 67):
```typescript
const markPendingShortcuts: Shortcut[] = [
  { key: 'a-z', label: 'select mark' },
  { key: 'esc', label: 'cancel' },
];
```

Update `getShortcuts` function to handle pending mark sequences (add after Priority 6, around line 98):
```typescript
// Priority 6.5: Pending mark action
if (pendingKeySequence && ['m', "'", 'c', 'C', 'x', '>', '<'].includes(pendingKeySequence)) {
  return markPendingShortcuts;
}

// Priority 6.6: Move pending
if (pendingKeySequence === 'M') {
  return [
    { key: 'k/↑', label: 'move up' },
    { key: 'j/↓', label: 'move down' },
    { key: 'esc', label: 'cancel' },
  ];
}
```

#### 2. Update HelpModal
**File**: `millflow/src/components/HelpModal.tsx`

Add new category for marks and update DAG Editing (replace existing categories around line 10):
```typescript
const categories: ShortcutCategory[] = [
  {
    title: 'Global',
    shortcuts: [
      { key: 'Cmd+K', action: 'Command palette' },
      { key: 'g h', action: 'Go home (dashboard)' },
      { key: 'g j', action: 'Go to jobs' },
      { key: 'g d', action: 'Go to deliveries' },
      { key: '?', action: 'Show help' },
      { key: 'esc', action: 'Back / close / cancel' },
    ],
  },
  {
    title: 'Navigation',
    shortcuts: [
      { key: 'j / ↓', action: 'Move down' },
      { key: 'k / ↑', action: 'Move up' },
      { key: 'h / ←', action: 'Move left (branches)' },
      { key: 'l / →', action: 'Move right (branches)' },
      { key: 'enter', action: 'Open / expand' },
      { key: 'g g', action: 'Jump to top' },
      { key: 'G', action: 'Jump to bottom' },
      { key: 'tab', action: 'Next actionable' },
      { key: 'shift+tab', action: 'Previous actionable' },
    ],
  },
  {
    title: 'Marks & Connect',
    shortcuts: [
      { key: 'm {a-z}', action: 'Set mark on current node' },
      { key: "' {a-z}", action: 'Jump to marked node' },
      { key: 'c {a-z}', action: 'Connect FROM mark TO current' },
      { key: 'C {a-z}', action: 'Connect FROM current TO mark' },
      { key: 'x {a-z}', action: 'Disconnect edge with mark' },
    ],
  },
  {
    title: 'DAG Editing',
    shortcuts: [
      { key: 'o', action: 'Insert node after' },
      { key: 'O', action: 'Insert node before' },
      { key: 't', action: 'Insert template' },
      { key: 's', action: 'Split (parallel branch)' },
      { key: 'J', action: 'Join (convergence)' },
      { key: 'M k/↑', action: 'Move node up a level' },
      { key: 'M j/↓', action: 'Move node down a level' },
      { key: '> {a-z}', action: 'Move node after mark' },
      { key: '< {a-z}', action: 'Move node before mark' },
      { key: 'd d', action: 'Delete node' },
      { key: 'u', action: 'Undo' },
      { key: 'ctrl+r', action: 'Redo' },
    ],
  },
  {
    title: 'Node Actions',
    shortcuts: [
      { key: 'a', action: 'Assign' },
      { key: 'b', action: 'Add blocker' },
      { key: 'n', action: 'Add note' },
      { key: 'D', action: 'Set due date' },
      { key: 'space', action: 'Toggle status' },
    ],
  },
  {
    title: 'Multi-Select',
    shortcuts: [
      { key: 'v', action: 'Enter visual mode' },
      { key: 'space', action: 'Toggle selection' },
      { key: 'J', action: 'Join to current' },
      { key: 'd d', action: 'Delete selected' },
      { key: 'esc', action: 'Exit visual mode' },
    ],
  },
];
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] ShortcutBar in sheet view shows mark/connect/move shortcuts
- [ ] Pressing `m` shows "a-z select mark" in shortcut bar
- [ ] Pressing `M` shows "k/↑ move up, j/↓ move down" options
- [ ] HelpModal shows new "Marks & Connect" category
- [ ] HelpModal "DAG Editing" section includes move operations
- [ ] All new shortcuts are documented and match implementation

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Visual Mark Indicators

### Overview
Show which nodes have marks set, displaying the mark letter on the node.

### Changes Required:

#### 1. Update DAGNode Component
**File**: `millflow/src/components/dag/DAGNode.tsx`

Add import for store:
```typescript
import { useAppStore } from '@/store/appStore';
```

Inside the component, get marks:
```typescript
const nodeMarks = useAppStore(state => state.nodeMarks);

// Find if this node has any marks
const marksForNode = Array.from(nodeMarks.entries())
  .filter(([_, nodeId]) => nodeId === node.id)
  .map(([mark, _]) => mark);
```

Add mark indicator in the node header (after the status icon, around line 55):
```typescript
{/* Mark indicators */}
{marksForNode.length > 0 && (
  <div className="flex gap-0.5">
    {marksForNode.map(mark => (
      <span
        key={mark}
        className="inline-flex items-center justify-center w-4 h-4 text-[10px] font-bold rounded bg-gruvbox-purple text-gruvbox-bg"
      >
        {mark}
      </span>
    ))}
  </div>
)}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Setting mark `m a` shows "a" badge on the node
- [ ] Setting multiple marks on same node shows all badges
- [ ] Setting mark on different node moves the badge
- [ ] Mark badges use gruvbox-purple for visibility
- [ ] Marks persist when navigating away and back

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 5.

---

## Phase 5: Edge Case Handling and Polish

### Overview
Handle edge cases and add polish for a robust experience.

### Changes Required:

#### 1. Add Mark Clearing on Node Delete
**File**: `millflow/src/store/appStore.ts`

Update `deleteNode` function to clear marks pointing to deleted node:
```typescript
deleteNode: (sheetId, nodeId) => {
  get().pushHistory('Delete node');
  set(state => {
    // Clear any marks pointing to this node
    const newMarks = new Map(state.nodeMarks);
    for (const [mark, markedNodeId] of newMarks) {
      if (markedNodeId === nodeId) {
        newMarks.delete(mark);
      }
    }

    const newSheets = state.sheets.map(sheet => {
      // ... existing delete logic ...
    });

    return { sheets: newSheets, nodeMarks: newMarks, selectedNodeId: null };
  });
},
```

Also update `deleteSelectedNodes` similarly.

#### 2. Add Feedback for Invalid Operations
**File**: `millflow/src/hooks/useKeyboard.ts`

Add visual/audio feedback when mark doesn't exist or operation is invalid:
```typescript
// In the mark sequence handlers, after checking if mark exists:
if (!fromNodeId) {
  // Mark doesn't exist - could add toast notification here
  resetSequence();
  return;
}
```

#### 3. Prevent Common Invalid DAG States
**File**: `millflow/src/store/appStore.ts`

Add to `connectNodes` to prevent obvious cycles:
```typescript
// Basic cycle prevention: don't connect if target is an ancestor
const isAncestor = (nodeId: string, potentialAncestorId: string, visited = new Set<string>()): boolean => {
  if (visited.has(nodeId)) return false;
  visited.add(nodeId);
  const node = sheet.nodes.find(n => n.id === nodeId);
  if (!node) return false;
  if (node.parents.includes(potentialAncestorId)) return true;
  return node.parents.some(p => isAncestor(p, potentialAncestorId, visited));
};

// In connectNodes, before creating edge:
if (isAncestor(fromNodeId, toNodeId)) {
  // Would create cycle, abort
  return sheet;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Deleting a marked node clears that mark
- [ ] Using a non-existent mark (e.g., `c z` when z not set) does nothing gracefully
- [ ] Cannot create obvious cycles with connect operation
- [ ] Move operations handle edge cases (moving root node up, leaf node down)
- [ ] All operations are undoable

---

## Testing Strategy

### Unit Tests
- Mark set/get/clear operations
- Connect creates bidirectional references
- Disconnect removes bidirectional references
- Move operations maintain graph integrity
- Split works on leaf nodes

### Integration Tests
- Full keyboard sequence flows (mark, navigate, connect)
- Complex DAG building scenarios
- Undo/redo for all new operations

### Manual Testing Steps
1. Create a simple linear DAG: A → B → C → D
2. Mark node A with `m a`
3. Navigate to D with `j j j`
4. Connect A to D with `c a` - verify A → D edge exists
5. Mark node B with `m b`
6. Disconnect B from C with `x b` while on C
7. Move node C up with `M k` - verify C becomes sibling of B
8. Undo all operations with repeated `u` - verify original state restored

### Edge Cases to Test
- Split on root node (no parents)
- Split on leaf node (no children)
- Move up on root node (should be no-op)
- Move down on leaf node (should be no-op)
- Connect node to itself (should be prevented)
- Connect creating cycle (should be prevented)
- Delete node that has marks

## Performance Considerations

- Mark storage uses Map for O(1) lookups
- Edge operations are O(n) where n = nodes in sheet
- No impact on render performance (marks are just state)
- History entries may grow larger with complex DAGs

## References

- Research: `thoughts/shared/research/2025-12-19-split-node-dag-architecture.md`
- Vim marks: https://vim.fandom.com/wiki/Using_marks
- Current keyboard implementation: `millflow/src/hooks/useKeyboard.ts`
- Current store: `millflow/src/store/appStore.ts`
