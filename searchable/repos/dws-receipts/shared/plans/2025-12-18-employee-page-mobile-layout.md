# Employee Page Mobile Layout Optimization

## Overview

Optimize the employee receipts page for mobile devices (minimum 320px width). Focus on making the receipts table fit properly on small screens and using Drawer (bottom sheet) for modals on mobile instead of Dialog.

## Current State Analysis

**Table Issues:**
- 5 columns (Date, Amount, Status, Photo, Actions) with no explicit widths
- Cells use `whitespace-nowrap` and `p-2` padding - too wide for mobile
- "Actions" column header is unnecessary (icon is self-explanatory)
- Date format "Dec 18, 2024" takes ~80px, could be shorter on mobile
- "View" text in Photo column wastes space on mobile

**Modal Issues:**
- Both edit dialog (`employee-receipt-table.tsx:226`) and upload confirmation (`receipt-uploader.tsx:344`) use `Dialog` component
- Dialog works but Drawer (bottom sheet) is more natural on mobile

**Available Infrastructure:**
- `useMobile()` hook exists at `dws-app/src/hooks/use-mobile.tsx` (768px breakpoint)
- `Drawer` component exists at `dws-app/src/components/ui/drawer.tsx` (unused)
- Both dialogs already use `ReceiptDetailsCard` component

## Desired End State

1. **Table fits on 320px screens** without horizontal scroll (or minimal scroll)
2. **Edit and upload confirmation modals** use Drawer on mobile, Dialog on desktop
3. **Cleaner table appearance** with empty Actions header and compact mobile styling

### Verification:
- Test on 320px viewport (Chrome DevTools iPhone SE)
- Test on 375px viewport (iPhone 12/13/14)
- Verify Drawer slides up from bottom on mobile
- Verify Dialog still works on desktop (≥768px)

## What We're NOT Doing

- Changing the table to a card layout
- Hiding columns on mobile
- Changing the admin dashboard table
- Modifying the `ReceiptDetailsCard` component itself

## Implementation Approach

Use responsive Tailwind classes and the existing `useMobile()` hook to:
1. Reduce table padding and text size on mobile
2. Use abbreviated date format on mobile
3. Remove "Actions" header text
4. Show icon-only for Photo column on mobile
5. Conditionally render Drawer vs Dialog based on screen size

---

## Phase 1: Add Mobile Date Formatting Utility

### Overview
Add a short date format function for mobile displays.

### Changes Required:

#### 1. Add formatDateShort utility
**File**: `dws-app/src/lib/utils.ts`
**Changes**: Add a new function for short mobile date format

```typescript
export function formatDateShort(dateString: string | null | undefined): string {
  if (!dateString) return "N/A"
  try {
    const date = new Date(dateString)
    // Format as M/D (e.g., "12/18")
    return `${date.getMonth() + 1}/${date.getDate()}`
  } catch {
    return "N/A"
  }
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd dws-app && npm run build`
- [x] Linting passes: `cd dws-app && npm run lint` (pre-existing lint errors in other files)

#### Manual Verification:
- [x] N/A for this phase (utility function only)

---

## Phase 2: Optimize Table for Mobile

### Overview
Make the receipts table fit on small screens by reducing padding, using smaller text, and abbreviating content on mobile.

### Changes Required:

#### 1. Update EmployeeReceiptTable component
**File**: `dws-app/src/components/employee-receipt-table.tsx`
**Changes**:
- Import `useMobile` hook and `formatDateShort` utility
- Add responsive classes to table cells
- Remove "Actions" header text
- Conditionally format date and Photo column content

