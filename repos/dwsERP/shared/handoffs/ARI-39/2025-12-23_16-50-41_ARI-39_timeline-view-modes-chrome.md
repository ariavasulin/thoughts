---
date: 2025-12-23T16:50:41-08:00
researcher: ariasulin
git_commit: 2660d75535156917eaa5c961be1e74917e090158
branch: main
repository: dwsERP
topic: "ARI-39 Timeline View Modes & Chrome Implementation Strategy"
tags: [implementation, strategy, timeline, view-modes, chrome-consolidation]
status: complete
last_updated: 2025-12-23
last_updated_by: ariasulin
type: implementation_strategy
---

# Handoff: ARI-39 Timeline View Modes & Chrome

## Task(s)

| Task | Status |
|------|--------|
| Create implementation plan for ARI-39 | Completed |
| Phase 1: Add Timeline to Command Palette | Planned |
| Phase 2: Implement View Mode Zoom Levels | Planned |
| Phase 3: Consolidate Chrome (Remove Duplicates) | Planned |

Created a comprehensive implementation plan for making Timeline view modes functional and consolidating redundant chrome. The plan is ready for implementation - no code changes made yet.

## Critical References

- Implementation plan: `thoughts/shared/plans/2025-12-23-ARI-39-timeline-view-modes-chrome.md`
- Research document: `thoughts/searchable/shared/research/2025-12-23-ARI-39-timeline-view-modes-chrome.md`

## Recent changes

No code changes made. Planning session only.

## Learnings

1. **TimelineView renders inside Layout** (`millflow/src/App.tsx:46-48`) but has its own header/footer, creating ~76px of redundant chrome.

2. **ShortcutBar already exists** (`millflow/src/components/ShortcutBar.tsx`) as a context-aware footer that shows different shortcuts per view - but it's missing `timeline` and `schedule` cases (falls back to dashboard shortcuts).

3. **View mode state exists but is unused** (`millflow/src/views/TimelineView.tsx:25`): `viewMode` is set by toggle buttons but `DAY_WIDTH = 24` is constant. The fix is simple - derive `DAY_WIDTH` from `viewMode` using `useMemo`.

4. **Command Palette missing Timeline** (`millflow/src/components/CommandPalette.tsx:37-99`): Has Dashboard, Jobs, Departments, Schedule but no Timeline. The `goToTimeline()` store method exists and `g t` keyboard shortcut works.

## Artifacts

- `thoughts/shared/plans/2025-12-23-ARI-39-timeline-view-modes-chrome.md` - Complete implementation plan with 3 phases

## Action Items & Next Steps

1. **Implement Phase 1**: Add "Go to Timeline" to CommandPalette with `Clock` icon and `g t` subtitle
   - File: `millflow/src/components/CommandPalette.tsx`

2. **Implement Phase 2**: Make view mode zoom functional
   - File: `millflow/src/views/TimelineView.tsx`
   - Replace constant `DAY_WIDTH = 24` with `useMemo` deriving from `viewMode`
   - Day: 72px, Week: 36px, Month: 24px

3. **Implement Phase 3**: Consolidate chrome
   - Add `timelineShortcuts` and `scheduleShortcuts` to `ShortcutBar.tsx`
   - Replace TimelineView header (lines 133-170) with compact toolbar (~28px)
   - Remove TimelineView footer (lines 413-432)

4. **Verify**: Run `npm run build` and `npm run lint` after each phase

## Other Notes

- Linear ticket: ARI-39 (Ariav team, DWS PMDB2 project)
- Zoom level values (Day=72px, Week=36px, Month=24px) were approved by user
- User explicitly wants footer consolidated into ShortcutBar, not removed entirely
