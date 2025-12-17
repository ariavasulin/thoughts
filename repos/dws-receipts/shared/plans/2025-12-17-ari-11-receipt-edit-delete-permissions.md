# Implementation Plan: ARI-11 Receipt Edit/Delete Permissions

**Linear Ticket**: [ARI-11](https://linear.app/ariav/issue/ARI-11/create-editing-approval-logic-for-receipts)
**Date**: 2025-12-17
**Status**: Ready for Implementation
**Branch**: `receipt-editing`

## Overview

Implement edit and delete permissions for receipts based on status and user role. Employees can modify their own pending receipts freely. Once a receipt is processed (approved/rejected/reimbursed), only admins can make changes.

## Requirements

### Permission Matrix

| Receipt Status | Employee (owner) | Admin |
|----------------|------------------|-------|
| **Pending** | Edit, Delete | Edit, Delete |
| **Approved** | No access | Edit, Delete |
| **Rejected** | No access | Edit, Delete |
| **Reimbursed** | No access | Edit, Delete |

### User Experience
- Employees see edit button for all their receipts
- Clicking edit on a pending receipt opens the edit dialog with delete option
- Clicking edit on a processed receipt shows "Contact admin" popup
- Admins see edit/delete in dashboard for all receipts regardless of status

## Current State

### What Exists
- **PATCH endpoint** (`dws-app/src/app/api/receipts/route.ts:246-391`):
  - Allows editing `receipt_date`, `amount`, `category_id`, `notes`
  - Ownership check: user must own OR be admin (lines 296-308)
  - Status restriction: only "Pending" receipts (lines 310-316) - **applies to both employees AND admins**

- **Employee edit UI** (`dws-app/src/components/employee-receipt-table.tsx:131-144`):
  - Edit button (pencil icon) only shown for pending receipts
  - Opens `ReceiptDetailsCard` in edit mode

- **ReceiptDetailsCard** (`dws-app/src/components/receipt-details-card.tsx`):
  - Supports `mode="edit"` with `receiptId` prop
  - Has Cancel and Save/Submit buttons in footer
  - No delete functionality currently

### What's Missing
1. **No DELETE endpoint** - delete functionality doesn't exist
2. **Admin status restriction** - admins cannot edit approved/rejected/reimbursed receipts (should be able to)
3. **No delete UI** - neither employee nor admin side
4. **No admin edit UI** - dashboard has no edit button
5. **No "contact admin" popup** for processed receipts on employee side

---

## Implementation Tasks

### Phase 1: Backend API Changes

#### Task 1.1: Add DELETE endpoint to `/api/receipts`

**File**: `dws-app/src/app/api/receipts/route.ts`

Add new DELETE handler after PATCH:

```typescript
export async function DELETE(request: Request) {
  const supabase = await createSupabaseServerClient();

  const { data: { session } } = await supabase.auth.getSession();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const userId = session.user.id;
  const { searchParams } = new URL(request.url);
  const receiptId = searchParams.get('id');

  if (!receiptId) {
    return NextResponse.json({ error: 'Receipt ID is required' }, { status: 400 });
  }

  // Fetch receipt to check ownership and status
  const { data: receipt, error: fetchError } = await supabase
    .from('receipts')
    .select('id, user_id, status, image_url')
    .eq('id', receiptId)
    .single();

  if (fetchError || !receipt) {
    return NextResponse.json({ error: 'Receipt not found' }, { status: 404 });
  }

  // Check permissions
  const isOwner = receipt.user_id === userId;
  const isPending = receipt.status.toLowerCase() === 'pending';

  // Check if admin
  let isAdmin = false;
  if (!isOwner || !isPending) {
    const { data: profile } = await supabase
      .from('user_profiles')
      .select('role')
      .eq('user_id', userId)
      .single();
    isAdmin = profile?.role === 'admin';
  }

  // Employees can only delete their own pending receipts
  // Admins can delete any receipt
  if (!isAdmin && (!isOwner || !isPending)) {
    if (!isOwner) {
      return NextResponse.json({ error: 'You can only delete your own receipts' }, { status: 403 });
    }
    return NextResponse.json({
      error: `Cannot delete receipt with status "${receipt.status}". Contact an admin.`
    }, { status: 403 });
  }

  // Delete the image from storage first
  if (receipt.image_url) {
    const { error: storageError } = await supabase.storage
      .from('receipt-images')
      .remove([receipt.image_url]);
    if (storageError) {
      console.error('Error deleting receipt image:', storageError);
      // Continue with deletion even if image removal fails
    }
  }

  // Delete the receipt record
  const { error: deleteError } = await supabase
    .from('receipts')
    .delete()
    .eq('id', receiptId);

  if (deleteError) {
    return NextResponse.json({ error: 'Failed to delete receipt' }, { status: 500 });
  }

  return NextResponse.json({ success: true }, { status: 200 });
}
```

#### Task 1.2: Modify PATCH endpoint for admin access

**File**: `dws-app/src/app/api/receipts/route.ts`

Change the status check (lines 310-316) to only apply to non-admins:

**Current code:**
```typescript
// Only allow editing receipts with "Pending" status
if (existingReceipt.status.toLowerCase() !== 'pending') {
  return NextResponse.json({
    error: `Cannot edit receipt with status "${existingReceipt.status}". Only pending receipts can be edited.`
  }, { status: 400 });
}
```

**New code:**
```typescript
// Check if user is admin (we may have already fetched this)
let userIsAdmin = false;
if (existingReceipt.user_id !== userId) {
  // Already checked admin status above, user is admin if we got here
  userIsAdmin = true;
} else {
  // Owner - need to check if also admin
  const { data: profile } = await supabase
    .from('user_profiles')
    .select('role')
    .eq('user_id', userId)
    .single();
  userIsAdmin = profile?.role === 'admin';
}

// Only allow editing non-pending receipts if user is admin
if (existingReceipt.status.toLowerCase() !== 'pending' && !userIsAdmin) {
  return NextResponse.json({
    error: `Cannot edit receipt with status "${existingReceipt.status}". Contact an admin.`
  }, { status: 403 });
}
```

**Note**: Refactor the admin check to avoid duplicate queries - move the admin check earlier and reuse the result.

---

### Phase 2: Employee UI Changes

#### Task 2.1: Show edit button for ALL receipts (not just pending)

**File**: `dws-app/src/components/employee-receipt-table.tsx`

**Current behavior** (lines 131-143):
- Edit button only appears for pending receipts
- Non-pending receipts show a dash "-"

**New behavior**:
- Edit button appears for ALL receipts
- Clicking opens either the edit dialog (pending) or contact admin popup (processed)

**Current code:**
```tsx
{receipt.status.toLowerCase() === 'pending' ? (
  <Button variant="ghost" size="sm" onClick={() => handleEditClick(receipt)}>
    <Pencil size={16} />
  </Button>
) : (
  <span className="text-gray-500 text-xs">-</span>
)}
```

**New code:**
```tsx
<Button
  variant="ghost"
  size="sm"
  onClick={() => handleEditClick(receipt)}
  className="text-blue-400 hover:text-blue-300 hover:bg-[#4e4e4e] p-2"
>
  <Pencil size={16} />
</Button>
```

#### Task 2.2: Add "Contact Admin" alert dialog for processed receipts

**File**: `dws-app/src/components/employee-receipt-table.tsx`

Add new state and dialog:

```tsx
// Add state
const [contactAdminReceipt, setContactAdminReceipt] = useState<Receipt | null>(null);

// Modify handleEditClick
const handleEditClick = (receipt: Receipt) => {
  if (receipt.status.toLowerCase() === 'pending') {
    setSelectedReceipt(receipt);
    setEditDialogOpen(true);
  } else {
    // Show contact admin dialog for processed receipts
    setContactAdminReceipt(receipt);
  }
};
```

**UI Wireframe - Contact Admin Dialog:**
```
┌────────────────────────────────────────────────────┐
│                Cannot Edit Receipt                 │
│────────────────────────────────────────────────────│
│                                                    │
│   This receipt has been [Approved/Rejected/        │
│   Reimbursed] and cannot be modified.              │
│                                                    │
│   Please contact your system administrator         │
│   if you need to make changes to this receipt.     │
│                                                    │
│                              ┌─────────────────┐   │
│                              │       OK        │   │
│                              └─────────────────┘   │
└────────────────────────────────────────────────────┘
```

**Implementation:**
```tsx
{/* Contact Admin Dialog */}
<AlertDialog open={!!contactAdminReceipt} onOpenChange={(open) => !open && setContactAdminReceipt(null)}>
  <AlertDialogContent className="bg-[#333333] border-[#444444]">
    <AlertDialogHeader>
      <AlertDialogTitle className="text-white">Cannot Edit Receipt</AlertDialogTitle>
      <AlertDialogDescription className="text-gray-300">
        This receipt has been <span className="font-semibold capitalize">{contactAdminReceipt?.status}</span> and cannot be modified.
        <br /><br />
        Please contact your system administrator if you need to make changes to this receipt.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogAction className="bg-[#2680FC] hover:bg-[#1a6cd9]">
        OK
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

#### Task 2.3: Add delete button INSIDE the edit dialog (ReceiptDetailsCard)

**File**: `dws-app/src/components/receipt-details-card.tsx`

Add delete functionality to the edit dialog footer:

**Current footer (lines 311-318):**
```tsx
<CardFooter className="flex justify-between">
  <Button variant="outline" onClick={onCancel} disabled={isCheckingDuplicate}>
    Cancel
  </Button>
  <Button type="submit" form="receipt-form" disabled={isCheckingDuplicate}>
    {isCheckingDuplicate ? (isEditMode ? "Saving..." : "Checking...") :
     (isEditMode ? "Save Changes" : "Submit Receipt")}
  </Button>
</CardFooter>
```

**New footer with delete:**

**UI Wireframe - Edit Dialog with Delete:**
```
┌─────────────────────────────────────────────────────┐
│                   Edit Receipt                      │
│─────────────────────────────────────────────────────│
│                                                     │
│  Date:        [  2024-01-15  ]                      │
│                                                     │
│  Amount:      [$  125.50     ]                      │
│                                                     │
│  Category:    [ Meals & Entertainment  ▼]           │
│                                                     │
│  Notes:       [ Team lunch at restaurant ]          │
│                                                     │
│─────────────────────────────────────────────────────│
│  ┌────────────────┐                                 │
│  │ Delete Receipt │   ┌────────┐  ┌──────────────┐  │
│  └────────────────┘   │ Cancel │  │ Save Changes │  │
│      (red/danger)     └────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Props changes:**
```typescript
interface ReceiptDetailsCardProps {
  onSubmit: (receiptData: Partial<Receipt>) => void
  onCancel: () => void
  initialData?: Partial<Receipt>
  mode?: 'create' | 'edit'
  receiptId?: string
  onEditSuccess?: (updatedReceipt: Receipt) => void
  onDelete?: () => void  // NEW: callback when delete succeeds
  allowDelete?: boolean   // NEW: whether to show delete button (default: true in edit mode)
}
```

**Delete implementation:**
```tsx
// Add state
const [isDeleting, setIsDeleting] = useState(false);
const [showDeleteConfirm, setShowDeleteConfirm] = useState(false);

// Delete handler
const handleDelete = async () => {
  if (!receiptId) return;

  setIsDeleting(true);
  try {
    const response = await fetch(`/api/receipts?id=${receiptId}`, {
      method: 'DELETE',
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error || 'Failed to delete receipt');
    }

    sonnerToast.success("Receipt Deleted", {
      description: "The receipt has been permanently deleted."
    });

    onDelete?.();
    onCancel();
  } catch (error) {
    sonnerToast.error("Delete Failed", {
      description: error instanceof Error ? error.message : "Failed to delete receipt"
    });
  } finally {
    setIsDeleting(false);
    setShowDeleteConfirm(false);
  }
};
```

**Delete confirmation dialog:**
```
┌────────────────────────────────────────────────────┐
│               Delete Receipt?                      │
│────────────────────────────────────────────────────│
│                                                    │
│   Are you sure you want to delete this receipt?    │
│   This action cannot be undone.                    │
│                                                    │
│        ┌──────────┐    ┌───────────────────┐       │
│        │  Cancel  │    │  Delete Receipt   │       │
│        └──────────┘    └───────────────────┘       │
│                              (red/danger)          │
└────────────────────────────────────────────────────┘
```

**New CardFooter:**
```tsx
<CardFooter className="flex justify-between">
  <div className="flex gap-2">
    {isEditMode && allowDelete !== false && (
      <Button
        type="button"
        variant="destructive"
        onClick={() => setShowDeleteConfirm(true)}
        disabled={isCheckingDuplicate || isDeleting}
        className="bg-red-600 hover:bg-red-700"
      >
        <Trash2 className="h-4 w-4 mr-2" />
        Delete
      </Button>
    )}
  </div>
  <div className="flex gap-2">
    <Button variant="outline" onClick={onCancel} disabled={isCheckingDuplicate || isDeleting}>
      Cancel
    </Button>
    <Button type="submit" form="receipt-form" disabled={isCheckingDuplicate || isDeleting}>
      {isCheckingDuplicate ? (isEditMode ? "Saving..." : "Checking...") :
       (isEditMode ? "Save Changes" : "Submit Receipt")}
    </Button>
  </div>
</CardFooter>

{/* Delete Confirmation Dialog */}
<AlertDialog open={showDeleteConfirm} onOpenChange={setShowDeleteConfirm}>
  <AlertDialogContent className="bg-[#333333] border-[#444444]">
    <AlertDialogHeader>
      <AlertDialogTitle className="text-white">Delete Receipt?</AlertDialogTitle>
      <AlertDialogDescription className="text-gray-300">
        Are you sure you want to delete this receipt? This action cannot be undone.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel disabled={isDeleting}>Cancel</AlertDialogCancel>
      <AlertDialogAction
        onClick={handleDelete}
        disabled={isDeleting}
        className="bg-red-600 hover:bg-red-700"
      >
        {isDeleting ? "Deleting..." : "Delete Receipt"}
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

#### Task 2.4: Update employee-receipt-table to handle delete callback

**File**: `dws-app/src/components/employee-receipt-table.tsx`

```tsx
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
    // Close dialog and refresh list after delete
    setEditDialogOpen(false);
    setSelectedReceipt(null);
    onReceiptUpdated?.();
  }}
