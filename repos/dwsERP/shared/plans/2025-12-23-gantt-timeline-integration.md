# Gantt Timeline View Integration Plan

## Overview

Integrate the standalone Gantt timeline prototype into MillFlow as a new `TimelineView`. This adds a visual timeline/Gantt chart view alongside the existing table-based `ScheduleView`, giving users two complementary ways to visualize sheet progress across departments.

## Current State Analysis

### What Exists
- **ScheduleView** (`millflow/src/views/ScheduleView.tsx:1-396`) - Table-based view showing sheets with department operation cells
- **scheduleData.ts** (`millflow/src/data/scheduleData.ts:1-161`) - Mock data generator with department operations
- **View registration pattern** - Type in `types/index.ts`, routing in `App.tsx`, store action, keyboard shortcut
- **Gruvbox Tailwind classes** - All colors use `gruvbox-*` classes, never hex values

### What's Missing
- Gantt/timeline visualization of sheet operations over time
- Project-based grouping with expand/collapse
- Visual date-based bars for each department operation
- Today marker and weekend shading

### Key Discoveries
- ScheduleView uses local state for data (`useState(scheduleSheets)`) - acceptable pattern for prototypes (`millflow/src/views/ScheduleView.tsx:216`)
- Keyboard navigation to schedule uses `g s` sequence (`millflow/src/hooks/useKeyboard.ts:210-216`)
- ViewType union must include new view type (`millflow/src/types/index.ts:150`)
- Store needs `goToTimeline` action following `goToSchedule` pattern (`millflow/src/store/appStore.ts:459-465`)

## Desired End State

A fully functional `TimelineView` component that:
1. Displays projects with expandable/collapsible sheet rows
2. Shows department operations as colored bars on a horizontal timeline
3. Includes today marker, weekend shading, and month/week headers
4. Supports j/k navigation between rows and h/l to scroll timeline
5. Shows sheet detail popup when clicking operation bars
6. Uses keyboard shortcut `g t` to access
7. Follows all MillFlow conventions (Gruvbox colors, memo wrappers, etc.)

### Verification
- `npm run build` passes with no type errors
- View accessible via `g t` keyboard shortcut
- Timeline scrolls to today on mount
- Projects expand/collapse on click
- Department bars display with correct colors
- Sheet detail popup opens on bar click

## What We're NOT Doing

- **Replacing ScheduleView** - Timeline is an additional visualization option
- **Sharing data with ScheduleView** - Timeline uses its own data model for cleaner separation
- **Persisting expand/collapse state** - Local component state only
- **Editing operations** - View-only; editing happens in existing views
- **Real data integration** - Mock data only for this prototype

## Implementation Approach

Create the timeline view in four phases:
1. Types and data setup
2. Core timeline component with Tailwind conversion
3. App integration (routing, store, types)
4. Keyboard navigation enhancement

---

## Phase 1: Types and Data File

### Overview
Create TypeScript types and mock data file for the timeline view with project-based structure.

### Changes Required:

#### 1. Add Timeline Types
**File**: `millflow/src/types/index.ts`
**Changes**: Add types at end of file after `ScheduleDepartment`

```typescript
// Timeline View Types
export interface TimelineProject {
  id: string;
  name: string;
  delivery: string; // ISO date
  sheets: TimelineSheet[];
}

export interface TimelineSheet {
  id: string;
  desc: string;
  release: string; // ISO date
  veneer: [string, string] | null; // [start, end] dates or null if skipped
  mill: [string, string] | null;
  cnc: [string, string] | null;
  bench: [string, string] | null;
  finish: [string, string] | null;
  assembly: [string, string] | null;
  delivery: string; // ISO date
  status: number; // Slack days (negative = behind schedule)
}

export type TimelineDepartment = 'release' | 'veneer' | 'mill' | 'cnc' | 'bench' | 'finish' | 'assembly' | 'storage' | 'delivery';
```

#### 2. Update ViewType
**File**: `millflow/src/types/index.ts`
**Changes**: Add 'timeline' to ViewType union at line 150

```typescript
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'departments' | 'queue' | 'schedule' | 'timeline';
```

#### 3. Create Timeline Data File
**File**: `millflow/src/data/timelineData.ts`
**Changes**: New file with mock project/sheet data

