---
date: 2025-12-22T23:43:20-08:00
researcher: claude
git_commit: 8680d53bc72ec5b434883bc787d132513b97a151
branch: main
repository: dwsERP
topic: "Shop Schedule View Integration into MillFlow"
tags: [implementation, shop-schedule, millflow, navigation]
status: complete
last_updated: 2025-12-22
last_updated_by: claude
type: implementation_strategy
---

# Handoff: Integrate Shop Schedule Prototype into MillFlow App

## Task(s)

| Task | Status |
|------|--------|
| Create shop schedule prototype with status cycling | Completed |
| Add duration editing via right-click | Completed |
| Generate realistic mock data | Completed |
| Integrate into millflow app as navigable view | **Planned** |

## Critical References

- `misc/shopview.jsx` - The working prototype to port
- `millflow/src/store/appStore.ts` - Store patterns and navigation actions
- `millflow/src/hooks/useKeyboard.ts:83-88` - Command+K implementation

## Recent Changes

- `misc/shopview.jsx` - Shop schedule React component with:
  - Status cycling: click actionable (yellow) → in-progress (green) → done (blue)
  - Duration editing: right-click opens number input, Enter to save
  - Gruvbox theming consistent with millflow
- `misc/mockData.js` - Mock data generator (~70 sheets across 10 projects)
- `misc/index.html` - Standalone dev server shell

## Learnings

### MillFlow Navigation Architecture

1. **No router** - Uses state-based view switching via `useAppStore().view`
2. **ViewType enum** at `millflow/src/types/index.ts:150`:
   ```typescript
   ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'departments' | 'queue'
   ```
3. **View switching** in `millflow/src/App.tsx:13-36` via switch/case
4. **Command+K** triggers `toggleCommandPalette()` which opens `CommandPalette.tsx`

### Key Patterns to Follow

- All views use `memo()` wrapper
- Access store via `useAppStore()` hook
- Track selection with `selectedIndex` from store
- Use `data-selected` attribute for keyboard navigation
- Scroll selected item into view in useEffect

### Keyboard Sequences

- `g` prefix starts sequences (500ms timeout)
- `g h` = home, `g j` = jobs, `g q` = departments
- Department shortcuts: `g c` = CNC queue, `g b` = bench queue, etc.

## Artifacts

- `misc/shopview.jsx` - Complete prototype (port this)
- `misc/mockData.js` - Mock data generator (adapt for store)
- `misc/index.html` - Dev shell (not needed for integration)

## Action Items & Next Steps

### 1. Add ViewType

**File:** `millflow/src/types/index.ts:150`

Add `'schedule'` to the ViewType union:
```typescript
export type ViewType = 'dashboard' | 'jobs' | 'job' | 'sheet' | 'departments' | 'queue' | 'schedule';
```

### 2. Create ScheduleView Component

**File:** `millflow/src/views/ScheduleView.tsx`

Port `misc/shopview.jsx` with these adaptations:
- Convert to TypeScript
- Replace inline styles with Tailwind (use existing `gruvbox-*` classes)
- Use `useAppStore()` instead of local useState for sheets data
- Use `createPortal` from react-dom for duration editor
- Wire up `selectedIndex` for keyboard navigation through rows

Key functions to port:
- `isActionable()`, `isInProgress()`, `isSheetComplete()`
- `getPriorityScore()` for sorting
- `DeptCell` component with status cycling and right-click
- `DurationEditor` component (simple number input)

### 3. Add to App.tsx Router

**File:** `millflow/src/App.tsx`

```typescript
// Add import
import { ScheduleView } from '@/views/ScheduleView';

// Add case in switch (around line 30)
case 'schedule':
  return <ScheduleView />;
```

### 4. Add Store Actions

**File:** `millflow/src/store/appStore.ts`

Add navigation action (near line 456 with other `goTo*` methods):
```typescript
goToSchedule: () => set({
  view: 'schedule',
  selectedIndex: 0,
}),
```

Add sheet status mutation (for clicking cells):
```typescript
setSheetOperationStatus: (sheetId: string, deptId: string, status: 'pending' | 'inProgress' | 'done') =>
  set((state) => {
    // pushHistory() first
    // Update sheet.operations[deptId].status
  }),

setSheetOperationDuration: (sheetId: string, deptId: string, duration: number) =>
  set((state) => {
    // pushHistory() first
    // Update sheet.operations[deptId].duration
  }),
```

### 5. Add Keyboard Shortcut

**File:** `millflow/src/hooks/useKeyboard.ts`

Add `g s` sequence for schedule (near line 200 with other `g` sequences):
```typescript
if (sequence === 'gs') {
  goToSchedule();
  return;
}
```

### 6. Add to Command Palette

**File:** `millflow/src/components/CommandPalette.tsx`

Add schedule to navigation items in the commands array:
```typescript
{ id: 'nav-schedule', label: 'Go to Schedule', shortcut: 'g s', action: () => goToSchedule() },
```

### 7. Update Sheet Type (if needed)

**File:** `millflow/src/types/index.ts`

Ensure Sheet type has operations with status field:
```typescript
interface SheetOperation {
  duration: number;
  status: 'pending' | 'inProgress' | 'done';
  dependsOn: string[];
}
```

### 8. Add Mock Data

**File:** `millflow/src/data/mockData.ts`

Port the mock data generator from `misc/mockData.js` or create static mock sheets with the operation structure.

## Other Notes

### Styling Reference

The prototype uses inline Gruvbox colors. MillFlow has Tailwind classes:
- `bg-gruvbox-bg0`, `bg-gruvbox-bg1`, `bg-gruvbox-bg2`
- `text-gruvbox-fg`, `text-gruvbox-yellow`, `text-gruvbox-blue`, etc.
- `border-gruvbox-bg3`

### Status Color Mapping

| Status | Color | Tailwind Class |
|--------|-------|----------------|
| Done | Blue | `bg-gruvbox-blue` |
| In Progress | Green | `bg-gruvbox-green` |
| Actionable | Yellow | `bg-gruvbox-yellow` |
| Urgent (slack <= 3) | Orange | `bg-gruvbox-orange` |
| Overdue (slack < 0) | Red | `bg-gruvbox-red` |
| Blocked | Gray | `bg-gruvbox-bg2` |

### Testing the Prototype

The standalone prototype can still be run for reference:
```bash
cd misc && python3 -m http.server 3456
# Open http://localhost:3456
```

### Priority/Urgency Calculation

Slack = days until install - remaining work days (sum of undone operation durations)
- Negative slack = overdue
- 0-3 days slack = urgent
- 4+ days slack = normal actionable
