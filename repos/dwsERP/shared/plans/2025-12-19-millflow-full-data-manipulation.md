# Millflow Full Local Data Manipulation Implementation Plan

## Overview

Enable complete CRUD operations for all entities in Millflow so local data is fully modifiable, with session persistence via localStorage. This transforms the prototype from a read-mostly demo into a fully interactive application where users can create, edit, and delete nodes, blockers, notes, and sheets.

## Current State Analysis

### What's Working
- Node CRUD: `insertNode`, `deleteNode`, `updateNode`, `assignNode`, `toggleNodeStatus`, `splitNode`
- Blockers: `addBlocker` (creates blocker, sets node status to 'blocked')
- Notes: `addNote` (creates note with auto-timestamp)
- History: Full undo/redo with 50-entry rolling history
- Keyboard: Comprehensive shortcuts for node operations

### What's Missing
1. **Blocker Management**: No resolve, edit, or delete
2. **Note Management**: No edit or delete (notes lack unique IDs)
3. **Multi-Select**: Not implemented (required for join and bulk operations)
4. **Sheet Management**: No create, edit metadata, or archive
5. **Persistence**: Data resets on page refresh

### Key Discoveries
- `Blocker` interface already has `resolved_at?: string` field (`types/index.ts:56`) - just unused
- `Note` interface lacks an `id` field (`types/index.ts:59-64`) - needs addition
- `joinNodes()` store method exists (`appStore.ts:704-733`) but has no UI trigger
- History only tracks `sheets` mutations, which is sufficient for all node/blocker/note operations

## Desired End State

After implementation:
1. Users can resolve blockers (marks `resolved_at`, clears blocked status when all resolved)
2. Resolved blockers appear strikethrough in NodeDetailPanel history
3. Users can delete blockers entirely
4. Users can edit and delete notes
5. Users can multi-select nodes for bulk operations (delete, assign, join)
6. Users can create, rename, and soft-delete sheets
7. All data persists to localStorage and survives page refresh
8. "Reset to Demo Data" action restores initial state

### Verification
- Create a blocker, resolve it, verify node unblocks and blocker shows strikethrough
- Multi-select 3 nodes, delete them all at once
- Add a note, edit it, delete it
- Create a sheet, add nodes, refresh page, verify data persists
- Reset to demo data, verify original mock data restored

## What We're NOT Doing

- Job creation/editing/deletion (per user decision)
- Person/team management
- Dashboard state mutations
- Backend API integration
- Real-time collaboration
- Data export/import

## Implementation Approach

We'll implement in 5 phases, each building on the previous:
1. **Blocker Management** - Fix the "blocked status trap"
2. **Note Management** - Add note IDs, edit, delete
3. **Multi-Select & Join** - Enable bulk operations
4. **Sheet Management** - Create, rename, soft-delete sheets
5. **localStorage Persistence** - Survive page refresh

---

## Phase 1: Blocker Management

### Overview
Enable resolving and deleting blockers. Resolved blockers disappear from active list but show strikethrough in history. When all blockers are resolved, node status auto-clears from 'blocked'.

### Changes Required:

#### 1. Store Methods (`millflow/src/store/appStore.ts`)

**Add to AppState interface (after line 85):**
```typescript
resolveBlocker: (sheetId: string, nodeId: string, blockerId: string) => void;
removeBlocker: (sheetId: string, nodeId: string, blockerId: string) => void;
```

**Add implementations (after `addBlocker` at line 827):**
```typescript
resolveBlocker: (sheetId, nodeId, blockerId) => {
  get().pushHistory('Resolve blocker');
  return set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        nodes: sheet.nodes.map(node => {
          if (node.id !== nodeId) return node;

          const updatedBlockers = node.blockers.map(b =>
            b.id === blockerId
              ? { ...b, resolved_at: new Date().toISOString().split('T')[0] }
              : b
          );

          // Check if all blockers are now resolved
          const hasActiveBlockers = updatedBlockers.some(b => !b.resolved_at);

          return {
            ...node,
            blockers: updatedBlockers,
            // Clear blocked status if no active blockers remain
            status: !hasActiveBlockers && node.status === 'blocked' ? 'ready' : node.status,
          };
        }),
      };
    });
    return { sheets: newSheets };
  });
},

removeBlocker: (sheetId, nodeId, blockerId) => {
  get().pushHistory('Remove blocker');
  return set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        nodes: sheet.nodes.map(node => {
          if (node.id !== nodeId) return node;

          const updatedBlockers = node.blockers.filter(b => b.id !== blockerId);

          // Check if all remaining blockers are resolved
          const hasActiveBlockers = updatedBlockers.some(b => !b.resolved_at);

          return {
            ...node,
            blockers: updatedBlockers,
            status: !hasActiveBlockers && node.status === 'blocked' ? 'ready' : node.status,
          };
        }),
      };
    });
    return { sheets: newSheets };
  });
},
```

