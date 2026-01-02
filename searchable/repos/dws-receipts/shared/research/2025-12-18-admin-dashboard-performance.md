---
date: 2025-12-18T20:25:19-0800
researcher: Claude
git_commit: 13eaf60dda5f35a03116a36ae4aa9f1313d9f423
branch: receipt-editing
repository: DWS-Receipts
topic: "Admin Dashboard Performance and Database Caching"
tags: [research, codebase, dashboard, performance, caching, supabase]
status: complete
last_updated: 2025-12-18
last_updated_by: Claude
---

# Research: Admin Dashboard Performance and Database Caching

**Date**: 2025-12-18T20:25:19-0800
**Researcher**: Claude
**Git Commit**: 13eaf60dda5f35a03116a36ae4aa9f1313d9f423
**Branch**: receipt-editing
**Repository**: DWS-Receipts

## Research Question
Explore database caching and opportunities for speeding up the admin dashboard, which is taking a while to load the various tables.

## Summary

The admin dashboard currently has **no caching layer** at any level. Data fetching follows a straightforward pattern: component mounts → `useEffect` fires → `fetch()` calls API → API queries Supabase → response rendered. Every tab switch, page navigation, or dialog close triggers a full data re-fetch. The database has **510 receipts** with joins across 4 tables but **no indexes beyond primary keys**.

Key findings:
- **No React Query/SWR**: Manual `useState` + `useEffect` for all data fetching
- **No HTTP caching**: No `Cache-Control` headers, no Next.js `revalidate` config
- **No client-side caching**: Data discarded on tab switch, refetched entirely
- **Single RPC function** handles admin queries with 4-table join
- **Client-side filtering**: Search happens after full data fetch

## Detailed Findings

### 1. Data Fetching Architecture

#### Admin Dashboard Flow
**File**: `dws-app/src/components/receipt-dashboard.tsx`

The dashboard uses manual state management (lines 40-63):
```typescript
const [receipts, setReceipts] = useState<Receipt[]>([])
const [loading, setLoading] = useState<boolean>(true)
const [filterStatus, setFilterStatus] = useState<string>("all")
const [refreshTrigger, setRefreshTrigger] = useState(0)
```

Data fetching happens in `useEffect` (lines 95-174):
- Triggers on: `filterStatus`, `dateRange`, or `refreshTrigger` changes
- Builds query params for status filter and date range
- Calls `GET /api/admin/receipts?status=X&fromDate=Y&toDate=Z`
- Normalizes status to lowercase after receipt

**Refresh Pattern** (lines 61-63):
```typescript
const [refreshTrigger, setRefreshTrigger] = useState(0)
const triggerRefresh = () => setRefreshTrigger(prev => prev + 1)
```

Every mutation (edit, delete, bulk update) increments this counter to trigger full re-fetch.

#### API Route Handler
**File**: `dws-app/src/app/api/admin/receipts/route.ts`

**Authentication** (lines 18-46):
1. Gets session from Supabase
2. Queries `user_profiles` for role check
3. Validates admin role

**Data Query** (lines 59-66):
```typescript
const { data, error } = await supabase.rpc('get_admin_receipts_with_phone', {
  status_filter: statusFilter || null,
  from_date: fromDate || null,
  to_date: toDate || null,
})
```

**Response Mapping** (lines 74-103):
- Transforms each receipt's `image_url` to public URL via `supabase.storage.getPublicUrl()`
- Maps database columns to frontend interface properties

---

### 2. Database Structure

#### Tables
| Table | Rows | RLS |
|-------|------|-----|
| receipts | 510 | disabled |
| user_profiles | 75 | enabled |
| categories | 5 | disabled |

#### RPC Function
**Name**: `get_admin_receipts_with_phone`

