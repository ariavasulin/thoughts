# Shop Schedule View Integration - Implementation Plan

## Overview

Integrate the shop schedule prototype (`misc/shopview.jsx`) into the MillFlow app as a navigable view. The schedule view provides a production-tracking interface where department heads see prioritized work across all active sheets, with the ability to cycle status and edit durations.

## Design Decision: Parallel Data Structure (Option A)

**Decision**: Add a separate `scheduleSheets` data structure rather than transforming existing DAG nodes.

**Rationale**:
- The schedule view uses a flat operations model: `sheet.operations[deptId]` with `{duration, status, dependsOn}`
- MillFlow's existing model uses DAG nodes with parent/child relationships for flow visualization
- These serve different purposes: schedule = production tracking, DAG = flow editing
- Keeping them separate is cleaner and matches the prototype exactly

**Trade-off**: Two data structures for similar concepts. Future work could unify them or derive schedule data from DAG nodes.

**Note for Linear ticket**: Document this architectural decision - schedule view uses parallel data structure, not derived from existing sheet nodes.

## Current State Analysis

### What Exists
- `misc/shopview.jsx` - Complete prototype (~640 lines) with:
  - Status cycling: pending → inProgress → done
  - Duration editing via right-click portal
  - Priority sorting by slack (days until install - remaining work)
  - Gruvbox theming (inline styles)
- `misc/mockData.js` - Mock data generator (~70 sheets across 10 projects)

### Key Discoveries
- ViewType at `millflow/src/types/index.ts:150` needs `'schedule'` added
- All views use consistent patterns: `memo()`, `useAppStore()`, `containerRef`, `data-selected`
- Gruvbox Tailwind classes available: `bg-gruvbox-{blue,green,yellow,orange,red}-bright`
- Store navigation pattern at `appStore.ts:441-456` for `goTo*` actions

## Desired End State

After implementation:
1. User can navigate to schedule view via `g s` keyboard shortcut or Command Palette
2. Schedule displays all active sheets with department columns showing operation status
3. Clicking actionable/in-progress cells cycles status
4. Right-clicking opens duration editor
5. Sheets sorted by priority (slack = days until install - remaining work)
6. Keyboard navigation (j/k) moves through rows

### Verification
- `npm run build` passes with no type errors
- `npm run dev` shows schedule view accessible via `g s`
- Status cycling and duration editing work as in prototype

## What We're NOT Doing

- **Not unifying data models** - Schedule uses separate data structure from DAG nodes
- **Not adding persistence** - Mock data only, no backend integration
- **Not adding filtering/search** - Basic view only
- **Not adding department-specific views from schedule** - Can drill down later

---

## Phase 1: Types and Mock Data

### Overview
Add TypeScript types for schedule data and port mock data generator.

### Changes Required

#### 1. Add ViewType
**File**: `millflow/src/types/index.ts`
**Line**: 150

```typescript
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'departments' | 'queue' | 'schedule';
```

#### 2. Add Schedule Types
**File**: `millflow/src/types/index.ts`
**Add after line 157** (after KeySequence interface):

```typescript
// Schedule View Types
export type ScheduleOperationStatus = 'pending' | 'inProgress' | 'done';

export interface ScheduleOperation {
  duration: number;
  status: ScheduleOperationStatus;
  dependsOn: string[]; // department IDs
}

export interface ScheduleSheet {
  id: string;
  pid: string;
  project: string;
  sheet: string;
  description: string;
  targetInstall: string; // ISO date
  notes: string;
  operations: Record<string, ScheduleOperation>; // keyed by department ID
}

export interface ScheduleDepartment {
  id: string;
  name: string;
}
```

#### 3. Create Schedule Mock Data
**File**: `millflow/src/data/scheduleData.ts` (new file)

