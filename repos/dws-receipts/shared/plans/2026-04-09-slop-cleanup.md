# Slop Cleanup Implementation Plan

## Overview

Systematic removal of AI-generated artifacts, dead code, debug logging, unnecessary comments, type slop, over-defensive code, and duplicated boilerplate across the DWS-Receipts codebase. Each phase is designed to be executed by parallel agents with non-overlapping file scopes.

## What We're NOT Doing

- Adding new features, tests, or documentation
- Refactoring architecture (e.g., extracting receipt-uploader into smaller components)
- Migrating the employee page to React Query (would be nice, but out of scope)
- Changing any user-facing behavior

## Desired End State

- Zero AI tool artifacts in git history or working tree
- Zero `console.log`/`console.warn` statements in client code; only meaningful `console.error` in API catch blocks
- Zero "restating the code" comments
- Clean, tight TypeScript types with no dual-case unions or duplicate fields
- No dead code, dead files, or unused imports
- DRY admin auth and shared utilities
- `npm run build` passes with zero warnings

---

## Phase 1: Security Purge

### Overview
Remove employee PII and exposed infrastructure IDs. This phase touches only root-level directories and docs — no `dws-app/src/` changes.

### Parallel Tasks

#### Task 1A: Gitignore + History Rewrite
**Files**: `.gitignore`, git history

DONE: `.claude/` and `thoughts/` already purged from history and gitignored. Remaining:

1. Add to `.gitignore`:
   ```
   Assets/
   Prototypes/
   Docs/Archive/
   ```
2. Run `git filter-repo --invert-paths --path Assets/ --path Prototypes/ --path Docs/Archive/ --force`
3. Re-add remote: `git remote add origin https://github.com/ariavasulin/DWS-Receipts-2.git`

#### Task 1B: Scrub Supabase Project ID from Docs
**Files**: `Docs/Archive/sprint1.md`, `Docs/Archive/progress.md`

If these files survive the history rewrite (they should be gone), ensure the Supabase project ID `qebbmojnqzwwdpkhuyyd` is not present in any remaining tracked file except `dws-app/next.config.ts` (which needs it for image remote patterns).

### Success Criteria

#### Automated Verification:
- [ ] `git log --all -p | grep -c 'Employee_Phone_List\|user_sync_queries\|SYNC_COMPLETE_REPORT'` returns 0
- [ ] `git ls-files | grep -E '^(Assets|Prototypes|Docs/Archive|\.claude|thoughts)/'` returns nothing
- [ ] `grep -r 'qebbmojnqzwwdpkhuyyd' --include='*.md'` returns nothing

#### Manual Verification:
- [ ] `git push --force origin main` succeeds

---

## Phase 2: Dead File Removal

### Overview
Delete files that serve no purpose. Pure deletion — no behavioral changes.

### Parallel Tasks

#### Task 2A: Delete Dead App Files
**Files**: Only these specific files
```
rm dws-app/src/app/dashboard/page.tsx.old
rm dws-app/src/lib/query-client.ts
rm dws-app/src/app/dashboard/layout.tsx
```

`page.tsx.old` is a backup artifact. `query-client.ts` is dead (duplicated in `query-provider.tsx`, never imported). `layout.tsx` is a passthrough `<>{children}</>` that does nothing.

#### Task 2B: Delete Unused shadcn/ui Components
**Files**: Only `dws-app/src/components/ui/` files

Delete these 31 files (none are imported by any app code):
```
accordion.tsx    alert.tsx        aspect-ratio.tsx  avatar.tsx
breadcrumb.tsx   carousel.tsx     chart.tsx         collapsible.tsx
command.tsx      context-menu.tsx form.tsx          hover-card.tsx
input-otp.tsx    menubar.tsx      navigation-menu.tsx
radio-group.tsx  resizable.tsx    scroll-area.tsx   separator.tsx
sheet.tsx        sidebar.tsx      skeleton.tsx      slider.tsx
sonner.tsx       switch.tsx       textarea.tsx      toast.tsx
toaster.tsx      toggle.tsx       toggle-group.tsx  tooltip.tsx
use-mobile.tsx   use-toast.ts
```