#### 2. NodeDetailPanel UI (`millflow/src/components/node/NodeDetailPanel.tsx`)

**Add imports (line 2-13):**
```typescript
import { Check, Trash2 } from 'lucide-react';
```

**Replace Blockers Section (lines 150-171) with:**
```typescript
{/* Blockers Section */}
<section className="border-t border-gruvbox-bg-2 pt-6">
  <h3 className="text-sm font-semibold text-gruvbox-fg-2 uppercase tracking-wider mb-3 flex items-center gap-2">
    <AlertTriangle size={14} className="text-gruvbox-red-bright" />
    Blockers
  </h3>

  {/* Active Blockers */}
  {node.blockers.filter(b => !b.resolved_at).length > 0 && (
    <div className="space-y-3 mb-4">
      {node.blockers.filter(b => !b.resolved_at).map((blocker) => (
        <div key={blocker.id} className="text-sm group flex items-start justify-between">
          <div className="flex-1">
            <div className="text-gruvbox-red-bright">{blocker.description}</div>
            <div className="text-xs text-gruvbox-fg-4 mt-1">
              Owner: {blocker.owner}
              {blocker.expected_resolution && (
                <> - Expected: {formatDate(blocker.expected_resolution)}</>
              )}
            </div>
          </div>
          <div className="flex gap-1 opacity-0 group-hover:opacity-100 transition-opacity">
            <button
              onClick={() => useAppStore.getState().resolveBlocker(sheet.id, node.id, blocker.id)}
              className="p-1 text-gruvbox-green-bright hover:bg-gruvbox-bg-soft rounded"
              title="Resolve blocker"
            >
              <Check size={14} />
            </button>
            <button
              onClick={() => useAppStore.getState().removeBlocker(sheet.id, node.id, blocker.id)}
              className="p-1 text-gruvbox-red hover:bg-gruvbox-bg-soft rounded"
              title="Delete blocker"
            >
              <Trash2 size={14} />
            </button>
          </div>
        </div>
      ))}
    </div>
  )}

  {/* Resolved Blockers (strikethrough) */}
  {node.blockers.filter(b => b.resolved_at).length > 0 && (
    <div className="space-y-2 opacity-60">
      <div className="text-xs text-gruvbox-fg-4 uppercase">Resolved</div>
      {node.blockers.filter(b => b.resolved_at).map((blocker) => (
        <div key={blocker.id} className="text-sm line-through text-gruvbox-fg-4">
          <span>{blocker.description}</span>
          <span className="text-xs ml-2">({formatDate(blocker.resolved_at!)})</span>
        </div>
      ))}
    </div>
  )}

  {node.blockers.length === 0 && (
    <div className="text-sm text-gruvbox-fg-4 italic">(none)</div>
  )}
</section>
```

#### 3. Update Footer Shortcuts (line 245-262)

