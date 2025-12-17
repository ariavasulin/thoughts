---
date: 2025-12-17T12:45:56Z
researcher: Claude
git_commit: a89e3ffdb1d816894e7817effc94e86cca4a4084
branch: receipt-editing
repository: DWS-Receipts
topic: "BAML + VLM Receipt Parsing Implementation"
tags: [implementation, baml, vlm, ocr, receipt-parsing]
status: in_progress
last_updated: 2025-12-17
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: BAML + VLM Receipt Parsing (Phase 1 Complete)

## Task(s)

**Working from**: `thoughts/shared/plans/2025-12-17-baml-vlm-receipt-parsing.md`

Replace Google Cloud Vision API + regex-based receipt parsing with BAML + GPT-4o-mini VLM via OpenRouter for more robust, accurate receipt data extraction.

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | **COMPLETED** | Install BAML and configure |
| Phase 2 | PENDING | Define BAML receipt schema and function |
| Phase 3 | PENDING | Replace OCR route with BAML |
| Phase 4 | PENDING | Update frontend to use new fields (vendor, suggested_category) |
| Phase 5 | PENDING | Cleanup and documentation |

## Critical References

1. `thoughts/shared/plans/2025-12-17-baml-vlm-receipt-parsing.md` - Full implementation plan
2. `dws-app/src/app/api/receipts/ocr/route.ts` - Current OCR route to be replaced (233 lines)
3. `dws-app/src/components/receipt-uploader.tsx:212-217` - Where extracted data is set

## Recent changes

- `dws-app/package.json` - Added `@boundaryml/baml@0.214.0` dependency
- `dws-app/baml_src/clients.baml` - Added GPT4oMini client via OpenRouter with `openai-generic` provider
- `dws-app/baml_src/generators.baml` - Auto-generated with version 0.214.0
- `dws-app/.gitignore:55-56` - Added `baml_client/` to gitignore
- Removed `dws-app/baml_src/resume.baml` - Deleted example file

## Learnings

1. **BAML version mismatch**: Plan specified v0.85.0 but npm installed v0.214.0. Updated generators.baml accordingly.

2. **Provider type for OpenRouter**: Validation report correctly identified that `provider openai` should be `provider "openai-generic"` for non-OpenAI endpoints. Applied this fix in clients.baml.

3. **Peer dependency conflict**: React 19 conflicts with react-day-picker. Used `--legacy-peer-deps` flag to install BAML.

4. **baml_client output location**: With `output_dir "../"` in generators.baml, the baml_client is generated in `dws-app/` (not inside `src/`). Import paths will need to account for this.

## Artifacts

- `thoughts/shared/plans/2025-12-17-baml-vlm-receipt-parsing.md` - Updated Phase 1 success criteria as complete
- `dws-app/baml_src/clients.baml` - OpenRouter GPT-4o-mini client configuration
- `dws-app/baml_src/generators.baml` - TypeScript generator config (v0.214.0)

## Action Items & Next Steps

1. **MANUAL VERIFICATION REQUIRED**: User must add `OPENROUTER_API_KEY=<key>` to `dws-app/.env.local`
   - Get API key from https://openrouter.ai/keys

2. **Phase 2**: Create receipt extraction schema
   - Create `dws-app/baml_src/receipts.baml` with ExtractedReceipt class and ExtractReceiptFromImage function
   - Run `npx baml-cli generate` to generate TypeScript client
   - Verify generated types in `baml_client/`

3. **Phase 3**: Replace OCR route
   - Replace `dws-app/src/app/api/receipts/ocr/route.ts` with BAML-based extraction
   - Note: Import path should be `import { b } from '../../baml_client'` or configure tsconfig alias

4. **Phase 4**: Update frontend
   - `receipt-uploader.tsx:212-217` - Add `notes` and `suggested_category` to extractedData
   - `types.ts:14` - Add `suggested_category?: string` to Receipt interface
   - `receipt-details-card.tsx:87-97` - Add category auto-selection logic

5. **Phase 5**: Cleanup
   - Consider removing `@google-cloud/vision` dependency after production verification
   - Update `.env.example` with OpenRouter instructions

## Other Notes

- **PDF support**: Per validation report, PDF support via GPT-4o-mini vision is unverified. The plan includes PDF in mediaTypeMap but this should be tested explicitly.
- **Rate limiting**: No retry logic beyond BAML's built-in Exponential retry_policy. Consider adding explicit handling for 429 responses in production.
- **Current OCR response format**: `{ success: true, data: { date: string, amount: number } }` - New format adds `vendor` and `suggested_category` fields.
