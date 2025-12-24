# ARI-39: Timeline View Modes & Chrome Implementation Plan

## Overview

Make the timeline view modes functional (Day/Week/Month = zoom levels) and consolidate header/footer chrome to maximize content area.

## Current State Analysis

**View Mode Toggle** (`TimelineView.tsx:25, 145-158`):
- `viewMode` state exists but is never used for rendering
- `DAY_WIDTH = 24` is constant regardless of mode
- Toggle buttons work visually but don't affect layout

**Chrome Structure** (redundant - TimelineView renders inside Layout):
- Layout header: "MillFlow" + Cmd+K + date + user (~48px)
- TimelineView header: "MillFlow | Timeline" + count + toggle (~40px) ← DUPLICATE
- TimelineView legend: Department colors (~32px)
- Timeline header: Months + weeks, sticky (60px)
- Layout footer: ShortcutBar (~36px) - missing timeline shortcuts!
- TimelineView footer: Keyboard hints + date (~36px) ← DUPLICATE
- **Total redundant chrome: ~76px wasted**

**Command Palette**: Missing "Go to Timeline" (`g t` works via keyboard but not in palette)

## Desired End State

1. **View modes change zoom level**:
   - Day: 72px per day (focused, ~5 days visible)
   - Week: 36px per day (balanced, ~10 days visible)
   - Month: 24px per day (overview, ~15 days visible - current)

2. **Consolidated chrome** (use Layout, remove duplicates):
   - Remove TimelineView's own header (Layout provides branding)
   - Keep compact toolbar: view toggle + legend inline (~28px)
   - Timeline header stays (sticky, 60px)
   - Remove TimelineView's footer (use ShortcutBar)
   - Add timeline shortcuts to ShortcutBar

3. **Command Palette includes Timeline navigation**

## What We're NOT Doing

- Data aggregation (bars always span actual days)
- Different date label formats per mode
- Persisting view mode preference to localStorage
- Animation between zoom levels

## Implementation Approach

Simple state-driven zoom. The `viewMode` state already exists - we just need to derive `DAY_WIDTH` from it and use that throughout.

---

## Phase 1: Add Timeline to Command Palette

### Overview
Quick fix to add missing navigation command.

### Changes Required:

#### 1. CommandPalette.tsx
**File**: `millflow/src/components/CommandPalette.tsx`
**Changes**: Add Timeline navigation command after Schedule command

```tsx
// Add to imports (line 11)
import {
  // ... existing imports
  Clock, // Add this for timeline icon
} from 'lucide-react';

// Add after nav-schedule command (around line 84)
items.push({
  id: 'nav-timeline',
  type: 'navigation',
  title: 'Timeline',
  subtitle: 'g t',
  icon: Clock,
  action: () => {
    useAppStore.getState().goToTimeline();
    toggleCommandPalette();
  },
});
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Cmd+K shows "Timeline" option with `g t` subtitle
- [ ] Selecting it navigates to Timeline view
- [ ] Typing "timeline" in palette filters to show the command

---

## Phase 2: Implement View Mode Zoom Levels

### Overview
Make `viewMode` state control the zoom level by deriving `DAY_WIDTH` from it.

### Changes Required:

#### 1. TimelineView.tsx - Dynamic DAY_WIDTH
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: Calculate DAY_WIDTH from viewMode instead of using constant

```tsx
// Replace lines 7 (destructuring) with:
const { ROW_HEIGHT, HEADER_HEIGHT, LABEL_PANEL_WIDTH } = TIMELINE_CONFIG;

// Add after viewMode state declaration (around line 26):
// Calculate day width based on view mode (zoom level)
const DAY_WIDTH = useMemo(() => {
  switch (viewMode) {
    case 'day': return 72;    // Focused: ~5 days visible
    case 'week': return 36;   // Balanced: ~10 days visible
    case 'month': return 24;  // Overview: ~15 days visible (current)
    default: return 24;
  }
}, [viewMode]);
```

#### 2. Update scroll amount for keyboard navigation
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: Adjust scroll amount based on zoom level (around line 117)

The scroll handler already uses `DAY_WIDTH * 7` which will now be dynamic. No change needed - it will automatically scroll proportionally.

#### 3. Update auto-scroll to today
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: The auto-scroll also uses `DAY_WIDTH` (line 102) which will now be dynamic. Add `DAY_WIDTH` to dependency array.

```tsx
// Update useEffect dependency array (around line 105):
}, [dateRange.start, DAY_WIDTH]);
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Clicking "day" makes timeline bars visibly wider (72px per day)
- [ ] Clicking "week" shows medium zoom (36px per day)
- [ ] Clicking "month" shows overview (24px per day - current behavior)
- [ ] Keyboard scroll (h/l) works correctly at all zoom levels
- [ ] Weekend shading scales correctly with zoom
- [ ] Today line stays in correct position across mode changes

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 3: Consolidate Chrome (Remove Duplicates)

