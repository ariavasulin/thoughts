---
date: 2025-12-19T22:37:39Z
researcher: Claude
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "MillFlow Remaining Features Implementation Plan"
tags: [implementation, planning, millflow, keyboard-first, pm-dashboard, react]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: MillFlow Remaining Features Implementation Plan

## Task(s)

**Completed:**
- Resumed from previous handoff documenting MillFlow PM Dashboard implementation
- Researched existing millflow codebase thoroughly (keyboard handling, store, DAG rendering, types, components)
- Created comprehensive implementation plan for all remaining features (4 phases)

**Status:** Planning complete, ready for implementation.

## Critical References

- Implementation plan: `thoughts/shared/plans/2025-12-19-millflow-remaining-features.md`
- Previous handoff: `thoughts/shared/handoffs/general/2025-12-19_13-07-16_millflow-pm-dashboard.md`
- Original implementation plan: `thoughts/shared/plans/2025-12-19-millflow-pm-dashboard.md`

## Recent changes

No code changes this session - planning only. The plan document was created at:
- `thoughts/shared/plans/2025-12-19-millflow-remaining-features.md`

## Learnings

1. **Keyboard handling architecture** (`millflow/src/hooks/useKeyboard.ts`): Uses custom sequence handler with 500ms timeout for multi-key sequences (g h, d d). Refs track pending keys. Already has placeholders for h/l navigation at lines 178-186.

2. **Store structure** (`millflow/src/store/appStore.ts`): All state and actions in single Zustand store. Has `addNode`, `deleteNode`, `updateNode`, `assignNode`, `addBlocker`, `addNote` already implemented. Missing: undo/redo, split/join, deliveries view.

3. **DAG linearization** (`millflow/src/components/dag/DAGRenderer.tsx:26-98`): Uses BFS to build "levels" - each level is single node or parallel nodes. Parallel detection checks if multiple children converge to same downstream node.

4. **Action form hints exist but not implemented**: NodeDetailPanel (`millflow/src/components/node/NodeDetailPanel.tsx:245-262`) already shows footer hints for a/b/n keys, but forms don't exist yet.

## Artifacts

- **Implementation Plan**: `thoughts/shared/plans/2025-12-19-millflow-remaining-features.md`
  - Phase 1: DAG Editing (o/O insert, s split, J join)
  - Phase 2: Node Action Forms (a assign, n note, b blocker, d due date)
  - Phase 3: Navigation Enhancements (h/l parallel, Tab/Shift+Tab, g d deliveries)
  - Phase 4: Polish (undo/redo, templates, key sequence indicator)

## Action Items & Next Steps

### Priority 1: Implement Phase 1 - DAG Editing
1. Create `millflow/src/components/dag/NodeCreator.tsx` - inline node creation UI
2. Add `insertNode`, `splitNode`, `joinNodes` actions to store
3. Add `nodeCreationMode` state to store
4. Add keyboard bindings for `o`, `O`, `s` in useKeyboard.ts
5. Integrate NodeCreator into SheetView.tsx

### Priority 2: Implement Phase 2 - Node Action Forms
1. Create `millflow/src/components/node/AssigneePicker.tsx`
2. Create `millflow/src/components/node/NoteForm.tsx`
3. Create `millflow/src/components/node/BlockerForm.tsx`
4. Create `millflow/src/components/node/DueDatePicker.tsx`
5. Add `activeActionForm` state and keyboard bindings

### Priority 3: Implement Phase 3 - Navigation
1. Add `selectedBranchIndex` to store for parallel branch tracking
2. Update DAGRenderer/ParallelBranch for branch highlighting
3. Implement Tab/Shift+Tab for actionable item jumping
4. Create `millflow/src/views/DeliveriesView.tsx`
5. Add `g d` sequence handling

### Priority 4: Implement Phase 4 - Polish
1. Add undo/redo history stack to store
2. Create `millflow/src/data/templates.ts` and TemplatePicker component
3. Add pending key sequence indicator to ShortcutBar
4. Update HelpModal with all new shortcuts

## Other Notes

**Running the app:**
```bash
cd millflow
npm run dev      # Dev server on localhost:5173
npm run build    # TypeScript check + production build
```

**Key files to modify:**
- `millflow/src/hooks/useKeyboard.ts` - Add all new key bindings
- `millflow/src/store/appStore.ts` - Add new state and actions
- `millflow/src/views/SheetView.tsx` - Integrate new overlays/forms
- `millflow/src/components/dag/DAGRenderer.tsx` - Branch navigation support

**Gruvbox theme:** All colors use `gruvbox-*` Tailwind classes from `millflow/tailwind.config.js`. Never use raw color values.

**Component patterns:** All components use `memo()` for optimization, named exports, `cn()` for conditional class merging.
