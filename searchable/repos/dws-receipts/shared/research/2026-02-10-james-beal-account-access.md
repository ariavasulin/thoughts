---
date: 2026-02-10T23:45:00-08:00
researcher: ariasulin
git_commit: 80ad11deacf87103a3aea275d883fbd01818fb27
branch: main
repository: dws-receipts
topic: "James Beal Account Access Issue - What Has Been Tried and Verified"
tags: [research, auth, user-account, james-beal, login, otp, supabase]
status: complete
last_updated: 2026-02-10
last_updated_by: ariasulin
---

# Research: James Beal Account Access Issue - What Has Been Tried and Verified

**Date**: 2026-02-10T23:45:00-08:00
**Researcher**: ariasulin
**Git Commit**: 80ad11deacf87103a3aea275d883fbd01818fb27
**Branch**: main
**Repository**: dws-receipts

## Research Question
James Beal is still unable to access his account despite this being a super persistent issue. What have we tried and verified?

## Summary

**There is no documented record of any prior troubleshooting or investigation for James Beal's account access issue.** No thoughts documents, no Linear tickets, and no code comments reference this problem. The only traces of James Beal in the codebase are from his initial account creation during the Nov 2025 user sync. This research documents everything we know about his account state and the authentication system to establish a baseline for debugging.

## Detailed Findings

### 1. James Beal's Account Creation History

James Beal's account was created on **2025-11-04** as part of a bulk user sync operation.

**Source data** (`Assets/Employee_Phone_List_(Design_Workshops).csv:8`):
- Name: Beal, James
- Employee ID: `50-0006382`
- Mobile Phone: `707-712-3107`

**Account created via SQL** (`Assets/user_sync_queries.sql:223-249`):
- Phone stored in E.164 format: `17077123107` (no `+` prefix in the SQL insert)
- Created in `auth.users` table with:
  - `phone_confirmed_at = NOW()` (phone pre-confirmed)
  - `role = 'authenticated'`
  - `aud = 'authenticated'`
- Profile created in `user_profiles` via the `on_auth_user_created` trigger, then updated with:
  - `full_name = 'Beal, James'`
  - `employee_id_internal = '50-0006382'`
  - `preferred_name = 'James'`

**Sync report confirmed** (`Assets/SYNC_COMPLETE_REPORT.md:50`):
- Status: "✅ Created"
- Listed in final verification at line 118

### 2. What Has NOT Been Done (No Prior Record Found)

After exhaustive search across all available sources:

| Source | Searched For | Result |
|--------|-------------|--------|
| `thoughts/` directory (all 56 docs) | "James Beal", "beal", "james", "account access", "login issue" | **Nothing found** |
| Linear (all 15 ARI tickets) | "James Beal", "beal", "account access", "can't login", "login issue" | **Nothing found** |
| Code comments | "beal", "james" | **Nothing found** (only in Assets/) |
| Git history | N/A | No commits reference James Beal troubleshooting |

**Conclusion: This issue has never been formally tracked or investigated in any documented system.**

### 3. The Authentication Flow James Would Use

When James tries to log in, here's what happens step by step:

1. **Phone entry** (`dws-app/src/app/login/page.tsx:87-114`): James enters `707-712-3107` or `7077123107`
2. **Client formatting** (`dws-app/src/lib/phone.ts:6-14`): Strips non-digits → `7077123107` (10 digits) → formatted to `+17077123107`
3. **Server validation** (`dws-app/src/app/api/auth/send-otp/route.ts:15-18`): Validates against E.164 regex `/^\+[1-9]\d{1,14}$/` → should pass
4. **OTP send** (`send-otp/route.ts:20-22`): Calls `supabase.auth.signInWithOtp({ phone: "+17077123107" })` → Supabase sends SMS via Twilio
5. **OTP verify** (`dws-app/src/app/api/auth/verify-otp/route.ts:26-30`): Verifies 4-digit code
6. **Session created** → client redirects to `/employee`
7. **Role check** (`dws-app/src/app/page.tsx:42-46`): Queries `user_profiles` for role → should return `employee`

### 4. Potential Failure Points (What Could Be Going Wrong)

#### A. Phone Number Format Mismatch in Supabase
The SQL insert at `user_sync_queries.sql:234` stores the phone as `'17077123107'` (no `+` prefix). However:
- The `signInWithOtp()` call sends `+17077123107` (with `+` prefix)
- Supabase internally stores phones with `+` prefix in the E.164 format
- **Question**: Did the SQL insert correctly store as `+17077123107` internally, or is it stored as `17077123107` without the `+`?

This is the **most likely suspect** — if Supabase stored the phone without `+`, the OTP lookup would fail silently because `signInWithOtp` sends `+17077123107` but the DB has `17077123107`.

#### B. Supabase OTP Not Sending SMS
The OTP send could succeed from the API's perspective (no error returned) but Twilio may not deliver the SMS because:
- James's carrier blocks short-code SMS
- The phone number `707-712-3107` is a landline (not SMS-capable)
- Twilio rate-limiting or configuration issues