```typescript
// Add imports at top
import { useMobile } from "@/hooks/use-mobile"
import { formatCurrency, formatDate, formatDateShort } from "@/lib/utils"

// Inside component, add hook
const isMobile = useMobile()

// Update TableHeader - remove "Actions" text
<TableHeader className="bg-[#3e3e3e] hover:bg-[#3e3e3e]">
  <TableRow className="border-[#4e4e4e]">
    <TableHead className="text-white text-xs sm:text-sm px-1.5 sm:px-2">Date</TableHead>
    <TableHead className="text-white text-xs sm:text-sm px-1.5 sm:px-2">Amount</TableHead>
    <TableHead className="text-white text-xs sm:text-sm px-1.5 sm:px-2">Status</TableHead>
    <TableHead className="text-white text-xs sm:text-sm px-1.5 sm:px-2">Photo</TableHead>
    <TableHead className="text-white px-1.5 sm:px-2"><span className="sr-only">Actions</span></TableHead>
  </TableRow>
</TableHeader>

// Update TableCell for Date - use short format on mobile
<TableCell className="text-xs sm:text-sm px-1.5 sm:px-2">
  {receipt.date ? (isMobile ? formatDateShort(receipt.date) : formatDate(receipt.date)) : 'N/A'}
</TableCell>

// Update TableCell for Amount
<TableCell className="text-xs sm:text-sm px-1.5 sm:px-2">{formatCurrency(receipt.amount)}</TableCell>

// Update TableCell for Status
<TableCell className="px-1.5 sm:px-2"><StatusBadge status={receipt.status} /></TableCell>

// Update TableCell for Photo - icon only on mobile
<TableCell className="px-1.5 sm:px-2">
  {receipt.image_url ? (
    <a
      href={receipt.image_url}
      target="_blank"
      rel="noopener noreferrer"
      className="flex items-center text-blue-400 hover:text-blue-300 transition-colors"
      aria-label={`View receipt photo for ${receipt.date ? formatDate(receipt.date) : 'this receipt'}`}
    >
      {!isMobile && <span className="mr-1">View</span>}
      <ExternalLink size={isMobile ? 14 : 14} />
    </a>
  ) : (
    <span className="text-gray-500 text-xs sm:text-sm">{isMobile ? "—" : "No photo"}</span>
  )}
</TableCell>

// Update TableCell for Actions
<TableCell className="px-1.5 sm:px-2">
  <Button
    variant="ghost"
    size="sm"
    onClick={() => handleEditClick(receipt)}
    className="text-blue-400 hover:text-blue-300 hover:bg-[#4e4e4e] p-1.5 sm:p-2"
    aria-label={`Edit receipt from ${receipt.date ? formatDate(receipt.date) : 'this date'}`}
  >
    <Pencil size={isMobile ? 14 : 16} />
  </Button>
</TableCell>
```

#### 2. Update StatusBadge for mobile
**File**: `dws-app/src/components/employee-receipt-table.tsx`
**Changes**: Make badge more compact on mobile

```typescript
// Update StatusBadge component to accept isMobile prop or use hook internally
function StatusBadge({ status }: { status: Receipt["status"] }) {
  const normalizedStatus = status.toLowerCase()

  const variants: Record<string, string> = {
    pending: "bg-yellow-900/30 text-yellow-300 hover:bg-yellow-900/30 border-yellow-700",
    approved: "bg-green-900/30 text-green-300 hover:bg-green-900/30 border-green-700",
    rejected: "bg-red-900/30 text-red-300 hover:bg-red-900/30 border-red-700",
    reimbursed: "bg-blue-900/30 text-blue-300 hover:bg-blue-900/30 border-blue-700",
  }

  const displayStatus = status.charAt(0).toUpperCase() + status.slice(1).toLowerCase()

  return (
    <Badge variant="outline" className={`${variants[normalizedStatus] || variants.pending} text-[10px] sm:text-xs px-1.5 sm:px-2 py-0.5`}>
      {displayStatus}
    </Badge>
  )
}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd dws-app && npm run build`
- [x] Linting passes: `cd dws-app && npm run lint` (pre-existing lint errors in other files)

#### Manual Verification:
- [x] Table fits on 320px viewport without horizontal scroll
- [x] Table fits on 375px viewport without horizontal scroll
- [x] Date shows as "12/18" format on mobile, "Dec 18, 2024" on desktop
- [x] Photo column shows icon only on mobile, "View" + icon on desktop
- [x] "No photo" shows as "—" on mobile
- [x] Actions column header is visually empty (sr-only text for accessibility)
- [x] Table still looks good on desktop

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Use Drawer for Edit Modal on Mobile

### Overview
Replace the Dialog with Drawer on mobile for the edit receipt modal in the employee table.

### Changes Required:

#### 1. Update EmployeeReceiptTable to use Drawer on mobile
**File**: `dws-app/src/components/employee-receipt-table.tsx`
**Changes**:
- Import Drawer components
- Conditionally render Drawer vs Dialog based on `isMobile`

