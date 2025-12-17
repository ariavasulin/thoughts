---
date: 2025-12-17T21:55:03Z
researcher: Claude
git_commit: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
branch: receipt-editing
repository: DWS-Receipts
topic: "DialogContent accessibility - adding DialogTitle for screen readers"
tags: [research, accessibility, dialog, shadcn-ui, radix-ui, ARI-5]
status: complete
last_updated: 2025-12-17
last_updated_by: Claude
---

# Research: DialogContent Accessibility - ARI-5

**Date**: 2025-12-17T21:55:03Z
**Researcher**: Claude
**Git Commit**: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
**Branch**: receipt-editing
**Repository**: DWS-Receipts

## Research Question
Research everything related to Linear issue ARI-5: "Fix DialogContent accessibility: add DialogTitle for screen readers"

## Summary

The codebase has 4 dialog instances using DialogContent. **One dialog is missing a DialogTitle**, causing accessibility issues for screen readers. The issue is in `employee-receipt-table.tsx:216` where the edit receipt dialog uses DialogContent without a DialogTitle. Three other dialogs in the codebase already implement the accessibility pattern correctly, providing clear examples to follow.

## Detailed Findings

### The Issue Location

**File**: `dws-app/src/components/employee-receipt-table.tsx:216`

The employee receipt table contains an edit dialog that lacks a DialogTitle:

```typescript
// Line 7 - Only imports Dialog and DialogContent, NOT DialogTitle
import { Dialog, DialogContent } from "@/components/ui/dialog";

// Lines 215-233 - Dialog has no DialogTitle
<Dialog open={editDialogOpen} onOpenChange={setEditDialogOpen}>
  <DialogContent className="bg-transparent border-none p-0 max-w-md">
    {selectedReceipt && (
      <ReceiptDetailsCard
        mode="edit"
        receiptId={selectedReceipt.id}
        // ... props
      />
    )}
  </DialogContent>
</Dialog>
```

**Purpose**: This dialog displays a receipt editing interface for pending receipts.

### Dialog Component Architecture

**File**: `dws-app/src/components/ui/dialog.tsx`

The dialog component is built on `@radix-ui/react-dialog` primitives and exports 10 components:

| Component | Lines | Description |
|-----------|-------|-------------|
| Dialog | 9-13 | Root wrapper |
| DialogTrigger | 15-19 | Trigger button |
| DialogPortal | 21-25 | Portal container |
| DialogClose | 27-31 | Close action |
| DialogOverlay | 33-47 | Background overlay |
| DialogContent | 49-73 | Main content container |
| DialogHeader | 75-83 | Header layout |
| DialogFooter | 85-96 | Footer layout |
| DialogTitle | 98-109 | Accessible title (wraps DialogPrimitive.Title) |
| DialogDescription | 111-122 | Accessible description |

**DialogContent** (lines 49-73) automatically includes:
- DialogPortal wrapper
- DialogOverlay background
- Close button with `sr-only` text for accessibility

**DialogTitle** is NOT automatically rendered - consumers must explicitly include it.

### All DialogContent Usages

| File | Line | Has DialogTitle? | Type | Purpose |
|------|------|-----------------|------|---------|
| `employee-receipt-table.tsx` | 216 | **NO** | - | Edit pending receipt |
| `receipt-uploader.tsx` | 328 | YES | sr-only | Confirm OCR-extracted details |
| `receipt-dashboard.tsx` | 789 | YES | visible | Bulk reimburse confirmation |
| `batch-review-dashboard.tsx` | 557 | YES | visible | Batch submission confirmation |
| `ui/command.tsx` | 29 | N/A | - | Wrapper component |

### Existing Accessibility Patterns

#### Pattern 1: Visually Hidden DialogTitle (sr-only)

**File**: `dws-app/src/components/receipt-uploader.tsx:334`

