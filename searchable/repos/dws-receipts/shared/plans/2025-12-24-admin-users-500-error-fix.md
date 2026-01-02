# Fix 500 Error When Adding New Users via Admin Frontend

## Overview

Fix the duplicate key violation error that occurs when admins create new users via the frontend. The API endpoint attempts to INSERT into `user_profiles` after a database trigger has already created a basic profile, causing a primary key conflict.

## Current State Analysis

### The Problem
1. Admin submits user creation form → POST `/api/admin/users`
2. API creates auth user via `supabaseAdmin.auth.admin.createUser()`
3. Database trigger `on_auth_user_created` fires and inserts: `{user_id, role: 'employee'}`
4. API tries to INSERT: `{user_id, role, full_name, preferred_name, employee_id_internal}`
5. INSERT fails with `duplicate key value violates unique constraint "user_profiles_pkey"`
6. API deletes auth user as cleanup, returns 500 error

### Key Discoveries
- Trigger `on_auth_user_created` runs `handle_new_user()` which always inserts a basic profile
- The trigger was designed for self-signup flow, not admin creation
- Existing pattern in `Assets/user_sync_queries.sql` uses `ON CONFLICT (user_id) DO UPDATE`
- Existing pattern in `scripts/onboard-users.mjs` uses try-UPDATE-then-INSERT

## Desired End State

Admin user creation via the frontend works correctly:
- Auth user is created
- Profile is created/updated with all provided fields (role, full_name, preferred_name, employee_id_internal)
- API returns 201 with the created user data
- No 500 errors

### Verification
- Create a new user via admin UI with all fields populated
- Verify user appears in user list with correct data
- Verify user can log in via SMS OTP

## What We're NOT Doing

- Modifying the database trigger (would require migration, adds complexity)
- Changing the self-signup flow (trigger behavior is correct for that use case)
- Adding new database columns or tables

## Implementation Approach

Change the INSERT operation to UPSERT using Supabase's `.upsert()` method. This handles both scenarios:
1. **Trigger succeeds** (normal case): UPSERT updates the trigger-created record with full profile data
2. **Trigger fails** (edge case): UPSERT inserts a new record

This matches the existing pattern in `user_sync_queries.sql` and is the most robust solution.

---

## Phase 1: Fix the UPSERT Operation

### Overview
Modify the POST handler in `/api/admin/users` to use `.upsert()` instead of `.insert()`.

### Changes Required

#### 1. Update POST /api/admin/users
**File**: `dws-app/src/app/api/admin/users/route.ts`
**Lines**: 245-261

**Current code:**
```typescript
// Create user_profiles record
const { error: profileInsertError } = await supabaseAdmin
  .from('user_profiles')
  .insert({
    user_id: authData.user.id,
    role,
    full_name,
    preferred_name: preferred_name || null,
    employee_id_internal: employee_id_internal || null,
  });

if (profileInsertError) {
  console.error('POST /api/admin/users: Profile insert error:', profileInsertError);
  // User was created in auth but profile failed - try to clean up
  await supabaseAdmin.auth.admin.deleteUser(authData.user.id);
  return NextResponse.json({ error: 'Failed to create user profile' }, { status: 500 });
}
```

**New code:**
```typescript
// Create or update user_profiles record
// Note: Database trigger on_auth_user_created may have already created a basic profile,
// so we use upsert to handle both cases (update existing or insert new)
const { error: profileUpsertError } = await supabaseAdmin
  .from('user_profiles')
  .upsert(
    {
      user_id: authData.user.id,
      role,
      full_name,
      preferred_name: preferred_name || null,
      employee_id_internal: employee_id_internal || null,
    },
    { onConflict: 'user_id' }
  );

if (profileUpsertError) {
  console.error('POST /api/admin/users: Profile upsert error:', profileUpsertError);
  // User was created in auth but profile failed - try to clean up
  await supabaseAdmin.auth.admin.deleteUser(authData.user.id);
  return NextResponse.json({ error: 'Failed to create user profile' }, { status: 500 });
}
```

### Success Criteria

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd dws-app && npm run build`
- [x] ESLint passes: `cd dws-app && npm run lint`

#### Manual Verification:
- [ ] Create a new user via admin UI at `/users` with all fields:
  - Phone number (10-digit US)
  - Full name
  - Preferred name
  - Employee ID
  - Role (try both 'employee' and 'admin')
- [ ] Verify user appears in the user list with correct data
- [ ] Verify no 500 error in browser console or network tab
- [ ] Verify API returns 201 status code

**Implementation Note**: After completing this phase and automated verification passes, pause for manual confirmation that user creation works correctly via the admin UI before considering this complete.

---

## Testing Strategy

### Manual Testing Steps
1. Navigate to admin users page (`/users`)
2. Click "Add User" button
3. Fill in all fields:
   - Phone: `555-123-4567`
   - Full Name: `Test User`
   - Preferred Name: `Testy`
   - Employee ID: `TEST-001`
   - Role: `employee`
4. Submit the form
5. Verify:
   - No error toast appears
   - User appears in the list with correct data
   - Network tab shows 201 response

### Edge Cases to Verify
1. Create user with only required fields (phone, full_name)
2. Create user with admin role
3. Attempt to create user with duplicate phone number (should show 409 error)

## References

- Research document: `thoughts/shared/research/2025-12-24-admin-users-500-error.md`
- Similar pattern: `Assets/user_sync_queries.sql:38-54` (ON CONFLICT upsert)
- API endpoint: `dws-app/src/app/api/admin/users/route.ts:216-261`
