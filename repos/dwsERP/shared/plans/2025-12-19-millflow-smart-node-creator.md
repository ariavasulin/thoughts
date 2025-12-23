# Smart Node Creator Implementation Plan

## Overview

Redesign the millflow NodeCreator popup to be a sleek, NLP-style autocomplete with automatic type inference and optional quick-add fields for date/assignee. Focused on fast node creation flow.

## Current State Analysis

**Current NodeCreator** (`millflow/src/components/dag/NodeCreator.tsx`):
- Simple text input for name
- Separate type selector grid (toggled with `/`)
- No autocomplete or suggestions
- Creates node with just name + type

**Existing patterns to leverage**:
- CommandPalette (`CommandPalette.tsx`) has fuzzy filter + keyboard nav pattern
- `nodeTypeIcons` and `nodeTypeLabels` in `lib/icons.ts`
- AssigneePicker and DueDatePicker already exist for quick-add fields

## Desired End State

A unified "Smart Node Creator" popup that:
1. Shows operation suggestions as user types (fuzzy matched)
2. Auto-infers node type from selection (CNC → shop, Delivery → delivery, etc.)
3. Supports custom names (defaults to `shop` type)
4. Optional Tab-to-expand for quick-add fields (date, assignee)
5. Keyboard-first, vim-friendly navigation

### Verification:
- Press `o` on a node → popup appears with cursor in input
- Type "cn" → "CNC" highlighted in suggestions
- Press `Enter` → node created as "CNC" with type `shop`
- OR press `Tab` → quick-add panel expands with date/assignee fields
- Press `Enter` → node created with all details

## What We're NOT Doing

- No backend integration (mock data only)
- No persistence of "recently used" suggestions
- No drag-and-drop from suggestions
- Not touching TemplatePicker (multi-node templates stay separate)

## Implementation Approach

Replace NodeCreator with a new SmartNodeCreator component that combines:
- Autocomplete input with operation suggestions
- Type inference mapping
- Expandable quick-add panel (inline, not separate modals)

## Phase 1: Operation Suggestions Data

### Overview
Create the operation templates data structure with type mappings.

### Changes Required:

#### 1. Create Operation Templates
**File**: `millflow/src/data/operations.ts` (new file)

```typescript
import type { NodeType } from '@/types';

export interface OperationSuggestion {
  id: string;
  name: string;
  type: NodeType;
  keywords?: string[]; // For fuzzy matching
}

// Borrowed from factory-floor, mapped to millflow types
export const OPERATION_SUGGESTIONS: OperationSuggestion[] = [
  // Shop operations
  { id: 'cnc', name: 'CNC', type: 'shop', keywords: ['machine', 'cut'] },
  { id: 'edge-banding', name: 'Edge Banding', type: 'shop', keywords: ['edge', 'band'] },
  { id: 'veneer', name: 'Veneer', type: 'shop', keywords: ['veneer', 'laminate'] },
  { id: 'milling', name: 'Milling', type: 'shop', keywords: ['mill', 'router'] },
  { id: 'finishing', name: 'Finishing', type: 'shop', keywords: ['finish', 'sand', 'coat'] },
  { id: 'bench', name: 'Bench', type: 'shop', keywords: ['bench', 'assembly'] },
  { id: 'drafting', name: 'Drafting', type: 'shop', keywords: ['draft', 'draw', 'cad'] },

  // External
  { id: 'outsourced', name: 'Outsourced', type: 'external', keywords: ['vendor', 'external', 'sub'] },

  // Material
  { id: 'order-material', name: 'Order Material', type: 'material', keywords: ['order', 'purchase'] },
  { id: 'receive-material', name: 'Receive Material', type: 'material', keywords: ['receive', 'delivery'] },

  // Approval
  { id: 'client-approval', name: 'Client Approval', type: 'approval', keywords: ['approve', 'sign-off'] },
  { id: 'shop-drawing', name: 'Shop Drawing Approval', type: 'approval', keywords: ['drawing', 'review'] },

  // QC
  { id: 'qc-check', name: 'QC Check', type: 'qc', keywords: ['quality', 'inspect'] },
  { id: 'final-inspection', name: 'Final Inspection', type: 'qc', keywords: ['final', 'inspect'] },

  // Delivery
  { id: 'staging', name: 'Staging', type: 'delivery', keywords: ['stage', 'prep'] },
  { id: 'delivery', name: 'Delivery', type: 'delivery', keywords: ['deliver', 'ship', 'transport'] },

  // Install
  { id: 'field-install', name: 'Field Install', type: 'install', keywords: ['install', 'site'] },
  { id: 'punch-list', name: 'Punch List', type: 'install', keywords: ['punch', 'fix', 'snag'] },
];

// Fuzzy search helper
export function filterOperations(query: string): OperationSuggestion[] {
  if (!query.trim()) return OPERATION_SUGGESTIONS.slice(0, 6);

  const lower = query.toLowerCase();
  return OPERATION_SUGGESTIONS.filter(op =>
    op.name.toLowerCase().includes(lower) ||
    op.keywords?.some(k => k.includes(lower))
  );
}
```

