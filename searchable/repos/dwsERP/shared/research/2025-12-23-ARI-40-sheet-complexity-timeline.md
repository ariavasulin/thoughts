# ARI-40 Research: Sheet Complexity Representation on Timeline

**Date**: 2025-12-23
**Issue**: [ARI-40](https://linear.app/ariav/issue/ARI-40/sheet-complexity-representation-on-timeline)

## Executive Summary

The timeline view uses a **completely separate, simplified data model** from the main app's DAG-based sheet structure. It cannot currently represent complex flows like same-department revisits, multiple deliveries, or asymmetric parallel branches.

---

## 1. Current Data Model Analysis

**Timeline Types** (`types/index.ts:184-206`):

```typescript
interface TimelineSheet {
  id: string;
  desc: string;
  release: string;                      // Single release date
  veneer: [string, string] | null;      // Fixed department slots
  mill: [string, string] | null;
  cnc: [string, string] | null;
  bench: [string, string] | null;
  finish: [string, string] | null;
  assembly: [string, string] | null;
  delivery: string;                     // Single delivery date
  status: number;                       // Slack days
}

type TimelineDepartment = 'release' | 'veneer' | 'mill' | 'cnc' | 'bench' | 'finish' | 'assembly' | 'storage' | 'delivery';
```

**Key Limitations:**
- **Fixed department slots**: Each department appears exactly once (or is null/skipped)
- **Single delivery**: Only one delivery date per sheet
- **No revisits**: Cannot model "CNC → Bench → CNC" flows
- **Simple date ranges**: Each operation is `[start, end]` with no sub-items

**Compare to Main App's Sheet Model** (`types/index.ts:36-45`):

```typescript
interface Sheet {
  id: string;
  nodes: Node[];  // Array of arbitrary DAG nodes
}

interface Node {
  id: string;
  type: NodeType;  // 'shop' | 'external' | 'delivery' | etc.
  children: string[];
  parents: string[];
  // ... duration, status, etc.
}
```

The main app model is **fully flexible** — any number of nodes, any dependencies, any department can appear multiple times.

---

## 2. Mock Data Structure

**`timelineData.ts`** contains hardcoded sheets with the fixed-column format:

```typescript
{
  id: 'SK-01',
  desc: 'Pantry 35043',
  release: '2025-11-11',
  veneer: ['2025-11-11', '2025-11-18'],  // One veneer operation
  mill: ['2025-11-18', '2025-11-25'],
  cnc: ['2025-11-21', '2025-11-26'],
  bench: ['2025-12-08', '2025-12-17'],
  finish: ['2025-12-09', '2025-12-13'],
  assembly: ['2025-12-10', '2025-12-18'],
  delivery: '2025-12-24',                // One delivery
  status: -22
}
```

**What's missing:**
- No example of same-dept revisits
- No multiple deliveries per sheet
- No sub-items or items within a department phase

---

## 3. Rendering Logic

**`TimelineView.tsx`** renders one bar per department:

```typescript
// Lines 355-385
{DEPARTMENTS.map(dept => {
  const sheet = row.data as TimelineSheet;
  const op = sheet[dept];  // Direct property access
  if (!op) return null;

  return <div style={{ left: x, width }} />;  // Single bar
})}
```

**No support for:**
- Multiple bars per department (sub-rows)
- Nested items within a phase
- Breaking a sheet's row into multiple lines

Each sheet gets exactly one row with one bar per department.

---

## 4. DAG Integration

**There is NO connection between the timeline and the DAG model.**

- `TimelineView` imports `timelineData.ts` directly
- The main app's `Sheet.nodes[]` array with DAG relationships is completely separate
- `dag.ts` functions (`buildLevels`, `buildRegions`) are not used in timeline

The timeline is a standalone prototype with mock data, not derived from the flow editor's DAG structure.

---

## 5. What Would Be Needed for Complexity

To support complex flows, the timeline would need:

### Option A: Extend TimelineSheet with arrays

```typescript
interface TimelineSheet {
  operations: Array<{
    department: TimelineDepartment;
    start: string;
    end: string;
    label?: string;
  }>;
  deliveries: Array<{ date: string; label: string }>;
}
```

### Option B: Derive timeline data from DAG

- Convert `Sheet.nodes[]` → timeline operations
- Use node types and department assignments
- Preserve DAG ordering for positioning

### Option C: Sub-rows for complex sheets

- Sheets with >1 operation per dept expand into multiple visual rows
- Parent row for the sheet, child rows for sub-operations

---

## 6. Schedule View Comparison

There's also a `ScheduleView` (`ScheduleView.tsx`) with `ScheduleSheet` type:

```typescript
interface ScheduleSheet {
  operations: Record<string, ScheduleOperation>;  // Keyed by department
}

interface ScheduleOperation {
  duration: number;
  status: ScheduleOperationStatus;
  dependsOn: string[];
}
```

This is slightly more flexible (operations keyed by department ID with dependencies), but still assumes one operation per department. It's a table view, not a Gantt.

---

## Summary Table

| Feature | TimelineSheet | Main Sheet (DAG) |
|---------|---------------|------------------|
| Same-dept revisits | No | Yes |
| Multiple deliveries | No | Yes |
| Parallel branches | Implicit | Explicit |
| Sub-items | No | Yes (nodes) |
| Connected to flow editor | No | N/A |
| Rendering | Single row | DAGRenderer |

---

## Files Examined

- `millflow/src/types/index.ts` - TimelineProject, TimelineSheet, TimelineDepartment types
- `millflow/src/data/timelineData.ts` - Mock timeline data
- `millflow/src/views/TimelineView.tsx` - Timeline rendering logic
- `millflow/src/lib/dag.ts` - DAG/flow structure (not used in timeline)
- `millflow/src/views/ScheduleView.tsx` - Comparison with schedule view
- `millflow/src/data/scheduleData.ts` - Schedule mock data
