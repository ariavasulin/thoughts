---
date: 2025-12-19T03:57:55+0000
researcher: claude
git_commit: 5137ec6af499fe9bea3e843194fe4a744d7ca1d3
branch: main
repository: Dealtrail
topic: "TraceWriter Email Data Cleanup"
tags: [data-cleanup, email-processing, tracewriter, mbox, threading]
status: ready_for_work
last_updated: 2025-12-18
last_updated_by: claude
type: data_cleanup
---

# Handoff: TraceWriter Email Data Cleanup

## Task(s)

**Completed:** Phase 2 of TraceWriter implementation - MBOX preprocessing pipeline is working. The Python script successfully converts the 3.5GB MBOX file into JSON format that TraceWriter can import.

**Ready for Work:** Email data cleanup. The core functionality works, but the actual email data needs significant cleanup:
- Emails are not properly grouped into meaningful threads
- Email bodies may have formatting issues
- Threading algorithm uses References/In-Reply-To headers with subject-based fallback, but results are messy

## Critical References

- `plans/2024-12-16-tracewriter-email-annotation-tool.md` - Full implementation plan (Phase 2 marked complete)
- `tracewriter/scripts/mbox_to_json.py` - The Python preprocessing script that needs modifications

## Recent changes

- `tracewriter/scripts/mbox_to_json.py:1-480` - Created Python MBOX→JSON converter
- `tracewriter/src/utils/emailParser.js:1-85` - Created browser-side JSON parser
- `tracewriter/src/App.jsx:3` - Added emailParser import
- `tracewriter/src/App.jsx:170-184` - Implemented handleImport function

## Learnings

1. **Current threading approach** (`mbox_to_json.py:124-149`):
   - Uses `References` header first (takes first message ID as thread root)
   - Falls back to `In-Reply-To` header
   - Falls back to subject-based threading (strips Re:/Fwd: prefixes)
   - This produces 1391 threads from 3745 emails, but groupings are not ideal

2. **Data statistics**:
   - Source: `Mail/Transaction Coordinator Emails.mbox` (3.5GB)
   - Output: `tracewriter/threads.json` (22MB)
   - 1391 threads, 3745 emails
   - 359 threads have >2 emails
   - Only 1 email has no body content

3. **Email body extraction** (`mbox_to_json.py:37-82`):
   - Prefers plain text over HTML
   - Falls back to HTML→text conversion via regex
   - Bodies are readable but may need further cleanup

## Artifacts

- `tracewriter/threads.json` - Generated JSON file with all email threads (22MB)
- `tracewriter/scripts/mbox_to_json.py` - Python preprocessing script
- `tracewriter/src/utils/emailParser.js` - Browser-side parser
- `Mail/Transaction Coordinator Emails.mbox` - Source MBOX file

## Action Items & Next Steps

1. **Analyze threading quality** - Review `threads.json` to understand why threads aren't grouping well
2. **Improve threading algorithm** - May need to:
   - Use different header combinations
   - Implement better subject normalization
   - Consider sender/recipient patterns
   - Group by transaction/property address
3. **Clean up email bodies** - Remove:
   - Signature blocks
   - Quoted reply chains
   - Excess whitespace
   - Image placeholders like `[image: ...]`
4. **Validate results** - Re-run preprocessing and test in TraceWriter UI

## Other Notes

- The TraceWriter UI works correctly - import/display/navigation all function
- Phases 3-5 (localStorage, export, UX polish) are ready to implement once data is clean
- The Python script uses only stdlib (`mailbox`, `email`, `json`) - no external dependencies
- Consider creating a separate cleanup script vs modifying `mbox_to_json.py`
