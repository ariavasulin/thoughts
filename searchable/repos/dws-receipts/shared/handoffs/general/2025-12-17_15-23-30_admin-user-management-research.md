---
date: 2025-12-17T23:23:30Z
researcher: Claude
git_commit: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
branch: receipt-editing
repository: DWS-Receipts
topic: "Admin User Management Features Implementation Strategy"
tags: [research, implementation, admin-dashboard, user-management, supabase-auth]
status: complete
last_updated: 2025-12-17
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: Admin User Management Features

## Task(s)
**Research (COMPLETED)**: Feasibility analysis for adding account management features to the admin dashboard - adding users, deleting users, changing credentials.

Research is complete. The next agent should create an implementation plan based on the research document.

## Critical References
- `thoughts/shared/research/2025-12-17-admin-user-management-feasibility.md` - Complete research document with API examples, design decisions, and implementation guidance
- `dws-app/scripts/onboard-users.mjs` - Existing script demonstrating user creation patterns via Supabase Admin API

## Recent changes
No code changes made - this was a research-only session.

## Learnings

### Supabase Admin API
- All admin operations require `service_role` key (not anon key)
- Must use `server-only` import to prevent client-side usage
- `listUsers()` has NO built-in search - must filter client-side or use database RPC
- `createUser()` does NOT send emails - use `inviteUserByEmail()` if needed
- `user_metadata` MERGES on update, doesn't replace

### Existing Codebase Patterns
- Admin role verification pattern already exists in `dws-app/src/app/api/admin/receipts/route.ts:37-46`
- Phone formatting function exists in `dws-app/src/app/login/page.tsx:15-39`
- User profile table uses `user_profiles` with FK to `auth.users`
- Onboarding script at `dws-app/scripts/onboard-users.mjs` shows exact patterns needed

### Design Decisions (Confirmed with User)
1. **Deletion = Ban** - Use `updateUserById(userId, { ban_duration: '876000h' })` instead of hard delete
   - Preserves receipt history
   - Avoids storage objects FK constraint issues
   - Reversible if needed
2. **Phone change** - Invalidate all sessions via `auth.admin.signOut(userId, 'global')`
3. **Self-ban prevention** - Admins CANNOT ban themselves (check `userId !== currentUserId`)
4. **Audit logging** - Not needed
5. **Admins CAN change their own role**

## Artifacts
- `thoughts/shared/research/2025-12-17-admin-user-management-feasibility.md` - Complete research document

## Action Items & Next Steps
The next agent should create an implementation plan covering:

1. **Environment Setup**
   - Add `SUPABASE_SERVICE_ROLE_KEY` to `.env.local`
   - Install `server-only` package if not present

2. **Database Migration**
   - Add `deleted_at` column to `user_profiles` table

3. **Create Admin Client**
   - New file: `dws-app/src/lib/supabaseAdminClient.ts`
   - Must import `server-only` to prevent client-side usage

4. **Implement API Routes**
   - `GET /api/admin/users` - List with pagination (exclude deleted)
   - `POST /api/admin/users` - Create with phone validation
   - `PATCH /api/admin/users/[id]` - Update user + sign out on phone change
   - `DELETE /api/admin/users/[id]` - Ban user + mark deleted_at + prevent self-ban

5. **Build UI Components**
   - User table component (similar to `receipt-table.tsx`)
   - User form modal (create/edit)
   - Ban confirmation dialog
   - Add "Manage Users" button to dashboard navigation (`receipt-dashboard.tsx:413-459`)

## Other Notes

### Key Files to Reference
- `dws-app/src/components/receipt-table.tsx` - Pattern for sortable table with pagination
- `dws-app/src/components/receipt-details-card.tsx` - Pattern for form modal with edit/delete
- `dws-app/src/app/api/admin/receipts/route.ts` - Pattern for admin API auth checking
- `dws-app/src/lib/types.ts:30-38` - UserProfile interface (may need `deleted_at` field added)

### Common Error Codes to Handle
| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `phone_exists` | 422 | Phone already registered |
| `user_not_found` | 404 | User doesn't exist |
| `over_request_rate_limit` | 429 | Too many requests |

### Open Questions (Low Priority)
1. Bulk Import UI - expose CSV import in UI or keep as script?
2. Search Performance - may need database RPC for server-side search if user count grows
