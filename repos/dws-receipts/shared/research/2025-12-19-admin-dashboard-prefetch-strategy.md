---
date: 2025-12-19T11:04:02Z
researcher: Claude
git_commit: 1d6b8ab8feb694607fae21e7e90cbcbd70b66811
branch: test-branch
repository: DWS-Receipts
topic: "Admin Dashboard Prefetch Strategy for Batch Receipts and User Accounts"
tags: [research, codebase, prefetching, react-query, admin-dashboard, batch-review, user-management]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
---

# Research: Admin Dashboard Prefetch Strategy for Batch Receipts and User Accounts

**Date**: 2025-12-19T11:04:02Z
**Researcher**: Claude
**Git Commit**: 1d6b8ab8feb694607fae21e7e90cbcbd70b66811
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question
How to implement background prefetching of batch receipts and user accounts data when an admin loads the dashboard, so those pages are instantly responsive when navigated to.

## Summary

The admin dashboard consists of three main pages:
1. **`/dashboard`** - Main receipt management (uses React Query with caching)
2. **`/batch-review`** - Batch approval UI (uses direct Supabase queries, no caching)
3. **`/users`** - User management (uses traditional fetch, no caching)

React Query is already set up with a 5-minute stale time and 10-minute garbage collection. However, only the main dashboard uses React Query - the batch-review and users pages use direct fetches without caching.

**Implementation Strategy**:
1. Create React Query hooks for batch-review and users data
2. Add a prefetch hook that triggers on dashboard mount
3. Use React Query's `queryClient.prefetchQuery()` to load data in background
4. Update batch-review and users pages to use the cached data

## Detailed Findings

### Admin Dashboard Structure

**Pages and Routes:**
- `/dashboard` - `dws-app/src/app/dashboard/page.tsx`
- `/batch-review` - `dws-app/src/app/batch-review/page.tsx`
- `/users` - `dws-app/src/app/users/page.tsx`

**Layout:**
- `dws-app/src/app/dashboard/layout.tsx` - Wraps all admin pages with `QueryProvider`

**Key Components:**
- `dws-app/src/components/receipt-dashboard.tsx` - Main receipt dashboard
- `dws-app/src/components/batch-review-dashboard.tsx` - Batch review interface
- `dws-app/src/components/user-management-dashboard.tsx` - User management

### Current Data Fetching Patterns

**React Query (with caching) - Dashboard Only:**
- `dws-app/src/hooks/use-admin-receipts.ts` - Custom hook for admin receipts
- Uses query key factory: `adminReceiptsKeys.list(params)`
- 5-minute stale time, 10-minute garbage collection
- Automatic cache invalidation after mutations

**Direct Supabase Queries (no caching) - Batch Review:**
- Queries `receipts` table filtered by `status = "Pending"`
- Joins `categories` and `user_profiles` tables
- Fetches on component mount via `useEffect`
- No caching between page navigations

**Traditional Fetch (no caching) - Users:**
- Calls `GET /api/admin/users?perPage=1000`
- Fetches on component mount via `useEffect` with `useCallback`
- No caching between page navigations

### Batch Review Data Requirements

**Current Query** (`batch-review-dashboard.tsx:44-63`):
```typescript
supabase
  .from("receipts")
  .select(`
    id, receipt_date, amount, status, category_id,
    categories!receipts_category_id_fkey (name),
    description, image_url,
    user_profiles (full_name, employee_id_internal)
  `)
  .eq("status", "Pending")
  .order("created_at", { ascending: true })
```

**Data Shape** - Maps to `Receipt` interface with:
- `id`, `employeeName`, `employeeId`, `date`, `amount`
- `category`, `description`, `status`, `image_url`

### User Management Data Requirements

**Current API Call** (`user-management-dashboard.tsx:42`):
```typescript
fetch("/api/admin/users?perPage=1000")
```

**API Endpoint** (`/api/admin/users/route.ts:17-144`):
1. Fetches auth users via `supabaseAdmin.auth.admin.listUsers()`
2. Fetches profiles via `user_profiles` table query
3. Merges auth and profile data into `AdminUser[]`

**Data Shape** - `AdminUser` interface with:
- Auth data: `id`, `phone`, `created_at`, `last_sign_in_at`, `banned_until`
- Profile data: `role`, `full_name`, `preferred_name`, `employee_id_internal`, `deleted_at`

### React Query Configuration

**Query Client** (`lib/query-client.ts`):
```typescript
defaultOptions: {
  queries: {
    staleTime: 5 * 60 * 1000,     // 5 minutes
    gcTime: 10 * 60 * 1000,        // 10 minutes
    refetchOnWindowFocus: false,
    retry: 1,
  }
}
```

**QueryProvider** wraps all admin routes via `dashboard/layout.tsx`.

## Code References

- `dws-app/src/app/dashboard/layout.tsx` - QueryProvider wrapper
- `dws-app/src/lib/query-client.ts` - React Query configuration
- `dws-app/src/hooks/use-admin-receipts.ts` - Existing React Query hook pattern
- `dws-app/src/components/receipt-dashboard.tsx:96-110` - React Query usage example
- `dws-app/src/components/batch-review-dashboard.tsx:40-88` - Batch review data fetch
- `dws-app/src/components/user-management-dashboard.tsx:37-60` - User fetch pattern

## Architecture Documentation

### Current State
- React Query is configured but only used for main receipts dashboard
- Batch review and users pages fetch data independently on each visit
- No prefetching or background loading mechanism exists

### Implementation Approach

**Step 1: Create React Query hooks for batch review and users**

New files needed:
- `dws-app/src/hooks/use-pending-receipts.ts` - For batch review data
- `dws-app/src/hooks/use-admin-users.ts` - For user management data

**Step 2: Create a prefetch hook**

New file:
- `dws-app/src/hooks/use-admin-prefetch.ts` - Triggers prefetch on mount

**Step 3: Update dashboard to trigger prefetch**

Modify `receipt-dashboard.tsx` to call prefetch hook on mount.

**Step 4: Update batch-review and users pages**

Update components to use the new React Query hooks instead of direct fetches.

## Open Questions

1. Should prefetch be delayed slightly (e.g., 100-500ms) to ensure main content loads first?
2. Should we use `prefetchQuery` (blocks until complete) or `fetchQuery` with lower priority?
3. Should we add loading skeletons to batch-review and users pages while cache populates?