#### C. User Profile Missing or Corrupt
If the `on_auth_user_created` trigger fired but the subsequent `UPDATE user_profiles` at `user_sync_queries.sql:241-246` failed, James could have:
- An auth user that can verify OTP
- A user_profiles row with `full_name = NULL` or `role = NULL`
- This would cause the role check at `page.tsx:42-46` to fail and redirect to `/login`

#### D. Session Not Being Set on Client
After OTP verification, the session setup at `login/page.tsx:142-146` could fail:
- `setSession()` fails silently
- Client redirects to `/employee` but session cookie isn't set
- Page immediately redirects back to `/login`

#### E. OTP Expiration / Timing
Supabase OTPs typically expire in 60 seconds. If James is slow to enter the code, or if SMS delivery is delayed by his carrier, the OTP could expire before verification.

### 5. What the Onboard Script Does Differently

The `onboard-users.mjs` script (`dws-app/scripts/onboard-users.mjs:131-133`) creates users differently than the SQL sync:
- Uses `supabase.auth.admin.createUser()` with `phone_confirm: true`
- This Supabase API method properly formats the phone number
- The SQL insert bypasses Supabase's formatting layer

### 6. Phone Number in Source Data

From `Assets/Employee_Phone_List_(Design_Workshops).csv:8`:
```
Beal, James | 50-0006382 | Mobile: 707-712-3107
```

**Note**: James only has a **mobile phone** — no work phone listed. Other employees with both work and mobile phones were merged to keep work phones. James's account was created fresh with just the mobile number.

## Code References

- `Assets/user_sync_queries.sql:223-249` - James Beal account creation SQL
- `Assets/SYNC_COMPLETE_REPORT.md:50` - Confirmation of account creation
- `Assets/Employee_Phone_List_(Design_Workshops).csv:8` - Source employee data
- `dws-app/src/app/login/page.tsx:87-114` - Login form submission logic
- `dws-app/src/lib/phone.ts:6-14` - Phone number formatting (E.164)
- `dws-app/src/app/api/auth/send-otp/route.ts:15-22` - OTP send with E.164 validation
- `dws-app/src/app/api/auth/verify-otp/route.ts:26-30` - OTP verification
- `dws-app/src/app/page.tsx:42-69` - Role-based routing after auth
- `dws-app/scripts/onboard-users.mjs:131-143` - Alternative user creation method

## Architecture Documentation

### Auth Flow
Phone input → client E.164 formatting → server E.164 validation → Supabase OTP send (Twilio SMS) → user enters 4-digit code → server OTP verify → session tokens → client setSession → redirect to /employee → role-based routing via user_profiles query

### User Creation Methods
1. **SQL direct insert** (`user_sync_queries.sql`) — bypasses Supabase SDK, inserts directly into `auth.users`
2. **Admin SDK** (`onboard-users.mjs`) — uses `supabase.auth.admin.createUser()` with proper phone formatting
3. **Admin Dashboard** (`dws-app/src/app/api/admin/users/route.ts`) — uses admin SDK for new user creation

### Phone Storage
- Database: E.164 format in `auth.users.phone` column
- SQL sync: stores as `'17077123107'` (without `+` in the INSERT value)
- SDK methods: handle `+` prefix automatically

## Historical Context (from thoughts/)

No prior research, plans, tickets, or handoff documents exist about James Beal's account issue or any individual user login problems. The thoughts directory contains documentation about:
- Admin user management features (`thoughts/shared/research/2025-12-18-user-management-admin-dashboard.md`)
- 500 error when adding users (`thoughts/shared/research/2025-12-24-admin-users-500-error.md`)
- Product scope including auth flow (`thoughts/shared/research/2025-12-22-product-scope-document.md`)

None of these address individual user account access troubleshooting.

## Recommended Investigation Steps

To actually diagnose James's issue, the following should be checked **in Supabase directly**:

1. **Verify auth.users record exists**: `SELECT id, phone, phone_confirmed_at, created_at FROM auth.users WHERE phone LIKE '%7077123107%';`
2. **Check phone format**: Is it stored as `+17077123107` or `17077123107`? (The `+` matters for OTP lookup)
3. **Verify user_profiles exists and has role**: `SELECT * FROM user_profiles WHERE user_id = '<james_user_id>';`
4. **Check Supabase auth logs**: Look for OTP send attempts for `+17077123107` — are they succeeding or failing?
5. **Check Twilio delivery logs**: Was the SMS actually delivered to `+17077123107`?
6. **Test OTP flow manually**: Try sending an OTP to James's number via Supabase dashboard or API and check if it arrives
7. **Verify the phone is SMS-capable**: 707 area code is Northern California — confirm it's a mobile number and not a landline/VoIP

## Open Questions

1. **What error does James see?** Does he never receive the OTP? Does he receive it but verification fails? Does he get logged in but then bounced back to login?
2. **Has anyone checked Supabase dashboard** for his auth.users record and confirmed the phone format?
3. **Has anyone checked Twilio logs** to see if SMS is being sent/delivered?
4. **Is 707-712-3107 actually his current phone number?** The CSV data may be outdated.
5. **Was the SQL sync the only method used to create his account?** Could `onboard-users.mjs` have also been run, creating a duplicate or overwriting?
6. **When was the last time he tried to log in?** Recent or months ago?
