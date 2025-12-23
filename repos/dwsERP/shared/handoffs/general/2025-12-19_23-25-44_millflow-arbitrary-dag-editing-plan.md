---
date: 2025-12-19T23:25:44-08:00
researcher: Claude
git_commit: 201f79e68db6ef380d44c9b4d64ee7e34b507206
branch: main
repository: dwsERP
topic: "Arbitrary DAG Editing with Marks, Connect, and Move"
tags: [implementation, millflow, dag, keyboard-shortcuts, marks, edges]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: Arbitrary DAG Editing Implementation Plan

## Task(s)

| Task | Status |
|------|--------|
| Research current split node/DAG architecture | Completed |
| Identify limitations of current keyboard operations | Completed |
| Design new keyboard shortcuts for arbitrary DAG building | Completed |
| Write 5-phase implementation plan | Completed |

**Plan created but not yet implemented.** The next agent should implement the plan starting with Phase 1.

## Critical References

1. **Implementation Plan**: `thoughts/shared/plans/2025-12-19-millflow-arbitrary-dag-editing.md` - The comprehensive 5-phase plan to implement
2. **Research Document**: `thoughts/shared/research/2025-12-19-split-node-dag-architecture.md` - Analysis of current limitations

## Recent changes

No code changes made - this session was research and planning only.

## Learnings

### Current System Limitations
- `splitNode` (`millflow/src/store/appStore.ts:678-727`) requires children and forces immediate reconvergence
- `joinNodes` (`millflow/src/store/appStore.ts:729-758`) is destructive - replaces all children arrays
- `insertNode` (`millflow/src/store/appStore.ts:514-596`) only creates linear connections
- Edges are implicit (children/parents arrays), not explicit objects

### Key Architecture Insights
- DAG levels are computed at render time via `buildLevels()` in `millflow/src/lib/dag.ts:67-111`
- Parallel detection is automatic: if `unvisited.length > 1` at same BFS depth = parallel
- No edge validation exists - system can create cycles or orphans

### Keyboard Handler Structure
- `millflow/src/hooks/useKeyboard.ts` uses sequence matching for multi-key commands
- Pending sequences tracked in `pendingKeySequence` state
- ShortcutBar (`millflow/src/components/ShortcutBar.tsx`) is context-aware based on view/mode

## Artifacts

- `thoughts/shared/plans/2025-12-19-millflow-arbitrary-dag-editing.md` - Full implementation plan (5 phases)
- `thoughts/shared/research/2025-12-19-split-node-dag-architecture.md` - Research on current DAG architecture

## Action Items & Next Steps

1. **Implement Phase 1**: State Foundation
   - Add `nodeMarks` Map and `pendingMarkAction` to AppState
   - Implement `setMark`, `jumpToMark`, `connectNodes`, `disconnectNodes`
   - Implement `moveNodeUp`, `moveNodeDown`, `moveNodeAfter`, `moveNodeBefore`
   - Fix `splitNode` to work on leaf nodes (create sibling instead of requiring children)

2. **Implement Phase 2**: Keyboard Handler Integration
   - Add mark sequences: `m {a-z}`, `' {a-z}`
   - Add edge sequences: `c {a-z}`, `C {a-z}`, `x {a-z}`
   - Add move sequences: `M k`, `M j`, `> {a-z}`, `< {a-z}`

3. **Implement Phase 3**: UI Updates
   - Update ShortcutBar with new shortcuts
   - Update HelpModal with new "Marks & Connect" category

4. **Implement Phase 4**: Visual mark indicators on DAGNode

5. **Implement Phase 5**: Edge case handling (cycle prevention, mark cleanup on delete)

## Other Notes

### New Keyboard Shortcuts (to be implemented)
| Key | Action |
|-----|--------|
| `m {a-z}` | Set mark on current node |
| `' {a-z}` | Jump to marked node |
| `c {a-z}` | Connect FROM mark TO current |
| `C {a-z}` | Connect FROM current TO mark |
| `x {a-z}` | Disconnect edge with mark |
| `M k/↑` | Move node up a level |
| `M j/↓` | Move node down a level |
| `> {a-z}` | Move node after mark |
| `< {a-z}` | Move node before mark |

### Key Files to Modify
- `millflow/src/store/appStore.ts` - Add marks state and new actions
- `millflow/src/hooks/useKeyboard.ts` - Add key sequence handlers
- `millflow/src/components/ShortcutBar.tsx` - Add context shortcuts
- `millflow/src/components/HelpModal.tsx` - Add new categories
- `millflow/src/components/dag/DAGNode.tsx` - Add mark indicator badges

### User Context
The user is building a keyboard-first, Superhuman-style PM dashboard for millwork production. Power users should be able to build any DAG structure entirely via keyboard. See `thoughts/searchable/shared/ariav/wiki/Millflow.md` for product vision.