```typescript
import type { ScheduleSheet, ScheduleDepartment } from '@/types';

// Department definitions for schedule view
export const SCHEDULE_DEPARTMENTS: ScheduleDepartment[] = [
  { id: 'release', name: 'Release' },
  { id: 'veneer', name: 'Veneer' },
  { id: 'mill', name: 'Mill' },
  { id: 'cnc', name: 'CNC' },
  { id: 'bench', name: 'Bench' },
  { id: 'finish', name: 'Finish' },
  { id: 'assembly', name: 'Assembly' },
  { id: 'delivery', name: 'Loading' },
];

// Project definitions
const PROJECTS = [
  { pid: '4084', name: 'Sidley Austin 2025' },
  { pid: '4070', name: 'Open AI 550 TF' },
  { pid: '4085', name: 'Goldman Sachs 2025' },
  { pid: '4086', name: 'Fenwick and West' },
  { pid: '4087', name: 'Allen Matkins' },
  { pid: '4049', name: 'Global Relay' },
  { pid: '4091', name: 'Kirkland Ellis' },
  { pid: '4092', name: 'Morrison Foerster' },
  { pid: '4093', name: 'Latham Watkins' },
  { pid: '4094', name: 'Paul Hastings' },
];

const SHEET_TYPES = [
  'Pantry', 'Closet', 'Restroom', 'Conference', 'Reception',
  'Library', 'Elevator Lobby', 'Cafe', 'Huddle', 'Wall Panel',
];

const NOTES_POOL = [
  'NPZ', 'BS5 M2 NPZ', 'V4 - revised design', 'In Production',
  'Manual stockbill', 'Partial Delivery', 'Rush - priority',
  'Material on order', 'Veneer matched', 'Template received', '', '', '', '',
];

function randomStatus(doneWeight = 0.3, inProgressWeight = 0.1): 'pending' | 'inProgress' | 'done' {
  const r = Math.random();
  if (r < doneWeight) return 'done';
  if (r < doneWeight + inProgressWeight) return 'inProgress';
  return 'pending';
}

function randomInstallDate(minDays: number, maxDays: number): string {
  const today = new Date();
  const days = minDays + Math.floor(Math.random() * (maxDays - minDays));
  const date = new Date(today.getTime() + days * 24 * 60 * 60 * 1000);
  return date.toISOString().split('T')[0];
}

function generateSheet(id: number, project: typeof PROJECTS[0], sheetNum: number): ScheduleSheet {
  const targetInstall = randomInstallDate(-5 + Math.floor(id / 7) * 3, 45 + Math.floor(id / 7) * 5);
  const today = new Date();
  const installDate = new Date(targetInstall);
  const daysUntilInstall = Math.floor((installDate.getTime() - today.getTime()) / (1000 * 60 * 60 * 24));
  let progressFactor = 1 - (daysUntilInstall / 60);
  progressFactor = Math.max(0, Math.min(1, progressFactor));

  const ops: Record<string, { duration: number; status: 'pending' | 'inProgress' | 'done'; dependsOn: string[] }> = {
    release: {
      duration: 1,
      status: progressFactor > 0.1 ? 'done' : randomStatus(0.8),
      dependsOn: []
    },
    veneer: {
      duration: [0, 2, 3, 4, 5, 6][Math.floor(Math.random() * 6)],
      status: 'pending',
      dependsOn: ['release']
    },
    mill: {
      duration: [1, 2, 2, 3, 3, 4, 5, 6][Math.floor(Math.random() * 8)],
      status: 'pending',
      dependsOn: ['release']
    },
    cnc: {
      duration: [1, 2, 2, 3, 3, 4, 5][Math.floor(Math.random() * 7)],
      status: 'pending',
      dependsOn: ['mill']
    },
    bench: {
      duration: [2, 3, 4, 5, 5, 6, 7, 8, 10][Math.floor(Math.random() * 9)],
      status: 'pending',
      dependsOn: ['veneer', 'cnc']
    },
    finish: {
      duration: [2, 3, 4, 5, 5, 6, 7][Math.floor(Math.random() * 7)],
      status: 'pending',
      dependsOn: ['bench']
    },
    assembly: {
      duration: [1, 2, 2, 3, 3, 4][Math.floor(Math.random() * 6)],
      status: 'pending',
      dependsOn: ['finish']
    },
    delivery: {
      duration: 1,
      status: 'pending',
      dependsOn: ['assembly']
    }
  };

  // Progress operations based on install date proximity
  if (ops.release.status === 'done') {
    if (progressFactor > 0.25) {
      if (ops.veneer.duration > 0) ops.veneer.status = Math.random() < 0.7 ? 'done' : 'inProgress';
      ops.mill.status = Math.random() < 0.7 ? 'done' : 'inProgress';
    }
    if (ops.mill.status === 'done' && progressFactor > 0.35) {
      ops.cnc.status = Math.random() < 0.6 ? 'done' : 'inProgress';
    }
    if (ops.cnc.status === 'done' && (ops.veneer.status === 'done' || ops.veneer.duration === 0)) {
      if (progressFactor > 0.5) ops.bench.status = Math.random() < 0.5 ? 'done' : 'inProgress';
    }
    if (ops.bench.status === 'done' && progressFactor > 0.65) {
      ops.finish.status = Math.random() < 0.4 ? 'done' : 'inProgress';
    }
    if (ops.finish.status === 'done' && progressFactor > 0.8) {
      ops.assembly.status = Math.random() < 0.3 ? 'done' : 'inProgress';
    }
  }

  const sheetType = SHEET_TYPES[Math.floor(Math.random() * SHEET_TYPES.length)];
  const roomNum = `${300 + Math.floor(Math.random() * 100)}${String(Math.floor(Math.random() * 99)).padStart(2, '0')}`;

  return {
    id: `sched-${id}`,
    pid: project.pid,
    project: project.name,
    sheet: `SK-${String(sheetNum).padStart(2, '0')}`,
    description: `${sheetType} ${roomNum}`,
    targetInstall,
    notes: NOTES_POOL[Math.floor(Math.random() * NOTES_POOL.length)],
    operations: ops,
  };
}

function generateScheduleSheets(): ScheduleSheet[] {
  const sheets: ScheduleSheet[] = [];
  let id = 1;

  PROJECTS.forEach((project) => {
    const numSheets = 4 + Math.floor(Math.random() * 9);
    for (let i = 0; i < numSheets; i++) {
      sheets.push(generateSheet(id++, project, i + 1));
    }
  });

  return sheets;
}

export const scheduleSheets: ScheduleSheet[] = generateScheduleSheets();
```

