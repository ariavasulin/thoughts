# ARI-44: Date Display Bug Around Dec 16

## Summary

Visual glitch where date rendering changes unexpectedly around Dec 16 in the TimelineView.

## Root Cause Analysis

**Primary issue**: Inconsistent UTC/local timezone handling in the `getDateX` function.

## Key Findings

### 1. Date Header Structure
The timeline uses a two-row header system:
- **Top row**: Month labels (e.g., "Dec 25")
- **Bottom row**: Week labels showing the **Monday date** of each week (e.g., "15" for the week of Dec 15-21)

### 2. No Day-Level Labels
The current implementation only shows week markers (Mondays). Dec 16 itself is **never displayed as a header label** because it's a Tuesday. The closest displayed date would be "15" (Dec 15, the Monday of that week).

### 3. Potential Timezone/UTC Bug
In `TimelineView.tsx:274`, when positioning week headers:
```typescript
const x = getDateX(week.toISOString().split('T')[0]);
```

**This is the likely bug location.** `toISOString()` converts to UTC, but when the local timezone is behind UTC, dates near midnight can shift to the previous day.

### 4. Inconsistent Date Handling
The code mixes:
- `new Date()` local time construction (lines 31-32, 45-48)
- `toISOString().split('T')[0]` UTC extraction (lines 254, 274)
- Direct `getTime()` millisecond arithmetic (line 72)

This inconsistency can cause 1-day offset errors at date boundaries.

### 5. Week Width Calculation
Each week header has a fixed width of `DAY_WIDTH * 7` (168px) regardless of whether the week contains a month boundary.

## Affected Files

| File | Lines | Issue |
|------|-------|-------|
| `millflow/src/views/TimelineView.tsx` | 70-74 | `getDateX` function mixes UTC and local time |
| `millflow/src/views/TimelineView.tsx` | 274 | Week header positioning uses `toISOString()` (UTC) |
| `millflow/src/views/TimelineView.tsx` | 254 | Month header positioning uses `toISOString()` (UTC) |
| `millflow/src/data/timelineData.ts` | - | Date constants and mock data |

## Technical Deep Dive

### getDateX Function (lines 70-74)
```typescript
const getDateX = (dateStr: string) => {
  const date = new Date(dateStr);  // String parsed as local midnight
  const diffDays = Math.floor((date.getTime() - dateRange.start.getTime()) / (1000 * 60 * 60 * 24));
  return diffDays * DAY_WIDTH;
};
```

The `dateRange.start` is created as `new Date('2025-11-01')` which is local time, while dates passed through `toISOString()` are in UTC. This can cause positioning misalignment.

### Week Generation (lines 43-52)
```typescript
const weeks = useMemo(() => {
  const w: Date[] = [];
  const weekStart = new Date(dateRange.start);
  while (weekStart.getDay() !== 1) weekStart.setDate(weekStart.getDate() + 1);
  while (weekStart <= dateRange.end) {
    w.push(new Date(weekStart));
    weekStart.setDate(weekStart.getDate() + 7);
  }
  return w;
}, [dateRange]);
```

Nov 1, 2025 is a Saturday (getDay() = 6), advancing to Nov 3, 2025 (Monday). Weeks generated:
- Nov 3, 10, 17, 24
- Dec 1, 8, **15**, 22, 29
- Jan 5, 12, 19, 26...

Dec 16 falls within the Dec 15 week.

## Data Context

Mock data has high activity around Dec 16:
- Dec 14: 9 date references
- Dec 15: 8 date references
- Dec 16: 9 date references
- Dec 17: 9 date references
- Dec 18: 5 date references

Visual misalignment would be more noticeable in this area due to overlapping operation bars.

## Recommended Fix

Use consistent UTC arithmetic throughout, or convert all dates to UTC midnight before calculations:

```typescript
// Option 1: Normalize to UTC
const getDateX = (dateStr: string) => {
  const date = new Date(dateStr + 'T00:00:00Z');
  const start = new Date(dateRange.start.toISOString().split('T')[0] + 'T00:00:00Z');
  const diffDays = Math.round((date.getTime() - start.getTime()) / (1000 * 60 * 60 * 24));
  return diffDays * DAY_WIDTH;
};

// Option 2: Use date-fns or similar library for reliable date arithmetic
```
