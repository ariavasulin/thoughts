---
date: 2025-12-19T17:50:35-08:00
researcher: claude
git_commit: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
branch: main
repository: dwsERP
topic: "Millflow Phase 4 Implementation - Sidebar Data Source Fix Needed"
tags: [implementation, millflow, sheet-management, data-binding, bug-fix]
status: in_progress
last_updated: 2025-12-19
last_updated_by: claude
type: implementation_strategy
---

# Handoff: Millflow Phase 4 Complete, Sidebar Data Source Bug Remaining

## Task(s)
Implementing Phase 4 (Sheet Management) of the Millflow Full Data Manipulation plan.

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1: Blocker Management | **Completed** | Resolve/delete blockers with status auto-update |
| Phase 2: Note Management | **Completed** | Note editing with inline textarea and deletion |
| Phase 3: Multi-Select & Bulk Operations | **Completed** | Visual mode with `v` key, bulk delete, join nodes |
| Phase 4: Sheet Management | **Completed** | Create, rename, archive sheets |
| Phase 5: localStorage Persistence | **Planned** | Data survives page refresh |

**Current Blocker**: The `NodeDetailPanel` (sidebar) still reads from static mockData instead of the store's live `sheets` state. This causes node changes (notes, blockers, assignees) to not appear in the sidebar even though they're saved in the store.

## Critical References
- Implementation Plan: `thoughts/shared/plans/2025-12-19-millflow-full-data-manipulation.md`
- Previous handoff: `thoughts/shared/handoffs/general/2025-12-19_17-37-46_millflow-data-manipulation-phases-1-3.md`

## Recent changes

### Phase 4 Implementation (this session)
- `millflow/src/types/index.ts:27-28` - Added `archived?: boolean` and `archived_at?: string` to Sheet interface
- `millflow/src/store/appStore.ts:1086-1157` - Added `createSheet`, `updateSheet`, `archiveSheet`, `restoreSheet` methods
- `millflow/src/store/appStore.ts:30,69` - Updated `activeActionForm` type to include `'newsheet'`
- `millflow/src/hooks/useKeyboard.ts:42,369-374` - Added `c` key handler for creating sheets in job view
- `millflow/src/components/node/NewSheetForm.tsx` - Created new component for sheet creation modal
- `millflow/src/views/JobView.tsx:2,7,10,15,87-123` - Integrated NewSheetForm, switched to store's sheets

### Data Source Fix Attempt (partial)
Fixed JobView and appStore getters to read from store's `sheets` instead of static mockData:
- `millflow/src/views/JobView.tsx:15` - Now uses `sheets.filter(s => s.job_id === selectedJobId && !s.archived)`
- `millflow/src/store/appStore.ts:275,1175,1204` - Replaced `getSheetsForJob()` calls with inline `state.sheets.filter(...)`
- `millflow/src/store/appStore.ts:3` - Removed `getSheetsForJob` from mockData import

**NOT YET FIXED**: `NodeDetailPanel` still uses `getSheetById` from mockData.

## Learnings

1. **Root Cause of Sidebar Issue**: Components using helper functions from `mockData.ts` (`getSheetById`, `getSheetsForJob`, etc.) read from the static `sheets` array, NOT from the store's dynamic `sheets` state. This means store mutations are invisible to those components.

2. **Pattern for Fix**: Replace mockData helper calls with store state:
   - Instead of `getSheetById(id)`, use `useAppStore().sheets.find(s => s.id === id)`
   - Instead of `getSheetsForJob(jobId)`, use `sheets.filter(s => s.job_id === jobId && !s.archived)`

3. **Affected Component**: `millflow/src/components/node/NodeDetailPanel.tsx:18,34` - Uses `getSheetById` from mockData

## Artifacts
- New component: `millflow/src/components/node/NewSheetForm.tsx`
- Updated plan with Phase 4 automated verification checked: `thoughts/shared/plans/2025-12-19-millflow-full-data-manipulation.md:949-950`

## Action Items & Next Steps

### Immediate Fix Required
1. **Fix NodeDetailPanel data source** - Update `millflow/src/components/node/NodeDetailPanel.tsx`:
   - Line 18: Remove `getSheetById` from mockData import
   - Line 34: Change `getSheetById(selectedSheetId)` to get sheets from the store: `const sheets = useAppStore(state => state.sheets); const sheet = sheets.find(s => s.id === selectedSheetId);`
   - This will make the sidebar reflect live node changes (notes, blockers, assignees)

2. **Audit other components** - Search for any other usages of mockData helpers that should read from store:
   ```bash
   grep -r "getSheetById\|getNodeById" millflow/src/components/
   ```

### After Sidebar Fix
3. **Manual verification for Phase 4**:
   - Press `c` in job view to open new sheet form
   - Create sheet, verify navigation to new sheet with "Start" node
   - Test archive functionality (no UI yet - may need to add button)
   - Test undo works for sheet operations

4. **Proceed to Phase 5: localStorage Persistence** - See plan for details

## Other Notes
- Build and lint pass with current changes
- The `getSheetsForJob` function in mockData.ts was updated to filter archived sheets, but this is moot since components should read from store
- Phase 4 implementation is functionally complete; only the data binding bug remains
- Key files for this feature area:
  - Store: `millflow/src/store/appStore.ts`
  - Sidebar: `millflow/src/components/node/NodeDetailPanel.tsx`
  - Job view: `millflow/src/views/JobView.tsx`
  - New form: `millflow/src/components/node/NewSheetForm.tsx`
