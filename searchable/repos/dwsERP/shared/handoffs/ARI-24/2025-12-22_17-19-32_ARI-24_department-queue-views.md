---
date: 2025-12-22T17:19:32-08:00
researcher: claude
git_commit: b473de4c1b64aa43e69b5c1ebcbcf9c4abcfa6b9
branch: main
repository: dwsERP
topic: "Department Queue Views Implementation"
tags: [implementation, millflow, departments, queues, navigation]
status: in_progress
last_updated: 2025-12-22
last_updated_by: claude
type: implementation_strategy
---

# Handoff: ARI-24 Department Queue Views Implementation

## Task(s)

Implementing department-specific queue views for MillFlow scheduling system per the plan at `thoughts/shared/plans/2025-12-22-ARI-24-department-queue-views.md`.

**Status by Phase:**
- **Phase 1: Data Layer** - COMPLETED
  - Added `DepartmentId` type and `Department`/`QueueItem` interfaces to types
  - Updated `ViewType` to replace 'deliveries' with 'departments' | 'queue'
  - Created `millflow/src/data/departments.ts` with 12 department configurations
  - Created `millflow/src/lib/queue.ts` with queue extraction logic

- **Phase 2: Store Changes** - COMPLETED
  - Added `selectedDepartment` state property
  - Replaced `goToDeliveries` with `goToDepartments` and `goToQueue` actions
  - Updated `goBack`, `openSelected`, `getCurrentListLength` for new views

- **Phase 3: View Components** - COMPLETED
  - Created `DepartmentsView.tsx` - index of all 12 departments with item counts
  - Created `QueueView.tsx` - department queue with vim-style selection
  - Updated `App.tsx` to register new views and remove DeliveriesView import

- **Phase 4: Navigation Updates** - IN PROGRESS (not started)
  - `useKeyboard.ts` still references old `goToDeliveries` - build fails on this
  - `CommandPalette.tsx` needs department navigation commands
  - `HelpModal.tsx` needs department shortcut documentation
  - `ShortcutBar.tsx` needs shortcuts for departments/queue views

## Critical References

- Implementation Plan: `thoughts/shared/plans/2025-12-22-ARI-24-department-queue-views.md`
- Research Document: `thoughts/shared/research/2025-12-22-department-queue-views.md`

## Recent changes

- `millflow/src/types/index.ts:7-20` - Added `DepartmentId` type
- `millflow/src/types/index.ts:100-118` - Added `Department` and `QueueItem` interfaces
- `millflow/src/types/index.ts:150` - Updated `ViewType`
- `millflow/src/data/departments.ts` - New file with 12 department configurations
- `millflow/src/lib/queue.ts` - New file with `getQueueForDepartment` function
- `millflow/src/store/appStore.ts:2,5-6` - Added imports for DepartmentId, queue, departments
- `millflow/src/store/appStore.ts:32` - Added `selectedDepartment` to state interface
- `millflow/src/store/appStore.ts:76-77` - Added `goToDepartments`/`goToQueue` action types
- `millflow/src/store/appStore.ts:151` - Added `selectedDepartment: null` initial state
- `millflow/src/store/appStore.ts:351-381` - Added departments/queue cases to `openSelected`
- `millflow/src/store/appStore.ts:378-381` - Added departments/queue cases to `goBack`
- `millflow/src/store/appStore.ts:402-417` - Replaced `goToDeliveries` with new actions
- `millflow/src/store/appStore.ts:1466-1472` - Added departments/queue to `getCurrentListLength`
- `millflow/src/views/DepartmentsView.tsx` - New component
- `millflow/src/views/QueueView.tsx` - New component
- `millflow/src/App.tsx:10-11,29-32` - Updated imports and view switch

## Learnings

1. **cn function location**: The `cn` utility is in `@/lib/styles`, not `@/lib/utils` - the plan had the wrong import path
2. **Store imports**: Need to import both `getQueueForDepartment` and `DEPARTMENTS` at top of store to use in actions
3. **Department mapping**: Operations are mapped to departments by node.name (lowercase matching against department.operations array)
4. **Priority vs Status**: 9 departments are priority queues (sorted by deadline), 3 are status lists (alphabetical sort)

## Artifacts

- `millflow/src/types/index.ts` - Updated with DepartmentId, Department, QueueItem types
- `millflow/src/data/departments.ts` - Department configuration with 12 departments
- `millflow/src/lib/queue.ts` - Queue extraction and sorting logic
- `millflow/src/store/appStore.ts` - Updated with department navigation state and actions
- `millflow/src/views/DepartmentsView.tsx` - Departments index view component
- `millflow/src/views/QueueView.tsx` - Queue view component
- `millflow/src/App.tsx` - Updated view router

## Action Items & Next Steps

1. **Fix useKeyboard.ts** (blocking - build currently fails):
   - Replace `goToDeliveries` import with `goToDepartments`, `goToQueue`
   - Import `DEPARTMENTS`, `DEPARTMENT_BY_SHORTCUT` from `@/data/departments`
   - Replace `g d` handler with `g q` for departments index
   - Add handler for `g + department.shortcut` pattern to navigate to specific queues
   - See plan Phase 4, section 1 for exact code changes

2. **Update CommandPalette.tsx**:
   - Import `DEPARTMENTS` from `@/data/departments`
   - Import `Building2`, `ListTodo` from lucide-react
   - Add "Departments" navigation command with `g q` shortcut
   - Add all 12 department queue navigation commands
   - See plan Phase 4, section 2

3. **Update HelpModal.tsx**:
   - Replace `g d` entry with `g q` for Departments
   - Add entries for all 12 department shortcuts (g r, g c, g m, etc.)
   - See plan Phase 4, section 3 for complete list

4. **Update ShortcutBar.tsx**:
   - Add case for `view === 'departments'` returning appropriate shortcuts
   - Add case for `view === 'queue'` returning appropriate shortcuts
   - See plan Phase 4, section 4

5. **Run verification**:
   - `npm run build` should pass
   - `npm run lint` should pass
   - Manual testing per plan's testing strategy

## Other Notes

- The 12 departments match `factory-floor/src/data/mockShopData.ts` StationId exactly
- Department shortcuts: r(Drafting), c(CNC), m(Milling), e(Edge Banding), v(Veneer), f(Finishing), b(Bench), d(Delivery), i(Field Install), s(Staging), p(Punch List), o(Outsourced)
- `g d` was previously "go to deliveries", now it's "go to Delivery queue" (similar intent)
- Priority queues show Due Date and Priority columns; status lists do not
