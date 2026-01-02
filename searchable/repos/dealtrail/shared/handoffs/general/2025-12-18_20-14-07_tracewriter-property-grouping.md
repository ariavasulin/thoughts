---
date: 2025-12-19T04:14:07+0000
researcher: claude
git_commit: 5137ec6af499fe9bea3e843194fe4a744d7ca1d3
branch: main
repository: Dealtrail
topic: "TraceWriter Email Data Cleanup - Property Grouping Strategy"
tags: [data-cleanup, email-processing, tracewriter, mbox, threading, property-grouping]
status: ready_for_implementation
last_updated: 2025-12-18
last_updated_by: claude
type: implementation_strategy
---

# Handoff: TraceWriter Property-Based Email Grouping

## Task(s)

**Completed:** Deep analysis of email threading issues and development of property-based grouping strategy.

**Ready for Implementation:** Modify `mbox_to_json.py` to group emails by property address instead of email threading headers.

The previous handoff (`2025-12-18_19-57-55_tracewriter-email-cleanup.md`) identified that emails weren't grouping well. This session diagnosed the root cause and developed a solution.

## Critical References

- `plans/2024-12-16-tracewriter-email-annotation-tool.md` - Full implementation plan (Phase 2 marked complete)
- `tracewriter/scripts/mbox_to_json.py` - Script to modify with new grouping strategy
- `thoughts/shared/handoffs/general/2025-12-18_19-57-55_tracewriter-email-cleanup.md` - Previous handoff with Phase 2 context

## Recent changes

- `tracewriter/scripts/analyze_unmatched_emails.py:1-170` - Created analysis script for understanding unmatched emails

## Learnings

1. **Root Cause of Poor Threading**: The current script groups by email headers (References/In-Reply-To), but transaction coordinators send many separate emails to different parties (buyer, seller, lender, listing agent, escrow) about the same property. These get scattered into separate "threads" because they're not email replies to each other.

2. **The Real Grouping Key is Property Address**: Emails should be grouped by property/transaction, not by email threading. Example: "7250 Franklin Ave" has 67 scattered threads that should be 1 transaction timeline with 180 emails.

3. **Matching Strategy Coverage**:
   - 68.7% (956 threads) have clear address in subject
   - 8.5% (118 threads) have address in email body only
   - 13.4% (187 threads) use property nicknames (e.g., "Franklin" for "7250 Franklin Ave")
   - **90.7% total matchable** (1261/1391 threads)
   - 9.3% truly unmatched (130 threads) - mostly administrative emails

4. **Property Nickname Mapping** (discovered from data analysis):
   ```python
   nickname_map = {
       'holly': '5693 holly oak',
       'franklin': '7250 franklin',
       'shetland': '12233 shetland',
       'cherokee': '746 n cherokee',
       'gretna': '321 s gretna green',
       'doheny': '818 n doheny',
       'century': '1 w century',
       'olympic': '8844 olympic',
       'magnolia': '11675 magnolia',
       'bosque': '16908 bosque',
       'kings': '118 n kings',
       'columbus': '4730 columbus',
       'sunnyslope': '4740 sunnyslope',
       'knowlton': '7127 knowlton',
       'hilldale': '1222 hilldale',
       'tower': '1571 tower',
       'castle': '435 castle',
       'newcastle': '5339 newcastle',
       'rimpau': '242 s rimpau',
       'coldwater': '3506 coldwater',
       'marina': '13700 marina',
       'loma': '5330 loma linda',
       'vicente': '1007 n san vicente',
       'lucerne': 'lucerne',
       'alta': '611 n alta'
   }
   ```

5. **Address Normalization Required**:
   - "7250 franklin avenue unit 1110", "7250 franklin ave #1110", "7250 franklin" all = same property
   - Must normalize: Ave/Avenue, Dr/Drive, Unit/#/Apt, collapse whitespace

## Artifacts

- `tracewriter/scripts/analyze_unmatched_emails.py` - Analysis script (can be deleted after implementation)
- `tracewriter/threads.json` - Current output (22MB, 1391 threads, 3745 emails)
- `Mail/Transaction Coordinator Emails.mbox` - Source MBOX file (3.5GB)

## Action Items & Next Steps

1. **Modify `mbox_to_json.py`** to implement new grouping strategy:
   - Add `normalize_address()` function for address standardization
   - Add `extract_property()` function that checks: subject → body → nickname map
   - Replace `get_thread_id()` with property-based grouping
   - Flatten all emails for a property into one transaction timeline
   - Sort chronologically within each transaction

2. **Add email body cleanup**:
   - Strip signature blocks (detect "THE*AGENCY*", phone numbers, email footers)
   - Remove quoted reply chains (lines starting with `>`, "On ... wrote:" blocks)
   - Remove `[image: ...]` placeholders

3. **Re-run preprocessing** on the MBOX file

4. **Test in TraceWriter UI** - verify transactions display as coherent timelines

## Other Notes

- The goal is to see complete transaction lifecycles, not scattered email threads
- Example: 7250 Franklin Ave transaction spans Feb 11 - Mar 14, 2022 with emails to buyer (Coleman King), lender (Connie Ramirez), listing agent (Martha Blake), escrow team, and internal VA team
- The 130 truly unmatched emails (9.3%) are mostly: Out of Office (16), invoices, "No Subject" (6), generic DocuSign completions, calendar invites - these don't belong to specific transactions
- Phases 3-5 of the implementation plan (localStorage, export, UX polish) are blocked on clean data