```typescript
import type { TimelineProject } from '@/types';

// Real data extracted from Sidley Austin 4084 schedule - dates adjusted to current
export const timelineProjects: TimelineProject[] = [
  {
    id: '4084',
    name: 'Sidley Austin 2025',
    delivery: '2026-01-20',
    sheets: [
      { id: 'SK-01', desc: 'Pantry 35043', release: '2025-11-11', veneer: ['2025-11-11', '2025-11-18'], mill: ['2025-11-18', '2025-11-25'], cnc: ['2025-11-21', '2025-11-26'], bench: ['2025-12-08', '2025-12-17'], finish: ['2025-12-09', '2025-12-13'], assembly: ['2025-12-10', '2025-12-18'], delivery: '2025-12-24', status: -22 },
      { id: 'SK-04', desc: 'Wall Panel 37820', release: '2025-11-11', veneer: null, mill: ['2025-11-18', '2025-11-25'], cnc: ['2025-11-21', '2025-11-29'], bench: ['2025-12-08', '2025-12-15'], finish: ['2025-12-12', '2025-12-19'], assembly: ['2025-12-17', '2025-12-18'], delivery: '2025-12-27', status: -17 },
      { id: 'SK-03', desc: 'Pantry 67525', release: '2025-11-11', veneer: ['2025-11-14', '2025-11-17'], mill: ['2025-11-14', '2025-11-17'], cnc: ['2025-11-17', '2025-11-18'], bench: ['2025-12-03', '2025-12-08'], finish: ['2025-12-09', '2025-12-11'], assembly: ['2025-12-12', '2025-12-14'], delivery: '2025-12-22', status: -15 },
      { id: 'SK-06', desc: 'Pantry 39017', release: '2025-11-11', veneer: ['2025-11-13', '2025-11-15'], mill: ['2025-11-14', '2025-11-18'], cnc: ['2025-11-18', '2025-11-21'], bench: ['2025-12-10', '2025-12-20'], finish: ['2025-12-15', '2025-12-18'], assembly: ['2025-12-17', '2025-12-20'], delivery: '2026-01-01', status: -15 },
      { id: 'SK-07', desc: 'Pantry 37975', release: '2025-11-18', veneer: ['2025-11-21', '2025-11-25'], mill: ['2025-11-22', '2025-11-25'], cnc: ['2025-11-25', '2025-11-30'], bench: ['2025-12-09', '2025-12-16'], finish: ['2025-12-12', '2025-12-18'], assembly: ['2025-12-16', '2025-12-17'], delivery: '2026-01-15', status: 4 },
      { id: 'SK-08', desc: 'Reception 35892', release: '2025-11-18', veneer: null, mill: ['2025-11-22', '2025-11-28'], cnc: ['2025-11-28', '2025-12-02'], bench: ['2025-12-13', '2025-12-17'], finish: ['2025-12-13', '2025-12-16'], assembly: ['2025-12-17', '2025-12-20'], delivery: '2026-01-17', status: 4 },
      { id: 'SK-02', desc: 'Conference 34734', release: '2025-11-17', veneer: ['2025-11-21', '2025-11-27'], mill: ['2025-11-21', '2025-11-25'], cnc: ['2025-11-25', '2025-11-29'], bench: ['2025-12-05', '2025-12-10'], finish: ['2025-12-10', '2025-12-15'], assembly: ['2025-12-15', '2025-12-17'], delivery: '2026-01-21', status: 8 },
    ],
  },
  {
    id: '4070',
    name: 'Open AI 550 TF',
    delivery: '2026-01-25',
    sheets: [
      { id: 'SK-03', desc: 'Reception 30887', release: '2025-11-11', veneer: ['2025-11-16', '2025-11-21'], mill: ['2025-11-11', '2025-11-12'], cnc: ['2025-11-12', '2025-11-19'], bench: ['2025-12-06', '2025-12-12'], finish: ['2025-12-08', '2025-12-14'], assembly: ['2025-12-14', '2025-12-16'], delivery: '2025-12-31', status: -8 },
      { id: 'SK-09', desc: 'Huddle 36818', release: '2025-11-17', veneer: ['2025-11-23', '2025-11-29'], mill: ['2025-11-23', '2025-11-26'], cnc: ['2025-11-26', '2025-11-29'], bench: ['2025-12-05', '2025-12-11'], finish: ['2025-12-09', '2025-12-14'], assembly: ['2025-12-14', '2025-12-17'], delivery: '2026-01-11', status: -4 },
      { id: 'SK-08', desc: 'Closet 30708', release: '2025-11-17', veneer: ['2025-11-23', '2025-11-29'], mill: ['2025-11-23', '2025-11-26'], cnc: ['2025-11-26', '2025-11-29'], bench: ['2025-12-02', '2025-12-06'], finish: ['2025-12-04', '2025-12-08'], assembly: ['2025-12-08', '2025-12-11'], delivery: '2026-01-12', status: 7 },
      { id: 'SK-07', desc: 'Library 31906', release: '2025-11-18', veneer: ['2025-11-22', '2025-11-26'], mill: ['2025-11-20', '2025-11-22'], cnc: ['2025-11-22', '2025-11-23'], bench: ['2025-12-08', '2025-12-16'], finish: ['2025-12-12', '2025-12-19'], assembly: ['2025-12-17', '2025-12-19'], delivery: '2026-01-20', status: 9 },
    ],
  },
  {
    id: '4085',
    name: 'Goldman Sachs',
    delivery: '2026-02-01',
    sheets: [
      { id: 'SK-07', desc: 'Conference 35781', release: '2025-11-20', veneer: ['2025-11-23', '2025-11-26'], mill: ['2025-11-23', '2025-11-28'], cnc: ['2025-11-28', '2025-12-01'], bench: ['2025-12-05', '2025-12-10'], finish: ['2025-12-09', '2025-12-14'], assembly: ['2025-12-14', '2025-12-16'], delivery: '2026-01-02', status: -10 },
      { id: 'SK-02', desc: 'Pantry 32451', release: '2025-11-17', veneer: ['2025-11-19', '2025-11-21'], mill: ['2025-11-19', '2025-11-25'], cnc: ['2025-11-25', '2025-11-29'], bench: ['2025-12-02', '2025-12-04'], finish: ['2025-12-03', '2025-12-05'], assembly: ['2025-12-05', '2025-12-07'], delivery: '2026-01-01', status: -3 },
      { id: 'SK-06', desc: 'Restroom 36648', release: '2025-11-21', veneer: ['2025-11-24', '2025-11-27'], mill: ['2025-11-24', '2025-11-25'], cnc: null, bench: ['2025-12-08', '2025-12-16'], finish: ['2025-12-10', '2025-12-12'], assembly: ['2025-12-12', '2025-12-15'], delivery: '2026-01-17', status: 3 },
      { id: 'SK-10', desc: 'Pantry 32979', release: '2025-11-22', veneer: ['2025-11-27', '2025-12-02'], mill: ['2025-11-25', '2025-11-27'], cnc: ['2025-11-27', '2025-12-02'], bench: ['2025-12-05', '2025-12-10'], finish: ['2025-12-09', '2025-12-14'], assembly: ['2025-12-14', '2025-12-16'], delivery: '2026-01-08', status: 3 },
      { id: 'SK-01', desc: 'Wall Panel 30611', release: '2025-11-18', veneer: ['2025-11-20', '2025-11-22'], mill: ['2025-11-20', '2025-11-25'], cnc: ['2025-11-25', '2025-11-29'], bench: ['2025-12-06', '2025-12-12'], finish: ['2025-12-09', '2025-12-12'], assembly: ['2025-12-12', '2025-12-15'], delivery: '2026-01-18', status: 9 },
    ],
  },
  {
    id: '4086',
    name: 'Fenwick and West',
    delivery: '2026-01-30',
    sheets: [
      { id: 'SK-04', desc: 'Huddle 32331', release: '2025-11-11', veneer: ['2025-11-15', '2025-11-19'], mill: ['2025-11-14', '2025-11-17'], cnc: ['2025-11-17', '2025-11-21'], bench: ['2025-12-05', '2025-12-11'], finish: ['2025-12-06', '2025-12-12'], assembly: ['2025-12-12', '2025-12-15'], delivery: '2025-12-30', status: -12 },
    ],
  },
  {
    id: '4049',
    name: 'Global Relay',
    delivery: '2026-01-16',
    sheets: [
      { id: 'SK-11', desc: 'Huddle 33944', release: '2025-11-26', veneer: ['2025-12-01', '2025-12-06'], mill: ['2025-11-29', '2025-12-04'], cnc: ['2025-12-04', '2025-12-08'], bench: ['2025-12-13', '2025-12-16'], finish: ['2025-12-15', '2025-12-20'], assembly: ['2025-12-18', '2025-12-20'], delivery: '2026-01-16', status: 8 },
    ],
  },
];

// Department colors for timeline bars (Tailwind class suffixes)
export const TIMELINE_DEPT_COLORS: Record<string, string> = {
  release: 'gruvbox-fg-4',
  veneer: 'gruvbox-purple-bright',
  mill: 'gruvbox-blue-bright',
  cnc: 'gruvbox-aqua-bright',
  bench: 'gruvbox-yellow-bright',
  finish: 'gruvbox-orange-bright',
  assembly: 'gruvbox-green-bright',
  storage: 'gruvbox-fg-4',
  delivery: 'gruvbox-red-bright',
};

// Timeline configuration constants
export const TIMELINE_CONFIG = {
  DAY_WIDTH: 24,
  ROW_HEIGHT: 28,
  HEADER_HEIGHT: 60,
  LABEL_PANEL_WIDTH: 280,
};
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Types are importable in other files

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 2.

---

## Phase 2: Timeline View Component

### Overview
Create the main `TimelineView` component with all prototype functionality converted to Tailwind.

### Changes Required:

#### 1. Create TimelineView Component
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: New file - full component implementation

The component should:
- Use `memo` wrapper with named function export
- Import from `@/store/appStore`, `@/lib/styles`, `@/data/timelineData`
- Maintain local state for: `expandedProjects`, `viewMode`, `hoveredRow`, `selectedSheet`
- Calculate date range (Nov 2025 - Mar 2026) using `useMemo`
- Auto-scroll to today on mount using `useEffect`
- Render:
  - Header with title, sheet count, view mode buttons
  - Legend with department color chips
  - Split panel: left labels (280px fixed) + right timeline (scrollable)
  - Today marker (red vertical line)
  - Weekend shading (darker background)
  - Sheet detail popup modal

Key Tailwind conversions from prototype:

| Prototype Style | Tailwind Class |
|----------------|----------------|
| `background: colors.bg0` | `bg-gruvbox-bg-hard` |
| `background: colors.bg` | `bg-gruvbox-bg` |
| `background: colors.bg1` | `bg-gruvbox-bg-soft` |
| `background: colors.bg2` | `bg-gruvbox-bg-1` |
| `color: colors.fg` | `text-gruvbox-fg` |
| `color: colors.fg4` | `text-gruvbox-fg-4` |
| `color: colors.yellow` | `text-gruvbox-yellow-bright` |
| `color: colors.aqua` | `text-gruvbox-aqua-bright` |
| `border: colors.bg2` | `border-gruvbox-bg-1` |

**Important**: Only use inline `style` for dynamic positioning (`left`, `width`, `top`) based on date calculations. All colors must use Tailwind classes.

Example structure:
```typescript
import { memo, useState, useMemo, useRef, useEffect } from 'react';
import { useAppStore } from '@/store/appStore';
import { cn } from '@/lib/styles';
import { timelineProjects, TIMELINE_DEPT_COLORS, TIMELINE_CONFIG } from '@/data/timelineData';
import type { TimelineProject, TimelineSheet } from '@/types';

