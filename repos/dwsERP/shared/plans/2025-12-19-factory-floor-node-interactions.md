# Factory Floor Node Interactions Implementation Plan

## Overview

Improve the Factory Floor Editor by fixing the aggressive zoom behavior when adding nodes, and adding essential node interaction features: deletion and editing (with support for descriptions, owners, and blockers).

## Current State Analysis

The Factory Floor Editor (`factory-floor/src/App.tsx`) uses React Flow for the visual node-based workflow designer. Currently:

- **Zoom Issue**: `fitView` prop causes massive zoom when first node is dropped (no constraints)
- **No Deletion**: Cannot remove nodes after adding them
- **No Editing**: Cannot modify node properties after creation; nodes have limited fields (`label`, `type`, `notes`, `duration`)

### Key Discoveries:
- `App.tsx:282` has `fitView` without `minZoom`/`maxZoom` constraints
- Dashboard (`Dashboard.tsx:105-108`) already uses zoom constraints (`minZoom={0.1}`, `maxZoom={2}`)
- React Flow's `deleteKeyCode` prop enables built-in keyboard deletion
- `onNodesChange` from `useNodesState` already handles deletion internally
- Node data type at `OperationNode.tsx:7-12` needs extension for owner/blockers

## Desired End State

After implementation:
1. Dropping first node shows it at normal size (max 100% zoom)
2. Users can delete nodes via Delete/Backspace keys and optional toolbar button
3. Users can double-click nodes to edit: label, duration, description, owner, blockers
4. Nodes visually display owner and blocker count when set
5. Export JSON includes all new fields

### Verification:
- `npm run build` passes without type errors
- Manual testing confirms zoom stays reasonable, deletion works, editing modal functions

## What We're NOT Doing

- Undo/redo functionality
- Persistence/saving to backend
- Multi-user collaboration
- Node copy/paste
- Zustand store migration (keeping local React state)

## Implementation Approach

Three phases, each independently testable:
1. Fix zoom constraints (simplest, highest impact)
2. Add node deletion (leverages React Flow built-ins)
3. Add node editing (new component + type extension)

---

## Phase 1: Fix Zoom Behavior

### Overview
Add zoom constraints to ReactFlow to prevent aggressive zoom when nodes are added to empty canvas.

### Changes Required:

#### 1. Update ReactFlow Props
**File**: `factory-floor/src/App.tsx`
**Lines**: 272-291

**Current code (line 282):**
```tsx
fitView
```

**Change to:**
```tsx
fitView
minZoom={0.5}
maxZoom={1.5}
fitViewOptions={{ maxZoom: 1, padding: 0.2 }}
```

**Full ReactFlow component after changes:**
```tsx
<ReactFlow
    nodes={nodes}
    edges={edges}
    onNodesChange={onNodesChange}
    onEdgesChange={onEdgesChange}
    onConnect={onConnect}
    onInit={setReactFlowInstance}
    onDrop={onDrop}
    onDragOver={onDragOver}
    nodeTypes={nodeTypes}
    fitView
    minZoom={0.5}
    maxZoom={1.5}
    fitViewOptions={{ maxZoom: 1, padding: 0.2 }}
    className="bg-gruvbox-bg-hard"
    defaultEdgeOptions={{
        type: 'smoothstep',
        style: { stroke: '#a89984', strokeWidth: 2 }
    }}
>
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd factory-floor && npm run build`
- [x] Lint passes: `cd factory-floor && npm run lint` (pre-existing warnings only)

#### Manual Verification:
- [ ] Drag first node to empty canvas - appears at normal size (not zoomed in huge)
- [ ] Drag multiple nodes - viewport fits them with ~20% padding
- [ ] Manual zoom with scroll wheel respects min/max limits

**Implementation Note**: Complete this phase and verify manually before proceeding.

---

## Phase 2: Add Node Deletion

### Overview
Enable node deletion via keyboard (Delete/Backspace) and optional toolbar button.

### Changes Required:

#### 1. Add deleteKeyCode Prop
**File**: `factory-floor/src/App.tsx`
**Location**: ReactFlow component props (around line 282)

Add this prop:
```tsx
deleteKeyCode={['Backspace', 'Delete']}
```