**Add resolve shortcut hint:**
```typescript
<span className="flex items-center gap-1">
  <kbd className="kbd">r</kbd>
  <span className="text-gruvbox-fg-4">resolve</span>
</span>
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Add a blocker to a node, verify status becomes 'blocked'
- [ ] Click resolve (checkmark), verify blocker moves to "Resolved" section with strikethrough
- [ ] Verify node status changes from 'blocked' to 'ready'
- [ ] Click delete (trash), verify blocker is completely removed
- [ ] Undo (u key), verify blocker reappears
- [ ] Multiple blockers: resolve one, verify node stays blocked until all resolved

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Note Management

### Overview
Add unique IDs to notes, enable inline editing and deletion. Notes use the same hover-reveal button pattern as blockers.

### Changes Required:

#### 1. Update Note Type (`millflow/src/types/index.ts`)

**Modify Note interface (lines 59-64):**
```typescript
export interface Note {
  id: string;  // Add this line
  text: string;
  author: string;
  source?: string;
  created_at: string;
}
```

#### 2. Update addNote Store Method (`millflow/src/store/appStore.ts`)

**Modify addNote implementation (line 832-837):**
```typescript
const newNote: Note = {
  id: `note-${Date.now()}`,  // Add this line
  text: note.text || '',
  author: note.author || 'p-009',
  created_at: new Date().toISOString().split('T')[0],
  ...note,
};
```

#### 3. Add Store Methods (`millflow/src/store/appStore.ts`)

**Add to AppState interface (after `addNote`):**
```typescript
updateNote: (sheetId: string, nodeId: string, noteId: string, text: string) => void;
removeNote: (sheetId: string, nodeId: string, noteId: string) => void;
```

**Add implementations (after `addNote`):**
```typescript
updateNote: (sheetId, nodeId, noteId, text) => {
  get().pushHistory('Update note');
  return set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        nodes: sheet.nodes.map(node => {
          if (node.id !== nodeId) return node;
          return {
            ...node,
            notes: node.notes.map(n =>
              n.id === noteId ? { ...n, text } : n
            ),
          };
        }),
      };
    });
    return { sheets: newSheets };
  });
},

removeNote: (sheetId, nodeId, noteId) => {
  get().pushHistory('Remove note');
  return set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        nodes: sheet.nodes.map(node => {
          if (node.id !== nodeId) return node;
          return {
            ...node,
            notes: node.notes.filter(n => n.id !== noteId),
          };
        }),
      };
    });
    return { sheets: newSheets };
  });
},
```

#### 4. Update Mock Data Notes (`millflow/src/data/mockData.ts`)

Add `id` field to all existing notes. Search for `notes:` arrays and add IDs:
```typescript
notes: [
  {
    id: 'note-mock-1',  // Add unique IDs
    text: '...',
    author: '...',
    // ...
  }
],
```

#### 5. NodeDetailPanel Note Editing (`millflow/src/components/node/NodeDetailPanel.tsx`)

**Add state for editing (after line 26):**
```typescript
const [editingNoteId, setEditingNoteId] = useState<string | null>(null);
const [editNoteText, setEditNoteText] = useState('');
```

**Add import for useState:**
```typescript
import { memo, useState } from 'react';
```

**Add Pencil icon to imports:**
```typescript
import { Pencil } from 'lucide-react';
```

**Replace Notes Section (lines 173-197) with:**
```typescript
{/* Notes Section */}
<section className="border-t border-gruvbox-bg-2 pt-6">
  <h3 className="text-sm font-semibold text-gruvbox-fg-2 uppercase tracking-wider mb-3">
    Notes
  </h3>
  {node.notes.length > 0 ? (
    <div className="space-y-4">
      {node.notes.map((note) => {
        const author = getPersonById(note.author);
        const isEditing = editingNoteId === note.id;

        if (isEditing) {
          return (
            <div key={note.id} className="text-sm">
              <textarea
                value={editNoteText}
                onChange={(e) => setEditNoteText(e.target.value)}
                className="w-full bg-gruvbox-bg-soft text-gruvbox-fg p-2 rounded border border-gruvbox-bg-3 focus:border-gruvbox-blue-bright outline-none resize-none"
                rows={3}
                autoFocus
              />
              <div className="flex gap-2 mt-2">
                <button
                  onClick={() => {
                    useAppStore.getState().updateNote(sheet.id, node.id, note.id, editNoteText);
                    setEditingNoteId(null);
                  }}
                  className="px-2 py-1 text-xs bg-gruvbox-green-bright text-gruvbox-bg rounded"
                >
                  Save
                </button>
                <button
                  onClick={() => setEditingNoteId(null)}
                  className="px-2 py-1 text-xs text-gruvbox-fg-4 hover:text-gruvbox-fg"
                >
                  Cancel
                </button>
              </div>
            </div>
          );
        }

        return (
          <div key={note.id} className="text-sm group">
            <div className="flex items-start justify-between">
              <div className="text-xs text-gruvbox-fg-4 mb-1">
                {formatDate(note.created_at)}
                {note.source && <span className="text-gruvbox-purple-bright"> (from {note.source})</span>}
                {author && <span> - {author.name}</span>}
              </div>
              <div className="flex gap-1 opacity-0 group-hover:opacity-100 transition-opacity">
                <button
                  onClick={() => {
                    setEditingNoteId(note.id);
                    setEditNoteText(note.text);
                  }}
                  className="p-1 text-gruvbox-fg-4 hover:text-gruvbox-fg hover:bg-gruvbox-bg-soft rounded"
                  title="Edit note"
                >
                  <Pencil size={12} />
                </button>
                <button
                  onClick={() => useAppStore.getState().removeNote(sheet.id, node.id, note.id)}
                  className="p-1 text-gruvbox-red hover:bg-gruvbox-bg-soft rounded"
                  title="Delete note"
                >
                  <Trash2 size={12} />
                </button>
              </div>
            </div>
            <div className="text-gruvbox-fg bg-gruvbox-bg-soft p-2 rounded border-l-2 border-gruvbox-bg-3">
              "{note.text}"
            </div>
          </div>
        );
      })}
    </div>
  ) : (
    <div className="text-sm text-gruvbox-fg-4 italic">(none)</div>
  )}
