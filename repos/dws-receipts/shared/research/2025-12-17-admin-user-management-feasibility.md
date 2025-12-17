---
date: 2025-12-17T23:12:24Z
researcher: Claude
git_commit: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
branch: receipt-editing
repository: DWS-Receipts
topic: "Admin User Management Features Feasibility Analysis"
tags: [research, codebase, admin-dashboard, user-management, supabase-auth]
status: complete
last_updated: 2025-12-17
last_updated_by: Claude
---

# Research: Admin User Management Features Feasibility Analysis

**Date**: 2025-12-17T23:12:24Z
**Researcher**: Claude
**Git Commit**: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
**Branch**: receipt-editing
**Repository**: DWS-Receipts

## Research Question
What is needed to add account management features to the admin dashboard? Features include: adding new users, deleting users, and changing credentials.

## Summary

Adding user management features to the admin dashboard is **feasible** with the existing architecture. The codebase already has:
- Admin role verification patterns
- Server-side Supabase clients
- An existing onboarding script that demonstrates user creation via Admin API
- User profile table structure with role support

**Key requirements:**
1. **Service Role Key**: User management requires the Supabase service_role key (not the anon key)
2. **Server-Side Only**: All admin operations must happen in API routes, never client-side
3. **New API Routes**: Need ~4-5 new API routes for user CRUD operations
4. **New UI Components**: User management page/modal for the admin dashboard
5. **Profile Sync**: Must manage both `auth.users` and `user_profiles` tables together

## Detailed Findings

### Current Admin Dashboard Architecture

**Location**: `dws-app/src/app/dashboard/page.tsx` and `dws-app/src/components/receipt-dashboard.tsx`

The admin dashboard currently handles receipt management with:
- Role verification via `user_profiles` table
- Client-side auth check with redirect to login
- Server API calls for data operations
- Header navigation with logout button

**Existing Admin Navigation** (`receipt-dashboard.tsx:413-459`):
- Company logo
- "Review Receipts" button → `/batch-review`
- "Refresh" button
- "Logout" button

A new "Manage Users" button would fit naturally in this navigation.

### Current User Profile System

**Table**: `user_profiles`
**Interface** (`dws-app/src/lib/types.ts:30-38`):
```typescript
interface UserProfile {
  user_id: string;              // FK to auth.users.id
  role: 'employee' | 'admin';
  full_name?: string;
  preferred_name?: string;
  employee_id_internal?: string;
  created_at?: string;
  updated_at?: string;
}
```

**Relationship to auth.users**:
- `auth.users` (Supabase managed): Phone, email, sessions, credentials
- `user_profiles` (Custom table): Application data (role, names, employee ID)

### Existing User Management Patterns

**Onboarding Script** (`dws-app/scripts/onboard-users.mjs`):
The codebase already has a script that demonstrates the exact patterns needed:

1. **List Users** (lines 57-67):
```javascript
const { data: { users }, error } = await supabase.auth.admin.listUsers({
  page: page,
  perPage: 1000
});
```

2. **Create User** (lines 127-143):
```javascript
const { data: authData, error: authError } = await supabase.auth.admin.createUser({
  phone: phonePlus,
  phone_confirm: true,
  user_metadata: {
    full_name: fullName,
    preferred_name: preferred,
    employee_id_internal: employeeId,
  }
});
```

3. **Update Profile** (lines 146-171):
```javascript
const { data: profileData, error: profileError, count } = await supabase
  .from('user_profiles')
  .update({
    role: 'employee',
    full_name: fullName,
    preferred_name: preferred || null,
    employee_id_internal: employeeId || null,
    updated_at: new Date().toISOString()
  })
  .eq('user_id', authData.user.id)
  .select();
```

### Supabase Admin API Methods Available

| Method | Purpose | Key Parameters |
|--------|---------|----------------|
| `auth.admin.createUser()` | Create new user | email, phone, password, email_confirm, phone_confirm, user_metadata, app_metadata, ban_duration |
| `auth.admin.listUsers()` | List all users | page (default: 1), perPage (default: 50) |
| `auth.admin.getUserById()` | Get single user | userId |
| `auth.admin.updateUserById()` | Update user | userId, email, phone, password, email_confirm, phone_confirm, user_metadata, app_metadata, ban_duration |
| `auth.admin.deleteUser()` | Delete user | userId, shouldSoftDelete (boolean) |
| `auth.admin.signOut()` | Invalidate sessions | userId, scope ('global' or 'local') |
| `auth.admin.inviteUserByEmail()` | Send invite email | email (sends confirmation email) |

