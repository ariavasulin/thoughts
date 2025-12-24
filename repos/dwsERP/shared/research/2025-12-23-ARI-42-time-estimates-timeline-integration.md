---
date: 2025-12-23T16:16:09-08:00
researcher: ariasulin
git_commit: 2660d75535156917eaa5c961be1e74917e090158
branch: main
repository: dwsERP
topic: "Time Estimates Integration on Timeline"
tags: [research, codebase, millflow, timeline, time-estimates, durations, ARI-42]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: ARI-42 Time Estimates Integration on Timeline

**Date**: 2025-12-23T16:16:09-08:00
**Researcher**: ariasulin
**Git Commit**: 2660d75535156917eaa5c961be1e74917e090158
**Branch**: main
**Repository**: dwsERP

## Research Question

How do time estimates work in the current system and what is their relationship to the timeline view?

Focus areas:
1. Time estimate types and where they are defined
2. Default durations per operation
3. How timeline bar widths are calculated
4. How the flow editor handles time estimates per step
5. Whether the store tracks time estimate data

## Summary

The system has **two separate time estimate models** that are currently disconnected:

1. **Node-level durations** (`duration_estimate`/`duration_actual` in hours) - Used in the flow editor and node detail view but rarely populated in mock data
2. **Schedule operation durations** (in days) - Used in ScheduleView and calculated with dependencies

The **TimelineView uses date ranges directly** (start/end dates) rather than calculating from durations. Bar widths are computed by subtracting start date X position from end date X position.

**No centralized default duration configuration exists.** The scheduleData.ts file has inline duration arrays for generating mock data, but these are not exposed as configuration.

---

## Detailed Findings

### 1. Time Estimate Types

Two distinct duration models exist in `millflow/src/types/index.ts`:

#### Node Interface (lines 47-64)

```typescript
export interface Node {
  // ...
  duration_estimate?: number;  // line 57 - optional, in HOURS
  duration_actual?: number;    // line 58 - optional, in HOURS
  // ...
}
```

- Both fields are optional
- Values represent hours (not days)
- Used for individual flow nodes

#### ScheduleOperation Interface (lines 161-165)

```typescript
export interface ScheduleOperation {
  duration: number;           // in DAYS
  status: ScheduleOperationStatus;
  dependsOn: string[];
}
```

- Duration is required (not optional)
- Values represent days (not hours)
- Includes dependency information

#### TimelineSheet Interface (lines 192-204)

```typescript
export interface TimelineSheet {
  id: string;
  desc: string;
  release: string;
  veneer: [string, string] | null;   // [startDate, endDate]
  mill: [string, string] | null;
  cnc: [string, string] | null;
  bench: [string, string] | null;
  finish: [string, string] | null;
  assembly: [string, string] | null;
  delivery: string;
  status: number;  // slack days
}
```

- Uses date ranges, NOT duration values
- Each department operation is a tuple of ISO dates
- Null indicates skipped operation

### 2. Default Durations

**No centralized configuration file for default durations exists.**

Default durations are only defined inline in the schedule data generator:

`millflow/src/data/scheduleData.ts:65-101`:

```typescript
const ops: Record<string, { duration: number; ... }> = {
  veneer: {
    duration: [0, 2, 3, 4, 5, 6][Math.floor(Math.random() * 6)],  // 0-6 days
    dependsOn: []
  },
  mill: {
    duration: [1, 2, 2, 3, 3, 4, 5, 6][Math.floor(Math.random() * 8)],  // 1-6 days, weighted 2-3
    dependsOn: []
  },
  cnc: {
    duration: [1, 2, 2, 3, 3, 4, 5][Math.floor(Math.random() * 7)],  // 1-5 days, weighted 2-3
    dependsOn: ['mill']
  },
  bench: {
    duration: [2, 3, 4, 5, 5, 6, 7, 8, 10][Math.floor(Math.random() * 9)],  // 2-10 days, weighted 5
    dependsOn: ['veneer', 'cnc']
  },
  finish: {
    duration: [2, 3, 4, 5, 5, 6, 7][Math.floor(Math.random() * 7)],  // 2-7 days, weighted 5
    dependsOn: ['bench']
  },
  assembly: {
    duration: [1, 2, 2, 3, 3, 4][Math.floor(Math.random() * 6)],  // 1-4 days, weighted 2-3
    dependsOn: ['finish']
  },
  delivery: {
    duration: 1,  // Fixed 1 day
    dependsOn: ['assembly']
  }
}
```

These values are used for mock data generation, not exposed as configurable defaults.

The `operations.ts` file (`millflow/src/data/operations.ts`) only contains operation names and types for the command palette - **no duration defaults**:

```typescript
export interface OperationSuggestion {
  id: string;
  name: string;
  type: NodeType;
  keywords?: string[];  // No duration field
}
```

### 3. Timeline Bar Width Calculation

`millflow/src/views/TimelineView.tsx:70-74` - Date position helper:

```typescript
const getDateX = (dateStr: string) => {
  const date = new Date(dateStr);
  const diffDays = Math.floor((date.getTime() - dateRange.start.getTime()) / (1000 * 60 * 60 * 24));
  return diffDays * DAY_WIDTH;
};
```

`millflow/src/views/TimelineView.tsx:360-367` - Bar rendering:

