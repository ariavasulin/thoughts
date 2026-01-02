# Department Queue Views Implementation Plan

## Overview

Implement department-specific queue views that show each department's list of operations. Navigation via Command-K search and vim-style keyboard shortcuts (`g c` for CNC, `g f` for Finishing, etc.).

**Two view types:**
- **Priority Queues** (9): Actual work centers where foremen need "what's next" — sorted by deadline
- **Status Lists** (3): Milestone tracking views — simple filtered lists, no priority ordering

**Departments index view**: Lists all 12 departments with item counts, similar to Jobs view. Navigate with `g q`, drill down into any department.

This matches the 12 departments from the factory-floor prototype exactly.

## Current State Analysis

**What exists:**
- View system with 5 views: dashboard, jobs, job, sheet, deliveries (`types/index.ts:115`)
- Command palette with dynamic command building (`CommandPalette.tsx:30-121`)
- Multi-key keyboard sequences (`useKeyboard.ts:163-205`)
- Operation definitions with type categorization (`operations.ts:11-43`)
- `StationTableNode` reference UI in factory-floor (`StationTableNode.tsx:14-107`)

**What's missing:**
- No formal Department entity or type
- No Operation → Department mapping
- No queue view or queue data extraction logic
- No department navigation shortcuts

### Key Discoveries:
- Department mapping should be by **operation type**, not assignee (`operations.ts:13-42`)
- Current `goToDeliveries` action (`appStore.ts:402-408`) will be replaced with `goToQueue`
- Queue view is a **display layer** - prioritization comes from solver (deadline-based for MVP)

## Desired End State

After implementation:
1. Users can navigate to Departments index via `g q` to see all departments with item counts
2. Users can navigate directly to any department via `g [key]` shortcuts or Command-K
3. Priority queues (9) show operations sorted by deadline with sheet/job context
4. Status lists (3) show simple filtered lists without priority ordering
5. Clicking any item opens the sheet view with that node selected
6. The deliveries view is replaced by department-specific views

### Verification:
- `npm run build` passes with no type errors
- All 12 department views accessible via keyboard shortcuts
- Command palette shows all department navigation options
- Queue items link to correct sheets/nodes

## What We're NOT Doing

Per CLAUDE.md direction:
- **Assignment tracking** — Foremen handle this on the floor
- **Blocker management** — System recalculates when inputs change instead

Not yet implemented (future phases):
- **Solver integration** — Using deadline-based sorting as temporary stand-in
- **Capacity modeling** — No hours/day tracking per department
- **Time estimate defaults** — Not implementing operation → duration mapping
- **PMDB integration** — Using mock data only

## Implementation Approach

Four phases:
1. **Data Layer** - Types, department mapping, queue extraction logic
2. **Store Changes** - Navigation state and actions
3. **QueueView Component** - Simple queue table with vim-style selection
4. **Navigation Updates** - Keyboard shortcuts and command palette

---

## Phase 1: Data Layer

### Overview
Add the Department type, operation-to-department mapping, and queue extraction utility.

### Changes Required:

#### 1. Add Department Types
**File**: `millflow/src/types/index.ts`
**Changes**: Add Department type and QueueItem interface

```typescript
// Add after line 4 (after NodeType)
// Matches factory-floor/src/data/mockShopData.ts StationId exactly
export type DepartmentId =
  | 'drafting'
  | 'cnc'
  | 'veneer'
  | 'milling'
  | 'edgeBanding'
  | 'finishing'
  | 'bench'
  | 'outsourced'
  | 'staging'
  | 'delivery'
  | 'fieldInstall'
  | 'punchList';

// Add after Person interface (around line 84)
export interface Department {
  id: DepartmentId;
  name: string;
  shortcut: string;  // Single letter for keyboard navigation
  operations: string[];  // Operation IDs from operations.ts
  isPriorityQueue: boolean;  // true = sorted by deadline, false = simple list
}

export interface QueueItem {
  id: string;  // Unique identifier for the queue item
  nodeId: string;
  nodeName: string;
  sheetId: string;
  sheetName: string;
  jobId: string;
  jobName: string;
  status: NodeStatus;
  deadline: string | null;  // From node.due_date or node.window?.latest
}
```

#### 2. Update ViewType
**File**: `millflow/src/types/index.ts`
**Changes**: Replace 'deliveries' with 'departments' and 'queue' in ViewType

