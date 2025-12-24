---
date: 2025-12-23T16:14:35-08:00
researcher: ariasulin
git_commit: 2660d75535156917eaa5c961be1e74917e090158
branch: main
repository: dwsERP
topic: "ARI-43: Department/Step Visibility Filter"
tags: [research, codebase, timeline, filtering, departments]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: ARI-43 Department/Step Visibility Filter

**Date**: 2025-12-23T16:14:35-08:00
**Researcher**: ariasulin
**Git Commit**: 2660d75535156917eaa5c961be1e74917e090158
**Branch**: main
**Repository**: dwsERP

## Research Question

Document current filtering capabilities in the timeline and related views. Focus areas:
1. Current filters - Does TimelineView have any filtering UI? What can be filtered?
2. Filter state - Where is filter state stored (local useState vs Zustand store)?
3. Other view filters - How do other views (Dashboard, Jobs) handle filtering? Are there patterns to follow?
4. Department list rendering - Where is the list of departments rendered that could become toggleable?

## Summary

**TimelineView has no department/step filtering capability**. It displays all departments without any toggle or filter mechanism. The only filtering-like behavior is view mode switching (day/week/month) and project expand/collapse.

The codebase uses **local useState for UI-only state** (view modes, sort options) and **Zustand for app-wide state** (selection, navigation, data). Filtering should likely use local useState for the timeline view, following the pattern in ScheduleView.

## Detailed Findings

### 1. Current TimelineView Filtering Capabilities

**Location**: `millflow/src/views/TimelineView.tsx`

**Current UI Controls (lines 144-159)**:
- **View mode toggle**: `day | week | month` - stored in local `useState`
- **No department filtering exists**

**Department list is hardcoded (line 10)**:
```typescript
const DEPARTMENTS = ['veneer', 'mill', 'cnc', 'bench', 'finish', 'assembly'] as const;
```

This constant controls:
1. Legend display (lines 163-170) - iterates `TIMELINE_DEPT_COLORS`
2. Operation bar rendering (lines 355-386) - iterates `DEPARTMENTS`
3. Detail popup (lines 466-491) - hardcoded array

**Expand/collapse** is per-project, not per-department (lines 22-24, 93-95):
```typescript
const [expandedProjects, setExpandedProjects] = useState<Record<string, boolean>>(
  () => Object.fromEntries(timelineProjects.map(p => [p.id, true]))
);
```

### 2. State Management Patterns