**Key Behaviors:**
- `createUser()` does NOT send confirmation emails - use `inviteUserByEmail()` for that
- `phone_confirm: true` marks phone as verified without sending SMS
- At least one of `email` or `phone` is required for createUser
- `listUsers()` has NO built-in search/filter - must filter client-side or use database RPC
- `deleteUser(userId, true)` = soft delete (sets deleted_at), `deleteUser(userId)` = hard delete
- `updateUserById` user_metadata MERGES with existing, doesn't replace

**Common Error Codes:**
| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `email_exists` | 422 | Email already registered |
| `phone_exists` | 422 | Phone already registered |
| `user_not_found` | 404 | User doesn't exist |
| `weak_password` | 422 | Password too weak |
| `over_request_rate_limit` | 429 | Too many requests |

### Security Requirements

**Service Role Key**: Required for admin operations
- Current server client uses anon key
- Need to create a separate admin client with service_role key
- Service role key should ONLY be used server-side (API routes)

**Current Server Client** (`dws-app/src/lib/supabaseServerClient.ts`):
```typescript
// Uses NEXT_PUBLIC_SUPABASE_ANON_KEY
// For admin operations, need service_role key
```

**Required Addition**:
```typescript
// lib/supabaseAdminClient.ts (new file)
import { createClient } from '@supabase/supabase-js'
import 'server-only' // Ensures this file can't be imported client-side

if (!process.env.SUPABASE_SERVICE_ROLE_KEY) {
  throw new Error('Missing SUPABASE_SERVICE_ROLE_KEY')
}

export const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY,
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false
    }
  }
)
```

**Important**: The `server-only` import is a Next.js feature that throws a build error if this module is accidentally imported in client code.

### API Routes Needed

#### 1. GET /api/admin/users
List all users with their profiles
- Admin role check (existing pattern)
- Use `auth.admin.listUsers()` with pagination
- Join with `user_profiles` table for application data
- Client-side search filtering (no server-side search in Admin API)
- Return combined user data

```typescript
// Example implementation
export async function GET(request: Request) {
  // ... auth check ...

  const { searchParams } = new URL(request.url)
  const page = parseInt(searchParams.get('page') || '1')
  const perPage = parseInt(searchParams.get('perPage') || '50')
  const search = searchParams.get('search') || ''

  const { data, error } = await supabaseAdmin.auth.admin.listUsers({ page, perPage })

  let users = data.users
  if (search) {
    users = users.filter(user =>
      user.phone?.includes(search) ||
      user.user_metadata?.full_name?.toLowerCase().includes(search.toLowerCase())
    )
  }

  return NextResponse.json({ users, page, perPage })
}
```

#### 2. POST /api/admin/users
Create new user
- Admin role check
- Validate input (phone format via existing `formatUSPhoneNumber`)
- Call `auth.admin.createUser()` with `phone_confirm: true`
- Create corresponding `user_profiles` record
- Handle `phone_exists` error (422)
- Return created user

```typescript
// Example implementation
export async function POST(request: Request) {
  // ... auth check ...

  const { phone, full_name, preferred_name, employee_id_internal, role } = await request.json()

  // Create auth user
  const { data: authData, error: authError } = await supabaseAdmin.auth.admin.createUser({
    phone: formatUSPhoneNumber(phone),
    phone_confirm: true,
    user_metadata: { full_name, preferred_name, employee_id_internal }
  })

  if (authError) {
    if (authError.code === 'phone_exists') {
      return NextResponse.json({ error: 'Phone number already registered' }, { status: 409 })
    }
    return NextResponse.json({ error: authError.message }, { status: 400 })
  }

  // Create profile record
  const { error: profileError } = await supabaseAdmin
    .from('user_profiles')
    .insert({
      user_id: authData.user.id,
      role: role || 'employee',
      full_name, preferred_name, employee_id_internal
    })

  return NextResponse.json({ user: authData.user }, { status: 201 })
}
```

#### 3. PATCH /api/admin/users/[id]
Update user
- Admin role check
- Validate input
- Update `auth.users` via `auth.admin.updateUserById()` (phone, phone_confirm)
- Update `user_profiles` record (role, names, employee_id)
- Optionally invalidate sessions if phone changed: `auth.admin.signOut(userId, 'global')`
- Return updated user