</section>
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Add a new note, verify it appears with edit/delete buttons on hover
- [ ] Click edit (pencil), verify inline textarea appears with note text
- [ ] Modify text and save, verify note updates
- [ ] Click delete (trash), verify note is removed
- [ ] Undo works for both edit and delete operations

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Multi-Select & Bulk Operations

### Overview
Add vim-style visual mode (`v` key) for multi-selecting nodes. Selected nodes can be bulk deleted, bulk assigned, or joined to a convergence point.

### Changes Required:

#### 1. Add Multi-Select State (`millflow/src/store/appStore.ts`)

**Add to AppState interface (after line 17):**
```typescript
selectedNodeIds: Set<string>;
multiSelectMode: boolean;
```

**Add to initial state (after line 100):**
```typescript
selectedNodeIds: new Set(),
multiSelectMode: false,
```

**Add to AppState interface actions:**
```typescript
toggleMultiSelectMode: () => void;
toggleNodeSelection: (nodeId: string) => void;
clearMultiSelect: () => void;
deleteSelectedNodes: (sheetId: string) => void;
assignSelectedNodes: (sheetId: string, personIds: string[]) => void;
```

**Add implementations:**
```typescript
toggleMultiSelectMode: () => set(state => ({
  multiSelectMode: !state.multiSelectMode,
  selectedNodeIds: state.multiSelectMode ? new Set() : new Set(state.selectedNodeId ? [state.selectedNodeId] : []),
})),

toggleNodeSelection: (nodeId) => set(state => {
  const newSet = new Set(state.selectedNodeIds);
  if (newSet.has(nodeId)) {
    newSet.delete(nodeId);
  } else {
    newSet.add(nodeId);
  }
  return { selectedNodeIds: newSet, selectedNodeId: nodeId };
}),

clearMultiSelect: () => set({
  multiSelectMode: false,
  selectedNodeIds: new Set(),
}),

deleteSelectedNodes: (sheetId) => {
  const { selectedNodeIds } = get();
  if (selectedNodeIds.size === 0) return;

  get().pushHistory(`Delete ${selectedNodeIds.size} nodes`);
  return set(state => {
    const nodeIdsToDelete = Array.from(selectedNodeIds);

    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;

      // Filter out deleted nodes and reconnect graph
      const nodes = sheet.nodes.filter(n => !nodeIdsToDelete.includes(n.id)).map(node => {
        // Remove deleted nodes from children/parents and reconnect
        const newChildren = node.children.filter(c => !nodeIdsToDelete.includes(c));
        const newParents = node.parents.filter(p => !nodeIdsToDelete.includes(p));

        // For each deleted child, add their children to this node
        for (const deletedId of nodeIdsToDelete) {
          if (node.children.includes(deletedId)) {
            const deleted = sheet.nodes.find(n => n.id === deletedId);
            if (deleted) {
              newChildren.push(...deleted.children.filter(c => !nodeIdsToDelete.includes(c)));
            }
          }
          if (node.parents.includes(deletedId)) {
            const deleted = sheet.nodes.find(n => n.id === deletedId);
            if (deleted) {
              newParents.push(...deleted.parents.filter(p => !nodeIdsToDelete.includes(p)));
            }
          }
        }

        return { ...node, children: [...new Set(newChildren)], parents: [...new Set(newParents)] };
      });

      return { ...sheet, nodes };
    });

    return {
      sheets: newSheets,
      selectedNodeId: null,
      selectedNodeIds: new Set(),
      multiSelectMode: false,
    };
  });
},

assignSelectedNodes: (sheetId, personIds) => {
  const { selectedNodeIds } = get();
  if (selectedNodeIds.size === 0) return;

  get().pushHistory(`Assign ${selectedNodeIds.size} nodes`);
  return set(state => {
    const newSheets = state.sheets.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        nodes: sheet.nodes.map(node =>
          selectedNodeIds.has(node.id) ? { ...node, assignees: personIds } : node
        ),
      };
    });
    return { sheets: newSheets };
  });
},
```