### Success Criteria

#### Automated Verification
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] No lint errors: `cd millflow && npm run lint`
- [x] Types are exported correctly (no import errors)

#### Manual Verification
- [ ] `scheduleSheets` array generates ~70 sheets when imported

---

## Phase 2: ScheduleView Component

### Overview
Port the prototype to TypeScript with Tailwind classes, following MillFlow view patterns.

### Changes Required

#### 1. Create ScheduleView Component
**File**: `millflow/src/views/ScheduleView.tsx` (new file)

```typescript
import { memo, useState, useMemo, useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';
import { Calendar } from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { cn } from '@/lib/styles';
import { SCHEDULE_DEPARTMENTS, scheduleSheets } from '@/data/scheduleData';
import type { ScheduleSheet, ScheduleOperationStatus } from '@/types';

// Check if an operation is actionable (dependencies met, pending, has duration)
function isActionable(sheet: ScheduleSheet, deptId: string): boolean {
  const op = sheet.operations[deptId];
  if (!op || op.status !== 'pending' || op.duration === 0) return false;
  return op.dependsOn.every(depId => {
    const depOp = sheet.operations[depId];
    return depOp && depOp.status === 'done';
  });
}

function isInProgress(sheet: ScheduleSheet, deptId: string): boolean {
  const op = sheet.operations[deptId];
  return op?.status === 'inProgress';
}

function isSheetComplete(sheet: ScheduleSheet): boolean {
  return SCHEDULE_DEPARTMENTS.every(dept => {
    const op = sheet.operations[dept.id];
    return op.duration === 0 || op.status === 'done';
  });
}

function getPriorityScore(sheet: ScheduleSheet): number {
  const hasActionable = SCHEDULE_DEPARTMENTS.some(
    dept => isActionable(sheet, dept.id) || isInProgress(sheet, dept.id)
  );
  if (!hasActionable) return Infinity;

  const today = new Date();
  today.setHours(0, 0, 0, 0);
  const target = new Date(sheet.targetInstall);
  target.setHours(0, 0, 0, 0);

  const daysUntilInstall = Math.floor((target.getTime() - today.getTime()) / (1000 * 60 * 60 * 24));

  let remainingDays = 0;
  SCHEDULE_DEPARTMENTS.forEach(dept => {
    const op = sheet.operations[dept.id];
    if (op.duration > 0 && op.status !== 'done') {
      remainingDays += op.duration;
    }
  });

  return daysUntilInstall - remainingDays;
}

function formatDate(date: string): string {
  if (!date) return '';
  const d = new Date(date);
  return `${(d.getMonth() + 1).toString().padStart(2, '0')}/${d.getDate().toString().padStart(2, '0')}`;
}

// Duration Editor Component
function DurationEditor({
  duration,
  onSave,
  onClose,
  position
}: {
  duration: number;
  onSave: (d: number) => void;
  onClose: () => void;
  position: { x: number; y: number };
}) {
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
    inputRef.current?.select();
  }, []);

  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      const val = parseFloat(e.currentTarget.value);
      if (!isNaN(val) && val >= 0) onSave(val);
      onClose();
    } else if (e.key === 'Escape') {
      onClose();
    }
  };

  return createPortal(
    <input
      ref={inputRef}
      type="number"
      step="0.5"
      min="0"
      defaultValue={duration}
      onKeyDown={handleKeyDown}
      onBlur={onClose}
      className="fixed w-14 px-2 py-1 bg-gruvbox-bg-1 text-gruvbox-fg border border-gruvbox-blue-bright rounded text-sm outline-none z-50"
      style={{ left: position.x, top: position.y }}
    />,
    document.body
  );
}

// Department Cell Component
function DeptCell({
  sheet,
  deptId,
  priorityScore,
  onStatusChange,
  onDurationChange,
}: {
  sheet: ScheduleSheet;
  deptId: string;
  priorityScore: number;
  onStatusChange: (sheetId: string, deptId: string, status: ScheduleOperationStatus) => void;
  onDurationChange: (sheetId: string, deptId: string, duration: number) => void;
}) {
  const [showEditor, setShowEditor] = useState(false);
  const [editorPos, setEditorPos] = useState({ x: 0, y: 0 });

  const op = sheet.operations[deptId];
  const actionable = isActionable(sheet, deptId);
  const inProgress = isInProgress(sheet, deptId);
  const isDone = op.status === 'done';
  const isSkipped = op.duration === 0;

  const handleClick = () => {
    if (actionable) {
      onStatusChange(sheet.id, deptId, 'inProgress');
    } else if (inProgress) {
      onStatusChange(sheet.id, deptId, 'done');
    }
  };

  const handleContextMenu = (e: React.MouseEvent) => {
    e.preventDefault();
    if (!isDone) {
      setEditorPos({ x: e.clientX, y: e.clientY });
      setShowEditor(true);
    }
  };

  if (isSkipped) {
    return <td className="text-center px-1 py-1.5 min-w-14 bg-gruvbox-bg-1 text-gruvbox-bg-3">-</td>;
  }

  const isClickable = actionable || inProgress;

  let cellClass = 'text-center px-1 py-1.5 min-w-14 font-mono text-xs ';
  if (isDone) {
    cellClass += 'bg-gruvbox-blue-bright text-gruvbox-bg-hard font-bold';
  } else if (inProgress) {
    cellClass += 'bg-gruvbox-green-bright text-gruvbox-bg-hard font-bold cursor-pointer';
  } else if (actionable) {
    if (priorityScore < 0) {
      cellClass += 'bg-gruvbox-red-bright text-gruvbox-bg-hard font-bold cursor-pointer';
    } else if (priorityScore <= 3) {
      cellClass += 'bg-gruvbox-orange-bright text-gruvbox-bg-hard font-bold cursor-pointer';
    } else {
      cellClass += 'bg-gruvbox-yellow-bright text-gruvbox-bg-hard font-bold cursor-pointer';
    }
  } else {
    cellClass += 'bg-gruvbox-bg-2 text-gruvbox-fg-4';
  }

  return (
    <td
      className={cellClass}
      onClick={isClickable ? handleClick : undefined}
      onContextMenu={handleContextMenu}
      title={isClickable ? 'Click to advance, right-click to edit duration' : 'Right-click to edit duration'}
    >
      {op.duration}
      {showEditor && (
        <DurationEditor
          duration={op.duration}
          onSave={(d) => onDurationChange(sheet.id, deptId, d)}
          onClose={() => setShowEditor(false)}
          position={editorPos}
        />
      )}
    </td>
  );
}

// Priority Badge Component
function PriorityBadge({ score }: { score: number }) {
  let className = 'inline-block px-1.5 py-0.5 rounded text-xs font-bold ';
  let label = '';

  if (score < 0) {
    className += 'bg-gruvbox-red-bright text-gruvbox-bg-hard';
    label = `${score}d`;
  } else if (score <= 3) {
    className += 'bg-gruvbox-orange-bright text-gruvbox-bg-hard';
    label = `+${score}d`;
  } else if (score <= 7) {
    className += 'bg-gruvbox-yellow-bright text-gruvbox-bg-hard';
    label = `+${score}d`;
  } else {
    className += 'bg-gruvbox-green-bright text-gruvbox-bg-hard';
    label = `+${score}d`;
  }

  return <span className={className}>{label}</span>;
}

// Main Component
export const ScheduleView = memo(function ScheduleView() {
  const { selectedIndex } = useAppStore();
  const containerRef = useRef<HTMLDivElement>(null);
  const [sheets, setSheets] = useState(scheduleSheets);
  const [sortBy, setSortBy] = useState<'priority' | 'project' | 'install'>('priority');

  // Scroll selected row into view
  useEffect(() => {
    const selected = containerRef.current?.querySelector('[data-selected="true"]');
    selected?.scrollIntoView({ block: 'nearest', behavior: 'smooth' });
  }, [selectedIndex]);

  const handleStatusChange = (sheetId: string, deptId: string, newStatus: ScheduleOperationStatus) => {
    setSheets(prev => prev.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        operations: {
          ...sheet.operations,
          [deptId]: { ...sheet.operations[deptId], status: newStatus },
        },
      };
    }));
  };

  const handleDurationChange = (sheetId: string, deptId: string, newDuration: number) => {
    setSheets(prev => prev.map(sheet => {
      if (sheet.id !== sheetId) return sheet;
      return {
        ...sheet,
        operations: {
          ...sheet.operations,
          [deptId]: { ...sheet.operations[deptId], duration: newDuration },
        },
      };
    }));
  };

  const activeSheets = useMemo(() => sheets.filter(s => !isSheetComplete(s)), [sheets]);

  const sortedSheets = useMemo(() => {
    return [...activeSheets].sort((a, b) => {
      if (sortBy === 'priority') return getPriorityScore(a) - getPriorityScore(b);
      if (sortBy === 'project') return a.project.localeCompare(b.project);
      if (sortBy === 'install') return new Date(a.targetInstall).getTime() - new Date(b.targetInstall).getTime();
      return 0;
    });
  }, [activeSheets, sortBy]);

  const completeCount = sheets.filter(s => isSheetComplete(s)).length;

  return (
    <div className="flex flex-col h-full bg-gruvbox-bg-hard">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-3 border-b border-gruvbox-bg-2 bg-gruvbox-bg">
        <div className="flex items-center gap-3">
          <Calendar size={20} className="text-gruvbox-blue-bright" />
          <h1 className="text-lg font-semibold text-gruvbox-fg-0">Shop Schedule</h1>
          <span className="text-xs font-mono text-gruvbox-fg-4 bg-gruvbox-bg-1 px-2 py-0.5 rounded">
            {activeSheets.length} active
          </span>
          <span className="text-xs text-gruvbox-fg-4">
            {completeCount} complete (hidden)
          </span>
        </div>
        <div className="flex items-center gap-2">
          <span className="text-xs text-gruvbox-fg-4 mr-2">Sort:</span>
          {(['priority', 'project', 'install'] as const).map(sort => (
            <button
              key={sort}
              onClick={() => setSortBy(sort)}
              className={cn(
                'px-2 py-1 text-xs rounded transition-colors',
                sortBy === sort
                  ? 'bg-gruvbox-blue-bright text-gruvbox-bg-hard'
                  : 'bg-gruvbox-bg-2 text-gruvbox-fg hover:bg-gruvbox-bg-3'
              )}
            >
              {sort.charAt(0).toUpperCase() + sort.slice(1)}
            </button>
          ))}
          <span className="text-xs text-gruvbox-fg-4 ml-4">
            <kbd className="kbd">g s</kbd>
          </span>
        </div>
      </div>

      {/* Legend */}
      <div className="flex items-center gap-4 px-4 py-2 border-b border-gruvbox-bg-2 bg-gruvbox-bg text-xs">
        {[
          { color: 'bg-gruvbox-blue-bright', label: 'Done' },
          { color: 'bg-gruvbox-green-bright', label: 'In Progress' },
          { color: 'bg-gruvbox-yellow-bright', label: 'Actionable' },
          { color: 'bg-gruvbox-orange-bright', label: 'Urgent' },
          { color: 'bg-gruvbox-red-bright', label: 'Overdue' },
          { color: 'bg-gruvbox-bg-2', label: 'Blocked' },
          { color: 'bg-gruvbox-bg-1', label: 'N/A' },
        ].map(({ color, label }) => (
          <div key={label} className="flex items-center gap-1.5">
            <div className={cn('w-3 h-3 rounded-sm', color)} />
            <span className="text-gruvbox-fg-4">{label}</span>
          </div>
        ))}
      </div>

      {/* Table */}
      <div ref={containerRef} className="flex-1 overflow-auto">
        <table className="w-full border-collapse text-xs">
          <thead className="sticky top-0 z-10">
            <tr className="bg-gruvbox-bg-1">
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-14">Slack</th>
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-12">PID</th>
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-32">Project</th>
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-16">Sheet</th>
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-48">Description</th>
              {SCHEDULE_DEPARTMENTS.map(dept => (
                <th key={dept.id} className="text-center px-1 py-2 text-gruvbox-fg-3 font-normal min-w-14">
                  {dept.name}
                </th>
              ))}
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-16">Delivery</th>
              <th className="text-left px-2 py-2 text-gruvbox-fg-3 font-normal w-40">Notes</th>
            </tr>
          </thead>
          <tbody>
            {sortedSheets.map((sheet, idx) => {
              const priorityScore = getPriorityScore(sheet);
              const isSelected = selectedIndex === idx;

              return (
                <tr
                  key={sheet.id}
                  data-selected={isSelected}
                  className={cn(
                    'border-b border-gruvbox-bg-1 transition-colors',
                    isSelected ? 'bg-gruvbox-bg-1' : 'hover:bg-gruvbox-bg-soft'
                  )}
                >
                  <td className="px-2 py-1.5">
                    <PriorityBadge score={priorityScore} />
                  </td>
                  <td className="px-2 py-1.5 text-gruvbox-fg-4 font-mono">{sheet.pid}</td>
                  <td className="px-2 py-1.5 text-gruvbox-fg truncate max-w-32" title={sheet.project}>
                    {sheet.project}
                  </td>
                  <td className="px-2 py-1.5 text-gruvbox-aqua-bright font-mono">{sheet.sheet}</td>
                  <td className="px-2 py-1.5 text-gruvbox-fg truncate max-w-48" title={sheet.description}>
                    {sheet.description}
                  </td>
                  {SCHEDULE_DEPARTMENTS.map(dept => (
                    <DeptCell
                      key={dept.id}
                      sheet={sheet}
                      deptId={dept.id}
                      priorityScore={priorityScore}
                      onStatusChange={handleStatusChange}
                      onDurationChange={handleDurationChange}
                    />
                  ))}
                  <td className="px-2 py-1.5 text-gruvbox-aqua-bright font-bold">
                    {formatDate(sheet.targetInstall)}
                  </td>
                  <td className="px-2 py-1.5 text-gruvbox-fg-4 italic truncate max-w-40" title={sheet.notes}>
                    {sheet.notes}
                  </td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>
    </div>
  );
});

export default ScheduleView;
```