```typescript
// Add Drawer imports
import {
  Drawer,
  DrawerContent,
  DrawerTitle,
} from "@/components/ui/drawer"

// Replace the Edit Receipt Dialog section (lines 225-251) with:
{/* Edit Receipt - Drawer on mobile, Dialog on desktop */}
{isMobile ? (
  <Drawer open={editDialogOpen} onOpenChange={setEditDialogOpen}>
    <DrawerContent className="bg-[#2e2e2e] border-[#4e4e4e]">
      <DrawerTitle className="sr-only">Edit Receipt</DrawerTitle>
      <div className="px-4 pb-4">
        {selectedReceipt && (
          <ReceiptDetailsCard
            mode="edit"
            receiptId={selectedReceipt.id}
            initialData={{
              receipt_date: selectedReceipt.date,
              amount: selectedReceipt.amount,
              category_id: selectedReceipt.category_id,
              notes: selectedReceipt.notes || selectedReceipt.description,
            }}
            onSubmit={() => {}}
            onCancel={() => setEditDialogOpen(false)}
            onEditSuccess={handleEditSuccess}
            onDelete={() => {
              setEditDialogOpen(false)
              setSelectedReceipt(null)
              onReceiptUpdated?.({} as Receipt)
            }}
          />
        )}
      </div>
    </DrawerContent>
  </Drawer>
) : (
  <Dialog open={editDialogOpen} onOpenChange={setEditDialogOpen}>
    <DialogContent className="bg-transparent border-none p-0 max-w-md">
      <DialogTitle className="sr-only">Edit Receipt</DialogTitle>
      {selectedReceipt && (
        <ReceiptDetailsCard
          mode="edit"
          receiptId={selectedReceipt.id}
          initialData={{
            receipt_date: selectedReceipt.date,
            amount: selectedReceipt.amount,
            category_id: selectedReceipt.category_id,
            notes: selectedReceipt.notes || selectedReceipt.description,
          }}
          onSubmit={() => {}}
          onCancel={() => setEditDialogOpen(false)}
          onEditSuccess={handleEditSuccess}
          onDelete={() => {
            setEditDialogOpen(false)
            setSelectedReceipt(null)
            onReceiptUpdated?.({} as Receipt)
          }}
        />
      )}
    </DialogContent>
  </Dialog>
)}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd dws-app && npm run build`
- [x] Linting passes: `cd dws-app && npm run lint` (pre-existing lint errors in other files)

#### Manual Verification:
- [x] On mobile (<768px): Edit button opens Drawer sliding up from bottom
- [x] Drawer has drag handle at top
- [x] Can swipe down to dismiss Drawer
- [x] Edit form works correctly inside Drawer
- [x] On desktop (≥768px): Edit button opens centered Dialog (unchanged behavior)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Use Drawer for Upload Confirmation on Mobile

### Overview
Replace the Dialog with Drawer on mobile for the receipt details fallback modal. Note: Since the auto-submit feature was added, this Dialog only appears when OCR couldn't extract all fields or a potential duplicate is detected. Most uploads auto-submit and show a toast with an "Edit" button instead.

### Changes Required:

#### 1. Update ReceiptUploader to use Drawer on mobile
**File**: `dws-app/src/components/receipt-uploader.tsx`
**Changes**:
- Import Drawer components
- Conditionally render Drawer vs Dialog based on `isMobile` (already imported)

