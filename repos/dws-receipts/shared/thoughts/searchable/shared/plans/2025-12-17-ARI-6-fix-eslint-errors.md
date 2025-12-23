# Fix ESLint Errors Across Codebase - Implementation Plan

## Overview

Resolve all 50 ESLint errors across 17 files in the DWS Receipts codebase. This includes fixing a critical React hooks violation, replacing `any` types with proper TypeScript types, removing unused imports/variables, and fixing hook dependency arrays.

## Current State Analysis

Running `npm run lint` reveals **50 issues**:
- `@typescript-eslint/no-explicit-any`: 13 errors
- `@typescript-eslint/no-unused-vars`: 26 errors
- `react-hooks/exhaustive-deps`: 8 warnings
- `react-hooks/rules-of-hooks`: 1 critical
- `@typescript-eslint/prefer-as-const`: 2 errors

### Key Discoveries:
- **Critical bug** at `receipt-uploader.tsx:47` - useEffect called conditionally
- `validateFile` function defined but never used - needs decision (use or remove)
- `getCurrentWeekEndingSaturday` in receipt-dashboard.tsx is unused
- Several `any` types in local-storage-polyfill.ts are intentional for SSR compatibility

## Desired End State

- `npm run lint` returns 0 errors and 0 warnings
- `npm run build` succeeds without type errors
- All React hooks follow rules of hooks
- No unused imports or variables in codebase
- Proper TypeScript types replace all `any` usages (except documented SSR exceptions)

## What We're NOT Doing

- Adding new ESLint rules
- Refactoring business logic
- Adding new features to unused functions (if validateFile is unused, we remove it)
- Creating extensive type definitions for external libraries

## Implementation Approach

Fix issues in order of severity and complexity, verifying after each phase.

---

## Phase 1: Critical Hook Fix

### Overview
Fix the React rules-of-hooks violation that can cause runtime bugs.

### Changes Required:

#### 1. receipt-uploader.tsx
**File**: `dws-app/src/components/receipt-uploader.tsx`
**Changes**: Move conditional logic inside the useEffect

```tsx
// Before (lines 45-50):
if (process.env.NODE_ENV === 'development') {
  useEffect(() => {
    testPlatformDetection()
  }, [])
}

// After:
useEffect(() => {
  if (process.env.NODE_ENV === 'development') {
    testPlatformDetection()
  }
}, [])
```

### Success Criteria:

#### Automated Verification:
- [ ] `cd dws-app && npm run lint 2>&1 | grep "rules-of-hooks"` returns empty (no critical hook errors)
- [ ] `cd dws-app && npm run build` succeeds

#### Manual Verification:
- [ ] Receipt upload still works in development mode

---

## Phase 2: Remove Unused Imports

### Overview
Remove all unused import statements - quick wins that don't affect functionality.

### Changes Required:

#### 1. API Routes
**File**: `dws-app/src/app/api/categories/route.ts`
- Line 5: Remove `request` parameter from function signature or prefix with `_`

**File**: `dws-app/src/app/api/receipts/route.ts`
- Line 147: Remove/prefix unused `request` parameter in GET handler

#### 2. Pages
**File**: `dws-app/src/app/batch-review/page.tsx`
- Line 3: Remove `useRef` from React imports

**File**: `dws-app/src/app/dashboard/page.tsx`
- Line 3: Remove `useRef` from React imports

**File**: `dws-app/src/app/login/page.tsx`
- Line 7: Remove `AuthChangeEvent`, `Session` from imports

#### 3. Components
**File**: `dws-app/src/components/receipt-table.tsx`
- Line 10: Remove `Select`, `SelectContent`, `SelectItem`, `SelectTrigger`, `SelectValue` imports

**File**: `dws-app/src/components/receipt-uploader.tsx`
- Line 6: Remove `Camera` import
- Line 9: Remove `DialogDescription` import
- Line 11: Remove `uuidv4` import