### Success Criteria

#### Automated Verification
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] No lint errors: `cd millflow && npm run lint`
- [x] Component exports correctly

#### Manual Verification
- [ ] Component renders table with departments as columns
- [ ] Status colors display correctly
- [ ] Right-click duration editor appears and works

**Implementation Note**: After completing this phase and automated verification passes, pause for manual confirmation before proceeding.

---

## Phase 3: Navigation Wiring

### Overview
Connect the schedule view to the app's navigation system.

### Changes Required

#### 1. Add to App Router
**File**: `millflow/src/App.tsx`
**Line 11**: Add import

```typescript
import ScheduleView from '@/views/ScheduleView';
```

**Line 30-32**: Add case in switch (after queue case)

```typescript
      case 'schedule':
        return <ScheduleView />;
```

#### 2. Add Store Action
**File**: `millflow/src/store/appStore.ts`
**Line 80**: Add to interface (after goToQueue)

```typescript
  goToSchedule: () => void;
```

**Line 456**: Add implementation (after goToQueue)

```typescript
  goToSchedule: () => {
    set({
      view: 'schedule',
      selectedIndex: 0,
      showNodeDetail: false,
    });
  },
```

#### 3. Add Keyboard Shortcut
**File**: `millflow/src/hooks/useKeyboard.ts`
**Line 30**: Add goToSchedule to destructuring