const { DAY_WIDTH, ROW_HEIGHT, HEADER_HEIGHT, LABEL_PANEL_WIDTH } = TIMELINE_CONFIG;

export const TimelineView = memo(function TimelineView() {
  const { selectedIndex } = useAppStore();
  const containerRef = useRef<HTMLDivElement>(null);
  const timelineRef = useRef<HTMLDivElement>(null);

  const [expandedProjects, setExpandedProjects] = useState<Record<string, boolean>>(
    () => Object.fromEntries(timelineProjects.map(p => [p.id, true]))
  );
  const [viewMode, setViewMode] = useState<'day' | 'week' | 'month'>('week');
  const [hoveredRow, setHoveredRow] = useState<number | null>(null);
  const [selectedSheet, setSelectedSheet] = useState<TimelineSheet | null>(null);

  // Date range calculation
  const dateRange = useMemo(() => {
    const start = new Date('2025-11-01');
    const end = new Date('2026-03-31');
    const days: Date[] = [];
    let current = new Date(start);
    while (current <= end) {
      days.push(new Date(current));
      current.setDate(current.getDate() + 1);
    }
    return { start, end, days };
  }, []);

  // Helper to get X position for a date
  const getDateX = (dateStr: string) => {
    const date = new Date(dateStr);
    const diffDays = Math.floor((date.getTime() - dateRange.start.getTime()) / (1000 * 60 * 60 * 24));
    return diffDays * DAY_WIDTH;
  };

  // Flatten rows for rendering
  const rows = useMemo(() => {
    const result: Array<{ type: 'project' | 'sheet'; data: TimelineProject | TimelineSheet; project?: TimelineProject }> = [];
    timelineProjects.forEach(project => {
      result.push({ type: 'project', data: project });
      if (expandedProjects[project.id]) {
        project.sheets.forEach(sheet => {
          result.push({ type: 'sheet', data: sheet, project });
        });
      }
    });
    return result;
  }, [expandedProjects]);

  // Auto-scroll to today on mount
  useEffect(() => {
    if (timelineRef.current) {
      const todayX = getDateX(new Date().toISOString().split('T')[0]);
      timelineRef.current.scrollLeft = Math.max(0, todayX - 300);
    }
  }, []);

  // ... rest of component
});

