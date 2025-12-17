# BAML + VLM Receipt Parsing Implementation Plan

## Overview

Replace the existing Google Cloud Vision API + regex-based receipt parsing with BAML (Boundary AI Markup Language) + GPT-4o-mini VLM (via OpenRouter) for more robust, accurate receipt data extraction. This change addresses ongoing parsing errors and enables extraction of additional fields (vendor, suggested category) that were previously impossible with OCR + regex.

**Model Routing**: We use OpenRouter as the model routing layer instead of calling OpenAI directly. This provides:
- Unified API for accessing multiple model providers
- Easy model switching without code changes
- Fallback options and load balancing
- Consolidated billing across providers

## Current State Analysis

### Current Implementation
- **File**: `dws-app/src/app/api/receipts/ocr/route.ts` (~233 lines)
- **Approach**: Google Cloud Vision `textDetection` -> regex parsing
- **Fields Extracted**: Only `date` and `amount`
- **Known Issues**:
  - Non-standard date formats fail (only 10 formats supported)
  - Receipts without TOTAL keyword fall back to risky "max amount" heuristic
  - European decimal separators (1.234,56) parse incorrectly
  - No vendor/merchant extraction
  - No category auto-suggestion

### Key Discoveries
- OCR route is called from `receipt-uploader.tsx:198-201` with `tempFilePath`
- Response format expected: `{ success: true, data: { date: string, amount: number } }`
- `receipt-details-card.tsx` receives extracted data via `initialData` prop
- Categories are fetched from `/api/categories` and stored in DB with `id` and `name`
- Current dependencies include `@google-cloud/vision`, `date-fns`, `sharp`

## Desired End State

After implementation:
1. Receipt images are processed by GPT-4o-mini via BAML + OpenRouter
2. Extraction returns: `date`, `amount`, `vendor`, `suggested_category`
3. `vendor` auto-fills the notes/description field
4. `suggested_category` pre-selects the category dropdown when matched
5. Fallback behavior: return whatever was parsed, let user fill blanks manually
6. Existing Google Cloud Vision code can be removed

### Verification
- Upload a receipt image with a non-standard date format -> date extracts correctly
- Upload a receipt without "TOTAL" keyword -> amount extracts correctly
- Upload a restaurant receipt -> category suggests "Food" or similar
- Vendor name appears in notes field automatically
- If VLM fails or returns partial data, user can still manually enter fields

## What We're NOT Doing

- Keeping Google Vision as a fallback (per user request)
- Extracting line items (not needed for current Receipt schema)
- Auto-submitting receipts (user confirmation still required)
- Changing the database schema (using existing fields)
- Modifying admin dashboard or batch review flows

## Implementation Approach

We'll use BAML to define a type-safe receipt extraction function that calls GPT-4o-mini via OpenRouter. BAML handles:
- Schema definition and validation
- Prompt engineering with structured output
- Type generation for TypeScript

OpenRouter serves as the model routing layer, providing:
- Unified API endpoint for multiple model providers
- Easy model switching (just change model name in config)
- Built-in fallbacks if primary model is unavailable
- Consolidated billing and usage tracking

The VLM sees the actual image (not OCR text), making it robust to formatting variations.

---

## Phase 1: Install BAML and Configure

### Overview
Set up BAML in the project and configure it for GPT-4o-mini.

### Changes Required

#### 1. Install BAML Package
**Command**:
```bash
cd dws-app && npm install @boundaryml/baml
```

#### 2. Initialize BAML
**Command**:
```bash
cd dws-app && npx baml-cli init
```

This creates:
- `baml_src/` directory for BAML source files
- `baml_client/` directory (auto-generated)

#### 3. Create BAML Configuration
**File**: `dws-app/baml_src/clients.baml`
```baml
// GPT-4o-mini client via OpenRouter for receipt extraction
// OpenRouter provides unified access to multiple model providers
client<llm> GPT4oMini {
  provider openai
  options {
    model "openai/gpt-4o-mini"
    api_key env.OPENROUTER_API_KEY
    base_url "https://openrouter.ai/api/v1"
    temperature 0
    headers {
      "HTTP-Referer" "https://dws-receipts.vercel.app"
      "X-Title" "DWS Receipts"
    }
  }
}
```