/>
```

---

### Phase 3: Admin Dashboard UI Changes

#### Task 3.1: Add Actions column to receipt-table.tsx

**File**: `dws-app/src/components/receipt-table.tsx`

**Current columns**: Checkbox, Date, Employee, Phone, Amount, Category, Description, Status, Image

**New columns**: Checkbox, Date, Employee, Phone, Amount, Category, Description, Status, Image, **Actions**

**UI Wireframe - Admin Table Row:**
```
┌────┬────────────┬──────────┬──────────────┬─────────┬──────────┬─────────────┬────────────┬───────┬─────────────┐
│ ☐  │ 2024-01-15 │ John Doe │ 555-123-4567 │ $125.50 │ Meals    │ Team lunch  │ [Approved] │ View  │ [⚙️ ▼]      │
└────┴────────────┴──────────┴──────────────┴─────────┴──────────┴─────────────┴────────────┴───────┴─────────────┘
                                                                                                    ↑
                                                                                            Actions dropdown
```

**Actions Dropdown Menu:**
```
┌─────────────────┐
│  ✏️  Edit       │
│─────────────────│
│  🗑️  Delete     │  (red text)
└─────────────────┘
```

**Props changes:**
```typescript
interface ReceiptTableProps {
  rowData?: Receipt[]
  height?: number | string | "auto"
  selectedRows?: Set<string>
  onSelectedRowsChange?: (selectedRows: Set<string>) => void
  currentPage?: number
  pageSize?: number
  onPageChange?: (page: number) => void
  onPageSizeChange?: (pageSize: number) => void
  // NEW: Action callbacks
  onEdit?: (receipt: Receipt) => void
  onDelete?: (receipt: Receipt) => void
  showActions?: boolean  // default: true
}
```

**Implementation:**
```tsx
// Add imports
import { MoreHorizontal, Pencil, Trash2 } from 'lucide-react';
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";

