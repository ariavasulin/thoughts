---
date: 2025-12-19T23:57:14Z
researcher: Claude
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "MillFlow Parallel Node Display & Node Creation Fix"
tags: [implementation, millflow, dag, parallel-branches, node-creation]
status: ready_for_implementation
last_updated: 2025-12-19
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: MillFlow Parallel Nodes & Node Creation Fix

## Task(s)

**Status: Ready for Implementation (Plan Complete)**

Two issues need fixing in MillFlow's DAG view:

1. **Parallel nodes look cramped** - When multiple nodes are at the same level (e.g., 3 parallel branches), they're squished together with excessive text truncation. User wants them **centered** with more horizontal room.

2. **Node creation not working** - The 'o'/'O' keyboard shortcuts to insert nodes aren't functioning. Investigation confirmed the system IS fully implemented, but needs verification that `selectedNodeId` is being set properly in all scenarios.

## Critical References

- **Implementation plan**: `thoughts/shared/plans/2025-12-19-millflow-parallel-nodes-and-creation-fix.md`
- **Research document**: `thoughts/shared/research/2025-12-19-millflow-dag-rendering-node-creation.md`
- **Screenshot reference**: `/Users/ariasulin/Desktop/JY CNC Maria.png` (shows the cramped parallel nodes)

## Recent changes

No code changes made this session - this was planning/research only.

## Learnings

1. **Parallel node layout issue**: `ParallelBranch.tsx:34-36` uses `flex gap-4` with `flex-1` on each node wrapper, forcing equal width distribution. With 3 nodes, each gets ~33% width causing excessive truncation.

2. **Node creation is implemented**: The keyboard bindings ('o'/'O'), NodeCreator component, and insertNode store action all exist and are wired up correctly:
   - `millflow/src/hooks/useKeyboard.ts:239-253` - keyboard bindings
   - `millflow/src/components/dag/NodeCreator.tsx` - UI component
   - `millflow/src/store/appStore.ts:426-502` - insertNode action
   - `millflow/src/views/SheetView.tsx:157-165` - overlay rendering

3. **Critical requirement for node creation**: The keyboard binding requires `selectedNodeId` to be truthy:
   ```typescript
   if (view === 'sheet' && selectedSheetId && selectedNodeId) {
     setNodeCreationMode('after');
   }
   ```

4. **selectedNodeId syncing was fixed earlier today**: Handoff `2025-12-19_15-41-02_millflow-keyboard-fixes.md` documents fixes to sync `selectedNodeId` during j/k navigation and when entering sheet view.

## Artifacts

- `thoughts/shared/plans/2025-12-19-millflow-parallel-nodes-and-creation-fix.md` - Implementation plan with two phases
- `thoughts/shared/research/2025-12-19-millflow-dag-rendering-node-creation.md` - Detailed research on how both systems work

## Action Items & Next Steps

### Phase 1: Fix Parallel Node Layout
1. Edit `millflow/src/components/dag/ParallelBranch.tsx`:
   - Line 34: Change `<div className="flex gap-4 pt-4">` to `<div className="flex justify-center gap-4 pt-4">`
   - Line 36: Change `<div key={node.id} className="flex-1 relative">` to `<div key={node.id} className="relative w-56 min-w-0">`
   - Lines 31, 50: Update split/join line widths to be dynamic based on node count

2. Verify with `cd millflow && npm run build && npm run dev`

### Phase 2: Verify Node Creation
1. Test node creation flow manually (navigate to sheet, press 'o')
2. If not working, add fallback useEffect in `SheetView.tsx` to ensure `selectedNodeId` is set on mount
3. Test insert before ('O'), escape to cancel, and undo ('u')

### Final
- Run build and lint
- Manual verification per plan's success criteria

## Other Notes

**Key files for implementation:**
- `millflow/src/components/dag/ParallelBranch.tsx` - Main file to edit for layout fix
- `millflow/src/views/SheetView.tsx` - May need fallback fix for node creation

**User preference:** Prefer no horizontal scrolling. Center nodes, give them more room (proposed `w-56` = 224px per node), only truncate when necessary.

**Testing sheet:** `sheet-012` (Reception Desk for Marriott Lobby job) has 3 parallel nodes - good for testing the layout fix.

**Run commands from millflow directory:**
```bash
cd millflow
npm run dev      # Dev server
npm run build    # Type check + build
npm run lint     # ESLint
```