**Note**: OpenRouter uses an OpenAI-compatible API, so we use the `openai` provider with a custom `base_url`. The model name uses OpenRouter's format: `provider/model-name`. The headers help OpenRouter track usage and are recommended by their documentation.

#### 4. Add Environment Variable
**File**: `dws-app/.env.local` (add line)
```
OPENROUTER_API_KEY=your_openrouter_api_key_here
```

Get your API key from https://openrouter.ai/keys

#### 5. Update .gitignore
**File**: `dws-app/.gitignore` (add if not present)
```
# BAML generated files
baml_client/
```

### Success Criteria

#### Automated Verification:
- [ ] Package installed: `cd dws-app && npm list @boundaryml/baml`
- [ ] BAML directory exists: `ls dws-app/baml_src/`
- [ ] No build errors: `cd dws-app && npm run build`

#### Manual Verification:
- [ ] `.env.local` has `OPENROUTER_API_KEY` placeholder

**Implementation Note**: After completing this phase, pause for confirmation before proceeding.

---

## Phase 2: Define BAML Receipt Schema and Function

### Overview
Create the BAML schema for receipt extraction with all relevant fields.

### Changes Required

#### 1. Create Receipt Extraction Schema
**File**: `dws-app/baml_src/receipts.baml`
```baml
// Receipt data extracted from image
class ExtractedReceipt {
  date string? @description("Receipt date in YYYY-MM-DD format. Extract the purchase/transaction date, not print date or expiry dates.")
  amount float? @description("Total amount paid after tax and tip. Look for TOTAL, GRAND TOTAL, AMOUNT DUE, or the final/largest amount.")
  vendor string? @description("Store or business name. Usually at the top of the receipt.")
  suggested_category string? @description("One of: Food, Gas, Office, Travel, Parking, Medical, Entertainment, Utilities, Other. Infer from vendor name and items if possible.")
}

// Main extraction function
function ExtractReceiptFromImage(img: image) -> ExtractedReceipt {
  client GPT4oMini
  prompt #"
    {{_.role("user")}}

    Extract receipt details from this image:
    {{ img }}

    Instructions:
    - date: Format as YYYY-MM-DD. Use the purchase/transaction date, not promotional or expiry dates. If year is unclear, assume current year.
    - amount: The final total amount paid (after tax, tip, discounts). Look for "TOTAL", "GRAND TOTAL", "AMOUNT DUE", "BALANCE DUE". If multiple totals exist, use the largest/final one.
    - vendor: The store or business name, usually at the top of the receipt.
    - suggested_category: Infer from context. Fast food, restaurants, groceries = Food. Gas stations = Gas. Office supplies = Office. Airlines, hotels, rideshare = Travel. Parking meters, garages = Parking. Pharmacies, medical = Medical. Movies, concerts = Entertainment. Bills, subscriptions = Utilities. Otherwise = Other.

    If a field cannot be determined, return null for that field.

    {{ ctx.output_format }}
  "#
}
```

#### 2. Configure Generator
**File**: `dws-app/baml_src/generators.baml`
```baml
generator target {
  output_type "typescript"
  output_dir "../baml_client"
  version "0.85.0"
}
```

#### 3. Generate TypeScript Client
**Command**:
```bash
cd dws-app && npx baml-cli generate
```

### Success Criteria

#### Automated Verification:
- [ ] BAML files exist: `ls dws-app/baml_src/*.baml`
- [ ] Client generated: `ls dws-app/baml_client/`
- [ ] No generation errors: `cd dws-app && npx baml-cli generate`
- [ ] Build passes: `cd dws-app && npm run build`

#### Manual Verification:
- [ ] Review generated types in `baml_client/` for correctness

**Implementation Note**: After completing this phase, pause for confirmation before proceeding.

---

## Phase 3: Replace OCR Route with BAML

### Overview
Replace the existing regex-based OCR route with a BAML-powered VLM extraction.

### Changes Required

#### 1. Replace OCR Route
**File**: `dws-app/src/app/api/receipts/ocr/route.ts`

