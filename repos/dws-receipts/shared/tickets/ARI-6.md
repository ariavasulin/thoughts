# ARI-6: Fix ESLint errors across codebase (43 issues)

**URL**: https://linear.app/ariav/issue/ARI-6/fix-eslint-errors-across-codebase-43-issues
**Status**: Backlog
**Priority**: High
**Created**: 2025-12-17
**Project**: DWS

## Problem to Solve

The codebase has accumulated 43 ESLint errors across 17 files that need to be resolved to maintain code quality and prevent bugs.

## Summary of Issues

| Category | Count | Severity |
| -- | -- | -- |
| `@typescript-eslint/no-explicit-any` | 11 | Error |
| `@typescript-eslint/no-unused-vars` | 17 | Error |
| `react-hooks/exhaustive-deps` | 6 | Warning |
| `react-hooks/rules-of-hooks` | 1 | Critical |
| `@typescript-eslint/prefer-as-const` | 2 | Error |

## Critical Issue

`receipt-uploader.tsx:47` - Hook called conditionally, violating React's rules of hooks. This must be fixed first as it can cause runtime bugs.

## Files Affected

### API Routes

* `src/app/api/admin/receipts/route.ts:74` - any type
* `src/app/api/categories/route.ts:5` - unused request param
* `src/app/api/receipts/check-duplicate/route.ts:52` - any type
* `src/app/api/receipts/route.ts:105,147,201` - unused vars, any type

### Pages

* `src/app/batch-review/page.tsx` - unused useRef, any type, missing deps
* `src/app/dashboard/page.tsx` - unused useRef, any type, missing deps
* `src/app/employee/page.tsx` - any type, multiple unused vars, missing deps
* `src/app/login/page.tsx` - unused imports, ref warning
* `src/app/page.tsx` - missing deps

### Components

* `src/components/batch-review-dashboard.tsx` - multiple any types, prefer-as-const
* `src/components/receipt-dashboard.tsx` - any type, unused function
* `src/components/receipt-details-card.tsx` - missing deps
* `src/components/receipt-table.tsx` - many unused imports
* `src/components/receipt-uploader.tsx` - **CRITICAL: conditional hook**, unused imports

### UI/Lib

* `src/components/ui/chart.tsx` - unused var
* `src/components/ui/progress.tsx` - unused var
* `src/components/ui/use-toast.ts` - unused var
* `src/lib/local-storage-polyfill.ts` - any types

## Approach

1. Fix critical `rules-of-hooks` violation first
2. Replace `any` types with proper TypeScript types
3. Remove unused imports and variables
4. Add missing dependencies to useEffect hooks or refactor
5. Apply `as const` assertions where needed
