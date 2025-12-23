---
date: 2025-12-19T23:47:59-08:00
researcher: ariasulin
git_commit: 201f79e68db6ef380d44c9b4d64ee7e34b507206
branch: main
repository: dwsERP
topic: "Draggable Sidebar and Node View Visibility"
tags: [research, codebase, sidebar, NodeDetailPanel, react-resizable-panels, layout]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Draggable Sidebar and Node View Visibility

**Date**: 2025-12-19T23:47:59-08:00
**Researcher**: ariasulin
**Git Commit**: 201f79e68db6ef380d44c9b4d64ee7e34b507206
**Branch**: main
**Repository**: dwsERP

## Research Question
Can the sidebar be made draggable like the old prototype, and can the node view be completely visible next to (or under) the sidebar?

## Summary

**Yes, this is feasible.** The current millflow sidebar uses a fixed overlay pattern that dims the DAG view behind it. The factory-floor prototype demonstrates a working implementation using `react-resizable-panels` that allows the canvas and sidebar to share screen real estate side-by-side with draggable resizing.

### Key Differences

| Aspect | Current millflow | Factory-floor prototype |
|--------|------------------|------------------------|
| **Positioning** | `fixed inset-0` overlay | `react-resizable-panels` side-by-side |
| **Width control** | Fixed `max-w-lg` (512px) | Draggable, min 20%, max 50% |
| **DAG visibility** | Dimmed behind overlay | Fully visible alongside |
| **Interaction** | Click backdrop to close | Drag separator to resize |

## Detailed Findings

### Current millflow Implementation (NodeDetailPanel)

**File**: `millflow/src/components/node/NodeDetailPanel.tsx`

The current sidebar is a full-screen overlay:

```tsx
// Lines 46-52
<div className="fixed inset-0 z-40 flex justify-end bg-black/40">
  <div className="w-full max-w-lg bg-gruvbox-bg border-l overflow-auto">
    {content}
  </div>
</div>
```

Key characteristics:
- `fixed inset-0`: Covers entire viewport
- `z-40`: Stacks above all content
- `bg-black/40`: Semi-transparent backdrop dims the DAG view
- `max-w-lg`: Fixed max width of 512px
- No resize/drag capability

The panel is rendered at root level in `App.tsx:45`, completely independent of the Layout structure.

### Factory-floor Prototype Implementation

**Files**:
- `factory-floor/src/App.tsx:459-556`
- `factory-floor/src/components/sidebar/SheetSidebar.tsx`

Uses `react-resizable-panels` library (version 4.0.8):

```tsx
// App.tsx lines 459-556
<PanelGroup direction="horizontal">
  <Panel defaultSize={70} minSize={50}>
    {/* React Flow canvas - fully visible */}
  </Panel>

  <PanelResizeHandle className="w-1 bg-gruvbox-fg-4/10 hover:bg-gruvbox-aqua/50" />

  <Panel defaultSize={30} minSize={20} maxSize={50}>
    <SheetSidebar />
  </Panel>
</PanelGroup>
```

Key characteristics:
- Canvas and sidebar share screen horizontally
- Draggable separator with 1px width
- Sidebar: default 30%, min 20%, max 50% of viewport
- Both views fully visible simultaneously
- Toggle button at `App.tsx:477-488` to show/hide entire sidebar

### Layout Structure Comparison

**Current millflow (overlay pattern)**:
```
┌─────────────────────────────┐
│      Layout (full width)     │
│  ┌─────────────────────────┐│
│  │    DAG View (dimmed)    ││
│  │                         ││
│  └─────────────────────────┘│
└─────────────────────────────┘
    ┌─────────────┐
    │   Sidebar   │ ← floating overlay
    │  (fixed)    │   on top
    └─────────────┘
```

**Factory-floor (side-by-side pattern)**:
```
┌───────────────────┬──│──────────┐
│                   │  │          │
│   React Flow      │  │ Sidebar  │
│   Canvas          │←→│ (resize) │
│   (fully visible) │  │          │
│                   │  │          │
└───────────────────┴──│──────────┘
                     drag handle
```

## Code References

- `millflow/src/components/node/NodeDetailPanel.tsx:46-52` - Current overlay positioning
- `millflow/src/App.tsx:45` - NodeDetailPanel rendered at root level
- `millflow/src/views/SheetView.tsx:171-178` - DAG content area
- `factory-floor/src/App.tsx:459-556` - Resizable panel implementation
- `factory-floor/src/App.tsx:547` - Panel resize handle styling
- `factory-floor/package.json:20` - `react-resizable-panels: ^4.0.8`

## Architecture Documentation

### Implementation Approach for millflow

To achieve the requested behavior, millflow would need to:

1. **Add react-resizable-panels dependency**
   ```bash
   cd millflow && npm install react-resizable-panels
   ```

2. **Wrap SheetView content in PanelGroup**
   - Replace current fixed overlay with side-by-side panels
   - DAG view in left Panel, NodeDetailPanel content in right Panel

3. **Modify NodeDetailPanel**
   - Remove `fixed inset-0` and backdrop
   - Convert to Panel-compatible component
   - Keep the same internal content structure

4. **Update toggle behavior**
   - Instead of overlay toggle, panel could collapse/expand
   - Or use conditional rendering with width transition

### Alternative: Draggable Overlay

If keeping the overlay pattern but making it draggable over the content:
- Add a drag handle (left edge of panel)
- Use CSS `resize` property or implement custom drag logic
- Store width in Zustand state for persistence
- Remove or make backdrop optional

## Historical Context (from thoughts/)

No existing research documents found on this topic.

## Related Research

No related research documents in thoughts/shared/research/.

## Open Questions

1. Should the sidebar collapse entirely or resize to a minimum width?
2. Should the draggable width persist across sessions (localStorage)?
3. Should clicking outside the sidebar close it, or only the Escape key?
4. Should there be preset width options (compact/full)?