**Definition**:
```sql
SELECT
    r.id, r.receipt_date, r.amount, r.status, r.description, r.image_url,
    r.category_id, r.user_id, r.created_at, r.updated_at,
    c.name as category_name,
    up.full_name, up.preferred_name, up.employee_id_internal,
    au.phone
FROM public.receipts r
LEFT JOIN public.user_profiles up ON r.user_id = up.user_id
LEFT JOIN auth.users au ON r.user_id = au.id
LEFT JOIN public.categories c ON r.category_id = c.id
WHERE
    (status_filter IS NULL OR status_filter = 'all' OR r.status = status_filter)
    AND (from_date IS NULL OR r.receipt_date >= from_date)
    AND (to_date IS NULL OR r.receipt_date < to_date)
ORDER BY r.receipt_date DESC, r.created_at DESC;
```

**Characteristics**:
- SECURITY DEFINER: Runs with elevated privileges
- 4-table join on every call
- No pagination in database query
- Returns ALL matching receipts

#### Indexes
| Table | Index | Type |
|-------|-------|------|
| categories | categories_pkey | btree(id) |
| categories | categories_name_key | btree(name) |
| receipts | receipts_pkey | btree(id) |
| user_profiles | user_profiles_pkey | btree(user_id) |

**Missing indexes**:
- `receipts.user_id` (used in join)
- `receipts.status` (used in WHERE filter)
- `receipts.receipt_date` (used in WHERE filter and ORDER BY)
- `receipts.category_id` (used in join)

---

### 3. Client-Side Data Processing

#### Filtering
**File**: `dws-app/src/components/receipt-dashboard.tsx` (lines 177-185)

Search filtering happens client-side AFTER server fetch:
```typescript
filteredReceipts = receipts.filter(receipt => {
  const searchLower = searchQuery.toLowerCase()
  return employeeName.includes(searchLower) || description.includes(searchLower)
})
```

This means ALL receipts are fetched even when searching for a specific term.

#### Sorting
**File**: `dws-app/src/components/receipt-table.tsx` (lines 77-97)

Sorting via `useMemo` on `rowData`:
```typescript
const sortedData = useMemo(() => {
  if (!sortField || !sortDirection) return rowData
  return [...rowData].sort((a, b) => { /* comparison logic */ })
}, [rowData, sortField, sortDirection])
```

#### Pagination
**File**: `dws-app/src/components/receipt-table.tsx` (lines 100-103)

Client-side pagination via slice:
```typescript
const paginatedData = useMemo(() => {
  const startIndex = (currentPage - 1) * pageSize
  return sortedData.slice(startIndex, startIndex + pageSize)
}, [sortedData, currentPage, pageSize])
```

**Key Pattern**: All 510 receipts are fetched, then sorted, then paginated client-side.

---

### 4. Refresh Triggers

| Action | Trigger Method | Effect |
|--------|---------------|--------|
| Edit receipt | `triggerRefresh()` | Full re-fetch |
| Delete receipt | `triggerRefresh()` | Full re-fetch |
| Bulk status update | `window.location.reload()` | Full page reload |
| Tab switch | `setFilterStatus()` | Full re-fetch |
| Date range change | `setDateRange()` | Full re-fetch |
| Search input | `setSearchQuery()` | Client-side filter only |

---

### 5. User Management Dashboard

**File**: `dws-app/src/components/user-management-dashboard.tsx`

Similar pattern with additional concern:

**API Call** (lines 42-43):
```typescript
const response = await fetch("/api/admin/users?perPage=1000")
```

Fetches all users with hardcoded high limit to ensure getting everyone.

**Refresh Pattern** (line 116):
```typescript
onClick={() => window.location.reload()}
```

Uses full page reload for refresh.

---

### 6. Supabase Client Configuration

#### Server Client
**File**: `dws-app/src/lib/supabaseServerClient.ts`

- Uses `@supabase/ssr` createServerClient
- Custom cookie lifetime: 6 months
- No request deduplication
- Fresh connection per request

#### Browser Client
**File**: `dws-app/src/lib/supabaseClient.ts`

- Lazy initialization pattern
- Single instance per page load
- No connection pooling

#### Admin Client
**File**: `dws-app/src/lib/supabaseAdminClient.ts`