```typescript
    goToSchedule,
```

**Line 207**: Add after `g q` sequence handler (around line 207)

```typescript
    // g s - go to schedule
    if (sequence === 'g s') {
      event.preventDefault();
      resetSequence();
      goToSchedule();
      return;
    }
```

**Line 530**: Add to dependencies array

```typescript
    goToSchedule,
```

#### 4. Add to Command Palette
**File**: `millflow/src/components/CommandPalette.tsx`
**Line 8**: Add Calendar import

```typescript
import {
  Search,
  Home,
  Briefcase,
  FileText,
  User,
  ArrowRight,
  Building2,
  ListTodo,
  Calendar,
} from 'lucide-react';
```

**Line 71**: Add after departments item (before individual department queues loop)

```typescript
    items.push({
      id: 'nav-schedule',
      type: 'navigation',
      title: 'Shop Schedule',
      subtitle: 'g s',
      icon: Calendar,
      action: () => {
        useAppStore.getState().goToSchedule();
        toggleCommandPalette();
      },
    });
```

### Success Criteria

#### Automated Verification
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] No lint errors: `cd millflow && npm run lint`

#### Manual Verification
- [ ] Press `g s` from any view to navigate to schedule
- [ ] Press `Cmd+K`, type "schedule", select to navigate
- [ ] Schedule view displays with data
- [ ] `Escape` navigates back