### Success Criteria:

#### Automated Verification:
- [x] File exists and exports correctly: `grep -l "OPERATION_SUGGESTIONS" millflow/src/data/operations.ts`
- [x] TypeScript compiles: `cd millflow && npm run build`

#### Manual Verification:
- [x] N/A for this phase

---

## Phase 2: SmartNodeCreator Component

### Overview
Build the new autocomplete-based node creator with keyboard navigation.

### Changes Required:

#### 1. Create SmartNodeCreator Component
**File**: `millflow/src/components/dag/SmartNodeCreator.tsx` (new file)

```typescript
import { memo, useState, useRef, useEffect, useMemo } from 'react';
import { X, ChevronRight } from 'lucide-react';
import type { NodeType } from '@/types';
import { nodeTypeIcons, nodeTypeLabels } from '@/lib/icons';
import { cn } from '@/lib/styles';
import { OPERATION_SUGGESTIONS, filterOperations, type OperationSuggestion } from '@/data/operations';
import { people } from '@/data/mockData';

interface SmartNodeCreatorProps {
  onSave: (data: {
    name: string;
    type: NodeType;
    assignees?: string[];
    dueDate?: string;
  }) => void;
  onCancel: () => void;
  position: 'before' | 'after';
}

export const SmartNodeCreator = memo(function SmartNodeCreator({
  onSave,
  onCancel,
  position,
}: SmartNodeCreatorProps) {
  const [query, setQuery] = useState('');
  const [selectedIndex, setSelectedIndex] = useState(0);
  const [expanded, setExpanded] = useState(false);
  const [selectedType, setSelectedType] = useState<NodeType>('shop');

  // Quick-add fields
  const [assignees, setAssignees] = useState<string[]>([]);
  const [dueDate, setDueDate] = useState('');

  const inputRef = useRef<HTMLInputElement>(null);
  const listRef = useRef<HTMLDivElement>(null);

  // Filter suggestions based on query
  const suggestions = useMemo(() => filterOperations(query), [query]);

  // Get selected suggestion (if any matches)
  const selectedSuggestion = suggestions[selectedIndex];

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  // Reset selection when suggestions change
  useEffect(() => {
    setSelectedIndex(0);
  }, [suggestions]);

  // Scroll selected into view
  useEffect(() => {
    const el = listRef.current?.children[selectedIndex] as HTMLElement;
    el?.scrollIntoView({ block: 'nearest' });
  }, [selectedIndex]);

  const handleSave = () => {
    const name = query.trim() || selectedSuggestion?.name;
    if (!name) return;

    const type = selectedSuggestion?.type ?? selectedType;

    onSave({
      name,
      type,
      assignees: assignees.length > 0 ? assignees : undefined,
      dueDate: dueDate || undefined,
    });
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
      case 'j':
        if (!expanded && e.key === 'j' && query) break; // Allow typing 'j'
        e.preventDefault();
        setSelectedIndex(i => Math.min(i + 1, suggestions.length - 1));
        break;
      case 'ArrowUp':
      case 'k':
        if (!expanded && e.key === 'k' && query) break;
        e.preventDefault();
        setSelectedIndex(i => Math.max(i - 1, 0));
        break;
      case 'Tab':
        e.preventDefault();
        if (selectedSuggestion) {
          setQuery(selectedSuggestion.name);
          setSelectedType(selectedSuggestion.type);
        }
        setExpanded(true);
        break;
      case 'Enter':
        e.preventDefault();
        handleSave();
        break;
      case 'Escape':
        e.preventDefault();
        if (expanded) {
          setExpanded(false);
        } else {
          onCancel();
        }
        break;
    }
  };

  const handleSuggestionClick = (suggestion: OperationSuggestion) => {
    setQuery(suggestion.name);
    setSelectedType(suggestion.type);
    handleSave();
  };

  const TypeIcon = nodeTypeIcons[selectedSuggestion?.type ?? selectedType];
  const internalPeople = people.filter(p => p.type === 'internal');

  return (
    <div className="bg-gruvbox-bg border border-gruvbox-bg-2 rounded-lg shadow-xl w-96 overflow-hidden">
      {/* Header */}
      <div className="flex items-center justify-between px-3 py-2 border-b border-gruvbox-bg-2">
        <span className="text-xs text-gruvbox-fg-4">
          Add node {position}
        </span>
        <button
          onClick={onCancel}
          className="p-1 text-gruvbox-fg-4 hover:text-gruvbox-fg transition-colors"
        >
          <X size={14} />
        </button>
      </div>

      {/* Input */}
      <div className="flex items-center gap-2 px-3 py-2 border-b border-gruvbox-bg-2">
        <TypeIcon size={16} className="text-gruvbox-fg-4 shrink-0" />
        <input
          ref={inputRef}
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Type operation name..."
          className="flex-1 bg-transparent text-gruvbox-fg placeholder:text-gruvbox-fg-4 outline-none text-sm"
        />
        {selectedSuggestion && (
          <span className="text-xs text-gruvbox-fg-4">
            {nodeTypeLabels[selectedSuggestion.type]}
          </span>
        )}
      </div>

      {/* Suggestions (hidden when expanded) */}
      {!expanded && suggestions.length > 0 && (
        <div ref={listRef} className="max-h-48 overflow-auto py-1">
          {suggestions.map((suggestion, idx) => {
            const Icon = nodeTypeIcons[suggestion.type];
            const isSelected = idx === selectedIndex;

            return (
              <button
                key={suggestion.id}
                onClick={() => handleSuggestionClick(suggestion)}
                onMouseEnter={() => setSelectedIndex(idx)}
                className={cn(
                  'w-full flex items-center gap-3 px-3 py-2 text-left transition-colors',
                  isSelected
                    ? 'bg-gruvbox-bg-1 text-gruvbox-fg-0'
                    : 'text-gruvbox-fg hover:bg-gruvbox-bg-soft'
                )}
              >
                <Icon size={14} className={isSelected ? 'text-gruvbox-blue-bright' : 'text-gruvbox-fg-4'} />
                <span className="flex-1 text-sm">{suggestion.name}</span>
                <span className="text-xs text-gruvbox-fg-4">{nodeTypeLabels[suggestion.type]}</span>
              </button>
            );
          })}
        </div>
      )}

      {/* Quick-add fields (shown when expanded) */}
      {expanded && (
        <div className="px-3 py-3 space-y-3 border-t border-gruvbox-bg-2">
          {/* Assignee */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Assignee</label>
            <select
              value={assignees[0] || ''}
              onChange={(e) => setAssignees(e.target.value ? [e.target.value] : [])}
              className="w-full bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded px-2 py-1.5 text-sm text-gruvbox-fg"
            >
              <option value="">Unassigned</option>
              {internalPeople.map(p => (
                <option key={p.id} value={p.id}>{p.name}</option>
              ))}
            </select>
          </div>

          {/* Due Date */}
          <div>
            <label className="block text-xs text-gruvbox-fg-4 mb-1">Due Date</label>
            <input
              type="date"
              value={dueDate}
              onChange={(e) => setDueDate(e.target.value)}
              className="w-full bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded px-2 py-1.5 text-sm text-gruvbox-fg"
            />
          </div>
        </div>
      )}

      {/* Footer */}
      <div className="flex items-center gap-3 px-3 py-2 border-t border-gruvbox-bg-2 text-xs text-gruvbox-fg-4">
        <span><kbd className="kbd">↑↓</kbd> nav</span>
        <span><kbd className="kbd">Tab</kbd> {expanded ? 'next' : 'details'}</span>
        <span><kbd className="kbd">Enter</kbd> create</span>
        <span><kbd className="kbd">Esc</kbd> {expanded ? 'back' : 'cancel'}</span>
      </div>
    </div>
  );
});

export default SmartNodeCreator;
```

