# Auto-Submit Receipt When OCR Extracts All Fields

## Overview

Implement automatic receipt submission when BAML successfully extracts all required fields (date, amount, category), skipping the manual confirmation dialog. This will significantly improve UX for the common case where receipts are clearly readable.

## Current State Analysis

**Current Flow:**
1. User uploads receipt image → temporary storage
2. OCR (BAML + GPT-4o-mini) extracts **only date and amount**
3. Confirmation dialog **always** opens (regardless of OCR success)
4. User must manually select category (defaults to "Parking")
5. Duplicate check runs on submit
6. Receipt created

**Key Files:**
- `dws-app/baml_src/receipts.baml:2-24` - BAML extraction (only extracts date/amount)
- `dws-app/src/app/api/receipts/ocr/route.ts` - OCR API endpoint
- `dws-app/src/components/receipt-uploader.tsx:160-232` - Upload flow, always shows dialog at line 230
- `dws-app/src/components/receipt-details-card.tsx` - Confirmation dialog
- `dws-app/src/app/api/receipts/check-duplicate/route.ts` - Duplicate detection

## Desired End State

**New Flow:**
1. User uploads receipt image → temporary storage
2. OCR extracts **date, amount, AND category** (constrained to valid categories)
3. System checks for duplicates using extracted date+amount
4. **If all fields extracted AND no duplicate:**
   - Auto-submit receipt (skip confirmation dialog)
   - Show toast: "Receipt submitted!" with "Edit" action button (5 seconds)
   - User can click "Edit" to open confirmation dialog and modify
5. **If any field missing OR duplicate detected:**
   - Show confirmation dialog (current behavior)
   - User fills in missing fields or adds distinguishing description

### Key Discoveries:
- BAML schema at `receipts.baml:2-5` is easily extensible
- Categories fetched from DB via `/api/categories` at `categories/route.ts:16-19`
- Duplicate check at `check-duplicate/route.ts:28-33` queries by user_id, receipt_date, amount
- Dialog control at `receipt-uploader.tsx:230` - currently unconditional `setShowDetailsCard(true)`

## What We're NOT Doing

- Confidence scoring for extraction quality
- Fuzzy matching for categories (using constrained list instead)
- User preference setting for "always review"
- Image-based duplicate detection
- Changes to edit mode behavior

## Implementation Approach

1. **Extend BAML schema** to extract category from a constrained list
2. **Update OCR API** to fetch categories and include in BAML prompt, plus run duplicate check
3. **Modify receipt-uploader** to conditionally auto-submit or show dialog based on OCR result
4. **Add undo/edit toast** for auto-submitted receipts

---

## Phase 1: Extend BAML to Extract Category

### Overview
Add category extraction to the BAML schema with a hardcoded list of valid categories in the prompt.

### Changes Required:

#### 1. BAML Schema Definition
**File**: `dws-app/baml_src/receipts.baml`
**Changes**: Add category field with hardcoded category list in prompt

```baml
// Receipt data extracted from image
class ExtractedReceipt {
  date string? @description("Receipt date in YYYY-MM-DD format. Extract the purchase/transaction date, not print date or expiry dates.")
  amount float? @description("Total amount paid after tax and tip. Look for TOTAL, GRAND TOTAL, AMOUNT DUE, or the final/largest amount.")
  category string? @description("The expense category. Must be exactly one of: Parking, Gas, Meals & Entertainment, Office Supplies, Other")
}

// Main extraction function
function ExtractReceiptFromImage(img: image) -> ExtractedReceipt {
  client GPT4oMini
  prompt #"
    {{_.role("user")}}

    Extract the date, total amount, and category from this receipt image:
    {{ img }}

    Instructions:
    - date: Format as YYYY-MM-DD. Use the purchase/transaction date, not promotional or expiry dates. If year is unclear, assume current year.
    - amount: The final total amount paid (after tax, tip, discounts). Look for "TOTAL", "GRAND TOTAL", "AMOUNT DUE", "BALANCE DUE". If multiple totals exist, use the largest/final one.
    - category: Choose exactly one from: Parking, Gas, Meals & Entertainment, Office Supplies, Other
      Common mappings:
      - Parking meters, garages, lots → "Parking"
      - Gas stations, fuel → "Gas"
      - Restaurants, fast food, cafes, groceries → "Meals & Entertainment"
      - Office supplies, printing, stationery → "Office Supplies"
      - If unclear or doesn't fit → "Other"

    If a field cannot be determined with reasonable confidence, return null for that field.

    {{ ctx.output_format }}
  "#
}
```