Keep: `badge.tsx`, `button.tsx`, `card.tsx`, `checkbox.tsx`, `dialog.tsx`, `drawer.tsx`, `dropdown-menu.tsx`, `input.tsx`, `label.tsx`, `pagination.tsx`, `progress.tsx`, `select.tsx`, `table.tsx`, `tabs.tsx`, `alert-dialog.tsx`, `popover.tsx`, `calendar.tsx`

#### Task 2C: Delete Default Next.js Assets + Boilerplate README
**Files**: Only `dws-app/public/` and `dws-app/README.md`
```
rm dws-app/public/next.svg
rm dws-app/public/vercel.svg
rm dws-app/public/globe.svg
rm dws-app/public/window.svg
rm dws-app/public/file.svg
rm dws-app/README.md
```

### Success Criteria

#### Automated Verification:
- [ ] `npm run build` in `dws-app/` succeeds
- [ ] `ls dws-app/src/components/ui/ | wc -l` drops from ~48 to ~17
- [ ] `cat dws-app/src/lib/query-client.ts` → file not found

---

## Phase 3: Dead Code Removal

### Overview
Remove unused exports, dead functions, unused imports, and commented-out code blocks. Each task scopes to specific files to avoid conflicts.

### Parallel Tasks

#### Task 3A: Clean `receipt-uploader.tsx`
**File**: `dws-app/src/components/receipt-uploader.tsx`

1. Delete `validateFile` function (lines 148-179) — defined but never called
2. Delete `testPlatformDetection` function (lines 50-63) and the conditional `useEffect` that calls it (lines 66-71) — this also fixes the Rules of Hooks violation
3. In `detectUserBehavior` (lines 110-145): remove the return statement and the unused `behaviorContext` object construction. The callsite at line 191 discards the return value. Either delete the entire function or strip it to only the `console.log` (which Phase 4 will delete anyway — so just delete the whole function and the callsite)

#### Task 3B: Clean `login/page.tsx`
**File**: `dws-app/src/app/login/page.tsx`

1. Remove unused imports `AuthChangeEvent`, `Session` from line 7
2. Remove `redirectTimeoutRef` (line 40) — never assigned a timeout
3. Remove `isRedirectingRef` (line 41) — set but never read
4. Remove the duplicate cleanup `useEffect` (lines 77-85) that references these dead refs
5. Remove the `formatUSPhoneNumber` wrapper (lines 17-27) — replace callsites with direct `formatPhone` import (renaming if needed). The "test number" backwards-compat logic should not be in production code.

#### Task 3C: Clean `employee/page.tsx`
**File**: `dws-app/src/app/employee/page.tsx`

1. Remove unused parameter `newReceipt` from `handleReceiptAdded` (line 85) — change to `(_: Receipt)` or `()`
2. Remove unused parameter `updatedReceipt` from `handleReceiptUpdated` (line 90) — same treatment
3. Remove the duplicate cleanup `useEffect` (lines 255-262)

#### Task 3D: Clean Hooks and Lib
**Files**: `dws-app/src/hooks/use-admin-receipts.ts`, `dws-app/src/app/api/receipts/route.ts`, `dws-app/src/lib/types.ts`

1. Delete `useUpdateReceipt` export (lines 99-122 in `use-admin-receipts.ts`) — never imported anywhere
2. Remove unused `moveData` destructuring (line 105 in `receipts/route.ts`) — change to `const { error: moveError } = ...`
3. Remove commented-out fields in `types.ts` (lines 16-17): `// jobCode` and `// driveLink`
4. Remove the comment on line 22: `// You might want to define other types here`

#### Task 3E: Clean `page.tsx` (root)
**File**: `dws-app/src/app/page.tsx`

1. Remove the duplicate cleanup `useEffect` (lines 111-118)

#### Task 3F: Clean Dead Commented-Out Code
**File**: `dws-app/src/app/api/categories/route.ts`

1. Delete the commented-out auth check block (lines 8-13)

### Success Criteria

#### Automated Verification:
- [ ] `npm run build` succeeds
- [ ] `npm run lint` passes (or has no new errors)
- [ ] `grep -r 'useUpdateReceipt' dws-app/src/` returns only deletion evidence or nothing
- [ ] `grep -r 'validateFile' dws-app/src/components/receipt-uploader.tsx` returns nothing
- [ ] `grep -r 'AuthChangeEvent' dws-app/src/` returns nothing

