# Factory Floor Editor - Research Findings

## Issues to Fix

1. **Zoom too aggressive** when dragging first node onto canvas
2. **No node deletion** - cannot remove nodes after adding them
3. **No node editing** - cannot modify descriptions, add owners, or blockers

---

## Issue 1: Aggressive Zoom on First Node

### Root Cause

In `factory-floor/src/App.tsx:282`, the `ReactFlow` component has `fitView` as a prop:

```tsx
<ReactFlow
    ...
    fitView  // <-- This is the problem
    ...
>
```

**Why it causes the issue:**

1. When canvas is empty, `fitView` has nothing to fit to
2. When first node is dropped, React Flow re-renders
3. `fitView` triggers and tries to fill the viewport with just one 256px wide node
4. Result: massive zoom level, node appears huge

**Contrast with Dashboard (`Dashboard.tsx:105-108`):**
```tsx
<ReactFlow
    fitView
    minZoom={0.1}
    maxZoom={2}
>
```
Dashboard has zoom constraints that prevent over-zooming.

### Fix Approach

**Option A: Add zoom constraints (Recommended)**

Add `minZoom`, `maxZoom`, and `fitViewOptions` to limit zoom behavior:

```tsx
// In App.tsx, update ReactFlow props:
<ReactFlow
    nodes={nodes}
    edges={edges}
    // ... other props ...
    fitView
    minZoom={0.5}
    maxZoom={1.5}
    fitViewOptions={{
        maxZoom: 1,  // Don't zoom in past 100% on fitView
        padding: 0.2 // Add 20% padding around content
    }}
>
```

**Option B: Remove fitView, set initial viewport**

Set a fixed initial viewport instead of auto-fitting:

```tsx
<ReactFlow
    // Remove fitView prop
    defaultViewport={{ x: 100, y: 100, zoom: 1 }}
    minZoom={0.5}
    maxZoom={2}
>
```

**Option C: Conditional fitView after layout**

Only call `fitView` programmatically after user clicks "Tidy Up":

```tsx
// Store instance
const [reactFlowInstance, setReactFlowInstance] = useState<ReactFlowInstance | null>(null);

// In onLayout callback, after setting nodes:
const onLayout = useCallback(() => {
    const { nodes: layoutedNodes, edges: layoutedEdges } = getLayoutedElements(nodes, edges);
    setNodes([...layoutedNodes]);
    setEdges([...layoutedEdges]);

    // Fit after layout with constraints
    setTimeout(() => {
        reactFlowInstance?.fitView({ maxZoom: 1, padding: 0.2 });
    }, 50);
}, [nodes, edges, setNodes, setEdges, reactFlowInstance]);
```

### Recommended Fix

Use **Option A** - it's the simplest change with best UX:

```tsx
// App.tsx line 282-287, change to:
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

### Files to Modify

- `factory-floor/src/App.tsx:272-291` - Add zoom constraints to ReactFlow

---

## Issue 2: Node Deletion

### Current State

- No deletion functionality exists
- No keyboard event handlers
- No delete button in UI
- `onNodesChange` from `useNodesState` already handles deletions internally, it just needs to be triggered

### Fix Approach

React Flow's `useNodesState` hook returns an `onNodesChange` handler that already supports deletions. We just need to:

1. Add `deleteKeyCode` prop to enable keyboard deletion
2. Optionally add a delete button for selected nodes

**Step 1: Enable keyboard deletion**

React Flow has built-in support for delete key. Add this prop:

```tsx
<ReactFlow
    // ... existing props ...
    deleteKeyCode={['Backspace', 'Delete']}
>
```

That's it for basic deletion! When nodes are selected and user presses Delete/Backspace, they're removed.

**Step 2 (Optional): Add delete button in toolbar**

For explicit delete button:

```tsx
// In Editor component, add selected node tracking:
const [selectedNodes, setSelectedNodes] = useState<string[]>([]);

// Add onSelectionChange handler:
const onSelectionChange = useCallback(({ nodes }: { nodes: Node[] }) => {
    setSelectedNodes(nodes.map(n => n.id));
}, []);

// Add handler:
const onDeleteSelected = useCallback(() => {
    setNodes((nds) => nds.filter((n) => !selectedNodes.includes(n.id)));
    setEdges((eds) => eds.filter((e) =>
        !selectedNodes.includes(e.source) && !selectedNodes.includes(e.target)
    ));
}, [selectedNodes, setNodes, setEdges]);