Replace entire file with:
```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { createSupabaseServerClient } from '@/lib/supabaseServerClient';
import { b } from '@/baml_client';
import { Image } from '@boundaryml/baml';

export async function POST(request: NextRequest) {
  try {
    const supabase = await createSupabaseServerClient();
    const { data: { session }, error: sessionError } = await supabase.auth.getSession();

    if (sessionError || !session) {
      console.error('OCR API: Unauthorized - Session error or no session', sessionError);
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

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

    // Call BAML extraction function
    console.log('OCR API: Calling BAML ExtractReceiptFromImage...');
    const extracted = await b.ExtractReceiptFromImage(image);
    console.log('OCR API: BAML extraction result:', extracted);

    // Return extracted data in the expected format
    return NextResponse.json({
      success: true,
      data: {
        date: extracted.date || null,
        amount: extracted.amount || null,
        vendor: extracted.vendor || null,
        suggested_category: extracted.suggested_category || null,
      },
    });

  } catch (error: any) {
    console.error('OCR API: Unexpected error:', error);
    return NextResponse.json({
      error: 'Internal server error',
      details: error.message
    }, { status: 500 });
  }
}
```

### Success Criteria

#### Automated Verification:
- [ ] No TypeScript errors: `cd dws-app && npx tsc --noEmit`
- [ ] Build passes: `cd dws-app && npm run build`
- [ ] Lint passes: `cd dws-app && npm run lint`

#### Manual Verification:
- [ ] Upload a receipt image and verify extraction returns data
- [ ] Check console logs for BAML extraction result
- [ ] Verify date format is YYYY-MM-DD
- [ ] Verify amount is a number (not string)

**Implementation Note**: After completing this phase and automated verification, pause for manual testing before proceeding.

---

## Phase 4: Update Frontend to Use New Fields

### Overview
Update the receipt uploader and details card to utilize the new `vendor` and `suggested_category` fields.

### Changes Required

#### 1. Update Receipt Uploader State
**File**: `dws-app/src/components/receipt-uploader.tsx`

**Change 1**: Update extractedData state (around line 212-217):
```typescript
// OLD:
setExtractedData({
  receipt_date: ocrResult.data.date || undefined,
  amount: ocrResult.data.amount !== null ? ocrResult.data.amount : undefined,
})

// NEW:
setExtractedData({
  receipt_date: ocrResult.data.date || undefined,
  amount: ocrResult.data.amount !== null ? ocrResult.data.amount : undefined,
  notes: ocrResult.data.vendor || undefined, // Auto-fill notes with vendor
  suggested_category: ocrResult.data.suggested_category || undefined,
})
```

#### 2. Update Receipt Type (temporary addition)
**File**: `dws-app/src/lib/types.ts`

**Add field to Receipt interface** (around line 14, after `notes`):
```typescript
suggested_category?: string; // VLM-suggested category name for auto-selection
```

#### 3. Update Receipt Details Card
**File**: `dws-app/src/components/receipt-details-card.tsx`

**Change 1**: Initialize notes from initialData (already done, but ensure vendor flows through)
The current code already uses `initialData?.notes` for the notes field, so vendor will auto-populate.

**Change 2**: Auto-select category based on suggestion (in useEffect, around line 87-97):
```typescript
// After fetching categories, check for suggested_category match
if (initialData?.suggested_category && data.categories.length > 0 && categoryId === '') {
  const suggestedCat = data.categories.find(
    (cat: CategoryType) => cat.name.toLowerCase() === initialData.suggested_category?.toLowerCase()
  );
  if (suggestedCat) {
    setCategoryId(suggestedCat.id);
    console.log('Auto-selected category from VLM suggestion:', suggestedCat.name);
  }
}
```

**Change 3**: Add suggested_category to useEffect dependency (around line 107):
```typescript
}, [initialData?.category_id, initialData?.receipt_date, initialData?.date, initialData?.suggested_category]);
```

### Success Criteria

#### Automated Verification:
- [ ] No TypeScript errors: `cd dws-app && npx tsc --noEmit`
- [ ] Build passes: `cd dws-app && npm run build`
- [ ] Lint passes: `cd dws-app && npm run lint`

#### Manual Verification:
- [ ] Upload a restaurant receipt -> notes field shows restaurant name
- [ ] Upload a gas station receipt -> category auto-selects "Gas" (if category exists)
- [ ] Upload a receipt where category can't be determined -> category remains unselected
- [ ] User can still edit all auto-filled fields