### Success Criteria:

#### Automated Verification:
- [x] BAML generates successfully: `cd dws-app && npx baml-cli generate`
- [x] TypeScript types include `category?: string | null` in generated `ExtractedReceipt`
- [x] No build errors: `npm run build`

#### Manual Verification:
- [ ] Generated TypeScript types in `baml_client/types.ts` include the new category field

**Implementation Note**: After completing this phase and all automated verification passes, regenerate BAML client and verify types before proceeding to Phase 2.

---

## Phase 2: Update OCR API to Return Category and Duplicate Status

### Overview
Modify the OCR API to:
1. Call BAML extraction (categories hardcoded in prompt)
2. Map extracted category name to category_id (single DB lookup)
3. Check for duplicates if date and amount are extracted
4. Return all data needed for auto-submit decision

### Changes Required:

#### 1. OCR API Endpoint
**File**: `dws-app/src/app/api/receipts/ocr/route.ts`
**Changes**: Map category name to ID, check duplicates, return `canAutoSubmit` flag

```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { createSupabaseServerClient } from '@/lib/supabaseServerClient';
import { b } from '@baml/index';
import { Image } from '@boundaryml/baml';

export async function POST(request: NextRequest) {
  try {
    const supabase = await createSupabaseServerClient();
    const { data: { session }, error: sessionError } = await supabase.auth.getSession();

    if (sessionError || !session) {
      console.error('OCR API: Unauthorized - Session error or no session', sessionError);
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const userId = session.user.id;
    const body = await request.json();
    const { tempFilePath } = body;

    if (!tempFilePath || typeof tempFilePath !== 'string') {
      return NextResponse.json({ error: 'Missing or invalid tempFilePath' }, { status: 400 });
    }

    // Download image from Supabase Storage
    const { data: fileData, error: downloadError } = await supabase.storage
      .from('receipt-images')
      .download(tempFilePath);

    if (downloadError || !fileData) {
      console.error(`OCR API: Error downloading file from Supabase Storage: ${tempFilePath}`, downloadError);
      return NextResponse.json({ error: 'Failed to download image from storage', details: downloadError?.message }, { status: 500 });
    }

    // Convert to base64 for BAML
    const arrayBuffer = await fileData.arrayBuffer();
    const base64 = Buffer.from(arrayBuffer).toString('base64');

    // Determine media type from file extension
    const extension = tempFilePath.split('.').pop()?.toLowerCase() || 'jpeg';
    const mediaTypeMap: Record<string, string> = {
      'jpg': 'image/jpeg',
      'jpeg': 'image/jpeg',
      'png': 'image/png',
      'webp': 'image/webp',
      'heic': 'image/heic',
      'heif': 'image/heif',
      'pdf': 'application/pdf',
    };
    const mediaType = mediaTypeMap[extension] || 'image/jpeg';

    // Create BAML Image from base64
    const image = Image.fromBase64(mediaType, base64);

    // Call BAML extraction function (categories are hardcoded in the prompt)
    console.log('OCR API: Calling BAML ExtractReceiptFromImage...');
    const extracted = await b.ExtractReceiptFromImage(image);
    console.log('OCR API: BAML extraction result:', extracted);

    // Map category name to category_id (single query)
    let categoryId: string | null = null;
    if (extracted.category) {
      const { data: categoryMatch } = await supabase
        .from('categories')
        .select('id')
        .ilike('name', extracted.category)
        .single();

      if (categoryMatch) {
        categoryId = categoryMatch.id;
      }
    }

    // Check for duplicates if we have date and amount
    let isDuplicate = false;
    let existingReceipts: { id: string; description: string }[] = [];

    if (extracted.date && extracted.amount !== null && extracted.amount !== undefined) {
      const { data: duplicates, error: dupError } = await supabase
        .from('receipts')
        .select('id, description')
        .eq('user_id', userId)
        .eq('receipt_date', extracted.date)
        .eq('amount', extracted.amount);

      if (!dupError && duplicates && duplicates.length > 0) {
        isDuplicate = true;
        existingReceipts = duplicates.map(r => ({
          id: r.id,
          description: r.description || ''
        }));
      }
    }

    // Determine if auto-submit is possible
    const canAutoSubmit = !!(
      extracted.date &&
      extracted.amount !== null &&
      extracted.amount !== undefined &&
      categoryId &&
      !isDuplicate
    );

    // Return extracted data with auto-submit guidance
    return NextResponse.json({
      success: true,
      data: {
        date: extracted.date || null,
        amount: extracted.amount ?? null,
        category: extracted.category || null,
        category_id: categoryId,
      },
      duplicate: {
        isDuplicate,
        existingReceipts,
      },
      canAutoSubmit,
    });

  } catch (error: unknown) {
    const errorMessage = error instanceof Error ? error.message : 'Unknown error';
    console.error('OCR API: Unexpected error:', error);
    return NextResponse.json({
      error: 'Internal server error',
      details: errorMessage
    }, { status: 500 });
  }
}
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `npm run build`
- [x] API returns expected shape with `data`, `duplicate`, and `canAutoSubmit` fields

#### Manual Verification:
- [ ] Upload a clear parking receipt → API returns `canAutoSubmit: true` with date, amount, category_id
- [ ] Upload same receipt again → API returns `canAutoSubmit: false` with `isDuplicate: true`
- [ ] Upload unclear receipt → API returns `canAutoSubmit: false` with null fields

**Implementation Note**: After completing this phase, test the OCR API directly before proceeding to Phase 3.

---

## Phase 3: Implement Conditional Auto-Submit with Undo Toast

### Overview
Modify the receipt uploader to auto-submit when OCR returns `canAutoSubmit: true`, showing a toast with an "Edit" button. Fall back to confirmation dialog otherwise.

### Changes Required:

#### 1. Receipt Uploader Component
**File**: `dws-app/src/components/receipt-uploader.tsx`
**Changes**: Add auto-submit logic after OCR, undo toast, and error recovery

```typescript
"use client"