export default TimelineView;
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Component renders without errors when imported
- [ ] Timeline displays projects and sheets correctly
- [ ] Department bars appear with correct colors
- [ ] Today marker visible
- [ ] Weekend shading visible
- [ ] Projects expand/collapse on click
- [ ] Sheet detail popup appears on bar click

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual testing confirmation before proceeding to Phase 3.

---

## Phase 3: App Integration

### Overview
Wire up the TimelineView to the app's routing, store, and keyboard navigation.

### Changes Required:

#### 1. Add View Import and Route
**File**: `millflow/src/App.tsx`
**Changes**: Add import and case to renderView switch

```typescript
// Add import after line 12 (after ScheduleView import)
import TimelineView from '@/views/TimelineView';

// Add case in renderView switch after 'schedule' case (around line 35)
case 'timeline':
  return <TimelineView />;
```

#### 2. Add Store Action
**File**: `millflow/src/store/appStore.ts`
**Changes**: Add goToTimeline action

Add to interface (around line 82):
```typescript
goToTimeline: () => void;
```

Add implementation (after goToSchedule, around line 466):
```typescript
goToTimeline: () => {
  set({
    view: 'timeline',
    selectedIndex: 0,
    showNodeDetail: false,
  });
},
```

#### 3. Add Keyboard Shortcut
**File**: `millflow/src/hooks/useKeyboard.ts`
**Changes**: Add `g t` sequence handler

