# Draggable Sidebar with Side-by-Side Node View

## Overview

Replace the current fixed overlay NodeDetailPanel with a side-by-side resizable layout in SheetView. The DAG view and node detail panel will share horizontal space with a draggable separator, matching the factory-floor prototype pattern.

## Current State Analysis

- `NodeDetailPanel.tsx:46-52`: Fixed overlay with `fixed inset-0 z-40 bg-black/40`
- `App.tsx:45`: Panel rendered globally, outside Layout
- `SheetView.tsx:171-178`: DAG content in `flex-1 overflow-auto` container
- No resizable panel library installed

### Key Discoveries:
- Factory-floor uses `react-resizable-panels@^4.0.8` (`factory-floor/package.json:20`)
- Factory-floor implementation at `factory-floor/src/App.tsx:459-556`
- Current `showNodeDetail` boolean in store (`appStore.ts:149, 393`)

## Desired End State

When a node is selected and Enter is pressed (or node clicked) in SheetView:
- The view splits horizontally: DAG on left, node details on right
- A draggable separator allows resizing (min 20%, max 50% for sidebar)
- Both views are fully visible simultaneously (no dimming)
- Escape removes the sidebar from DOM
- Next time sidebar opens, it remembers the previous width
- Width persists across browser sessions via localStorage

### Verification:
- DAG view visible and interactive while sidebar is open
- Sidebar can be resized by dragging separator
- Width persists after closing/reopening sidebar
- Width persists after page refresh
- Escape key closes sidebar
- All existing keyboard shortcuts still work

## What We're NOT Doing

- Changing behavior for other views (dashboard, jobs, job, deliveries)
- Adding sidebar to global Layout (keeping SheetView-specific)
- Changing the NodeDetailPanel content/styling
- Adding collapse-to-minimum behavior (remove from DOM only)

## Implementation Approach

1. Install `react-resizable-panels`
2. Add `sidebarWidth` state to Zustand with localStorage persistence
3. Create `NodeDetailContent` component (panel content without overlay wrapper)
4. Modify `SheetView` to use `PanelGroup` when sidebar is visible
5. Remove global `NodeDetailPanel` rendering from `App.tsx`
6. Update keyboard handler for new panel behavior

---

## Phase 1: Add Dependency and State

### Overview
Install the resizable panels library and add sidebar width state with localStorage persistence.

### Changes Required:

#### 1. Install dependency
```bash
cd millflow && npm install react-resizable-panels
```

#### 2. Update appStore.ts - Add sidebar width state
**File**: `millflow/src/store/appStore.ts`

Add to state interface (around line 20):
```typescript
sidebarWidth: number; // percentage (20-50)
```

Add to initial state (around line 149):
```typescript
sidebarWidth: 30, // default 30%
```

Add action (around line 393, near toggleNodeDetail):
```typescript
setSidebarWidth: (width: number) => set({ sidebarWidth: Math.min(50, Math.max(20, width)) }),
```

Add localStorage persistence - modify the store creation to load/save sidebarWidth:
```typescript
// At top of file, add helper
const loadSidebarWidth = (): number => {
  try {
    const saved = localStorage.getItem('millflow-sidebar-width');
    if (saved) {
      const width = parseFloat(saved);
      if (width >= 20 && width <= 50) return width;
    }
  } catch {}
  return 30;
};

// In initial state
sidebarWidth: loadSidebarWidth(),

// In setSidebarWidth action
setSidebarWidth: (width: number) => {
  const clamped = Math.min(50, Math.max(20, width));
  localStorage.setItem('millflow-sidebar-width', String(clamped));
  set({ sidebarWidth: clamped });
},
```

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `cd millflow && npm run build`
- [x] No TypeScript errors: `cd millflow && npx tsc --noEmit`

#### Manual Verification:
- [ ] `react-resizable-panels` appears in package.json dependencies

**Implementation Note**: After completing this phase, pause for manual confirmation.

---