#### 2. Update Keyboard Hooks (`millflow/src/hooks/useKeyboard.ts`)

**Add new destructured items (around line 30):**
```typescript
const {
  // ... existing ...
  multiSelectMode,
  selectedNodeIds,
  toggleMultiSelectMode,
  toggleNodeSelection,
  clearMultiSelect,
  deleteSelectedNodes,
  joinNodes,
} = useAppStore();
```

**Add `v` key handler (in switch statement):**
```typescript
case 'v':
  // Toggle visual/multi-select mode
  if (view === 'sheet') {
    event.preventDefault();
    toggleMultiSelectMode();
  }
  break;
```

**Update space bar in multi-select mode (modify existing space handler):**
```typescript
case ' ':
  event.preventDefault();
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    if (multiSelectMode) {
      // In multi-select: toggle node selection
      toggleNodeSelection(selectedNodeId);
    } else {
      // Normal: toggle status
      toggleNodeStatus(selectedSheetId, selectedNodeId);
    }
  }
  break;
```

**Update `d d` sequence for multi-delete:**
```typescript
if (sequence === 'd d' && view === 'sheet' && selectedSheetId) {
  event.preventDefault();
  resetSequence();
  if (multiSelectMode && selectedNodeIds.size > 0) {
    useAppStore.getState().deleteSelectedNodes(selectedSheetId);
  } else if (selectedNodeId) {
    useAppStore.getState().deleteNode(selectedSheetId, selectedNodeId);
  }
  return;
}
```

**Implement `J` (Shift+J) for join (replace TODO at lines 271-278):**
```typescript
case 'J':
  // Join selected nodes to current node (convergence)
  if (shiftKey && view === 'sheet' && selectedSheetId && selectedNodeId && multiSelectMode) {
    event.preventDefault();
    const nodeIds = Array.from(selectedNodeIds).filter(id => id !== selectedNodeId);
    if (nodeIds.length > 0) {
      joinNodes(selectedSheetId, nodeIds, selectedNodeId);
      clearMultiSelect();
    }
  }
  break;
```

**Add Escape handler for multi-select:**
```typescript
// Add early in the Escape handling (around line 124)
if (multiSelectMode) {
  clearMultiSelect();
  return;
}
```

#### 3. Visual Feedback in DAGNode (`millflow/src/components/dag/DAGNode.tsx`)

**Add multi-select visual indicator:**

**Get multi-select state:**
```typescript
const { multiSelectMode, selectedNodeIds } = useAppStore();
const isMultiSelected = selectedNodeIds.has(node.id);
```

**Update node styling to show multi-selection:**
```typescript
className={cn(
  "...",
  isMultiSelected && "ring-2 ring-gruvbox-purple-bright",
  multiSelectMode && !isMultiSelected && "opacity-60"
)}
```

#### 4. Multi-Select Status Bar (`millflow/src/views/SheetView.tsx`)

