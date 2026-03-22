---
date: 2026-03-21T17:30:00-07:00
researcher: Claude
git_commit: no-commits (uncommitted repo)
branch: main
repository: dwsTables
topic: "Shop Schedule View: Production Workflow at Design Workshops"
tags: [research, codebase, tblSheetNEW, qrySheetList, production-workflow, shop-schedule, tblDefaultSheetProc]
status: complete
last_updated: 2026-03-21
last_updated_by: Claude
---

# Research: Shop Schedule View — Production Workflow at Design Workshops

**Date**: 2026-03-21T17:30:00-07:00
**Researcher**: Claude
**Git Commit**: no-commits (uncommitted repo)
**Branch**: main
**Repository**: dwsTables

## Research Question
Find the shop schedule view from Access. What exactly does it reveal about the production workflow at Design Workshops?

## Summary

There is no single query named "shop schedule" in the Access database. Instead, the shop schedule is implemented as a **family of 30+ queries** all built on a central master query called **`qrySheetList`** (68 calculated fields, 5 JOINs). This query is the backbone of DWS's production tracking system — it transforms the raw `tblSheetNEW` table (117 columns, 15,932 rows) into a real-time status dashboard showing where every shop drawing sits in a **17-stage production pipeline**.

The production workflow revealed is a linear state machine tracking each "sheet" (a shop drawing representing a piece of millwork) through:

```
Draft → Submit → Approve → Measure → Correct → Stock/Program → Release to Shop →
  Veneer → Mill → Saw → CNC → Other → Bench → Finish → Assembly →
    Storage → Field Install → Complete
```

Each stage can be **done**, **in progress**, **skipped**, **overdue**, or **future**. The system was designed around a weekly **Monday Report** (hence the field `MondayRptNo`) where management reviews every active sheet's position in the pipeline.

## Detailed Findings

### 1. The Core Query: `qrySheetList`

**Location**: `metadata/access/queries.csv:2270-2276`

This is the master scheduling query. It joins 4 tables and produces 68+ calculated fields from `tblSheetNEW`'s 117 raw columns:

```sql
FROM dbo_tblSheetNEW
INNER JOIN dbo_tblProject ON tblSheetNEW.ProjectID = tblProject.ProjectID
LEFT JOIN dbo_tblDefaultSheetProc ON tblSheetNEW.HoldDept = tblDefaultSheetProc.SheetProcess
LEFT JOIN dbo_tblDefaultSheetProc_2 ON tblSheetNEW.CurDept = tblDefaultSheetProc_2.MondayRptNo
LEFT JOIN dbo_tblContact ON tblProject.PM = tblContact.ContactID
WHERE tblSheetNEW.SheetNo Is Not Null
```

**Key calculated fields:**

| Field | Purpose |
|-------|---------|
| `C1` through `C15`, `CSub`, `CStor` | **Status indicator per stage** (0=done, 1=active/overdue, "2"=skipped, 3=future, 4=started) |
| `CurProc` | Current process name (resolved from `tblDefaultSheetProc`) |
| `Stage` | Lifecycle stage (1=Engineering, 2=Programming, 3=Shop, 4=Field, 5=Complete) |
| `EngDur` | Total engineering days (Draft + Approve + Measure + Correct + Stock + Program) |
| `ShopDur` | Total shop days (Veneer + Mill + Saw + CNC + Other + Bench + Finish) |
| `TotDur` | Grand total days (Eng + Shop + Field + Storage) |
| `Ready` | Boolean: sheet is queued at a station but work hasn't started |
| `ShopStarted` | Boolean: any shop process has begun |
| `NoShop` | Boolean: all shop stages skipped (field-install-only item) |
| `DeliveryStatusRed/Black` | Days early/late vs delivery target |
| `FinalStatusRed/Black` | Days early/late vs final completion target |
| `HoldDisp` | "HOLD" display text when sheet is on hold |
| `DeptDate` | Target date for current department |
| `sort1/sort2/sort3` | Hierarchical sheet number sorting (e.g., "3.2.1") |

### 2. The 17-Stage Production Pipeline

**Location**: `Tables/tblDefaultSheetProc.csv` (38 rows defining all processes)