import type React from "react"
import { useState, useRef, useEffect } from "react"
import { Button } from "@/components/ui/button"
import { Upload, Camera, FileImage } from "lucide-react"
import { toast as sonnerToast } from "sonner"
import { useMobile } from "@/hooks/use-mobile"
import { Dialog, DialogContent, DialogTitle, DialogDescription } from "@/components/ui/dialog"
import { ReceiptDetailsCard } from "@/components/receipt-details-card"
import { v4 as uuidv4 } from "uuid"
import type { Receipt } from "@/lib/types"

interface ReceiptUploaderProps {
  onReceiptAdded?: (receipt: Receipt) => void
}

// OCR API response type
interface OcrResponse {
  success: boolean
  data: {
    date: string | null
    amount: number | null
    category: string | null
    category_id: string | null
  }
  duplicate: {
    isDuplicate: boolean
    existingReceipts: { id: string; description: string }[]
  }
  canAutoSubmit: boolean
  error?: string
}

export default function ReceiptUploader({ onReceiptAdded }: ReceiptUploaderProps) {
  const [isProcessingFile, setIsProcessingFile] = useState(false)
  const [isSubmitting, setIsSubmitting] = useState(false)
  const [showDetailsCard, setShowDetailsCard] = useState(false)
  const [uploadedFile, setUploadedFile] = useState<File | null>(null)
  const [tempFilePathState, setTempFilePathState] = useState<string | null>(null)
  const [extractedData, setExtractedData] = useState<Partial<Receipt>>({})
  const fileInputRef = useRef<HTMLInputElement>(null)
  const isMobile = useMobile()

  // ... (keep existing helper functions: testPlatformDetection, getUserAgent, getAcceptAttribute,
  //      getButtonText, getButtonIcon, detectUserBehavior, validateFile)

  const handleFileSelect = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0]
    if (!file) return

    setIsProcessingFile(true)
    setUploadedFile(file)
    setExtractedData({})

    // Detect and log user behavior
    if (isMobile) {
      detectUserBehavior(file)
    }

    try {
      // Step 1: Upload the file to get a temporary path (for OCR)
      const formData = new FormData()
      formData.append("file", file)

      const uploadResponse = await fetch("/api/receipts/upload", {
        method: "POST",
        body: formData,
      })

      if (!uploadResponse.ok) {
        const errorData = await uploadResponse.json()
        throw new Error(errorData.error || "Failed to pre-upload image for OCR.")
      }

      const uploadResult = await uploadResponse.json()
      if (!uploadResult.success || !uploadResult.tempFilePath) {
        throw new Error(uploadResult.error || "Image pre-upload did not return a valid path.")
      }

      const tempFilePath = uploadResult.tempFilePath
      setTempFilePathState(tempFilePath)

      // Step 2: Call OCR API with the temporary file path
      sonnerToast.info("Extracting receipt details...", { id: "ocr-toast" })
      const ocrResponse = await fetch("/api/receipts/ocr", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ tempFilePath }),
      })

      if (!ocrResponse.ok) {
        const errorData = await ocrResponse.json()
        sonnerToast.error("OCR Failed", { id: "ocr-toast", description: errorData.error || "Could not extract details." })
        setExtractedData({})
        setIsProcessingFile(false)
        setShowDetailsCard(true)
        return
      }

      const ocrResult: OcrResponse = await ocrResponse.json()

      // Store extracted data for potential dialog use
      const extractedReceiptData: Partial<Receipt> = {
        receipt_date: ocrResult.data.date || undefined,
        amount: ocrResult.data.amount !== null ? ocrResult.data.amount : undefined,
        category_id: ocrResult.data.category_id || undefined,
      }
      setExtractedData(extractedReceiptData)

      // Step 3: Decide whether to auto-submit or show dialog
      if (ocrResult.canAutoSubmit && extractedReceiptData.receipt_date &&
          extractedReceiptData.amount !== undefined && extractedReceiptData.category_id) {

        // Auto-submit the receipt
        sonnerToast.dismiss("ocr-toast")
        await handleAutoSubmit(extractedReceiptData, tempFilePath)

      } else {
        // Show confirmation dialog - either fields missing or duplicate detected
        setIsProcessingFile(false)

        if (ocrResult.duplicate?.isDuplicate) {
          sonnerToast.warning("Potential duplicate detected", {
            id: "ocr-toast",
            description: "A receipt with the same date and amount exists. Please add a description to differentiate.",
            duration: 5000
          })
        } else if (!ocrResult.data.date || ocrResult.data.amount === null || !ocrResult.data.category_id) {
          sonnerToast.info("Please confirm details", {
            id: "ocr-toast",
            description: "Some fields couldn't be extracted automatically.",
            duration: 3000
          })
        }

        setShowDetailsCard(true)
      }

    } catch (error) {
      console.error("Error during file processing or OCR:", error)
      const errorMessage = error instanceof Error ? error.message : "File processing error."
      sonnerToast.error("Processing Error", { id: "ocr-toast", description: errorMessage })
      setExtractedData({})
      setIsProcessingFile(false)
      setShowDetailsCard(true)
    }
  }

  // New function for auto-submit with undo toast
  const handleAutoSubmit = async (receiptData: Partial<Receipt>, tempFilePath: string) => {
    setIsSubmitting(true)

    try {
      const createReceiptPayload = {
        receipt_date: receiptData.receipt_date,
        amount: receiptData.amount,
        category_id: receiptData.category_id,
        notes: '', // Empty for auto-submit
        tempFilePath,
      }

      const createReceiptResponse = await fetch("/api/receipts", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(createReceiptPayload),
      })

      if (!createReceiptResponse.ok) {
        const errorData = await createReceiptResponse.json()
        throw new Error(errorData.error || "Failed to create receipt record.")
      }

      const createReceiptResult = await createReceiptResponse.json()
      if (!createReceiptResult.success || !createReceiptResult.receipt) {
        throw new Error(createReceiptResult.error || "Receipt creation did not return a valid receipt.")
      }

      const createdReceipt = createReceiptResult.receipt

      // Notify parent of new receipt
      if (onReceiptAdded) {
        onReceiptAdded(createdReceipt)
      }

      // Show success toast with Edit action
      sonnerToast.success("Receipt submitted!", {
        id: "auto-submit-toast",
        description: `$${receiptData.amount?.toFixed(2)} on ${receiptData.receipt_date}`,
        duration: 5000,
        action: {
          label: "Edit",
          onClick: () => {
            // Open the details card in edit mode with the created receipt
            setExtractedData({
              ...receiptData,
              id: createdReceipt.id,
            })
            setShowDetailsCard(true)
          }
        }
      })

      // Reset uploader state
      if (fileInputRef.current) {
        fileInputRef.current.value = ""
      }
      setUploadedFile(null)
      setTempFilePathState(null)
      setExtractedData({})

    } catch (error) {
      console.error("Error during auto-submit:", error)
      const errorMessage = error instanceof Error ? error.message : "Auto-submit failed."

      // Fall back to showing dialog with extracted data
      sonnerToast.error("Auto-submit failed", {
        id: "auto-submit-toast",
        description: `${errorMessage}. Please review and submit manually.`,
        duration: 5000
      })
      setShowDetailsCard(true)

    } finally {
      setIsSubmitting(false)
      setIsProcessingFile(false)
    }
  }

  const handleDetailsSubmit = async (receiptData: Partial<Receipt>) => {
    // ... (keep existing implementation unchanged)
  }

  const handleCancel = () => {
    // ... (keep existing implementation unchanged)
  }

  return (
    // ... (keep existing JSX, but update ReceiptDetailsCard to support edit mode)
    <>
      <div className="w-full">
        {/* ... existing file input and button ... */}
      </div>

      <Dialog open={showDetailsCard} onOpenChange={setShowDetailsCard}>
        <DialogContent className="sm:max-w-md bg-[#2e2e2e] p-0 border-none">
          <ReceiptDetailsCard
            onSubmit={handleDetailsSubmit}
            onCancel={handleCancel}
            initialData={extractedData}
            // If extractedData has an id, we're editing an auto-submitted receipt
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
    </>
  )
}
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript errors: `npm run build`
- [x] No lint errors: `npm run lint` (pre-existing lint issues in codebase, no new issues introduced)

