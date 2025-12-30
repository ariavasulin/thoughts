---
date: 2025-12-30T13:47:41-08:00
researcher: ariasulin
git_commit: 3ef7e7d7caf88b7ae16f80ce3f05acaee8bfcc32
branch: main
repository: DWS-Receipts
topic: "Replace react-day-picker with Native Date Inputs"
tags: [datepicker, native-inputs, react-day-picker, bug-fix]
status: in_progress
last_updated: 2025-12-30
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: Replace react-day-picker with Native Date Inputs

## Task(s)

**Completed:**
1. Replaced desktop Calendar+Popover datepicker with native `<input type="date">` in `receipt-details-card.tsx`
2. Replaced Calendar-based date range picker with button-triggered native inputs in `date-range-picker.tsx`
3. Removed unused `calendar.tsx` component
4. Removed react-day-picker type imports from `receipt-dashboard.tsx`

**Work in Progress:**
1. **BUG**: After selecting a date range, React Query throws: `Error: Expected enabled to be a boolean or a callback that returns a boolean`

## Critical References
- `dws-app/src/components/receipt-dashboard.tsx:97-98` - The `shouldFetch` variable causing the bug
- `dws-app/src/components/date-range-picker.tsx` - New native date input implementation

## Recent changes

- `dws-app/src/components/receipt-details-card.tsx:288-294` - Replaced conditional mobile/desktop rendering with single native date input
- `dws-app/src/components/receipt-details-card.tsx` - Removed imports for Calendar, Popover, CalendarIcon, useMobile
- `dws-app/src/components/date-range-picker.tsx` - Complete rewrite to use hidden native inputs triggered by a button
- `dws-app/src/components/receipt-dashboard.tsx:66-71` - Updated handleDateChange type signature to remove react-day-picker dependency
- Deleted `dws-app/src/components/ui/calendar.tsx`

## Learnings

**Root cause of original datepicker issues:**
- react-day-picker v8.10.1 had compatibility issues with the shadcn/ui Calendar wrapper
- The desktop datepicker wasn't responding to clicks, possibly due to component API changes in react-day-picker v8.7+
- The date range picker was crashing with "Application error" after selecting both dates

**Current bug root cause:**
The `shouldFetch` boolean expression in `receipt-dashboard.tsx:97-98`:
```tsx
const shouldFetch = !dateRange.from || (dateRange.from && dateRange.to)
```

When `dateRange.from` and `dateRange.to` are both Date objects, `(dateRange.from && dateRange.to)` returns the Date object itself (truthy but not a boolean), not `true`. React Query's `enabled` option requires a strict boolean.

**Fix needed:**
```tsx
const shouldFetch = !dateRange.from || Boolean(dateRange.from && dateRange.to)
```
or
```tsx
const shouldFetch = !dateRange.from || (dateRange.from !== undefined && dateRange.to !== undefined)
```

## Artifacts

- `thoughts/shared/research/2025-12-30-desktop-datepicker-implementation.md` - Initial research documenting the datepicker implementation
- `dws-app/src/components/date-range-picker.tsx` - New implementation using native inputs
- `dws-app/src/components/receipt-details-card.tsx` - Updated to use native date input

## Action Items & Next Steps

1. **Fix the React Query `enabled` bug** in `receipt-dashboard.tsx:97-98` - ensure `shouldFetch` is always a boolean
2. **Test date range selection** on the dashboard after the fix
3. **Test single date selection** in receipt edit dialog to confirm it works
4. **Consider removing react-day-picker** from `package.json` dependencies since it's no longer used (calendar.tsx was deleted)
5. **Run build** to verify no TypeScript errors

## Other Notes

**New date-range-picker behavior:**
- Single button shows "Pick a date range" or the selected range
- Clicking opens native "from" date picker
- After selecting "from", automatically opens "to" date picker
- Uses hidden `<input type="date">` elements with `showPicker()` API

**Files that may still reference react-day-picker:**
- `package.json` - Can be removed as a dependency
- No other source files import it (verified with grep)

**useMobile hook** at `dws-app/src/hooks/use-mobile.tsx` is no longer used by receipt-details-card.tsx but may be used elsewhere - check before removing.