### Overview
TimelineView renders inside Layout but has its own header/footer, creating redundancy. Remove duplicates and use Layout's chrome instead.

### Changes Required:

#### 1. Add timeline shortcuts to ShortcutBar
**File**: `millflow/src/components/ShortcutBar.tsx`
**Changes**: Add timeline and schedule shortcuts

```tsx
// Add after queueShortcuts (around line 51)
const timelineShortcuts: Shortcut[] = [
  { key: 'h/l', label: 'scroll' },
  { key: 'click', label: 'expand/collapse' },
  { key: 'esc', label: 'back' },
];

const scheduleShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'navigate' },
  { key: 'esc', label: 'back' },
];

// Update switch statement (around line 151) to add cases:
case 'timeline':
  return timelineShortcuts;
case 'schedule':
  return scheduleShortcuts;
```

#### 2. Replace TimelineView header with compact toolbar
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: Remove redundant "MillFlow | Timeline" header, keep just view toggle + legend in a compact bar

Replace header section (lines 133-170, the `{/* Header */}` and `{/* Legend */}` divs) with:

```tsx
{/* Compact Toolbar: View toggle + Legend */}
<div className="flex items-center justify-between px-3 py-1.5 border-b border-gruvbox-bg-1 bg-gruvbox-bg">
  {/* Left: View toggle */}
  <div className="flex items-center gap-1">
    {(['day', 'week', 'month'] as const).map(mode => (
      <button
        key={mode}
        onClick={() => setViewMode(mode)}
        className={cn(
          'px-2 py-0.5 rounded text-[10px] capitalize transition-colors',
          viewMode === mode
            ? 'bg-gruvbox-bg-2 text-gruvbox-fg'
            : 'text-gruvbox-fg-4 hover:bg-gruvbox-bg-1'
        )}
      >
        {mode}
      </button>
    ))}
  </div>

  {/* Center: Legend */}
  <div className="flex items-center gap-3 text-[10px]">
    {Object.entries(TIMELINE_DEPT_COLORS).filter(([dept]) =>
      DEPARTMENTS.includes(dept as typeof DEPARTMENTS[number])
    ).map(([dept, colorClass]) => (
      <div key={dept} className="flex items-center gap-1">
        <div className={cn('w-2.5 h-1.5 rounded-sm', colorClass)} />
        <span className="text-gruvbox-fg-4 capitalize">{dept}</span>
      </div>
    ))}
  </div>

  {/* Right: Stats */}
  <div className="text-[10px] text-gruvbox-fg-4">
    {totalSheets} sheets · {timelineProjects.length} projects
  </div>
</div>
```

#### 3. Remove TimelineView footer
**File**: `millflow/src/views/TimelineView.tsx`
**Changes**: Delete footer section (lines 413-432) - ShortcutBar handles this now

Delete:
```tsx
{/* Footer */}
<div className="flex items-center justify-between px-4 py-2 border-t border-gruvbox-bg-1 bg-gruvbox-bg text-[11px] text-gruvbox-fg-4">
  ...
</div>
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd millflow && npm run build`
- [ ] Lint passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Layout header shows at top (with Cmd+K, date, user)
- [ ] Compact toolbar shows view toggle + legend inline (~28px)
- [ ] No duplicate "MillFlow" text
- [ ] Timeline content area visibly taller
- [ ] ShortcutBar at bottom shows timeline shortcuts (h/l scroll, etc.)
- [ ] All department colors readable in compact legend

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Testing Strategy

### Manual Testing Steps:
1. Navigate to Timeline via `g t` or Command Palette
2. Verify Layout header shows (Cmd+K, date, user) - no duplicate "MillFlow"
3. Click through Day → Week → Month modes, verify zoom changes
4. Check bars maintain correct relative positions at all zoom levels
5. Scroll with h/l keys at each zoom level
6. Verify ShortcutBar shows timeline shortcuts (h/l scroll, click expand)
7. Confirm compact toolbar shows view toggle + legend
8. Measure vertical space savings (should be ~76px gained)

## Performance Considerations

- `DAY_WIDTH` is memoized, only recalculates on mode change
- No additional re-renders beyond what mode toggle already caused
- Scroll performance unaffected (using same refs and handlers)

## References

- Research: `thoughts/searchable/shared/research/2025-12-23-ARI-39-timeline-view-modes-chrome.md`
- Linear ticket: ARI-39
- Related: `millflow/src/views/TimelineView.tsx`
