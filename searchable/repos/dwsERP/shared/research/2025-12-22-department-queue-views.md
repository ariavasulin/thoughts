---
date: 2025-12-22T11:41:41-08:00
researcher: ariasulin
git_commit: 29fda99ff7f459051770495eb75fb56b1ab9d8ac
branch: main
repository: dwsERP
topic: "Department Queue Views Implementation Research"
tags: [research, codebase, department-queues, command-palette, navigation]
status: complete
last_updated: 2025-12-22
last_updated_by: ariasulin
last_updated_note: "Corrected department mapping model - operation type determines department, not assignees. Prioritization from solver, not frontend."
---

# Research: Department Queue Views Implementation

**Date**: 2025-12-22T11:41:41-08:00
**Researcher**: ariasulin
**Git Commit**: 29fda99ff7f459051770495eb75fb56b1ab9d8ac
**Branch**: main
**Repository**: dwsERP

## Research Question

How to add custom queue views for each department showing which sheets are in their court, with prioritized ordering. Navigation via Command-K to search by department name.

## Summary

The millflow codebase has the navigation infrastructure needed for department queue views:

1. **Command Palette** (`CommandPalette.tsx`) - dynamic command building with fuzzy search, easily extended for department navigation
2. **View System** - simple `ViewType` enum + switch statement in `App.tsx`
3. **Keyboard shortcuts** - multi-key sequences like `g c` for CNC queue
4. **Factory-floor prototype** - reference UI for queue tables (visual indicators, scrollable lists)

**Critical Conceptual Model** (corrected from initial research):

- **Department mapping is by operation type, not assignee** - CNC operations go to CNC queue regardless of who's assigned. Assignment is foreman's job, not PM's.
- **Prioritization comes from the scheduling solver** - OR-Tools calculates the queue order based on deadlines, dependencies, and capacity. The queue view just displays solver output.
- **No blocker tracking** - When inputs change (delays, rush jobs), the solver recalculates and priorities shift automatically.

## Detailed Findings

### 1. Current View System Architecture

**File**: `millflow/src/App.tsx:18-33`

Views are rendered via a switch statement based on `view` state from Zustand:

```typescript
const renderView = () => {
  switch (view) {
    case 'dashboard':
      return <DashboardView />;
    case 'jobs':
      return <JobsView />;
    case 'job':
      return <JobView />;
    case 'sheet':
      return <SheetView />;
    case 'deliveries':
      return <DeliveriesView />;
    default:
      return <DashboardView />;
  }
};
```

**ViewType Definition** (`millflow/src/types/index.ts:115`):
```typescript
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'deliveries';
```

Adding a new view requires:
1. Add to `ViewType` union
2. Add case to `App.tsx` switch
3. Create view component in `src/views/`
4. Add navigation action to store
5. Add keyboard shortcut and/or command palette entry

### 2. Command Palette Implementation

**File**: `millflow/src/components/CommandPalette.tsx`

The palette builds commands dynamically from multiple sources (lines 30-121):
- **Navigation commands** (lines 34-56): Static navigation targets with keyboard hints
- **Job items** (lines 59-72): Dynamic from `jobs` array
- **Sheet items** (lines 74-90): Dynamic from `sheets` array
- **People** (lines 93-105): Dynamic from `people` array
- **Actions** (lines 108-118): Meta actions like "Show Help"

**Command Structure** (lines 13-20):
```typescript
interface CommandItem {
  id: string;
  type: 'navigation' | 'job' | 'sheet' | 'person' | 'action';
  title: string;
  subtitle?: string;  // Shows keyboard hint or context
  icon: LucideIcon;
  action: () => void;
}
```

**Search Implementation** (lines 127-133):
- Lowercase substring match on title and subtitle
- Limited to 10 results
- Results grouped by type with section headers

**To add department navigation**: Insert department commands in the dynamic building section, using type `'navigation'` with subtitle showing the keyboard shortcut (e.g., `'g c'` for CNC).

### 3. Operation Type → Department Mapping (NEW MODEL)

**Key insight**: The operation type itself determines which department queue an item belongs to. A "CNC" operation goes to the CNC queue regardless of who's assigned. Assignment is the foreman's job on the floor, not the PM's concern.

