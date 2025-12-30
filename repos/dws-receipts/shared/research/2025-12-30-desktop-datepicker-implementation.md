---
date: 2025-12-30T13:14:47-08:00
researcher: ariasulin
git_commit: 3ef7e7d7caf88b7ae16f80ce3f05acaee8bfcc32
branch: main
repository: DWS-Receipts
topic: "Desktop Datepicker Implementation in Receipt Editor"
tags: [research, codebase, datepicker, calendar, receipt-details-card, shadcn-ui]
status: complete
last_updated: 2025-12-30
last_updated_by: ariasulin
---

# Research: Desktop Datepicker Implementation in Receipt Editor

**Date**: 2025-12-30 13:14:47 PST
**Researcher**: ariasulin
**Git Commit**: 3ef7e7d7caf88b7ae16f80ce3f05acaee8bfcc32
**Branch**: main
**Repository**: DWS-Receipts

## Research Question

The desktop datepicker on the receipt editor or when submitting isn't working - it won't select dates.

## Summary

The desktop datepicker uses the standard shadcn/ui pattern: a Popover containing a Calendar component from react-day-picker v8.10.1. The implementation follows the documented approach with `mode="single"`, controlled `selected` state, and an `onSelect` callback. The receipt-details-card.tsx component conditionally renders either a native HTML5 date input (mobile) or the Popover+Calendar (desktop) based on the `useMobile()` hook result.

## Detailed Findings

### Component Architecture

**Desktop Datepicker Location**: `dws-app/src/components/receipt-details-card.tsx:297-312`

The component uses conditional rendering based on device detection:
- **Mobile (isMobile = true)**: Native HTML5 `<input type="date">` at lines 288-296
- **Desktop (isMobile = false)**: Popover + Calendar at lines 297-312

### Desktop Datepicker Implementation

**File**: `dws-app/src/components/receipt-details-card.tsx`

```tsx
// Lines 297-312
<Popover>
  <PopoverTrigger asChild>
    <Button
      variant="outline"
      className="w-full justify-start text-left font-normal bg-[#3e3e3e] border-[#3e3e3e] text-white hover:bg-[#4e4e4e]"
    >
      <CalendarIcon className="mr-2 h-4 w-4" />
      {date ? format(date, "PPP") : <span>Select date</span>}
    </Button>
  </PopoverTrigger>
  <PopoverContent className="w-auto p-0">
    <Calendar mode="single" selected={date} onSelect={handleDateChange} initialFocus />
  </PopoverContent>
</Popover>
```

**Key Props Passed to Calendar**:
- `mode="single"` - Single date selection mode
- `selected={date}` - Controlled component, current date state
- `onSelect={handleDateChange}` - Callback when date is selected
- `initialFocus` - Auto-focus calendar when popover opens

### Date State Management

**State Declaration** (`receipt-details-card.tsx:71-73`):
```tsx
const [date, setDate] = useState<Date | undefined>(
  parseDateString(initialData?.receipt_date ?? initialData?.date)
);
```

**Date Change Handler** (`receipt-details-card.tsx:263-274`):
```tsx
const handleDateChange = (selectedDate: Date | undefined | string) => {
  if (typeof selectedDate === 'string') {
    // From native date input (yyyy-MM-dd)
    setDate(parseISO(selectedDate + "T00:00:00"));
  } else {
    // From Calendar component
    setDate(selectedDate);
  }
};
```

The handler accepts both:
- `string` - From mobile native date input (e.g., "2025-12-30")
- `Date | undefined` - From Calendar component's onSelect callback

### Calendar Component

**File**: `dws-app/src/components/ui/calendar.tsx`

Wrapper around react-day-picker's DayPicker component:
- **Library**: react-day-picker version 8.10.1 (from package.json:33)
- **Props spread**: All props are passed through via `{...props}` at line 70
- **Custom icons**: Uses ChevronLeft/ChevronRight from lucide-react for navigation (lines 62-68)
- **Styling**: Extensive Tailwind classes for dark theme via `classNames` prop (lines 20-60)

