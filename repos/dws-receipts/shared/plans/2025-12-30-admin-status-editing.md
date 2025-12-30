# Admin Receipt Status Editing Implementation Plan

## Overview

Enable admin users to edit receipt status directly from the edit dialog, providing the same flexibility they have with other receipt fields (date, amount, category, notes).

## Current State Analysis

### Current Edit Capabilities
- **Editable fields**: Date, Amount, Category, Notes/Description
- **Not editable via form**: Status (only changeable via batch review or bulk reimburse)
- **Component**: `ReceiptDetailsCard` is shared between employee and admin contexts

### Key Discoveries:
- `dws-app/src/components/receipt-details-card.tsx:284-359` - Edit form fields (no status)
- `dws-app/src/app/api/receipts/route.ts:272` - PATCH accepts `id, receipt_date, amount, category_id, notes` but NOT `status`
- `dws-app/src/components/receipt-dashboard.tsx:871-887` - Admin edit dialog passes initial data but not status
- `dws-app/src/lib/types.ts:10` - Status type includes both cases: `"Pending" | "Approved" | "Rejected" | "Reimbursed" | "pending" | "approved" | "rejected" | "reimbursed"`

### Status Conventions:
- **Database**: Stores capitalized values (`"Pending"`, `"Approved"`, `"Rejected"`, `"Reimbursed"`)
- **Frontend display**: Normalized to lowercase
- **When writing to DB**: Must capitalize

## Desired End State

After implementation:
1. Admin users opening the edit dialog will see a "Status" dropdown below the existing fields
2. The dropdown will show all four status options: Pending, Approved, Rejected, Reimbursed
3. Saving will update the status in the database
4. Employees will NOT see the status dropdown (their edit form remains unchanged)

### Verification:
1. Admin edits a receipt → sees Status dropdown → can change status → saves successfully
2. Employee edits a receipt → does NOT see Status dropdown
3. Status changes persist correctly in the database (capitalized)

## What We're NOT Doing

- Adding status editing to batch review (already has approve/reject buttons)
- Adding status editing for employees (they should not control their own receipt status)
- Changing the bulk reimburse functionality
- Adding status history/audit logging (could be a future enhancement)

## Implementation Approach

Since `ReceiptDetailsCard` is shared between employee and admin contexts, we'll add an `isAdmin` prop to conditionally show the status field. The API already has admin detection, so we just need to add `status` to the accepted fields.

---

## Phase 1: API Update

### Overview
Add `status` as an accepted parameter in the PATCH `/api/receipts` endpoint.

### Changes Required:

#### 1. PATCH Endpoint
**File**: `dws-app/src/app/api/receipts/route.ts`

**Change 1**: Add `status` to destructured request body (line 272)
```typescript
// Before:
const { id, receipt_date, amount, category_id, notes } = body;

// After:
const { id, receipt_date, amount, category_id, notes, status } = body;
```

**Change 2**: Update the validation check to include `status` (lines 278-280)
```typescript
// Before:
if (receipt_date === undefined && amount === undefined && category_id === undefined && notes === undefined) {
  return NextResponse.json({ error: 'At least one field to update is required' }, { status: 400 });
}

// After:
if (receipt_date === undefined && amount === undefined && category_id === undefined && notes === undefined && status === undefined) {
  return NextResponse.json({ error: 'At least one field to update is required' }, { status: 400 });
}
```

**Change 3**: Add status to updateData object (after line 338)
```typescript
if (notes !== undefined) updateData.description = notes;
// Add this line:
if (status !== undefined) {
  // Validate status value
  const validStatuses = ['Pending', 'Approved', 'Rejected', 'Reimbursed'];
  const capitalizedStatus = status.charAt(0).toUpperCase() + status.slice(1).toLowerCase();
  if (!validStatuses.includes(capitalizedStatus)) {
    return NextResponse.json({ error: `Invalid status value. Must be one of: ${validStatuses.join(', ')}` }, { status: 400 });
  }
  updateData.status = capitalizedStatus;
}
```

### Success Criteria:

#### Automated Verification:
- [x] Build passes: `cd dws-app && npm run build`
- [x] Lint passes: `cd dws-app && npm run lint` (pre-existing lint warnings unrelated to changes)
- [x] Type checking passes (implicit in build)

#### Manual Verification:
- [ ] API accepts status parameter and updates database correctly
- [ ] Invalid status values are rejected with 400 error
- [ ] Status is properly capitalized when stored

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the API changes work correctly before proceeding to Phase 2.

---

## Phase 2: Frontend Update

### Overview
Add a status dropdown to `ReceiptDetailsCard` that only appears for admin users in edit mode.

### Changes Required:

#### 1. ReceiptDetailsCard Props Interface
**File**: `dws-app/src/components/receipt-details-card.tsx`