Add `goToTimeline` to destructured imports (around line 31):
```typescript
goToTimeline,
```

Add handler after `g s` handler (around line 216):
```typescript
// g t - go to timeline
if (sequence === 'g t') {
  event.preventDefault();
  resetSequence();
  goToTimeline();
  return;
}
```

Add to dependency array (around line 512):
```typescript
goToTimeline,
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`
- [ ] Dev server starts: `cd millflow && npm run dev`

#### Manual Verification:
- [ ] `g t` keyboard shortcut navigates to timeline view
- [ ] Timeline view renders correctly in the app
- [ ] Escape key goes back from timeline view
- [ ] Command palette can access timeline (if applicable)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual testing confirmation before proceeding to Phase 4.

---

## Phase 4: Keyboard Navigation Enhancement

### Overview
Add j/k row navigation and h/l timeline scrolling within the TimelineView.

### Changes Required:

#### 1. Add Timeline-Specific Keyboard Handlers
**File**: `millflow/src/hooks/useKeyboard.ts`
**Changes**: Add handlers for h/l scrolling in timeline view

In the h/ArrowLeft handler (around line 363):
```typescript
case 'h':
case 'ArrowLeft':
  if (view === 'sheet') {
    event.preventDefault();
    navigateLeft();
  } else if (view === 'timeline') {
    // Scroll timeline left (handled by component)
    event.preventDefault();
    // Dispatch custom event for timeline to handle
    window.dispatchEvent(new CustomEvent('timeline-scroll', { detail: { direction: 'left' } }));
  }
  break;
```