```typescript
// Add Drawer imports (near other imports)
import {
  Drawer,
  DrawerContent,
  DrawerTitle,
} from "@/components/ui/drawer"

// Replace the Dialog section (lines 453-474) with:
{/* Receipt Details - Drawer on mobile, Dialog on desktop */}
{isMobile ? (
  <Drawer open={showDetailsCard} onOpenChange={setShowDetailsCard}>
    <DrawerContent className="bg-[#2e2e2e] border-[#4e4e4e]">
      <DrawerTitle className="sr-only">
        {extractedData?.id ? 'Edit Receipt Details' : 'Confirm Receipt Details'}
      </DrawerTitle>
      <div className="px-4 pb-4">
        <ReceiptDetailsCard
          onSubmit={handleDetailsSubmit}
          onCancel={handleCancel}
          initialData={extractedData}
          mode={extractedData?.id ? 'edit' : 'create'}
          receiptId={extractedData?.id as string | undefined}
          onEditSuccess={(updatedReceipt) => {
            if (onReceiptAdded) {
              onReceiptAdded(updatedReceipt)
            }
            setShowDetailsCard(false)
            setExtractedData({})
          }}
        />
      </div>
    </DrawerContent>
  </Drawer>
) : (
  <Dialog open={showDetailsCard} onOpenChange={setShowDetailsCard}>
    <DialogContent className="sm:max-w-md bg-[#2e2e2e] p-0 border-none">
      <ReceiptDetailsCard
        onSubmit={handleDetailsSubmit}
        onCancel={handleCancel}
        initialData={extractedData}
        mode={extractedData?.id ? 'edit' : 'create'}
        receiptId={extractedData?.id as string | undefined}
        onEditSuccess={(updatedReceipt) => {
          if (onReceiptAdded) {
            onReceiptAdded(updatedReceipt)
          }
          setShowDetailsCard(false)
          setExtractedData({})
        }}
      />
      <DialogTitle className="sr-only">
        {extractedData?.id ? 'Edit Receipt Details' : 'Confirm Receipt Details'}
      </DialogTitle>
    </DialogContent>
  </Dialog>
)}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles without errors: `cd dws-app && npm run build`
- [x] Linting passes: `cd dws-app && npm run lint`

#### Manual Verification:
- [ ] On mobile (fallback case - missing fields): After taking photo where OCR can't extract all fields, Drawer slides up from bottom
- [ ] On mobile (fallback case - duplicate): When duplicate detected, Drawer slides up with warning
- [ ] On mobile (edit via toast): After auto-submit, tap "Edit" in toast → Drawer opens in edit mode
- [ ] OCR-extracted data populates correctly in Drawer form
- [ ] Can submit receipt from Drawer
- [ ] Can cancel/dismiss by swiping down
- [ ] On desktop: Dialog still works as before (both fallback and edit-via-toast scenarios)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Testing Strategy

### Unit Tests:
- N/A - primarily UI/layout changes

### Integration Tests:
- N/A - primarily UI/layout changes

### Manual Testing Steps:

**Desktop Testing (≥768px viewport):**
1. Verify table displays normally with full date format and "View" text
2. Verify edit modal opens as centered Dialog
3. Verify auto-submit shows toast with "Edit" button (for clear receipts)
4. Verify fallback Dialog opens when OCR can't extract all fields

**Mobile Testing (320px viewport - iPhone SE):**
1. Open employee page
2. Verify table fits without horizontal scroll
3. Verify dates show in short format (M/D)
4. Verify Photo column shows icon only
5. Verify Actions header is empty
6. Tap edit icon on pending receipt - verify Drawer slides up from bottom
7. Test editing a receipt via Drawer
8. Tap "Take Photo" button and select clear receipt image
9. Verify auto-submit: toast appears with "Edit" button (if all fields extracted)
10. Tap "Edit" in toast - verify Drawer opens in edit mode
11. Test fallback: upload blurry/partial receipt image
12. Verify fallback Drawer appears when OCR can't extract all fields

**Mobile Testing (375px viewport - iPhone 12/13/14):**
1. Repeat steps 1-12
2. Verify comfortable spacing (table should have more breathing room)

**Tablet Testing (768px viewport):**
1. Verify it uses desktop behavior (Dialog, not Drawer)
2. Table should show full formatting

## Performance Considerations

- `useMobile()` hook uses `window.matchMedia` which is efficient
- No additional API calls or data fetching changes
- Drawer component is already installed (vaul library)

## Migration Notes

- No database changes
- No API changes
- Purely frontend UI changes
- Backwards compatible - desktop behavior unchanged

## References

- Research document: `thoughts/shared/research/2025-12-18-employee-page-mobile-layout.md`
- Employee table: `dws-app/src/components/employee-receipt-table.tsx`
- Receipt uploader: `dws-app/src/components/receipt-uploader.tsx`
- Drawer component: `dws-app/src/components/ui/drawer.tsx`
- useMobile hook: `dws-app/src/hooks/use-mobile.tsx`
