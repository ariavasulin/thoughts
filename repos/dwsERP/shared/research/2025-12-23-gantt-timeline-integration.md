---
date: 2025-12-23T00:50:19-08:00
researcher: ariasulin
git_commit: 8680d53bc72ec5b434883bc787d132513b97a151
branch: main
repository: dwsERP
topic: "Integrating Gantt Timeline Component with MillFlow"
tags: [research, codebase, millflow, gantt, timeline, integration]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: Integrating Gantt Timeline Component with MillFlow

**Date**: 2025-12-23T00:50:19-08:00
**Researcher**: ariasulin
**Git Commit**: 8680d53bc72ec5b434883bc787d132513b97a151
**Branch**: main
**Repository**: dwsERP

## Research Question

How to integrate a standalone Gantt/timeline chart component (using inline styles with Gruvbox colors) with the existing MillFlow application.

## Summary

The MillFlow app already has a similar `ScheduleView` prototype that can serve as a reference. Integration requires:
1. Converting inline hex colors to Tailwind gruvbox-* classes
2. Registering the view in App.tsx and types
3. Adapting the data model or creating a compatible mock data file
4. Adding keyboard navigation support

## Detailed Findings

### Existing ScheduleView Reference

MillFlow already has a schedule prototype at `millflow/src/views/ScheduleView.tsx:1-396` that demonstrates the same patterns needed:

- **Header with icon, title, and metadata** (lines 266-298)
- **Legend showing color meanings** (lines 300-316)
- **Scrollable table with sticky headers** (lines 318-390)
- **Local state for data** (line 216: `const [sheets, setSheets] = useState(scheduleSheets)`)
- **Sort controls** (lines 278-297)
- **Interactive cells with status cycling** (lines 131-136)

### Data Model Comparison

**Provided Gantt Component Data**:
```typescript
{
  id: '4084',
  name: 'Sidley Austin 2025',
  delivery: '2026-01-20',
  sheets: [{
    id: 'SK-01',
    desc: 'Pantry 35043',
    release: '2025-11-11',
    veneer: ['2025-11-11', '2025-11-18'],  // [start, end]
    mill: ['2025-11-18', '2025-11-25'],
    cnc: ['2025-11-21', '2025-11-26'],
    bench: ['2025-12-08', '2025-12-17'],
    finish: ['2025-12-09', '2025-12-13'],
    assembly: ['2025-12-10', '2025-12-18'],
    delivery: '2025-12-24',
    status: -22  // Slack days (negative = behind)
  }]
}
```

**Existing ScheduleSheet Type** (`millflow/src/types/index.ts:167-177`):
```typescript
interface ScheduleSheet {
  id: string;
  pid: string;
  project: string;
  sheet: string;
  description: string;
  releaseDate: string | null;
  targetInstall: string;
  notes: string;
  operations: Record<string, ScheduleOperation>;
}
```

Key differences:
- Gantt uses date ranges `[start, end]` per department
- ScheduleSheet uses duration + status + dependencies
- Gantt calculates slack as days ahead/behind
- ScheduleSheet calculates slack via `getPriorityScore()`

### Color Mapping Required

The provided component uses inline hex styles. These must be converted to Tailwind gruvbox classes:

| Hex Value | Gruvbox Class |
|-----------|---------------|
| `#282828` (bg) | `bg-gruvbox-bg` |
| `#1d2021` (bg0) | `bg-gruvbox-bg-hard` |
| `#3c3836` (bg1) | `bg-gruvbox-bg-soft` |
| `#504945` (bg2) | `bg-gruvbox-bg-1` |
| `#665c54` (bg3) | `bg-gruvbox-bg-2` |
| `#ebdbb2` (fg) | `text-gruvbox-fg` |
| `#fbf1c7` (fg0) | `text-gruvbox-fg-0` |
| `#d5c4a1` (fg2) | `text-gruvbox-fg-2` |
| `#a89984` (fg4) | `text-gruvbox-fg-4` |
| `#fb4934` (red) | `text-gruvbox-red-bright` |
| `#b8bb26` (green) | `text-gruvbox-green-bright` |
| `#fabd2f` (yellow) | `text-gruvbox-yellow-bright` |
| `#83a598` (blue) | `text-gruvbox-blue-bright` |
| `#d3869b` (purple) | `text-gruvbox-purple-bright` |
| `#8ec07c` (aqua) | `text-gruvbox-aqua-bright` |
| `#fe8019` (orange) | `text-gruvbox-orange-bright` |

### Integration Steps

#### Step 1: Create Data File

Create `millflow/src/data/ganttData.ts` following the pattern from `scheduleData.ts`:

```typescript
export interface GanttProject {
  id: string;
  name: string;
  delivery: string;
  sheets: GanttSheet[];
}

export interface GanttSheet {
  id: string;
  desc: string;
  release: string;
  veneer: [string, string] | null;
  mill: [string, string] | null;
  cnc: [string, string] | null;
  bench: [string, string] | null;
  finish: [string, string] | null;
  assembly: [string, string] | null;
  delivery: string;
  status: number;  // Slack days
}

export const ganttProjects: GanttProject[] = [
  // Data here
];
```

#### Step 2: Create View Component

Create `millflow/src/views/TimelineView.tsx`:

