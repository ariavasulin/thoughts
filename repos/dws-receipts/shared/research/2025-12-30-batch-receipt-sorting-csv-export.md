---
date: 2025-12-30T12:44:27-08:00
researcher: ariasulin
git_commit: d085b71aecb5d16937184cf1d73b22840f8fd9c1
branch: test-branch
repository: DWS-Receipts
topic: "Batch receipt ordering by submission date and CSV export ordering by dept # then last name"
tags: [research, batch-review, csv-export, sorting, receipts]
status: complete
last_updated: 2025-12-30
last_updated_by: ariasulin
---

# Research: Batch Receipt & CSV Export Sorting

**Date**: 2025-12-30T12:44:27-08:00
**Researcher**: ariasulin
**Git Commit**: d085b71aecb5d16937184cf1d73b22840f8fd9c1
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question

What would it take to:
1. Make the batch receipt review order receipts by submission date
2. Make the CSV export order by department number, then last name

## Summary

**Finding 1: Batch Receipt Ordering by Submission Date - Already Implemented**

The batch review dashboard already orders receipts by submission date (`created_at`). The current implementation in `usePendingReceipts` hook uses `.order("created_at", { ascending: true })`, which sorts by submission date with oldest receipts first. If the user wants newest first, it's a one-line change.

**Finding 2: CSV Export Ordering by Dept # Then Last Name - Requires Schema Change**

The CSV export cannot currently sort by department number because **no department field exists in the database schema**. The `user_profiles` table has no `department` or `department_number` column. Implementation would require:
1. Database schema migration to add department field
2. Update to CSV export logic to sort by department first

The last name sorting is already partially implemented (currently sorts by lastName, then firstName).

---

## Detailed Findings

### 1. Batch Receipt Ordering (Already Works)

**Current Implementation**

| File | Line | Code |
|------|------|------|
| `dws-app/src/hooks/use-pending-receipts.ts` | 32 | `.order("created_at", { ascending: true })` |

The `usePendingReceipts` hook fetches pending receipts and orders them by `created_at` (submission timestamp) in ascending order (oldest first).

**Full Query (use-pending-receipts.ts:13-32)**:
```typescript
const { data, error } = await supabase
  .from("receipts")
  .select(`
    id, receipt_date, amount, status, category_id,
    categories!receipts_category_id_fkey (name),
    description, image_url,
    user_profiles (full_name, employee_id_internal)
  `)
  .eq("status", "Pending")
  .order("created_at", { ascending: true })  // <-- Submission date ordering
```

**What This Means**:
- Receipts are already sorted by submission date (oldest first)
- To change to newest first: change `ascending: true` to `ascending: false`
- No other code changes required

---

### 2. CSV Export Current Implementation

**Location**: `dws-app/src/components/receipt-dashboard.tsx:164-221`

**Current Sorting Logic (lines 196-199)**:
```typescript
const rows = Array.from(totalsMap.values()).sort((a, b) => {
  const ln = a.lastName.localeCompare(b.lastName)
  return ln !== 0 ? ln : a.firstName.localeCompare(b.firstName)
})
```

**Current Sort Order**:
1. Primary: Last name (alphabetical)
2. Secondary: First name (alphabetical)

**CSV Headers (line 165)**:
```typescript
const headers = ['LastName', 'FirstName', 'EmployeeNumber', 'TotalAmount']
```

---

### 3. Missing Department Field

**Problem**: No department field exists in the current schema.

**user_profiles table columns** (from `Docs/Database.md`):
| Column | Type | Description |
|--------|------|-------------|
| `user_id` | uuid | Primary Key |
| `role` | text | 'employee' or 'admin' |
| `full_name` | text | Full name (may be "Lastname, Firstname" format) |
| `preferred_name` | text | Display/nickname |
| `employee_id_internal` | text | Internal employee identifier |
| `created_at` | timestamp | Auto-generated |
| `updated_at` | timestamp | Auto-updated |
| `deleted_at` | timestamp | Soft delete |

**No department column exists.**

---

## What It Would Take

### Option A: If Only Submission Date Sorting Needed

**Effort**: ~1 minute (one-line change)

For batch review - already works (oldest first). To reverse to newest first:

```typescript
// dws-app/src/hooks/use-pending-receipts.ts:32
.order("created_at", { ascending: false })  // Change true to false
```

### Option B: Add Department # to CSV Export

**Effort**: Schema change + data migration + code updates

#### Step 1: Database Schema Migration

Create migration to add `department_number` column to `user_profiles`:

```sql
ALTER TABLE public.user_profiles
ADD COLUMN department_number TEXT;
```

#### Step 2: Populate Department Data

Either:
- Bulk update existing employees with department numbers
- Update admin user management UI to include department field
- Import department data from external system

#### Step 3: Update TypeScript Types

`dws-app/src/lib/types.ts` - Add to UserProfile and AdminUser interfaces:
```typescript
department_number?: string;
```

#### Step 4: Update CSV Export Logic

`dws-app/src/components/receipt-dashboard.tsx`:

1. Update data aggregation to include department (around line 182-194):
```typescript
// Add department_number to the totals map structure
const totalsMap = new Map<string, {
  lastName: string;
  firstName: string;
  employeeNumber: string;
  departmentNumber: string;  // Add this
  total: number;
}>()
```

2. Update sorting logic (lines 196-199):
```typescript
const rows = Array.from(totalsMap.values()).sort((a, b) => {
  // Sort by department first
  const dept = a.departmentNumber.localeCompare(b.departmentNumber)
  if (dept !== 0) return dept
  // Then by last name
  return a.lastName.localeCompare(b.lastName)
})
```

3. Optionally add department to CSV headers and output.

#### Step 5: Update Receipt Fetching

The `get_admin_receipts_with_phone` RPC function would need to include `department_number` in its SELECT:

```sql
SELECT
    ...
    up.department_number,
    ...
FROM public.receipts r
LEFT JOIN public.user_profiles up ON r.user_id = up.user_id
...
```

---

## Code References

- `dws-app/src/hooks/use-pending-receipts.ts:32` - Current submission date ordering
- `dws-app/src/components/receipt-dashboard.tsx:164-221` - CSV export function
- `dws-app/src/components/receipt-dashboard.tsx:196-199` - Current sorting logic
- `dws-app/src/lib/types.ts:30-39` - UserProfile interface (no department field)
- `Docs/Database.md:92-103` - user_profiles schema documentation

## Architecture Documentation

### Data Flow for Batch Review
```
usePendingReceipts hook
  → Supabase query with .order("created_at", { ascending: true })
  → React Query caching
  → batch-review-dashboard.tsx renders receipts in query order
```

### Data Flow for CSV Export
```
receipt-dashboard.tsx
  → filteredReceipts (from current filters/tabs)
  → Aggregate by employee into totalsMap
  → Sort by lastName, firstName
  → Generate CSV blob
  → Trigger browser download
```

## Open Questions

1. Should batch review show newest or oldest submissions first?
2. Where would department data come from? (HR system integration, manual entry, etc.)
3. Should department number be a numeric field or text? (Affects sorting behavior)
4. Is the CSV export meant to show individual receipts or just employee totals? (Currently shows totals only)