**Local useState is used for**:
- `TimelineView`: viewMode, expandedProjects, hoveredRow, selectedSheet
- `ScheduleView`: sheets, sortBy (line 217)
- `DashboardSection`: expandedSections via Zustand (but it's UI state)

**Zustand store (`appStore.ts`) is used for**:
- Navigation: view, selectedJobId, selectedSheetId, selectedNodeId, selectedIndex
- UI modals: showCommandPalette, showNodeDetail, showHelpModal
- Data: jobs, sheets, people, dashboardState
- expandedSections (lines 46, 168) - section expand/collapse on dashboard

**Pattern**: View-specific UI state (sort order, view mode) uses local useState. App-wide state (selection, navigation, data) uses Zustand.

### 3. Other Views Filtering Patterns

#### ScheduleView (best example for filtering)
**Location**: `millflow/src/views/ScheduleView.tsx:217, 251-260`

Sort toggle with local state:
```typescript
const [sortBy, setSortBy] = useState<'priority' | 'project' | 'install'>('priority');

const sortedSheets = useMemo(() => {
  return [...activeSheets].sort((a, b) => {
    if (sortBy === 'priority') return getPriorityScore(a) - getPriorityScore(b);
    // ...
  });
}, [activeSheets, sortBy]);
```

UI buttons (lines 280-293):
```typescript
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
```

#### DashboardView
**Location**: `millflow/src/views/DashboardView.tsx`

Uses Zustand's `expandedSections` for section visibility, but this is expand/collapse, not filtering. The sections are fixed categories (your_move, at_risk, waiting_on_others).

#### DashboardSection Component
**Location**: `millflow/src/components/dashboard/DashboardSection.tsx:19-20`

```typescript
const { expandedSections, toggleSection } = useAppStore();
const isExpanded = expandedSections.has(id);
```

This pattern toggles visibility via a Set in Zustand store.

### 4. Department List Rendering Locations

**TimelineView legend (lines 163-170)**:
```typescript
<div className="flex items-center gap-4 px-4 py-2 border-b border-gruvbox-bg-1 bg-gruvbox-bg text-[10px]">
  {Object.entries(TIMELINE_DEPT_COLORS).map(([dept, colorClass]) => (
    <div key={dept} className="flex items-center gap-1">
      <div className={cn('w-3 h-2 rounded-sm', colorClass)} />
      <span className="text-gruvbox-fg-4 capitalize">{dept}</span>
    </div>
  ))}
</div>
```

**TimelineView sheet rows (lines 355-386)**:
```typescript
{DEPARTMENTS.map(dept => {
  const sheet = row.data as TimelineSheet;
  const op = sheet[dept];
  if (!op) return null;
  // ... render bar
})}
```

**Related constants**:
- `TIMELINE_DEPT_COLORS` in `millflow/src/data/timelineData.ts:61-71` - 9 departments (includes release, storage, delivery which aren't in DEPARTMENTS array)
- `DEPARTMENTS` in `millflow/src/views/TimelineView.tsx:10` - 6 departments (veneer, mill, cnc, bench, finish, assembly)
- `SCHEDULE_DEPARTMENTS` in `millflow/src/data/scheduleData.ts` - used by ScheduleView

## Code References

- `millflow/src/views/TimelineView.tsx:10` - DEPARTMENTS constant
- `millflow/src/views/TimelineView.tsx:22-24` - expandedProjects state
- `millflow/src/views/TimelineView.tsx:25` - viewMode state
- `millflow/src/views/TimelineView.tsx:144-159` - view mode toggle buttons
- `millflow/src/views/TimelineView.tsx:163-170` - department legend
- `millflow/src/views/TimelineView.tsx:355-386` - department bar rendering
- `millflow/src/views/ScheduleView.tsx:217` - sortBy state (pattern to follow)
- `millflow/src/views/ScheduleView.tsx:280-293` - sort toggle buttons (pattern to follow)
- `millflow/src/store/appStore.ts:46` - expandedSections type
- `millflow/src/store/appStore.ts:168` - expandedSections initial value
- `millflow/src/data/timelineData.ts:61-71` - TIMELINE_DEPT_COLORS
- `millflow/src/data/departments.ts:5-92` - DEPARTMENTS array (12 departments)

## Architecture Documentation

### State Management Patterns

1. **View-local UI state** (sort order, view mode, department visibility) - Use `useState`
2. **Cross-view state** (selection, navigation) - Use Zustand store
3. **Persistent preferences** - Use `localStorage` (see `sidebarWidth` in appStore.ts)

### Filtering Implementation Pattern

Based on ScheduleView, a department filter would:
1. Add local state: `const [visibleDepts, setVisibleDepts] = useState<Set<string>>(new Set(DEPARTMENTS))`
2. Filter in useMemo before rendering bars
3. Render toggle buttons in header/legend area

### Department Data Sources

Two different department lists exist:
1. **Timeline-specific**: `TimelineView.tsx:10` - 6 departments
2. **App-wide**: `departments.ts` - 12 departments (9 priority queues + 3 status lists)

These serve different purposes:
- TimelineView uses a subset for Gantt-style visualization
- DepartmentsView/QueueView use the full list for work queues

## Open Questions

1. Should department visibility be persisted to localStorage like sidebarWidth?
2. Should the legend items be clickable toggles, or should there be a separate filter panel?
3. Should visibility be per-view (Timeline only) or shared across views?
