# MillFlow Remaining Features Implementation Plan

## Overview

Implement all remaining MillFlow keyboard-first PM dashboard features: DAG editing, node action forms, navigation enhancements, and polish. This completes the original PRD specification.

## Current State Analysis

### What Exists:
- Full project scaffolding (Vite + React 19 + TypeScript + Tailwind + Zustand)
- Core views: Dashboard, Jobs, Job Detail, Sheet DAG
- Keyboard navigation: j/k, Enter, Esc, g h, g j, g g, G, ?, Cmd+K
- Command palette with fuzzy search
- Node detail panel (read-only display)
- Store actions: `addNode`, `deleteNode`, `updateNode`, `assignNode`, `addBlocker`, `addNote`, `toggleNodeStatus`

### What's Missing:
- DAG editing UI (inline node creation, split/join)
- Node action forms (assign picker, note form, blocker form, due date picker)
- Parallel branch navigation (h/l keys)
- Tab/Shift+Tab for actionable items
- Deliveries view
- Undo/redo stack
- Template system
- Visual key sequence feedback

### Key Files:
- `millflow/src/hooks/useKeyboard.ts` - Keyboard handler with sequence support
- `millflow/src/store/appStore.ts` - Zustand store with all state/actions
- `millflow/src/components/dag/DAGRenderer.tsx` - DAG linearization and rendering
- `millflow/src/components/dag/DAGNode.tsx` - Individual node cards
- `millflow/src/components/dag/ParallelBranch.tsx` - Parallel node rendering
- `millflow/src/components/node/NodeDetailPanel.tsx` - Node detail slide-in panel
- `millflow/src/views/SheetView.tsx` - Sheet DAG view container

## Desired End State

After implementation:
1. Users can create/edit DAG structure entirely via keyboard (o/O insert, s split, J join, d d delete)
2. Users can perform node actions via keyboard (a assign, n note, b blocker, d due date)
3. Users can navigate parallel branches with h/l keys
4. Users can jump between actionable items with Tab/Shift+Tab
5. Users can access deliveries view with g d
6. All edits can be undone/redone with u/Ctrl+r
7. Visual feedback shows pending key sequences (e.g., "g..." in status bar)

### Verification:
- `cd millflow && npm run build` passes
- All keyboard shortcuts work as specified
- Forms save data correctly to store
- Undo/redo works for all edit operations

## What We're NOT Doing

- Backend integration (remains frontend-only with mock data)
- Drag-and-drop DAG editing (keyboard-only)
- Real-time collaboration
- Persistence (state resets on refresh)
- Mobile/touch support

---

## Phase 1: DAG Editing

### Overview
Enable users to create and restructure DAG nodes using keyboard shortcuts: `o` (insert after), `O` (insert before), `s` (split into parallel), `J` (join branches), and `d d` (delete, already implemented).

### Changes Required:

#### 1. NodeCreator Component
**File**: `millflow/src/components/dag/NodeCreator.tsx` (new file)

Inline node creation UI that appears when pressing `o` or `O`.

```tsx
import { memo, useState, useRef, useEffect } from 'react';
import { X } from 'lucide-react';
import type { NodeType } from '@/types';
import { nodeTypeIcons, nodeTypeLabels } from '@/lib/icons';
import { cn } from '@/lib/styles';

interface NodeCreatorProps {
  onSave: (name: string, type: NodeType) => void;
  onCancel: () => void;
  position: 'before' | 'after';
}

const nodeTypes: NodeType[] = ['shop', 'external', 'material', 'approval', 'qc', 'delivery', 'install'];

export const NodeCreator = memo(function NodeCreator({
  onSave,
  onCancel,
  position,
}: NodeCreatorProps) {
  const [name, setName] = useState('');
  const [type, setType] = useState<NodeType>('shop');
  const [showTypeSelector, setShowTypeSelector] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && name.trim()) {
      e.preventDefault();
      onSave(name.trim(), type);
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    } else if (e.key === '/' && !showTypeSelector) {
      e.preventDefault();
      setShowTypeSelector(true);
    }
  };

  const handleTypeSelect = (selectedType: NodeType) => {
    setType(selectedType);
    setShowTypeSelector(false);
    inputRef.current?.focus();
  };

  const TypeIcon = nodeTypeIcons[type];

  return (
    <div className="bg-gruvbox-bg-1 border border-gruvbox-blue-bright rounded-lg p-3 shadow-lg">
      <div className="flex items-center gap-2 mb-2 text-xs text-gruvbox-fg-4">
        <span>Insert {position}</span>
        <span className="text-gruvbox-blue-bright">/ to change type</span>
      </div>

      <div className="flex items-center gap-2">
        <button
          onClick={() => setShowTypeSelector(!showTypeSelector)}
          className={cn(
            'p-2 rounded border transition-colors',
            showTypeSelector
              ? 'border-gruvbox-blue-bright bg-gruvbox-bg-2'
              : 'border-gruvbox-bg-3 hover:border-gruvbox-fg-4'
          )}
        >
          <TypeIcon size={16} className="text-gruvbox-fg" />
        </button>

        <input
          ref={inputRef}
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Node name..."
          className="flex-1 bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg placeholder:text-gruvbox-fg-4 focus:outline-none focus:border-gruvbox-blue-bright"
        />

        <button
          onClick={onCancel}
          className="p-2 text-gruvbox-fg-4 hover:text-gruvbox-fg transition-colors"
        >
          <X size={16} />
        </button>
      </div>

      {showTypeSelector && (
        <div className="mt-2 grid grid-cols-4 gap-1">
          {nodeTypes.map((t) => {
            const Icon = nodeTypeIcons[t];
            return (
              <button
                key={t}
                onClick={() => handleTypeSelect(t)}
                className={cn(
                  'flex items-center gap-2 px-2 py-1.5 rounded text-xs transition-colors',
                  type === t
                    ? 'bg-gruvbox-blue-bright/20 text-gruvbox-blue-bright'
                    : 'hover:bg-gruvbox-bg-2 text-gruvbox-fg-4'
                )}
              >
                <Icon size={12} />
                {nodeTypeLabels[t]}
              </button>
            );
          })}
        </div>
      )}

      <div className="mt-2 flex items-center gap-3 text-xs text-gruvbox-fg-4">
        <span><kbd className="kbd">Enter</kbd> save</span>
        <span><kbd className="kbd">Esc</kbd> cancel</span>
      </div>
    </div>
  );
});

export default NodeCreator;
```