```typescript
// Line 115: Change from
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'deliveries';
// To
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'departments' | 'queue';
```

#### 3. Create Department Configuration
**File**: `millflow/src/data/departments.ts` (new file)
**Changes**: Define departments with operation mappings

```typescript
import type { Department, DepartmentId } from '@/types';

// Matches factory-floor/src/data/mockShopData.ts exactly
// 9 priority queues + 3 status lists
export const DEPARTMENTS: Department[] = [
  // Priority Queues (actual work centers)
  {
    id: 'drafting',
    name: 'Drafting',
    shortcut: 'r',  // 'd' taken by delivery
    operations: ['drafting'],
    isPriorityQueue: true,
  },
  {
    id: 'cnc',
    name: 'CNC',
    shortcut: 'c',
    operations: ['cnc'],
    isPriorityQueue: true,
  },
  {
    id: 'milling',
    name: 'Milling',
    shortcut: 'm',
    operations: ['milling'],
    isPriorityQueue: true,
  },
  {
    id: 'edgeBanding',
    name: 'Edge Banding',
    shortcut: 'e',
    operations: ['edge-banding', 'edgebanding'],
    isPriorityQueue: true,
  },
  {
    id: 'veneer',
    name: 'Veneer',
    shortcut: 'v',
    operations: ['veneer'],
    isPriorityQueue: true,
  },
  {
    id: 'finishing',
    name: 'Finishing',
    shortcut: 'f',
    operations: ['finishing'],
    isPriorityQueue: true,
  },
  {
    id: 'bench',
    name: 'Bench',
    shortcut: 'b',
    operations: ['bench'],
    isPriorityQueue: true,
  },
  {
    id: 'delivery',
    name: 'Delivery',
    shortcut: 'd',
    operations: ['delivery'],
    isPriorityQueue: true,
  },
  {
    id: 'fieldInstall',
    name: 'Field Install',
    shortcut: 'i',
    operations: ['field-install', 'fieldinstall'],
    isPriorityQueue: true,
  },
  // Status Lists (milestone tracking, not priority queues)
  {
    id: 'staging',
    name: 'Staging',
    shortcut: 's',
    operations: ['staging'],
    isPriorityQueue: false,
  },
  {
    id: 'punchList',
    name: 'Punch List',
    shortcut: 'p',
    operations: ['punch-list', 'punchlist'],
    isPriorityQueue: false,
  },
  {
    id: 'outsourced',
    name: 'Outsourced',
    shortcut: 'o',
    operations: ['outsourced'],
    isPriorityQueue: false,
  },
];

export const DEPARTMENT_BY_ID: Record<DepartmentId, Department> =
  Object.fromEntries(DEPARTMENTS.map(d => [d.id, d])) as Record<DepartmentId, Department>;

export const DEPARTMENT_BY_SHORTCUT: Record<string, Department> =
  Object.fromEntries(DEPARTMENTS.map(d => [d.shortcut, d]));

// Map operation name (node.name lowercase) to department
export function getDepartmentForOperation(operationName: string): Department | null {
  const normalized = operationName.toLowerCase();
  return DEPARTMENTS.find(d =>
    d.operations.some(op => normalized.includes(op.replace('-', ' ')) || normalized === op)
  ) || null;
}
```

#### 4. Create Queue Utility
**File**: `millflow/src/lib/queue.ts` (new file)
**Changes**: Queue extraction and sorting logic

