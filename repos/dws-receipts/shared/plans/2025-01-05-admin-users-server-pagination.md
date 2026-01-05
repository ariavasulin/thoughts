# Admin Users Server-Side Pagination Implementation Plan

## Overview

Add server-side pagination to the `/api/admin/users` endpoint by updating the `get_auth_users_for_admin` RPC function to handle pagination, search, and filtering at the database level instead of client-side.

## Current State Analysis

**Current Flow:**
1. API calls `get_auth_users_for_admin()` → returns ALL users from `auth.users`
2. API makes second query to `user_profiles` for all user IDs
3. API merges data in JavaScript
4. API filters (deleted, search) in JavaScript
5. API paginates with `slice()` in JavaScript

**Problems:**
- Loads all users into memory before filtering/pagination
- Two separate database queries that must be joined client-side
- Search filtering happens after full data load
- Inefficient at scale (currently 82 users, but won't scale)

### Key Discoveries:
- `get_auth_users_for_admin` exists in prod DB: `dws-app/src/app/api/admin/users/route.ts:57-66`
- Similar pattern exists in `get_admin_receipts_with_phone` which joins 4 tables with `SECURITY DEFINER`
- Current function already filters `deleted_at IS NULL` at SQL level

## Desired End State

**New Flow:**
1. API calls `get_auth_users_for_admin(page, per_page, search, include_deleted)` → returns paginated, filtered, joined data
2. API calls `get_auth_users_count(search, include_deleted)` → returns total count
3. API returns results directly (no client-side processing)

**Verification:**
- Admin users page loads with pagination working
- Search filters results correctly
- Total count is accurate for pagination UI
- Performance is the same or better

## What We're NOT Doing

- Not adding pagination to receipts RPC function
- Not changing the frontend pagination UI (already works)
- Not adding cursor-based pagination (offset is fine for this scale)
- Not adding indexes (82 users doesn't need them)

## Implementation Approach

Update the RPC function to join `auth.users` with `user_profiles` directly and handle all filtering/pagination in SQL. Create a separate count function for total. Simplify the API route to just call these functions.

---

## Phase 1: Update RPC Function

### Overview
Replace the current `get_auth_users_for_admin` function with a new version that accepts pagination and search parameters, and joins with `user_profiles`.

### Changes Required:

#### 1. Update RPC Function in Supabase
**Location**: Supabase SQL Editor (prod database)

Drop and recreate the function with new signature:

```sql
-- Drop existing function
DROP FUNCTION IF EXISTS public.get_auth_users_for_admin();

-- Create new paginated version with user_profiles join
CREATE OR REPLACE FUNCTION public.get_auth_users_for_admin(
  page_num INTEGER DEFAULT 1,
  page_size INTEGER DEFAULT 50,
  search_query TEXT DEFAULT NULL,
  include_deleted BOOLEAN DEFAULT FALSE
)
RETURNS TABLE(
  id UUID,
  phone TEXT,
  created_at TIMESTAMPTZ,
  last_sign_in_at TIMESTAMPTZ,
  banned_until TIMESTAMPTZ,
  role TEXT,
  full_name TEXT,
  preferred_name TEXT,
  employee_id_internal TEXT,
  deleted_at TIMESTAMPTZ
)
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'auth', 'public'
AS $$
DECLARE
  offset_val INTEGER;
BEGIN
  offset_val := (page_num - 1) * page_size;

  RETURN QUERY
  SELECT
    au.id,
    au.phone,
    au.created_at,
    au.last_sign_in_at,
    au.banned_until,
    COALESCE(up.role, 'employee') AS role,
    up.full_name,
    up.preferred_name,
    up.employee_id_internal,
    up.deleted_at
  FROM auth.users au
  LEFT JOIN public.user_profiles up ON au.id = up.user_id
  WHERE
    -- Exclude auth-deleted users
    au.deleted_at IS NULL
    -- Handle soft-deleted profiles
    AND (include_deleted OR up.deleted_at IS NULL)
    -- Search filter (case-insensitive partial match)
    AND (
      search_query IS NULL
      OR search_query = ''
      OR LOWER(up.full_name) LIKE '%' || LOWER(search_query) || '%'
      OR LOWER(up.preferred_name) LIKE '%' || LOWER(search_query) || '%'
      OR LOWER(au.phone) LIKE '%' || LOWER(search_query) || '%'
      OR LOWER(up.employee_id_internal) LIKE '%' || LOWER(search_query) || '%'
    )
  ORDER BY au.created_at DESC
  LIMIT page_size
  OFFSET offset_val;
END;
$$;
```

#### 2. Create Count Function
**Location**: Supabase SQL Editor (prod database)

```sql
-- Create count function for pagination
CREATE OR REPLACE FUNCTION public.get_auth_users_count(
  search_query TEXT DEFAULT NULL,
  include_deleted BOOLEAN DEFAULT FALSE
)
RETURNS INTEGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path TO 'auth', 'public'
AS $$
DECLARE
  total INTEGER;
BEGIN
  SELECT COUNT(*)::INTEGER INTO total
  FROM auth.users au
  LEFT JOIN public.user_profiles up ON au.id = up.user_id
  WHERE
    au.deleted_at IS NULL
    AND (include_deleted OR up.deleted_at IS NULL)
    AND (
      search_query IS NULL
      OR search_query = ''
      OR LOWER(up.full_name) LIKE '%' || LOWER(search_query) || '%'
      OR LOWER(up.preferred_name) LIKE '%' || LOWER(search_query) || '%'
      OR LOWER(au.phone) LIKE '%' || LOWER(search_query) || '%'
      OR LOWER(up.employee_id_internal) LIKE '%' || LOWER(search_query) || '%'
    );

  RETURN total;
END;
$$;
```

### Success Criteria:

#### Automated Verification:
- [x] Functions exist: Run `SELECT routine_name FROM information_schema.routines WHERE routine_name IN ('get_auth_users_for_admin', 'get_auth_users_count');`
- [x] Test paginated query: `SELECT * FROM get_auth_users_for_admin(1, 10, NULL, FALSE);`
- [x] Test search: `SELECT * FROM get_auth_users_for_admin(1, 10, 'john', FALSE);`
- [x] Test count: `SELECT get_auth_users_count(NULL, FALSE);`

#### Manual Verification:
- [ ] Query returns expected columns
- [ ] Pagination returns correct slice of data
- [ ] Search filters correctly across all 4 fields

---

## Phase 2: Update API Route

### Overview
Simplify the API route to call the new RPC functions and return results directly.

### Changes Required:

#### 1. Update GET Handler
**File**: `dws-app/src/app/api/admin/users/route.ts`

Replace the data fetching logic (lines 55-150) with:

```typescript
export async function GET(request: Request) {
  const supabase = await createSupabaseServerClient();

  // Verify authentication
  const {
    data: { session },
    error: sessionError,
  } = await supabase.auth.getSession();

  if (sessionError) {
    console.error("GET /api/admin/users: Session error:", sessionError);
    return NextResponse.json({ error: 'Failed to get session' }, { status: 500 });
  }

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Verify admin role
  const { data: profile, error: profileError } = await supabase
    .from('user_profiles')
    .select('role')
    .eq('user_id', session.user.id)
    .single();

  if (profileError || !profile || profile.role !== 'admin') {
    return NextResponse.json({ error: 'Admin access required' }, { status: 403 });
  }

  try {
    // Parse query parameters
    const { searchParams } = new URL(request.url);
    const page = Math.max(1, parseInt(searchParams.get('page') || '1', 10));
    const perPage = Math.min(1000, Math.max(1, parseInt(searchParams.get('perPage') || '50', 10)));
    const search = searchParams.get('search') || null;
    const includeDeleted = searchParams.get('includeDeleted') === 'true';

    // Fetch paginated users with server-side filtering
    const { data: users, error: usersError } = await supabaseAdmin.rpc(
      'get_auth_users_for_admin',
      {
        page_num: page,
        page_size: perPage,
        search_query: search,
        include_deleted: includeDeleted,
      }
    );

    if (usersError) {
      console.error('GET /api/admin/users: Users query error:', usersError);
      return NextResponse.json({ error: usersError.message }, { status: 500 });
    }

    // Fetch total count for pagination
    const { data: total, error: countError } = await supabaseAdmin.rpc(
      'get_auth_users_count',
      {
        search_query: search,
        include_deleted: includeDeleted,
      }
    );

    if (countError) {
      console.error('GET /api/admin/users: Count query error:', countError);
      return NextResponse.json({ error: countError.message }, { status: 500 });
    }

    // Map to AdminUser type (handle null values)
    const mappedUsers: AdminUser[] = (users || []).map((u: {
      id: string;
      phone: string | null;
      created_at: string;
      last_sign_in_at: string | null;
      banned_until: string | null;
      role: string;
      full_name: string | null;
      preferred_name: string | null;
      employee_id_internal: string | null;
      deleted_at: string | null;
    }) => ({
      id: u.id,
      phone: u.phone || '',
      created_at: u.created_at,
      last_sign_in_at: u.last_sign_in_at || undefined,
      banned_until: u.banned_until || undefined,
      role: u.role as 'employee' | 'admin',
      full_name: u.full_name || undefined,
      preferred_name: u.preferred_name || undefined,
      employee_id_internal: u.employee_id_internal || undefined,
      deleted_at: u.deleted_at || null,
    }));

    return NextResponse.json({
      users: mappedUsers,
      page,
      perPage,
      total: total || 0,
    });

  } catch (error) {
    console.error('GET /api/admin/users: Unhandled error:', error);
    const errorMessage = error instanceof Error ? error.message : 'An unknown error occurred';
    return NextResponse.json({ error: errorMessage }, { status: 500 });
  }
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd dws-app && npm run build`
- [x] Linting passes: `cd dws-app && npm run lint`

#### Manual Verification:
- [ ] Admin users page loads correctly
- [ ] Pagination controls work (next/prev page)
- [ ] Search filters results in real-time
- [ ] Total count updates correctly when searching
- [ ] "Include deleted" toggle works

---

## Phase 3: Update JSDoc and Clean Up

### Overview
Update documentation comments and remove any dead code.

### Changes Required:

#### 1. Update JSDoc Comment
**File**: `dws-app/src/app/api/admin/users/route.ts`

Update the file header comment:

```typescript
/**
 * GET /api/admin/users
 *
 * Admin-only endpoint that returns all users with profile information.
 * Uses server-side pagination and filtering via database RPC functions.
 *
 * Query parameters:
 * - page: Page number (default: 1)
 * - perPage: Results per page (default: 50, max: 1000)
 * - search: Search query for name, phone, employee ID (server-side ILIKE)
 * - includeDeleted: Include soft-deleted users (default: false)
 */
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd dws-app && npm run build`

#### Manual Verification:
- [ ] Code is clean and readable

---

## Testing Strategy

### Manual Testing Steps:
1. Load admin users page - verify users display
2. Click page 2 - verify different users appear
3. Type in search box - verify results filter
4. Clear search - verify all users return
5. Toggle "include deleted" - verify count changes
6. Search + pagination - verify they work together

### Edge Cases:
- Empty search string should return all users
- Page beyond total should return empty array
- Very long search string should not break
- Special characters in search (apostrophes, etc.)

## Performance Considerations

- Two database round trips (data + count) vs one with window function
- At 82 users, difference is negligible (<5ms)
- ILIKE search without index is O(n) but fine at this scale
- If user count exceeds 1000, consider adding indexes on search fields

## References

- Current route: `dws-app/src/app/api/admin/users/route.ts`
- Similar pattern: `get_admin_receipts_with_phone` RPC function
- Research: `thoughts/shared/research/2025-12-18-admin-dashboard-performance.md`