#### 2. Add Delete Button (Optional but Recommended)
**File**: `factory-floor/src/App.tsx`

**Add import at top (line 3):**
```tsx
import { Layers, MousePointer2, Download, LayoutDashboard, Trash2 } from 'lucide-react';
```

**Add state and handlers inside Editor component (after line 185):**
```tsx
const [selectedNodes, setSelectedNodes] = useState<string[]>([]);

const onSelectionChange = useCallback(({ nodes }: { nodes: any[] }) => {
    setSelectedNodes(nodes.map(n => n.id));
}, []);

const onDeleteSelected = useCallback(() => {
    setNodes((nds) => nds.filter((n) => !selectedNodes.includes(n.id)));
    setEdges((eds) => eds.filter((e) =>
        !selectedNodes.includes(e.source) && !selectedNodes.includes(e.target)
    ));
    setSelectedNodes([]);
}, [selectedNodes, setNodes, setEdges]);
```

**Add onSelectionChange to ReactFlow:**
```tsx
<ReactFlow
    // ... existing props ...
    onSelectionChange={onSelectionChange}
>
```

**Add delete button in toolbar (after line 261, before Export button):**
```tsx
{selectedNodes.length > 0 && (
    <button
        onClick={onDeleteSelected}
        className="bg-gruvbox-red border border-gruvbox-red text-gruvbox-bg-hard px-3 py-1.5 rounded text-sm font-bold hover:bg-gruvbox-red/80 transition-colors shadow-lg flex items-center gap-2"
    >
        <Trash2 className="w-4 h-4" />
        Delete ({selectedNodes.length})
    </button>
)}
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd factory-floor && npm run build`
- [x] Lint passes: `cd factory-floor && npm run lint` (pre-existing warnings only)

#### Manual Verification:
- [ ] Click node to select, press Delete key - node removed
- [ ] Click node to select, press Backspace key - node removed
- [ ] Delete button appears when node selected
- [ ] Delete button shows count of selected nodes
- [ ] Clicking Delete button removes all selected nodes
- [ ] Connected edges removed when source/target node deleted
- [ ] Multi-select (Shift+Click or drag box) and delete works

**Implementation Note**: Complete this phase and verify manually before proceeding.

---

## Phase 3: Add Node Editing

### Overview
Create edit modal for nodes, extend data type to include owner and blockers, and enable double-click to edit.

### Changes Required:

#### 1. Extend OperationNodeData Type
**File**: `factory-floor/src/components/nodes/OperationNode.tsx`
**Lines**: 7-12

**Current:**
```typescript
export type OperationNodeData = {
  label: string;
  type: 'CNC' | 'EdgeBanding' | 'Assembly' | 'Veneer' | 'Generic';
  notes?: string;
  duration?: number;
};
```

**Change to:**
```typescript
export type OperationNodeData = {
  label: string;
  type: 'CNC' | 'EdgeBanding' | 'Assembly' | 'Veneer' | 'Generic';
  notes?: string;
  duration?: number;
  owner?: string;
  blockers?: string[];
};
```

#### 2. Create NodeEditModal Component
**File**: `factory-floor/src/components/nodes/NodeEditModal.tsx` (NEW FILE)