// Add Actions column header (after Image column)
<TableHead className="text-white text-center p-3 w-20">
  Actions
</TableHead>

// Add Actions cell in row (after Image cell)
<TableCell className="text-center p-3">
  <DropdownMenu>
    <DropdownMenuTrigger asChild>
      <Button variant="ghost" size="sm" className="h-8 w-8 p-0">
        <MoreHorizontal className="h-4 w-4" />
      </Button>
    </DropdownMenuTrigger>
    <DropdownMenuContent align="end" className="bg-[#333333] border-[#444444]">
      <DropdownMenuItem
        onClick={() => onEdit?.(receipt)}
        className="text-white hover:bg-[#444444] cursor-pointer"
      >
        <Pencil className="mr-2 h-4 w-4" />
        Edit
      </DropdownMenuItem>
      <DropdownMenuItem
        onClick={() => onDelete?.(receipt)}
        className="text-red-400 hover:bg-[#444444] hover:text-red-300 cursor-pointer"
      >
        <Trash2 className="mr-2 h-4 w-4" />
        Delete
      </DropdownMenuItem>
    </DropdownMenuContent>
  </DropdownMenu>
</TableCell>
```

#### Task 3.2: Add edit dialog to receipt-dashboard.tsx

**File**: `dws-app/src/components/receipt-dashboard.tsx`

**Add state:**
```tsx
const [editingReceipt, setEditingReceipt] = useState<Receipt | null>(null);
const [deletingReceipt, setDeletingReceipt] = useState<Receipt | null>(null);
```

**Pass callbacks to ReceiptTable:**
```tsx
<ReceiptTable
  rowData={paginatedReceipts}
  height="auto"
  selectedRows={selectedRows}
  onSelectedRowsChange={setSelectedRows}
  currentPage={currentPage}
  pageSize={pageSize}
  onPageChange={setCurrentPage}
  onPageSizeChange={setPageSize}
  onEdit={(receipt) => setEditingReceipt(receipt)}
  onDelete={(receipt) => setDeletingReceipt(receipt)}
