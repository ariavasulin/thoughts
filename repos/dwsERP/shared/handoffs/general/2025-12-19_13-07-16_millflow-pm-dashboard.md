---
date: 2025-12-19T21:07:16Z
researcher: Claude
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "MillFlow PM Dashboard Implementation"
tags: [implementation, millflow, keyboard-first, pm-dashboard, react, vite]
status: in_progress
last_updated: 2025-12-19
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: MillFlow Keyboard-First PM Dashboard

## Task(s)

**Completed:**
- Full project scaffolding (Vite + React 19 + TypeScript + Tailwind + Zustand)
- Gruvbox dark theme matching existing factory-floor project
- Type definitions for all data models (Job, Sheet, Node, Blocker, Note, Person, etc.)
- Complete mock data (5 jobs, 6 sheets with DAG structures, 17 people/departments/vendors)
- Main Dashboard view with collapsible sections (Your Move, At Risk, Waiting On Others, Flowing)
- Jobs list view with priority stars, progress, blocker counts
- Job detail view with sheets list, upcoming deliveries, blockers summary
- Sheet DAG view with linearized workflow and parallel branch visualization
- Node detail panel with full info display (status, assignees, blockers, notes, references, history)
- Command palette (Cmd+K) with fuzzy search across jobs, sheets, people, actions
- Help modal with full keyboard reference
- Core keyboard navigation (j/k, Enter, Esc, g h, g j, g g, G, ?)
- Contextual shortcut bar that updates per view

**Work In Progress / Not Implemented:**
- DAG editing (o/O insert node, s split, J join, template insertion)
- Node action forms (a assign dropdown, n note form, b blocker form, d due date)
- Parallel branch navigation (h/l keys)
- Tab/shift+tab for next/previous actionable item
- Deliveries view (g d)
- Undo/redo (u/ctrl+r)

## Critical References

- Original PRD with full spec: User provided inline (not saved as file)
- Implementation plan: `thoughts/shared/plans/2025-12-19-millflow-pm-dashboard.md`
- Gruvbox theme reference: `factory-floor/tailwind.config.js`

## Recent changes

All changes are new files in the `millflow/` directory:

- `millflow/package.json` - Dependencies and scripts
- `millflow/tailwind.config.js` - Gruvbox theme with extended bg levels
- `millflow/src/types/index.ts` - All TypeScript types
- `millflow/src/store/appStore.ts` - Zustand store with all state and actions
- `millflow/src/data/mockData.ts` - Complete mock data with 5 jobs, sheets, DAGs
- `millflow/src/hooks/useKeyboard.ts` - Keyboard handler with sequence support (g h, d d, etc.)
- `millflow/src/components/Layout.tsx` - Main layout shell
- `millflow/src/components/ShortcutBar.tsx` - Contextual keyboard hints
- `millflow/src/components/HelpModal.tsx` - Full keyboard reference
- `millflow/src/components/CommandPalette.tsx` - Fuzzy search command palette
- `millflow/src/components/dashboard/DashboardSection.tsx` - Collapsible sections
- `millflow/src/components/dashboard/DashboardItem.tsx` - Dashboard list items
- `millflow/src/components/job/SheetListItem.tsx` - Sheet list items
- `millflow/src/components/dag/DAGRenderer.tsx` - Linearizes and renders DAG
- `millflow/src/components/dag/DAGNode.tsx` - Individual node cards
- `millflow/src/components/dag/ParallelBranch.tsx` - Side-by-side parallel nodes
- `millflow/src/components/node/NodeDetailPanel.tsx` - Slide-in node detail panel
- `millflow/src/views/DashboardView.tsx` - Main dashboard
- `millflow/src/views/JobsView.tsx` - Jobs list
- `millflow/src/views/JobView.tsx` - Single job detail
- `millflow/src/views/SheetView.tsx` - Sheet DAG view
- `millflow/src/App.tsx` - Main app with view routing

## Learnings

1. **Keyboard sequence handling**: Implemented custom sequence handler in `useKeyboard.ts` with 500ms timeout for multi-key sequences like `g h`, `g j`, `d d`. Uses a ref to track pending keys.

2. **DAG linearization**: The DAG renderer in `DAGRenderer.tsx:30-80` uses BFS to build "levels" - each level is either a single node or parallel nodes. Parallel detection works by checking if a node has multiple children that all converge to the same downstream node.

3. **State structure**: View routing is done via `view` state in Zustand store ('dashboard' | 'jobs' | 'job' | 'sheet'). Selection tracking uses `selectedIndex` for keyboard navigation and `selectedNodeId` for specific node selection.

4. **Gruvbox theme**: Extended the theme in `tailwind.config.js` to include `bg-1` through `bg-4` levels for proper dark mode hierarchy. Use `gruvbox-bg-hard` (#1d2021) as the main background.

5. **Lucide icons**: All node types and statuses map to Lucide icons in `lib/icons.ts`. This replaces emoji from the original PRD spec.

## Artifacts

- Implementation plan: `thoughts/shared/plans/2025-12-19-millflow-pm-dashboard.md`
- Full application: `millflow/` directory (self-contained Vite project)
- Type definitions: `millflow/src/types/index.ts`
- Mock data: `millflow/src/data/mockData.ts`
- Store with all actions: `millflow/src/store/appStore.ts`
- Keyboard handling: `millflow/src/hooks/useKeyboard.ts`

## Action Items & Next Steps

### Priority 1: DAG Editing
1. Create `NodeCreator.tsx` component - inline node creation UI with name input and type selector
2. Add keyboard bindings for `o` (insert after) and `O` (insert before) in `useKeyboard.ts`
3. Implement split (`s`) to create parallel branches
4. Implement join (`J`) to set convergence points

### Priority 2: Node Action Forms
1. Create `AssigneePicker.tsx` - dropdown showing suggested people first, filterable, shows load
2. Create `NoteForm.tsx` - inline textarea that auto-timestamps
3. Create `BlockerForm.tsx` - type dropdown, description, owner, expected resolution
4. Add keyboard bindings: `a` for assign, `n` for note, `b` for blocker, `d` for due date

### Priority 3: Navigation Enhancements
1. Implement `h/l` for parallel branch navigation in DAG view
2. Add `tab`/`shift+tab` for jumping to next/previous actionable item
3. Create Deliveries view and add `g d` navigation

### Priority 4: Polish
1. Add undo/redo stack for DAG edits
2. Implement template system for common node sequences
3. Add visual feedback for pending key sequences (e.g., show "g..." in status)

## Other Notes

**Running the app:**
```bash
cd millflow
npm install  # Already done
npm run dev  # Starts on localhost:5173 (or next available port)
npm run build  # TypeScript check + production build
```

**Project structure:**
- All views are in `src/views/`
- Reusable components in `src/components/` organized by feature
- The store in `src/store/appStore.ts` contains ALL state and actions - add new actions there
- Mock data in `src/data/mockData.ts` includes helper functions like `getJobById`, `getSheetsForJob`, etc.

**Key patterns:**
- All components use `memo()` for React optimization
- Named exports preferred
- Use `cn()` from `lib/styles.ts` for conditional class merging (clsx + tailwind-merge)
- Keyboard events are handled globally in `useKeyboard.ts` - add new bindings there
- Icons come from Lucide React - mapped in `lib/icons.ts`

**The original PRD** was provided inline by the user and contains detailed ASCII mockups of all views, complete keyboard shortcut tables, and the full data model. It's comprehensive and should be referenced for UI details.