#### 2. Update Store with Split/Join Actions
**File**: `millflow/src/store/appStore.ts`

Add new state and actions for node creation mode and split/join operations.

```typescript
// Add to AppState interface:
interface AppState {
  // ... existing state ...

  // Node creation mode
  nodeCreationMode: 'before' | 'after' | null;

  // Actions - DAG editing
  setNodeCreationMode: (mode: 'before' | 'after' | null) => void;
  insertNode: (sheetId: string, targetNodeId: string, position: 'before' | 'after', node: Partial<Node>) => void;
  splitNode: (sheetId: string, nodeId: string) => void;
  joinNodes: (sheetId: string, nodeIds: string[], targetNodeId: string) => void;
}

// Add to initial state:
nodeCreationMode: null,

// Add actions:
setNodeCreationMode: (mode) => set({ nodeCreationMode: mode }),

insertNode: (sheetId, targetNodeId, position, nodeData) => set(state => {
  const newSheets = state.sheets.map(sheet => {
    if (sheet.id !== sheetId) return sheet;

    const targetNode = sheet.nodes.find(n => n.id === targetNodeId);
    if (!targetNode) return sheet;

    const newNode: Node = {
      id: `n-${Date.now()}`,
      name: nodeData.name || 'New Node',
      type: nodeData.type || 'shop',
      status: 'not_started',
      assignees: [],
      blockers: [],
      notes: [],
      references: [],
      children: [],
      parents: [],
      ...nodeData,
    };

    const nodes = [...sheet.nodes];

    if (position === 'after') {
      // Insert after: new node takes target's children, target points to new node
      newNode.parents = [targetNodeId];
      newNode.children = [...targetNode.children];

      // Update target node
      const targetIdx = nodes.findIndex(n => n.id === targetNodeId);
      nodes[targetIdx] = { ...targetNode, children: [newNode.id] };

      // Update children's parents
      for (const childId of newNode.children) {
        const childIdx = nodes.findIndex(n => n.id === childId);
        if (childIdx >= 0) {
          nodes[childIdx] = {
            ...nodes[childIdx],
            parents: nodes[childIdx].parents.map(p => p === targetNodeId ? newNode.id : p),
          };
        }
      }
    } else {
      // Insert before: new node takes target's parents, target points from new node
      newNode.children = [targetNodeId];
      newNode.parents = [...targetNode.parents];

      // Update target node
      const targetIdx = nodes.findIndex(n => n.id === targetNodeId);
      nodes[targetIdx] = { ...targetNode, parents: [newNode.id] };

      // Update parents' children
      for (const parentId of newNode.parents) {
        const parentIdx = nodes.findIndex(n => n.id === parentId);
        if (parentIdx >= 0) {
          nodes[parentIdx] = {
            ...nodes[parentIdx],
            children: nodes[parentIdx].children.map(c => c === targetNodeId ? newNode.id : c),
          };
        }
      }
    }

    nodes.push(newNode);
    return { ...sheet, nodes };
  });

  return {
    sheets: newSheets,
    nodeCreationMode: null,
    selectedNodeId: `n-${Date.now()}`, // Select the new node
  };
}),

splitNode: (sheetId, nodeId) => set(state => {
  const newSheets = state.sheets.map(sheet => {
    if (sheet.id !== sheetId) return sheet;

    const node = sheet.nodes.find(n => n.id === nodeId);
    if (!node || node.children.length === 0) return sheet;

    // Create a parallel branch node
    const newNode: Node = {
      id: `n-${Date.now()}`,
      name: 'Parallel Task',
      type: 'shop',
      status: 'not_started',
      assignees: [],
      blockers: [],
      notes: [],
      references: [],
      parents: [nodeId],
      children: [...node.children], // Same children as original path
    };

    const nodes = [...sheet.nodes];

    // Add new node as additional child of current node
    const nodeIdx = nodes.findIndex(n => n.id === nodeId);
    nodes[nodeIdx] = {
      ...node,
      children: [...node.children, newNode.id],
    };

    // Update the children to have the new node as additional parent
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
}),

joinNodes: (sheetId, nodeIds, targetNodeId) => set(state => {
  const newSheets = state.sheets.map(sheet => {
    if (sheet.id !== sheetId) return sheet;

    const nodes = sheet.nodes.map(node => {
      if (nodeIds.includes(node.id)) {
        // These nodes now converge to the target
        return {
          ...node,
          children: [targetNodeId],
        };
      }
      if (node.id === targetNodeId) {
        // Target receives all the joining nodes as parents
        return {
          ...node,
          parents: [...new Set([...node.parents, ...nodeIds])],
        };
      }
      return node;
    });

    return { ...sheet, nodes };
  });

  return { sheets: newSheets };
}),
```

#### 3. Update Keyboard Handler
**File**: `millflow/src/hooks/useKeyboard.ts`

Add bindings for `o`, `O`, `s`, `J`.

```typescript
// Add to imports from store:
const {
  // ... existing ...
  nodeCreationMode,
  setNodeCreationMode,
  splitNode,
} = useAppStore();

// Add to single key handlers (after existing switch cases):

case 'o':
  // Insert node after (in sheet view with node selected)
  if (view === 'sheet' && selectedSheetId && selectedNodeId && !nodeCreationMode) {
    event.preventDefault();
    setNodeCreationMode('after');
  }
  break;

case 'O':
  // Insert node before (in sheet view with node selected)
  if (shiftKey && view === 'sheet' && selectedSheetId && selectedNodeId && !nodeCreationMode) {
    event.preventDefault();
    setNodeCreationMode('before');
  }
  break;

case 's':
  // Split - create parallel branch (in sheet view with node selected)
  if (view === 'sheet' && selectedSheetId && selectedNodeId) {
    event.preventDefault();
    splitNode(selectedSheetId, selectedNodeId);
  }
  break;

case 'J':
  // Join - set convergence point (in sheet view)
  // This requires selecting multiple nodes first - for now, just show a toast/hint
  if (shiftKey && view === 'sheet') {
    event.preventDefault();
    // TODO: Implement multi-select mode for joining
    console.log('Join mode - select nodes to join');
  }
  break;
```

#### 4. Update SheetView to Show NodeCreator
**File**: `millflow/src/views/SheetView.tsx`