```typescript
import type { Sheet, Job, QueueItem, DepartmentId } from '@/types';
import { DEPARTMENT_BY_ID, getDepartmentForOperation } from '@/data/departments';

export function getQueueForDepartment(
  departmentId: DepartmentId,
  sheets: Sheet[],
  jobs: Job[]
): QueueItem[] {
  const department = DEPARTMENT_BY_ID[departmentId];
  if (!department) return [];

  const items: QueueItem[] = [];
  const jobMap = new Map(jobs.map(j => [j.id, j]));

  for (const sheet of sheets) {
    if (sheet.archived) continue;

    const job = jobMap.get(sheet.job_id);
    if (!job) continue;

    for (const node of sheet.nodes) {
      // Skip completed or skipped nodes
      if (node.status === 'done' || node.status === 'skipped') continue;

      // Check if this node's operation belongs to this department
      const nodeDept = getDepartmentForOperation(node.name);
      if (!nodeDept || nodeDept.id !== departmentId) continue;

      // Extract deadline from node
      const deadline = node.due_date || node.window?.latest || null;

      items.push({
        id: `${sheet.id}-${node.id}`,
        nodeId: node.id,
        nodeName: node.name,
        sheetId: sheet.id,
        sheetName: sheet.name,
        jobId: job.id,
        jobName: job.name,
        status: node.status,
        deadline,
      });
    }
  }

  // Sort based on department type
  if (department.isPriorityQueue) {
    // Priority queues: sort by deadline (nulls last), then job, then sheet
    return items.sort((a, b) => {
      if (a.deadline && !b.deadline) return -1;
      if (!a.deadline && b.deadline) return 1;
      if (a.deadline && b.deadline) {
        const cmp = a.deadline.localeCompare(b.deadline);
        if (cmp !== 0) return cmp;
      }
      const jobCmp = a.jobName.localeCompare(b.jobName);
      if (jobCmp !== 0) return jobCmp;
      return a.sheetName.localeCompare(b.sheetName);
    });
  } else {
    // Status lists: simple alphabetical sort by job, then sheet
    return items.sort((a, b) => {
      const jobCmp = a.jobName.localeCompare(b.jobName);
      if (jobCmp !== 0) return jobCmp;
      return a.sheetName.localeCompare(b.sheetName);
    });
  }
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] New files exist: `src/data/departments.ts`, `src/lib/queue.ts`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Import and call `getQueueForDepartment('cnc', sheets, jobs)` in browser console - returns expected array

**Implementation Note**: After completing this phase, pause for review before proceeding.

---

## Phase 2: Store Changes

### Overview
Update the Zustand store to support queue navigation with department selection.

### Changes Required:

#### 1. Add State Properties
**File**: `millflow/src/store/appStore.ts`
**Changes**: Add selectedDepartment to state

```typescript
// Around line 31 (after selectedIndex)
selectedDepartment: DepartmentId | null;
```

Add import at top:
```typescript
import type { ViewType, Job, Sheet, Person, DashboardState, Node, Note, DepartmentId } from '@/types';
```

#### 2. Add Initial State
**File**: `millflow/src/store/appStore.ts`
**Changes**: Initialize selectedDepartment

```typescript
// Around line 150 (in initial state)
selectedDepartment: null,
```

#### 3. Replace goToDeliveries with goToDepartments and goToQueue
**File**: `millflow/src/store/appStore.ts`
**Changes**: Add two navigation actions

In interface (around line 76):
```typescript
// Replace: goToDeliveries: () => void;
// With:
goToDepartments: () => void;
goToQueue: (departmentId: DepartmentId) => void;
```

In implementation (around line 402):
```typescript
// Replace goToDeliveries implementation with:
goToDepartments: () => {
  set({
    view: 'departments',
    selectedDepartment: null,
    selectedIndex: 0,
    showNodeDetail: false,
  });
},
goToQueue: (departmentId) => {
  set({
    view: 'queue',
    selectedDepartment: departmentId,
    selectedIndex: 0,
    showNodeDetail: false,
  });
},
```

#### 4. Update goBack for Departments and Queue Views
**File**: `millflow/src/store/appStore.ts`
**Changes**: Add departments and queue cases to goBack

```typescript
// In goBack function, add after line 378 (after jobs case):
} else if (view === 'departments') {
  set({ view: 'dashboard', selectedIndex: 0 });
} else if (view === 'queue') {
  set({ view: 'departments', selectedDepartment: null, selectedIndex: 0 });
```

#### 5. Update openSelected for Departments and Queue Views
**File**: `millflow/src/store/appStore.ts`
**Changes**: Add departments and queue drill-down behavior

```typescript
// Add after the deliveries case (around line 351), before the closing brace:
} else if (view === 'queue' && state.selectedDepartment) {
  // Get queue items for current department
  const { getQueueForDepartment } = await import('@/lib/queue');
  const queueItems = getQueueForDepartment(state.selectedDepartment, state.sheets, state.jobs);
  const item = queueItems[selectedIndex];

  if (item) {
    // Navigate to sheet with node selected
    const sheet = state.sheets.find(s => s.id === item.sheetId);
    if (sheet) {
      const linearized = linearizeNodes(sheet.nodes);
      const nodeIdx = linearized.findIndex(n => n.id === item.nodeId);

      set({
        view: 'sheet',
        selectedJobId: item.jobId,
        selectedSheetId: item.sheetId,
        selectedNodeId: item.nodeId,
        selectedIndex: nodeIdx >= 0 ? nodeIdx : 0,
      });
    }
  }
}
```

Note: Since this uses dynamic import, make openSelected async or refactor to import at top.

Better approach - import at top and use synchronously:
```typescript
// Add import at top of file
import { getQueueForDepartment } from '@/lib/queue';
import { DEPARTMENTS } from '@/data/departments';

