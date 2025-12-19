---
date: 2025-12-18T00:00:00-08:00
researcher: Claude
git_commit: 13eaf60dda5f35a03116a36ae4aa9f1313d9f423
branch: receipt-editing
repository: DWS-Receipts
topic: "User Management in Admin Dashboard - Adding New Users"
tags: [research, codebase, user-management, admin-dashboard, supabase-auth]
status: complete
last_updated: 2025-12-18
last_updated_by: Claude
---

# Research: User Management in Admin Dashboard - Adding New Users

**Date**: 2025-12-18
**Researcher**: Claude
**Git Commit**: 13eaf60dda5f35a03116a36ae4aa9f1313d9f423
**Branch**: receipt-editing
**Repository**: DWS-Receipts

## Research Question

We need to add the ability to add new users via the account management in the admin dashboard.

## Summary

**The ability to add new users already exists in the codebase.** A complete user management system has been implemented with full CRUD operations (Create, Read, Update, Delete/Ban). The system is accessible at `/users` from the admin dashboard and includes:

- A dedicated page for user management (`/users`)
- API endpoints for user operations (`/api/admin/users`)
- UI components for listing, creating, editing, and banning users
- Integration with Supabase Auth Admin API for user creation

The "Add User" functionality is implemented via the `UserFormModal` component which creates users through the `POST /api/admin/users` endpoint.

## Detailed Findings

### 1. User Management Page

**File**: `dws-app/src/app/users/page.tsx`

The user management page is a dedicated admin-only route that:
- Verifies the user is authenticated and has admin role (lines 16-97)
- Redirects employees to `/employee` and unauthenticated users to `/login`
- Renders the `UserManagementDashboard` component with logout callback

**Access from Dashboard**: The main receipt dashboard (`/dashboard`) has a "Manage Users" button in the header (lines 428-437 in `receipt-dashboard.tsx`) that links to `/users`.

### 2. User Management Dashboard Component

**File**: `dws-app/src/components/user-management-dashboard.tsx`

Main features:
- **User List**: Fetches all users via `GET /api/admin/users?perPage=1000` (line 42)
- **Search**: Client-side filtering by full_name, preferred_name, phone, or employee_id_internal (lines 62-83)
- **Add User**: "Add User" button opens `UserFormModal` in "create" mode (lines 85-89, 185-192)
- **Edit User**: Edit action opens `UserFormModal` in "edit" mode (lines 91-95)
- **Ban User**: Ban action opens `BanUserDialog` for confirmation (lines 97-100)

### 3. Add User Flow

**Component**: `dws-app/src/components/user-form-modal.tsx`

The `UserFormModal` component handles both creating and editing users:

**Form Fields** (lines 158-244):
- Phone number (required) - accepts any US format
- Full name (required) - placeholder "Last, First"
- Preferred name (optional)
- Employee ID (optional)
- Role (dropdown: "Employee" or "Admin")

**Validation** (lines 84-93):
- Phone and full_name are required
- Phone validated using `isValidUSPhoneNumber()` from `dws-app/src/lib/phone.ts`

**Submission** (lines 79-137):
- In "create" mode, sends `POST /api/admin/users`
- Request body includes: phone, full_name, and optionally preferred_name, employee_id_internal, role
- On success, calls `onSuccess` callback which updates the dashboard's local state

### 4. API Endpoint for Creating Users

**File**: `dws-app/src/app/api/admin/users/route.ts` (POST handler, lines 158-284)

**Process**:
1. Authenticates caller and verifies admin role (lines 159-185)
2. Validates required fields: phone and full_name (lines 192-197)
3. Formats phone to E.164 format (+1XXXXXXXXXX) (lines 200-206)
4. Validates role is 'employee' or 'admin' (lines 209-214)
5. Creates auth user via `supabaseAdmin.auth.admin.createUser()` (lines 217-225):
   - Sets `phone_confirm: true` to skip phone verification
   - Stores metadata in `user_metadata`
6. Creates `user_profiles` record with role information (lines 246-254)
7. If profile creation fails, deletes the auth user for cleanup (line 259)
8. Returns the created `AdminUser` object (lines 264-276)

**Error Handling**:
- 400: Missing required fields or invalid phone/role format
- 409: Phone number already registered (duplicate)
- 500: Auth or profile creation failure

### 5. User Table Component

**File**: `dws-app/src/components/user-table.tsx`

