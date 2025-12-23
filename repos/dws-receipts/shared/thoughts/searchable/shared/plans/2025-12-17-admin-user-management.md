# Implementation Plan: Admin User Management

**Date**: 2025-12-17
**Feature**: Add user management capabilities to the admin dashboard
**Research**: `thoughts/shared/research/2025-12-17-admin-user-management-feasibility.md`
**Branch**: `admin-user-management` (create from `receipt-editing`)

## Overview

Add the ability for admins to manage users directly from the dashboard: list users, create new users, edit user details (phone, name, role), and ban users (soft delete).

## Design Decisions (Pre-Confirmed)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Deletion strategy | Ban (~100 years) | Preserves receipt history, avoids FK issues, reversible |
| Phone change | Invalidate all sessions | Security: force re-authentication |
| Self-ban | Prevented | Admins cannot ban themselves |
| Self-role-change | Allowed | Admins can change their own role |
| Audit logging | Not needed | Keep implementation simple |

## Success Criteria

- [ ] Admins can view a paginated list of all users with search
- [ ] Admins can create new users with phone, name, role, employee ID
- [ ] Admins can edit existing user details
- [ ] Admins can ban users (soft delete) with confirmation
- [ ] Banned users cannot log in
- [ ] Phone changes force user to re-authenticate
- [ ] All operations require admin role
- [ ] Service role key is never exposed to client

---

## Phase 1: Infrastructure Setup

### 1.1 Environment Variable

Add `SUPABASE_SERVICE_ROLE_KEY` to `.env.local`:

```bash
# Get from Supabase Dashboard > Settings > API > service_role key
SUPABASE_SERVICE_ROLE_KEY=eyJhbG...
```

### 1.2 Install `server-only` Package

```bash
cd dws-app && npm install server-only
```

### 1.3 Create Admin Supabase Client

**File**: `dws-app/src/lib/supabaseAdminClient.ts`

```typescript
import { createClient } from '@supabase/supabase-js'
import 'server-only' // Prevents client-side import

if (!process.env.SUPABASE_SERVICE_ROLE_KEY) {
  throw new Error('Missing SUPABASE_SERVICE_ROLE_KEY environment variable')
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

### 1.4 Database Migration

Add `deleted_at` column to `user_profiles` table via Supabase migration:

```sql
ALTER TABLE public.user_profiles
ADD COLUMN deleted_at TIMESTAMPTZ DEFAULT NULL;

