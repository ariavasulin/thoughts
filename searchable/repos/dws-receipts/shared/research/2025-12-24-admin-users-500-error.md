---
date: 2025-12-24T16:38:22-08:00
researcher: ariasulin
git_commit: a82a08c74be48153d3eb86ba41722e5182f5eae7
branch: test-branch
repository: DWS-Receipts
topic: "500 Error When Adding New Users via Admin Frontend"
tags: [research, codebase, admin-users, database-trigger, user-profiles]
status: complete
last_updated: 2025-12-24
last_updated_by: ariasulin
---

# Research: 500 Error When Adding New Users via Admin Frontend

**Date**: 2025-12-24T16:38:22-08:00
**Researcher**: ariasulin
**Git Commit**: a82a08c74be48153d3eb86ba41722e5182f5eae7
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question
When trying to add new users using the admin frontend, the following error occurs:
```
api/admin/users:1 Failed to load resource: the server responded with a status of 500 ()
```

## Summary

The 500 error is caused by a **duplicate key violation** when inserting into the `user_profiles` table. This happens because:

1. A database trigger (`on_auth_user_created`) automatically inserts a `user_profiles` record whenever a new auth user is created
2. The API endpoint then tries to insert another `user_profiles` record for the same user
3. This violates the primary key constraint on `user_profiles.user_id`

## Detailed Findings

### Database Trigger: `on_auth_user_created`

There is a trigger on `auth.users` that fires on every INSERT:

```sql
-- Trigger definition
CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW
  EXECUTE FUNCTION handle_new_user();
```

The `handle_new_user()` function inserts a default profile:

```sql
BEGIN
  INSERT INTO public.user_profiles (user_id, role)
  VALUES (NEW.id, 'employee'); -- Default role is 'employee'
  RETURN NEW;
END;
```

### API Endpoint: POST /api/admin/users

Location: `dws-app/src/app/api/admin/users/route.ts:158-284`

The endpoint flow is:
1. Validates admin authentication (lines 159-185)
2. Validates request body (lines 187-214)
3. Creates auth user via `supabaseAdmin.auth.admin.createUser()` (lines 216-239)
4. **Attempts to insert user_profiles record** (lines 245-261)
5. On profile insert failure, deletes the auth user and returns 500 (lines 256-261)

The conflict occurs at step 4 because the trigger already created the profile in step 3.

### Evidence from Logs

**Auth logs** showed a user created then immediately deleted:
- `2025-12-25T00:36:33Z`: `user_signedup` action succeeded
- `2025-12-25T00:36:34Z`: `user_deleted` action (cleanup after profile insert failed)

**Postgres logs** showed the exact error:
```
ERROR: duplicate key value violates unique constraint "user_profiles_pkey"
```

### Frontend Component

Location: `dws-app/src/components/user-form-modal.tsx`

The form correctly sends the request to `/api/admin/users` with the required fields. The frontend is not the source of the issue.

## Code References

- `dws-app/src/app/api/admin/users/route.ts:216-239` - Auth user creation
- `dws-app/src/app/api/admin/users/route.ts:245-261` - Profile insertion that fails
- `dws-app/src/lib/supabaseAdminClient.ts:1-17` - Admin client configuration
- `dws-app/src/components/user-form-modal.tsx:97-136` - Frontend form submission

## Architecture Documentation

### User Creation Flow (Current)

```
1. Admin submits form → POST /api/admin/users
2. API creates auth user → supabaseAdmin.auth.admin.createUser()
3. Trigger fires → handle_new_user() inserts basic profile (user_id, role='employee')
4. API tries to insert full profile → FAILS with duplicate key
5. API deletes auth user (cleanup)
6. API returns 500 error
```

### The Conflict

The trigger was designed for self-signup flow where users create their own accounts and only need a basic profile. The admin creation flow needs to insert a full profile with additional fields (`full_name`, `preferred_name`, `employee_id_internal`, custom `role`).

## Root Cause

The database trigger `on_auth_user_created` and the API endpoint both attempt to insert into `user_profiles` for the same `user_id`, causing a primary key violation.

## Open Questions

1. Should the trigger be modified to handle admin-created users differently?
2. Should the API use UPSERT instead of INSERT for the profile?
3. Is there a way to distinguish between self-signup and admin-created users in the trigger?
