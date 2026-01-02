---
date: 2025-12-19T16:47:57-08:00
researcher: ariasulin
git_commit: 96bdd2df9b027434f43a051b614a8e705e96f1ce
branch: main
repository: dwsERP
topic: "Millflow SmartNodeCreator with Autofill"
tags: [implementation, millflow, node-creator, autofill, react]
status: in_progress
last_updated: 2025-12-19
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Millflow SmartNodeCreator with NLP-style Autofill

## Task(s)

**Implementing a redesigned node creation popup for millflow** with:
- Autocomplete suggestions for operation names (borrowed from factory-floor)
- Automatic type inference (CNC → shop, Delivery → delivery, etc.)
- Quick-add panel (Tab to expand) for assignee + due date
- Keyboard-first UX

**Status:**
- Phase 1: ✅ Completed - Created `operations.ts` with 18 operation suggestions
- Phase 2: ✅ Completed - Created `SmartNodeCreator.tsx` component
- Phase 3: ✅ Completed - Integrated into `SheetView.tsx`
- Phase 4: 🔄 In Progress - Cleanup (delete old NodeCreator)

**Known Bugs to Fix:**
1. **Crash on Enter**: When pressing Enter to create a node, the app crashes
2. **Tab navigation broken**: After pressing Tab once to expand quick-add fields, Tab doesn't continue moving focus to the assignee/date fields

**Working from:**
- Plan: `thoughts/shared/plans/2025-12-19-millflow-smart-node-creator.md`
- Research: `thoughts/shared/research/2025-12-19-millflow-node-creator-autofill.md`

## Critical References

- `millflow/src/components/dag/SmartNodeCreator.tsx` - The new component with bugs
- `millflow/src/store/appStore.ts:431-513` - `insertNode` function that receives the data

## Recent changes

- `millflow/src/data/operations.ts:1-54` - NEW: Operation suggestions data with fuzzy search
- `millflow/src/components/dag/SmartNodeCreator.tsx:1-214` - NEW: Smart autocomplete node creator
- `millflow/src/views/SheetView.tsx:7` - Changed import from NodeCreator to SmartNodeCreator
- `millflow/src/views/SheetView.tsx:85-99` - Updated handleNodeCreate to accept new data structure
- `millflow/src/views/SheetView.tsx:172` - Changed JSX to use SmartNodeCreator

## Learnings

1. **insertNode already accepts spread data**: The `insertNode` function in appStore.ts uses `...nodeData` spread (line 452), so it will accept `assignees` and `due_date` without modification.

2. **No autocomplete libraries exist in codebase**: Neither millflow nor factory-floor have autocomplete. The CommandPalette (`CommandPalette.tsx`) is a custom implementation with fuzzy filtering that was used as the pattern for SmartNodeCreator.

3. **Type inference mapping**: Factory-floor has 12 OPERATION_TYPES mapped to millflow's 7 NodeTypes:
   - shop: CNC, Edge Banding, Veneer, Milling, Finishing, Bench, Drafting
   - external: Outsourced
   - material: Order Material, Receive Material
   - approval: Client Approval, Shop Drawing Approval
   - qc: QC Check, Final Inspection
   - delivery: Staging, Delivery
   - install: Field Install, Punch List

## Artifacts

- `thoughts/shared/plans/2025-12-19-millflow-smart-node-creator.md` - Implementation plan (Phases 1-2 checked off)
- `thoughts/shared/research/2025-12-19-millflow-node-creator-autofill.md` - Research document
- `millflow/src/data/operations.ts` - NEW: Operation suggestions
- `millflow/src/components/dag/SmartNodeCreator.tsx` - NEW: Component with bugs

## Action Items & Next Steps

1. **Fix the crash on Enter**: Debug why `handleSave()` crashes. Likely issue:
   - Check if `query.trim()` or `selectedSuggestion?.name` returns undefined when it shouldn't
   - Verify the data passed to `onSave` matches expected shape
   - Check if `insertNode` throws due to missing required fields

2. **Fix Tab navigation**: After expanding with Tab, focus should move through:
   - Input → Assignee dropdown → Due date input → back to Input
   - Currently Tab only expands but doesn't cycle through fields
   - Need to add `tabIndex` and focus management in expanded state

3. **Complete Phase 4**: Delete old `NodeCreator.tsx` once bugs are fixed

4. **Manual testing checklist** (from plan):
   - Press `o` on a node → SmartNodeCreator appears
   - Type "cnc" → CNC suggestion appears
   - Press Enter → node created with name "CNC", type "shop"
   - Press Tab → quick-add fields appear
   - Fill assignee/date → Enter → node created with all fields

## Other Notes

- The old `NodeCreator.tsx` at `millflow/src/components/dag/NodeCreator.tsx` still exists and should be deleted after bugs are fixed
- Build passes: `npm run build` works
- Dev server works: Returns 200 on localhost
- Gruvbox theme is used throughout - all colors should use `gruvbox-*` Tailwind classes