**Implementation Note**: After completing this phase and automated verification passes, pause for manual confirmation before proceeding.

---

## Phase 4: Store Integration (Optional Enhancement)

### Overview
Move schedule state from component-local to store for persistence across navigation.

### Changes Required

#### 1. Add Schedule State to Store
**File**: `millflow/src/store/appStore.ts`

Add to state interface (around line 60):
```typescript
  // Schedule data
  scheduleSheets: ScheduleSheet[];
```

Add to actions interface (around line 82):
```typescript
  setScheduleOperationStatus: (sheetId: string, deptId: string, status: ScheduleOperationStatus) => void;
  setScheduleOperationDuration: (sheetId: string, deptId: string, duration: number) => void;
```

Add import at top:
```typescript
import { scheduleSheets as initialScheduleSheets } from '@/data/scheduleData';
import type { ScheduleSheet, ScheduleOperationStatus } from '@/types';
```

Add initial state (around line 179):
```typescript
  scheduleSheets: initialScheduleSheets,
```

Add actions (around line 460):
```typescript
  setScheduleOperationStatus: (sheetId, deptId, status) => set(state => ({
    scheduleSheets: state.scheduleSheets.map(sheet =>
      sheet.id === sheetId
        ? {
            ...sheet,
            operations: {
              ...sheet.operations,
              [deptId]: { ...sheet.operations[deptId], status },
            },
          }
        : sheet
    ),
  })),

  setScheduleOperationDuration: (sheetId, deptId, duration) => set(state => ({
    scheduleSheets: state.scheduleSheets.map(sheet =>
      sheet.id === sheetId
        ? {
            ...sheet,
            operations: {
              ...sheet.operations,
              [deptId]: { ...sheet.operations[deptId], duration },
            },
          }
        : sheet
    ),
  })),
```