```typescript
const startDate = op[0];  // First element of [start, end] tuple
const endDate = op[1];    // Second element
const x = getDateX(startDate);
const width = Math.max(
  DAY_WIDTH * 0.8,  // Minimum width
  getDateX(endDate) - x + DAY_WIDTH  // End date minus start date
);
```

**Key observation**: Bar widths are calculated purely from date ranges. There is no conversion from duration values to dates - the TimelineView consumes pre-computed date ranges.

Configuration constants in `millflow/src/data/timelineData.ts:74-79`:

```typescript
export const TIMELINE_CONFIG = {
  DAY_WIDTH: 24,      // pixels per day
  ROW_HEIGHT: 28,     // pixels per row
  HEADER_HEIGHT: 60,  // header area
  LABEL_PANEL_WIDTH: 280,  // left label panel
};
```

### 4. Flow Editor Time Estimates

The flow editor (DAG editor) **does not actively handle time estimates**.

- Nodes can have `duration_estimate` set but there is no UI for editing it
- The NodeDetailContent.tsx displays duration if present (`millflow/src/components/node/NodeDetailContent.tsx:105-113`)
- No keyboard shortcuts exist for setting durations
- Templates in `templates.ts` do not include durations

The only display of duration in the flow editor is passive (read-only when present):

```typescript
{node.duration_estimate && (
  <div className="flex items-center gap-1.5 px-2 py-1 ...">
    <Clock size={12} className="text-gruvbox-blue-bright" />
    <span className="text-gruvbox-fg-3">~{node.duration_estimate} hours</span>
  </div>
)}
```

### 5. Store State

`millflow/src/store/appStore.ts` - **No time estimate actions exist**

The store:
- Stores sheets/nodes with their duration fields
- Has `updateNode` action that can modify any node field including `duration_estimate`
- Does NOT have dedicated actions for time estimate management
- Does NOT track default durations
- Does NOT calculate schedules from durations

The node update action (`appStore.ts:991-1005`) is generic:

```typescript
updateNode: (sheetId, nodeId, updates) => {
  // ... can update duration_estimate via updates object
}
```

### 6. Mock Data Duration Usage

`millflow/src/data/mockData.ts` contains some hardcoded durations:

- Line 126: CNC node with `duration_actual: 4` (completed work)
- Line 139: Assembly node with `duration_actual: 8`
- Line 209: Finishing node with `duration_estimate: 6`
- Line 230: QC node with `duration_estimate: 1`
- Line 254: Install node with `duration_estimate: 4`

Most nodes in mock data do NOT have duration values set.

---

## Architecture Documentation

### Current Data Flow

```
TimelineView                 ScheduleView              Flow Editor (DAG)
     |                            |                          |
     v                            v                          v
TimelineSheet               ScheduleOperation               Node
(date ranges)                 (duration days)           (duration hours)
     |                            |                          |
     v                            v                          v
timelineData.ts             scheduleData.ts              mockData.ts
(hardcoded dates)           (random durations)          (sparse durations)
```

### Unit Inconsistency

- Node durations: **hours**
- Schedule durations: **days**
- Timeline: **dates directly**

### Missing Pieces for Integration

1. **No duration-to-date conversion** - Would need to calculate start/end dates from durations and dependencies
2. **No centralized defaults** - Each data file defines its own
3. **No UI for editing durations** - Duration fields exist but aren't exposed
4. **Different units** - Nodes use hours, schedules use days

---

## Code References

- `millflow/src/types/index.ts:57-58` - Node duration fields
- `millflow/src/types/index.ts:162` - ScheduleOperation duration
- `millflow/src/types/index.ts:192-204` - TimelineSheet with date ranges
- `millflow/src/views/TimelineView.tsx:70-74` - getDateX position calculation
- `millflow/src/views/TimelineView.tsx:360-367` - Bar width calculation
- `millflow/src/data/scheduleData.ts:65-101` - Duration arrays for mock data
- `millflow/src/data/timelineData.ts:74-79` - Timeline display constants
- `millflow/src/data/operations.ts:1-55` - Operation suggestions (no durations)
- `millflow/src/components/node/NodeDetailContent.tsx:105-113` - Duration display
- `millflow/src/store/appStore.ts:991-1005` - updateNode action

---

## Historical Context (from thoughts/)

- `thoughts/shared/research/2025-12-23-gantt-timeline-integration.md` - Recent integration of TimelineView, noted difference between date ranges and durations
- `thoughts/shared/research/2025-12-22-millflow-product-scope.md` - Documents that scheduling optimization (PyJobShop/OR-Tools) is NOT BUILT
- `thoughts/shared/plans/2025-12-19-assignment-constraint-system.md` - Mentions time estimates for future constraint system

---

## Open Questions

1. **Should durations convert to dates?** - For timeline integration, need to decide if durations should be stored or if date ranges should be computed from durations + dependencies
2. **What are the correct default durations?** - The current mock data uses weighted random values; what are the real defaults for each operation type?
3. **Hours vs days?** - Should the system standardize on one unit?
4. **Where should defaults live?** - Should `operations.ts` include default durations per operation type?
5. **Integration with OR-Tools** - Will PyJobShop compute optimal dates from durations, making the manual date ranges obsolete?
