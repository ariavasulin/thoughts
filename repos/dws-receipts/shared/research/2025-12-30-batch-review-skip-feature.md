---
date: 2025-12-30T12:35:11-08:00
researcher: ariasulin
git_commit: d085b71aecb5d16937184cf1d73b22840f8fd9c1
branch: test-branch
repository: DWS-Receipts
topic: "Skip functionality for batch receipt submission"
tags: [research, codebase, batch-review, feature-analysis]
status: complete
last_updated: 2025-12-30
last_updated_by: ariasulin
---

# Research: Skip Functionality for Batch Receipt Submission

**Date**: 2025-12-30T12:35:11-08:00
**Researcher**: ariasulin
**Git Commit**: d085b71aecb5d16937184cf1d73b22840f8fd9c1
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question

What would it take to be able to skip some receipts in the batch receipt submission, i.e., leave them pending but still be able to submit others?

## Summary

**Good news: The submission logic already supports partial submissions.** The `handleSubmitAll` function only processes receipts that have decisions - skipped receipts are automatically left as "Pending". The changes needed are purely **UI/UX modifications**, not backend logic.

### What Already Works

- `handleSubmitAll` iterates only over `Object.entries(decisions)` (line 102)
- Receipts without a decision are not included in the update
- Submit is allowed when `decisionsCount > 0` (line 91)
- After submission, skipped receipts remain pending and will appear in the next batch review

### What Needs to Change

Only UI/UX flow changes:

1. **Add a "Skip" button** - Currently only Approve/Reject exist
2. **Update completion screen trigger** - Currently requires ALL receipts to have decisions
3. **Add accessible "Submit" button** - Allow submission before reaching the end

## Detailed Findings

### Current Decision Flow

**File**: `dws-app/src/components/batch-review-dashboard.tsx`

```typescript
// Line 33: Decisions only track approve/reject
const [decisions, setDecisions] = useState<Record<string, "approved" | "rejected">>({})

// Lines 102-105: Only receipts WITH decisions are submitted
const updatePromises = Object.entries(decisions).map(([id, status]) => {
  const dbStatus = status.charAt(0).toUpperCase() + status.slice(1)
  return supabase.from("receipts").update({ status: dbStatus }).eq("id", id)
})
```

### What Blocks Skip Currently

1. **No skip button** (lines 299-316): Only Reject and Approve buttons exist

2. **Completion screen requires all decisions** (lines 71-74, 135-139):
   ```typescript
   } else if (decisionsCount === receipts.length) {
     setShowCompletionScreen(true)
   }
   ```

3. **Submit button only on completion screen** (lines 378-384): Users can't submit until they've made decisions on all receipts

## Implementation Changes Required

### 1. Add Skip Button (Minimal Change)

Add a "Skip" button between Reject and Approve that just calls `moveNext()` without adding to `decisions`:

```typescript
// New handler - just moves forward without recording a decision
const handleSkip = () => {
  if (!currentReceipt) return;
  moveNext()
}
```

Add button in the UI (around line 307):
```tsx
<Button size="sm" onClick={handleSkip} disabled={isSubmitting}>
  Skip
</Button>
```

### 2. Update Completion Screen Logic

Change the trigger to show completion when at the end OR when user wants to submit:

```typescript
// Option A: Show when at end regardless of decisions
const moveNext = () => {
  if (currentIndex < receipts.length - 1) {
    setCurrentIndex(currentIndex + 1)
  } else {
    // Always show completion at end, even with skips
    setShowCompletionScreen(true)
  }
}

// Option B: Add a "Done Reviewing" button that shows completion anytime
```

### 3. Add Always-Visible Submit Option

Add a "Submit X Decisions" button in the header or footer that's always accessible when `decisionsCount > 0`.

### 4. UI Updates for Skipped Receipts

Update the pagination dots to show three states:
- Reviewed (approved/rejected): Green
- Skipped: Gray/neutral
- Not yet viewed: Default

## Code References

- `dws-app/src/components/batch-review-dashboard.tsx:33` - decisions state definition
- `dws-app/src/components/batch-review-dashboard.tsx:68-75` - moveNext function
- `dws-app/src/components/batch-review-dashboard.tsx:90-96` - handleSubmitAllClick (already allows partial)
- `dws-app/src/components/batch-review-dashboard.tsx:102-105` - submit logic (already only submits decisions)
- `dws-app/src/components/batch-review-dashboard.tsx:299-316` - approve/reject buttons

## Effort Estimate

**Small change** - Primarily UI modifications:
- Add Skip button: ~10 lines
- Update completion screen trigger: ~5 lines
- Add persistent submit button: ~15 lines
- Update pagination styling: ~10 lines

Total: ~40 lines of changes, all in `batch-review-dashboard.tsx`

## Architecture Insights

The current architecture cleanly separates "viewing" from "deciding" - the `reviewedCount` tracks what's been seen while `decisions` tracks actual choices. This makes adding skip functionality straightforward since skipped receipts simply have no entry in `decisions`.

## Open Questions

1. Should skipped receipts be visually differentiated from "not yet viewed"?
2. Should there be a way to "undo skip" and go back to make a decision?
3. Should the summary screen show skipped count separately?
