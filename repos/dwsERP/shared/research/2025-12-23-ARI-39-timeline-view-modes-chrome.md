---
date: 2025-12-23T16:13:18-08:00
researcher: ariasulin
git_commit: 2660d75535156917eaa5c961be1e74917e090158
branch: main
repository: dwsERP
topic: "Timeline View Modes & Chrome"
tags: [research, codebase, timeline, view-modes, command-palette, keyboard-navigation]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
---

# Research: ARI-39 Timeline View Modes & Chrome

**Date**: 2025-12-23T16:13:18-08:00
**Researcher**: ariasulin
**Git Commit**: 2660d75535156917eaa5c961be1e74917e090158
**Branch**: main
**Repository**: dwsERP

## Research Question

Document the current implementation of timeline view modes and chrome elements in MillFlow:
1. Day/Week/Month toggle behavior
2. Header/footer structure and scroll behavior
3. Command-K integration for timeline actions

## Summary

The TimelineView has a visually functional Day/Week/Month toggle that **only updates state without changing behavior**. The header is sticky, footer is fixed at bottom, and scroll behavior is well-implemented. Command-K is **missing Timeline navigation** despite `g t` working via keyboard.

## Detailed Findings

### 1. Day/Week/Month Toggle

**Location**: `millflow/src/views/TimelineView.tsx:25, 145-158`

**Current State**:
```tsx
const [viewMode, setViewMode] = useState<'day' | 'week' | 'month'>('week');
```

**UI Implementation** (lines 145-158):
- Three buttons rendered for 'day', 'week', 'month'
- Visual toggle works - selected mode gets different styling
- `onClick={() => setViewMode(mode)}` updates state

**Key Finding**: **The toggle is UI-only**
- `viewMode` state is set but never read elsewhere in the component
- Layout always uses fixed `DAY_WIDTH: 24` constant regardless of mode
- No zoom/scale behavior tied to the mode
- The toggle buttons exist and look functional but have no effect on the timeline rendering

### 2. Header/Footer Structure

**Layout Constants** (`millflow/src/data/timelineData.ts:74-78`):
```typescript
TIMELINE_CONFIG = {
  DAY_WIDTH: 24,
  ROW_HEIGHT: 28,
  HEADER_HEIGHT: 60,
  LABEL_PANEL_WIDTH: 280,
}
```

**Main Header** (lines 134-160):
- Contains "MillFlow | Timeline" branding
- Shows sheet/project count badge
- Houses the Day/Week/Month toggle
- Class: `bg-gruvbox-bg border-b border-gruvbox-bg-1`
- Not sticky - scrolls with page

**Legend Bar** (lines 163-170):
- Shows department color codes
- Class: `border-b border-gruvbox-bg-1 bg-gruvbox-bg`
- Not sticky - scrolls with page

**Timeline Header** (lines 247-286):
- Contains month row and week row
- **IS STICKY**: `sticky top-0 z-10 bg-gruvbox-bg`
- Height: `HEADER_HEIGHT` (60px)
- Months show "Nov '25", "Dec '25" format
- Weeks show day-of-month (1, 8, 15, etc.)

**Footer** (lines 414-432):
- Fixed at bottom (not sticky, just at end of flex container)
- Shows keyboard hints: `← → scroll`, `click project to expand/collapse`
- Shows current date/time: "Mon, Dec 23 • 4:13 PM"
- Class: `border-t border-gruvbox-bg-1 bg-gruvbox-bg`

**Scroll Behavior**:
- Left panel (row labels): `overflow-y-auto overflow-x-hidden` (line 186)
- Right panel (timeline): `overflow-auto` for both axes (line 244)
- Timeline header stays visible during vertical scroll via `sticky top-0`
- Auto-scroll to today on mount (lines 98-105)

### 3. Command-K Integration

**Location**: `millflow/src/components/CommandPalette.tsx`

**Existing Navigation Commands** (lines 37-99):
| Command | Shortcut | Action |
|---------|----------|--------|
| Go to Dashboard | `g h` | `goHome()` |
| Go to Jobs | `g j` | `goToJobs()` |
| Departments | `g q` | `goToDepartments()` |
| Shop Schedule | `g s` | `goToSchedule()` |
| [Dept] Queue | `g [shortcut]` | `goToQueue(dept.id)` |

**Key Finding**: **No Timeline navigation in Command Palette**
- There is no "Go to Timeline" command
- Shop Schedule exists (`g s`) but Timeline doesn't
- The keyboard handler has `g t` for timeline (useKeyboard.ts:219-225)
- This is a gap - command palette users cannot navigate to Timeline

**Keyboard Navigation** (`millflow/src/hooks/useKeyboard.ts`):

| Sequence | View | Action |
|----------|------|--------|
| `g t` | Any | Navigate to timeline (`goToTimeline()`) |
| `h` / `←` | Timeline | Dispatch 'timeline-scroll' left |
| `l` / `→` | Timeline | Dispatch 'timeline-scroll' right |

**Timeline Scroll Event Handler** (TimelineView.tsx:114-127):
```tsx
const handleTimelineScroll = (e: CustomEvent<{ direction: 'left' | 'right' }>) => {
  const scrollAmount = DAY_WIDTH * 7; // Scroll one week at a time
  if (e.detail.direction === 'left') {
    timelineRef.current.scrollLeft -= scrollAmount;
  } else {
    timelineRef.current.scrollLeft += scrollAmount;
  }
};
```

### 4. Store Integration

**Location**: `millflow/src/store/appStore.ts:82, 468-470`

```typescript
goToTimeline: () => {
  set({
    view: 'timeline',
    // ... other state resets
  });
}
```

The store correctly exposes `goToTimeline()` method used by useKeyboard.

## Code References

- `millflow/src/views/TimelineView.tsx:25` - viewMode state declaration
- `millflow/src/views/TimelineView.tsx:145-158` - Day/Week/Month toggle buttons
- `millflow/src/views/TimelineView.tsx:247-286` - Sticky timeline header
- `millflow/src/views/TimelineView.tsx:414-432` - Footer with keyboard hints
- `millflow/src/components/CommandPalette.tsx:37-99` - Navigation commands (missing Timeline)
- `millflow/src/hooks/useKeyboard.ts:219-225` - `g t` keyboard navigation
- `millflow/src/hooks/useKeyboard.ts:374-391` - Timeline scroll via h/l keys
- `millflow/src/data/timelineData.ts:74-78` - TIMELINE_CONFIG constants

## Architecture Documentation

**View Mode Toggle Pattern**:
The toggle follows a React state pattern but lacks the reducer logic to modify layout calculations. The `viewMode` would need to:
1. Adjust `DAY_WIDTH` calculation (e.g., week mode = 7x wider days)
2. Modify date label display (show week numbers vs day numbers)
3. Aggregate data differently (daily detail vs weekly summary)

**Sticky Header Pattern**:
Uses CSS `sticky` positioning with `top-0` and `z-10` for layer stacking. This is the standard Tailwind approach for sticky headers.

**Custom Event Pattern**:
Timeline scroll uses `window.dispatchEvent` with custom events rather than direct ref access. This decouples the keyboard handler from the view component.

## Historical Context (from thoughts/)

No prior research documents found for timeline implementation.

## Related Research

- `thoughts/shared/research/2025-12-23-gantt-timeline-integration.md` - Recent Gantt integration work

## Open Questions

1. Should viewMode affect zoom level or data aggregation?
2. Should Command Palette include "Go to Timeline" with `g t` shortcut?
3. Should the main header also be sticky above the legend?
