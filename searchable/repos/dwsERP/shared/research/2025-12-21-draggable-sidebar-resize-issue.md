---
date: 2025-12-21T00:51:49-08:00
researcher: ariasulin
git_commit: 56c02369e91bfd15c379d9b7df2524340d80e7a2
branch: main
repository: dwsERP
topic: "Draggable sidebar resize only works about an inch"
tags: [research, codebase, react-resizable-panels, sidebar, sheetview]
status: complete
last_updated: 2025-12-21
last_updated_by: ariasulin
---

# Research: Draggable Sidebar Resize Issue

**Date**: 2025-12-21T00:51:49-08:00
**Researcher**: ariasulin
**Git Commit**: 56c02369e91bfd15c379d9b7df2524340d80e7a2
**Branch**: main
**Repository**: dwsERP

## Research Question
The sidebar only goes like an inch at most - investigating how the current resizable sidebar implementation works.

## Summary

The resizable sidebar in `SheetView.tsx` uses `react-resizable-panels` v4.0.13. The implementation uses `Group`, `Panel`, and `Separator` components. Compared to the working factory-floor reference implementation, there is a key difference: the `Group` component in millflow lacks `className="flex-1"` which the factory-floor has to make the panel group take up the full available vertical space.

## Detailed Findings

### Current Implementation in SheetView.tsx

**File**: `millflow/src/views/SheetView.tsx:3`
```typescript
import { Group, Panel, Separator, type Layout } from 'react-resizable-panels';
```

**Lines 176-202** - Panel group structure:
```typescript
<div className="flex-1 overflow-hidden">
  {showNodeDetail ? (
    <Group
      orientation="horizontal"
      onLayoutChange={(layout: Layout) => {
        if (layout['sidebar'] !== undefined) {
          setSidebarWidth(layout['sidebar']);
        }
      }}
    >
      <Panel id="dag" defaultSize={100 - sidebarWidth} minSize={50}>
        <div className="h-full overflow-auto p-6">
          <DAGRenderer ... />
        </div>
      </Panel>

      <Separator className="w-1 bg-gruvbox-bg-3 hover:bg-gruvbox-aqua-bright transition-colors cursor-col-resize" />

      <Panel id="sidebar" defaultSize={sidebarWidth} minSize={20} maxSize={50}>
        <NodeDetailContent onClose={toggleNodeDetail} />
      </Panel>
    </Group>
  ) : (
    ...
  )}
</div>
```

### Factory-Floor Reference Implementation

**File**: `factory-floor/src/App.tsx:459-556`

Key difference in factory-floor:
```typescript
<Group orientation="horizontal" className="flex-1">
  <Panel defaultSize="70%" minSize="50%">
    ...
  </Panel>
  <Separator className="w-1 bg-gruvbox-fg-4/10 hover:bg-gruvbox-aqua/50 transition-colors" />
  <Panel defaultSize="30%" minSize="20%" maxSize="50%">
    ...
  </Panel>
</Group>
```

### Key Differences Between Implementations

| Aspect | Millflow | Factory-Floor |
|--------|----------|---------------|
| Group className | (none) | `flex-1` |
| Size format | Numeric: `30` | String: `"30%"` |
| Panel IDs | Uses `id="dag"`, `id="sidebar"` | No IDs |

### Height Chain Analysis

1. **Layout.tsx:48**: Root container has `h-screen`
2. **Layout.tsx:75**: Main area has `flex-1 overflow-auto`
3. **SheetView.tsx:161**: Sheet container has `h-full flex flex-col`
4. **SheetView.tsx:176**: Panel wrapper has `flex-1 overflow-hidden`
5. **SheetView.tsx:178**: `Group` has **no className** (missing `flex-1`)

### State Management (appStore.ts)

**Lines 7-18** - Loading sidebar width from localStorage:
```typescript
const loadSidebarWidth = (): number => {
  try {
    const saved = localStorage.getItem('millflow-sidebar-width');
    if (saved) {
      const width = parseFloat(saved);
      if (width >= 20 && width <= 50) return width;
    }
  } catch {}
  return 30; // Default 30%
};
```

**Line 41** - State definition:
```typescript
sidebarWidth: number; // percentage (20-50)
```

**Line 159** - Initial value:
```typescript
sidebarWidth: loadSidebarWidth(),
```

**Lines 424-428** - setSidebarWidth action:
```typescript
setSidebarWidth: (width: number) => {
  const clamped = Math.min(50, Math.max(20, width));
  localStorage.setItem('millflow-sidebar-width', String(clamped));
  set({ sidebarWidth: clamped });
},
```

### NodeDetailContent Component

**File**: `millflow/src/components/node/NodeDetailContent.tsx:49-50`
```typescript
return (
  <div className="h-full flex flex-col bg-gruvbox-bg border-l border-gruvbox-bg-2 overflow-hidden">
```

The component uses `h-full` to fill parent height.

### Package Versions

- millflow: `react-resizable-panels@^4.0.13`
- factory-floor: `react-resizable-panels@^4.0.8`

## Code References

- `millflow/src/views/SheetView.tsx:176-202` - Panel group implementation
- `millflow/src/store/appStore.ts:7-18` - loadSidebarWidth function
- `millflow/src/store/appStore.ts:41` - sidebarWidth state definition
- `millflow/src/store/appStore.ts:424-428` - setSidebarWidth action
- `millflow/src/components/node/NodeDetailContent.tsx:49-50` - Sidebar content root
- `millflow/src/components/Layout.tsx:47-82` - App layout structure
- `factory-floor/src/App.tsx:459-556` - Working reference implementation

## Architecture Documentation

The sidebar uses `react-resizable-panels` library which requires:
1. A parent container with defined dimensions
2. `Group` component wrapping `Panel` components
3. `Separator` component between panels for the resize handle
4. Panel sizes specified as percentages (0-100 numeric or "X%" string format)

The current implementation stores sidebar width in Zustand state and persists to localStorage under key `millflow-sidebar-width`. Width is constrained to 20-50% range.

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2025-12-20-millflow-draggable-sidebar.md` - Original implementation plan
- `thoughts/shared/research/2025-12-19-draggable-sidebar-node-visibility.md` - Earlier research on sidebar visibility

## Open Questions

1. Does the `Group` component need `className="flex-1"` to properly fill the flex container?
2. Does the size format (numeric `30` vs string `"30%"`) affect behavior?
3. Is there a CSS height calculation issue preventing the Group from expanding?