1. Import memo, useState, useMemo, useRef, useEffect from 'react'
2. Import useAppStore from '@/store/appStore'
3. Import cn from '@/lib/styles'
4. Import data from '@/data/ganttData'
5. Use memo wrapper with named function
6. Add containerRef and auto-scroll effect
7. Convert all inline styles to Tailwind gruvbox classes

Key structure:
```typescript
export const TimelineView = memo(function TimelineView() {
  const { selectedIndex } = useAppStore();
  const containerRef = useRef<HTMLDivElement>(null);
  const timelineRef = useRef<HTMLDivElement>(null);
  const [expandedProjects, setExpandedProjects] = useState<Record<string, boolean>>({});
  const [viewMode, setViewMode] = useState<'day' | 'week' | 'month'>('week');

  // Auto-scroll selected row
  useEffect(() => {
    const selected = containerRef.current?.querySelector('[data-selected="true"]');
    selected?.scrollIntoView({ block: 'nearest', behavior: 'smooth' });
  }, [selectedIndex]);

  return (
    <div className="flex flex-col h-full bg-gruvbox-bg-hard">
      {/* Header */}
      {/* Legend */}
      {/* Main content: left panel (labels) + right panel (timeline) */}
    </div>
  );
});
```

#### Step 3: Add View Type

Edit `millflow/src/types/index.ts:150`:
```typescript
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'departments' | 'queue' | 'schedule' | 'timeline';
```

#### Step 4: Register in App Router

Edit `millflow/src/App.tsx`:
```typescript
import TimelineView from '@/views/TimelineView';

// In renderView switch:
case 'timeline':
  return <TimelineView />;
```

#### Step 5: Add Navigation Action

Edit `millflow/src/store/appStore.ts`:
```typescript
goToTimeline: () => set({
  view: 'timeline',
  selectedIndex: 0,
}),
```

#### Step 6: Add Keyboard Shortcut

Edit `millflow/src/hooks/useKeyboard.ts`:
```typescript
// In key sequence handling:
if (sequence === 'g t') {
  event.preventDefault();
  resetSequence();
  goToTimeline();
  return;
}
```

### Converting Component Styles

The most significant work is converting inline styles to Tailwind. Examples:

**Before (inline)**:
```jsx
<div style={{
  background: colors.bg0,
  color: colors.fg,
  fontFamily: "'JetBrains Mono', 'Fira Code', monospace",
  fontSize: 12,
  height: '100vh',
}}>
```

**After (Tailwind)**:
```jsx
<div className="bg-gruvbox-bg-hard text-gruvbox-fg font-mono text-xs h-screen">
```

**Before (status colors)**:
```jsx
style={{
  color: row.data.status < -10 ? colors.red :
    row.data.status < 0 ? colors.orange :
    row.data.status < 5 ? colors.yellow : colors.fg4,
}}
```

**After (Tailwind with cn)**:
```jsx
className={cn(
  'text-xs font-semibold w-8',
  status < -10 && 'text-gruvbox-red-bright',
  status >= -10 && status < 0 && 'text-gruvbox-orange-bright',
  status >= 0 && status < 5 && 'text-gruvbox-yellow-bright',
  status >= 5 && 'text-gruvbox-fg-4',
)}
```

### Positioning Exception

The timeline uses absolute positioning for date-based elements. This requires inline styles for `left` and `width` calculations:

```jsx
<div
  className="absolute top-1.5 h-4 bg-gruvbox-aqua-bright rounded opacity-85"
  style={{
    left: getDateX(startDate),
    width: Math.max(DAY_WIDTH * 0.8, getDateX(endDate) - getDateX(startDate) + DAY_WIDTH),
  }}
/>
```

This is acceptable - only use inline styles for dynamic positioning, never for colors.

## Code References

- `millflow/src/views/ScheduleView.tsx:1-396` - Reference schedule view implementation
- `millflow/src/types/index.ts:150` - ViewType definition to extend
- `millflow/src/App.tsx:20-39` - View routing switch statement
- `millflow/src/store/appStore.ts:459-465` - goToSchedule pattern to follow
- `millflow/src/hooks/useKeyboard.ts:154-228` - Key sequence handling
- `millflow/src/data/scheduleData.ts:1-161` - Mock data pattern to follow

## Architecture Documentation

### Current View Registration Pattern

1. Type in `types/index.ts` ViewType union
2. Switch case in `App.tsx` renderView
3. Navigation action in `appStore.ts`
4. Keyboard shortcut in `useKeyboard.ts` (optional)

### Styling Conventions

- All colors use `gruvbox-*` Tailwind classes
- Never use hex values or inline color styles
- Inline styles only for dynamic positioning
- Use `cn()` utility for conditional classes
- Font: system monospace via `font-mono`

### State Management Pattern

For prototype views that don't need persistence, local state with useState is acceptable (see ScheduleView line 216). For production, data would move to the Zustand store.

## Open Questions

1. **Data source**: Should this use the existing mock data or maintain its own dataset?
2. **View relationship**: Is this a replacement for ScheduleView or an additional view?
3. **Keyboard navigation**: Should j/k navigate rows, and h/l scroll the timeline?
4. **Interaction model**: Should clicking bars open the sheet detail view?