Integrate the NodeCreator component.

```tsx
// Add imports:
import NodeCreator from '@/components/dag/NodeCreator';

// Inside SheetView component, add:
const { nodeCreationMode, setNodeCreationMode, insertNode } = useAppStore();

const handleNodeCreate = (name: string, type: NodeType) => {
  if (selectedSheetId && selectedNodeId && nodeCreationMode) {
    insertNode(selectedSheetId, selectedNodeId, nodeCreationMode, { name, type });
  }
};

const handleNodeCreateCancel = () => {
  setNodeCreationMode(null);
};

// In the render, after DAGRenderer, add:
{nodeCreationMode && selectedNodeId && (
  <div className="fixed inset-0 z-30 flex items-center justify-center bg-black/40">
    <NodeCreator
      position={nodeCreationMode}
      onSave={handleNodeCreate}
      onCancel={handleNodeCreateCancel}
    />
  </div>
)}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`
- [ ] No TypeScript errors

#### Manual Verification:
- [ ] Press `o` on a node to see NodeCreator appear (insert after)
- [ ] Press `O` (shift+o) on a node to see NodeCreator appear (insert before)
- [ ] Type a name and press Enter to create the node
- [ ] Press Esc to cancel node creation
- [ ] Press `/` in NodeCreator to see type selector
- [ ] Press `s` on a node to create a parallel branch
- [ ] Verify DAG structure updates correctly after insertions
- [ ] Verify `d d` still works to delete nodes

---

## Phase 2: Node Action Forms

### Overview
Enable users to perform node actions via keyboard shortcuts: `a` (assign), `n` (add note), `b` (add blocker), `d` (set due date). These forms appear inline in the NodeDetailPanel or as overlays.

### Changes Required:

#### 1. AssigneePicker Component
**File**: `millflow/src/components/node/AssigneePicker.tsx` (new file)

```tsx
import { memo, useState, useRef, useEffect, useMemo } from 'react';
import { Check, User, Users, Building2 } from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { cn } from '@/lib/styles';
import type { Person } from '@/types';

interface AssigneePickerProps {
  currentAssignees: string[];
  suggestedAssignees?: string[];
  onSave: (assigneeIds: string[]) => void;
  onCancel: () => void;
}

const personTypeIcons = {
  internal: User,
  external: Building2,
  department: Users,
};

export const AssigneePicker = memo(function AssigneePicker({
  currentAssignees,
  suggestedAssignees = [],
  onSave,
  onCancel,
}: AssigneePickerProps) {
  const { people } = useAppStore();
  const [selected, setSelected] = useState<Set<string>>(new Set(currentAssignees));
  const [filter, setFilter] = useState('');
  const [highlightIndex, setHighlightIndex] = useState(0);
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  const filteredPeople = useMemo(() => {
    const filterLower = filter.toLowerCase();

    // Suggested first, then others
    const suggested = people.filter(p => suggestedAssignees.includes(p.id));
    const others = people.filter(p => !suggestedAssignees.includes(p.id));
    const all = [...suggested, ...others];

    if (!filter) return all;
    return all.filter(p =>
      p.name.toLowerCase().includes(filterLower) ||
      p.role.toLowerCase().includes(filterLower) ||
      p.department?.toLowerCase().includes(filterLower)
    );
  }, [people, suggestedAssignees, filter]);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      e.preventDefault();
      onSave(Array.from(selected));
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    } else if (e.key === 'ArrowDown' || e.key === 'j') {
      e.preventDefault();
      setHighlightIndex(i => Math.min(i + 1, filteredPeople.length - 1));
    } else if (e.key === 'ArrowUp' || e.key === 'k') {
      e.preventDefault();
      setHighlightIndex(i => Math.max(i - 1, 0));
    } else if (e.key === ' ' && filteredPeople[highlightIndex]) {
      e.preventDefault();
      togglePerson(filteredPeople[highlightIndex].id);
    }
  };

  const togglePerson = (id: string) => {
    const newSelected = new Set(selected);
    if (newSelected.has(id)) {
      newSelected.delete(id);
    } else {
      newSelected.add(id);
    }
    setSelected(newSelected);
  };

  return (
    <div className="bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded-lg shadow-lg w-80 max-h-96 overflow-hidden">
      <div className="p-3 border-b border-gruvbox-bg-2">
        <input
          ref={inputRef}
          type="text"
          value={filter}
          onChange={(e) => { setFilter(e.target.value); setHighlightIndex(0); }}
          onKeyDown={handleKeyDown}
          placeholder="Search people..."
          className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg placeholder:text-gruvbox-fg-4 focus:outline-none focus:border-gruvbox-blue-bright"
        />
      </div>

      <div className="overflow-auto max-h-64">
        {suggestedAssignees.length > 0 && !filter && (
          <div className="px-3 py-1.5 text-xs text-gruvbox-fg-4 uppercase bg-gruvbox-bg-soft">
            Suggested
          </div>
        )}

        {filteredPeople.map((person, idx) => {
          const Icon = personTypeIcons[person.type];
          const isSuggested = suggestedAssignees.includes(person.id);
          const isSelected = selected.has(person.id);
          const isHighlighted = idx === highlightIndex;

          // Show "Others" divider
          const showOthersDivider = !filter &&
            idx > 0 &&
            isSuggested === false &&
            suggestedAssignees.includes(filteredPeople[idx - 1]?.id);

          return (
            <div key={person.id}>
              {showOthersDivider && (
                <div className="px-3 py-1.5 text-xs text-gruvbox-fg-4 uppercase bg-gruvbox-bg-soft">
                  Others
                </div>
              )}
              <button
                onClick={() => togglePerson(person.id)}
                className={cn(
                  'w-full flex items-center gap-3 px-3 py-2 text-left transition-colors',
                  isHighlighted && 'bg-gruvbox-bg-2',
                  isSelected && 'bg-gruvbox-blue-bright/10'
                )}
              >
                <div className={cn(
                  'w-5 h-5 rounded border flex items-center justify-center',
                  isSelected
                    ? 'bg-gruvbox-blue-bright border-gruvbox-blue-bright'
                    : 'border-gruvbox-bg-3'
                )}>
                  {isSelected && <Check size={12} className="text-gruvbox-bg" />}
                </div>

                <Icon size={14} className="text-gruvbox-fg-4" />

                <div className="flex-1 min-w-0">
                  <div className="text-sm text-gruvbox-fg truncate">{person.name}</div>
                  <div className="text-xs text-gruvbox-fg-4 truncate">
                    {person.role}
                    {person.load !== undefined && ` - ${person.load} jobs`}
                  </div>
                </div>
              </button>
            </div>
          );
        })}
      </div>

      <div className="p-3 border-t border-gruvbox-bg-2 flex items-center justify-between">
        <span className="text-xs text-gruvbox-fg-4">
          {selected.size} selected
        </span>
        <div className="flex items-center gap-2 text-xs">
          <span><kbd className="kbd">Space</kbd> toggle</span>
          <span><kbd className="kbd">Enter</kbd> save</span>
          <span><kbd className="kbd">Esc</kbd> cancel</span>
        </div>
      </div>
    </div>
  );
});

export default AssigneePicker;
```