---

## Phase 4: Console.log Purge

### Overview
Remove all `console.log` and `console.warn` from the codebase. Keep `console.error` in API route catch blocks only (server-side error logging is defensible). Remove ALL console statements from client components.

### Parallel Tasks

Each task handles a non-overlapping set of files.

#### Task 4A: Client Pages
**Files**:
- `dws-app/src/app/page.tsx` — lines 18, 34, 51, 63, 67, 75, 90, 101
- `dws-app/src/app/login/page.tsx` — lines 52, 56, 65, 110, 123, 136, 149, 154, 159, 164, 169, 175
- `dws-app/src/app/dashboard/page.tsx` — lines 17, 25, 32, 37, 44, 55, 60, 76, 85, 92, 105, 114, 122, 126
- `dws-app/src/app/batch-review/page.tsx` — lines 17, 25, 32, 37, 44, 55, 60, 68, 77, 84, 97, 106, 114, 118
- `dws-app/src/app/employee/page.tsx` — all ~40 console statements
- `dws-app/src/app/users/page.tsx` — line 72

Remove ALL `console.*` calls from these files.

#### Task 4B: Client Components + Hooks
**Files**:
- `dws-app/src/components/receipt-uploader.tsx` — lines 55, 129, 276, 377, 444
- `dws-app/src/components/receipt-details-card.tsx` — lines 63, 114, 170, 221
- `dws-app/src/components/receipt-dashboard.tsx` — lines 232, 238, 245, 247, 290
- `dws-app/src/components/batch-review-dashboard.tsx` — line 127
- `dws-app/src/hooks/use-mobile.tsx` — lines 13, 14

Remove ALL `console.*` calls from these files.

#### Task 4C: API Routes — Strip debug logs, keep catch-block errors
**Files**: All files in `dws-app/src/app/api/`

Strategy: Remove all `console.log` and `console.warn`. Keep `console.error` ONLY inside catch blocks. Remove `console.error` that is used for non-error logging (e.g., "not admin" messages).

Key files:
- `receipts/route.ts` — remove ~40 `console.log` lines, keep ~5 `console.error` in catch blocks
- `receipts/upload/route.ts` — remove ~15 `console.log` lines
- `receipts/bulk-update/route.ts` — remove ~6 `console.log` lines
- `admin/receipts/route.ts` — remove ~7 `console.log` lines
- `receipts/ocr/route.ts` — remove 2 `console.log` lines
- `auth/send-otp/route.ts`, `auth/verify-otp/route.ts`, `categories/route.ts`, `check-duplicate/route.ts`, `admin/users/route.ts`, `admin/users/[id]/route.ts` — keep only catch-block `console.error`

### Success Criteria

#### Automated Verification:
- [ ] `grep -rn 'console\.\(log\|warn\)' dws-app/src/ --include='*.ts' --include='*.tsx' | grep -v node_modules | wc -l` returns 0
- [ ] `npm run build` succeeds

---

## Phase 5: Comment Cleanup

### Overview
Remove all comments that restate what the code does. Keep comments that explain *why* something non-obvious is done. ~370 single-line comments and ~12 JSDoc blocks across non-ui files; most should be deleted.

### Parallel Tasks

#### Task 5A: API Routes
**Files**: All files in `dws-app/src/app/api/`

Top offenders:
- `receipts/route.ts` — ~36 comments (remove all step-numbering, "Normalize to lowercase", "Map to frontend interface", etc.)
- `admin/users/[id]/route.ts` — ~25 comments
- `admin/users/route.ts` — ~21 comments
- `receipts/upload/route.ts` — ~12 comments (remove "Changed import", "Max width", "Max height", etc.)
- `admin/receipts/route.ts` — ~8 comments
- `categories/route.ts` — remove "Ensure these columns exist", "Order by name for consistent dropdown"
- `auth/send-otp/route.ts` — remove "Use the utility and await it", "Ensure the response is a NextResponse object"
- `auth/verify-otp/route.ts` — remove "Use the utility and await it", "Specify the type directly", the 3-line session explanation block

Delete all JSDoc blocks on route handlers and on `formatUSPhoneNumber` (which Phase 7 will consolidate anyway).

