---
date: 2025-12-19T09:38:40Z
researcher: Claude
git_commit: N/A
branch: N/A
repository: dwsERP
topic: "Expand Station System & Blocker Categories Implementation Plan"
tags: [implementation, planning, stations, blockers, shop-floor, logistics]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
type: implementation_strategy
---

# Handoff: Expand Stations & Blocker System Plan

## Task(s)

| Task | Status |
|------|--------|
| Create implementation plan to expand station system from 5 to 12 stations | **Complete** |
| Define expanded blocker categories for external dependencies | **Complete** |
| Integrate with existing blocker management plan | **Complete** |
| Document future PM view requirements | **Complete** |
| Document owner time estimate feature (future) | **Discussed only** |

This session focused on **planning only** - no code was implemented.

## Critical References

1. `.claude/plans/2025-12-19-expand-stations.md` - The main implementation plan created this session
2. `.claude/plans/2025-12-19-sheet-sidebar-blocker-management.md` - Related blocker plan (pre-existing)
3. `factory-floor/src/data/mockShopData.ts` - Core data types that will be modified

## Recent changes

No code changes - this was a planning session. Created:
- `.claude/plans/2025-12-19-expand-stations.md` (new plan document)

## Learnings

### Station Model
- **12 stations** organized into 3 zones:
  - **Shop (7)**: drafting, cnc, edgeBanding, veneer, milling, finishing, bench
  - **External (1)**: outsourced (sheet physically left the building)
  - **Logistics (4)**: staging, delivery, fieldInstall, punchList

### Blocker Model
- External blockers are the biggest source of delays
- **9 blocker categories**: equipment, labor, quality, architect, client, vendor, subcontractor, site, other
- Key distinction: `outsourced` station = sheet physically at external shop; `subcontractor` blocker = sheet in-house waiting on external parts

### Key Insight
- Stations = where work physically is
- Blockers = who/what is stopping progress (tells PM who to call)

## Artifacts

- `.claude/plans/2025-12-19-expand-stations.md` - Full implementation plan with 4 phases

## Action Items & Next Steps

1. **Implement Phase 1**: Update `StationId` type, `OWNERS` map, and mock data in `mockShopData.ts`
2. **Implement Phase 2**: Update Dashboard layout with 3-row design (shop/external/logistics)
3. **Implement Phase 3**: Update Editor palette with all 12 operation types
4. **Implement Phase 4**: Update PyJobShop export with new resource mappings
5. **After stations**: Implement blocker management plan (`.claude/plans/2025-12-19-sheet-sidebar-blocker-management.md`)
6. **Future**: Create plan for owner time estimates feature (see below)

## Future Feature: Owner Time Estimates

User wants to add a feature where **each owner can estimate how long their work will take**:

- Owners set their own time estimates when picking up a sheet
- System tracks estimate vs actual completion time
- Creates "low-key accountability" - no one else sets their deadlines, but there's a record
- Over time, patterns emerge: who's accurate, who's optimistic, who's pessimistic
- Feeds into PM view for better project planning and follow-up

**Not yet planned** - should be a separate plan after stations/blockers are implemented.

## Other Notes

### Dashboard Layout
The plan specifies a 3-row layout:
- Row 1 (y: 0): Shop stations spread horizontally
- Row 2 (y: 350): Outsourced station (centered, visually distinct)
- Row 3 (y: 700): Logistics stations

### Non-Linear Flow
The workflow is explicitly non-linear. Sheets can:
- Go back to previous stations (veneer → bench → mill → veneer)
- Skip stations entirely
- Ship in phases

The edges in the Dashboard show common paths but don't enforce linear flow.

### PM View (Future)
The blocker categories are designed to feed a future PM dashboard that answers:
- "What's blocked by architects?" → follow up on drawings
- "What's blocked by clients?" → schedule decision meeting
- "What's our $ exposure to vendor delays?" → consider alternates
- "Which projects are stuck on site?" → escalate to GC