#### 2. Update ScheduleView to Use Store
**File**: `millflow/src/views/ScheduleView.tsx`

Replace local state with store:
```typescript
const { selectedIndex, scheduleSheets, setScheduleOperationStatus, setScheduleOperationDuration } = useAppStore();
```

Remove `useState` for sheets and use store data directly.

### Success Criteria

#### Automated Verification
- [ ] TypeScript compiles: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification
- [ ] Navigate away from schedule and back - state persists
- [ ] Status changes persist across navigation
- [ ] Duration changes persist across navigation

---

## Testing Strategy

### Manual Testing Steps
1. Start dev server: `cd millflow && npm run dev`
2. Navigate to schedule via `g s`
3. Verify table displays with ~70 active sheets
4. Click an actionable (yellow) cell - should turn green
5. Click in-progress (green) cell - should turn blue
6. Right-click any non-done cell - duration editor appears
7. Change duration and press Enter - value updates
8. Sort buttons work (Priority, Project, Install)
9. Keyboard navigation (j/k) highlights rows
10. `Escape` returns to previous view

### Edge Cases
- Sheets with 0 duration operations (skip with `-`)
- Overdue sheets (negative slack, red highlighting)
- Complete sheets (hidden from view)
- Empty notes field

## References

- Handoff document: `thoughts/shared/handoffs/general/2025-12-22_23-43-20_shop-schedule-integration.md`
- Prototype: `misc/shopview.jsx`
- Mock data generator: `misc/mockData.js`
- View patterns: `millflow/src/views/QueueView.tsx`
- Store patterns: `millflow/src/store/appStore.ts:441-456`
