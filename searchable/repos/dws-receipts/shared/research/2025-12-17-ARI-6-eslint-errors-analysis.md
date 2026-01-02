# Research: ESLint Errors Analysis - ARI-6

**Ticket**: [ARI-6](https://linear.app/ariav/issue/ARI-6/fix-eslint-errors-across-codebase-43-issues)
**Date**: 2025-12-17
**Status**: Research Complete

## Summary

Running `npm run lint` reveals **50 issues** (not 43 as originally estimated) across 17 files:

| Category | Count | Severity |
|----------|-------|----------|
| `@typescript-eslint/no-explicit-any` | 13 | Error |
| `@typescript-eslint/no-unused-vars` | 26 | Error |
| `react-hooks/exhaustive-deps` | 8 | Warning |
| `react-hooks/rules-of-hooks` | 1 | **Critical** |
| `@typescript-eslint/prefer-as-const` | 2 | Error |

## Critical Issue Analysis

### 1. `react-hooks/rules-of-hooks` - CRITICAL FIX FIRST

**File**: `src/components/receipt-uploader.tsx:47`

```tsx
// Current code (lines 45-50):
if (process.env.NODE_ENV === 'development') {
  // Test on component mount
  useEffect(() => {
    testPlatformDetection()
  }, [])
}
```

**Problem**: Hook is called conditionally based on `NODE_ENV`, violating React's rules that hooks must be called in the exact same order in every render.

**Fix**: Move the condition inside the useEffect:
```tsx
useEffect(() => {
  if (process.env.NODE_ENV === 'development') {
    testPlatformDetection()
  }
}, [])
```

---

## Issue-by-Issue Analysis

### `@typescript-eslint/no-explicit-any` (13 instances)

| File | Line | Variable | Recommended Fix |
|------|------|----------|-----------------|
| `api/admin/receipts/route.ts` | 74 | `item: any` in map | Define interface for Supabase response type |
| `api/receipts/check-duplicate/route.ts` | 52 | `item: any` | Define typed response |
| `api/receipts/route.ts` | 201 | `item: any` | Define typed response |
| `batch-review/page.tsx` | 11 | `getSession` typing | Use `Session \| null` |
| `dashboard/page.tsx` | 11 | `getSession` typing | Use `Session \| null` |
| `employee/page.tsx` | 16 | `user: any` | Use `User` from `@supabase/supabase-js` |
| `batch-review-dashboard.tsx` | 67 | `item: any` | Define `ReceiptResponse` interface |
| `batch-review-dashboard.tsx` | 80 | `err: any` in catch | Use `unknown` and type guard |
| `batch-review-dashboard.tsx` | 184 | `item: any` | Same as line 67 |
| `batch-review-dashboard.tsx` | 193 | `err: any` | Use `unknown` |
| `receipt-dashboard.tsx` | 128 | `item: any` | Define response interface |
| `local-storage-polyfill.ts` | 8, 17 | `globalThis as any` | Required for SSR polyfill - use `// eslint-disable-next-line` |

**Approach**:
- Most can be fixed by defining proper interfaces for Supabase query responses
- Error catches should use `unknown` with type guards
- The `local-storage-polyfill.ts` `any` usages are intentional for SSR - add eslint-disable comments

### `@typescript-eslint/no-unused-vars` (26 instances)

**Unused Imports (easy fixes - just remove)**:
| File | Variables to Remove |
|------|---------------------|
| `api/categories/route.ts:5` | `request` parameter |
| `api/receipts/route.ts:147` | `request` parameter in GET |
| `batch-review/page.tsx:3` | `useRef` import |
| `dashboard/page.tsx:3` | `useRef` import |
| `login/page.tsx:7` | `AuthChangeEvent`, `Session` imports |
| `receipt-table.tsx:10` | `Select`, `SelectContent`, `SelectItem`, `SelectTrigger`, `SelectValue` |
| `receipt-uploader.tsx:6` | `Camera` |
| `receipt-uploader.tsx:9` | `DialogDescription` |
| `receipt-uploader.tsx:11` | `uuidv4` |
| `chart.tsx:72` | `_` (underscore variable) |
| `use-toast.ts:21` | `actionTypes` |

**Unused Variables (need investigation)**:
| File | Variable | Action |
|------|----------|--------|
| `api/receipts/route.ts:105` | `moveData` | Remove - result of `.move()` not used |
| `employee/page.tsx:49` | `e` in catch | Replace with `_e` or remove |
| `employee/page.tsx:85` | `newReceipt` param | Prefix with `_` - callback receives value not used |
| `employee/page.tsx:90` | `updatedReceipt` param | Prefix with `_` |
| `login/page.tsx:76` | `session` | Prefix with `_` - destructured but unused |
| `receipt-dashboard.tsx:175` | `getCurrentWeekEndingSaturday` | Remove if unused, or investigate intended use |
| `receipt-table.tsx:35,36` | `onPageChange`, `onPageSizeChange` | These are interface props - need `_` prefix |
| `receipt-uploader.tsx:127` | `validateFile` | Function defined but never called - either use it or remove |
| `progress.tsx:12` | `max` | Destructured but unused in component |

### `react-hooks/exhaustive-deps` (8 instances)

| File | Line | Missing Dependency | Recommended Fix |
|------|------|--------------------|-----------------|
| `batch-review/page.tsx` | 111 | `router` | Add to deps array |
| `dashboard/page.tsx` | 119 | `router` | Add to deps array |
| `employee/page.tsx` | 82 | `fetchReceipts` | Wrap `fetchReceipts` in `useCallback` |
| `employee/page.tsx` | 252 | `loading` | Review if intentional - may need `// eslint-disable-next-line` |
| `login/page.tsx` | 94 | ref cleanup | Copy ref to variable before cleanup |
| `page.tsx` | 108 | `loading` | Review if intentional |
| `receipt-details-card.tsx` | 107 | `categoryId` | Add to deps array |
| `receipt-uploader.tsx` | 49 | `testPlatformDetection` | Will be fixed when addressing rules-of-hooks |

**Note on `router` dependency**: The warning is technically correct, but `router` from `next/navigation` is stable and shouldn't cause re-renders. Options:
1. Add `router` to deps (recommended for lint compliance)
2. Use `// eslint-disable-next-line` with comment explaining stability

### `@typescript-eslint/prefer-as-const` (2 instances)

**File**: `src/components/batch-review-dashboard.tsx`

```tsx
// Line 97:
const decision: "approved" = "approved";

// Line 108:
const decision: "rejected" = "rejected";
```

**Fix**: Use `as const` instead of type annotation:
```tsx
const decision = "approved" as const;
const decision = "rejected" as const;
```

---

## Implementation Plan (Prioritized)

### Phase 1: Critical Fix
1. Fix `receipt-uploader.tsx` conditional hook violation

### Phase 2: Easy Wins (Quick cleanup)
2. Remove unused imports across all files
3. Fix `prefer-as-const` issues in batch-review-dashboard.tsx

### Phase 3: Unused Variables
4. Prefix intentionally unused callback parameters with `_`
5. Remove truly unused variables/functions
6. Remove or use `validateFile` in receipt-uploader.tsx

### Phase 4: Type Safety
7. Define interfaces for Supabase responses to replace `any` types
8. Fix error catch types (use `unknown` with guards)
9. Add eslint-disable for intentional `any` in local-storage-polyfill.ts

### Phase 5: Hook Dependencies
10. Add missing dependencies to useEffect arrays
11. Wrap functions in useCallback where needed
12. Add comments/disables for intentional omissions

---

## Files by Fix Complexity

### Simple (import/variable removal only)
- `src/app/api/categories/route.ts`
- `src/components/receipt-table.tsx`
- `src/components/ui/chart.tsx`
- `src/components/ui/progress.tsx`
- `src/components/ui/use-toast.ts`
- `src/app/login/page.tsx`
- `src/app/batch-review/page.tsx` (import only)
- `src/app/dashboard/page.tsx` (import only)

### Medium (type fixes + some refactoring)
- `src/components/batch-review-dashboard.tsx`
- `src/components/receipt-dashboard.tsx`
- `src/app/api/receipts/route.ts`
- `src/app/api/admin/receipts/route.ts`
- `src/app/api/receipts/check-duplicate/route.ts`
- `src/lib/local-storage-polyfill.ts`

### Complex (logic changes needed)
- `src/components/receipt-uploader.tsx` - Critical hook fix + unused function decision
- `src/app/employee/page.tsx` - Multiple hook deps + useCallback refactoring
- `src/app/page.tsx` - Review loading dependency
- `src/components/receipt-details-card.tsx`

---

## Estimated Effort

- **Phase 1**: 5 minutes
- **Phase 2**: 15 minutes
- **Phase 3**: 20 minutes
- **Phase 4**: 45 minutes (need to create proper types)
- **Phase 5**: 30 minutes

**Total**: ~2 hours of focused work

---

## Risks and Concerns

1. **`validateFile` function**: Defined in receipt-uploader.tsx but never called. Was this intended functionality that was never wired up? Should verify with product requirements.

2. **`getCurrentWeekEndingSaturday` function**: Defined in receipt-dashboard.tsx but unused. May have been intended for a feature - should verify before removing.

3. **Hook dependencies**: Some omissions may be intentional (e.g., `loading` state to prevent loops). Will need careful review to not introduce bugs.

4. **Type definitions**: Creating proper Supabase response interfaces may reveal other type issues. Should be prepared to handle those.

---

## Notes for Implementation

- Run `npm run lint` after each phase to verify fixes
- Run `npm run build` after completing all phases to ensure no runtime errors introduced
- Consider adding a pre-commit hook to prevent future ESLint errors