### Success Criteria:

#### Automated Verification:
- [ ] `cd dws-app && npm run lint 2>&1 | grep -c "is defined but never used"` shows reduced count
- [ ] `cd dws-app && npm run build` succeeds

---

## Phase 3: Fix prefer-as-const Issues

### Overview
Apply TypeScript `as const` assertions instead of literal type annotations.

### Changes Required:

**File**: `dws-app/src/components/batch-review-dashboard.tsx`
```tsx
// Line 97 - Before:
const decision: "approved" = "approved";
// After:
const decision = "approved" as const;

// Line 108 - Before:
const decision: "rejected" = "rejected";
// After:
const decision = "rejected" as const;
```

### Success Criteria:

#### Automated Verification:
- [ ] `cd dws-app && npm run lint 2>&1 | grep "prefer-as-const"` returns empty

---

## Phase 4: Fix Unused Variables

### Overview
Remove or prefix unused variables. Remove dead code.

### Changes Required:

#### 1. Remove truly unused code
**File**: `dws-app/src/app/api/receipts/route.ts`
- Line 105: Remove `moveData` variable assignment (result not used)

**File**: `dws-app/src/components/receipt-dashboard.tsx`
- Line 175: Remove `getCurrentWeekEndingSaturday` function (unused)

**File**: `dws-app/src/components/receipt-uploader.tsx`
- Line 127: Remove `validateFile` function (defined but never called)

#### 2. Prefix intentionally unused callback parameters
**File**: `dws-app/src/app/employee/page.tsx`
- Line 49: Change `e` to `_e` in catch block
- Line 85: Change `newReceipt` to `_newReceipt` in callback
- Line 90: Change `updatedReceipt` to `_updatedReceipt` in callback

**File**: `dws-app/src/app/login/page.tsx`
- Line 76: Change destructured `session` to `_session`

**File**: `dws-app/src/components/receipt-table.tsx`
- Line 35: Prefix `onPageChange` with `_`
- Line 36: Prefix `onPageSizeChange` with `_`

**File**: `dws-app/src/components/ui/progress.tsx`
- Line 12: Prefix `max` with `_` or remove from destructuring

**File**: `dws-app/src/components/ui/chart.tsx`
- Line 72: Change `_` variable name to avoid conflict or remove

**File**: `dws-app/src/components/ui/use-toast.ts`
- Line 21: Remove `actionTypes` if only used as type, or adjust

### Success Criteria:

#### Automated Verification:
- [ ] `cd dws-app && npm run lint 2>&1 | grep -c "no-unused-vars"` shows 0
- [ ] `cd dws-app && npm run build` succeeds

---

## Phase 5: Replace `any` Types

### Overview
Define proper TypeScript types for Supabase responses and error handling.

### Changes Required:

#### 1. Define response type interfaces
**File**: `dws-app/src/lib/types.ts` (add new interfaces)
```typescript
// Add Supabase response type for receipt queries
export interface ReceiptQueryResponse {
  id: string;
  user_id: string;
  merchant_name: string | null;
  amount: number | null;
  receipt_date: string | null;
  category_id: string | null;
  status: string;
  image_url: string | null;
  notes: string | null;
  created_at: string;
  user_profiles?: {
    full_name: string | null;
    phone_number: string | null;
  };
  categories?: {
    name: string;
  };
}
```

#### 2. Apply types to API routes
**File**: `dws-app/src/app/api/admin/receipts/route.ts`
- Line 74: Type the map callback parameter with `ReceiptQueryResponse`

**File**: `dws-app/src/app/api/receipts/check-duplicate/route.ts`
- Line 52: Type the response

**File**: `dws-app/src/app/api/receipts/route.ts`
- Line 201: Type the map callback parameter

#### 3. Fix page component types
**File**: `dws-app/src/app/batch-review/page.tsx`
- Line 11: Use `Session | null` instead of `any` for getSession