Each stage in `tblSheetNEW` follows a consistent data pattern:

| Column Pattern | Example (Stage 12: Bench) | Purpose |
|---------------|---------------------------|---------|
| `{Name}Dt{N}` | `BenchDt12` | Target completion date |
| `{Name}Dur{N}` | `BenchDur12` | Allocated days |
| `{Name}Done{N}` | `BenchDone12` | Completion flag ("-1"=done, "0"=pending) |
| `{Name}Skip{N}` | `BenchSkip12` | Skip flag ("-1"=skip, "0"=include) |
| `{Name}Started` | `BenchStarted` | In-progress flag ("-1"=started, "0"=not) |

**Complete stage sequence with `MondayRptNo` mapping:**

| MondayRptNo | Process | Stage Group | Columns |
|-------------|---------|-------------|---------|
| 0 | Not Started | — | — |
| 1 | **Drafting** | Engineering | DraftDt1, DraftDur1, DraftDone1, DraftSkip1, DraftStart |
| 2 | **Submit/Re-Submit** | Engineering | SubmitDt1, SubmitDt2, SubmitDone, SubmitSkip, ResubStarted |
| 3 | **Approval** | Engineering | ApprDt2, ApprDur2, ApprDone2, ApprSkip2 |
| 4 | **Measure** | Engineering | MeasureDt3, MeasureDur3, MeasureDone3, MeasureSkip3, MeasStarted |
| 5 | **Correct** | Engineering | CorrectDt4, CorrectDur4, CorrectDone4, CorrectSkip4, CorStarted |
| 6 | **Stock/Program** | Programming | StockDt5, StockDur5, StockDone5, StockSkip5 |
| 7 | **Release to Shop** | Programming | PgmDur6, PgmDone6, PgmSkip6, ToShopAct |
| 8 | **Veneer** | Shop | VenDt7, VenDur7, VenDone7, VenSkip7, VenStarted |
| 9 | **Mill** | Shop | MillDt8, MillDur8, MillDone8, MillSkip8, MillStarted |
| 10 | **Saw** | Shop | SawDt9, SawDur9, SawDone9, SawSkip9, SawStarted |
| 11 | **CNC** | Shop | CNCDt10, CNCDur10, CNCDone10, CNCSkip10, CNCStarted |
| 12 | **Other** | Shop | OthDt11, OtherDur11, OthDone11, OthSkip11, OthStarted |
| 13 | **Bench** | Shop | BenchDt12, BenchDur12, BenchDone12, BenchSkip12, BenchStarted |
| 14 | **Finish** | Shop | FinishDt13, FinishDur13, FinishDone13, FinishSkip13, FinStarted |
| 15 | **Assembly** | Shop | AssemblyDt, AssemblyDur14, AssemblyDone14, AssemblySkip14, AssemStarted |
| 16 | **Storage** | Shop | StorDt, StorDur, StorDone, StorSkip |
| 17 | **Field** | Field | FieldDt, FieldDur, FieldDone, FieldSkip, FieldStarted |
| 18 | **Completed** | Complete | SheetCompleteDate, Delivered |

### 3. The Status Indicator System (C1-C15)

The most revealing part of the query is the **color-coded status system** computed for every stage:

```
0 = Done (green)      — stage completed
1 = Active/Overdue    — current stage OR target date has passed (red indicator)
"2" = Skipped         — stage bypassed for this sheet
3 = Future            — not yet reached, target date in future
4 = Started/In Progress — work has begun but not finished
```

The status logic cascades through conditions:
1. Is it done? → 0
2. Is it skipped? → "2"
3. Has the Monday Report number reached this dept? → 1 (active)
4. Has work started? → 4 (in progress)
5. Is the target date past? → 1 (overdue)
6. Otherwise → 3 (future)

This creates a visual grid where each sheet shows a row of 17 status indicators — the **Monday Report** that management uses weekly to track every active piece of millwork.

### 4. Query Family Built on qrySheetList

**Filtering variants** (`metadata/access/queries.csv:2277-2369`):