#### 2. NoteForm Component
**File**: `millflow/src/components/node/NoteForm.tsx` (new file)

```tsx
import { memo, useState, useRef, useEffect } from 'react';

interface NoteFormProps {
  onSave: (text: string) => void;
  onCancel: () => void;
}

export const NoteForm = memo(function NoteForm({
  onSave,
  onCancel,
}: NoteFormProps) {
  const [text, setText] = useState('');
  const textareaRef = useRef<HTMLTextAreaElement>(null);

  useEffect(() => {
    textareaRef.current?.focus();
  }, []);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && (e.metaKey || e.ctrlKey) && text.trim()) {
      e.preventDefault();
      onSave(text.trim());
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    }
  };

  return (
    <div className="bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded-lg shadow-lg w-96 p-3">
      <div className="text-xs text-gruvbox-fg-4 mb-2">Add Note</div>

      <textarea
        ref={textareaRef}
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="Enter note..."
        rows={4}
        className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg placeholder:text-gruvbox-fg-4 focus:outline-none focus:border-gruvbox-blue-bright resize-none"
      />

      <div className="mt-2 flex items-center justify-between text-xs text-gruvbox-fg-4">
        <span>Auto-timestamped</span>
        <div className="flex items-center gap-3">
          <span><kbd className="kbd">Cmd+Enter</kbd> save</span>
          <span><kbd className="kbd">Esc</kbd> cancel</span>
        </div>
      </div>
    </div>
  );
});

export default NoteForm;
```

#### 3. BlockerForm Component
**File**: `millflow/src/components/node/BlockerForm.tsx` (new file)

```tsx
import { memo, useState, useRef, useEffect } from 'react';
import type { BlockerType } from '@/types';
import { cn } from '@/lib/styles';

interface BlockerFormProps {
  onSave: (blocker: { type: BlockerType; description: string; owner: string; expected_resolution?: string }) => void;
  onCancel: () => void;
}

const blockerTypes: { value: BlockerType; label: string }[] = [
  { value: 'internal', label: 'Internal' },
  { value: 'external', label: 'External' },
  { value: 'material', label: 'Material' },
  { value: 'approval', label: 'Approval' },
  { value: 'site', label: 'Site' },
];

export const BlockerForm = memo(function BlockerForm({
  onSave,
  onCancel,
}: BlockerFormProps) {
  const [type, setType] = useState<BlockerType>('internal');
  const [description, setDescription] = useState('');
  const [owner, setOwner] = useState('');
  const [expectedResolution, setExpectedResolution] = useState('');
  const descRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    descRef.current?.focus();
  }, []);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && (e.metaKey || e.ctrlKey) && description.trim()) {
      e.preventDefault();
      onSave({
        type,
        description: description.trim(),
        owner: owner.trim() || 'Unknown',
        expected_resolution: expectedResolution || undefined,
      });
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    }
  };

  return (
    <div className="bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded-lg shadow-lg w-96 p-3">
      <div className="text-xs text-gruvbox-fg-4 mb-3">Add Blocker</div>

      {/* Type selector */}
      <div className="flex gap-1 mb-3">
        {blockerTypes.map((bt) => (
          <button
            key={bt.value}
            onClick={() => setType(bt.value)}
            className={cn(
              'px-2 py-1 text-xs rounded transition-colors',
              type === bt.value
                ? 'bg-gruvbox-red-bright/20 text-gruvbox-red-bright'
                : 'bg-gruvbox-bg-2 text-gruvbox-fg-4 hover:text-gruvbox-fg'
            )}
          >
            {bt.label}
          </button>
        ))}
      </div>

      {/* Description */}
      <input
        ref={descRef}
        type="text"
        value={description}
        onChange={(e) => setDescription(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="What's blocking this?"
        className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg placeholder:text-gruvbox-fg-4 focus:outline-none focus:border-gruvbox-red-bright mb-2"
      />

      {/* Owner */}
      <input
        type="text"
        value={owner}
        onChange={(e) => setOwner(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="Who owns resolving this?"
        className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg placeholder:text-gruvbox-fg-4 focus:outline-none focus:border-gruvbox-red-bright mb-2"
      />

      {/* Expected resolution */}
      <input
        type="date"
        value={expectedResolution}
        onChange={(e) => setExpectedResolution(e.target.value)}
        onKeyDown={handleKeyDown}
        className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg focus:outline-none focus:border-gruvbox-red-bright"
      />

      <div className="mt-3 flex items-center justify-end gap-3 text-xs text-gruvbox-fg-4">
        <span><kbd className="kbd">Cmd+Enter</kbd> save</span>
        <span><kbd className="kbd">Esc</kbd> cancel</span>
      </div>
    </div>
  );
});

export default BlockerForm;
```

#### 4. DueDatePicker Component
**File**: `millflow/src/components/node/DueDatePicker.tsx` (new file)

