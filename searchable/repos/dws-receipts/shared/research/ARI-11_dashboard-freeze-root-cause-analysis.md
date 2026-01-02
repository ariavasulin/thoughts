# ARI-11: Dashboard Freeze on Edit Dialog Close - Root Cause Analysis

**Date:** 2025-12-17
**Status:** Research Complete
**Ticket:** ARI-11

## Executive Summary

The admin dashboard freezes when closing the edit receipt dialog due to **multiple contributing factors**:

1. **Incorrect Dialog rendering pattern** - conditional unmount prevents Radix cleanup
2. **Known Radix bug #3445** - nested dialogs cause `pointer-events: none` persistence
3. **Global `overflow-hidden` on body** - conflicts with Radix scroll lock restoration
4. **State updates after unmount** - `finally` blocks execute on unmounted component

## Root Cause Analysis

### Primary Cause: Conditional Rendering Pattern (80% confidence)

**Location:** `dws-app/src/components/receipt-dashboard.tsx:920-944`

The admin edit dialog uses a pattern that **differs from all other working dialogs**:

```tsx
// BROKEN PATTERN (admin edit dialog)
{editingReceipt && (
  <Dialog open onOpenChange={(open) => !open && setEditingReceipt(null)}>
    <DialogContent>
      <ReceiptDetailsCard ... />
    </DialogContent>
  </Dialog>
)}
```

**Why it fails:**
1. When `setEditingReceipt(null)` is called, React immediately re-renders
2. The `{editingReceipt && ...}` evaluates to `false`
3. The entire Dialog is removed from DOM **synchronously**
4. Radix's cleanup effects (scroll lock removal, `pointer-events` restoration) **never execute**
5. Body remains locked with `overflow: hidden` and/or `pointer-events: none`

**Working pattern used elsewhere:**
```tsx
// WORKING PATTERN (delete dialog, employee edit, etc.)
<Dialog open={!!editingReceipt} onOpenChange={(open) => !open && setEditingReceipt(null)}>
  <DialogContent>
    {editingReceipt && <ReceiptDetailsCard ... />}
  </DialogContent>
</Dialog>
```

**Key difference:** Dialog stays mounted in DOM, only the `open` prop changes. Radix can properly animate out and clean up.

### Contributing Cause: Known Radix Bug #3445 (60% confidence)

**GitHub:** https://github.com/radix-ui/primitives/issues/3445

ReceiptDetailsCard contains a nested AlertDialog for delete confirmation:

```
receipt-dashboard.tsx Dialog (editingReceipt)
  └─ ReceiptDetailsCard
       └─ AlertDialog (showDeleteConfirm)  ← Nested modal
```

When multiple `DismissableLayer` instances unmount in sequence:
1. Nested AlertDialog saves body's `pointer-events` state
2. Parent Dialog also saves state (which is now `none`)
3. Cleanup order causes incorrect restoration
4. `pointer-events: none` persists on body

### Contributing Cause: Global overflow-hidden (40% confidence)

**Location:** `dws-app/src/app/layout.tsx:39`

```tsx
<body className={`${geistSans.variable} ${geistMono.variable} antialiased overflow-hidden`}>
```

The body has **permanent** `overflow-hidden` applied. This conflicts with Radix's scroll lock:
- Radix uses `react-remove-scroll` (v2.7.0) internally
- When dialog opens, Radix adds `data-scroll-locked` attribute and inline styles
- When dialog closes, Radix tries to restore original state
- The Tailwind class persists, potentially causing conflicts

### Minor Cause: State Updates After Unmount (20% confidence)

**Location:** `dws-app/src/components/receipt-details-card.tsx`

```tsx
// Lines 173-174
} finally {
  setIsCheckingDuplicate(false);  // Runs after onEditSuccess unmounts component
}

// Lines 257-260
} finally {
  setIsDeleting(false);
  setShowDeleteConfirm(false);  // Runs after onDelete unmounts component
}
```

These typically cause React warnings, not freezes, but indicate lifecycle issues.

## Evidence: Pattern Comparison

| Component | Location | Pattern | Works? |
|-----------|----------|---------|--------|
| Bulk Update Dialog | receipt-dashboard.tsx:849 | `open={boolean}` | ✅ |
| Delete AlertDialog | receipt-dashboard.tsx:892 | `open={!!object}` | ✅ |
| **Edit Dialog** | receipt-dashboard.tsx:921 | `{obj && <Dialog open>}` | ❌ |
| Employee Edit | employee-receipt-table.tsx:226 | `open={boolean}` | ✅ |
| Receipt Upload | receipt-uploader.tsx:327 | `open={boolean}` | ✅ |
| Batch Review | batch-review-dashboard.tsx:556 | `open={boolean}` | ✅ |