## Phase 2: Extract NodeDetailContent Component

### Overview
Extract the panel content from NodeDetailPanel into a reusable component that can be embedded in the resizable panel.

### Changes Required:

#### 1. Create NodeDetailContent.tsx
**File**: `millflow/src/components/node/NodeDetailContent.tsx`

Extract lines 54-359 from NodeDetailPanel.tsx into a new component:

```typescript
import { memo, useState } from 'react';
import {
  X,
  Clock,
  Calendar,
  AlertTriangle,
  FileText,
  Mail,
  Image,
  Link,
  History,
  UserPlus,
  Check,
  Trash2,
  Pencil,
} from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { nodeTypeIcons, statusIcons, statusLabels, nodeTypeLabels } from '@/lib/icons';
import { cn, statusColors, formatDate } from '@/lib/styles';

const referenceIcons = {
  email: Mail,
  document: FileText,
  image: Image,
  link: Link,
};

interface NodeDetailContentProps {
  onClose: () => void;
}

export const NodeDetailContent = memo(function NodeDetailContent({ onClose }: NodeDetailContentProps) {
  const { selectedSheetId, selectedNodeId, sheets, people } = useAppStore();
  const [editingNoteId, setEditingNoteId] = useState<string | null>(null);
  const [editingText, setEditingText] = useState('');

  const sheet = selectedSheetId ? sheets.find(s => s.id === selectedSheetId) : null;
  const node = sheet?.nodes.find(n => n.id === selectedNodeId);

  if (!node || !sheet) return null;

  const TypeIcon = nodeTypeIcons[node.type];
  const StatusIcon = statusIcons[node.status];
  const statusColor = statusColors[node.status];

  const assigneeNames = node.assignees.map(id => people.find(p => p.id === id)?.name).filter(Boolean) as string[];
  const suggestedPeople = node.suggested_assignees?.map(id => people.find(p => p.id === id)).filter(Boolean) || [];

  return (
    <div className="h-full flex flex-col bg-gruvbox-bg border-l border-gruvbox-bg-2 overflow-hidden">
      {/* Header */}
      <div className="flex items-center justify-between px-6 py-4 border-b border-gruvbox-bg-2">
        <div className="flex items-center gap-3">
          <span className={statusColor}>
            <TypeIcon size={20} />
          </span>
          <h2 className="text-lg font-semibold text-gruvbox-fg-0">{node.name}</h2>
        </div>
        <button
          onClick={onClose}
          className="p-1 text-gruvbox-fg-4 hover:text-gruvbox-fg transition-colors"
        >
          <X size={20} />
        </button>
      </div>

      {/* Scrollable Content */}
      <div className="flex-1 overflow-auto p-6 space-y-6">
        {/* ... rest of content sections (lines 72-338 from NodeDetailPanel) ... */}
      </div>

      {/* Footer Shortcuts */}
      <div className="flex items-center gap-4 px-6 py-3 border-t border-gruvbox-bg-2 text-xs">
        <span className="flex items-center gap-1">
          <kbd className="kbd">a</kbd>
          <span className="text-gruvbox-fg-4">assign</span>
        </span>
        <span className="flex items-center gap-1">
          <kbd className="kbd">b</kbd>
          <span className="text-gruvbox-fg-4">blocker</span>
        </span>
        <span className="flex items-center gap-1">
          <kbd className="kbd">n</kbd>
          <span className="text-gruvbox-fg-4">note</span>
        </span>
        <span className="flex items-center gap-1">
          <kbd className="kbd">esc</kbd>
          <span className="text-gruvbox-fg-4">back</span>
        </span>
      </div>
    </div>
  );
});

export default NodeDetailContent;
```