**Add status bar when in multi-select mode:**
```typescript
{multiSelectMode && (
  <div className="fixed bottom-4 left-1/2 -translate-x-1/2 bg-gruvbox-bg-soft border border-gruvbox-purple-bright rounded-lg px-4 py-2 flex items-center gap-4 text-sm z-50">
    <span className="text-gruvbox-purple-bright font-semibold">
      VISUAL MODE
    </span>
    <span className="text-gruvbox-fg">
      {selectedNodeIds.size} selected
    </span>
    <span className="text-gruvbox-fg-4">
      <kbd className="kbd">Space</kbd> toggle
      <kbd className="kbd ml-2">dd</kbd> delete
      <kbd className="kbd ml-2">J</kbd> join
      <kbd className="kbd ml-2">Esc</kbd> exit
    </span>
  </div>
)}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Press `v` to enter multi-select mode, see "VISUAL MODE" indicator
- [ ] Navigate with j/k, press Space to select/deselect nodes
- [ ] Selected nodes show purple ring, unselected nodes are dimmed
- [ ] Press `dd` to delete all selected nodes
- [ ] Select multiple parallel nodes, navigate to convergence point, press `J` to join
- [ ] Press `Esc` to exit multi-select mode
- [ ] Undo works for bulk operations

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Sheet Management

### Overview
Enable creating new sheets, renaming sheets, and soft-deleting (archiving) sheets. Archived sheets are hidden from normal view but preserved for data integrity.

### Changes Required:

#### 1. Update Sheet Type (`millflow/src/types/index.ts`)

**Modify Sheet interface (lines 21-28):**
```typescript
export interface Sheet {
  id: string;
  job_id: string;
  name: string;
  description: string;
  created_at: string;
  archived?: boolean;  // Add this line
  archived_at?: string;  // Add this line
  nodes: Node[];
}
```

#### 2. Add Store Methods (`millflow/src/store/appStore.ts`)

**Add to AppState interface:**
```typescript
createSheet: (jobId: string, name: string, description?: string) => void;
updateSheet: (sheetId: string, updates: { name?: string; description?: string }) => void;
archiveSheet: (sheetId: string) => void;
restoreSheet: (sheetId: string) => void;
```

**Add implementations:**
```typescript
createSheet: (jobId, name, description = '') => {
  get().pushHistory('Create sheet');
  const newSheetId = `sheet-${Date.now()}`;
  const rootNodeId = `n-${Date.now()}`;

  return set(state => {
    const newSheet: Sheet = {
      id: newSheetId,
      job_id: jobId,
      name,
      description,
      created_at: new Date().toISOString().split('T')[0],
      nodes: [{
        id: rootNodeId,
        name: 'Start',
        type: 'shop',
        status: 'not_started',
        assignees: [],
        blockers: [],
        notes: [],
        references: [],
        children: [],
        parents: [],
      }],
    };

    return {
      sheets: [...state.sheets, newSheet],
      selectedSheetId: newSheetId,
      selectedNodeId: rootNodeId,
      view: 'sheet' as const,
    };
  });
},

updateSheet: (sheetId, updates) => {
  get().pushHistory('Update sheet');
  return set(state => ({
    sheets: state.sheets.map(sheet =>
      sheet.id === sheetId ? { ...sheet, ...updates } : sheet
    ),
  }));
},

archiveSheet: (sheetId) => {
  get().pushHistory('Archive sheet');
  return set(state => ({
    sheets: state.sheets.map(sheet =>
      sheet.id === sheetId
        ? { ...sheet, archived: true, archived_at: new Date().toISOString().split('T')[0] }
        : sheet
    ),
    // Navigate away if viewing archived sheet
    ...(state.selectedSheetId === sheetId
      ? { view: 'job' as const, selectedSheetId: null, selectedNodeId: null }
      : {}
    ),
  }));
},

restoreSheet: (sheetId) => {
  get().pushHistory('Restore sheet');
  return set(state => ({
    sheets: state.sheets.map(sheet =>
      sheet.id === sheetId
        ? { ...sheet, archived: false, archived_at: undefined }
        : sheet
    ),
  }));
},
```

#### 3. Update getSheetsForJob Helper (`millflow/src/data/mockData.ts`)

**Modify to filter archived sheets:**
```typescript
export function getSheetsForJob(jobId: string): Sheet[] {
  // Note: This reads from mockData, not store. Update to use store in Phase 5.
  return sheets.filter(s => s.job_id === jobId && !s.archived);
}
```

#### 4. Add Keyboard Shortcut for New Sheet (`millflow/src/hooks/useKeyboard.ts`)

**Add `c` key handler for create:**
```typescript
case 'c':
  // Create new sheet (in job view)
  if (view === 'job' && selectedJobId) {
    event.preventDefault();
    // Open sheet creation modal
    useAppStore.getState().setActiveActionForm('newsheet');
  }
  break;