In the l/ArrowRight handler (around line 371):
```typescript
case 'l':
case 'ArrowRight':
  if (view === 'sheet') {
    event.preventDefault();
    navigateRight();
  } else if (view === 'timeline') {
    // Scroll timeline right (handled by component)
    event.preventDefault();
    window.dispatchEvent(new CustomEvent('timeline-scroll', { detail: { direction: 'right' } }));
  }
  break;
```

#### 2. Listen for Scroll Events in TimelineView
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: Add event listener for custom scroll events

```typescript
// Add in useEffect for keyboard scrolling
useEffect(() => {
  const handleTimelineScroll = (e: CustomEvent<{ direction: 'left' | 'right' }>) => {
    if (!timelineRef.current) return;
    const scrollAmount = DAY_WIDTH * 7; // Scroll one week at a time
    if (e.detail.direction === 'left') {
      timelineRef.current.scrollLeft -= scrollAmount;
    } else {
      timelineRef.current.scrollLeft += scrollAmount;
    }
  };

  window.addEventListener('timeline-scroll', handleTimelineScroll as EventListener);
  return () => window.removeEventListener('timeline-scroll', handleTimelineScroll as EventListener);
}, []);
```

#### 3. Add Row Selection State
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: Use selectedIndex from store for row highlighting

The existing j/k handlers in useKeyboard already update `selectedIndex`. TimelineView should:
- Highlight the row at `selectedIndex`
- Auto-scroll selected row into view (already implemented in Phase 2)

```typescript
// In row rendering
<div
  data-selected={idx === selectedIndex}
  className={cn(
    'h-7 relative border-b border-gruvbox-bg-1',
    idx === selectedIndex && 'bg-gruvbox-bg-1',
    hoveredRow === idx && 'bg-gruvbox-bg-soft/50'
  )}
>
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] j/k moves selection between rows
- [ ] h/l scrolls timeline horizontally
- [ ] Selected row is visually highlighted
- [ ] Selected row auto-scrolls into view

**Implementation Note**: After completing this phase, the integration is complete.

---

## Testing Strategy

### Unit Tests
Not applicable for this prototype phase.

### Integration Tests
Not applicable for this prototype phase.

### Manual Testing Steps

1. **Navigation**
   - Start app with `npm run dev`
   - Press `g t` to navigate to timeline view
   - Verify header shows correct sheet count
   - Press Escape to go back

2. **Timeline Interactions**
   - Click project row to collapse/expand
   - Verify sheet rows hide/show
   - Click department bar to open sheet detail popup
   - Click outside popup or Close button to dismiss

3. **Visual Elements**
   - Verify today marker (red line) is visible
   - Verify weekend columns have darker background
   - Verify month/week headers are correct
   - Verify slack badges show correct colors (red for negative, green for positive)

4. **Keyboard Navigation**
   - Use j/k to move between rows
   - Use h/l to scroll timeline
   - Verify selection highlight follows
   - Verify auto-scroll keeps selection visible

## Performance Considerations

- `useMemo` for date range calculation (runs once)
- `useMemo` for flattened rows (recalculates only when expandedProjects changes)
- Virtualization NOT implemented - acceptable for ~200 sheets, may need if scaling significantly

## Migration Notes

Not applicable - this is a new view addition with no data migration required.

## References

- Research document: `thoughts/shared/research/2025-12-23-gantt-timeline-integration.md`
- ScheduleView reference: `millflow/src/views/ScheduleView.tsx:1-396`
- View registration pattern: `millflow/src/App.tsx:20-39`
- Store action pattern: `millflow/src/store/appStore.ts:459-465`
- Keyboard handling: `millflow/src/hooks/useKeyboard.ts:210-216`