**Rule**: If the comment just says what the next line does, delete it. If it explains a non-obvious business rule or workaround, keep it.

#### Task 5B: Components
**Files**: All files in `dws-app/src/components/` (excluding `ui/`)

Top offenders:
- `receipt-dashboard.tsx` — ~39 comments
- `receipt-uploader.tsx` — ~33 comments
- `receipt-details-card.tsx` — ~26 comments
- `batch-review-dashboard.tsx` — ~18 comments
- `user-management-dashboard.tsx` — ~13 comments

#### Task 5C: Hooks, Lib, Pages
**Files**: `dws-app/src/hooks/`, `dws-app/src/lib/`, page files in `dws-app/src/app/`

Hooks — remove identical boilerplate comments from all 4 hook files:
- "Query key factory for consistent cache keys"
- "Fetch function"
- "Main query hook"
- "Hook to invalidate..."
- "Export fetch function for prefetching"

Lib:
- `types.ts` — remove all "From prototype" comments, remove "Updated to include both cases"
- `utils.ts` — remove "Month is 0-indexed", "Check if date is valid", "Create date in local timezone"
- `phone.ts` — remove JSDoc blocks (function names + TypeScript signatures are sufficient)
- `supabaseClient.ts` — remove the lazy-init explanation comment
- `supabaseServerClient.ts` — remove "Make function async", "Await cookies()"

Pages:
- Remove "Cleanup on unmount" comments (the cleanup useEffects themselves are being removed in Phase 3)

### Success Criteria

#### Automated Verification:
- [ ] `npm run build` succeeds
- [ ] Total comment count drops by >60%: `grep -rn '//' dws-app/src/ --include='*.ts' --include='*.tsx' | grep -v node_modules | grep -v 'components/ui' | wc -l` should be < 150

---

## Phase 6: Type System Cleanup

### Overview
Fix the `Receipt` type, eliminate `any` types, and remove redundant type annotations.

### Parallel Tasks

#### Task 6A: Fix `Receipt` Type
**File**: `dws-app/src/lib/types.ts`

1. Collapse the dual-case status union on line 10:
   ```ts
   // Before:
   status: "Pending" | "Approved" | "Rejected" | "Reimbursed" | "pending" | "approved" | "rejected" | "reimbursed";
   // After:
   status: "pending" | "approved" | "rejected" | "reimbursed";
   ```
   The API already normalizes to lowercase (`route.ts:225`). Update any code that compares against capitalized values.

2. Remove the `date` field (line 7) — use `receipt_date` only. Update all consumers to use `receipt_date`. Main consumers:
   - `use-admin-receipts.ts:50` — the `r.date || r.receipt_date` fallback
   - `receipt-details-card.tsx` — any reference to `.date`
   - `employee-receipt-table.tsx` — any reference to `.date`
   - `receipt-table.tsx` — any reference to `.date`

3. Remove the `notes` field (line 14) — use `description` only. Update API responses to stop sending `notes`.

**CAUTION**: This task has the widest blast radius. Verify every consumer compiles after changes.

#### Task 6B: Replace `any` Types
**Files**: `dws-app/src/app/dashboard/page.tsx`, `dws-app/src/app/employee/page.tsx`, `dws-app/src/app/batch-review/page.tsx`

Replace `useState<any>(null)` with `useState<User | null>(null)` (import `User` from `@supabase/supabase-js`):
- `dashboard/page.tsx:11`
- `employee/page.tsx:16`
- `batch-review/page.tsx:11`

#### Task 6C: Remove Redundant Annotations
**Files**: `dws-app/src/components/batch-review-dashboard.tsx`, `dws-app/src/app/api/receipts/route.ts`, `dws-app/src/app/api/auth/send-otp/route.ts`, `dws-app/src/app/api/auth/verify-otp/route.ts`

1. `batch-review-dashboard.tsx` lines 34, 35, 38: `useState<boolean>(false)` → `useState(false)`
2. `batch-review-dashboard.tsx` lines 48, 59: `const decision: "approved" = "approved"` → `const decision = "approved" as const` (or just inline the string at the usage site)
3. `send-otp/route.ts:20`: `{ phone: phone }` → `{ phone }`
4. `verify-otp/route.ts:26-28`: `{ phone: phone, token: token }` → `{ phone, token }`
5. `receipts/route.ts:36`: `{ receipt_date: receipt_date, amount: amount, category_id: category_id }` → shorthand

