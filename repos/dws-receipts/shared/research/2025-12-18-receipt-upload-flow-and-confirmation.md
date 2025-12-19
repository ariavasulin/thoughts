---
date: 2025-12-18T20:46:37-08:00
researcher: Claude
git_commit: 9c97ecd0dab4a93347949e543a1d31978613ef2b
branch: receipt-editing
repository: DWS-Receipts
topic: "Receipt Upload Flow - BAML Extraction, Confirmation Page, and Duplicate Detection"
tags: [research, codebase, receipt-upload, baml, ocr, categories, confirmation-flow, duplicate-detection]
status: complete
last_updated: 2025-12-18
last_updated_by: Claude
last_updated_note: "Added comprehensive duplicate detection research"
---

# Research: Receipt Upload Flow - BAML Extraction and Confirmation Page

**Date**: 2025-12-18T20:46:37-08:00
**Researcher**: Claude
**Git Commit**: 9c97ecd0dab4a93347949e543a1d31978613ef2b
**Branch**: receipt-editing
**Repository**: DWS-Receipts

## Research Question

How does the current receipt upload flow work? Specifically:
1. What does the BAML function extract (date, amount, category)?
2. When is the confirmation details page shown?
3. How are categories handled currently?
4. How does duplicate receipt detection work, and what edge cases exist?

The goal is to understand the existing implementation to enable skipping the confirmation page when BAML successfully extracts all required fields.

## Summary

The current receipt upload flow **always** shows the confirmation details page (`ReceiptDetailsCard`) after OCR extraction, regardless of extraction success. The BAML function currently extracts only **date** and **amount** - it does not extract categories. Categories are fetched separately from the database API and must be manually selected by the user. There is no confidence scoring or validation on extracted values; null values simply indicate the field couldn't be determined.

**Duplicate Detection**: The system checks for duplicates based on exact match of `user_id`, `receipt_date`, and `amount`. If a match is found, the `description` field is used as a differentiator - submission is **blocked** if descriptions match or both are empty. This check occurs client-side after the user clicks submit, before the actual receipt creation. The duplicate check is entirely bypassed in edit mode, and there's no image-based comparison.

## Detailed Findings

### 1. BAML Receipt Extraction

**Location**: `dws-app/baml_src/receipts.baml`

#### Current Extracted Fields (lines 2-5)

```baml
class ExtractedReceipt {
  date string? @description("Receipt date in YYYY-MM-DD format...")
  amount float? @description("Total amount paid after tax and tip...")
}
```

The BAML function extracts exactly **two fields**:
- `date` (string, nullable) - Format: YYYY-MM-DD
- `amount` (float, nullable) - Total amount paid

**Note**: Category is NOT currently extracted by BAML.

#### Function Definition (lines 8-24)

```baml
function ExtractReceiptFromImage(img: image) -> ExtractedReceipt {
  client GPT4oMini
  prompt #"
    Extract the date and total amount from this receipt image:
    {{ img }}

    If a field cannot be determined, return null for that field.
  "#
}
```

#### API Route (`dws-app/src/app/api/receipts/ocr/route.ts`)

The OCR endpoint:
1. Receives `tempFilePath` (line 17)
2. Downloads image from Supabase Storage (lines 24-31)
3. Converts to base64 (lines 34-35)
4. Calls BAML `ExtractReceiptFromImage` (line 55)
5. Returns extracted data (lines 59-65):

```typescript
return NextResponse.json({
  success: true,
  data: {
    date: extracted.date || null,
    amount: extracted.amount || null,
  },
});
```

#### No Confidence/Validation

There is no confidence scoring. The BAML prompt instructs: "If a field cannot be determined, return null for that field." The API simply passes through whatever the model returns.

### 2. Receipt Upload Flow

**Primary Component**: `dws-app/src/components/receipt-uploader.tsx`

#### Complete Flow Sequence

1. **File Selection** (lines 160-172)
   - User selects/captures image
   - `handleFileSelect` triggered

2. **Upload to Storage** (lines 173-195)
   - POST to `/api/receipts/upload`
   - Returns `tempFilePath`

3. **OCR Extraction** (lines 196-222)
   - POST to `/api/receipts/ocr` with `tempFilePath`
   - Returns `{ date, amount }`
   - Sets `extractedData` state (lines 212-217)

4. **Show Confirmation Dialog** (line 230)
   ```typescript
   setShowDetailsCard(true);  // ALWAYS opens regardless of extraction success
   ```

