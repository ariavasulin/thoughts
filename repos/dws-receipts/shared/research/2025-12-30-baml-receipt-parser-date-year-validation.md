---
date: 2025-12-30T13:13:44-08:00
researcher: ariasulin
git_commit: 3ef7e7d7caf88b7ae16f80ce3f05acaee8bfcc32
branch: main
repository: DWS-Receipts
topic: "BAML Receipt Parser Date Year Validation"
tags: [research, codebase, baml, receipts, ocr, date-validation]
status: complete
last_updated: 2025-12-30
last_updated_by: ariasulin
---

# Research: BAML Receipt Parser Date Year Validation

**Date**: 2025-12-30T13:13:44-08:00
**Researcher**: ariasulin
**Git Commit**: 3ef7e7d7caf88b7ae16f80ce3f05acaee8bfcc32
**Branch**: main
**Repository**: DWS-Receipts

## Research Question
How does the BAML receipt parser handle date extraction, and where can year validation (restricting to 2025-2026) be added?

## Summary

The BAML receipt parser is defined in `dws-app/baml_src/receipts.baml`. The `ExtractedReceipt` class has a `date` field that extracts dates in YYYY-MM-DD format. Currently, there is **no year validation** - the LLM is instructed to assume the current year if unclear, but any year can be returned.

To restrict dates to 2025-2026 only, the most effective approach is to add a `@check` decorator to the date field in the BAML schema, which will validate the extracted year and report whether the check passed or failed.

## Detailed Findings

### BAML Schema Location

The receipt parser is defined in a single file:
- `dws-app/baml_src/receipts.baml` - Contains the `ExtractedReceipt` class and `ExtractReceiptFromImage` function

### Current Date Field Definition

The date field is defined at line 3:
```baml
date string? @description("Receipt date in YYYY-MM-DD format. Extract the purchase/transaction date, not print date or expiry dates.")
```

### Current Prompt Instructions

The prompt instructions for date extraction (lines 17-18):
```
- date: Format as YYYY-MM-DD. Use the purchase/transaction date, not promotional or expiry dates. If year is unclear, assume current year.
```

### How Date Flows Through the System

1. **BAML Extraction** (`baml_src/receipts.baml:9-32`): `ExtractReceiptFromImage` function calls GPT-4.1-nano via OpenRouter to extract date from image
2. **OCR API** (`src/app/api/receipts/ocr/route.ts:56`): Calls `b.ExtractReceiptFromImage(image)` and returns `extracted.date`
3. **Frontend** (`src/components/receipt-uploader.tsx`): Receives date from OCR API and maps to `receipt_date` field
4. **Database**: Stores as `receipt_date` column in `receipts` table

### Generated TypeScript Types

The generated type at `dws-app/baml_client/types.ts:50-55`:
```typescript
export interface ExtractedReceipt {
  date?: string | null
  amount?: number | null
  category?: string | null
}
```

### BAML Check Feature

BAML supports the `@check` decorator for field validation. When a check fails, the value is still returned but wrapped in a `Checked<T>` type that includes check status. This allows the application to decide how to handle invalid years.

## Code References

- `dws-app/baml_src/receipts.baml:3` - Date field definition
- `dws-app/baml_src/receipts.baml:9-32` - ExtractReceiptFromImage function
- `dws-app/baml_src/receipts.baml:18` - Prompt instructions for date
- `dws-app/baml_src/clients.baml:6-18` - GPT4oMini client configuration (uses OpenRouter)
- `dws-app/src/app/api/receipts/ocr/route.ts:56` - BAML function call
- `dws-app/baml_client/types.ts:50-55` - Generated ExtractedReceipt interface

## Architecture Documentation

### BAML File Structure
```
dws-app/
├── baml_src/           # Source BAML files (edit these)
│   ├── receipts.baml   # Receipt extraction schema
│   ├── clients.baml    # LLM client configurations
│   └── generators.baml # Code generation settings
└── baml_client/        # Generated TypeScript (auto-generated, do not edit)
    ├── types.ts        # Generated interfaces
    ├── async_client.ts # Async client
    └── ...
```

### Implementation Options for Year Validation

**Option 1: BAML @check decorator** (Recommended)
Add validation directly in the BAML schema using the `@check` decorator. This validates at the BAML layer and reports check status.

**Option 2: Prompt modification**
Update the prompt instructions to explicitly reject dates outside 2025-2026. Less reliable as LLMs may not follow instructions perfectly.

**Option 3: Post-processing validation**
Add validation in `src/app/api/receipts/ocr/route.ts` after BAML extraction. Works but adds another validation layer.

## Historical Context (from thoughts/)

No prior research documents found specifically about BAML date validation.

## Related Research

- `thoughts/shared/plans/2025-12-17-baml-vlm-receipt-parsing.md` - Original BAML implementation plan
- `thoughts/shared/research/2025-12-18-receipt-upload-flow-and-confirmation.md` - Receipt upload flow research

## Open Questions

1. Should invalid year dates return null, or return the date with a failed check status?
2. Should the frontend show a warning for dates outside 2025-2026, or reject them entirely?