**Change 1**: Add `isAdmin` prop and `status` to initialData type (around line 29)
```typescript
interface ReceiptDetailsCardProps {
  onSubmit: (receiptData: Partial<Receipt>) => void
  onCancel: () => void
  initialData?: Partial<Receipt> & { status?: string } // Add status to initialData
  mode?: 'create' | 'edit'
  receiptId?: string
  onEditSuccess?: (updatedReceipt: Receipt) => void
  onDelete?: () => void
  allowDelete?: boolean
  isAdmin?: boolean // Add this prop
}
```

**Change 2**: Destructure `isAdmin` from props (around line 40)
```typescript
export function ReceiptDetailsCard({
  onSubmit,
  onCancel,
  initialData,
  mode = 'create',
  receiptId,
  onEditSuccess,
  onDelete,
  allowDelete = true,
  isAdmin = false, // Add with default false
}: ReceiptDetailsCardProps) {
```

**Change 3**: Add status state (after line 76)
```typescript
const [notes, setNotes] = useState(initialData?.notes || "");
// Add status state:
const [status, setStatus] = useState<string>(initialData?.status || "pending");
```

**Change 4**: Add status to form submission (inside handleSubmit, around line 134-139)
```typescript
const currentReceiptData = {
  receipt_date: format(date, "yyyy-MM-dd"),
  amount: Number.parseFloat(amount),
  category_id: categoryId,
  notes: notes.trim(),
  ...(isAdmin && isEditMode && { status }), // Only include status for admin edits
};
```

**Change 5**: Add status dropdown UI (after the Notes field, around line 359)
```typescript
            </div>
            {/* Status dropdown - only shown for admin in edit mode */}
            {isAdmin && isEditMode && (
              <div className="flex flex-col space-y-1.5">
                <Label htmlFor="status" className="text-white">
                  Status
                </Label>
                <Select value={status} onValueChange={setStatus}>
                  <SelectTrigger id="status" className="w-full bg-[#3e3e3e] border-[#3e3e3e] text-white">
                    <SelectValue placeholder="Select status" />
                  </SelectTrigger>
                  <SelectContent position="popper" className="bg-[#2e2e2e] text-white border-[#4e4e4e]">
                    <SelectItem value="pending" className="hover:bg-[#4e4e4e]">Pending</SelectItem>
                    <SelectItem value="approved" className="hover:bg-[#4e4e4e]">Approved</SelectItem>
                    <SelectItem value="rejected" className="hover:bg-[#4e4e4e]">Rejected</SelectItem>
                    <SelectItem value="reimbursed" className="hover:bg-[#4e4e4e]">Reimbursed</SelectItem>
                  </SelectContent>
                </Select>
              </div>
            )}
          </div>
```

#### 2. Admin Dashboard - Pass isAdmin and status
**File**: `dws-app/src/components/receipt-dashboard.tsx`

**Change**: Update ReceiptDetailsCard call (around lines 871-887)
```typescript
<ReceiptDetailsCard
  mode="edit"
  receiptId={editingReceipt.id}
  initialData={{
    receipt_date: editingReceipt.date,
    amount: editingReceipt.amount,
    category_id: editingReceipt.category_id,
    notes: editingReceipt.notes || editingReceipt.description,
    status: editingReceipt.status, // Add status
  }}
  onSubmit={() => {}}
  onCancel={() => setEditingReceipt(null)}
  onEditSuccess={handleEditSuccess}
  onDelete={() => {
    setEditingReceipt(null);
    invalidateReceipts();
  }}
  isAdmin={true} // Add isAdmin prop
/>
```

### Success Criteria:

#### Automated Verification:
- [ ] Build passes: `cd dws-app && npm run build`
- [ ] Lint passes: `cd dws-app && npm run lint`
- [ ] Type checking passes (implicit in build)

#### Manual Verification:
- [ ] Admin dashboard: Edit dialog shows Status dropdown below Notes
- [ ] Admin can change status from any value to any other value
- [ ] Status change persists after save (verify in database or by refreshing)
- [ ] Employee page: Edit dialog does NOT show Status dropdown
- [ ] Status dropdown shows correct current value when editing

**Implementation Note**: After completing this phase, verify both admin and employee flows work correctly.

---

## Testing Strategy

### Manual Testing Steps:
1. **Admin status edit flow**:
   - Log in as admin
   - Go to dashboard
   - Click edit on a receipt
   - Verify Status dropdown appears with correct current value
   - Change status to each option and save
   - Verify status updates correctly each time

2. **Employee edit flow (no status)**:
   - Log in as employee
   - Go to employee page
   - Click edit on a receipt
   - Verify Status dropdown does NOT appear
   - Edit other fields and save
   - Verify status is unchanged

3. **Status validation**:
   - Use API directly (curl/Postman) to send invalid status
   - Verify 400 error response

## References

- Research document: `thoughts/shared/research/2025-12-30-receipt-status-and-dates.md`
- Current batch review: `dws-app/src/components/batch-review-dashboard.tsx:98-131`
- Status type definition: `dws-app/src/lib/types.ts:10`
