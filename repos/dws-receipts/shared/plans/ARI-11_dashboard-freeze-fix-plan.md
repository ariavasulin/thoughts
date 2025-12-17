# ARI-11: Dashboard Freeze Fix - Implementation Plan

**Date:** 2025-12-17
**Status:** Ready for Implementation
**Ticket:** ARI-11
**Based on:** `thoughts/shared/research/ARI-11_dashboard-freeze-root-cause-analysis.md`

## Problem Summary

The admin dashboard becomes unresponsive after closing the edit receipt dialog. The root cause is a conditional rendering pattern that removes the Dialog from DOM before Radix UI can clean up scroll locks and pointer-events.

## Success Criteria

1. Dashboard remains interactive after closing edit dialog via any method
2. No `data-scroll-locked` attribute persists on body after dialog close
3. No `pointer-events: none` inline style persists on body after dialog close
4. Page remains scrollable after dialog operations
5. All existing dialog functionality continues to work

## Implementation Steps

### Step 1: Fix Dialog Rendering Pattern (Required)

**File:** `dws-app/src/components/receipt-dashboard.tsx`
**Lines:** 920-944

Change from conditional rendering to controlled open prop:

**Before:**
```tsx
{editingReceipt && (
  <Dialog open onOpenChange={(open) => !open && setEditingReceipt(null)}>
    <DialogContent className="bg-transparent border-none p-0 max-w-md">
      <DialogTitle className="sr-only">Edit Receipt</DialogTitle>
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
    </DialogContent>
  </Dialog>
)}
```

**After:**
```tsx
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

**Key changes:**
1. Remove outer `{editingReceipt && ...}` conditional wrapper
2. Change `open` to `open={!!editingReceipt}` (controlled by state)
3. Move conditional rendering inside DialogContent around ReceiptDetailsCard only

### Step 2: Add CSS Safety Net (Recommended)

**File:** `dws-app/src/app/globals.css`
**Location:** End of file (after line 123)

Add the following CSS:

```css
/* Fix for Radix Dialog scroll lock cleanup issues (ARI-11) */
body[data-scroll-locked] {
  margin-right: 0px !important;
}

/* Prevent pointer-events: none from persisting after dialog close */
body:not([data-scroll-locked]) {
  pointer-events: auto !important;
}
```

**Purpose:** Acts as a safety net to ensure body interactivity is restored even if Radix cleanup fails in edge cases.

### Step 3: Review overflow-hidden (Optional)

**File:** `dws-app/src/app/layout.tsx`
**Line:** 39

Current:
```tsx
className={`${geistSans.variable} ${geistMono.variable} antialiased overflow-hidden`}
```

**Decision needed:** The `overflow-hidden` class on body may interfere with Radix scroll lock restoration. Consider:
- **Option A:** Remove it entirely if not needed
- **Option B:** Change to `overflow-y-auto` for controlled scrolling
- **Option C:** Keep as-is if the CSS fix in Step 2 is sufficient

**Recommendation:** Start with Steps 1-2, test, and only modify layout.tsx if issues persist.

### Step 4: Nested AlertDialog Fix (Optional)

**File:** `dws-app/src/components/receipt-details-card.tsx`
**Line:** 395

If nested dialog issues persist after Steps 1-2, add `modal={false}`:

```tsx
<AlertDialog
  open={showDeleteConfirm}
  onOpenChange={setShowDeleteConfirm}
  modal={false}
>
```

**Note:** Only apply this if the delete confirmation dialog within the edit dialog causes issues.

## Verification Checklist

After implementation, test each close method:

### Close Methods to Test
- [ ] Click "Cancel" button
- [ ] Click "Save Changes" button (successful save)
- [ ] Click "Delete" button (complete deletion flow)
- [ ] Click X button (top right)
- [ ] Press Escape key
- [ ] Click backdrop (outside dialog)

### For Each Close Method, Verify
- [ ] Dashboard remains interactive (can click buttons, scroll)
- [ ] No console errors related to React state updates
- [ ] Body element has no `data-scroll-locked` attribute
- [ ] Body element has no `pointer-events: none` inline style
- [ ] Body element has no unexpected `overflow: hidden` inline style

### DevTools Inspection Steps
1. Open Chrome DevTools before testing
2. Select Elements tab
3. Navigate to `<body>` element
4. Open edit dialog, then close it
5. Inspect body for lingering attributes/styles

## Rollback Plan

If issues arise:
1. Revert `receipt-dashboard.tsx` changes to restore conditional rendering
2. Remove CSS additions from `globals.css`
3. Document any new findings for further investigation

## Files Modified

| File | Change | Priority |
|------|--------|----------|
| `dws-app/src/components/receipt-dashboard.tsx` | Dialog pattern fix | Required |
| `dws-app/src/app/globals.css` | CSS safety net | Recommended |
| `dws-app/src/app/layout.tsx` | Remove overflow-hidden | Optional |
| `dws-app/src/components/receipt-details-card.tsx` | modal={false} | Optional |

## References

- Research document: `thoughts/shared/research/ARI-11_dashboard-freeze-root-cause-analysis.md`
- Radix UI Dialog docs: https://www.radix-ui.com/primitives/docs/components/dialog
- Related Radix issue: https://github.com/radix-ui/primitives/issues/3445