```

#### 5. Create NewSheetForm Component (`millflow/src/components/node/NewSheetForm.tsx`)

```typescript
import { useState, useEffect, useRef } from 'react';

interface NewSheetFormProps {
  onSave: (name: string, description: string) => void;
  onCancel: () => void;
}

export function NewSheetForm({ onSave, onCancel }: NewSheetFormProps) {
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  const handleSubmit = () => {
    if (name.trim()) {
      onSave(name.trim(), description.trim());
    }
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && (e.metaKey || e.ctrlKey)) {
      e.preventDefault();
      handleSubmit();
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    }
  };

  return (
    <div className="bg-gruvbox-bg border border-gruvbox-bg-2 rounded-lg p-4 w-80" onKeyDown={handleKeyDown}>
      <h3 className="text-sm font-semibold text-gruvbox-fg mb-4">New Sheet</h3>

      <input
        ref={inputRef}
        type="text"
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Sheet name..."
        className="w-full bg-gruvbox-bg-soft text-gruvbox-fg px-3 py-2 rounded border border-gruvbox-bg-3 focus:border-gruvbox-blue-bright outline-none mb-3"
      />

      <textarea
        value={description}
        onChange={(e) => setDescription(e.target.value)}
        placeholder="Description (optional)..."
        className="w-full bg-gruvbox-bg-soft text-gruvbox-fg px-3 py-2 rounded border border-gruvbox-bg-3 focus:border-gruvbox-blue-bright outline-none resize-none mb-4"
        rows={2}
      />

      <div className="flex justify-end gap-2">
        <button
          onClick={onCancel}
          className="px-3 py-1.5 text-sm text-gruvbox-fg-4 hover:text-gruvbox-fg"
        >
          Cancel
        </button>
        <button
          onClick={handleSubmit}
          disabled={!name.trim()}
          className="px-3 py-1.5 text-sm bg-gruvbox-blue-bright text-gruvbox-bg rounded disabled:opacity-50"
        >
          Create
        </button>
      </div>

      <div className="mt-3 text-xs text-gruvbox-fg-4">
        <kbd className="kbd">Cmd+Enter</kbd> to save
      </div>
    </div>
  );
}
```

#### 6. Add Form to JobView (`millflow/src/views/JobView.tsx`)

**Add import and render NewSheetForm when activeActionForm === 'newsheet'**

#### 7. Update activeActionForm Type

**In `appStore.ts`, update the type:**
```typescript
activeActionForm: 'assign' | 'note' | 'blocker' | 'duedate' | 'newsheet' | null;
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] In job view, press `c` to open new sheet form
- [ ] Enter name and description, create sheet
- [ ] Verify navigates to new sheet with single "Start" node
- [ ] Archive sheet, verify it disappears from job view
- [ ] Undo, verify sheet reappears

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 5: localStorage Persistence

### Overview
Persist all sheet data to localStorage using Zustand's persist middleware. Data survives page refresh. Include "Reset to Demo Data" functionality.

### Changes Required:

#### 1. Add Zustand Persist Middleware (`millflow/src/store/appStore.ts`)

**Update imports:**
```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
```

**Wrap store creation with persist:**
```typescript
export const useAppStore = create<AppState>()(
  persist(
    (set, get) => ({
      // ... all existing state and actions ...

      // Add reset action
      resetToDemo: () => {
        // Clear localStorage
        localStorage.removeItem('millflow-storage');
        // Reload page to reset state
        window.location.reload();
      },
    }),
    {
      name: 'millflow-storage',
      storage: createJSONStorage(() => localStorage),
      version: 1,
      partialize: (state) => ({
        // Only persist data, not UI state
        sheets: state.sheets,
        // Persist navigation context for better UX
        view: state.view,
        selectedJobId: state.selectedJobId,
        selectedSheetId: state.selectedSheetId,
      }),
      onRehydrateStorage: () => (state) => {
        // Handle Set serialization for expandedSections
        if (state) {
          state.expandedSections = new Set(['your_move', 'at_risk', 'waiting_on_others']);
          state.selectedNodeIds = new Set();
        }
      },
      migrate: (persistedState: unknown, version: number) => {
        // Handle future migrations here
        if (version === 0) {
          // Migration from version 0 to 1
        }
        return persistedState as AppState;
      },
    }
  )
);
```