**Implementation Note**: After completing this phase, pause for comprehensive manual testing.

---

## Phase 5: Cleanup and Documentation

### Overview
Remove unused Google Vision dependencies and document the new system.

### Changes Required

#### 1. Remove Google Vision Dependency (Optional)
**Command**:
```bash
cd dws-app && npm uninstall @google-cloud/vision
```

**Note**: Only do this after confirming BAML is working in production. You may want to keep it temporarily for comparison testing.

#### 2. Remove Unused date-fns Imports from Old Route
The new route doesn't need the extensive date-fns imports. These are already removed by the route replacement.

#### 3. Update Environment Variables Documentation
**File**: `dws-app/.env.example` (create or update)
```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key

# OpenRouter (for BAML VLM receipt extraction via model routing)
# Get your API key from https://openrouter.ai/keys
OPENROUTER_API_KEY=your_openrouter_api_key

# Legacy (can be removed after BAML migration)
# GOOGLE_APPLICATION_CREDENTIALS_JSON=your_google_credentials_json
```

#### 4. Remove Legacy Google Credentials (After Verification)
Remove `GOOGLE_APPLICATION_CREDENTIALS_JSON` from `.env.local` and Vercel environment variables after confirming BAML works in production.

### Success Criteria

#### Automated Verification:
- [ ] Build passes without Google Vision: `cd dws-app && npm run build`
- [ ] No unused dependency warnings

#### Manual Verification:
- [ ] Full end-to-end test: upload receipt -> extraction -> submission -> view in dashboard
- [ ] Test on mobile device
- [ ] Test with various receipt types (restaurant, gas, office supplies, etc.)

---

## Testing Strategy

### Unit Tests (Future Enhancement)
- Mock BAML client responses
- Test category matching logic
- Test error handling when VLM returns null fields

### Integration Tests (Future Enhancement)
- Test full upload -> extract -> submit flow
- Test with various image formats (JPEG, PNG, HEIC, PDF)

### Manual Testing Steps
1. **Standard receipt**: Upload a clear grocery store receipt
   - Verify: date, amount, vendor name, category suggestion
2. **Restaurant receipt with tip**: Upload a receipt with tip line
   - Verify: total amount includes tip
3. **Non-English receipt**: Upload receipt with non-English text
   - Verify: extraction still works or gracefully returns nulls
4. **Blurry image**: Upload a low-quality photo
   - Verify: partial extraction or graceful failure
5. **PDF receipt**: Upload a PDF receipt
   - Verify: extraction works on PDFs
6. **Mobile capture**: Take a photo on mobile device
   - Verify: HEIC format handled correctly

## Performance Considerations

- **Latency**: GPT-4o-mini via OpenRouter is typically 1-3 seconds per image (similar to current Google Vision + parsing). OpenRouter adds minimal overhead (~50-100ms).
- **Cost**: ~$0.001-0.003 per receipt via OpenRouter (comparable to or less than current Google Vision costs). OpenRouter may have a small markup over direct API pricing but provides flexibility.
- **Reliability**: VLM is more robust to formatting variations than regex parsing
- **Model Flexibility**: With OpenRouter, you can easily switch to other models (e.g., `anthropic/claude-3-5-sonnet`, `google/gemini-2.0-flash`) by changing just the model name in `clients.baml`

## Migration Notes

- **Rollback**: Keep Google Vision code commented out for first 2 weeks
- **A/B Testing**: Could implement feature flag to compare approaches (not in scope)
- **Monitoring**: Watch for extraction failure rates in Vercel logs

## References

- BAML Documentation: https://docs.boundaryml.com/home
- BAML TypeScript Guide: https://docs.boundaryml.com/guide/installation-language/typescript
- OpenRouter Documentation: https://openrouter.ai/docs
- OpenRouter API Keys: https://openrouter.ai/keys
- OpenRouter Models: https://openrouter.ai/models (GPT-4o-mini: `openai/gpt-4o-mini`)
- Current OCR implementation: `dws-app/src/app/api/receipts/ocr/route.ts`
- Receipt uploader: `dws-app/src/components/receipt-uploader.tsx`
- Receipt details card: `dws-app/src/components/receipt-details-card.tsx`