Displays users in a sortable table with:
- Columns: Name, Phone, Role, Employee ID, Created date, Actions
- Sorting on Name, Role, and Created columns
- Actions dropdown with Edit and Ban options
- Role displayed as color-coded badges

### 6. Ban User Dialog

**File**: `dws-app/src/components/ban-user-dialog.tsx`

Confirmation dialog that:
- Shows consequences of banning (prevents login, signs out sessions, preserves receipt history)
- Calls `DELETE /api/admin/users/${user.id}` to perform soft delete
- Bans user for ~100 years (876000 hours) rather than hard deleting

### 7. Database Schema

**Tables involved**:

1. `auth.users` (Supabase-managed):
   - `id`: UUID primary key
   - `phone`: E.164 format (+1XXXXXXXXXX)
   - `created_at`, `last_sign_in_at`: timestamps
   - `banned_until`: ban expiration

2. `user_profiles` (custom table):
   - `user_id`: FK to auth.users, also PK
   - `role`: 'employee' | 'admin'
   - `full_name`, `preferred_name`, `employee_id_internal`: optional strings
   - `created_at`, `updated_at`, `deleted_at`: timestamps

### 8. TypeScript Types

**File**: `dws-app/src/lib/types.ts`

```typescript
// Lines 42-54
interface AdminUser {
  id: string;
  phone: string;
  created_at: string;
  last_sign_in_at?: string;
  banned_until?: string;
  role: 'employee' | 'admin';
  full_name?: string;
  preferred_name?: string;
  employee_id_internal?: string;
  deleted_at?: string | null;
}
```

### 9. Phone Number Utilities

**File**: `dws-app/src/lib/phone.ts`

- `formatUSPhoneNumber()`: Converts various formats to E.164 (+1XXXXXXXXXX)
- `formatPhoneForDisplay()`: Converts E.164 to (XXX) XXX-XXXX for display
- `isValidUSPhoneNumber()`: Validates US phone numbers

### 10. Supabase Admin Client

**File**: `dws-app/src/lib/supabaseAdminClient.ts`

Server-only client using service role key that:
- Bypasses Row Level Security (RLS)
- Accesses Supabase Auth Admin API for user creation/management
- Protected by `'server-only'` import to prevent client-side bundling

## Code References

- `dws-app/src/app/users/page.tsx` - User management page (admin-only route)
- `dws-app/src/components/user-management-dashboard.tsx` - Main dashboard component
- `dws-app/src/components/user-form-modal.tsx` - Create/edit user form
- `dws-app/src/components/user-table.tsx` - User list table with sorting
- `dws-app/src/components/ban-user-dialog.tsx` - Ban confirmation dialog
- `dws-app/src/app/api/admin/users/route.ts` - GET (list) and POST (create) endpoints
- `dws-app/src/app/api/admin/users/[id]/route.ts` - GET, PATCH, DELETE endpoints
- `dws-app/src/lib/types.ts:42-54` - AdminUser type definition
- `dws-app/src/lib/phone.ts` - Phone number formatting utilities
- `dws-app/src/lib/supabaseAdminClient.ts` - Admin Supabase client

## Architecture Documentation

### Navigation Flow
```
/dashboard (Receipt Dashboard)
    └── "Manage Users" button → /users (User Management)

/users (User Management)
    └── "Back to Dashboard" link → /dashboard
```

### Component Hierarchy
```
/users (page.tsx)
    └── UserManagementDashboard
            ├── UserTable
            │       └── Actions dropdown (Edit, Ban)
            ├── UserFormModal (create/edit modes)
            └── BanUserDialog
```

### API Pattern
All admin user routes follow this pattern:
1. Create server-side Supabase client
2. Verify session exists (401 if not)
3. Query user_profiles for caller's role (403 if not admin)
4. Execute operation using admin client
5. Return JSON response

### Data Flow for Adding Users
1. Admin clicks "Add User" button
2. UserFormModal opens in "create" mode
3. Admin fills form (phone required, full_name required)
4. On submit, modal validates fields
5. POST request to /api/admin/users
6. API creates auth.users record via Supabase Admin API
7. API creates user_profiles record
8. Response contains new AdminUser object
9. Dashboard adds user to local state
10. Toast notification shows success

## Open Questions

1. **Untracked files**: The user management components appear as untracked in git status. Are these files meant to be committed?

2. **Profile creation trigger**: The codebase references auto-creation of user_profiles via database trigger, but the trigger definition was not found in the codebase. Where is this defined?

3. **Email field**: The AdminUser type and forms don't include email, but some Supabase operations reference email. Is email authentication planned?