#### 2. Add Reset Action to Interface

**Add to AppState interface:**
```typescript
resetToDemo: () => void;
```

#### 3. Add Reset Button to UI

**Option A: Command Palette**
Add a "Reset to Demo Data" command in CommandPalette.tsx

**Option B: Help Modal**
Add a reset button in HelpModal.tsx:
```typescript
<button
  onClick={() => {
    if (confirm('This will reset all data to the original demo state. Continue?')) {
      useAppStore.getState().resetToDemo();
    }
  }}
  className="mt-4 px-3 py-1.5 text-sm text-gruvbox-red-bright border border-gruvbox-red rounded hover:bg-gruvbox-red hover:text-gruvbox-bg transition-colors"
>
  Reset to Demo Data
</button>
```

#### 4. Update Mock Data Imports

**In `appStore.ts`, rename imports to avoid confusion:**
```typescript
import {
  jobs as initialJobs,
  sheets as initialSheets,
  people,
  dashboardState
} from '@/data/mockData';

// Use in initial state
jobs: initialJobs,
sheets: initialSheets,
```

#### 5. Fix getSheetsForJob to Use Store

**Update `mockData.ts` or create a new hook:**
```typescript
// In store or as utility
export function getStoreSheetsForJob(sheets: Sheet[], jobId: string): Sheet[] {
  return sheets.filter(s => s.job_id === jobId && !s.archived);
}
```

**Update all usages of `getSheetsForJob` to use store's sheets instead of mock data.**

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd millflow && npm run build`
- [ ] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Make changes (add node, resolve blocker, add note)
- [ ] Refresh page, verify all changes persist
- [ ] Open DevTools → Application → Local Storage, verify `millflow-storage` key exists
- [ ] Click "Reset to Demo Data", confirm, verify original state restored
- [ ] Create a new sheet, refresh, verify sheet persists
- [ ] Archive a sheet, refresh, verify it stays archived

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding.

---

## Testing Strategy

### Unit Tests
Not implementing unit tests for this prototype phase.

### Integration Tests
Not implementing for this prototype phase.

### Manual Testing Checklist

#### Blocker Flow
1. Create blocker on node → verify blocked status
2. Resolve blocker → verify strikethrough, status clears
3. Add second blocker → verify stays blocked
4. Resolve second → verify unblocked
5. Delete blocker → verify removed entirely
6. Undo all operations

#### Note Flow
1. Add note → verify appears with timestamp
2. Edit note → verify text updates
3. Delete note → verify removed
4. Undo all operations

#### Multi-Select Flow
1. Enter visual mode → verify indicator
2. Select multiple nodes → verify rings
3. Bulk delete → verify all removed, graph reconnects
4. Select parallel nodes → join to convergence
5. Exit visual mode → verify cleanup

#### Sheet Flow
1. Create new sheet → verify navigation
2. Add nodes → verify graph builds
3. Archive sheet → verify hidden
4. Restore sheet → verify visible

#### Persistence Flow
1. Make various changes
2. Refresh page
3. Verify all changes persist
4. Reset to demo
5. Verify original state

---

## Performance Considerations

- **History size**: Capped at 50 entries to prevent memory bloat
- **Deep cloning**: Using `JSON.parse(JSON.stringify())` is sufficient for this data size
- **localStorage**: Single key simplifies persistence, size is well under 5MB limit
- **Set serialization**: Zustand persist doesn't serialize Sets natively; we reset them on rehydrate

---

## Migration Notes

### From Current State
- Existing mock data notes lack IDs → Add IDs to all notes in mockData.ts
- No existing localStorage data → Clean migration

### Future Considerations
- `version` field in persist config enables future migrations
- If data format changes, increment version and add migration logic

---

## References

- Research document: `thoughts/shared/research/2025-12-19-millflow-data-manipulation-state.md`
- Store implementation: `millflow/src/store/appStore.ts`
- Types: `millflow/src/types/index.ts`
- NodeDetailPanel: `millflow/src/components/node/NodeDetailPanel.tsx`
- Keyboard hooks: `millflow/src/hooks/useKeyboard.ts`