/>
```

**Edit Dialog (reuses ReceiptDetailsCard):**

**UI Wireframe - Admin Edit Dialog:**
```
┌─────────────────────────────────────────────────────┐
│                 Edit Receipt                        │
│         Employee: John Doe (EMP-123)                │
│         Status: Approved                            │
│─────────────────────────────────────────────────────│
│                                                     │
│  Date:        [  2024-01-15  ]                      │
│                                                     │
│  Amount:      [$  125.50     ]                      │
│                                                     │
│  Category:    [ Meals & Entertainment  ▼]           │
│                                                     │
│  Notes:       [ Team lunch at restaurant ]          │
│                                                     │
│─────────────────────────────────────────────────────│
│  ┌────────────────┐                                 │
│  │ Delete Receipt │   ┌────────┐  ┌──────────────┐  │
│  └────────────────┘   │ Cancel │  │ Save Changes │  │
│      (red/danger)     └────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Note**: Admin edit dialog shows employee name and current status as read-only info header.

```tsx
{/* Admin Edit Dialog */}
<Dialog open={!!editingReceipt} onOpenChange={(open) => !open && setEditingReceipt(null)}>
  <DialogContent className="bg-[#333333] border-[#444444] max-w-md">
    {editingReceipt && (
      <>
        {/* Header info - read only */}
        <div className="mb-4 p-3 bg-[#2a2a2a] rounded-lg">
          <p className="text-gray-300 text-sm">
            <span className="font-medium">Employee:</span> {editingReceipt.employeeName}
            {editingReceipt.employeeId && ` (${editingReceipt.employeeId})`}
          </p>
          <p className="text-gray-300 text-sm">
            <span className="font-medium">Status:</span>{' '}
            <span className={`capitalize ${
              editingReceipt.status === 'approved' ? 'text-green-400' :
              editingReceipt.status === 'rejected' ? 'text-red-400' :
              editingReceipt.status === 'reimbursed' ? 'text-blue-400' :
              'text-yellow-400'
            }`}>
              {editingReceipt.status}
            </span>
          </p>
        </div>

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
          onEditSuccess={(updatedReceipt) => {
            setEditingReceipt(null);
            fetchReceipts(); // Refresh the list
          }}
          onDelete={() => {
            setEditingReceipt(null);
            fetchReceipts(); // Refresh the list
          }}
        />
      </>
    )}
  </DialogContent>
</Dialog>
```

