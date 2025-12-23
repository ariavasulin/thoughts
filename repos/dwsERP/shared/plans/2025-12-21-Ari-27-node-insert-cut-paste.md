# Node Insert/Cut/Paste Keyboard Shortcuts Implementation Plan

## Overview

Implement single-key shortcuts for node insertion, cutting, and pasting operations. This replaces the current sequence-based `o j`, `o k`, `o l`, `o h` with single keys and adds new clipboard functionality.

## Current State Analysis

### Existing Keyboard Mappings (useKeyboard.ts)
- `o j` / `o ArrowDown` → Insert node AFTER (sets `nodeCreationMode: 'after'`)
- `o k` / `o ArrowUp` → Insert node BEFORE (sets `nodeCreationMode: 'before'`)
- `o l` / `o ArrowRight` → Insert sibling to RIGHT
- `o h` / `o ArrowLeft` → Insert sibling to LEFT
- `d d` → Delete node(s)

### Existing Store Functions (appStore.ts)
- `insertNode(sheetId, targetNodeId, position, nodeData)` - handles before/after/left/right
- `deleteNode(sheetId, nodeId)` - removes node and reconnects graph
- No clipboard state exists

## Desired End State

### New Single-Key Mappings
| Key | Action | Description |
|-----|--------|-------------|
| `o` | Insert AFTER | Creates new node below focused node |
| `O` | Insert BEFORE | Creates new node above focused node |
| `S` | Insert sibling RIGHT | Creates parallel node to the right |
| `x` | Cut node | Stores node in clipboard, removes from DAG |
| `p` | Paste AFTER | Inserts clipboard node after focused node |
| `P` | Paste BEFORE | Inserts clipboard node before focused node |
| `s` | Paste RIGHT | Inserts clipboard node as sibling to right |

### Verification
- All new keys work in sheet view when a node is selected
- Cut node is stored in clipboard and can be pasted multiple times
- Paste operations preserve node properties (name, type, status, etc.)
- Undo/redo works for all operations
- `o j`/`o k`/`o l`/`o h` sequences are removed (replaced by single keys)

## What We're NOT Doing

- Copy functionality (only cut for now)
- Paste to left (only paste right/before/after)
- Multi-node cut/paste
- Cross-sheet paste
- Two-edge constraint enforcement (keeping current DAG model)

## Implementation Approach

Phase 1 adds clipboard state and store actions. Phase 2 updates keyboard mappings.

---

## Phase 1: Add Clipboard State and Actions

### Overview
Add clipboard state to the store and implement `cutNode` and `pasteNode` actions.

### Changes Required:

#### 1. Add Clipboard State to AppState Interface
**File**: `millflow/src/store/appStore.ts`
**Lines**: 11-46 (interface definition)

Add after line 35 (after `nodeMarks`):
```typescript
  // Clipboard (for cut/paste)
  clipboard: {
    node: Node | null;
    sourceSheetId: string | null;
  };
```

#### 2. Add Clipboard Actions to AppState Interface
**File**: `millflow/src/store/appStore.ts`
**Lines**: 83-115 (actions section)

Add after line 89 (after `deleteNode`):
```typescript
  cutNode: (sheetId: string, nodeId: string) => void;
  pasteNode: (sheetId: string, targetNodeId: string, position: 'before' | 'after' | 'right') => void;
  clearClipboard: () => void;
```

#### 3. Initialize Clipboard State
**File**: `millflow/src/store/appStore.ts`
**Lines**: 128-160 (initial state)

Add after line 151 (after `nodeMarks: new Map()`):
```typescript
  // Clipboard
  clipboard: {
    node: null,
    sourceSheetId: null,
  },
```

#### 4. Implement cutNode Action
**File**: `millflow/src/store/appStore.ts`
**Location**: After `deleteNode` action (around line 900)

```typescript
  cutNode: (sheetId, nodeId) => {
    const sheet = get().sheets.find(s => s.id === sheetId);
    if (!sheet) return;

    const nodeToCut = sheet.nodes.find(n => n.id === nodeId);
    if (!nodeToCut) return;

    // Store node in clipboard (deep clone to preserve data)
    const clipboardNode: Node = JSON.parse(JSON.stringify(nodeToCut));

    // Clear any marks pointing to the cut node
    const newMarks = new Map(get().nodeMarks);
    for (const [mark, markedNodeId] of newMarks) {
      if (markedNodeId === nodeId) {
        newMarks.delete(mark);
      }
    }

    get().pushHistory('Cut node');

    set(state => {
      const newSheets = state.sheets.map(s => {
        if (s.id !== sheetId) return s;

        const nodeToDelete = s.nodes.find(n => n.id === nodeId);
        if (!nodeToDelete) return s;

        // Reconnect parents to children (same as deleteNode)
        const nodes = s.nodes.filter(n => n.id !== nodeId).map(node => {
          if (nodeToDelete.parents.includes(node.id)) {
            return {
              ...node,
              children: node.children.filter(c => c !== nodeId).concat(nodeToDelete.children),
            };
          }
          if (nodeToDelete.children.includes(node.id)) {
            return {
              ...node,
              parents: node.parents.filter(p => p !== nodeId).concat(nodeToDelete.parents),
            };
          }
          return node;
        });

        return { ...s, nodes };
      });

      return {
        sheets: newSheets,
        selectedNodeId: null,
        nodeMarks: newMarks,
        clipboard: {
          node: clipboardNode,
          sourceSheetId: sheetId,
        },
      };
    });
  },
```