**Only the admin edit dialog uses the conditional rendering pattern.**

## Technical Environment

| Package | Version |
|---------|---------|
| React | 19.0.0 |
| Next.js | 15.3.3 |
| @radix-ui/react-dialog | ^1.1.14 |
| @radix-ui/react-alert-dialog | ^1.1.15 |
| react-remove-scroll | 2.7.0 (transitive) |

## Recommended Fixes

### Fix 1: Change Dialog Pattern (Primary - Must Do)

**File:** `dws-app/src/components/receipt-dashboard.tsx:920-944`

Change from conditional rendering to controlled open prop:

```tsx
// Before (broken):
{editingReceipt && (
  <Dialog open onOpenChange={(open) => !open && setEditingReceipt(null)}>
    <DialogContent className="bg-transparent border-none p-0 max-w-md">
      <DialogTitle className="sr-only">Edit Receipt</DialogTitle>
      <ReceiptDetailsCard ... />
    </DialogContent>
  </Dialog>
)}

// After (fixed):
<Dialog open={!!editingReceipt} onOpenChange={(open) => !open && setEditingReceipt(null)}>
  <DialogContent className="bg-transparent border-none p-0 max-w-md">
    <DialogTitle className="sr-only">Edit Receipt</DialogTitle>
    {editingReceipt && (
      <ReceiptDetailsCard
        mode="edit"
        receiptId={editingReceipt.id}
        initialData={{
          receipt_date: editingReceipt.date,
          amount: editingReceipt.amount,
          category_id: editingReceipt.category_id,
          notes: editingReceipt.notes || editingReceipt.description,
        }}
        onSubmit={() => {}}
        onCancel={() => setEditingReceipt(null)}
        onEditSuccess={handleEditSuccess}
        onDelete={() => {
          setEditingReceipt(null);
          triggerRefresh();
        }}
      />
    )}
  </DialogContent>
</Dialog>
```

### Fix 2: CSS Override for Radix Scroll Lock (Secondary)

**File:** `dws-app/src/app/globals.css`

Add at the end of the file:

```css
/* Fix for Radix Dialog scroll lock cleanup issues */
body[data-scroll-locked] {
  margin-right: 0px !important;
}

/* Prevent pointer-events: none from persisting */
body:not([data-scroll-locked]) {
  pointer-events: auto !important;
}
```

### Fix 3: Remove or Scope Global overflow-hidden (Optional)

**File:** `dws-app/src/app/layout.tsx:39`

Either remove `overflow-hidden` or scope it to specific pages that need it:

```tsx
// Option A: Remove it
<body className={`${geistSans.variable} ${geistMono.variable} antialiased`}>

// Option B: Use overflow-y-auto instead (allows scrolling but contains content)
<body className={`${geistSans.variable} ${geistMono.variable} antialiased overflow-y-auto`}>
```

### Fix 4: Handle Nested AlertDialog (Optional)

**File:** `dws-app/src/components/receipt-details-card.tsx:395`

Add `modal={false}` to prevent nested modal issues:

```tsx
<AlertDialog
  open={showDeleteConfirm}
  onOpenChange={setShowDeleteConfirm}
  modal={false}  // Prevents nested scroll lock issues
>
```

## Verification Steps

After implementing fixes, verify by:

1. Open browser DevTools before testing
2. Click Edit on any receipt in admin dashboard
3. Close the dialog (Cancel, Save, Delete, X, Escape, or click backdrop)
4. Check `<body>` element for:
   - No `data-scroll-locked` attribute after close
   - No `pointer-events: none` inline style
   - No unexpected `overflow: hidden` inline style
5. Verify page is scrollable and interactive
6. Repeat for all close methods

## Files Modified During Research

None - this was research only.

## Files to Modify During Implementation

1. `dws-app/src/components/receipt-dashboard.tsx:920-944` - **Required**
2. `dws-app/src/app/globals.css` - **Recommended**
3. `dws-app/src/app/layout.tsx:39` - **Optional**
4. `dws-app/src/components/receipt-details-card.tsx:395` - **Optional**

## References

- [Radix UI Dialog Docs](https://www.radix-ui.com/primitives/docs/components/dialog)
- [GitHub Issue #3445 - pointer-events cleanup bug](https://github.com/radix-ui/primitives/issues/3445)
- [Shadcn UI Issue #6988 - Dialog scroll blocking](https://github.com/shadcn-ui/ui/issues/6988)
- [react-remove-scroll package](https://www.npmjs.com/package/react-remove-scroll)