```tsx
import { memo, useState, useRef, useEffect } from 'react';
import { formatDate } from '@/lib/styles';

interface DueDatePickerProps {
  currentDate?: string;
  onSave: (date: string | undefined) => void;
  onCancel: () => void;
}

export const DueDatePicker = memo(function DueDatePicker({
  currentDate,
  onSave,
  onCancel,
}: DueDatePickerProps) {
  const [date, setDate] = useState(currentDate || '');
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      e.preventDefault();
      onSave(date || undefined);
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    }
  };

  // Quick date options
  const quickDates = [
    { label: 'Tomorrow', days: 1 },
    { label: 'In 3 days', days: 3 },
    { label: 'In 1 week', days: 7 },
    { label: 'In 2 weeks', days: 14 },
  ];

  const setQuickDate = (days: number) => {
    const d = new Date();
    d.setDate(d.getDate() + days);
    setDate(d.toISOString().split('T')[0]);
  };

  return (
    <div className="bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded-lg shadow-lg w-72 p-3">
      <div className="text-xs text-gruvbox-fg-4 mb-2">Set Due Date</div>

      <input
        ref={inputRef}
        type="date"
        value={date}
        onChange={(e) => setDate(e.target.value)}
        onKeyDown={handleKeyDown}
        className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm text-gruvbox-fg focus:outline-none focus:border-gruvbox-yellow-bright mb-3"
      />

      <div className="flex flex-wrap gap-1 mb-3">
        {quickDates.map((qd) => (
          <button
            key={qd.days}
            onClick={() => setQuickDate(qd.days)}
            className="px-2 py-1 text-xs bg-gruvbox-bg-2 text-gruvbox-fg-4 rounded hover:text-gruvbox-fg hover:bg-gruvbox-bg-3 transition-colors"
          >
            {qd.label}
          </button>
        ))}
        {currentDate && (
          <button
            onClick={() => setDate('')}
            className="px-2 py-1 text-xs bg-gruvbox-red-bright/20 text-gruvbox-red-bright rounded hover:bg-gruvbox-red-bright/30 transition-colors"
          >
            Clear
          </button>
        )}
      </div>

      {date && (
        <div className="text-sm text-gruvbox-fg mb-2">
          Selected: {formatDate(date)}
        </div>
      )}

      <div className="flex items-center justify-end gap-3 text-xs text-gruvbox-fg-4">
        <span><kbd className="kbd">Enter</kbd> save</span>
        <span><kbd className="kbd">Esc</kbd> cancel</span>
      </div>
    </div>
  );
});

export default DueDatePicker;
```

#### 5. Update Store with Action Form State
**File**: `millflow/src/store/appStore.ts`

```typescript
// Add to AppState interface:
activeActionForm: 'assign' | 'note' | 'blocker' | 'duedate' | null;

// Add action:
setActiveActionForm: (form: AppState['activeActionForm']) => void;

// Add to initial state:
activeActionForm: null,

// Add action implementation:
setActiveActionForm: (form) => set({ activeActionForm: form }),
```

#### 6. Update Keyboard Handler for Action Keys
**File**: `millflow/src/hooks/useKeyboard.ts`

```typescript
// Add to imports:
const { activeActionForm, setActiveActionForm } = useAppStore();

// If an action form is open, only handle Escape
if (activeActionForm) {
  if (key === 'Escape') {
    event.preventDefault();
    setActiveActionForm(null);
  }
  return;
}

// Add to single key handlers:
case 'a':
  // Assign (in sheet view with node selected)
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    setActiveActionForm('assign');
  }
  break;

case 'n':
  // Add note (in sheet view with node selected)
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    setActiveActionForm('note');
  }
  break;

case 'b':
  // Add blocker (in sheet view with node selected)
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    setActiveActionForm('blocker');
  }
  break;

case 'd':
  // Due date - but only if not starting a 'd d' sequence
  // We need special handling here since 'd' could be start of 'd d' delete
  // For now, require 'd' followed by Enter or just use the form
  break;
```

#### 7. Update SheetView/NodeDetailPanel with Action Forms
**File**: `millflow/src/views/SheetView.tsx` or integrate into `NodeDetailPanel.tsx`

Add action form overlays that appear when the corresponding key is pressed.

```tsx
// In SheetView.tsx, add:
import AssigneePicker from '@/components/node/AssigneePicker';
import NoteForm from '@/components/node/NoteForm';
import BlockerForm from '@/components/node/BlockerForm';
import DueDatePicker from '@/components/node/DueDatePicker';

const { activeActionForm, setActiveActionForm, assignNode, addNote, addBlocker, updateNode } = useAppStore();

const currentNode = sheet?.nodes.find(n => n.id === selectedNodeId);

const handleAssignSave = (assigneeIds: string[]) => {
  if (selectedSheetId && selectedNodeId) {
    assignNode(selectedSheetId, selectedNodeId, assigneeIds);
    setActiveActionForm(null);
  }
};

const handleNoteSave = (text: string) => {
  if (selectedSheetId && selectedNodeId) {
    addNote(selectedSheetId, selectedNodeId, { text });
    setActiveActionForm(null);
  }
};

const handleBlockerSave = (blocker: Parameters<typeof addBlocker>[2]) => {
  if (selectedSheetId && selectedNodeId) {
    addBlocker(selectedSheetId, selectedNodeId, blocker);
    setActiveActionForm(null);
  }
};

const handleDueDateSave = (date: string | undefined) => {
  if (selectedSheetId && selectedNodeId) {
    updateNode(selectedSheetId, selectedNodeId, { due_date: date });
    setActiveActionForm(null);
  }
};

// In render, add overlays:
{activeActionForm === 'assign' && currentNode && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <AssigneePicker
      currentAssignees={currentNode.assignees}
      suggestedAssignees={currentNode.suggested_assignees}
      onSave={handleAssignSave}
      onCancel={() => setActiveActionForm(null)}
    />
  </div>
)}

{activeActionForm === 'note' && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <NoteForm
      onSave={handleNoteSave}
      onCancel={() => setActiveActionForm(null)}
    />
  </div>
)}

{activeActionForm === 'blocker' && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <BlockerForm
      onSave={handleBlockerSave}
      onCancel={() => setActiveActionForm(null)}
    />
  </div>
)}

{activeActionForm === 'duedate' && currentNode && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <DueDatePicker
      currentDate={currentNode.due_date}
      onSave={handleDueDateSave}
      onCancel={() => setActiveActionForm(null)}
    />
  </div>
)}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`
- [ ] No TypeScript errors

#### Manual Verification:
- [ ] Press `a` on a selected node to open AssigneePicker
- [ ] Filter people by typing, navigate with j/k/arrows
- [ ] Space toggles selection, Enter saves, Esc cancels
- [ ] Press `n` to open NoteForm, Cmd+Enter saves
- [ ] Press `b` to open BlockerForm with type selector
- [ ] Press `d` twice quickly still deletes (d d sequence)
- [ ] Verify data persists in store after form submissions
- [ ] Verify NodeDetailPanel shows updated data