#### 5. Implement pasteNode Action
**File**: `millflow/src/store/appStore.ts`
**Location**: After `cutNode` action

```typescript
  pasteNode: (sheetId, targetNodeId, position) => {
    const { clipboard } = get();
    if (!clipboard.node) return;

    const sheet = get().sheets.find(s => s.id === sheetId);
    if (!sheet) return;

    const targetNode = sheet.nodes.find(n => n.id === targetNodeId);
    if (!targetNode) return;

    get().pushHistory('Paste node');

    // Create new node from clipboard with fresh ID
    const newNodeId = `n-${Date.now()}`;
    const newNode: Node = {
      ...clipboard.node,
      id: newNodeId,
      children: [],
      parents: [],
    };

    set(state => {
      const newSheets = state.sheets.map(s => {
        if (s.id !== sheetId) return s;

        const nodes = [...s.nodes];
        const target = nodes.find(n => n.id === targetNodeId);
        if (!target) return s;

        if (position === 'after') {
          newNode.parents = [targetNodeId];
          newNode.children = [...target.children];

          const targetIdx = nodes.findIndex(n => n.id === targetNodeId);
          nodes[targetIdx] = { ...target, children: [newNodeId] };

          for (const childId of newNode.children) {
            const childIdx = nodes.findIndex(n => n.id === childId);
            if (childIdx >= 0) {
              nodes[childIdx] = {
                ...nodes[childIdx],
                parents: nodes[childIdx].parents.map(p => p === targetNodeId ? newNodeId : p),
              };
            }
          }
        } else if (position === 'before') {
          newNode.children = [targetNodeId];
          newNode.parents = [...target.parents];

          const targetIdx = nodes.findIndex(n => n.id === targetNodeId);
          nodes[targetIdx] = { ...target, parents: [newNodeId] };

          for (const parentId of newNode.parents) {
            const parentIdx = nodes.findIndex(n => n.id === parentId);
            if (parentIdx >= 0) {
              nodes[parentIdx] = {
                ...nodes[parentIdx],
                children: nodes[parentIdx].children.map(c => c === targetNodeId ? newNodeId : c),
              };
            }
          }
        } else {
          // position === 'right' - create sibling
          if (target.parents.length === 0) {
            // Root node fallback - insert after
            newNode.parents = [targetNodeId];
            newNode.children = [...target.children];

            const targetIdx = nodes.findIndex(n => n.id === targetNodeId);
            nodes[targetIdx] = { ...target, children: [newNodeId] };

            for (const childId of newNode.children) {
              const childIdx = nodes.findIndex(n => n.id === childId);
              if (childIdx >= 0) {
                nodes[childIdx] = {
                  ...nodes[childIdx],
                  parents: nodes[childIdx].parents.map(p => p === targetNodeId ? newNodeId : p),
                };
              }
            }
          } else {
            // Normal sibling insertion
            newNode.parents = [...target.parents];
            newNode.children = [...target.children];

            for (const parentId of newNode.parents) {
              const parentIdx = nodes.findIndex(n => n.id === parentId);
              if (parentIdx >= 0) {
                const parent = nodes[parentIdx];
                const targetChildIdx = parent.children.indexOf(targetNodeId);
                if (targetChildIdx >= 0) {
                  const newChildren = [...parent.children];
                  newChildren.splice(targetChildIdx + 1, 0, newNodeId);
                  nodes[parentIdx] = { ...parent, children: newChildren };
                }
              }
            }

            for (const childId of newNode.children) {
              const childIdx = nodes.findIndex(n => n.id === childId);
              if (childIdx >= 0) {
                const child = nodes[childIdx];
                if (!child.parents.includes(newNodeId)) {
                  nodes[childIdx] = {
                    ...child,
                    parents: [...child.parents, newNodeId],
                  };
                }
              }
            }
          }
        }

        nodes.push(newNode);
        return { ...s, nodes };
      });

      // Find new index in linearized order
      const updatedSheet = newSheets.find(s => s.id === sheetId);
      const linearized = updatedSheet ? linearizeNodes(updatedSheet.nodes) : [];
      const newIndex = linearized.findIndex(n => n.id === newNodeId);

      return {
        sheets: newSheets,
        selectedNodeId: newNodeId,
        selectedIndex: newIndex >= 0 ? newIndex : state.selectedIndex,
      };
    });
  },

  clearClipboard: () => set({
    clipboard: { node: null, sourceSheetId: null },
  }),
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd millflow && npm run build`
- [x] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Store has `clipboard` state visible in React DevTools
- [ ] `cutNode` stores node data and removes node from DAG
- [ ] `pasteNode` creates new node with clipboard data at correct position
- [ ] Undo restores cut node

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Update Keyboard Mappings