**File**: `dws-app/src/app/dashboard/page.tsx`
- Line 11: Use `Session | null` instead of `any`

**File**: `dws-app/src/app/employee/page.tsx`
- Line 16: Use `User` type from Supabase

#### 4. Fix component types
**File**: `dws-app/src/components/batch-review-dashboard.tsx`
- Lines 67, 184: Type map callbacks with response interface
- Lines 80, 193: Change `any` to `unknown` in catch blocks with type guards

**File**: `dws-app/src/components/receipt-dashboard.tsx`
- Line 128: Type map callback

#### 5. Handle intentional any in polyfill
**File**: `dws-app/src/lib/local-storage-polyfill.ts`
- Lines 8, 17: Add `// eslint-disable-next-line @typescript-eslint/no-explicit-any` comments with explanation

### Success Criteria:

#### Automated Verification:
- [ ] `cd dws-app && npm run lint 2>&1 | grep -c "no-explicit-any"` shows 0 (or only disabled lines)
- [ ] `cd dws-app && npm run build` succeeds with no type errors

---

## Phase 6: Fix Hook Dependencies

### Overview
Add missing dependencies to useEffect hooks or refactor with useCallback.

### Changes Required:

#### 1. Add router to dependency arrays
**File**: `dws-app/src/app/batch-review/page.tsx`
- Line 111: Add `router` to useEffect dependencies

**File**: `dws-app/src/app/dashboard/page.tsx`
- Line 119: Add `router` to useEffect dependencies

#### 2. Wrap functions in useCallback
**File**: `dws-app/src/app/employee/page.tsx`
- Wrap `fetchReceipts` in useCallback
- Review `loading` dependency at line 252 - may need eslint-disable comment if intentional

#### 3. Fix ref cleanup warning
**File**: `dws-app/src/app/login/page.tsx`
- Line 94: Copy ref to variable before cleanup:
```tsx
useEffect(() => {
  const timeoutRef = redirectTimeoutRef.current;
  return () => {
    if (timeoutRef) clearTimeout(timeoutRef);
  };
}, []);
```

#### 4. Add missing dependencies
**File**: `dws-app/src/app/page.tsx`
- Line 108: Review and add `loading` if appropriate, or add eslint-disable comment

**File**: `dws-app/src/components/receipt-details-card.tsx`
- Line 107: Add `categoryId` to useEffect dependencies

### Success Criteria:

#### Automated Verification:
- [ ] `cd dws-app && npm run lint` returns 0 errors and 0 warnings
- [ ] `cd dws-app && npm run build` succeeds

#### Manual Verification:
- [ ] Login flow works correctly (no redirect issues)
- [ ] Employee page loads receipts correctly
- [ ] Receipt details card displays categories
- [ ] Dashboard and batch-review pages load correctly

---

## Testing Strategy

### Unit Tests:
- Run existing test suite to ensure no regressions

### Integration Tests:
- Full app build with `npm run build`
- Type checking with TypeScript compiler

### Manual Testing Steps:
1. Login with phone OTP - verify redirect works
2. Navigate to employee page - verify receipts load
3. Upload a new receipt - verify OCR and form work
4. Navigate to admin dashboard - verify receipts display
5. Use batch review - verify approve/reject works
6. View receipt details - verify category displays

## Performance Considerations

- No performance impact expected - these are code quality fixes
- useCallback additions may slightly improve performance by reducing re-renders

## Migration Notes

- No database changes required
- No API changes - all fixes are internal code quality

## References

- Original ticket: [ARI-6](https://linear.app/ariav/issue/ARI-6/fix-eslint-errors-across-codebase-43-issues)
- Research document: `thoughts/shared/research/2025-12-17-ARI-6-eslint-errors-analysis.md`
- ESLint rules documentation: https://typescript-eslint.io/rules/
- React hooks rules: https://react.dev/reference/rules/rules-of-hooks