---

## Phase 3: Navigation Enhancements

### Overview
Improve navigation with parallel branch support (h/l keys), actionable item jumping (Tab/Shift+Tab), and a new Deliveries view (g d).

### Changes Required:

#### 1. Add Branch Tracking to DAG State
**File**: `millflow/src/store/appStore.ts`

```typescript
// Add to AppState:
selectedBranchIndex: number; // For parallel branch navigation

// Add to initial state:
selectedBranchIndex: 0,

// Update navigateLeft/Right:
navigateLeft: () => {
  const state = get();
  if (state.view === 'sheet' && state.selectedBranchIndex > 0) {
    set({ selectedBranchIndex: state.selectedBranchIndex - 1 });
  }
},

navigateRight: () => {
  const state = get();
  // Need to know max branches at current level - this requires DAG analysis
  set({ selectedBranchIndex: state.selectedBranchIndex + 1 });
},

// Reset branch index when changing nodes:
// Update setSelectedIndex to reset branch:
setSelectedIndex: (index) => set({ selectedIndex: index, selectedBranchIndex: 0 }),
```

#### 2. Update DAGRenderer for Branch Navigation
**File**: `millflow/src/components/dag/DAGRenderer.tsx`

Pass branch index to ParallelBranch and highlight the selected branch.

```tsx
// Add prop:
interface DAGRendererProps {
  // ... existing props ...
  selectedBranchIndex: number;
  onBranchSelect: (branchIndex: number) => void;
}

// Pass to ParallelBranch:
<ParallelBranch
  nodes={level.nodes}
  selectedNodeId={effectiveSelectedId}
  selectedBranchIndex={selectedBranchIndex}
  onSelectNode={onSelectNode}
  onBranchSelect={onBranchSelect}
/>
```

#### 3. Update ParallelBranch Component
**File**: `millflow/src/components/dag/ParallelBranch.tsx`

```tsx
interface ParallelBranchProps {
  nodes: Node[];
  selectedNodeId: string | null;
  selectedBranchIndex: number;
  onSelectNode: (nodeId: string) => void;
  onBranchSelect: (branchIndex: number) => void;
}

// Highlight the selected branch based on index
// When a parallel level is selected, use selectedBranchIndex to determine which node is active
```

#### 4. Update Keyboard Handler for h/l
**File**: `millflow/src/hooks/useKeyboard.ts`

```typescript
case 'h':
case 'ArrowLeft':
  if (view === 'sheet') {
    event.preventDefault();
    navigateLeft();
  }
  break;

case 'l':
case 'ArrowRight':
  if (view === 'sheet') {
    event.preventDefault();
    navigateRight();
  }
  break;
```

#### 5. Add Tab Navigation for Actionable Items
**File**: `millflow/src/hooks/useKeyboard.ts`

```typescript
// Add helper to find next actionable item
const findNextActionableIndex = (nodes: Node[], currentIndex: number, direction: 1 | -1): number => {
  const actionableStatuses = ['ready', 'blocked', 'not_started'];
  let index = currentIndex + direction;

  while (index >= 0 && index < nodes.length) {
    if (actionableStatuses.includes(nodes[index].status)) {
      return index;
    }
    index += direction;
  }

  return currentIndex; // No actionable item found
};

// In keyboard handler:
case 'Tab':
  if (view === 'sheet' && selectedSheetId) {
    event.preventDefault();
    const sheet = useAppStore.getState().sheets.find(s => s.id === selectedSheetId);
    if (sheet) {
      const nextIndex = shiftKey
        ? findNextActionableIndex(sheet.nodes, selectedIndex, -1)
        : findNextActionableIndex(sheet.nodes, selectedIndex, 1);
      useAppStore.getState().setSelectedIndex(nextIndex);
    }
  }
  break;
```

#### 6. Create DeliveriesView
**File**: `millflow/src/views/DeliveriesView.tsx` (new file)

```tsx
import { memo, useMemo } from 'react';
import { Truck, Calendar, MapPin, AlertTriangle } from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { getJobById, getSheetById } from '@/data/mockData';
import { cn, formatDate, formatDateRange } from '@/lib/styles';

interface DeliveryItem {
  nodeId: string;
  sheetId: string;
  jobId: string;
  jobName: string;
  sheetName: string;
  window?: { earliest: string; latest: string };
  status: string;
  hasBlocker: boolean;
}

export const DeliveriesView = memo(function DeliveriesView() {
  const { sheets, selectedIndex, goBack } = useAppStore();

  const deliveries = useMemo(() => {
    const items: DeliveryItem[] = [];

    for (const sheet of sheets) {
      for (const node of sheet.nodes) {
        if (node.type === 'delivery') {
          const job = getJobById(sheet.job_id);
          items.push({
            nodeId: node.id,
            sheetId: sheet.id,
            jobId: sheet.job_id,
            jobName: job?.name || 'Unknown Job',
            sheetName: sheet.name,
            window: node.window,
            status: node.status,
            hasBlocker: node.blockers.length > 0,
          });
        }
      }
    }

    // Sort by earliest date
    return items.sort((a, b) => {
      const dateA = a.window?.earliest || '9999-99-99';
      const dateB = b.window?.earliest || '9999-99-99';
      return dateA.localeCompare(dateB);
    });
  }, [sheets]);

  return (
    <div className="h-full flex flex-col">
      <div className="px-6 py-4 border-b border-gruvbox-bg-2">
        <h1 className="text-xl font-semibold text-gruvbox-fg-0">Upcoming Deliveries</h1>
        <p className="text-sm text-gruvbox-fg-4 mt-1">{deliveries.length} scheduled</p>
      </div>

      <div className="flex-1 overflow-auto p-6">
        <div className="space-y-2 max-w-2xl mx-auto">
          {deliveries.map((delivery, idx) => (
            <div
              key={`${delivery.sheetId}-${delivery.nodeId}`}
              className={cn(
                'p-4 rounded-lg border transition-colors cursor-pointer',
                idx === selectedIndex
                  ? 'bg-gruvbox-bg-1 border-gruvbox-blue-bright'
                  : 'bg-gruvbox-bg-soft border-gruvbox-bg-2 hover:border-gruvbox-bg-3'
              )}
            >
              <div className="flex items-start gap-3">
                <Truck size={18} className={cn(
                  delivery.hasBlocker ? 'text-gruvbox-red-bright' : 'text-gruvbox-fg-4'
                )} />

                <div className="flex-1">
                  <div className="flex items-center gap-2">
                    <span className="font-medium text-gruvbox-fg">{delivery.jobName}</span>
                    {delivery.hasBlocker && (
                      <AlertTriangle size={14} className="text-gruvbox-red-bright" />
                    )}
                  </div>
                  <div className="text-sm text-gruvbox-fg-4">{delivery.sheetName}</div>

                  {delivery.window && (
                    <div className="flex items-center gap-2 mt-2 text-sm">
                      <Calendar size={14} className="text-gruvbox-fg-4" />
                      <span className="text-gruvbox-fg">
                        {delivery.window.earliest === delivery.window.latest
                          ? formatDate(delivery.window.earliest)
                          : formatDateRange(delivery.window.earliest, delivery.window.latest)
                        }
                      </span>
                    </div>
                  )}
                </div>
              </div>
            </div>
          ))}

          {deliveries.length === 0 && (
            <div className="text-center text-gruvbox-fg-4 py-12">
              No deliveries scheduled
            </div>
          )}
        </div>
      </div>
    </div>
  );
});

export default DeliveriesView;
```