5. **User Reviews/Edits** in `ReceiptDetailsCard`
   - Pre-fills date and amount from OCR
   - User must select category manually
   - User can add notes

6. **Submit Receipt** (lines 234-289)
   - Validates all required fields
   - Checks for duplicates
   - Creates receipt record

#### Key Insight

The confirmation dialog opens unconditionally in the `finally` block (line 228-231):
```typescript
} finally {
  setIsProcessing(false);
  setShowDetailsCard(true);
}
```

This means the dialog opens even if OCR fails or returns null values.

### 3. Categories System

**API Route**: `dws-app/src/app/api/categories/route.ts`

#### Database Schema

```
categories table:
- id (uuid, primary key)
- name (text, not null, unique)
- created_at (timestamptz)
```

#### Category Selection in Confirmation

**Location**: `dws-app/src/components/receipt-details-card.tsx`

Categories are fetched on component mount (lines 87-121):
```typescript
useEffect(() => {
  async function fetchCategories() {
    const response = await fetch('/api/categories');
    const data = await response.json();
    setCategories(data.categories);

    // Default to "Parking" if no initial category
    if (!initialData?.category_id) {
      const parkingCategory = data.categories.find(
        cat => cat.name.toLowerCase() === 'parking'
      );
      if (parkingCategory) setCategoryId(parkingCategory.id);
    }
  }
  fetchCategories();
}, []);
```

#### Category is Required for Submission

Validation at line 129-132:
```typescript
if (!date || !amount || !categoryId) {
  sonnerToast.error("Missing details", {
    description: "Please fill in Date, Amount, and Category."
  });
  return;
}
```

### 4. Confirm Receipt Details Component

**Location**: `dws-app/src/components/receipt-details-card.tsx`

#### Form Fields (lines 284-359)

1. **Date** - Calendar picker (desktop) or native date input (mobile)
2. **Amount** - Number input with step="0.01"
3. **Category** - Select dropdown (fetched from API)
4. **Notes** - Text input (optional, used for duplicate detection)

#### How Data is Received

For new uploads, `initialData` comes from `extractedData` in `receipt-uploader.tsx`:
```typescript
// After OCR completes (receipt-uploader.tsx:212-217)
setExtractedData({
  receipt_date: ocrResult?.data?.date || undefined,
  amount: ocrResult?.data?.amount || undefined,
});

// Passed to ReceiptDetailsCard (receipt-uploader.tsx:331-332)
<ReceiptDetailsCard
  initialData={extractedData}
  ...
/>
```

#### Duplicate Check Flow (lines 179-216)

Before final submission:
1. Calls `/api/receipts/check-duplicate` with `receipt_date` and `amount`
2. If duplicate found, compares descriptions
3. Blocks submission if description matches or both are empty

### 5. Duplicate Receipt Detection (Detailed)

**API Endpoint**: `dws-app/src/app/api/receipts/check-duplicate/route.ts`

#### Duplicate Detection Criteria

**Primary Criteria (Server-Side):**
1. **`user_id`** - Only checks within the same user's receipts (line 31)
2. **`receipt_date`** - Exact string match in YYYY-MM-DD format (line 32)
3. **`amount`** - Exact numeric match (line 33)

**Secondary Criteria (Client-Side - `receipt-details-card.tsx:200-205`):**
4. **`description`/`notes`** - Case-insensitive, trimmed comparison
   - Used as a differentiator when date+amount match
   - If descriptions match OR both are empty → **blocked**
   - If descriptions differ → **allowed**

**What is NOT checked:**
- Image hash or visual similarity
- Receipt vendor/merchant name
- Category
- Fuzzy matching on date or amount

#### When Duplicate Check Occurs

The check happens **after** the user clicks "Submit Receipt" in the confirmation dialog, **before** the actual receipt creation API call:

```
User clicks Submit → Duplicate Check API → If blocked: show warning toast
                                         → If allowed: POST /api/receipts
```

#### User Experience When Duplicate Found

**Toast Message** (`receipt-details-card.tsx:206-212`):
- Title: "Potential Duplicate Found"
- Description: "A receipt with the same date and amount already exists with a similar or empty description. Please provide a unique description or cancel."
- Duration: 8000ms (8 seconds)
- Form remains open for user to modify description

**Submission Behavior:**
- **BLOCKED** (not just warned) until user changes description or cancels
- User cannot bypass by clicking submit again with same data

