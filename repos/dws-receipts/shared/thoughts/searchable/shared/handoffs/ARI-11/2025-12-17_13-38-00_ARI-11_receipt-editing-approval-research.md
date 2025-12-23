---
date: 2025-12-17T13:38:00-08:00
researcher: Claude
git_commit: 49af201228f7575ab6f3e8d5ea33b39ce4e2b212
branch: receipt-editing
repository: DWS-Receipts
topic: "Receipt Editing Approval Logic - Research Complete"
tags: [research, receipt-editing, approval-workflow, ARI-11]
status: complete
last_updated: 2025-12-17
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: ARI-11 Receipt Editing Approval Research

## Task(s)

| Task | Status |
|------|--------|
| Research current receipt editing implementation | Completed |
| Document PATCH endpoint permissions | Completed |
| Document employee frontend editing flow | Completed |
| Document admin dashboard capabilities | Completed |
| Find existing plans/notes for approval workflow | Completed (none found) |
| Identify Linear ticket for this work | Completed (ARI-11) |
| Move ARI-11 to In Progress | Completed |
| Update CLAUDE.md with Linear context | Completed |

**Summary**: Completed comprehensive research on the receipt editing feature. The employee-side editing is implemented, but the approval workflow (ARI-11) has not been started. Ready for implementation planning.

## Critical References

- `thoughts/shared/research/2025-12-17-receipt-editing-feature.md` - Full research document with code references
- Linear ticket: https://linear.app/ariav/issue/ARI-11/create-editing-approval-logic-for-receipts

## Recent changes

- `CLAUDE.md:44-49` - Added Linear Integration section with workspace/team/project info

## Learnings

### Current Implementation State

1. **PATCH Endpoint** (`dws-app/src/app/api/receipts/route.ts:246-391`):
   - Allows editing: `receipt_date`, `amount`, `category_id`, `notes`
   - Ownership check at lines 296-308: user must own receipt OR be admin
   - Status restriction at lines 310-316: only "Pending" receipts editable
   - Admins CAN edit any user's pending receipts via API

2. **Employee Frontend** (`dws-app/src/components/employee-receipt-table.tsx`):
   - Edit button (pencil icon) at lines 131-144, only for pending receipts
   - Opens `ReceiptDetailsCard` in edit mode (lines 214-233)
   - Edits save immediately - no approval required

3. **Admin Dashboard** (`dws-app/src/components/receipt-dashboard.tsx`):
   - NO edit UI exists for admins
   - Can only approve/reject and bulk reimburse
   - Backend supports admin editing but no frontend

### What's Missing (ARI-11 scope)

- No mechanism to flag edited receipts for re-review
- No "edited" status or `needs_reapproval` flag
- No edit history/audit trail
- No admin edit UI

### Linear Integration

- Workspace: `ariav`
- Team: `ARI` (prefix ARI-XXX)
- Project: `DWS`
- Config in `.linear.toml`

## Artifacts

- `thoughts/shared/research/2025-12-17-receipt-editing-feature.md` - Comprehensive research document
- `CLAUDE.md:44-49` - Updated with Linear integration context

## Action Items & Next Steps

1. **Create implementation plan for ARI-11** - Define approval workflow approach:
   - Option A: New status value (e.g., "PendingEdit")
   - Option B: Boolean flag (`needs_reapproval`) on receipts table
   - Option C: Separate edits table for pending changes

2. **Design decisions needed**:
   - Which field changes trigger re-approval? (amount/category vs notes)
   - Can admins edit non-pending receipts?
   - Should editing reset status to Pending?
   - What audit trail is needed?

3. **Implementation phases** (suggested):
   - Phase 1: Database schema changes (add tracking fields/table)
   - Phase 2: Update PATCH endpoint to flag edits
   - Phase 3: Admin UI for reviewing edited receipts
   - Phase 4: Admin edit capability from dashboard

## Other Notes

### Related Tickets
- **ARI-12** (In Review): Employee frontend editing - the implementation this builds on
- **ARI-11** (In Progress): This ticket - approval workflow

### Key Files for Implementation
- `dws-app/src/app/api/receipts/route.ts` - PATCH endpoint to modify
- `dws-app/src/components/receipt-dashboard.tsx` - Add admin edit UI here
- `dws-app/src/components/batch-review-dashboard.tsx` - Could add edit review here
- `dws-app/src/lib/types.ts` - Update Receipt interface for new fields

### Database Status Values
Stored capitalized: "Pending", "Approved", "Rejected", "Reimbursed"
Frontend normalizes to lowercase for display.