| Query | Purpose | Filter Logic |
|-------|---------|-------------|
| `qrySheetListActive` | Active sheets only | SheetCompleteDate IS NULL AND Delivered IS NULL AND Sheet <> "tbd" |
| `qrySheetListSortDeliveryTargetShopOnly` | Shop-stage sheets sorted by delivery | Stage=3 OR any shop process started while in earlier stage |
| `qrySheetListSortDeliveryTargetShopOnlyRevised` | Same but includes more early-start scenarios | Stage=3 OR VenStarted/SawStarted/CNCStarted/VenDone/MillDone/SawDone |
| `qrySheetListSortDept` | Sorted by current department | Groups sheets by CurDept, shows dept start date and duration |
| `qrySheetListDept` | Active sheets with department timing | Calculates DeptStart (target date minus duration) per current dept |
| `qrySheetListDeptUp` | Department workload with upcoming | Cross-joins with tblRptCriteria to show "In {process}" vs "Upcoming" |
| `qrySheetListEngWorkload` | Engineering phase only | CurDept < 7 AND project not completed |
| `qrySheetListEngWorkloadActive` | Active engineering work | Currently started in drafting, resubmit, corrections, or stockbill |
| `qrySheetListEngWorkloadReady` | Ready to start engineering | At department but work not yet begun |
| `qrySheetListEngWorkloadNotReady` | Blocked engineering | On hold or waiting for prerequisite |
| `qrySheetListProjDeets` | With project details | Adds location, company, contact, OT finish date |
| `qrySheetListSortProjDelivery` | PM delivery view | Sorted by PM initials, then project, then delivery target |

### 5. Physical Shop Layout

**Location**: `Tables/tblShopAreas.csv` (11 rows)

The physical shop at DWS consists of:

| ID | Area |
|----|------|
| 1 | Office - Main |
| 2 | Shop - Main |
| 3 | Shop - Ship Station |
| 4 | Veneer Shop |
| 11 | Outside - Front of Bldg. |
| 12 | Shop - Finish Dept. |
| 13 | Shop - Work Bench |
| 14 | Shop - Woodworking Area |
| 15 | Shop - Metal Working Area |
| 16 | Outside - Back Alley |
| 17 | Shop - Mill Department |

These map to the production stages: Veneer Shop (stage 8), Mill Department (stage 9), Work Bench (stage 13), Finish Dept (stage 14), Woodworking Area (saw/CNC).

### 6. Duration Templates

**Location**: `Tables/tblDefaulDurations.csv` via `~sq_flistShopDurations` query

`tblDefaulDurations` stores preset workflow templates with default day counts and skip flags for each of the 17 stages. When a new sheet is created, a template is applied to pre-populate all the `{X}Dur` and `{X}Skip` columns, establishing the expected timeline through the shop.

### 7. Field Scheduling Subsystem

**Location**: `metadata/access/queries.csv:1047-1070`

Separate from the shop pipeline, a parallel field scheduling system tracks installer assignments:

- `tblFieldDates` — calendar of field work days
- `tblFieldDateProject` — links dates to projects with position ordering
- `tblFieldDateEmp` — assigns employees to date+project with hours, department, OT flags
- `qryFieldActiveDate` — generates availability matrix (employees × dates between hire/termination)
- `qryFieldEmpScheduled` — deduplicates multi-project assignments per employee per day

### 8. Aggregate Process Groups

`tblDefaultSheetProc` defines rollup codes for reporting:

| MondayRptNo | SheetProcess | Aggregates |
|-------------|-------------|------------|
| 107 | Engineering | All CurDept where Stage < 3 |
| 109 | Drawing Phase | Stage=1 excluding Approval |
| 129 | Program/Stockbill | Stage = 2 |
| 130 | Shop | Stage = 3 |
| 150 | Machine | CurDept 9-12 (Mill/Saw/CNC/Other) |
| 151 | Bench/Assembly | CurDept 13-15 |
| 152 | Not Delivered | Delivered IS NULL |
| 153 | Install | Stage = 4 |
| 154 | Waiting for Delivery | Stage=4 AND Delivered IS NULL |

### 9. Billing Integration

**Location**: `Tables/tblProcCd.csv` (49 process codes)