#### Manual Verification:
- [ ] Upload clear parking receipt → auto-submits, shows toast with "Edit" button
- [ ] Click "Edit" on toast → opens edit dialog with receipt data
- [ ] Upload unclear receipt → shows confirmation dialog
- [ ] Upload duplicate receipt → shows confirmation dialog with duplicate warning
- [ ] Auto-submit network failure → falls back to confirmation dialog

**Implementation Note**: After completing this phase, test the full flow on mobile and desktop before finalizing.

---

## Phase 4: Handle Edit from Toast

### Overview
When user clicks "Edit" on the auto-submit success toast, the confirmation dialog should open in edit mode for the just-created receipt.

### Changes Required:

The changes in Phase 3 already handle this by:
1. Storing the created receipt ID in `extractedData`
2. Passing `mode='edit'` and `receiptId` to `ReceiptDetailsCard` when ID is present
3. Using `onEditSuccess` callback to update the parent

No additional changes needed if ReceiptDetailsCard edit mode is already working.

### Success Criteria:

#### Manual Verification:
- [ ] Auto-submit → click "Edit" → dialog shows "Edit Receipt Details" header
- [ ] Modify amount → Save → receipt updates in list
- [ ] Cancel edit → dialog closes, receipt unchanged

---

## Testing Strategy