#### Task 3.3: Add delete confirmation dialog for admin

**File**: `dws-app/src/components/receipt-dashboard.tsx`

**UI Wireframe - Admin Delete Confirmation:**
```
┌────────────────────────────────────────────────────┐
│               Delete Receipt?                      │
│────────────────────────────────────────────────────│
│                                                    │
│   Are you sure you want to delete this receipt?    │
│                                                    │
│   Employee: John Doe                               │
│   Date: January 15, 2024                           │
│   Amount: $125.50                                  │
│   Status: Approved                                 │
│                                                    │
│   This action cannot be undone.                    │
│                                                    │
│        ┌──────────┐    ┌───────────────────┐       │
│        │  Cancel  │    │  Delete Receipt   │       │
│        └──────────┘    └───────────────────┘       │
│                              (red/danger)          │
└────────────────────────────────────────────────────┘
```

```tsx
// Add state
const [isDeleting, setIsDeleting] = useState(false);

// Delete handler
const handleDeleteReceipt = async () => {
  if (!deletingReceipt) return;

  setIsDeleting(true);
  try {
    const response = await fetch(`/api/receipts?id=${deletingReceipt.id}`, {
      method: 'DELETE',
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error || 'Failed to delete receipt');
    }

    toast.success("Receipt deleted successfully");
    setDeletingReceipt(null);
    fetchReceipts(); // Refresh the list
  } catch (error) {
    toast.error(error instanceof Error ? error.message : "Failed to delete receipt");
  } finally {
    setIsDeleting(false);
  }
};

{/* Delete Confirmation Dialog */}
<AlertDialog open={!!deletingReceipt} onOpenChange={(open) => !open && setDeletingReceipt(null)}>
  <AlertDialogContent className="bg-[#333333] border-[#444444]">
    <AlertDialogHeader>
      <AlertDialogTitle className="text-white">Delete Receipt?</AlertDialogTitle>
      <AlertDialogDescription asChild>
        <div className="text-gray-300">
          <p className="mb-3">Are you sure you want to delete this receipt?</p>
          {deletingReceipt && (
            <div className="bg-[#2a2a2a] p-3 rounded-lg text-sm space-y-1">
              <p><span className="font-medium">Employee:</span> {deletingReceipt.employeeName}</p>
              <p><span className="font-medium">Date:</span> {formatDate(deletingReceipt.date)}</p>
              <p><span className="font-medium">Amount:</span> {formatCurrency(deletingReceipt.amount)}</p>
              <p><span className="font-medium">Status:</span> <span className="capitalize">{deletingReceipt.status}</span></p>
            </div>
          )}
          <p className="mt-3 text-red-400">This action cannot be undone.</p>
        </div>
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel disabled={isDeleting}>Cancel</AlertDialogCancel>
      <AlertDialogAction
        onClick={handleDeleteReceipt}
        disabled={isDeleting}
        className="bg-red-600 hover:bg-red-700"
      >
        {isDeleting ? "Deleting..." : "Delete Receipt"}
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

---

## File Changes Summary

| File | Changes |
|------|---------|
| `dws-app/src/app/api/receipts/route.ts` | Add DELETE handler, modify PATCH status check for admin bypass |
| `dws-app/src/components/receipt-details-card.tsx` | Add delete button in edit mode, delete confirmation dialog, new props |
| `dws-app/src/components/employee-receipt-table.tsx` | Show edit for all receipts, add "contact admin" dialog, pass onDelete callback |
| `dws-app/src/components/receipt-table.tsx` | Add Actions column with dropdown (Edit/Delete), new props |
| `dws-app/src/components/receipt-dashboard.tsx` | Add edit dialog with ReceiptDetailsCard, delete confirmation, action handlers |

---

## UI Flow Summary

### Employee Flow

```
Employee Receipt Table
├── Receipt is PENDING
│   └── Click Edit → Opens Edit Dialog
│       ├── Modify fields → Save Changes
│       └── Click Delete → Confirmation → Delete
│
└── Receipt is PROCESSED (Approved/Rejected/Reimbursed)
    └── Click Edit → Opens "Contact Admin" Alert
        └── Click OK → Alert closes