### Popover Component

**File**: `dws-app/src/components/ui/popover.tsx`

Standard Radix UI popover primitive wrapper:
- **Library**: @radix-ui/react-popover version 1.1.14 (from package.json:19)
- **Portal**: Content rendered in portal via `PopoverPrimitive.Portal` (line 27)
- **Defaults**: `align="center"`, `sideOffset={4}` (lines 22-23)
- **Animations**: CSS animations for open/close transitions (line 33)

### Mobile Detection Hook

**File**: `dws-app/src/hooks/use-mobile.tsx`

Determines if device should use native date input:
```tsx
const checkIfMobile = () => {
  const widthIsMobile = window.innerWidth < 768;
  const agentIsMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);
  setIsMobile(widthIsMobile || agentIsMobile);
}
```

- Threshold: 768px window width
- Also checks user agent for iOS/Android devices
- Has console.log statements for debugging (lines 13-14)

### PopoverContent Default Width

The PopoverContent component has a default `className` that includes `w-72` (288px width) in `popover.tsx:33`:
```tsx
className={cn(
  "bg-popover text-popover-foreground ... w-72 ... rounded-md border p-4 shadow-md ...",
  className
)}
```

However, in receipt-details-card.tsx:308, the className is overridden to `className="w-auto p-0"`, removing the fixed width and padding.

## Code References

- `dws-app/src/components/receipt-details-card.tsx:297-312` - Desktop datepicker JSX
- `dws-app/src/components/receipt-details-card.tsx:263-274` - handleDateChange function
- `dws-app/src/components/receipt-details-card.tsx:71-73` - Date state initialization
- `dws-app/src/components/receipt-details-card.tsx:50` - isMobile hook usage
- `dws-app/src/components/ui/calendar.tsx:10-73` - Calendar component wrapper
- `dws-app/src/components/ui/calendar.tsx:62-68` - Custom navigation icon components
- `dws-app/src/components/ui/popover.tsx:20-40` - PopoverContent with portal
- `dws-app/src/hooks/use-mobile.tsx:5-27` - Mobile detection hook

## Architecture Documentation

### Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| react-day-picker | 8.10.1 | Calendar widget library |
| @radix-ui/react-popover | 1.1.14 | Popover primitive |
| date-fns | 3.6.0 | Date formatting (devDependency) |
| lucide-react | 0.511.0 | Icons including CalendarIcon |

### Data Flow

1. User clicks Button trigger (line 300)
2. Popover opens via Radix UI state management
3. Calendar renders with current `selected={date}` value
4. User clicks a day in the calendar
5. react-day-picker calls `onSelect` callback with Date object
6. `handleDateChange(Date)` is called (line 309)
7. Since it's a Date object (not string), line 272 executes: `setDate(selectedDate)`
8. State updates, triggering re-render
9. Button text updates to show formatted date via `format(date, "PPP")`

### Important Implementation Details

1. **Calendar mode="single"**: In react-day-picker v8, this mode expects the `onSelect` callback signature: `(date: Date | undefined) => void`

2. **No explicit popover close**: The implementation does not explicitly close the popover after date selection. Standard behavior relies on Radix UI's modal/focus management.

3. **Custom classNames on Calendar**: The calendar.tsx component applies extensive custom `classNames` for styling, which override default react-day-picker classes.

4. **IconLeft/IconRight components**: Custom navigation icons defined at lines 62-68 of calendar.tsx.

## Open Questions

1. Does the Popover close after date selection, or does it remain open?
2. Is there a click handler conflict between the Calendar day buttons and any parent elements?
3. Are the react-day-picker v8 props being passed correctly through the spread operator?
4. Does the `initialFocus` prop work correctly with Radix UI's focus management?
