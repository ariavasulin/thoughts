---
date: 2025-12-21T02:33:31-08:00
researcher: claude
git_commit: 3b9f9d4ad02d3656b6fcb8e12eec504d9384cd97
branch: ari-27-node-insert-cut-paste
repository: dwsERP
topic: "Node Cut/Paste and Single-Key Shortcuts - Sibling Paste Bug"
tags: [implementation, millflow, keyboard-shortcuts, dag, bug]
status: in_progress
last_updated: 2025-12-21
last_updated_by: claude
type: implementation_strategy
---

# Handoff: Ari-27 Node Cut/Paste Sibling Bug

## Task(s)

**Implementing plan**: `thoughts/shared/plans/2025-12-21-Ari-27-node-insert-cut-paste.md`

| Task | Status |
|------|--------|
| Phase 1: Add clipboard state and actions to store | Completed |
| Phase 2: Update keyboard mappings | Completed |
| Manual testing: `o`, `O`, `p`, `P` keys | Working |
| Manual testing: `x` cut | Working |
| Manual testing: `s` paste sibling / `S` insert sibling | **BUG** |

### Bug Description
The `s` (paste as sibling) and `S` (insert sibling) commands don't work correctly:
- **Expected**: Node should appear as a parallel sibling to the selected node (same parent, same children)
- **Actual**: Node appears "below" (as if inserted after) instead of as a sibling
- **Exception**: Works correctly if there's already a parallel branch at that level

## Critical References

1. Implementation plan: `/Users/ariasulin/git/dwsERP/thoughts/shared/plans/2025-12-21-Ari-27-node-insert-cut-paste.md`
2. DAG region building: `millflow/src/lib/dag.ts` - `buildRegions()` function determines how parallel branches are detected and rendered

## Recent changes

- `millflow/src/store/appStore.ts:37-41` - Added clipboard state interface
- `millflow/src/store/appStore.ts:90-92` - Added cutNode, pasteNode, clearClipboard action types
- `millflow/src/store/appStore.ts:153-157` - Initialized clipboard state
- `millflow/src/store/appStore.ts:880-1071` - Implemented cutNode, pasteNode, clearClipboard actions
- `millflow/src/hooks/useKeyboard.ts:60-62` - Imported clipboard operations
- `millflow/src/hooks/useKeyboard.ts:245-249` - Removed 'o' from sequence starters (now single key)
- `millflow/src/hooks/useKeyboard.ts:462-516` - Added new single-key handlers (o, O, S, x, p, P, s)
- `millflow/src/hooks/useKeyboard.ts:535-538` - Added clipboard to useCallback dependencies

## Learnings

### DAG Region Detection
The `buildRegions()` function in `dag.ts` detects splits by checking `node.children.length > 1`. When a node has 2+ children, it creates a `BranchRegion` with:
- `splitNode`: The parent that has multiple children
- `branches`: Array of branch paths (each child and its descendants until convergence)
- `convergenceNode`: Where branches merge back

### Sibling Insertion Logic (pasteNode 'right')
Located at `appStore.ts:1000-1049`:
1. If target has no parents (root), falls back to insert-after
2. Otherwise:
   - newNode gets same parents as target
   - newNode gets same children as target
   - Parent's children array is updated via splice to add newNode after target
   - Children's parents array gets newNode added

### The Mystery
Static analysis shows the logic should work:
- Parent's children should become `[B, X]` after inserting X as sibling to B
- `buildRegions()` should detect the split and create a BranchRegion
- `BranchRegionView` should render branches side-by-side

But the user reports it only works if there's already a parallel branch. This suggests either:
1. The DAG structure isn't being updated correctly in some edge case
2. The region detection isn't recognizing new splits
3. React's useMemo on `buildRegions(sheet.nodes)` isn't invalidating

## Artifacts

- Plan file: `/Users/ariasulin/git/dwsERP/thoughts/shared/plans/2025-12-21-Ari-27-node-insert-cut-paste.md`
- Modified store: `millflow/src/store/appStore.ts`
- Modified keyboard hook: `millflow/src/hooks/useKeyboard.ts`
- Commit: `3b9f9d4` - "feat(millflow): add node cut/paste and single-key shortcuts"

## Action Items & Next Steps

1. **Debug the sibling paste issue**:
   - Add console.log in `pasteNode` 'right' branch to verify parent.children is updated
   - Add console.log in `buildRegions` to see if it detects the new split
   - Check browser React DevTools to see Zustand store state after paste

2. **Compare with insertNode**:
   - `insertNode` at `appStore.ts:628-660` has similar 'right' logic
   - Check if there's a difference causing the same behavior

3. **Potential fixes to investigate**:
   - Is `target` stale when we use it? (It's captured before modifications)
   - Is there a race condition with React rendering?
   - Does the `useMemo` dependency `[sheet.nodes]` properly detect the change?

4. **After fix**: Complete manual verification checklist in plan

## Other Notes

### Key Files for DAG Rendering
- `millflow/src/lib/dag.ts` - All DAG algorithms (linearize, buildRegions, navigation)
- `millflow/src/components/dag/DAGRenderer.tsx` - Main renderer, calls buildRegions
- `millflow/src/components/dag/BranchRegionView.tsx` - Renders parallel branches side-by-side

### Keyboard Shortcut Summary
| Key | Action |
|-----|--------|
| `o` | Insert node after |
| `O` | Insert node before |
| `S` | Insert sibling right (buggy) |
| `x` | Cut node |
| `p` | Paste after |
| `P` | Paste before |
| `s` | Paste sibling right (buggy) |

### Test Scenario
1. Navigate to any sheet with a linear DAG (A -> B -> C)
2. Select node B
3. Cut another node with `x`
4. Press `s` to paste as sibling
5. **Expected**: New node appears next to B (parallel branch)
6. **Actual**: New node appears below B (in series)