### Unit Tests:
- BAML extraction returns category when clear receipt provided
- OCR API returns correct `canAutoSubmit` value based on extraction results
- Duplicate detection correctly identifies matching date+amount

### Integration Tests:
- Full upload flow with auto-submit
- Full upload flow with dialog fallback
- Edit flow from toast action

### Manual Testing Steps:
1. Upload a clear parking meter receipt → verify auto-submit and toast
2. Click "Edit" on toast → verify edit dialog opens with correct data
3. Upload the same receipt again → verify duplicate warning and dialog
4. Upload a receipt with unclear date → verify dialog opens with amount+category pre-filled
5. Upload a receipt with unknown category → verify dialog opens with date+amount pre-filled
6. Simulate network failure during auto-submit → verify graceful fallback to dialog

## Performance Considerations

- Categories are now fetched in OCR API (server-side) rather than client-side, reducing one client request
- Duplicate check happens during OCR call, not as separate request during auto-submit
- Toast with action button should dismiss after 5 seconds if no interaction

## Migration Notes

No database migrations required. The change is purely in application logic:
- BAML schema change requires regenerating client
- OCR API response shape changes (additive, not breaking)
- Client handles new response shape with graceful fallback

## References

- Original research: `thoughts/shared/research/2025-12-18-receipt-upload-flow-and-confirmation.md`
- BAML documentation: https://docs.boundaryml.com/
- Sonner toast actions: https://sonner.emilkowal.ski/