**Current NodeType** (`millflow/src/types/index.ts:4`):
```typescript
export type NodeType = 'shop' | 'external' | 'material' | 'approval' | 'qc' | 'delivery' | 'install';
```

**Current Operation Names** (`millflow/src/data/operations.ts:10-43`):
- `cnc`, `edge-banding`, `veneer`, `milling`, `finishing`, `bench`, `drafting` (shop)
- `outsourced` (external)
- `order-material`, `receive-material` (material)
- `staging`, `delivery` (delivery)
- `field-install`, `punch-list` (install)

**Required: Operation → Department Map** (NOT YET IMPLEMENTED):
```typescript
// New first-class concept needed
const DEPT_OPERATIONS: Record<string, string[]> = {
  'CNC': ['cnc', 'cnc-programming', 'cnc-nesting'],
  'Finishing': ['finishing', 'stain', 'lacquer', 'paint'],
  'Assembly': ['bench', 'assembly', 'hardware', 'edge-banding'],
  'Veneer': ['veneer'],
  'Drafting': ['drafting', 'shop-drawing'],
  'Delivery': ['staging', 'delivery'],
  'Install': ['field-install', 'punch-list'],
};

// Finding nodes for a department (simple!)
const cncNodes = sheets.flatMap(s =>
  s.nodes.filter(n => DEPT_OPERATIONS['CNC'].includes(n.name))
);
```

This is simpler than the assignee-based approach and matches the business model: operations imply departments.

### 4. Time Estimate Defaults (MISSING)

Per the updated scope, time estimates should be:
- **Defaults per operation type** - Standard duration for "CNC" operation
- **Adjustable per sheet** - If something's unusually complex

**Current State**: `duration_estimate` field exists on Node but:
- Not populated in mock data
- No operation type → default duration mapping
- No UI for adjusting estimates

**Needed Data Structure**:
```typescript
// Default durations per operation type
const OPERATION_DEFAULTS: Record<string, number> = {
  'cnc': 4,        // hours
  'finishing': 8,
  'bench': 6,
  // etc.
};

// Node-level override
interface Node {
  // existing fields...
  duration_estimate?: number;  // Overrides default if set
}
```

### 5. Factory-Floor Reference Implementation (UI PATTERNS ONLY)

**File**: `factory-floor/src/components/dashboard/Dashboard.tsx`

The prototype's **UI patterns** are useful reference, but its **data model** is different:

**Station Table UI** (`factory-floor/src/components/dashboard/StationTableNode.tsx:14-107`):
- Shows station name + count in header
- Table with columns: ID, Project, Description, Due Date
- Scrollable body (max 300px)
- Visual indicators:
  - Project color dots
  - Red bold text for critical urgency due dates

**Key Differences from New Direction**:

| Factory-Floor | New MillFlow |
|--------------|--------------|
| Filters sheets by `currentStations` (location) | Filters nodes by operation type → department |
| Shows "what's physically here now" | Shows "what to work on next" (solver output) |
| No prioritization order | Ordered by solver-calculated priority |
| Tracks blocker status | No blocker concept - solver recalculates on input changes |

The factory-floor UI layout (station tables with queue items) is good reference. The underlying data flow is completely different.

### 6. Keyboard Navigation Patterns

**File**: `millflow/src/hooks/useKeyboard.ts`

**Multi-key sequences** for navigation (lines 164-205):
- `g h` - Go home (dashboard)
- `g j` - Go to jobs
- `g d` - Go to deliveries
- `g g` - Jump to top

**Adding department shortcuts**: Follow pattern in lines 164-205:
```typescript
if (sequence === 'g c') {  // 'c' for CNC
  event.preventDefault();
  resetSequence();
  goToQueue('CNC');
  return;
}
```

**Pending sequence indicator** (lines 286-289): Shows visual feedback when waiting for second key.

### 7. Solver-Based Queue Architecture (NEW)

The queue view is a **display layer** for solver output, not a place where prioritization happens.

