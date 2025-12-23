---
date: 2025-12-21T01:02:39-08:00
researcher: ariasulin
git_commit: 56c02369e91bfd15c379d9b7df2524340d80e7a2
branch: main
repository: dwsERP
topic: "Millflow Sidebar Resize Issue Investigation"
tags: [debugging, react-resizable-panels, sidebar, sheetview]
status: in_progress
last_updated: 2025-12-21
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Millflow Sidebar Resize Issue

## Task(s)
**Status: IN PROGRESS - not yet resolved**

User reported that the resizable sidebar in SheetView "only goes about an inch" - meaning the resize functionality is severely constrained. The sidebar should be resizable between 20-50% of the viewport width.

## Critical References
- `thoughts/shared/plans/2025-12-20-millflow-draggable-sidebar.md` - Original implementation plan
- `factory-floor/src/App.tsx:459-556` - Working reference implementation using same library

## Recent changes
Made two attempts to fix, neither resolved the issue:

1. `millflow/src/views/SheetView.tsx:180` - Added `className="flex-1"` to Group component (didn't work because parent isn't a flex container)
2. `millflow/src/views/SheetView.tsx:180` - Changed to `className="h-full"` to fill parent height (still not working per user)

## Learnings

### Library API confirmed correct
- `react-resizable-panels@4.0.13` exports `Group`, `Panel`, `Separator`, `Layout` (verified in `node_modules/react-resizable-panels/dist/react-resizable-panels.d.ts`)
- Size values can be numeric (0-100) representing percentages

### Key differences from working factory-floor implementation
| Aspect | Millflow | Factory-Floor |
|--------|----------|---------------|
| Size format | Numeric: `30` | String: `"30%"` |
| Panel IDs | Uses `id="dag"`, `id="sidebar"` | No IDs |
| Group className | `h-full` (after fix) | `flex-1` |

### Height chain analysis
1. `Layout.tsx:48` - Root: `h-screen`
2. `Layout.tsx:75` - Main: `flex-1 overflow-auto`
3. `SheetView.tsx:161` - Container: `h-full flex flex-col`
4. `SheetView.tsx:176` - Panel wrapper: `flex-1 overflow-hidden`
5. `SheetView.tsx:178` - Group: `h-full`

### Panel configuration
```tsx
<Panel id="dag" defaultSize={100 - sidebarWidth} minSize={50}>
<Panel id="sidebar" defaultSize={sidebarWidth} minSize={20} maxSize={50}>
```
- `sidebarWidth` defaults to 30, stored in Zustand and persisted to localStorage
- Constrained to 20-50% range in `appStore.ts:424-428`

## Artifacts
- `thoughts/shared/research/2025-12-21-draggable-sidebar-resize-issue.md` - Initial research document

## Action Items & Next Steps

1. **Try string percentage format** - Change size props from numeric `30` to string `"30%"` to match factory-floor
2. **Check if Layout's overflow-auto affects sizing** - `Layout.tsx:75` has `overflow-auto` which might interfere
3. **Add explicit width to parent container** - The `flex-1 overflow-hidden` wrapper might need `w-full`
4. **Debug with browser DevTools** - Inspect the actual rendered dimensions of:
   - The Group container
   - The Panel elements
   - The parent containers
5. **Check if Separator needs width** - Current Separator has `w-1` (4px) - might need adjustment

## Other Notes

### Key files to examine
- `millflow/src/views/SheetView.tsx:176-203` - Panel group implementation
- `millflow/src/store/appStore.ts:7-18, 424-428` - Sidebar width state management
- `millflow/src/components/Layout.tsx:47-82` - App layout structure
- `factory-floor/src/App.tsx:459-556` - Working reference to compare against

### Library documentation
- react-resizable-panels: https://github.com/bvaughn/react-resizable-panels

### User's exact symptom
"the sidebar only goes like an inch at most" - This suggests:
- The sidebar IS rendering and IS resizable
- But the resize range is extremely limited
- Could be a min/max constraint issue, a container sizing issue, or CSS overflow issue