// Then in openSelected, add these cases:

// Departments view -> drill into selected department's queue
} else if (view === 'departments') {
  const dept = DEPARTMENTS[selectedIndex];
  if (dept) {
    set({
      view: 'queue',
      selectedDepartment: dept.id,
      selectedIndex: 0,
    });
  }

// Queue view -> drill into selected item's sheet
} else if (view === 'queue' && state.selectedDepartment) {
  const queueItems = getQueueForDepartment(state.selectedDepartment, state.sheets, state.jobs);
  const item = queueItems[selectedIndex];

  if (item) {
    const sheet = state.sheets.find(s => s.id === item.sheetId);
    if (sheet) {
      const linearized = linearizeNodes(sheet.nodes);
      const nodeIdx = linearized.findIndex(n => n.id === item.nodeId);

      set({
        view: 'sheet',
        selectedJobId: item.jobId,
        selectedSheetId: item.sheetId,
        selectedNodeId: item.nodeId,
        selectedIndex: nodeIdx >= 0 ? nodeIdx : 0,
      });
    }
  }
}
```

#### 6. Update getCurrentListLength for Departments and Queue Views
**File**: `millflow/src/store/appStore.ts`
**Changes**: Add departments and queue cases

```typescript
// In getCurrentListLength, add before the return 0 (around line 1466):
if (view === 'departments') {
  return DEPARTMENTS.length;  // Always 12
}
if (view === 'queue' && state.selectedDepartment) {
  const queueItems = getQueueForDepartment(state.selectedDepartment, state.sheets, state.jobs);
  return queueItems.length;
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] Store exports `goToDepartments` and `goToQueue` actions
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Call `useAppStore.getState().goToDepartments()` in console - view changes to 'departments'
- [ ] Call `useAppStore.getState().goToQueue('cnc')` in console - view changes to 'queue'

**Implementation Note**: After completing this phase, pause for review before proceeding.

---

## Phase 3: View Components

### Overview
Create the DepartmentsView (index of all departments) and QueueView (individual department queue) components with vim-style navigation.

### Changes Required:

#### 1. Create DepartmentsView Component
**File**: `millflow/src/views/DepartmentsView.tsx` (new file)
**Changes**: Index view showing all departments with item counts