```tsx
import { X, Save } from 'lucide-react';
import { useState } from 'react';
import type { OperationNodeData } from './OperationNode';

type Props = {
  nodeId: string;
  data: OperationNodeData;
  onSave: (nodeId: string, data: OperationNodeData) => void;
  onClose: () => void;
};

export const NodeEditModal = ({ nodeId, data, onSave, onClose }: Props) => {
  const [formData, setFormData] = useState<OperationNodeData>(data);
  const [newBlocker, setNewBlocker] = useState('');

  const handleSave = () => {
    onSave(nodeId, formData);
    onClose();
  };

  const addBlocker = () => {
    if (newBlocker.trim()) {
      setFormData(prev => ({
        ...prev,
        blockers: [...(prev.blockers || []), newBlocker.trim()]
      }));
      setNewBlocker('');
    }
  };

  const removeBlocker = (index: number) => {
    setFormData(prev => ({
      ...prev,
      blockers: prev.blockers?.filter((_, i) => i !== index)
    }));
  };

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50" onClick={onClose}>
      <div
        className="bg-gruvbox-bg border border-gruvbox-fg-4/20 rounded-lg shadow-xl w-full max-w-md"
        onClick={(e) => e.stopPropagation()}
      >
        {/* Header */}
        <div className="flex items-center justify-between p-4 border-b border-gruvbox-fg-4/20">
          <h2 className="text-lg font-bold text-gruvbox-fg">Edit Operation</h2>
          <button onClick={onClose} className="text-gruvbox-fg-4 hover:text-gruvbox-fg">
            <X className="w-5 h-5" />
          </button>
        </div>

        {/* Body */}
        <div className="p-4 space-y-4">
          {/* Label */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Label</label>
            <input
              type="text"
              value={formData.label}
              onChange={(e) => setFormData(prev => ({ ...prev, label: e.target.value }))}
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg focus:outline-none focus:border-gruvbox-aqua"
            />
          </div>

          {/* Duration */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Duration (minutes)</label>
            <input
              type="number"
              min={1}
              value={formData.duration || 15}
              onChange={(e) => setFormData(prev => ({ ...prev, duration: parseInt(e.target.value) || 15 }))}
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg focus:outline-none focus:border-gruvbox-aqua"
            />
          </div>

          {/* Owner */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Owner</label>
            <input
              type="text"
              value={formData.owner || ''}
              onChange={(e) => setFormData(prev => ({ ...prev, owner: e.target.value || undefined }))}
              placeholder="Assign an owner..."
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg placeholder:text-gruvbox-fg-4/50 focus:outline-none focus:border-gruvbox-aqua"
            />
          </div>

          {/* Description/Notes */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Description</label>
            <textarea
              value={formData.notes || ''}
              onChange={(e) => setFormData(prev => ({ ...prev, notes: e.target.value || undefined }))}
              placeholder="Add description or notes..."
              rows={3}
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg placeholder:text-gruvbox-fg-4/50 resize-none focus:outline-none focus:border-gruvbox-aqua"
            />
          </div>

          {/* Blockers */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 uppercase tracking-wider mb-1">Blockers</label>
            <div className="flex gap-2 mb-2">
              <input
                type="text"
                value={newBlocker}
                onChange={(e) => setNewBlocker(e.target.value)}
                onKeyDown={(e) => e.key === 'Enter' && (e.preventDefault(), addBlocker())}
                placeholder="Add a blocker..."
                className="flex-1 bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg placeholder:text-gruvbox-fg-4/50 focus:outline-none focus:border-gruvbox-red"
              />
              <button
                type="button"
                onClick={addBlocker}
                className="px-3 py-2 bg-gruvbox-red/20 text-gruvbox-red rounded hover:bg-gruvbox-red/30 font-bold text-sm"
              >
                Add
              </button>
            </div>
            {formData.blockers && formData.blockers.length > 0 && (
              <div className="space-y-1">
                {formData.blockers.map((blocker, i) => (
                  <div key={i} className="flex items-center gap-2 bg-gruvbox-red/10 border border-gruvbox-red/30 rounded px-2 py-1">
                    <span className="flex-1 text-sm text-gruvbox-red">{blocker}</span>
                    <button
                      type="button"
                      onClick={() => removeBlocker(i)}
                      className="text-gruvbox-red/50 hover:text-gruvbox-red"
                    >
                      <X className="w-4 h-4" />
                    </button>
                  </div>
                ))}
              </div>
            )}
          </div>
        </div>

        {/* Footer */}
        <div className="flex justify-end gap-2 p-4 border-t border-gruvbox-fg-4/20">
          <button
            type="button"
            onClick={onClose}
            className="px-4 py-2 text-gruvbox-fg-4 hover:text-gruvbox-fg"
          >
            Cancel
          </button>
          <button
            type="button"
            onClick={handleSave}
            className="px-4 py-2 bg-gruvbox-aqua text-gruvbox-bg-hard rounded font-bold hover:bg-gruvbox-aqua/80 flex items-center gap-2"
          >
            <Save className="w-4 h-4" />
            Save
          </button>
        </div>
      </div>
    </div>
  );
};
```

#### 3. Update OperationNode Display
**File**: `factory-floor/src/components/nodes/OperationNode.tsx`
**Lines**: 73-89