COMMENT ON COLUMN public.user_profiles.deleted_at IS 'Timestamp when user was banned/soft-deleted';
```

### 1.5 Update TypeScript Types

**File**: `dws-app/src/lib/types.ts`

Add to `UserProfile` interface:
```typescript
export interface UserProfile {
  user_id: string;
  role: 'employee' | 'admin';
  full_name?: string;
  preferred_name?: string;
  employee_id_internal?: string;
  created_at?: string;
  updated_at?: string;
  deleted_at?: string | null;  // NEW
}
```

Add new interface for admin user list:
```typescript
export interface AdminUser {
  id: string;
  phone: string;
  created_at: string;
  last_sign_in_at?: string;
  banned_until?: string;
  // From user_profiles join
  role: 'employee' | 'admin';
  full_name?: string;
  preferred_name?: string;
  employee_id_internal?: string;
  deleted_at?: string | null;
}
```

---

## Phase 2: API Routes

### 2.1 GET /api/admin/users - List Users

**File**: `dws-app/src/app/api/admin/users/route.ts`

**Features**:
- Pagination via `page` and `perPage` query params
- Client-side search filtering (Supabase Admin API has no server-side search)
- Exclude deleted users by default (optional `includeDeleted` param)
- Join with `user_profiles` for application data

**Request**: `GET /api/admin/users?page=1&perPage=50&search=john`

**Response**:
```json
{
  "users": [...],
  "page": 1,
  "perPage": 50,
  "total": 123
}
```

**Implementation pattern**:
1. Auth check (session + admin role) - copy from `api/admin/receipts/route.ts:16-46`
2. Parse query params
3. Call `supabaseAdmin.auth.admin.listUsers({ page, perPage })`
4. Fetch `user_profiles` for all returned user IDs
5. Merge auth data with profile data
6. Apply search filter if provided
7. Filter out deleted users unless `includeDeleted=true`

### 2.2 POST /api/admin/users - Create User

**File**: `dws-app/src/app/api/admin/users/route.ts` (same file, POST handler)

**Request body**:
```json
{
  "phone": "5551234567",
  "full_name": "Smith, John",
  "preferred_name": "Johnny",
  "employee_id_internal": "EMP001",
  "role": "employee"
}
```

**Implementation**:
1. Auth check (session + admin role)
2. Validate phone format using existing `formatUSPhoneNumber` pattern
3. Create auth user: `supabaseAdmin.auth.admin.createUser({ phone, phone_confirm: true, user_metadata })`
4. Handle `phone_exists` error (return 409 Conflict)
5. Create `user_profiles` record with role
6. Return created user

**Error handling**:
| Error | Status | Message |
|-------|--------|---------|
| `phone_exists` | 409 | Phone number already registered |
| Invalid phone | 400 | Invalid phone number format |
| Missing required fields | 400 | Phone and full_name are required |

### 2.3 GET/PATCH/DELETE /api/admin/users/[id]

**File**: `dws-app/src/app/api/admin/users/[id]/route.ts`

#### GET - Single User
- Return user with profile data
- Include ban status

#### PATCH - Update User
**Request body** (all optional):
```json
{
  "phone": "5559876543",
  "full_name": "Smith, Jonathan",
  "preferred_name": "Jon",
  "employee_id_internal": "EMP002",
  "role": "admin"
}
```

**Implementation**:
1. Auth check
2. Fetch current user data
3. If phone changed:
   - Validate new phone format
   - Update via `supabaseAdmin.auth.admin.updateUserById(id, { phone, phone_confirm: true })`
   - Sign out all sessions: `supabaseAdmin.auth.admin.signOut(id, 'global')`
4. Update `user_profiles` record
5. Return updated user

#### DELETE - Ban User
**Implementation**:
1. Auth check
2. **Self-ban check**: If `userId === currentUserId`, return 400 "Cannot ban yourself"
3. Ban user: `supabaseAdmin.auth.admin.updateUserById(id, { ban_duration: '876000h' })`
4. Mark in profiles: `update({ deleted_at: new Date().toISOString() })`
5. Sign out all sessions: `supabaseAdmin.auth.admin.signOut(id, 'global')`
6. Return success

---

## Phase 3: UI Components

### 3.1 User Table Component

**File**: `dws-app/src/components/user-table.tsx`

**Pattern**: Follow `receipt-table.tsx` structure

**Columns**:
| Column | Source | Sortable |
|--------|--------|----------|
| Name | `preferred_name` or `full_name` | Yes |
| Phone | `phone` (formatted) | No |
| Role | `role` (badge) | Yes |
| Employee ID | `employee_id_internal` | No |
| Created | `created_at` | Yes |
| Actions | Edit, Ban buttons | No |

**Features**:
- Search input (filters by name, phone, employee ID)
- Pagination controls
- Role badge (admin = gold, employee = gray)
- Sortable columns

### 3.2 User Form Modal

**File**: `dws-app/src/components/user-form-modal.tsx`

**Pattern**: Follow `receipt-details-card.tsx` dialog pattern

**Fields**:
| Field | Type | Validation | Required |
|-------|------|------------|----------|
| Phone | Input | 10-digit US number | Yes |
| Full Name | Input | Non-empty | Yes |
| Preferred Name | Input | None | No |
| Employee ID | Input | None | No |
| Role | Select | employee/admin | Yes |

**Modes**:
- Create: All fields empty, submit calls POST
- Edit: Pre-filled from user data, submit calls PATCH

### 3.3 Ban Confirmation Dialog

**File**: `dws-app/src/components/ban-user-dialog.tsx`

**Content**:
```
Are you sure you want to ban [User Name]?

This will:
- Prevent the user from logging in
- Sign out all active sessions
- Preserve their receipt history

[Cancel] [Ban User]
```

### 3.4 User Management Modal (Container)

**File**: `dws-app/src/components/user-management-modal.tsx`

**Structure**:
- Full-screen modal (similar to batch-review)
- Header with "Manage Users" title and close button
- "Add User" button
- UserTable component
- UserFormModal (triggered by Add/Edit)
- BanUserDialog (triggered by Ban)

### 3.5 Dashboard Integration

**File**: `dws-app/src/components/receipt-dashboard.tsx`

Add "Manage Users" button to header navigation (around line 437):

```tsx
<Button
  variant="ghost"
  size="sm"
  onClick={() => setShowUserManagement(true)}
  className="bg-[#333333] text-white hover:bg-[#444444]"