```

### Admin Flow

```
Admin Dashboard
└── Receipt Table (any status)
    └── Click Actions (⚙️) dropdown
        ├── Edit → Opens Edit Dialog (shows employee info header)
        │   ├── Modify fields → Save Changes
        │   └── Click Delete → Confirmation → Delete
        │
        └── Delete → Opens Delete Confirmation
            └── Shows receipt details → Confirm → Delete
```

---

## Testing Checklist

### Employee Tests
- [ ] Edit button visible for ALL receipts (not just pending)
- [ ] Clicking edit on pending receipt opens edit dialog
- [ ] Clicking edit on processed receipt shows "Contact Admin" alert
- [ ] Delete button appears inside edit dialog for pending receipts
- [ ] Delete confirmation dialog works
- [ ] Cannot delete processed receipts (blocked by API)
- [ ] List refreshes after edit/delete

### Admin Tests
- [ ] Actions dropdown appears for all receipts in table
- [ ] Edit action opens dialog with employee info header
- [ ] Can edit receipts of any status
- [ ] Delete button works inside admin edit dialog
- [ ] Direct delete from dropdown opens confirmation
- [ ] Confirmation shows receipt details
- [ ] List refreshes after edit/delete

### API Tests
- [ ] DELETE endpoint returns 403 for non-owner, non-admin
- [ ] DELETE endpoint returns 403 for employee deleting processed receipt
- [ ] DELETE endpoint works for admin on any receipt
- [ ] PATCH endpoint allows admin to edit processed receipts
- [ ] PATCH endpoint blocks employee from editing processed receipts
- [ ] Image deleted from storage when receipt deleted

### Edge Cases
- [ ] Deleting receipt with no image doesn't error
- [ ] Network error during delete shows appropriate message
- [ ] Concurrent edit/delete handled gracefully
- [ ] Long description doesn't break delete confirmation layout

---

## Success Criteria

1. Employees can edit and delete their own pending receipts via the edit dialog
2. Employees see "Contact admin" popup when clicking edit on processed receipts
3. Admins can edit and delete any receipt from the dashboard via actions dropdown
4. Delete includes confirmation dialog with receipt details
5. No orphaned images left in storage after delete
6. All changes properly refresh the UI