### Success Criteria:

#### Automated Verification:
- [ ] Component compiles: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Component renders correctly when imported
- [ ] Typing filters suggestions
- [ ] Arrow keys navigate suggestions
- [ ] Tab expands quick-add fields
- [ ] Enter creates node

**Implementation Note**: After completing this phase, pause for manual testing of the component in isolation before integrating.

---

## Phase 3: Integration

### Overview
Replace NodeCreator with SmartNodeCreator in SheetView and update the store/handlers.

### Changes Required:

#### 1. Update SheetView imports and usage
**File**: `millflow/src/views/SheetView.tsx`

Replace NodeCreator import and usage:

```typescript
// Change import
import SmartNodeCreator from '@/components/dag/SmartNodeCreator';

// Update handler (around line 85)
const handleNodeCreate = (data: {
  name: string;
  type: NodeType;
  assignees?: string[];
  dueDate?: string;
}) => {
  if (selectedSheetId && selectedNodeId && nodeCreationMode) {
    insertNode(selectedSheetId, selectedNodeId, nodeCreationMode, {
      name: data.name,
      type: data.type,
      assignees: data.assignees,
      due_date: data.dueDate,
    });
  }
};

// Update JSX (around line 170-178)
{nodeCreationMode && selectedNodeId && (
  <div className="fixed inset-0 z-30 flex items-center justify-center bg-black/40">
    <SmartNodeCreator
      position={nodeCreationMode}
      onSave={handleNodeCreate}
      onCancel={handleNodeCreateCancel}
    />
  </div>
)}
```