Note: The full content sections (Status, Assigned, Blockers, Notes, References, History) should be copied from NodeDetailPanel.tsx lines 72-338. The key changes are:
- Remove `fixed inset-0` and backdrop wrapper
- Change root to `h-full flex flex-col` for panel integration
- Remove `sticky top-0` and `sticky bottom-0` (use flex layout instead)
- Accept `onClose` prop instead of using `toggleNodeDetail` directly
- Add `overflow-hidden` to root, `flex-1 overflow-auto` to content

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `cd millflow && npm run build`
- [x] No TypeScript errors

#### Manual Verification:
- [ ] New file exists at `millflow/src/components/node/NodeDetailContent.tsx`

**Implementation Note**: After completing this phase, pause for manual confirmation.

---

## Phase 3: Update SheetView with Resizable Panels

### Overview
Modify SheetView to use PanelGroup when the sidebar is visible, integrating the DAG and NodeDetailContent side-by-side.

### Changes Required:

#### 1. Update SheetView.tsx
**File**: `millflow/src/views/SheetView.tsx`

Add imports:
```typescript
import { Panel, PanelGroup, PanelResizeHandle } from 'react-resizable-panels';
import NodeDetailContent from '@/components/node/NodeDetailContent';
```

Add to destructured store values:
```typescript
showNodeDetail,
sidebarWidth,
setSidebarWidth,
```

Replace the main return JSX (lines 155-258) with:

```typescript
return (
  <div ref={containerRef} className="h-full flex flex-col">
    {/* Sheet Header */}
    <div className="px-6 py-4 border-b border-gruvbox-bg-2">
      <button
        onClick={goBack}
        className="flex items-center gap-2 text-sm text-gruvbox-fg-4 hover:text-gruvbox-fg transition-colors mb-2"
      >
        <ArrowLeft size={14} />
        {job?.name || 'Back'}
      </button>
      <h1 className="text-xl font-semibold text-gruvbox-fg-0">{sheet.name}</h1>
      <p className="text-sm text-gruvbox-fg-4 mt-1">{sheet.description}</p>
    </div>

    {/* Main Content Area with Optional Sidebar */}
    <div className="flex-1 overflow-hidden">
      {showNodeDetail ? (
        <PanelGroup direction="horizontal" onLayout={(sizes) => setSidebarWidth(sizes[1])}>
          <Panel defaultSize={100 - sidebarWidth} minSize={50}>
            <div className="h-full overflow-auto p-6">
              <DAGRenderer
                sheet={sheet}
                selectedNodeId={selectedNodeId}
                selectedIndex={selectedIndex}
                onSelectNode={handleNodeClick}
              />
            </div>
          </Panel>

          <PanelResizeHandle className="w-1 bg-gruvbox-bg-3 hover:bg-gruvbox-aqua-bright transition-colors cursor-col-resize" />

          <Panel defaultSize={sidebarWidth} minSize={20} maxSize={50}>
            <NodeDetailContent onClose={toggleNodeDetail} />
          </Panel>
        </PanelGroup>
      ) : (
        <div className="h-full overflow-auto p-6">
          <DAGRenderer
            sheet={sheet}
            selectedNodeId={selectedNodeId}
            selectedIndex={selectedIndex}
            onSelectNode={handleNodeClick}
          />
        </div>
      )}
    </div>

    {/* Overlays remain the same (node creator, action forms, template picker, multi-select bar) */}
    {/* ... lines 180-257 unchanged ... */}
  </div>
);
```

#### 2. Update handleNodeClick
The existing `handleNodeClick` (line 83-86) remains the same - it already calls `toggleNodeDetail()`.

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `cd millflow && npm run build`
- [x] No TypeScript errors

#### Manual Verification:
- [ ] Clicking a node opens sidebar side-by-side with DAG
- [ ] DAG is fully visible (not dimmed)
- [ ] Resize handle is draggable
- [ ] Sidebar respects min/max constraints

**Implementation Note**: After completing this phase, pause for manual confirmation.

---

## Phase 4: Remove Global NodeDetailPanel

### Overview
Remove the global NodeDetailPanel from App.tsx since it's now handled within SheetView.

### Changes Required:

#### 1. Update App.tsx
**File**: `millflow/src/App.tsx`

Remove the import (line 6):
```typescript
// Remove: import NodeDetailPanel from '@/components/node/NodeDetailPanel';
```

Remove from render (line 45):
```typescript
// Remove: <NodeDetailPanel />
```

The file should look like:
```typescript
import { useAppStore } from '@/store/appStore';
import { useKeyboard } from '@/hooks/useKeyboard';
import Layout from '@/components/Layout';
import HelpModal from '@/components/HelpModal';
import CommandPalette from '@/components/CommandPalette';
import DashboardView from '@/views/DashboardView';
// ... other view imports

function App() {
  // ... existing code

  return (
    <>
      <Layout>
        {renderView()}
      </Layout>

      {/* Modals and overlays */}
      <HelpModal />
      <CommandPalette />
      {/* NodeDetailPanel removed - now in SheetView */}
    </>
  );
}
```

#### 2. Optionally delete NodeDetailPanel.tsx
**File**: `millflow/src/components/node/NodeDetailPanel.tsx`

This file can be deleted since NodeDetailContent.tsx replaces it. However, you may want to keep it temporarily for reference.

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `cd millflow && npm run build`
- [x] No unused import warnings

#### Manual Verification:
- [ ] App still loads without errors
- [ ] Node detail still works in SheetView

**Implementation Note**: After completing this phase, pause for manual confirmation.

---

## Phase 5: Verify Keyboard Shortcuts and Polish

### Overview
Ensure all existing keyboard shortcuts work correctly with the new panel layout.

### Changes Required:

#### 1. Verify useKeyboard.ts behavior
**File**: `millflow/src/hooks/useKeyboard.ts`

The existing keyboard handler should work unchanged because:
- `toggleNodeDetail()` still toggles the `showNodeDetail` state
- Escape key handler at lines 340-341 calls `toggleNodeDetail()`
- Action shortcuts (a, b, n, D) work via `showNodeDetail` state check

No changes needed if everything works, but verify:
- `j/k` navigation works with sidebar open
- `Escape` closes sidebar
- `Enter` on a node opens sidebar
- Action shortcuts work when sidebar is open

#### 2. Optional: Improve resize handle visibility
**File**: `millflow/src/views/SheetView.tsx`

If the resize handle is hard to see/grab, enhance the styling:
```typescript
<PanelResizeHandle className="w-1 bg-gruvbox-bg-3 hover:bg-gruvbox-aqua-bright hover:w-1.5 transition-all cursor-col-resize" />
```

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `cd millflow && npm run build`
- [x] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] `j/k` navigation works with sidebar open
- [ ] `Escape` closes sidebar and removes from DOM
- [ ] `Enter` opens sidebar at remembered width
- [ ] Resize handle is easy to grab
- [ ] Width persists after page refresh

**Implementation Note**: This completes the implementation.

---

## Testing Strategy

### Unit Tests:
- localStorage mock for sidebarWidth persistence
- Store action tests for setSidebarWidth clamping

### Manual Testing Steps:
1. Open a sheet, click a node - sidebar should appear on right
2. Drag resize handle - both panels should resize smoothly
3. Press Escape - sidebar should disappear
4. Press Enter - sidebar should reappear at previous width
5. Resize to a new width, refresh page - width should persist
6. Navigate with j/k while sidebar open - should work
7. Use action shortcuts (a, b, n) - should work

## Performance Considerations

- PanelGroup uses CSS flexbox, minimal re-renders
- NodeDetailContent is memoized
- localStorage access only on width change

## References

- Research: `thoughts/shared/research/2025-12-19-draggable-sidebar-node-visibility.md`
- Factory-floor implementation: `factory-floor/src/App.tsx:459-556`
- Current NodeDetailPanel: `millflow/src/components/node/NodeDetailPanel.tsx`
- react-resizable-panels docs: https://github.com/bvaughn/react-resizable-panels