```typescript
import { memo, useMemo, useEffect, useRef } from 'react';
import { Building2, ListTodo, Package } from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { DEPARTMENTS } from '@/data/departments';
import { getQueueForDepartment } from '@/lib/queue';
import { cn } from '@/lib/utils';

export const DepartmentsView = memo(function DepartmentsView() {
  const { selectedIndex, sheets, jobs, openSelected } = useAppStore();
  const containerRef = useRef<HTMLDivElement>(null);

  // Calculate item counts for each department
  const departmentCounts = useMemo(() => {
    return DEPARTMENTS.map(dept => ({
      ...dept,
      count: getQueueForDepartment(dept.id, sheets, jobs).length,
    }));
  }, [sheets, jobs]);

  // Scroll selected item into view
  useEffect(() => {
    const selected = containerRef.current?.querySelector('[data-selected="true"]');
    selected?.scrollIntoView({ block: 'nearest', behavior: 'smooth' });
  }, [selectedIndex]);

  return (
    <div className="flex flex-col h-full">
      {/* Header */}
      <div className="flex items-center justify-between px-6 py-4 border-b border-gruvbox-bg-2">
        <div className="flex items-center gap-3">
          <Building2 size={20} className="text-gruvbox-blue-bright" />
          <h1 className="text-xl font-semibold text-gruvbox-fg-0">Departments</h1>
          <span className="text-sm font-mono text-gruvbox-fg-4 bg-gruvbox-bg-1 px-2 py-0.5 rounded">
            {DEPARTMENTS.length} departments
          </span>
        </div>
        <div className="text-sm text-gruvbox-fg-4">
          <kbd className="px-1.5 py-0.5 bg-gruvbox-bg-1 rounded text-gruvbox-fg-2">g q</kbd>
        </div>
      </div>

      {/* Department List */}
      <div ref={containerRef} className="flex-1 overflow-y-auto">
        <div className="divide-y divide-gruvbox-bg-2">
          {departmentCounts.map((dept, idx) => {
            const isSelected = selectedIndex === idx;
            const Icon = dept.isPriorityQueue ? ListTodo : Package;

            return (
              <button
                key={dept.id}
                data-selected={isSelected}
                onClick={openSelected}
                className={cn(
                  'w-full flex items-center gap-4 px-6 py-4 text-left transition-colors',
                  'hover:bg-gruvbox-bg-soft',
                  isSelected && 'bg-gruvbox-bg-1 ring-1 ring-gruvbox-blue-bright'
                )}
              >
                {/* Selection indicator */}
                <span className={cn(
                  'text-gruvbox-blue-bright shrink-0 w-4',
                  isSelected ? 'opacity-100' : 'opacity-0'
                )}>
                  {'>'}
                </span>

                {/* Icon */}
                <Icon size={18} className={cn(
                  dept.isPriorityQueue ? 'text-gruvbox-blue' : 'text-gruvbox-fg-4'
                )} />

                {/* Department name */}
                <div className="flex-1 min-w-0">
                  <div className={cn(
                    'font-medium',
                    isSelected ? 'text-gruvbox-fg-0' : 'text-gruvbox-fg'
                  )}>
                    {dept.name}
                  </div>
                  <div className="text-xs text-gruvbox-fg-4">
                    {dept.isPriorityQueue ? 'Priority Queue' : 'Status List'}
                  </div>
                </div>

                {/* Item count */}
                <span className={cn(
                  'font-mono text-sm px-2 py-0.5 rounded',
                  dept.count > 0
                    ? 'bg-gruvbox-blue/20 text-gruvbox-blue-bright'
                    : 'bg-gruvbox-bg-2 text-gruvbox-fg-4'
                )}>
                  {dept.count}
                </span>

                {/* Shortcut hint */}
                <kbd className="text-xs px-1.5 py-0.5 bg-gruvbox-bg-2 rounded text-gruvbox-fg-4">
                  g {dept.shortcut}
                </kbd>
              </button>
            );
          })}
        </div>
      </div>
    </div>
  );
});

export default DepartmentsView;
```

#### 2. Create QueueView Component
**File**: `millflow/src/views/QueueView.tsx` (new file)
**Changes**: Full component implementation