>
  <Users className="mr-2 h-4 w-4" />
  Manage Users
</Button>
```

Add state and modal:
```tsx
const [showUserManagement, setShowUserManagement] = useState(false);

// In render, after other modals:
<UserManagementModal
  open={showUserManagement}
  onOpenChange={setShowUserManagement}
/>
```

---

## Phase 4: Phone Number Utility

### 4.1 Extract Phone Formatting to Shared Utility

**File**: `dws-app/src/lib/phone.ts`

Move from `login/page.tsx:15-39` to shared location:

```typescript
/**
 * Formats a US phone number to E.164 format (+1XXXXXXXXXX)
 * @param input - Phone number in various formats
 * @returns E.164 formatted number or null if invalid
 */
export function formatUSPhoneNumber(input: string): string | null {
  const digits = input.replace(/\D/g, '');
  if (digits.length === 10) {
    return `+1${digits}`;
  } else if (digits.length === 11 && digits.startsWith('1')) {
    return `+${digits}`;
  }
  return null;
}

/**
 * Formats E.164 phone number for display: (555) 123-4567
 */
export function formatPhoneForDisplay(e164: string): string {
  const digits = e164.replace(/\D/g, '');
  const national = digits.startsWith('1') ? digits.slice(1) : digits;
  if (national.length !== 10) return e164;
  return `(${national.slice(0,3)}) ${national.slice(3,6)}-${national.slice(6)}`;
}
```

Update `login/page.tsx` to import from this shared file.

---

## Implementation Order

1. **Phase 1**: Infrastructure (30 min)
   - Add env var
   - Install server-only
   - Create admin client
   - Run migration
   - Update types

2. **Phase 2**: API Routes (1-2 hours)
   - GET /api/admin/users (list)
   - POST /api/admin/users (create)
   - GET /api/admin/users/[id] (single)
   - PATCH /api/admin/users/[id] (update)
   - DELETE /api/admin/users/[id] (ban)

3. **Phase 3**: UI (2-3 hours)
   - Phone utility extraction
   - User table component
   - User form modal
   - Ban confirmation dialog
   - User management modal
   - Dashboard integration

4. **Testing** (30 min)
   - Manual testing of all CRUD operations
   - Verify ban prevents login
   - Verify phone change forces re-auth
   - Verify self-ban prevention

---

## File Summary

| File | Action | Description |
|------|--------|-------------|
| `.env.local` | Edit | Add SUPABASE_SERVICE_ROLE_KEY |
| `package.json` | Edit | Add server-only dependency |
| `src/lib/supabaseAdminClient.ts` | Create | Service role client |
| `src/lib/types.ts` | Edit | Add deleted_at to UserProfile, add AdminUser |
| `src/lib/phone.ts` | Create | Phone formatting utilities |
| `src/app/api/admin/users/route.ts` | Create | GET (list) and POST (create) |
| `src/app/api/admin/users/[id]/route.ts` | Create | GET, PATCH, DELETE |
| `src/components/user-table.tsx` | Create | User list table |
| `src/components/user-form-modal.tsx` | Create | Create/edit user form |
| `src/components/ban-user-dialog.tsx` | Create | Ban confirmation |
| `src/components/user-management-modal.tsx` | Create | Container modal |
| `src/components/receipt-dashboard.tsx` | Edit | Add Manage Users button |
| `src/app/login/page.tsx` | Edit | Import phone util from shared |

---

## Error Codes Reference

| Error Code | HTTP | User Message |
|------------|------|--------------|
| `phone_exists` | 409 | Phone number already registered |
| `user_not_found` | 404 | User not found |
| `over_request_rate_limit` | 429 | Too many requests, try again later |
| Self-ban attempt | 400 | Cannot ban yourself |
| Invalid phone | 400 | Invalid phone number format |
| Missing auth | 401 | Unauthorized |
| Not admin | 403 | Admin access required |

---

## Open Questions (Low Priority, Defer)

1. **Bulk Import UI**: Keep CSV import as script-only for now
2. **Search Performance**: If user count grows significantly, consider database RPC for server-side search