**Replace the body section (starting at `{/* Body */}`):**
```tsx
{/* Body */}
<div className="p-3">
  <div className="font-mono text-sm font-bold text-gruvbox-fg-1 mb-1 truncate">
    {data.label}
  </div>

  {/* Owner badge */}
  {data.owner && (
    <div className="text-xs text-gruvbox-aqua mb-1 flex items-center gap-1">
      <span className="opacity-60">Owner:</span> {data.owner}
    </div>
  )}

  {/* Blockers warning */}
  {data.blockers && data.blockers.length > 0 && (
    <div className="text-xs text-gruvbox-red bg-gruvbox-red/10 px-2 py-1 rounded mb-1 font-bold">
      {data.blockers.length} blocker{data.blockers.length > 1 ? 's' : ''}
    </div>
  )}

  {/* Notes Preview */}
  {data.notes ? (
    <div className="text-xs text-gruvbox-fg-4 line-clamp-2 italic bg-gruvbox-bg-soft/50 p-1.5 rounded border border-transparent group-hover:border-gruvbox-fg-4/20 transition-colors">
      "{data.notes}"
    </div>
  ) : (
    <div className="text-xs text-gruvbox-fg-4 italic opacity-50">
      Double-click to edit...
    </div>
  )}
</div>
```

#### 4. Add Edit Modal to App.tsx
**File**: `factory-floor/src/App.tsx`

**Add import at top (after line 5):**
```tsx
import { NodeEditModal } from './components/nodes/NodeEditModal';
import type { OperationNodeData } from './components/nodes/OperationNode';
import type { Node } from '@xyflow/react';
```

**Add state in Editor component (after selectedNodes state):**
```tsx
const [editingNode, setEditingNode] = useState<OperationNodeType | null>(null);
```

**Add handlers (after onDeleteSelected):**
```tsx
const onNodeDoubleClick = useCallback((_: React.MouseEvent, node: Node) => {
  if (node.type === 'operation') {
    setEditingNode(node as OperationNodeType);
  }
}, []);

const onNodeSave = useCallback((nodeId: string, data: OperationNodeData) => {
  setNodes((nds) => nds.map((n) =>
    n.id === nodeId ? { ...n, data } : n
  ));
  setEditingNode(null);
}, [setNodes]);
```

**Add onNodeDoubleClick to ReactFlow:**
```tsx
<ReactFlow
    // ... existing props ...
    onNodeDoubleClick={onNodeDoubleClick}
>
```

**Add modal before closing `</main>` tag (around line 293):**
```tsx
{editingNode && (
  <NodeEditModal
    nodeId={editingNode.id}
    data={editingNode.data}
    onSave={onNodeSave}
    onClose={() => setEditingNode(null)}
  />
)}
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd factory-floor && npm run build`
- [ ] Lint passes: `cd factory-floor && npm run lint`

#### Manual Verification:
- [ ] Double-click node - edit modal opens
- [ ] Edit label - updates on node after save
- [ ] Edit duration - updates in header after save
- [ ] Add owner - displays "Owner: [name]" on node
- [ ] Add blockers - displays red "N blockers" badge
- [ ] Click blocker X - removes that blocker
- [ ] Press Enter in blocker field - adds blocker
- [ ] Cancel button - discards all changes
- [ ] Click outside modal - closes without saving
- [ ] Export JSON - includes owner and blockers fields

---

## Testing Strategy

### Unit Tests:
- No existing test infrastructure; skip for now

### Manual Testing Steps:
1. Start dev server: `cd factory-floor && npm run dev`
2. **Zoom Test**: Drag first node, verify normal size
3. **Delete Test**: Add 3 nodes, connect them, select middle node, delete
4. **Edit Test**: Double-click node, modify all fields, save
5. **Export Test**: Add nodes with owners/blockers, export, verify JSON

## Performance Considerations

- `NodeEditModal` uses local state only; no performance impact
- Node selection state is minimal (array of IDs)
- All changes are O(n) where n is node count (acceptable for expected usage)

## References

- Research document: `thoughts/shared/factory-floor-fixes-research.md`
- React Flow docs: https://reactflow.dev/docs/api/react-flow-props/
- Current implementation: `factory-floor/src/App.tsx`