#### Edge Cases and Limitations

| Edge Case | Current Behavior | Impact |
|-----------|------------------|--------|
| Edit mode | Duplicate check **bypassed entirely** (`receipt-details-card.tsx:144`) | User can create duplicates via editing |
| Different users | Only checks current user's receipts | Cross-user duplicates possible |
| Same image, different description | **Allowed** - no image comparison | Duplicate images can slip through |
| Amount $10.00 vs $10.01 | Not detected (exact match only) | OCR errors bypass detection |
| Rapid double-click | No race condition protection | Could create duplicates |
| Network failure | Shows error toast, blocks submission | Cannot submit if check fails |

#### Database Query (`check-duplicate/route.ts:28-33`)

```typescript
const { data: existingReceipts, error } = await supabase
  .from('receipts')
  .select('id, description')
  .eq('user_id', userId)
  .eq('receipt_date', receipt_date)
  .eq('amount', numericAmount);
```

No database-level unique constraints exist on (user_id, receipt_date, amount, description).

## Code References

- `dws-app/baml_src/receipts.baml:2-24` - BAML extraction definition
- `dws-app/src/app/api/receipts/ocr/route.ts:55` - BAML function call
- `dws-app/src/components/receipt-uploader.tsx:160-232` - Upload and OCR flow
- `dws-app/src/components/receipt-uploader.tsx:230` - Dialog always opens
- `dws-app/src/components/receipt-details-card.tsx:40-49` - Component props interface
- `dws-app/src/components/receipt-details-card.tsx:87-123` - Category fetching
- `dws-app/src/components/receipt-details-card.tsx:129-132` - Required field validation
- `dws-app/src/components/receipt-details-card.tsx:179-227` - Duplicate check flow in handleSubmit
- `dws-app/src/components/receipt-details-card.tsx:200-215` - Description comparison logic
- `dws-app/src/app/api/categories/route.ts:16-19` - Categories query
- `dws-app/src/app/api/receipts/check-duplicate/route.ts:28-33` - Duplicate check database query

## Architecture Documentation

### Current Data Flow

```
User selects image
       ↓
POST /api/receipts/upload
       ↓ tempFilePath
POST /api/receipts/ocr
       ↓ { date, amount } (no category)
ReceiptDetailsCard dialog opens (ALWAYS)
       ↓ User must select category
User reviews/edits, clicks Submit
       ↓
POST /api/receipts/check-duplicate
       ↓
POST /api/receipts (creates receipt)
```

### Required Changes for Auto-Submit

To implement skipping the confirmation page when BAML extracts all fields successfully:

1. **Extend BAML to extract category** (`baml_src/receipts.baml`)
   - Add `category` field to `ExtractedReceipt` class
   - Update prompt to infer category from available options

2. **Define success criteria** (could be in `receipt-uploader.tsx`)
   - Check if date, amount, AND category are all non-null
   - Optionally add confidence thresholds

3. **Conditional dialog logic** (`receipt-uploader.tsx:228-231`)
   - If all fields valid: skip dialog, auto-submit
   - If any field missing/null: show confirmation dialog

4. **Handle category matching**
   - BAML returns category name
   - Need to map to category_id from database

## Open Questions

1. **Category inference approach**: Should BAML receive the list of valid categories in the prompt, or should it guess freely and then fuzzy-match?

2. **Confidence thresholds**: Should we add confidence scores to determine "proper" extraction vs uncertain?

3. **User override**: Should there be a way for users to still review before submit (e.g., a "Review before submit" setting)?

4. **Duplicate handling in auto-submit flow**: When auto-submitting (skipping confirmation), how should duplicates be handled?
   - Option A: Run duplicate check first, show confirmation dialog only if duplicate detected
   - Option B: Auto-submit fails silently, then show confirmation dialog with warning
   - Option C: Always show a brief "submitting..." state that can be interrupted if duplicate found

5. **Duplicate detection timing**: Currently duplicate check happens after user clicks submit. For auto-submit, should it:
   - Run during OCR phase (before auto-submit decision)?
   - Run as part of auto-submit (and fallback to dialog if duplicate)?

6. **Description requirement**: If auto-submitting, notes/description will be empty. This would be blocked by duplicate detection if a same date+amount receipt already exists with empty description. Should auto-submit:
   - Skip duplicate check entirely?
   - Use a generated description (e.g., category name)?
   - Always go to confirmation if potential duplicate detected?