#### 4. DELETE /api/admin/users/[id]
Ban user (soft delete)
- Admin role check
- Ban user: `updateUserById(userId, { ban_duration: '876000h' })`
- Mark in user_profiles: `update({ deleted_at, role: 'deleted' })`
- Sign out all sessions: `auth.admin.signOut(userId, 'global')`
- Return success

```typescript
// Example implementation
export async function DELETE(request: Request, { params }: { params: { id: string } }) {
  // ... auth check (returns currentUserId) ...

  const userId = params.id

  // Prevent self-ban
  if (userId === currentUserId) {
    return NextResponse.json({ error: 'Cannot ban yourself' }, { status: 400 })
  }

  // 1. Ban the user (~100 years)
  await supabaseAdmin.auth.admin.updateUserById(userId, {
    ban_duration: '876000h'
  })

  // 2. Mark in user_profiles
  await supabaseAdmin
    .from('user_profiles')
    .update({ deleted_at: new Date().toISOString() })
    .eq('user_id', userId)

  // 3. Sign out all sessions
  await supabaseAdmin.auth.admin.signOut(userId, 'global')

  return NextResponse.json({ success: true })
}
```

### UI Components Needed

#### 1. User Management Page/Modal
**Options**:
- New route: `/admin/users` (separate page)
- Modal within dashboard (more integrated)

**Features needed**:
- User list table with columns: Name, Phone, Role, Employee ID, Created
- Search/filter functionality
- "Add User" button → form modal
- Row actions: Edit, Delete
- Role toggle (employee ↔ admin)

#### 2. User Form Component
Fields:
- Phone number (required, E.164 validation)
- Full name (required)
- Preferred name (optional)
- Employee ID (optional)
- Role selector (employee/admin)

#### 3. Confirmation Dialogs
- Delete user confirmation
- Role change confirmation (especially promoting to admin)

### Phone Number Handling

**Current Pattern** (`dws-app/src/app/login/page.tsx:15-39`):
```typescript
function formatUSPhoneNumber(input: string): string | null {
  const digits = input.replace(/\D/g, '');
  if (digits.length === 10) {
    return `+1${digits}`;
  } else if (digits.length === 11 && digits.startsWith('1')) {
    return `+${digits}`;
  }
  return null;
}
```

This same validation should be reused for user creation.

### Data Cascade Considerations

When deleting a user:
1. **Receipts**: User has receipts in the system. Options:
   - Prevent deletion if user has receipts
   - Soft delete (ban user instead)
   - Transfer receipts to "deleted user" placeholder
   - Hard delete with CASCADE (loses audit trail)

2. **User Profiles**: FK constraint to auth.users
   - Need to delete profile before user, or set up CASCADE
   - SQL to add cascade: `ALTER TABLE public.user_profiles ADD CONSTRAINT user_profiles_user_id_fkey FOREIGN KEY (user_id) REFERENCES auth.users(id) ON DELETE CASCADE;`

3. **Storage Objects**: If users have uploaded receipt images, you may hit:
   ```
   update or delete on table "users" violates foreign key constraint "objects_owner_fkey" on table "objects"
   ```
   - Solution: Delete user's storage objects first, or modify constraint to `ON DELETE SET NULL`

**Deletion Strategies:**

| Strategy | Method | Pros | Cons |
|----------|--------|------|------|
| Hard Delete | `deleteUser(userId)` | Clean removal | Loses audit trail, cascade issues |
| Soft Delete | `deleteUser(userId, true)` | Preserves data | May not work with all queries |
| Ban (Recommended) | `updateUserById(userId, { ban_duration: '876000h' })` | Account disabled, data preserved, can unban | User still exists in system |

**Recommended Soft Delete Implementation:**
```typescript
async function softDeleteUser(userId: string) {
  // 1. Ban the user (prevents login)
  await supabaseAdmin.auth.admin.updateUserById(userId, {
    ban_duration: '876000h' // ~100 years
  })

  // 2. Mark in user_profiles table
  await supabaseAdmin
    .from('user_profiles')
    .update({
      deleted_at: new Date().toISOString(),
      role: 'deleted' // or add a 'deleted' boolean
    })
    .eq('user_id', userId)

  // 3. Sign out all sessions
  await supabaseAdmin.auth.admin.signOut(userId, 'global')
}
```

### Existing Role Verification Pattern

**Used Throughout** (`dws-app/src/app/api/admin/receipts/route.ts:37-46`):
```typescript
// Verify admin role
const { data: profile } = await supabase
  .from('user_profiles')
  .select('role')
  .eq('user_id', session.user.id)
  .single();

if (!profile || profile.role !== 'admin') {
  return NextResponse.json({ error: 'Admin access required' }, { status: 403 });
}
```

