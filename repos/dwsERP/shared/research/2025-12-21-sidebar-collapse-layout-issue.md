---
date: 2025-12-21T01:55:12-08:00
researcher: ariasulin
git_commit: 56c02369e91bfd15c379d9b7df2524340d80e7a2
branch: main
repository: dwsERP
topic: "Sidebar collapse leaves empty space on right side"
tags: [research, codebase, react-resizable-panels, sidebar, sheetview, flexbox, layout]
status: complete
last_updated: 2025-12-21
last_updated_by: ariasulin
---

# Research: Sidebar Collapse Layout Issue

**Date**: 2025-12-21T01:55:12-08:00
**Researcher**: ariasulin
**Git Commit**: 56c02369e91bfd15c379d9b7df2524340d80e7a2
**Branch**: main
**Repository**: dwsERP

## Research Question

When the sidebar is closed (`showNodeDetail` is false), empty space remains on the right side of the screen where the sidebar used to be. The DAG content should be full-screen and centered when the sidebar is closed.

## Summary

**Root Cause Confirmed**: The fallback div at `SheetView.tsx:206` is missing `flex-1`, causing it to not fill the parent flex row container's width.

The parent container at line 176 is a **flex row** (`flex-1 flex overflow-hidden`). In flexbox row direction:
- Children do NOT automatically stretch horizontally (width)
- Children only stretch vertically (height) by default
- Without `flex-grow` (like `flex-1`), a child takes only its content width

When the sidebar is shown, the `<Group className="flex-1">` correctly fills the parent. When hidden, the fallback `<div className="h-full overflow-auto p-6">` lacks `flex-1`, so it only takes its content width, leaving empty space on the right.

## Detailed Findings

### 1. Layout Chain Analysis

```
Layout.tsx:48      → <div className="flex flex-col h-screen">
Layout.tsx:75      →   <main className="flex-1 overflow-auto">
SheetView.tsx:161  →     <div className="h-full flex flex-col">
SheetView.tsx:176  →       <div className="flex-1 flex overflow-hidden">  ← FLEX ROW CONTAINER
                               ↓
When showNodeDetail=true:      <Group className="flex-1">     ← HAS flex-1 ✓
When showNodeDetail=false:     <div className="h-full ...">   ← MISSING flex-1 ✗
```

### 2. Flexbox Behavior in Row Direction

In a flex row container:
- `align-items: stretch` (default) stretches children in the **cross axis** (height)
- Main axis (width) sizing uses `flex-grow`, `flex-shrink`, `flex-basis`
- Without `flex-1` (which sets `flex-grow: 1`), a child takes only its content width

**Current code** (SheetView.tsx:206):
```tsx
<div className="h-full overflow-auto p-6">
```
- `h-full` - correctly fills height (cross axis)
- No width specification - takes content width only

**When Group is shown** (SheetView.tsx:178):
```tsx
<Group className="flex-1" ...>
```
- `flex-1` - correctly fills width (main axis)

### 3. DAGRenderer Content Sizing

DAGRenderer has no intrinsic width constraints. Its content determines width:
- Single nodes: `max-w-2xl mx-auto` (672px max, centered)
- Branch regions: 600-1000px fixed width, centered with `flex justify-center`

These elements are intentionally narrower than full-width and use `mx-auto` for centering. The centering works correctly **within the container's width**. The problem is the container itself isn't full-width.

### 4. Comparison with Other Views

Other views (DashboardView, JobView, JobsView) use:
```tsx
<div className="p-6 max-w-4xl mx-auto">
```

These work correctly because:
1. They render directly inside `<main className="flex-1 overflow-auto">` (Layout.tsx:75)
2. The `main` element is NOT a flex container - it's just `overflow-auto`
3. Block elements naturally take full width in non-flex parents
4. `max-w-4xl mx-auto` centers the content

SheetView is different because it has its own internal flex structure for the resizable panel system.

### 5. react-resizable-panels Behavior

The library does NOT cause the issue:
- When Group unmounts, library state is destroyed
- State persistence is handled externally via Zustand + localStorage
- The conditional rendering itself is correct

The issue is purely CSS/flexbox related.

## Code References

- `millflow/src/views/SheetView.tsx:176` - Parent flex row container
- `millflow/src/views/SheetView.tsx:178` - Group with `flex-1` (correct)
- `millflow/src/views/SheetView.tsx:206` - Fallback div missing `flex-1` (BUG)
- `millflow/src/components/dag/DAGRenderer.tsx:25` - DAGRenderer root (no width constraints)
- `millflow/src/components/dag/DAGRenderer.tsx:36` - Single node centering (`max-w-2xl mx-auto`)
- `millflow/src/components/Layout.tsx:75` - Layout main element

## Architecture Documentation

### Flex Container Hierarchy in SheetView

```
SheetView root (h-full flex flex-col)
├── Header (fixed height)
└── Content area (flex-1 flex overflow-hidden) ← FLEX ROW
    ├── When sidebar open: Group (flex-1) ← FILLS WIDTH
    │   ├── Panel (dag)
    │   ├── Separator
    │   └── Panel (sidebar)
    └── When sidebar closed: div (h-full) ← CONTENT WIDTH ONLY
        └── DAGRenderer (content-sized)
```

### DAGRenderer Content Structure

```
DAGRenderer (space-y-3, no width)
├── Single region → max-w-2xl mx-auto (centered at 672px max)
└── Branch region → flex justify-center
    └── Fixed width container (600-1000px)
        └── Branch columns (flex-1 each)
```

## The Fix

Change `SheetView.tsx:206` from:
```tsx
<div className="h-full overflow-auto p-6">
```

To:
```tsx
<div className="flex-1 h-full overflow-auto p-6">
```

This makes the div grow to fill the parent flex container's width, providing full-width space for the centered DAGRenderer content.

## Historical Context (from thoughts/)

- `thoughts/shared/handoffs/general/2025-12-21_01-43-57_millflow-sidebar-resize-fix.md` - Previous handoff documenting the resize issue fix and identifying this new issue
- `thoughts/shared/research/2025-12-21-draggable-sidebar-resize-issue.md` - Earlier research on the "inch" resize constraint

## Open Questions

None - root cause is confirmed and fix is straightforward.
