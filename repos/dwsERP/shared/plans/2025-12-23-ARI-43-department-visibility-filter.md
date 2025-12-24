# ARI-43: Department/Step Visibility Filter Implementation Plan

## Overview

Add clickable department toggles to the timeline legend, allowing users to show/hide specific department bars on the timeline. This is a simple local state implementation - no persistence.

## Current State Analysis

**TimelineView.tsx** renders all 6 departments (veneer, mill, cnc, bench, finish, assembly) unconditionally:
- Line 10: `DEPARTMENTS` constant defines the display order
- Lines 165-174: Legend displays color swatches for each department
- Lines 365-395: Department bars render for all departments on each sheet row

**No filtering mechanism exists.** Every department is always visible.

## Desired End State

- Clicking a department in the legend toggles its visibility
- Hidden departments don't render bars on the timeline
- Legend shows visual indication of hidden departments (faded/strikethrough)
- All departments visible by default

### Verification:
1. Click "veneer" in legend → veneer bars disappear from all sheet rows
2. Click "veneer" again → veneer bars reappear
3. Multiple departments can be hidden simultaneously
4. Page refresh resets all departments to visible (no persistence)

## What We're NOT Doing

- **No localStorage persistence** - Visibility resets on page refresh
- **No Zustand store integration** - Using local useState only
- **No keyboard shortcuts** - Click-only interaction for now
- **No "show all / hide all" buttons** - Individual toggles only

## Implementation Approach

Add a `visibleDepts` Set state initialized with all departments. Make legend items clickable to toggle membership. Filter department bar rendering to only include visible departments.

---

## Phase 1: Add Visibility State and Toggle Logic

### Overview
Add local state to track which departments are visible, with all departments visible by default.

### Changes Required:

#### 1. Add visibleDepts state
**File**: `millflow/src/views/TimelineView.tsx`
**Location**: After line 25 (after viewMode state)

```typescript
const [visibleDepts, setVisibleDepts] = useState<Set<string>>(
  () => new Set(DEPARTMENTS)
);

const toggleDept = (dept: string) => {
  setVisibleDepts(prev => {
    const next = new Set(prev);
    if (next.has(dept)) {
      next.delete(dept);
    } else {
      next.add(dept);
    }
    return next;
  });
};
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] No TypeScript errors
- [x] No lint errors: `npm run lint`

#### Manual Verification:
- [x] N/A - state not yet connected to UI

---

## Phase 2: Make Legend Items Clickable

### Overview
Convert the static legend items to clickable buttons that toggle department visibility. Add visual feedback for hidden departments.

### Changes Required:

#### 1. Update legend rendering
**File**: `millflow/src/views/TimelineView.tsx`
**Location**: Lines 165-174 (legend section)

Replace the existing legend div with:

```typescript
{/* Center: Legend (clickable toggles) */}
<div className="flex items-center gap-3 text-[10px]">
  {DEPARTMENTS.map(dept => {
    const isVisible = visibleDepts.has(dept);
    const colorClass = TIMELINE_DEPT_COLORS[dept];
    return (
      <button
        key={dept}
        onClick={() => toggleDept(dept)}
        className={cn(
          'flex items-center gap-1 px-1 py-0.5 rounded transition-colors',
          isVisible
            ? 'hover:bg-gruvbox-bg-1'
            : 'opacity-40 hover:opacity-60'
        )}
        title={isVisible ? `Hide ${dept}` : `Show ${dept}`}
      >
        <div className={cn('w-2.5 h-1.5 rounded-sm', colorClass)} />
        <span className={cn(
          'capitalize',
          isVisible ? 'text-gruvbox-fg-4' : 'text-gruvbox-fg-4 line-through'
        )}>
          {dept}
        </span>
      </button>
    );
  })}
</div>
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] No TypeScript errors
- [x] No lint errors: `npm run lint`

#### Manual Verification:
- [ ] Legend items are clickable (cursor changes, hover state)
- [ ] Clicking a department toggles its visual state (opacity/strikethrough)
- [ ] Tooltip shows "Hide X" or "Show X" based on current state

**Implementation Note**: After completing this phase, pause for manual confirmation before proceeding.

---

## Phase 3: Filter Department Bar Rendering

### Overview
Only render department bars for visible departments on each sheet row.

### Changes Required:

#### 1. Filter DEPARTMENTS.map in bar rendering
**File**: `millflow/src/views/TimelineView.tsx`
**Location**: Lines 365-395 (sheet operation bars)

Change line 365 from:
```typescript
{DEPARTMENTS.map(dept => {
```

To:
```typescript
{DEPARTMENTS.filter(dept => visibleDepts.has(dept)).map(dept => {
```

This is a one-line change - adding `.filter(dept => visibleDepts.has(dept))` before `.map()`.

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd millflow && npm run build`
- [x] No TypeScript errors
- [x] No lint errors: `npm run lint`

#### Manual Verification:
- [ ] Clicking "veneer" in legend hides all veneer bars on timeline
- [ ] Clicking "veneer" again shows veneer bars
- [ ] Hiding multiple departments works (e.g., hide veneer + mill)
- [ ] Hidden department bars don't render (not just invisible)
- [ ] Page refresh resets all departments to visible

---

## Testing Strategy

### Manual Testing Steps:
1. Navigate to Timeline view (`g t`)
2. Verify all 6 departments visible in legend (no strikethrough)
3. Click "veneer" - verify bars disappear, legend shows strikethrough
4. Click "cnc" - verify bars disappear, legend shows strikethrough
5. Click "veneer" again - verify bars reappear
6. Scroll through timeline to verify filtering applies to all sheets
7. Refresh page - verify all departments reset to visible

### Edge Cases:
- Hiding all departments (timeline should show only project rows and delivery markers)
- Sheets with null departments (should not affect behavior)

## References

- Original ticket: ARI-43 (Linear)
- Research doc: `thoughts/searchable/shared/research/2025-12-23-ARI-43-department-visibility-filter.md`
- Similar pattern: `millflow/src/views/TimelineView.tsx:147-161` (view mode toggle)
- Set toggle pattern: `millflow/src/store/appStore.ts:496-504` (expandedSections)