**Data Flow**:
```
┌─────────────────────────────────────────────────────────────┐
│                     SOLVER INPUTS                           │
├─────────────────────────────────────────────────────────────┤
│ • Active sheets + delivery/install dates                    │
│ • Sheet flows (operation sequences, parallel branches)      │
│ • Time estimates per operation (defaults + overrides)       │
│ • Department capacity (hours/day)                           │
│ • Current status from PMDB (what's done)                   │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    PyJobShop (OR-Tools)
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                     SOLVER OUTPUT                           │
├─────────────────────────────────────────────────────────────┤
│ Per department, per day:                                    │
│ • Ranked queue of operations                                │
│ • Calculated start/end times                                │
│ • Resource assignments (optional)                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
                    Queue View (display only)
```

**What Queue View Needs from Solver**:
```typescript
interface DepartmentQueue {
  department: string;
  date: string;
  items: QueueItem[];
}

interface QueueItem {
  rank: number;              // Solver-determined priority
  sheetId: string;
  sheetName: string;
  jobId: string;
  jobName: string;
  operationType: string;     // 'cnc', 'finishing', etc.
  estimatedDuration: number; // hours
  deadline: string;          // Propagated from sheet delivery date
  // No blocker field - not a concept
}
```

**What's NOT in the Queue View**:
- Prioritization logic (solver's job)
- Blocker status (doesn't exist as concept)
- Assignment tracking (foreman's job on floor)

**Current Data Gaps**:
- No solver integration (OR-Tools planned but not implemented)
- No PMDB integration (reading current status)
- No capacity model for departments
- Time estimate defaults not defined

## Code References

- `millflow/src/App.tsx:18-33` - View rendering switch
- `millflow/src/types/index.ts:115` - ViewType definition
- `millflow/src/components/CommandPalette.tsx:30-121` - Dynamic command building
- `millflow/src/hooks/useKeyboard.ts:164-205` - Multi-key navigation sequences
- `millflow/src/data/mockData.ts:66-91` - Department definitions
- `millflow/src/types/index.ts:32-49` - Node interface with assignees
- `factory-floor/src/components/dashboard/Dashboard.tsx` - Station queue reference
- `factory-floor/src/components/dashboard/StationTableNode.tsx:14-107` - Queue table UI

## Architecture Documentation

### View Navigation Pattern

```
Dashboard ← g h
    ↓ Enter
Jobs ← g j
    ↓ Enter
Job
    ↓ Enter
Sheet
```

Adding queue views creates a parallel navigation path:
```
Dashboard ← g h
    ↓ Cmd+K → search "CNC"
QueueView (CNC)
    ↓ Enter on item
Sheet (focused on selected node)
```

### Command Palette Extension Pattern

```typescript
// In CommandPalette.tsx, add to items array:
const departments = ['CNC', 'Finishing', 'Assembly', 'Install', 'Delivery'];
departments.forEach(dept => {
  items.push({
    id: `queue-${dept.toLowerCase()}`,
    type: 'navigation',
    title: `${dept} Queue`,
    subtitle: `g ${dept[0].toLowerCase()}`,  // e.g., 'g c' for CNC
    icon: ListTodo,
    action: () => {
      toggleCommandPalette();
      useAppStore.getState().goToQueue(dept);
    },
  });
});
```

### Store Extension Pattern

```typescript
// In appStore.ts:
selectedDepartment: string | null,  // Add to state

goToQueue: (department: string) => {
  set({
    view: 'queue',
    selectedDepartment: department,
    selectedIndex: 0,
    showNodeDetail: false,
  });
},
```

## Historical Context (from thoughts/)

No existing research documents found for department queues or scheduling.

## Related Research

None found.

## Open Questions

1. **Status Update Source**: How does current completion status get updated?
   - Manual check-off in MillFlow UI?
   - Automatic sync from PMDB?
   - Both (PMDB as source of truth, MillFlow for overrides)?

2. **Recalculation Trigger**: When does the solver re-run?
   - Daily batch (morning queue generation)?
   - On-demand ("recalculate now" button)?
   - Real-time as data changes?

3. **Time Estimate Ownership**: Who sets/adjusts time estimates?
   - PM sets defaults?
   - Department heads adjust per-sheet?
   - Both?

4. **Navigation Context**: When drilling from queue to sheet, should we:
   - Focus on the specific node for that department?
   - Show full sheet with node highlighted?

5. **MVP Sequencing**: For v1 without solver integration, should queue view:
   - Show placeholder data / mock queues?
   - Use simple deadline-based ordering as temporary stand-in?
   - Wait for solver integration before building view?