#### 7. Update View Types and Routing
**File**: `millflow/src/types/index.ts`

```typescript
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'deliveries';
```

**File**: `millflow/src/App.tsx`

```tsx
import DeliveriesView from '@/views/DeliveriesView';

// In view routing:
{view === 'deliveries' && <DeliveriesView />}
```

#### 8. Add g d Navigation
**File**: `millflow/src/store/appStore.ts`

```typescript
goToDeliveries: () => {
  set({
    view: 'deliveries',
    selectedIndex: 0,
    showNodeDetail: false,
  });
},
```

**File**: `millflow/src/hooks/useKeyboard.ts`

```typescript
// In sequence handlers:
if (sequence === 'g d') {
  event.preventDefault();
  resetSequence();
  goToDeliveries();
  return;
}

// Update the sequence check to include 'g d':
if (['g', 'd'].includes(key) && prevSequence.length === 0) {
  return;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`
- [ ] No TypeScript errors

#### Manual Verification:
- [ ] In sheet view with parallel nodes, h/l navigates between branches
- [ ] Selected branch is visually highlighted
- [ ] Tab jumps to next actionable item (ready, blocked, not_started)
- [ ] Shift+Tab jumps to previous actionable item
- [ ] `g d` navigates to Deliveries view
- [ ] Deliveries view shows all delivery nodes sorted by date
- [ ] j/k navigation works in Deliveries view
- [ ] Esc returns from Deliveries view

---

## Phase 4: Polish

### Overview
Add undo/redo stack, template system, and visual feedback for pending key sequences.

### Changes Required:

#### 1. Undo/Redo Stack in Store
**File**: `millflow/src/store/appStore.ts`

```typescript
// Add history tracking
interface HistoryEntry {
  sheets: Sheet[];
  description: string;
}

// Add to AppState:
history: HistoryEntry[];
historyIndex: number;
canUndo: boolean;
canRedo: boolean;

// Add actions:
pushHistory: (description: string) => void;
undo: () => void;
redo: () => void;

// Implementation:
history: [],
historyIndex: -1,
canUndo: false,
canRedo: false,

pushHistory: (description) => set(state => {
  const newHistory = state.history.slice(0, state.historyIndex + 1);
  newHistory.push({
    sheets: JSON.parse(JSON.stringify(state.sheets)), // Deep clone
    description,
  });

  // Keep max 50 history entries
  if (newHistory.length > 50) {
    newHistory.shift();
  }

  return {
    history: newHistory,
    historyIndex: newHistory.length - 1,
    canUndo: true,
    canRedo: false,
  };
}),

undo: () => set(state => {
  if (state.historyIndex <= 0) return state;

  const newIndex = state.historyIndex - 1;
  const entry = state.history[newIndex];

  return {
    sheets: JSON.parse(JSON.stringify(entry.sheets)),
    historyIndex: newIndex,
    canUndo: newIndex > 0,
    canRedo: true,
  };
}),

redo: () => set(state => {
  if (state.historyIndex >= state.history.length - 1) return state;

  const newIndex = state.historyIndex + 1;
  const entry = state.history[newIndex];

  return {
    sheets: JSON.parse(JSON.stringify(entry.sheets)),
    historyIndex: newIndex,
    canUndo: true,
    canRedo: newIndex < state.history.length - 1,
  };
}),
```

Update all mutation actions (addNode, deleteNode, updateNode, etc.) to call `pushHistory` before making changes.

#### 2. Add u and Ctrl+r Keyboard Bindings
**File**: `millflow/src/hooks/useKeyboard.ts`

```typescript
case 'u':
  if (!ctrlKey && !metaKey) {
    event.preventDefault();
    useAppStore.getState().undo();
  }
  break;

case 'r':
  if (ctrlKey) {
    event.preventDefault();
    useAppStore.getState().redo();
  }
  break;
```

#### 3. Template System
**File**: `millflow/src/data/templates.ts` (new file)

```typescript
import type { Node, NodeType } from '@/types';

export interface NodeTemplate {
  id: string;
  name: string;
  description: string;
  nodes: Omit<Node, 'id' | 'parents' | 'blockers' | 'notes' | 'references'>[];
}

export const templates: NodeTemplate[] = [
  {
    id: 'standard-shop',
    name: 'Standard Shop Flow',
    description: 'CNC → Assembly → Finishing → QC',
    nodes: [
      { name: 'CNC', type: 'shop', status: 'not_started', assignees: [], children: ['$1'] },
      { name: 'Assembly', type: 'shop', status: 'not_started', assignees: [], children: ['$2'] },
      { name: 'Finishing', type: 'shop', status: 'not_started', assignees: [], children: ['$3'] },
      { name: 'QC', type: 'qc', status: 'not_started', assignees: [], children: [] },
    ],
  },
  {
    id: 'delivery-install',
    name: 'Delivery + Install',
    description: 'Delivery → Install',
    nodes: [
      { name: 'Delivery', type: 'delivery', status: 'not_started', assignees: [], children: ['$1'] },
      { name: 'Install', type: 'install', status: 'not_started', assignees: [], children: [] },
    ],
  },
  {
    id: 'approval-gate',
    name: 'Approval Gate',
    description: 'Single approval node',
    nodes: [
      { name: 'Approval', type: 'approval', status: 'not_started', assignees: [], children: [] },
    ],
  },
];

export function getTemplateById(id: string): NodeTemplate | undefined {
  return templates.find(t => t.id === id);
}
```