```typescript
import { memo, useMemo, useEffect, useRef } from 'react';
import { ListTodo } from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { getQueueForDepartment } from '@/lib/queue';
import { DEPARTMENT_BY_ID } from '@/data/departments';
import { cn } from '@/lib/utils';

export const QueueView = memo(function QueueView() {
  const {
    selectedDepartment,
    selectedIndex,
    sheets,
    jobs,
    openSelected
  } = useAppStore();

  const containerRef = useRef<HTMLDivElement>(null);

  const department = selectedDepartment ? DEPARTMENT_BY_ID[selectedDepartment] : null;

  const queueItems = useMemo(() => {
    if (!selectedDepartment) return [];
    return getQueueForDepartment(selectedDepartment, sheets, jobs);
  }, [selectedDepartment, sheets, jobs]);

  // Scroll selected item into view
  useEffect(() => {
    const selected = containerRef.current?.querySelector('[data-selected="true"]');
    selected?.scrollIntoView({ block: 'nearest', behavior: 'smooth' });
  }, [selectedIndex]);

  if (!department) {
    return (
      <div className="flex items-center justify-center h-full text-gruvbox-fg-4">
        No department selected
      </div>
    );
  }

  return (
    <div className="flex flex-col h-full">
      {/* Header */}
      <div className="flex items-center justify-between px-6 py-4 border-b border-gruvbox-bg-2">
        <div className="flex items-center gap-3">
          <ListTodo size={20} className="text-gruvbox-blue-bright" />
          <h1 className="text-xl font-semibold text-gruvbox-fg-0">
            {department.name} {department.isPriorityQueue ? 'Queue' : ''}
          </h1>
          <span className="text-sm font-mono text-gruvbox-fg-4 bg-gruvbox-bg-1 px-2 py-0.5 rounded">
            {queueItems.length} items
          </span>
          {!department.isPriorityQueue && (
            <span className="text-xs text-gruvbox-fg-4 italic">Status list</span>
          )}
        </div>
        <div className="text-sm text-gruvbox-fg-4">
          <kbd className="px-1.5 py-0.5 bg-gruvbox-bg-1 rounded text-gruvbox-fg-2">g {department.shortcut}</kbd>
        </div>
      </div>

      {/* Table Header */}
      <div className={cn(
        'grid gap-4 px-6 py-2 bg-gruvbox-bg-hard border-b border-gruvbox-bg-2 text-xs font-semibold text-gruvbox-fg-4 uppercase tracking-wide',
        department.isPriorityQueue ? 'grid-cols-12' : 'grid-cols-10'
      )}>
        <div className="col-span-3">Job / Sheet</div>
        <div className="col-span-3">Operation</div>
        <div className="col-span-2">Status</div>
        {department.isPriorityQueue && <div className="col-span-2">Due Date</div>}
        {department.isPriorityQueue && <div className="col-span-2 text-right">Priority</div>}
      </div>

      {/* Table Body */}
      <div ref={containerRef} className="flex-1 overflow-y-auto">
        {queueItems.length === 0 ? (
          <div className="flex flex-col items-center justify-center h-48 text-gruvbox-fg-4">
            <ListTodo size={32} className="mb-2 opacity-50" />
            <p>No active items in queue</p>
          </div>
        ) : (
          <div className="divide-y divide-gruvbox-bg-2">
            {queueItems.map((item, idx) => {
              const isSelected = selectedIndex === idx;

              return (
                <button
                  key={item.id}
                  data-selected={isSelected}
                  onClick={openSelected}
                  className={cn(
                    'w-full grid gap-4 px-6 py-3 text-left transition-colors',
                    department.isPriorityQueue ? 'grid-cols-12' : 'grid-cols-10',
                    'hover:bg-gruvbox-bg-soft',
                    isSelected && 'bg-gruvbox-bg-1 ring-1 ring-gruvbox-blue-bright'
                  )}
                >
                  {/* Job / Sheet */}
                  <div className="col-span-3 min-w-0">
                    <div className="flex items-center gap-2">
                      {isSelected && (
                        <span className="text-gruvbox-blue-bright shrink-0">{'>'}</span>
                      )}
                      <div className="min-w-0">
                        <div className={cn(
                          'font-medium truncate',
                          isSelected ? 'text-gruvbox-fg-0' : 'text-gruvbox-fg'
                        )}>
                          {item.jobName}
                        </div>
                        <div className="text-sm text-gruvbox-fg-4 truncate">
                          {item.sheetName}
                        </div>
                      </div>
                    </div>
                  </div>

                  {/* Operation */}
                  <div className="col-span-3 flex items-center">
                    <span className={cn(
                      'font-mono text-sm',
                      isSelected ? 'text-gruvbox-fg-0' : 'text-gruvbox-fg-2'
                    )}>
                      {item.nodeName}
                    </span>
                  </div>

                  {/* Status */}
                  <div className="col-span-2 flex items-center">
                    <StatusBadge status={item.status} />
                  </div>

                  {/* Due Date - only for priority queues */}
                  {department.isPriorityQueue && (
                    <div className="col-span-2 flex items-center text-sm">
                      {item.deadline ? (
                        <span className={cn(
                          'font-mono',
                          isUrgent(item.deadline) ? 'text-gruvbox-red-bright font-bold' : 'text-gruvbox-fg-4'
                        )}>
                          {formatDate(item.deadline)}
                        </span>
                      ) : (
                        <span className="text-gruvbox-fg-4 italic">—</span>
                      )}
                    </div>
                  )}

                  {/* Priority (rank in queue) - only for priority queues */}
                  {department.isPriorityQueue && (
                    <div className="col-span-2 flex items-center justify-end">
                      <span className="font-mono text-sm text-gruvbox-fg-4">
                        #{idx + 1}
                      </span>
                    </div>
                  )}
                </button>
              );
            })}
          </div>
        )}
      </div>
    </div>
  );
});

// Helper components
function StatusBadge({ status }: { status: string }) {
  const config: Record<string, { bg: string; text: string; label: string }> = {
    not_started: { bg: 'bg-gruvbox-bg-2', text: 'text-gruvbox-fg-4', label: 'Not Started' },
    ready: { bg: 'bg-gruvbox-blue/20', text: 'text-gruvbox-blue-bright', label: 'Ready' },
    in_progress: { bg: 'bg-gruvbox-yellow/20', text: 'text-gruvbox-yellow-bright', label: 'In Progress' },
  };

  const c = config[status] || config.not_started;

  return (
    <span className={cn('text-xs px-2 py-0.5 rounded', c.bg, c.text)}>
      {c.label}
    </span>
  );
}

function formatDate(dateStr: string): string {
  const date = new Date(dateStr);
  return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}

function isUrgent(dateStr: string): boolean {
  const date = new Date(dateStr);
  const today = new Date();
  const diffDays = Math.ceil((date.getTime() - today.getTime()) / (1000 * 60 * 60 * 24));
  return diffDays <= 3;
}

export default QueueView;
```

