# Admin Dashboard Caching Implementation Plan

## Overview

Implement TanStack Query (React Query) caching for the admin dashboard to eliminate unnecessary re-fetching when switching between tabs. Tab switches will be instant (served from in-memory cache), while page refreshes and mutations will trigger fresh data fetches.

## Current State Analysis

- Tab switches trigger full API re-fetches (`receipt-dashboard.tsx:95-174`)
- No caching infrastructure exists
- Mutations use `window.location.reload()` or `triggerRefresh()` counter
- 510 receipts with 4-table joins, response time ~500ms-1s

### Key Discoveries:
- Dual state pattern for tabs: `activeTab` + `filterStatus` (`receipt-dashboard.tsx:40-41`)
- `filterStatus` changes trigger `useEffect` refetch (`receipt-dashboard.tsx:174`)
- Client-side search filtering after fetch (`receipt-dashboard.tsx:176-185`)
- No database indexes beyond primary keys

## Desired End State

- Tab switches are instant (< 50ms) using cached data
- Page refresh fetches fresh data (cache is in-memory only)
- After mutations (edit, delete, status change), data updates immediately
- Background refetching keeps data fresh without blocking UI

### Verification:
1. Switch between tabs rapidly - no loading spinners, instant data
2. Refresh page - see loading state, then fresh data
3. Edit a receipt - see changes immediately without page reload
4. Open Network tab - only one API call per unique filter combination

## What We're NOT Doing

- Server-side pagination (would require significant API refactoring)
- Persistent cache (localStorage/sessionStorage) - data should be fresh on page load
- Optimistic updates - unnecessary complexity for this scale
- Caching for employee dashboard (out of scope, can be added later)
- Caching for batch-review dashboard (out of scope, can be added later)

## Implementation Approach

Use TanStack Query v5 with:
- `staleTime: 5 * 60 * 1000` (5 minutes) - data considered fresh for 5 min
- `gcTime: 10 * 60 * 1000` (10 minutes) - cache garbage collected after 10 min
- Query keys based on filter parameters: `['admin-receipts', { status, dateRange }]`
- Mutation hooks that invalidate queries on success

---

## Phase 1: Install TanStack Query and Add Provider

### Overview
Install the package and wrap the app with QueryClientProvider.

### Changes Required:

#### 1. Install Dependencies
**Command**: Run in `dws-app/` directory
```bash
npm install @tanstack/react-query @tanstack/react-query-devtools
```

#### 2. Create Query Client Configuration
**File**: `dws-app/src/lib/query-client.ts` (new file)

```typescript
import { QueryClient } from '@tanstack/react-query'

export function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        // Data considered fresh for 5 minutes - no refetch on tab switch
        staleTime: 5 * 60 * 1000,
        // Cache kept for 10 minutes after last use
        gcTime: 10 * 60 * 1000,
        // Don't refetch on window focus (can enable later if desired)
        refetchOnWindowFocus: false,
        // Retry failed requests once
        retry: 1,
      },
    },
  })
}

// Singleton for client-side
let browserQueryClient: QueryClient | undefined = undefined

export function getQueryClient() {
  if (typeof window === 'undefined') {
    // Server: always make a new query client
    return makeQueryClient()
  } else {
    // Browser: reuse client across renders
    if (!browserQueryClient) {
      browserQueryClient = makeQueryClient()
    }
    return browserQueryClient
  }
}
```

#### 3. Create Query Provider Component
**File**: `dws-app/src/components/providers/query-provider.tsx` (new file)

```typescript
'use client'

import { QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { getQueryClient } from '@/lib/query-client'

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const queryClient = getQueryClient()

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

#### 4. Wrap Dashboard Layout with Provider
**File**: `dws-app/src/app/dashboard/layout.tsx` (new file)

```typescript
import { QueryProvider } from '@/components/providers/query-provider'