#### 4. Template Picker Component
**File**: `millflow/src/components/dag/TemplatePicker.tsx` (new file)

```tsx
import { memo, useState, useRef, useEffect } from 'react';
import { templates, type NodeTemplate } from '@/data/templates';
import { cn } from '@/lib/styles';

interface TemplatePickerProps {
  onSelect: (template: NodeTemplate) => void;
  onCancel: () => void;
}

export const TemplatePicker = memo(function TemplatePicker({
  onSelect,
  onCancel,
}: TemplatePickerProps) {
  const [selectedIndex, setSelectedIndex] = useState(0);
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    containerRef.current?.focus();
  }, []);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      e.preventDefault();
      onSelect(templates[selectedIndex]);
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    } else if (e.key === 'ArrowDown' || e.key === 'j') {
      e.preventDefault();
      setSelectedIndex(i => Math.min(i + 1, templates.length - 1));
    } else if (e.key === 'ArrowUp' || e.key === 'k') {
      e.preventDefault();
      setSelectedIndex(i => Math.max(i - 1, 0));
    }
  };

  return (
    <div
      ref={containerRef}
      tabIndex={0}
      onKeyDown={handleKeyDown}
      className="bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded-lg shadow-lg w-80 overflow-hidden focus:outline-none"
    >
      <div className="px-3 py-2 border-b border-gruvbox-bg-2 text-xs text-gruvbox-fg-4">
        Insert Template
      </div>

      <div className="py-1">
        {templates.map((template, idx) => (
          <button
            key={template.id}
            onClick={() => onSelect(template)}
            className={cn(
              'w-full px-3 py-2 text-left transition-colors',
              idx === selectedIndex ? 'bg-gruvbox-bg-2' : 'hover:bg-gruvbox-bg-soft'
            )}
          >
            <div className="text-sm font-medium text-gruvbox-fg">{template.name}</div>
            <div className="text-xs text-gruvbox-fg-4">{template.description}</div>
          </button>
        ))}
      </div>

      <div className="px-3 py-2 border-t border-gruvbox-bg-2 flex items-center gap-3 text-xs text-gruvbox-fg-4">
        <span><kbd className="kbd">j/k</kbd> navigate</span>
        <span><kbd className="kbd">Enter</kbd> insert</span>
        <span><kbd className="kbd">Esc</kbd> cancel</span>
      </div>
    </div>
  );
});

export default TemplatePicker;
```

#### 5. Pending Key Sequence Indicator
**File**: `millflow/src/store/appStore.ts`

```typescript
// Add to AppState:
pendingKeySequence: string;

// Add action:
setPendingKeySequence: (seq: string) => void;

// Implementation:
pendingKeySequence: '',
setPendingKeySequence: (seq) => set({ pendingKeySequence: seq }),
```

**File**: `millflow/src/hooks/useKeyboard.ts`

Update to track pending sequence:

```typescript
// After adding key to sequence:
const { setPendingKeySequence } = useAppStore.getState();

// When starting a sequence:
if (['g', 'd'].includes(key) && prevSequence.length === 0) {
  setPendingKeySequence(key);
  return;
}

// When completing or resetting sequence:
const resetSequence = useCallback(() => {
  sequenceRef.current = [];
  useAppStore.getState().setPendingKeySequence('');
  // ... rest of reset logic
}, []);
```

**File**: `millflow/src/components/ShortcutBar.tsx`

Add pending sequence indicator:

```tsx
const { pendingKeySequence } = useAppStore();

// In render:
{pendingKeySequence && (
  <span className="flex items-center gap-1 text-gruvbox-yellow-bright">
    <kbd className="kbd">{pendingKeySequence}...</kbd>
    <span>waiting for next key</span>
  </span>
)}
```

#### 6. Update HelpModal with New Shortcuts
**File**: `millflow/src/components/HelpModal.tsx`

Add sections for:
- DAG Editing: o, O, s, J, d d
- Node Actions: a, b, n, d
- Navigation: h, l, Tab, Shift+Tab, g d
- History: u, Ctrl+r

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`
- [ ] No TypeScript errors

#### Manual Verification:
- [ ] Make an edit (add node, add note, etc.)
- [ ] Press `u` to undo - change is reverted
- [ ] Press `Ctrl+r` to redo - change is restored
- [ ] Verify undo/redo works for multiple operations
- [ ] Press `t` (if implemented) to open template picker
- [ ] Select a template and verify nodes are inserted
- [ ] Press `g` and see "g..." indicator in shortcut bar
- [ ] Complete sequence (g h) or let it timeout
- [ ] Verify indicator clears after sequence completes/times out
- [ ] Help modal shows all new shortcuts

---

## Testing Strategy

### Unit Tests (if adding test infrastructure):
- Store actions (insertNode, splitNode, undo, redo)
- DAG linearization with various graph structures
- Keyboard sequence handling

### Integration Tests:
- Full keyboard workflow: navigate to sheet, select node, insert new node, add note, undo
- Parallel branch creation and navigation
- Template insertion

### Manual Testing Steps:
1. Navigate with keyboard only through all views
2. Create a complete sheet from scratch using keyboard
3. Edit existing nodes (assign, add blockers, notes)
4. Test undo/redo across multiple operation types
5. Navigate parallel branches with h/l
6. Jump between actionable items with Tab
7. Access deliveries view and return

## Performance Considerations

- History stack limited to 50 entries to prevent memory bloat
- Deep clone for history uses JSON parse/stringify (simple but not most efficient)
- DAG linearization is memoized and only recalculates on sheet changes
- Form components use React.memo for render optimization

## References

- Original PRD: User provided inline
- Implementation plan: `thoughts/shared/plans/2025-12-19-millflow-pm-dashboard.md`
- Handoff document: `thoughts/shared/handoffs/general/2025-12-19_13-07-16_millflow-pm-dashboard.md`
- Gruvbox theme: `factory-floor/tailwind.config.js`
