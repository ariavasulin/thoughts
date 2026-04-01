---
date: 2026-04-01T08:36:43-07:00
researcher: ariasulin
git_commit: 80ad11deacf87103a3aea275d883fbd01818fb27
branch: main
repository: dws-receipts
topic: "James Beal Account Access - Follow-up Investigation and Fix Options"
tags: [research, auth, user-account, james-beal, login, otp, supabase, phone-format]
status: complete
last_updated: 2026-04-01
last_updated_by: ariasulin
---

# Research: James Beal Account Access - Follow-up Investigation and Fix Options

**Date**: 2026-04-01T08:36:43-07:00
**Researcher**: ariasulin
**Git Commit**: 80ad11deacf87103a3aea275d883fbd01818fb27
**Branch**: main
**Repository**: dws-receipts

## Research Question
James Beal's account still isn't working. What research and plans indicate what we have tried, and what are our options to fix this?

## Summary

There is **one prior research document** (`thoughts/shared/research/2026-02-10-james-beal-account-access.md`) from ~7 weeks ago that established a baseline investigation. **No Linear tickets, no implementation plans, and no code fixes have ever been created for this issue.** The prior research identified the most likely root cause — a phone number format mismatch from the bulk SQL import — but the recommended Supabase-side verification steps were never executed. The app now has a full admin user management UI that can fix the phone number or recreate the account entirely.

## What Has Been Tried

### Prior Research (2026-02-10)
The only documented investigation is `thoughts/shared/research/2026-02-10-james-beal-account-access.md`. It found:

1. **Account creation**: James's account was created 2025-11-04 via direct SQL insert into `auth.users` (`Assets/user_sync_queries.sql:223-249`)
2. **Phone stored without `+` prefix**: The SQL insert used `'17077123107'` — no `+` prefix — while the login flow sends `+17077123107` via `signInWithOtp()`
3. **No prior troubleshooting existed**: No tickets, no thoughts docs, no code comments, no git history
4. **Five potential failure points identified**:
   - **A. Phone format mismatch** (rated most likely) — `17077123107` vs `+17077123107`
   - **B. SMS not delivering** — carrier blocking, landline, or Twilio issues
   - **C. User profile missing/corrupt** — trigger may have partially failed
   - **D. Session not being set on client** — `setSession()` silent failure
   - **E. OTP expiration** — 60-second window too short

### What Was NOT Done
The prior research recommended 7 specific Supabase verification steps. **None appear to have been executed:**
1. ❌ Verify `auth.users` record exists and check phone format
2. ❌ Check if phone is stored as `+17077123107` or `17077123107`
3. ❌ Verify `user_profiles` exists with correct role
4. ❌ Check Supabase auth logs for OTP send attempts
5. ❌ Check Twilio delivery logs
6. ❌ Test OTP flow manually
7. ❌ Verify phone is SMS-capable (not landline)

### Linear Tickets
**Zero tickets exist** for this issue. Searched all ARI tickets — no results for "James Beal", "beal", "login issue", or "account access".

## The Root Cause (Most Likely)

The SQL bulk import inserted the phone as `'17077123107'` (without `+`):

```sql
-- Assets/user_sync_queries.sql:234
'17077123107',  -- Mobile phone
```

But the login flow formats to `+17077123107`:
- `lib/phone.ts:9` → `+1${digits}` → `+17077123107`
- `send-otp/route.ts:20` → `signInWithOtp({ phone: "+17077123107" })`

If Supabase stored it literally as `17077123107` without normalizing to `+17077123107`, the OTP lookup would fail silently — Supabase would treat it as a new user signup attempt rather than finding the existing account. The `signInWithOtp` call would return success (no error), an SMS might be sent, but the OTP would be associated with a different (possibly auto-created) auth record for `+17077123107`, while the original profile remains attached to the `17077123107` record.

**Contrast with the SDK method**: The `onboard-users.mjs` script (line 131) uses `supabase.auth.admin.createUser({ phone: phonePlus, phone_confirm: true })` which properly handles E.164 formatting through the Supabase SDK.

## Options to Fix

### Option 1: Fix Phone via Admin Dashboard UI (Recommended — Simplest)
The app has a full admin user management page at `/users` (implemented via ARI-15).

**Steps:**
1. Log in as admin, navigate to `/users`
2. Find "Beal, James" in the user table
3. Click Edit from the dropdown menu
4. The `UserFormModal` will show the current phone number
5. Re-enter the correct phone: `707-712-3107` (or `7077123107`)
6. Submit — the `PATCH /api/admin/users/[id]` handler will:
   - Format to `+17077123107` via `formatUSPhoneNumber()`
   - Call `supabaseAdmin.auth.admin.updateUserById()` with `phone_confirm: true`
   - Sign out all sessions globally
   - Return the updated user