- Uses service role key (bypasses RLS)
- No session persistence
- Used for admin user operations

---

### 7. Batch Review Dashboard

**File**: `dws-app/src/components/batch-review-dashboard.tsx`

Different pattern - queries Supabase directly from client (lines 44-60):
```typescript
const { data, error: supabaseError } = await supabase
  .from("receipts")
  .select(`
    id, receipt_date, amount, status, category_id,
    categories!receipts_category_id_fkey (name),
    description, image_url,
    user_profiles (full_name, employee_id_internal)
  `)
  .eq("status", "Pending")
```

This bypasses the API route but still has no caching.

## Code References

### Data Fetching
- `dws-app/src/components/receipt-dashboard.tsx:95-174` - Main admin dashboard fetch logic
- `dws-app/src/app/api/admin/receipts/route.ts:59-66` - Admin API RPC call
- `dws-app/src/components/batch-review-dashboard.tsx:44-60` - Direct Supabase query

### State Management
- `dws-app/src/components/receipt-dashboard.tsx:40-63` - Dashboard state definitions
- `dws-app/src/components/receipt-dashboard.tsx:61-63` - Refresh trigger pattern

### Client-Side Processing
- `dws-app/src/components/receipt-dashboard.tsx:177-185` - Search filtering
- `dws-app/src/components/receipt-table.tsx:77-97` - Sorting with useMemo
- `dws-app/src/components/receipt-table.tsx:100-103` - Client-side pagination

### Database
- RPC: `get_admin_receipts_with_phone` - 4-table join function
- Table: `receipts` - 510 rows, primary key index only

### API Routes
- `dws-app/src/app/api/admin/receipts/route.ts` - Admin receipts endpoint
- `dws-app/src/app/api/admin/users/route.ts` - User management endpoint
- `dws-app/src/app/api/receipts/route.ts` - General receipts endpoint

## Architecture Documentation

### Current Data Flow
```
User Action
    ↓
Component State Change (setFilterStatus, triggerRefresh)
    ↓
useEffect Triggers
    ↓
fetch() to API Route
    ↓
API: Auth Check (2 Supabase queries)
    ↓
API: RPC Call (4-table join)
    ↓
API: Transform each receipt.image_url to public URL
    ↓
JSON Response
    ↓
Component: setState(receipts)
    ↓
Component: Client-side search filter
    ↓
ReceiptTable: useMemo sorting
    ↓
ReceiptTable: useMemo pagination
    ↓
Render
```

### Existing Optimization Techniques
1. **useMemo for sorting/pagination** - Prevents recalculation on unrelated re-renders
2. **useCallback for fetch functions** - Stable function references
3. **Conditional fetch** - Only fetches when date range is complete

### Missing Caching Layers
1. **React Query / SWR** - For request deduplication, background refetch, cache invalidation
2. **HTTP Cache Headers** - For browser/CDN caching of GET requests
3. **Next.js Data Cache** - For route handler response caching
4. **Server-side pagination** - Database-level limit/offset
5. **Database indexes** - For query performance on filtered columns

## Historical Context (from thoughts/)

Relevant findings from previous research:

- `thoughts/shared/research/ARI-11_dashboard-freeze-root-cause-analysis.md` - Dashboard freeze caused by Radix UI dialog cleanup issues, unrelated to data fetching
- `thoughts/shared/research/2025-12-17-admin-user-management-feasibility.md` - Notes: "Search Performance: With many users, client-side filtering may be slow"
- `thoughts/shared/plans/2025-12-17-admin-user-management.md` - Notes: "If user count grows significantly, consider database RPC for server-side search"

## Related Research

No previous research on caching or database performance optimization exists in the thoughts directory.

## Open Questions

1. What is the actual load time breakdown (network vs render vs database query)?
2. How many concurrent admin users typically access the dashboard?
3. Is there a specific table or view that's slowest?
4. What's the expected data growth rate (receipts per month)?
5. Are there any Supabase connection pool settings configured?
