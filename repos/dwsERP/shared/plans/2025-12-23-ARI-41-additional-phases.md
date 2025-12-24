# ARI-41: Add Field Measurements, Drafting, and Installation Phases

## Overview

Extend the timeline view to include pre-production and post-production phases:
- **Field Measurements** - Pre-production (field work)
- **Drafting** - Pre-production (office work)
- **Installation** - Post-production (field work)

This extends the left-to-right span of the timeline beyond shop floor operations.

## Current State Analysis

The `TimelineView` currently renders 6 shop floor departments:
- **File**: `millflow/src/views/TimelineView.tsx:10`
- **Departments**: `['veneer', 'mill', 'cnc', 'bench', 'finish', 'assembly']`

The `TimelineSheet` interface has fixed properties for each department as `[startDate, endDate] | null`.

### Key Discoveries:
- Timeline rendering is already generic - iterates over `DEPARTMENTS` array and renders bars for each
- Colors defined in `TIMELINE_DEPT_COLORS` (`timelineData.ts:61-71`)
- `TimelineDepartment` type already includes some phases not rendered (`release`, `storage`, `delivery`)
- Mock data in `timelineData.ts` would need updates to include new phases

## Desired End State

The timeline displays phases in this order (left to right):
```
[Drafting] → [Field Measurements] → [Shop Floor: veneer/mill/cnc/bench/finish/assembly] → [Field Install] → [Delivery marker]
```

### Verification:
- Timeline legend shows all 9 departments with correct colors
- Clicking legend toggles visibility of each phase
- Mock data shows bars for new phases on some sheets
- Sheet detail popup includes new phases

## What We're NOT Doing

- Adding these phases to the main department list (`departments.ts`) - that's a separate concern
- Changing the schedule view (`ScheduleView.tsx`) - different data structure
- Adding real data - just mock data for visualization

## Implementation Approach

The timeline is already built to iterate over a departments array and render bars. We just need to:
1. Add new properties to the type
2. Add colors for the new phases
3. Expand the `DEPARTMENTS` array with correct ordering
4. Add mock data to verify rendering

---

## Phase 1: Update Types & Colors

### Overview
Add TypeScript types and color definitions for the new phases.

### Changes Required:

#### 1. Update TimelineSheet Interface
**File**: `millflow/src/types/index.ts`
**Lines**: 192-204

Add new optional properties for the phases:

```typescript
export interface TimelineSheet {
  id: string;
  desc: string;
  release: string; // ISO date
  drafting: [string, string] | null; // NEW: Pre-production
  fieldMeasurements: [string, string] | null; // NEW: Pre-production
  veneer: [string, string] | null;
  mill: [string, string] | null;
  cnc: [string, string] | null;
  bench: [string, string] | null;
  finish: [string, string] | null;
  assembly: [string, string] | null;
  fieldInstall: [string, string] | null; // NEW: Post-production
  delivery: string; // ISO date
  status: number; // Slack days (negative = behind schedule)
}
```

#### 2. Update TimelineDepartment Type
**File**: `millflow/src/types/index.ts`
**Line**: 206

```typescript
export type TimelineDepartment = 'release' | 'drafting' | 'fieldMeasurements' | 'veneer' | 'mill' | 'cnc' | 'bench' | 'finish' | 'assembly' | 'fieldInstall' | 'storage' | 'delivery';
```

#### 3. Add Colors for New Phases
**File**: `millflow/src/data/timelineData.ts`
**Lines**: 61-71

```typescript
export const TIMELINE_DEPT_COLORS: Record<string, string> = {
  release: 'bg-gruvbox-fg-4',
  drafting: 'bg-gruvbox-blue-dim', // NEW: Office/design work
  fieldMeasurements: 'bg-gruvbox-purple-dim', // NEW: Field work
  veneer: 'bg-gruvbox-purple-bright',
  mill: 'bg-gruvbox-blue-bright',
  cnc: 'bg-gruvbox-aqua-bright',
  bench: 'bg-gruvbox-yellow-bright',
  finish: 'bg-gruvbox-orange-bright',
  assembly: 'bg-gruvbox-green-bright',
  fieldInstall: 'bg-gruvbox-purple-dim', // NEW: Field work (matches fieldMeasurements)
  storage: 'bg-gruvbox-fg-4',
  delivery: 'bg-gruvbox-red-bright',
};
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd millflow && npm run build`
- [ ] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] N/A for this phase (no visual changes yet)

---

## Phase 2: Update Mock Data

### Overview
Add sample data for the new phases to some sheets in the mock data.

### Changes Required:

#### 1. Update Mock Sheet Data
**File**: `millflow/src/data/timelineData.ts`
**Lines**: 4-58

Add the new phase properties to existing sheet objects. Not all sheets need all phases (some sheets skip certain operations). Example pattern:

```typescript
{
  id: 'SK-01',
  desc: 'Pantry 35043',
  release: '2025-11-11',
  drafting: ['2025-10-28', '2025-11-01'], // NEW: ~2 weeks before release
  fieldMeasurements: ['2025-11-04', '2025-11-06'], // NEW: ~1 week before release
  veneer: ['2025-11-11', '2025-11-18'],
  mill: ['2025-11-18', '2025-11-25'],
  cnc: ['2025-11-21', '2025-11-26'],
  bench: ['2025-12-08', '2025-12-17'],
  finish: ['2025-12-09', '2025-12-13'],
  assembly: ['2025-12-10', '2025-12-18'],
  fieldInstall: ['2025-12-26', '2025-12-28'], // NEW: After delivery
  delivery: '2025-12-24',
  status: -22
}
```

**Guidelines for mock data**:
- `drafting`: 1-2 weeks before release, ~3-5 days duration
- `fieldMeasurements`: After drafting, before release, ~2-3 days duration
- `fieldInstall`: After delivery date, ~2-4 days duration
- Some sheets can have `null` for these phases (not all sheets need field measurements)

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd millflow && npm run build`

#### Manual Verification:
- [ ] N/A for this phase (view not updated yet)

---

## Phase 3: Update TimelineView

### Overview
Modify the departments array to include new phases in the correct order and update the sheet detail popup.

### Changes Required:

#### 1. Update DEPARTMENTS Array
**File**: `millflow/src/views/TimelineView.tsx`
**Line**: 10

Change from:
```typescript
const DEPARTMENTS = ['veneer', 'mill', 'cnc', 'bench', 'finish', 'assembly'] as const;
```

To:
```typescript
const DEPARTMENTS = ['drafting', 'fieldMeasurements', 'veneer', 'mill', 'cnc', 'bench', 'finish', 'assembly', 'fieldInstall'] as const;
```

#### 2. Update Sheet Detail Popup
**File**: `millflow/src/views/TimelineView.tsx`
**Line**: 487

The popup iterates over a hardcoded list. Update to include new phases:

Change from:
```typescript
{['release', 'veneer', 'mill', 'cnc', 'bench', 'finish', 'assembly', 'delivery'].map(dept => {
```

To:
```typescript
{['release', 'drafting', 'fieldMeasurements', 'veneer', 'mill', 'cnc', 'bench', 'finish', 'assembly', 'fieldInstall', 'delivery'].map(dept => {
```

#### 3. Extend Date Range (if needed)
**File**: `millflow/src/views/TimelineView.tsx`
**Lines**: 56-66

The current date range is Nov 2025 - Mar 2026. Since drafting happens before release, we may need to extend the start date:

```typescript
const dateRange = useMemo(() => {
  const start = new Date('2025-10-01'); // Extended back 1 month for pre-production
  const end = new Date('2026-03-31');
  // ... rest unchanged
}, []);
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `cd millflow && npm run build`
- [ ] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Dev server runs: `cd millflow && npm run dev`
- [ ] Timeline view shows all 9 departments in legend (drafting → fieldMeasurements → veneer → mill → cnc → bench → finish → assembly → fieldInstall)
- [ ] Legend toggles work for all departments
- [ ] Pre-production bars (drafting, fieldMeasurements) render to the LEFT of shop floor operations
- [ ] Post-production bar (fieldInstall) renders to the RIGHT of shop floor operations, after delivery marker for some sheets
- [ ] Clicking a sheet row opens popup with all phases listed
- [ ] Field phases (fieldMeasurements, fieldInstall) use the same purple-dim color

---

## Testing Strategy

### Manual Testing Steps:
1. Start dev server (`npm run dev`)
2. Navigate to Timeline view
3. Verify legend shows 9 departments with correct colors
4. Verify pre-production phases appear left of veneer
5. Verify fieldInstall appears right of assembly
6. Toggle each department in legend - bars should hide/show
7. Click a sheet row - popup should show all phases with dates or N/A
8. Scroll left to see drafting/fieldMeasurements bars (before November if date range extended)

---

## References

- Ticket: [ARI-41](https://linear.app/ariav/issue/ARI-41)
- Parent: [ARI-38 - Timeline View Improvements](https://linear.app/ariav/issue/ARI-38)
- Research: `thoughts/shared/research/2025-12-23-ARI-41-additional-phases.md`