export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return <QueryProvider>{children}</QueryProvider>
}
```

### Success Criteria:

#### Automated Verification:
- [ ] Dependencies install without errors: `cd dws-app && npm install`
- [ ] Build succeeds: `npm run build`
- [ ] Lint passes: `npm run lint`
- [ ] Dev server starts: `npm run dev`

#### Manual Verification:
- [ ] Dashboard page loads without errors
- [ ] React Query Devtools icon appears in bottom-left corner (dev mode only)
- [ ] No console errors related to QueryClient

---

## Phase 2: Create Admin Receipts Query Hook

### Overview
Create a custom hook that wraps the admin receipts fetch with TanStack Query.

### Changes Required:

#### 1. Create Admin Receipts Hook
**File**: `dws-app/src/hooks/use-admin-receipts.ts` (new file)

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { Receipt } from '@/lib/types'

// Types for query parameters
interface AdminReceiptsParams {
  status?: string
  fromDate?: string
  toDate?: string
}

// Query key factory for consistent cache keys
export const adminReceiptsKeys = {
  all: ['admin-receipts'] as const,
  list: (params: AdminReceiptsParams) => ['admin-receipts', params] as const,
}

// Fetch function
async function fetchAdminReceipts(params: AdminReceiptsParams): Promise<Receipt[]> {
  const urlParams = new URLSearchParams()

  if (params.status && params.status !== 'all') {
    const dbStatus = params.status.charAt(0).toUpperCase() + params.status.slice(1)
    urlParams.append('status', dbStatus)
  }

  if (params.fromDate) {
    urlParams.append('fromDate', params.fromDate)
  }

  if (params.toDate) {
    urlParams.append('toDate', params.toDate)
  }

  const response = await fetch(`/api/admin/receipts?${urlParams.toString()}`)

  if (!response.ok) {
    const errorData = await response.json().catch(() => ({}))
    throw new Error(errorData.error || `Failed to fetch receipts: ${response.status}`)
  }

  const data = await response.json()

  if (!data.success) {
    throw new Error(data.error || 'Failed to fetch receipts')
  }

  // Normalize the data (matching existing logic in receipt-dashboard.tsx)
  return data.receipts.map((r: any) => ({
    ...r,
    date: r.date || r.receipt_date,
    category: r.category || r.category_name || 'Uncategorized',
  }))
}

// Main query hook
export function useAdminReceipts(params: AdminReceiptsParams) {
  return useQuery({
    queryKey: adminReceiptsKeys.list(params),
    queryFn: () => fetchAdminReceipts(params),
  })
}

// Hook to invalidate all admin receipts queries (used after mutations)
export function useInvalidateAdminReceipts() {
  const queryClient = useQueryClient()

  return () => {
    queryClient.invalidateQueries({ queryKey: adminReceiptsKeys.all })
  }
}

// Delete mutation hook
export function useDeleteReceipt() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async (receiptId: string) => {
      const response = await fetch(`/api/receipts?id=${receiptId}`, {
        method: 'DELETE',
      })

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}))
        throw new Error(errorData.error || 'Failed to delete receipt')
      }

      return response.json()
    },
    onSuccess: () => {
      // Invalidate all admin receipts queries to refetch
      queryClient.invalidateQueries({ queryKey: adminReceiptsKeys.all })
    },
  })
}

// Update mutation hook
export function useUpdateReceipt() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: async ({ id, data }: { id: string; data: Partial<Receipt> }) => {
      const response = await fetch(`/api/receipts?id=${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      })

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}))
        throw new Error(errorData.error || 'Failed to update receipt')
      }

      return response.json()
    },
    onSuccess: () => {
      // Invalidate all admin receipts queries to refetch
      queryClient.invalidateQueries({ queryKey: adminReceiptsKeys.all })
    },
  })
}
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `npm run build`
- [ ] Lint passes: `npm run lint`

#### Manual Verification:
- [ ] N/A - hook not yet integrated

---

## Phase 3: Integrate Query Hook into Receipt Dashboard

### Overview
Replace the manual `useEffect` fetch logic with the new `useAdminReceipts` hook.

### Changes Required:

#### 1. Update Receipt Dashboard Component
**File**: `dws-app/src/components/receipt-dashboard.tsx`

**Remove these state variables and effects:**
- Remove `receipts` state (line ~90)
- Remove `loading` state (line ~91)
- Remove `error` state (line ~92)
- Remove `refreshTrigger` state and `triggerRefresh` function (lines ~62-63)
- Remove the entire `useEffect` that fetches receipts (lines ~95-174)

**Add the query hook:**

```typescript
// At top of file, add import
import { useAdminReceipts, useDeleteReceipt, useInvalidateAdminReceipts } from '@/hooks/use-admin-receipts'

// Inside the component, replace state and effect with:
const {
  data: receipts = [],
  isLoading: loading,
  error: queryError,
  refetch
} = useAdminReceipts({
  status: filterStatus,
  fromDate: dateRange.from?.toISOString().split('T')[0],
  toDate: dateRange.to?.toISOString().split('T')[0],
})

const error = queryError?.message || null

const invalidateReceipts = useInvalidateAdminReceipts()
const deleteReceiptMutation = useDeleteReceipt()
```

**Update handleDeleteReceipt function:**

Replace the existing `handleDeleteReceipt` function with:

```typescript
const handleDeleteReceipt = async (receiptId: string) => {
  if (!receiptToDelete) return

  setIsDeleting(true)

  try {
    await deleteReceiptMutation.mutateAsync(receiptId)
    toast.success("Receipt deleted successfully")
    setReceiptToDelete(null)
  } catch (err) {
    toast.error(err instanceof Error ? err.message : "Failed to delete receipt")
  } finally {
    setIsDeleting(false)
  }
}
```