```typescript
// Line 9 - Imports DialogTitle
import { Dialog, DialogContent, DialogTitle } from "@/components/ui/dialog";

// Lines 327-336
<Dialog open={showDetailsCard} onOpenChange={setShowDetailsCard}>
  <DialogContent className="sm:max-w-md bg-[#2e2e2e] p-0 border-none">
    <ReceiptDetailsCard
      onSubmit={handleDetailsSubmit}
      onCancel={handleCancel}
      initialData={extractedData}
    />
    <DialogTitle className="sr-only">Confirm Receipt Details</DialogTitle>
  </DialogContent>
</Dialog>
```

This pattern provides accessibility without a visible title - appropriate when the dialog content is self-explanatory.

#### Pattern 2: Visible DialogTitle with DialogHeader

**File**: `dws-app/src/components/receipt-dashboard.tsx:789-800`

```typescript
// Line 21 - Imports
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";

// Lines 789-800
<DialogContent className="bg-[#333333] text-white border-[#444444]">
  <DialogHeader>
    <DialogTitle className="flex items-center gap-2">
      <AlertCircle className="h-5 w-5 text-yellow-500" />
      Confirm Bulk Update
    </DialogTitle>
    <DialogDescription className="text-gray-300">
      Are you sure you want to mark {pendingBulkUpdateCount} approved receipt...
    </DialogDescription>
  </DialogHeader>
  {/* Footer with buttons */}
</DialogContent>
```

**File**: `dws-app/src/components/batch-review-dashboard.tsx:557-566` uses the same pattern with "Confirm Batch Submission" title.

### VisuallyHidden Component Status

- **No dedicated VisuallyHidden component exists** in the codebase
- `@radix-ui/react-visually-hidden` is available as a transitive dependency (version 1.2.3)
- The codebase uses Tailwind CSS's `sr-only` utility class throughout for the same effect:
  - `dialog.tsx:68` - Close button screen reader text
  - `receipt-uploader.tsx:311,334` - Input label and DialogTitle
  - `carousel.tsx:220,249` - Navigation buttons
  - `sheet.tsx:70` - Close button
  - `sidebar.tsx:282` - Toggle button
  - `pagination.tsx:114` - More pages indicator

## Code References

- `dws-app/src/components/employee-receipt-table.tsx:7` - Missing DialogTitle import
- `dws-app/src/components/employee-receipt-table.tsx:215-233` - Dialog without DialogTitle
- `dws-app/src/components/ui/dialog.tsx:98-109` - DialogTitle component definition
- `dws-app/src/components/receipt-uploader.tsx:334` - sr-only DialogTitle example
- `dws-app/src/components/receipt-dashboard.tsx:791` - Visible DialogTitle example
- `dws-app/src/components/batch-review-dashboard.tsx:559` - Visible DialogTitle example

## Architecture Documentation

### Dialog Composition Pattern

The shadcn/ui dialog follows a composition pattern where:
1. DialogContent provides structure (portal, overlay, close button)
2. Consumers compose content with DialogHeader, DialogTitle, DialogDescription, DialogFooter
3. DialogTitle wraps Radix UI's DialogPrimitive.Title for ARIA labeling

### Accessibility Implementation Options

1. **Visible title**: Use DialogHeader + DialogTitle for dialogs needing clear headings
2. **Hidden title**: Use `<DialogTitle className="sr-only">` for screen readers only
3. **Radix VisuallyHidden**: Import from `@radix-ui/react-visually-hidden` (available but unused)

### Related Radix Dependencies

From `dws-app/package.json`:
- `@radix-ui/react-dialog: ^1.1.14` - Core dialog primitives
- Other Radix packages for consistent accessibility patterns

## Historical Context (from thoughts/)

- `thoughts/shared/research/2025-12-17-receipt-editing-feature.md` - Documents the receipt editing dialog implementation and component reuse pattern
- `thoughts/shared/handoffs/ARI-11/2025-12-17_13-38-00_ARI-11_receipt-editing-approval-research.md` - Notes that employee edit dialog uses ReceiptDetailsCard component

## Related Research

- `thoughts/shared/research/2025-12-17-receipt-editing-feature.md` - Receipt editing feature analysis

## Open Questions

None - the fix path is clear based on existing patterns in the codebase.