This pattern should be reused in all new admin API routes.

## Code References
- `dws-app/src/app/dashboard/page.tsx:47-82` - Admin role verification pattern
- `dws-app/src/components/receipt-dashboard.tsx:413-459` - Admin dashboard navigation
- `dws-app/scripts/onboard-users.mjs:57-171` - User creation/update patterns
- `dws-app/src/lib/types.ts:30-38` - UserProfile interface
- `dws-app/src/lib/supabaseServerClient.ts` - Server-side Supabase client
- `dws-app/src/app/login/page.tsx:15-39` - Phone number formatting
- `dws-app/src/app/api/admin/receipts/route.ts:37-46` - Admin API auth pattern

## Architecture Documentation

### Required Environment Variables
```bash
# Existing
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...

# New (required for admin operations)
SUPABASE_SERVICE_ROLE_KEY=...  # NOT public, server-only
```

### Suggested File Structure
```
dws-app/src/
├── app/
│   └── api/
│       └── admin/
│           └── users/
│               ├── route.ts          # GET (list), POST (create)
│               └── [id]/
│                   └── route.ts      # GET (single), PATCH (update), DELETE
├── components/
│   ├── user-management-modal.tsx     # Or user-table.tsx
│   └── user-form.tsx
├── lib/
│   └── supabaseAdminClient.ts        # Service role client
```

## Implementation Estimate

| Component | Complexity | Notes |
|-----------|------------|-------|
| Admin client setup | Low | Add env var, create `supabaseAdminClient.ts` with `server-only` |
| GET /api/admin/users | Low-Medium | Pagination built-in, search is client-side filter |
| POST /api/admin/users | Medium | Phone validation, error handling for duplicates |
| PATCH /api/admin/users/[id] | Medium | Update both auth.users and user_profiles |
| DELETE /api/admin/users/[id] | Medium-High | Depends on cascade/ban decision, storage cleanup |
| User list UI | Medium | Table, pagination, search, shadcn/ui components exist |
| User form (create/edit) | Medium | Reuse existing phone validation from login |
| Dashboard integration | Low | Add button to existing navigation |

## Design Decisions (Confirmed)

1. **Deletion Strategy**: **Ban** (not hard delete)
   - Use `updateUserById(userId, { ban_duration: '876000h' })` (~100 years)
   - Preserves receipt history and audit trail
   - Avoids storage objects FK constraint issues
   - Can be undone if needed (set `ban_duration: 'none'`)

2. **Phone Change Impact**: **Yes**, invalidate sessions
   - Call `auth.admin.signOut(userId, 'global')` after phone change

3. **Self-Admin Prevention**: **Prevent self-ban**
   - Admins CANNOT ban themselves (check `userId !== currentUserId`)
   - Admins CAN change their own role

4. **Audit Logging**: **Not needed**

## Open Questions (Remaining)

1. **Bulk Import UI**: Expose the CSV import in the UI, or keep as script-only?
   - Current script: `dws-app/scripts/onboard-users.mjs`

2. **Search Performance**: With many users, client-side filtering may be slow
   - Option: Create database RPC function for server-side search

## Dependencies

```bash
# Required npm package (likely already installed)
npm install server-only

# Environment variable needed
SUPABASE_SERVICE_ROLE_KEY=eyJhbG...  # Get from Supabase Dashboard > Settings > API
```

## Related Research
- None yet in thoughts/shared/research/ for user management

## Next Steps

1. **Environment Setup**
   - Add `SUPABASE_SERVICE_ROLE_KEY` to `.env.local`
   - Add `server-only` package if not present

2. **Database Migration**
   - Add `deleted_at` column to `user_profiles` table

3. **Create Admin Client**
   - File: `dws-app/src/lib/supabaseAdminClient.ts`
   - Import `server-only` to prevent client-side usage

4. **Implement API Routes** (in order)
   - `GET /api/admin/users` - List with pagination (exclude deleted)
   - `POST /api/admin/users` - Create with validation
   - `PATCH /api/admin/users/[id]` - Update user + sign out on phone change
   - `DELETE /api/admin/users/[id]` - Ban user + mark deleted_at

5. **Build UI Components**
   - User table component (similar to receipt-table.tsx)
   - User form modal (create/edit)
   - Ban confirmation dialog
   - Add "Manage Users" button to dashboard navigation