**Update handleEditSuccess callback:**

Replace `triggerRefresh()` calls with `invalidateReceipts()`:

```typescript
const handleEditSuccess = (updatedReceipt: Receipt) => {
  setEditingReceipt(null)
  invalidateReceipts()
  toast.success("Receipt updated successfully")
}
```

**Update refresh button:**

Replace `window.location.reload()` with `refetch()`:

```typescript
<Button variant="outline" size="sm" onClick={() => refetch()}>
  <RefreshCw className="h-4 w-4 mr-2" />
  Refresh
</Button>
```

**Remove unused state:**
- Remove `refreshTrigger` state
- Remove `triggerRefresh` function

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles without errors: `npm run build`
- [ ] Lint passes: `npm run lint`
- [ ] Dev server runs without errors: `npm run dev`

#### Manual Verification:
- [ ] Dashboard loads and displays receipts
- [ ] Switching tabs is instant (no loading spinner after first load)
- [ ] Date range filter works correctly
- [ ] Search filter works correctly (client-side filtering should still work)
- [ ] Editing a receipt shows changes immediately
- [ ] Deleting a receipt removes it immediately
- [ ] Refresh button fetches fresh data
- [ ] Page refresh fetches fresh data (not from cache)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Add Database Indexes

### Overview
Add indexes to improve query performance for the admin receipts query.

### Changes Required:

#### 1. Create Migration for Indexes
**File**: Apply via Supabase MCP or SQL editor

```sql
-- Index for filtering by status (most common filter)
CREATE INDEX IF NOT EXISTS idx_receipts_status ON receipts(status);

-- Index for date range queries
CREATE INDEX IF NOT EXISTS idx_receipts_receipt_date ON receipts(receipt_date);

-- Index for user lookups (used in joins)
CREATE INDEX IF NOT EXISTS idx_receipts_user_id ON receipts(user_id);

-- Index for category lookups (used in joins)
CREATE INDEX IF NOT EXISTS idx_receipts_category_id ON receipts(category_id);

-- Composite index for common query pattern: status + date
CREATE INDEX IF NOT EXISTS idx_receipts_status_date ON receipts(status, receipt_date);
```

### Success Criteria:

#### Automated Verification:
- [ ] Migration applies successfully via Supabase

#### Manual Verification:
- [ ] Initial page load feels faster
- [ ] Switching to filtered tabs (Pending, Approved, etc.) is responsive

---

## Phase 5: Clean Up and Remove Legacy Code

### Overview
Remove any remaining legacy refresh patterns now that caching is in place.

### Changes Required:

#### 1. Review and Remove window.location.reload() Calls
**File**: `dws-app/src/components/receipt-dashboard.tsx`

Search for and remove any remaining `window.location.reload()` calls that were used for refreshing data. These should now use `invalidateReceipts()` or `refetch()` instead.

#### 2. Remove Unused Imports
Clean up any unused imports from the refactoring.

### Success Criteria:

#### Automated Verification:
- [ ] No `window.location.reload()` calls remain for data refresh purposes
- [ ] Build succeeds: `npm run build`
- [ ] Lint passes: `npm run lint`

#### Manual Verification:
- [ ] All user flows work correctly
- [ ] No regressions in existing functionality

---

## Testing Strategy

### Unit Tests:
- N/A - no unit tests currently exist for this component

### Integration Tests:
- N/A - no integration tests currently exist

### Manual Testing Steps:
1. Load dashboard, verify receipts display
2. Switch between all 5 tabs rapidly - should be instant after first load
3. Apply date range filter, verify data updates
4. Clear date range filter, verify data updates
5. Search for a receipt, verify client-side filtering works
6. Edit a receipt, verify changes appear immediately
7. Delete a receipt, verify it disappears immediately
8. Click refresh button, verify data refetches
9. Hard refresh page (Cmd+R), verify fresh data loads
10. Open React Query Devtools, verify cache keys look correct

## Performance Considerations

- Initial load unchanged (~500ms-1s for first fetch)
- Subsequent tab switches: < 50ms (from cache)
- After mutations: ~500ms-1s (cache invalidation triggers refetch)
- Database indexes should improve initial fetch by ~20-50%

## Migration Notes

- No data migration required
- Backwards compatible - just adds caching layer
- React Query Devtools only appear in development mode
- Can be rolled back by reverting to previous implementation

## References

- Research document: `thoughts/shared/research/2025-12-18-admin-dashboard-performance.md`
- TanStack Query docs: https://tanstack.com/query/latest
- Current implementation: `dws-app/src/components/receipt-dashboard.tsx`
