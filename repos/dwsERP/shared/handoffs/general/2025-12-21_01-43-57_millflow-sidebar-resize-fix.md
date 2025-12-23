---
date: 2025-12-21T01:43:57-08:00
researcher: ariasulin
git_commit: 56c02369e91bfd15c379d9b7df2524340d80e7a2
branch: main
repository: dwsERP
topic: "Millflow Sidebar Resize Fix"
tags: [debugging, react-resizable-panels, sidebar, sheetview]
status: in_progress
last_updated: 2025-12-21
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Millflow Sidebar Resize Fix - Partial Resolution

## Task(s)
**Status: IN PROGRESS - Partially resolved, new issue discovered**

Original issue: User reported resizable sidebar "only goes about an inch" - resize severely constrained.

**Resolved:**
- Sidebar now resizes properly between 20-50%
- Sidebar content is now vertically scrollable

**New issue discovered:**
When the sidebar is closed (`showNodeDetail` is false), it's still taking up space on the right side of the screen. The sidebar panel appears to not be collapsing - staying the same size or even larger when closed.

## Critical References
- `thoughts/shared/handoffs/general/2025-12-21_01-02-39_millflow-sidebar-resize-issue.md` - Previous handoff with initial investigation
- `factory-floor/src/App.tsx:459-556` - Working reference implementation

## Recent changes
Made several fixes to `millflow/src/views/SheetView.tsx`:

1. **Line 181**: Added `defaultLayout` prop to Group using numeric percentages (the correct format for this prop)
2. **Line 176**: Changed parent container from `flex-1 overflow-hidden` to `flex-1 flex overflow-hidden` (made it a flex container)
3. **Line 180**: Changed Group className from `h-full` to `flex-1`
4. **Lines 188, 201**: Removed `defaultSize` from individual Panels (now handled by Group's `defaultLayout`)
5. **Line 201**: Changed `minSize` and `maxSize` to string percentages ("20%", "50%")

## Learnings

### Root cause of original "inch" issue - SIZE FORMAT
The react-resizable-panels library interprets size values as:
- **Numeric values = PIXELS** (not percentages!)
- **Strings without units = percentages**
- **Strings with % = explicit percentages**

Original code used `defaultSize={30}` which meant **30 pixels**, not 30%.

### Correct API usage
- `defaultLayout` on Group: Uses numbers as percentages (0-100) - e.g., `{ dag: 70, sidebar: 30 }`
- `defaultSize` on Panel: Uses strings for percentages - e.g., `"30%"` or `"30"`
- `onLayoutChange` callback: Receives Layout object with numbers as percentages (0-100)

### Parent container requirements
The Group's parent must be a flex container for `flex-1` on Group to work. Changed parent to include `flex` class.

### Library API verified
- Library exports: `Group`, `Panel`, `Separator`, `Layout` (type)
- Version: `react-resizable-panels@4.0.13`
- Group cannot override: `display`, `flex-direction`, `flex-wrap`, `overflow`
- Separator cannot override: `flex-grow`, `flex-shrink`

## Artifacts
- `millflow/src/views/SheetView.tsx:176-204` - Updated panel group implementation
- `thoughts/shared/research/2025-12-21-draggable-sidebar-resize-issue.md` - Research document

## Action Items & Next Steps

1. **Investigate why sidebar doesn't collapse when closed**
   - The current implementation uses conditional rendering (lines 177-214)
   - When `showNodeDetail` is false, should render only the DAG without the Group
   - Check if there's CSS/layout issue causing space to be reserved

2. **Check the conditional rendering logic**
   - `SheetView.tsx:177`: `{showNodeDetail ? ( <Group>...</Group> ) : ( <div>...</div> )}`
   - The Group should not be in the DOM when showNodeDetail is false
   - May need to use browser DevTools to verify what's actually rendering

3. **Compare with factory-floor pattern**
   - Factory-floor conditionally renders only the second Panel + Separator inside the Group
   - Millflow conditionally renders the entire Group
   - May need to switch to factory-floor's pattern

## Other Notes

### Current SheetView structure (lines 176-214)
```
<div className="flex-1 flex overflow-hidden">
  {showNodeDetail ? (
    <Group defaultLayout={...} onLayoutChange={...}>
      <Panel id="dag">...</Panel>
      <Separator />
      <Panel id="sidebar">...</Panel>
    </Group>
  ) : (
    <div className="h-full overflow-auto p-6">
      <DAGRenderer />
    </div>
  )}
</div>
```

### Factory-floor pattern (different approach)
Factory-floor keeps Group always mounted and conditionally renders only the Separator + second Panel inside it. This might be needed to properly handle the collapse behavior.

### Key files to examine
- `millflow/src/views/SheetView.tsx:176-214` - Conditional rendering logic
- `millflow/src/components/node/NodeDetailContent.tsx` - Sidebar content component
- `millflow/src/store/appStore.ts:149, 393` - showNodeDetail state and toggleNodeDetail action