### Success Criteria

#### Automated Verification:
- [ ] `npm run build` succeeds with zero type errors
- [ ] `grep -rn 'useState<any>' dws-app/src/` returns nothing
- [ ] `grep -rn '"Pending"\|"Approved"\|"Rejected"\|"Reimbursed"' dws-app/src/lib/types.ts` returns nothing

---

## Phase 7: Over-Defensive Code Removal

### Overview
Remove null checks, guards, and branches that can never execute.

### Parallel Tasks

#### Task 7A: API Route Dead Branches
**Files**: All `dws-app/src/app/api/` files

1. `categories/route.ts:26-30` — delete `if (!categories)` block (`.select()` never returns null)
2. `receipts/route.ts:194-197` — delete `if (!receipts)` block (same reason)
3. `receipts/route.ts:209-214` — simplify `getPublicUrl` usage: remove `if (publicUrlData && publicUrlData.publicUrl)` guard, just use `publicUrlData.publicUrl` directly
4. `receipts/route.ts:383-385` — same `getPublicUrl` simplification
5. `admin/receipts/route.ts:82` — remove `publicUrlData?.publicUrl` optional chain, use `publicUrlData.publicUrl`
6. `receipts/route.ts:53` — remove optional chain on `insertError?.message` (it's inside an `if (insertError)` block, so it's non-null)

#### Task 7B: Client Page Defensive Code
**Files**: `dws-app/src/app/dashboard/page.tsx`, `dws-app/src/app/batch-review/page.tsx`

1. Remove redundant `abortController.signal.aborted` checks — keep only `isCancelled`:
   - `dashboard/page.tsx:31, 54, 84, 93` — remove `|| abortController.signal.aborted` / `&& !abortController.signal.aborted`
   - `batch-review/page.tsx:31, 54, 76, 85` — same

2. Simplify the post-auth render guards:
   - `dashboard/page.tsx:150` — the triple check `!user || !userProfile || userProfile.role !== 'admin'` can drop the role check (already enforced before setting state)
   - `batch-review/page.tsx:142` — same

#### Task 7C: Miscellaneous Over-Defensive Code
**Files**: `dws-app/src/hooks/use-admin-receipts.ts`, `dws-app/src/components/batch-review-dashboard.tsx`, `dws-app/src/components/receipt-dashboard.tsx`, `dws-app/src/components/user-management-dashboard.tsx`

1. `use-admin-receipts.ts:50` — remove `r.date || r.receipt_date` fallback (after Phase 6A removes `date` field, this becomes just `r.receipt_date`)
2. `batch-review-dashboard.tsx:29` — `queryError?.message || null` → `queryError?.message ?? null` or just `queryError?.message` (the `|| null` is redundant since the truthiness check works with undefined)
3. `receipt-dashboard.tsx:112` — same `|| null` pattern
4. `user-management-dashboard.tsx:28` — same pattern
5. `user-management-dashboard.tsx:30-31, 41-61` — convert `filteredUsers` from `useState` + `useEffect` to `useMemo`

### Success Criteria

#### Automated Verification:
- [ ] `npm run build` succeeds
- [ ] `grep -n 'signal.aborted' dws-app/src/app/dashboard/page.tsx dws-app/src/app/batch-review/page.tsx` returns nothing

---

## Phase 8: DRY Up Duplication

### Overview
Extract shared patterns and eliminate copy-pasted code. This phase has the highest risk of introducing bugs — each task must verify the build after changes.

### Parallel Tasks

#### Task 8A: Extract Admin Auth Middleware
**Files**: Create `dws-app/src/lib/admin-auth.ts`, modify all 4 API route files

Create a shared function:
```ts
// dws-app/src/lib/admin-auth.ts
import { NextResponse } from 'next/server'
import { createSupabaseServerClient } from './supabaseServerClient'

export async function requireAdmin() {
  const supabase = await createSupabaseServerClient()
  const { data: { session }, error: sessionError } = await supabase.auth.getSession()

  if (sessionError) {
    return { error: NextResponse.json({ error: 'Failed to get session' }, { status: 500 }) }
  }
  if (!session) {
    return { error: NextResponse.json({ error: 'Unauthorized' }, { status: 401 }) }
  }

  const { data: profile } = await supabase
    .from('user_profiles')
    .select('role')
    .eq('user_id', session.user.id)
    .single()

  if (!profile || profile.role !== 'admin') {
    return { error: NextResponse.json({ error: 'Admin access required' }, { status: 403 }) }
  }

  return { supabase, session }
}
```