// In ReactFlow:
<ReactFlow
    onSelectionChange={onSelectionChange}
>

// In toolbar (around line 255):
{selectedNodes.length > 0 && (
    <button
        onClick={onDeleteSelected}
        className="bg-gruvbox-red border border-gruvbox-red text-gruvbox-bg-hard px-3 py-1.5 rounded text-sm font-bold hover:bg-gruvbox-red-bright transition-colors shadow-lg"
    >
        Delete ({selectedNodes.length})
    </button>
)}
```

### Files to Modify

- `factory-floor/src/App.tsx:272-291` - Add `deleteKeyCode` prop
- `factory-floor/src/App.tsx:255-269` - Optionally add delete button

---

## Issue 3: Node Editing (Descriptions, Owners, Blockers)

### Current State

**Current `OperationNodeData` type (`OperationNode.tsx:7-12`):**
```typescript
export type OperationNodeData = {
  label: string;
  type: 'CNC' | 'EdgeBanding' | 'Assembly' | 'Veneer' | 'Generic';
  notes?: string;
  duration?: number;
};
```

- No `owner` field
- No `blockers` field
- `notes` exists but can't be edited after creation
- No edit UI or modal

### Fix Approach

**Step 1: Extend the data type**

```typescript
// OperationNode.tsx - Update type definition
export type OperationNodeData = {
  label: string;
  type: 'CNC' | 'EdgeBanding' | 'Assembly' | 'Veneer' | 'Generic';
  notes?: string;
  duration?: number;
  // New fields:
  owner?: string;
  blockers?: string[];
  description?: string;  // More detailed than notes
};
```

**Step 2: Create an edit modal component**

Create new file `factory-floor/src/components/nodes/NodeEditModal.tsx`:

```tsx
import { X, Save } from 'lucide-react';
import { useState, useEffect } from 'react';
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
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div className="bg-gruvbox-bg border border-gruvbox-fg-4/20 rounded-lg shadow-xl w-full max-w-md">
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
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Label</label>
            <input
              type="text"
              value={formData.label}
              onChange={(e) => setFormData(prev => ({ ...prev, label: e.target.value }))}
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg"
            />
          </div>

          {/* Duration */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Duration (minutes)</label>
            <input
              type="number"
              value={formData.duration || 15}
              onChange={(e) => setFormData(prev => ({ ...prev, duration: parseInt(e.target.value) || 15 }))}
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg"
            />
          </div>

          {/* Owner */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Owner</label>
            <input
              type="text"
              value={formData.owner || ''}
              onChange={(e) => setFormData(prev => ({ ...prev, owner: e.target.value }))}
              placeholder="Assign an owner..."
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg placeholder:text-gruvbox-fg-4/50"
            />
          </div>

          {/* Description/Notes */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Description</label>
            <textarea
              value={formData.notes || ''}
              onChange={(e) => setFormData(prev => ({ ...prev, notes: e.target.value }))}
              placeholder="Add description or notes..."
              rows={3}
              className="w-full bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg placeholder:text-gruvbox-fg-4/50 resize-none"
            />
          </div>

          {/* Blockers */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Blockers</label>
            <div className="flex gap-2 mb-2">
              <input
                type="text"
                value={newBlocker}
                onChange={(e) => setNewBlocker(e.target.value)}
                onKeyDown={(e) => e.key === 'Enter' && addBlocker()}
                placeholder="Add a blocker..."
                className="flex-1 bg-gruvbox-bg-hard border border-gruvbox-fg-4/20 rounded px-3 py-2 text-gruvbox-fg placeholder:text-gruvbox-fg-4/50"
              />
              <button
                onClick={addBlocker}
                className="px-3 py-2 bg-gruvbox-red/20 text-gruvbox-red rounded hover:bg-gruvbox-red/30"
              >
                Add
              </button>
            </div>
            {formData.blockers && formData.blockers.length > 0 && (
              <div className="space-y-1">
                {formData.blockers.map((blocker, i) => (
                  <div key={i} className="flex items-center gap-2 bg-gruvbox-red/10 border border-gruvbox-red/30 rounded px-2 py-1">
                    <span className="flex-1 text-sm text-gruvbox-red">{blocker}</span>
                    <button onClick={() => removeBlocker(i)} className="text-gruvbox-red/50 hover:text-gruvbox-red">
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
            onClick={onClose}
            className="px-4 py-2 text-gruvbox-fg-4 hover:text-gruvbox-fg"
          >
            Cancel
          </button>
          <button
            onClick={handleSave}
            className="px-4 py-2 bg-gruvbox-aqua text-gruvbox-bg-hard rounded font-bold hover:bg-gruvbox-aqua-bright flex items-center gap-2"
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

**Step 3: Add edit modal state and handlers in App.tsx**

```tsx
// In Editor component, add state:
const [editingNode, setEditingNode] = useState<OperationNodeType | null>(null);

// Add node click handler to open modal:
const onNodeDoubleClick = useCallback((_: React.MouseEvent, node: Node) => {
  if (node.type === 'operation') {
    setEditingNode(node as OperationNodeType);
  }
}, []);

// Add save handler:
const onNodeSave = useCallback((nodeId: string, data: OperationNodeData) => {
  setNodes((nds) => nds.map((n) =>
    n.id === nodeId ? { ...n, data } : n
  ));
}, [setNodes]);

// Add to ReactFlow:
<ReactFlow
    onNodeDoubleClick={onNodeDoubleClick}
>

// Add modal at end of Editor component, before closing </div>:
{editingNode && (
    <NodeEditModal
        nodeId={editingNode.id}
        data={editingNode.data}
        onSave={onNodeSave}
        onClose={() => setEditingNode(null)}
    />
)}
```

**Step 4: Update OperationNode to show new fields**

Update the node component to display owner and blockers:

```tsx
// In OperationNode.tsx body section (around line 74-89):
<div className="p-3">
    <div className="font-mono text-sm font-bold text-gruvbox-fg-1 mb-1 truncate">
        {data.label}
    </div>

    {/* Owner badge */}
    {data.owner && (
        <div className="text-xs text-gruvbox-aqua mb-1">
            Owner: {data.owner}
        </div>
    )}

    {/* Blockers warning */}
    {data.blockers && data.blockers.length > 0 && (
        <div className="text-xs text-gruvbox-red bg-gruvbox-red/10 px-2 py-1 rounded mb-1">
            {data.blockers.length} blocker{data.blockers.length > 1 ? 's' : ''}
        </div>
    )}

    {/* Notes Preview */}
    {data.notes ? (
        <div className="text-xs text-gruvbox-fg-4 line-clamp-2 italic bg-gruvbox-bg-soft/50 p-1.5 rounded">
            "{data.notes}"
        </div>
    ) : (
        <div className="text-xs text-gruvbox-fg-4 italic opacity-50">
            Double-click to edit...
        </div>
    )}
</div>
```

### Files to Create/Modify

1. **Create**: `factory-floor/src/components/nodes/NodeEditModal.tsx` - New edit modal
2. **Modify**: `factory-floor/src/components/nodes/OperationNode.tsx:7-12` - Extend type
3. **Modify**: `factory-floor/src/components/nodes/OperationNode.tsx:74-89` - Show new fields
4. **Modify**: `factory-floor/src/App.tsx` - Add modal state, handlers, and `onNodeDoubleClick`

---

## Summary of Changes

| Issue | File | Change |
|-------|------|--------|
| Zoom | `App.tsx:282` | Add `minZoom`, `maxZoom`, `fitViewOptions` |
| Delete | `App.tsx:282` | Add `deleteKeyCode={['Backspace', 'Delete']}` |
| Delete (optional) | `App.tsx:255` | Add delete button for selected nodes |
| Edit - Type | `OperationNode.tsx:7` | Add `owner`, `blockers`, `description` fields |
| Edit - Modal | New file | Create `NodeEditModal.tsx` |
| Edit - Display | `OperationNode.tsx:74` | Show owner and blockers in node |
| Edit - Handler | `App.tsx` | Add `onNodeDoubleClick` handler and modal state |

## Testing Checklist

After implementation:

- [ ] Drag first node - zoom stays reasonable (not more than 100%)
- [ ] Drag multiple nodes - viewport fits them with padding
- [ ] Select node and press Delete/Backspace - node removed
- [ ] Connected edges removed when node deleted
- [ ] Double-click node - edit modal opens
- [ ] Edit label - updates on save
- [ ] Edit duration - updates on save
- [ ] Add owner - displays on node
- [ ] Add blockers - displays count on node, shown in modal
- [ ] Remove blocker - removed from list
- [ ] Cancel edit - changes discarded
- [ ] Export JSON - includes new fields