#### 2. Update insertNode in appStore (if needed)
**File**: `millflow/src/store/appStore.ts`

The `insertNode` function should already accept assignees and due_date in the node data. Verify and update if needed:

```typescript
// In insertNode function, ensure the new node includes:
const newNode: Node = {
  id: crypto.randomUUID(),
  name: nodeData.name,
  type: nodeData.type,
  status: 'not_started',
  assignees: nodeData.assignees ?? [],
  due_date: nodeData.due_date,
  // ... rest of node properties
};
```

### Success Criteria:

#### Automated Verification:
- [ ] App builds: `cd millflow && npm run build`
- [ ] No type errors: `cd millflow && npm run build`
- [ ] Dev server runs: `cd millflow && npm run dev`

#### Manual Verification:
- [ ] Press `o` on a node → SmartNodeCreator appears
- [ ] Type "cnc" → CNC suggestion appears
- [ ] Press Enter → node created with name "CNC", type "shop"
- [ ] Press Tab instead → quick-add fields appear
- [ ] Fill assignee/date → Enter → node created with all fields
- [ ] Press Esc → popup closes without creating

**Implementation Note**: After completing this phase, the feature is complete. Verify the full flow works end-to-end.

---

## Phase 4: Polish & Cleanup

### Overview
Remove old NodeCreator and add any final polish.

### Changes Required:

#### 1. Delete old NodeCreator
**File**: `millflow/src/components/dag/NodeCreator.tsx`

Delete this file (or keep as reference, but remove from exports).

#### 2. Update any remaining imports
Search codebase for any other imports of NodeCreator and update to SmartNodeCreator.

### Success Criteria:

#### Automated Verification:
- [ ] No references to old NodeCreator: `grep -r "NodeCreator" millflow/src --include="*.tsx" --include="*.ts"`
- [ ] App builds clean: `cd millflow && npm run build`

#### Manual Verification:
- [ ] Full node creation flow works
- [ ] No console errors

---

## Testing Strategy

### Manual Testing Steps:
1. Navigate to a sheet view
2. Select a node with `j`/`k`
3. Press `o` to add after
4. Type "ven" → verify "Veneer" suggestion highlighted
5. Press Enter → verify node created as "Veneer" with type "shop"
6. Press `o` again
7. Type "deliver" → verify "Delivery" highlighted
8. Press Tab → verify quick-add expands
9. Select an assignee, pick a date
10. Press Enter → verify node created with all fields
11. Press `O` to add before → verify works same way
12. Type custom name "Paint Booth" → verify creates with default shop type

### Edge Cases:
- Empty input + Enter → should do nothing
- Esc from expanded → should collapse, not close
- Esc from collapsed → should close popup
- Very long operation names → should truncate gracefully

## References

- Research: `thoughts/shared/research/2025-12-19-millflow-node-creator-autofill.md`
- CommandPalette pattern: `millflow/src/components/CommandPalette.tsx`
- Current NodeCreator: `millflow/src/components/dag/NodeCreator.tsx`
- Factory-floor operations: `factory-floor/src/App.tsx:24-40`