Replace the 7 duplicated blocks in:
- `admin/receipts/route.ts:20-46`
- `receipts/bulk-update/route.ts:8-34`
- `admin/users/route.ts:21-44` (GET) and `139-162` (POST)
- `admin/users/[id]/route.ts:17-39` (GET), `99-121` (PATCH), `251-273` (DELETE)

Each handler becomes:
```ts
const auth = await requireAdmin()
if ('error' in auth) return auth.error
const { supabase, session } = auth
```

#### Task 8B: Deduplicate `formatUSPhoneNumber`
**Files**: `dws-app/src/app/api/admin/users/route.ts`, `dws-app/src/app/api/admin/users/[id]/route.ts`

1. Delete the local `formatUSPhoneNumber` function from both files:
   - `admin/users/route.ts:274-282`
   - `admin/users/[id]/route.ts:332-340`
2. Add `import { formatUSPhoneNumber } from '@/lib/phone'` to both files

#### Task 8C: Fix `query-provider.tsx` Duplication
**Files**: `dws-app/src/components/providers/query-provider.tsx`

1. Delete the inline `makeQueryClient` and `getQueryClient` (lines 7-37)
2. Add `import { getQueryClient } from '@/lib/query-client'`
3. (Do NOT delete `lib/query-client.ts` — Phase 2 originally planned to, but now it becomes the single source of truth)

**UPDATE**: If Phase 2 already deleted `query-client.ts`, restore it from git. Or alternatively, keep the functions in `query-provider.tsx` and skip this task. The real win was deleting the dead file — pick one location and stick with it.

#### Task 8D: Remove Backwards-Compat Shims
**Files**: `dws-app/src/app/api/receipts/route.ts`

1. In GET handler mapping (~line 221-229): remove `employeeName: "Employee"` and `employeeId: "N/A"`
2. In GET handler mapping: remove `notes: item.description` (keep only `description`)
3. In PATCH handler response mapping (~lines 397-398): same — remove `notes` duplication

Verify the `Receipt` interface (updated in Phase 6A) no longer requires these fields.

### Success Criteria

#### Automated Verification:
- [ ] `npm run build` succeeds
- [ ] `grep -rn 'formatUSPhoneNumber' dws-app/src/app/api/` returns only import statements, no function definitions
- [ ] `grep -c 'auth.getSession' dws-app/src/app/api/admin/receipts/route.ts` returns 0 (moved to shared util)
- [ ] `npm run lint` passes

#### Manual Verification:
- [ ] Login flow works (employee + admin)
- [ ] Receipt submission with OCR auto-fill works
- [ ] Admin dashboard loads receipts correctly
- [ ] Batch review approve/reject flow works
- [ ] User management CRUD works
- [ ] Receipt edit/delete works
- [ ] Bulk reimbursement works
- [ ] CSV export produces correct output

---

## Execution Order

Phases 1-5 are safe to run in any order (pure deletion/removal). Phases 6-8 modify behavior and should run after 1-5 are complete.

Within each phase, all lettered tasks (A, B, C, etc.) can run as parallel agents with no file conflicts.

```
Phase 1 (Security)     ─┐
Phase 2 (Dead Files)    │── All can run in parallel
Phase 3 (Dead Code)     │   (non-overlapping file scopes)
Phase 4 (Console.log)   │
Phase 5 (Comments)     ─┘
         │
         ▼
     npm run build (gate check)
         │
         ▼
Phase 6 (Types)        ─┐
Phase 7 (Over-defense)  │── Run in parallel
Phase 8 (DRY)          ─┘   (mostly non-overlapping, but 6A touches types.ts
         │                    which 7C and 8D depend on — run 6A first)
         ▼
     npm run build + manual smoke test
         │
         ▼
     git push --force origin main
```

## References

- Audit findings: conversation context (2026-04-09)
- File inventory: 6 parallel codebase-analyzer agents