#### 3. Register Views in App.tsx
**File**: `millflow/src/App.tsx`
**Changes**: Add departments and queue cases to view router

Add imports:
```typescript
import DepartmentsView from '@/views/DepartmentsView';
import QueueView from '@/views/QueueView';
```

Add cases to switch (around line 30):
```typescript
case 'departments':
  return <DepartmentsView />;
case 'queue':
  return <QueueView />;
```

Remove deliveries case and import if it exists.

#### 4. Delete DeliveriesView (if separate file exists)
**File**: `millflow/src/views/DeliveriesView.tsx`
**Changes**: Delete file (deliveries view is replaced by queue)

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] DepartmentsView file exists: `src/views/DepartmentsView.tsx`
- [ ] QueueView file exists: `src/views/QueueView.tsx`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Navigate to departments view - shows all 12 departments with item counts
- [ ] j/k moves selection up/down in departments list
- [ ] Enter on department drills into that department's queue
- [ ] Navigate to queue view - displays correctly with header and table
- [ ] Enter on queue item navigates to sheet view with correct node selected
- [ ] Esc from queue returns to departments view

**Implementation Note**: After completing this phase, pause for review before proceeding.

---

## Phase 4: Navigation Updates

### Overview
Update keyboard shortcuts and command palette for department navigation.

### Changes Required:

#### 1. Update Keyboard Shortcuts
**File**: `millflow/src/hooks/useKeyboard.ts`
**Changes**: Replace goToDeliveries sequences with department navigation

Import department config:
```typescript
import { DEPARTMENTS, DEPARTMENT_BY_SHORTCUT } from '@/data/departments';
```

Update destructuring (around line 28):
```typescript
// Replace: goToDeliveries,
// With:
goToDepartments,
goToQueue,
```

Replace the `g d` handler (around line 199-205) with departments and department shortcuts:
```typescript
// Replace the g d block with this:

// g q -> Departments index view
if (sequence === 'g q') {
  event.preventDefault();
  resetSequence();
  goToDepartments();
  return;
}

// Check for department shortcuts (g + department letter)
if (sequence.startsWith('g ') && sequence.length === 3) {
  const shortcut = sequence[2];
  const dept = DEPARTMENT_BY_SHORTCUT[shortcut];
  if (dept) {
    event.preventDefault();
    resetSequence();
    goToQueue(dept.id);
    return;
  }
}
```

This handles `g q` for departments index plus all individual department shortcuts.

#### 2. Update Command Palette
**File**: `millflow/src/components/CommandPalette.tsx`
**Changes**: Add department navigation commands

Import:
```typescript
import { DEPARTMENTS } from '@/data/departments';
import { Building2, ListTodo } from 'lucide-react';
```

Replace deliveries navigation command with departments index + individual department commands (around line 56):
```typescript
// Remove the "Go to Deliveries" command and add:

// Departments index view
{
  id: 'nav-departments',
  type: 'navigation' as const,
  title: 'Departments',
  subtitle: 'g q',
  icon: Building2,
  action: () => {
    useAppStore.getState().goToDepartments();
    toggleCommandPalette();
  },
},

// Individual department queues
...DEPARTMENTS.map(dept => ({
  id: `nav-queue-${dept.id}`,
  type: 'navigation' as const,
  title: `${dept.name}${dept.isPriorityQueue ? ' Queue' : ''}`,
  subtitle: `g ${dept.shortcut}`,
  icon: ListTodo,
  action: () => {
    useAppStore.getState().goToQueue(dept.id);
    toggleCommandPalette();
  },
})),
```

#### 3. Update Help Modal
**File**: `millflow/src/components/HelpModal.tsx`
**Changes**: Update keyboard shortcut documentation