Each production stage connects to billing rates via `tblDefaultSheetProc.SheetProcess` → `tblProcCd.Process`:

- Regular, OT, and DT billing rates per process
- Group categories: Eng, PM, Prod, Field, Other
- Example: Bench/Assembly bills at $102.67/hr regular, $119.17 OT, $135.67 DT

## What This Reveals About DWS Production Workflow

### The Business Process

1. **A project comes in** → Project manager creates sheets (shop drawings) for each piece of millwork needed
2. **Engineering phase** (Stage 1): Drafters create drawings, submit for approval, field-measure existing conditions, correct drawings, create material stockbills
3. **Programming phase** (Stage 2): CNC programmers create toolpaths, stock material is ordered/pulled, sheets are "released to shop" (`ToShopAct` date)
4. **Shop production** (Stage 3): Physical fabrication flows through specialized departments — veneer, milling, sawing, CNC machining, bench assembly, finishing, final assembly
5. **Delivery** (Stage 3→4): Completed items go to storage or directly to field
6. **Field installation** (Stage 4): Installers are scheduled per day per project with hour allocations
7. **Completion** (Stage 5): Sheet marked complete with `SheetCompleteDate` and `Delivered` date

### Key Design Patterns

- **Skip logic** allows any stage to be bypassed — not every piece needs veneer, CNC, or finishing
- **NoShop flag** identifies items that skip the entire shop (field-install-only, like hardware or purchased items)
- **Hold system** freezes a sheet at any department with status tracking
- **Ready vs Started** distinguishes "queued at a station" from "actively being worked on"
- **Parallel tracking** — the `ShopStarted` flag detects when shop work begins ahead of the normal sequence (e.g., veneer starting while still in engineering)
- **Duration tracking** stores both planned days per stage and actual completion dates, enabling schedule variance analysis

### The Monday Report

The entire system is built around a weekly management review called the **Monday Report** (hence `MondayRptNo`, `listSheetMondayRpt` form). Each Monday, management reviews a grid of every active sheet with 17 color-coded status indicators, sorted by delivery target. This is the operational heartbeat of the shop.

## Code References

- `metadata/access/queries.csv:2270-2276` — qrySheetList (master query, 68 fields)
- `metadata/access/queries.csv:2277-2280` — qrySheetListActive
- `metadata/access/queries.csv:2350-2353` — qrySheetListSortDeliveryTargetShopOnly
- `metadata/access/queries.csv:2297-2301` — qrySheetListDept (department timing)
- `metadata/access/queries.csv:2302-2334` — qrySheetListEngWorkload family (8 queries)
- `metadata/access/queries.csv:3842-3843` — ~sq_flistShopDurations
- `metadata/access/queries.csv:667-669` — qryCalMonday
- `metadata/access/queries.csv:1047-1070` — Field scheduling queries
- `metadata/sql_server/columns.csv:1085-1201` — tblSheetNEW (117 columns)
- `metadata/sql_server/columns.csv:381-394` — tblDefaultSheetProc (14 columns)
- `metadata/sql_server/columns.csv:347-380` — tblDefaulDurations (duration templates)
- `Tables/tblDefaultSheetProc.csv` — Process definitions (38 rows)
- `Tables/tblShopAreas.csv` — Physical shop areas (11 rows)
- `Tables/tblProcCd.csv` — Process codes with billing rates (49 rows)

## Open Questions

1. **How are sheets advanced between departments?** The `CurDept` field is updated, but is this manual (operator clicks "Done") or automated? The Access forms would reveal this.
2. **What does the Monday Report form actually look like?** The `listSheetMondayRpt` form is referenced extensively but we only see its data queries, not its layout.
3. **Are machine stages (Mill/Saw/CNC/Other) sequential or parallel?** The `MondayRptNo` numbering implies sequential, but the aggregate code `Machine` (MondayRptNo 150) with `[CurDept] between 9 and 12` suggests they may operate in parallel for a given sheet.
4. **What triggers the `Started` flags?** These appear to be manually set by operators when they begin work on a sheet at their station.
5. **How do duration templates work in practice?** Does the system auto-calculate target dates from durations, or are dates set manually?