### Overview
Replace sequence-based shortcuts with single-key shortcuts and add cut/paste keys.

### Changes Required:

#### 1. Import New Store Actions
**File**: `millflow/src/hooks/useKeyboard.ts`
**Lines**: 10-59 (destructuring from store)

Add to the destructured values (around line 52, after `swapNodeDown`):
```typescript
    // Clipboard operations
    cutNode,
    pasteNode,
    clipboard,
```

#### 2. Remove Old Sequence Handlers
**File**: `millflow/src/hooks/useKeyboard.ts`
**Lines**: 245-283

Remove or comment out these sequence handlers:
- `o j` / `o ArrowDown` (lines 245-253)
- `o k` / `o ArrowUp` (lines 255-263)
- `o l` / `o ArrowRight` (lines 265-273)
- `o h` / `o ArrowLeft` (lines 275-283)

#### 3. Update Pending Sequence Check
**File**: `millflow/src/hooks/useKeyboard.ts`
**Line**: 286

Change from:
```typescript
if (['g', 'd', 'm', "'", 'M', 'o'].includes(key) && prevSequence.length === 0) {
```
To:
```typescript
if (['g', 'd', 'm', "'", 'M'].includes(key) && prevSequence.length === 0) {
```

Remove `'o'` from the sequence starters since it's now a single-key action.

#### 4. Add New Single-Key Handlers
**File**: `millflow/src/hooks/useKeyboard.ts`
**Location**: In the switch statement (around lines 295-464), add new cases

Add before the `default` case (around line 462):

```typescript
      case 'o':
        // Insert node after (replaces o j sequence)
        if (!shiftKey && view === 'sheet' && selectedNodeId) {
          event.preventDefault();
          setNodeCreationMode('after');
        }
        break;

      case 'O':
        // Insert node before (replaces o k sequence)
        if (shiftKey && view === 'sheet' && selectedNodeId) {
          event.preventDefault();
          setNodeCreationMode('before');
        }
        break;

      case 'S':
        // Insert sibling to right (replaces o l sequence)
        if (shiftKey && view === 'sheet' && selectedNodeId) {
          event.preventDefault();
          setNodeCreationMode('right');
        }
        break;

      case 'x':
        // Cut node
        if (view === 'sheet' && selectedSheetId && selectedNodeId) {
          event.preventDefault();
          cutNode(selectedSheetId, selectedNodeId);
        }
        break;

      case 'p':
        // Paste after
        if (!shiftKey && view === 'sheet' && selectedSheetId && selectedNodeId && clipboard.node) {
          event.preventDefault();
          pasteNode(selectedSheetId, selectedNodeId, 'after');
        }
        break;

      case 'P':
        // Paste before
        if (shiftKey && view === 'sheet' && selectedSheetId && selectedNodeId && clipboard.node) {
          event.preventDefault();
          pasteNode(selectedSheetId, selectedNodeId, 'before');
        }
        break;

      case 's':
        // Paste right (sibling)
        if (!shiftKey && view === 'sheet' && selectedSheetId && selectedNodeId && clipboard.node) {
          event.preventDefault();
          pasteNode(selectedSheetId, selectedNodeId, 'right');
        }
        break;
```

#### 5. Add Dependencies to useCallback
**File**: `millflow/src/hooks/useKeyboard.ts`
**Lines**: 465-515 (dependency array)

Add to the dependency array:
```typescript
    // Clipboard operations
    cutNode,
    pasteNode,
    clipboard,
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd millflow && npm run build`
- [x] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] `o` key opens node creation form in "after" position
- [ ] `O` (shift+o) key opens node creation form in "before" position
- [ ] `S` (shift+s) key opens node creation form in "right" (sibling) position
- [ ] `x` key cuts the selected node (node disappears, can be pasted)
- [ ] `p` key pastes clipboard node after selected node
- [ ] `P` (shift+p) key pastes clipboard node before selected node
- [ ] `s` key pastes clipboard node as sibling to the right
- [ ] Old sequences `o j`, `o k`, `o l`, `o h` no longer trigger actions
- [ ] `u` key undoes all operations correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Testing Strategy

### Unit Tests:
- Test `cutNode` stores correct node data in clipboard
- Test `cutNode` properly reconnects graph edges
- Test `pasteNode` with all three positions (before/after/right)
- Test paste preserves node properties

### Manual Testing Steps:
1. Select a node in sheet view, press `o` → creation form opens for "after"
2. Select a node, press `O` → creation form opens for "before"
3. Select a node, press `S` → creation form opens for "right" (sibling)
4. Select a node, press `x` → node is removed, verify graph still connected
5. Navigate to another node, press `p` → clipboard node appears after
6. Press `P` → clipboard node appears before (can paste multiple times)
7. Press `s` → clipboard node appears as sibling
8. Press `u` → undo works for all operations

## References

- Research document: `thoughts/shared/research/2025-12-21-node-behavior-keyboard-actions.md`
- Current keyboard hook: `millflow/src/hooks/useKeyboard.ts`
- Store implementation: `millflow/src/store/appStore.ts`
- Node types: `millflow/src/types/index.ts:32-49`