In the shortcuts array, replace the `g d` entry with departments and department shortcuts:
```typescript
// Replace { key: 'g d', action: 'Go to deliveries' } with:
{ key: 'g q', action: 'Departments' },
// Priority Queues
{ key: 'g r', action: 'Drafting' },
{ key: 'g c', action: 'CNC' },
{ key: 'g m', action: 'Milling' },
{ key: 'g e', action: 'Edge Banding' },
{ key: 'g v', action: 'Veneer' },
{ key: 'g f', action: 'Finishing' },
{ key: 'g b', action: 'Bench' },
{ key: 'g d', action: 'Delivery' },
{ key: 'g i', action: 'Field Install' },
// Status Lists
{ key: 'g s', action: 'Staging' },
{ key: 'g p', action: 'Punch List' },
{ key: 'g o', action: 'Outsourced' },
```

#### 4. Update ShortcutBar Context
**File**: `millflow/src/components/ShortcutBar.tsx`
**Changes**: Update shortcut hints for departments and queue views

Add view shortcuts (in the `getShortcuts` function):
```typescript
// Add case for departments view
if (view === 'departments') {
  return [
    { key: 'j/k', label: 'navigate' },
    { key: 'Enter', label: 'open queue' },
    { key: 'Esc', label: 'back' },
  ];
}

// Add case for queue view
if (view === 'queue') {
  return [
    { key: 'j/k', label: 'navigate' },
    { key: 'Enter', label: 'open sheet' },
    { key: 'Esc', label: 'back' },
  ];
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] `g q` navigates to departments index view
- [ ] Priority queue shortcuts work: `g r` (Drafting), `g c` (CNC), `g m` (Milling), `g e` (Edge Banding), `g v` (Veneer), `g f` (Finishing), `g b` (Bench), `g d` (Delivery), `g i` (Field Install)
- [ ] Status list shortcuts work: `g s` (Staging), `g p` (Punch List), `g o` (Outsourced)
- [ ] Cmd+K shows "Departments" option plus all 12 individual departments
- [ ] Typing "CNC" in command palette shows CNC option
- [ ] Help modal (?) shows `g q` and all department shortcuts
- [ ] ShortcutBar shows correct hints in departments and queue views

**Implementation Note**: After completing this phase, pause for final review and testing.

---

## Testing Strategy

### Unit Tests:
- `getQueueForDepartment` returns correct items for each department
- `getDepartmentForOperation` maps operation names correctly
- Priority queues sort by deadline; status lists sort alphabetically

### Integration Tests:
- Navigation flow: Dashboard → Departments → Queue → Sheet → Back
- Keyboard shortcuts trigger correct navigation for all views
- Command palette search finds Departments and all individual departments

### Manual Testing Steps:
1. Start app, press `g q` - should show Departments index with all 12 departments
2. Press `j` repeatedly - selection should move down the list
3. Press `Enter` on CNC - should open CNC Queue
4. Press `Esc` - should return to Departments view
5. Press `g c` - should go directly to CNC Queue (priority queue with deadline/rank columns)
6. Press `g s` - should show Staging list (status list without deadline/rank columns)
7. Press `Enter` on a queue item - should open sheet view with that node selected
8. Press `Cmd+K`, type "departments" - should show Departments option
9. Press `Cmd+K`, type "CNC" - should show CNC Queue option
10. Press `?` - should show help modal with `g q` and all department shortcuts

## Performance Considerations

- Queue items are computed via `useMemo` to avoid recalculation on every render
- Sorting happens once when sheets/jobs change
- Scroll-into-view uses `behavior: 'smooth'` for better UX without layout thrashing

## Migration Notes

- The `deliveries` view type is removed - any saved state referencing it will fall back to dashboard
- No database migration needed (frontend-only change)
- Existing keyboard muscle memory for `g d` will now go to Delivery Queue (similar destination)

## References

- Original ticket: `https://linear.app/ariav/issue/ARI-24/department-queue-views`
- Research document: `thoughts/shared/research/2025-12-22-department-queue-views.md`
- Department definitions (source of truth): `factory-floor/src/data/mockShopData.ts:2-14` (StationId type)
- UI reference: `factory-floor/src/components/dashboard/StationTableNode.tsx:14-107`
- Navigation patterns: `millflow/src/hooks/useKeyboard.ts:163-205`