**Key code**: `dws-app/src/app/api/admin/users/[id]/route.ts:95-226` (PATCH handler)

**Caveat**: This only works if the existing `auth.users` record is findable. If the phone is stored as `17077123107`, the admin user list (which queries via `get_auth_users_for_admin` RPC) should still show James since it joins on `user_id`, not phone. Updating the phone through the admin SDK will properly set it to `+17077123107`.

### Option 2: Verify and Fix Directly in Supabase SQL
Run the diagnostic queries from the prior research, then fix:

```sql
-- 1. Find James's auth record (try both formats)
SELECT id, phone, phone_confirmed_at, created_at, last_sign_in_at
FROM auth.users
WHERE phone LIKE '%7077123107%';

-- 2. If phone is '17077123107' (no +), fix it:
UPDATE auth.users
SET phone = '+17077123107', updated_at = NOW()
WHERE phone = '17077123107';

-- 3. Verify user_profiles is intact
SELECT * FROM user_profiles
WHERE user_id = '<uuid from step 1>';
```

### Option 3: Delete and Recreate via Admin UI
If the account is in a bad state:
1. Ban the existing account via the admin dashboard (soft delete)
2. Create a new account via "Add User" with phone `707-712-3107`, name `Beal, James`, employee ID `50-0006382`
3. The `POST /api/admin/users` handler uses `supabase.auth.admin.createUser()` which properly handles phone formatting

**Downside**: James would get a new `user_id`. Since he has never logged in successfully, he likely has zero receipts, so this is low-risk.

### Option 4: Run the Onboard Script
The `onboard-users.mjs` script could re-process James from the CSV:

```bash
cd dws-app
node scripts/onboard-users.mjs --csv ../Assets/joined_employees.csv
# Review dry-run output, then:
node scripts/onboard-users.mjs --csv ../Assets/joined_employees.csv --execute
```

This uses `supabase.auth.admin.createUser()` with proper phone formatting. However, it may skip James if it finds an existing user with matching phone digits.

## Recommended Action Plan

1. **First**: Check Supabase dashboard to confirm the phone format issue (Option 2, step 1 only)
2. **Then**: Use the Admin UI (Option 1) to update the phone number — this is the safest and most auditable path
3. **If that fails**: Delete and recreate (Option 3)
4. **After fix**: Have James attempt login to confirm it works
5. **File a Linear ticket** to track this resolution

## Code References

- `Assets/user_sync_queries.sql:223-249` — Original SQL that created James's account with `'17077123107'`
- `Assets/SYNC_COMPLETE_REPORT.md:50` — Confirmation of account creation
- `dws-app/src/lib/phone.ts:6-14` — `formatUSPhoneNumber()` adds `+1` prefix
- `dws-app/src/app/api/auth/send-otp/route.ts:15-22` — E.164 validation and OTP send
- `dws-app/src/app/api/admin/users/[id]/route.ts:95-226` — PATCH handler for updating phone
- `dws-app/src/app/api/admin/users/route.ts:136-260` — POST handler for creating users
- `dws-app/src/components/user-form-modal.tsx` — Admin UI for editing users
- `dws-app/scripts/onboard-users.mjs:131-143` — SDK-based user creation (proper phone formatting)

## Historical Context (from thoughts/)

- `thoughts/shared/research/2026-02-10-james-beal-account-access.md` — **Only prior investigation**. Established baseline, identified phone format as primary suspect, listed 7 verification steps that were never executed.
- `thoughts/shared/plans/2025-12-17-admin-user-management.md` — Plan for ARI-15 (admin user management), now implemented
- `thoughts/shared/research/2025-12-17-admin-user-management-feasibility.md` — Feasibility research for phone editing via admin API
- `thoughts/shared/research/2025-12-18-user-management-admin-dashboard.md` — Implementation details for user management

## Open Questions

1. **What exactly does James see?** Does the OTP send succeed (he sees "code sent")? Does the SMS arrive? Does verification fail? This narrows the diagnosis significantly.
2. **Has the phone format already been fixed?** Someone may have corrected it in Supabase without documenting it. Check the current `auth.users` state.
3. **Is there a duplicate auth record?** If `signInWithOtp("+17077123107")` created a new auth user (separate from the `17077123107` one), there may be two records — one with the profile, one without.
4. **Is 707-712-3107 still his current number?** The CSV data is from late 2025.
