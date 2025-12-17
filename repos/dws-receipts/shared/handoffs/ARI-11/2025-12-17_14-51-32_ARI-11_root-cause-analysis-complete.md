---
date: 2025-12-17T14:51:32-08:00
researcher: Claude
git_commit: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
branch: receipt-editing
repository: DWS-Receipts
topic: "Dashboard Freeze Bug - Root Cause Analysis Complete"
tags: [bugfix, dialog, radix-ui, react, admin-dashboard, research]
status: complete
last_updated: 2025-12-17
last_updated_by: Claude
type: research
---

# Handoff: ARI-11 Dashboard Freeze Root Cause Analysis Complete

## Task(s)

**Status: Research Complete - Ready for Implementation**

Conducted comprehensive root cause analysis of the dashboard freeze bug where the admin dashboard becomes unresponsive after closing the edit receipt dialog. Used 8 parallel research agents to thoroughly investigate all potential causes.

## Critical References

- `thoughts/shared/research/ARI-11_dashboard-freeze-root-cause-analysis.md` - Full research findings and fix recommendations
- `dws-app/src/components/receipt-dashboard.tsx:920-944` - The broken edit dialog implementation

## Recent changes

No code changes - this was research only. Created research document at `thoughts/shared/research/ARI-11_dashboard-freeze-root-cause-analysis.md`.

## Learnings

### Primary Root Cause (80% confidence)
The admin edit dialog at `receipt-dashboard.tsx:920-944` uses a **conditional rendering pattern** that differs from all other working dialogs:

```tsx
// BROKEN - Dialog unmounts before Radix cleanup runs
{editingReceipt && (
  <Dialog open onOpenChange={...}>
```

vs the working pattern used everywhere else:

```tsx
// WORKING - Dialog stays mounted, only open prop changes
<Dialog open={!!editingReceipt} onOpenChange={...}>
```

When `setEditingReceipt(null)` is called, the Dialog is ripped from DOM before Radix can clean up scroll locks and pointer-events.

### Contributing Causes
1. **Known Radix Bug #3445**: Nested dialogs (ReceiptDetailsCard has an AlertDialog inside) cause `pointer-events: none` to persist on body when unmounting sequence is wrong
2. **Global overflow-hidden**: `layout.tsx:39` has permanent `overflow-hidden` on body which conflicts with Radix scroll lock restoration
3. **State updates after unmount**: `receipt-details-card.tsx` has `finally` blocks that update state after async operations complete, potentially after unmount

### Pattern Comparison
Only the admin edit dialog uses the broken pattern. All 6 other dialogs in the codebase use controlled `open` prop and work correctly:
- `receipt-dashboard.tsx:849` - Bulk update confirmation (works)
- `receipt-dashboard.tsx:892` - Delete confirmation (works)
- `employee-receipt-table.tsx:226` - Employee edit (works)
- `receipt-uploader.tsx:327` - Receipt upload (works)
- `batch-review-dashboard.tsx:556` - Batch submission (works)

## Artifacts

- `thoughts/shared/research/ARI-11_dashboard-freeze-root-cause-analysis.md` - Complete research document with fix recommendations
- `thoughts/shared/handoffs/ARI-11/2025-12-17_14-41-13_ARI-11_dashboard-freeze-on-edit-dialog-close.md` - Previous handoff (superseded by this one)

## Action Items & Next Steps

1. **Primary Fix (Required)**: Change `receipt-dashboard.tsx:920-944` from conditional rendering to controlled open prop pattern:
   ```tsx
   <Dialog open={!!editingReceipt} onOpenChange={(open) => !open && setEditingReceipt(null)}>
     <DialogContent>
       {editingReceipt && <ReceiptDetailsCard ... />}
     </DialogContent>
   </Dialog>
   ```

2. **CSS Fix (Recommended)**: Add to `globals.css`:
   ```css
   body[data-scroll-locked] { margin-right: 0px !important; }
   body:not([data-scroll-locked]) { pointer-events: auto !important; }
   ```

3. **Optional**: Remove `overflow-hidden` from `layout.tsx:39`

4. **Optional**: Add `modal={false}` to nested AlertDialog in `receipt-details-card.tsx:395`

5. **Verify**: Test all close methods (Cancel, Save, Delete, X, Escape, backdrop click) and check body element for lingering `data-scroll-locked`, `pointer-events: none`, or `overflow: hidden`

## Other Notes

### Technical Environment
- React 19.0.0, Next.js 15.3.3
- @radix-ui/react-dialog ^1.1.14
- react-remove-scroll 2.7.0 (transitive dependency)
- No React StrictMode enabled

### Relevant Radix Issues
- https://github.com/radix-ui/primitives/issues/3445 - pointer-events cleanup bug
- https://github.com/shadcn-ui/ui/issues/6988 - Dialog scroll blocking

### Verification Steps
1. Open DevTools before reproducing
2. Click Edit on receipt, then close dialog
3. Check `<body>` for: `data-scroll-locked` attr, `pointer-events: none`, `overflow: hidden` inline styles
4. Verify page is scrollable and interactive after close
