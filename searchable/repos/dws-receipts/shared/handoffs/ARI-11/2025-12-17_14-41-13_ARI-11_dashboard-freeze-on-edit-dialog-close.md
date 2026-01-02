---
date: 2025-12-17T14:41:13-08:00
researcher: Claude
git_commit: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
branch: receipt-editing
repository: DWS-Receipts
topic: "Dashboard Freeze Bug - Edit Dialog Close"
tags: [bugfix, dialog, radix-ui, react, admin-dashboard]
status: in_progress
last_updated: 2025-12-17
last_updated_by: Claude
type: bugfix
---

# Handoff: ARI-11 Dashboard freezes when closing edit receipt dialog

## Task(s)
**Status: Work in Progress - Bug not yet resolved**

User reported that on the admin dashboard, when clicking "Edit" on a receipt and then exiting the dialog (via Cancel, Save, or any action), the entire dashboard freezes. The user cannot scroll or interact with anything until manually refreshing the page.

This is a regression introduced after ARI-11 (receipt editing approval logic) was completed.

## Critical References
- `dws-app/src/components/receipt-dashboard.tsx` - Main admin dashboard with edit dialog
- `dws-app/src/components/receipt-details-card.tsx` - Edit form component
- `dws-app/src/components/ui/dialog.tsx` - shadcn/Radix Dialog component

## Recent changes

Changes made attempting to fix the issue (NOT YET WORKING):

1. `dws-app/src/components/receipt-details-card.tsx:163-169` - Changed edit success flow to avoid calling both `onEditSuccess` AND `onCancel` (which caused double state updates)

2. `dws-app/src/components/receipt-details-card.tsx:245-250` - Changed delete success flow similarly

3. `dws-app/src/components/receipt-dashboard.tsx:920-944` - Changed Dialog rendering pattern from always-mounted with controlled `open` prop to conditionally rendered:
   ```tsx
   // Before (not working):
   <Dialog open={!!editingReceipt} onOpenChange={...}>
     {editingReceipt && <ReceiptDetailsCard />}
   </Dialog>

   // After (still not working):
   {editingReceipt && (
     <Dialog open onOpenChange={...}>
       <ReceiptDetailsCard />
     </Dialog>
   )}
   ```

## Learnings

1. **Symptom**: Page completely freezes - user cannot scroll or click anything. This strongly indicates Radix Dialog's modal overlay (`fixed inset-0 z-50`) is getting stuck, and/or the body scroll lock (`overflow: hidden` on body) is not being released.

2. **Root cause NOT yet identified**: Attempted fixes focused on:
   - Eliminating redundant `setEditingReceipt(null)` calls
   - Changing from controlled `open` prop to conditional rendering
   - Neither fixed the issue

3. **Dialog overlay structure** (`dialog.tsx:33-46`):
   ```tsx
   <DialogPrimitive.Overlay className="fixed inset-0 z-50 bg-black/50" />
   ```
   This overlay blocks all interaction when visible.

4. **DialogContent** in dashboard uses custom styling: `className="bg-transparent border-none p-0 max-w-md"` - this overrides default styles but shouldn't affect modal behavior.

## Artifacts
- `dws-app/src/components/receipt-dashboard.tsx:920-944` - Edit dialog implementation
- `dws-app/src/components/receipt-details-card.tsx:126-176` - handleSubmit for edit mode
- `dws-app/src/components/receipt-details-card.tsx:227-258` - handleDelete
- `dws-app/src/components/ui/dialog.tsx` - shadcn Dialog component

## Action Items & Next Steps

1. **Debug further** - Need to inspect browser DevTools when bug occurs:
   - Check if `<body>` still has `overflow: hidden` or similar styles
   - Check if DialogOverlay element is still in DOM after closing
   - Check for any React errors in console
   - Check if the Dialog portal is being properly cleaned up

2. **Alternative fixes to try**:
   - Add `modal={false}` prop to Dialog to disable modal behavior (test if this confirms modal is the issue)
   - Use `forceMount` prop on DialogOverlay/DialogContent for more control
   - Try adding a key prop to force remount: `<Dialog key={editingReceipt?.id}>`
   - Check if there's a Radix UI version issue or known bug
   - Try wrapping state updates in `setTimeout` or `requestAnimationFrame` to let Dialog animations complete

3. **Verify the flow**:
   - Add console.logs to trace exactly when `setEditingReceipt(null)` is called
   - Check if `onOpenChange` is being called and with what value
   - Verify React isn't throwing errors during unmount

4. **Consider alternative approaches**:
   - Use a Sheet/Drawer component instead of Dialog
   - Use a separate route/page for editing instead of modal
   - Implement custom modal without Radix

## Other Notes

- The issue happens with ALL exit methods (Cancel button, Save button, Delete, clicking X, pressing Escape, clicking backdrop) suggesting the problem is fundamental to how the Dialog closes, not specific button handlers.

- The ReceiptDetailsCard component has `finally` blocks that update state after async operations - these could be updating state on an unmounted component, though this typically causes warnings not freezes.

- The edit dialog was added as part of ARI-11. The delete confirmation dialog (`AlertDialog`) uses a different pattern and may or may not have the same issue - worth checking.

- Relevant Radix Dialog docs: https://www.radix-ui.com/primitives/docs/components/dialog
