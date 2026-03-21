---
date: 2026-03-21T19:00:00-07:00
researcher: Claude
git_commit: no-commits (uncommitted repo)
branch: main
repository: dwsTables
topic: "Estimation Variance & Calibration: EDA Feasibility for DWS Production Data"
tags: [research, codebase, estimation, variance, calibration, tblSheetNEW, tblDefaulDurations, tblBid, duration, scheduling, EDA]
status: complete
last_updated: 2026-03-21
last_updated_by: Claude
---

# Research: Estimation Variance & Calibration — EDA Feasibility

**Date**: 2026-03-21T19:00:00-07:00
**Researcher**: Claude
**Prerequisite**: [Shop Schedule Production Workflow](2026-03-21-shop-schedule-production-workflow.md)

## Research Question

Can the DWS database support analysis of estimation variance and calibration across the 17-stage production pipeline? What data exists for comparing planned durations vs actual elapsed time, and what segmentation dimensions enable bias analysis?

## Executive Summary

**Yes — the data strongly supports estimation variance analysis**, but with important caveats about what "estimate" and "actual" mean in this system. The key findings:

1. **Per-stage duration estimates exist** for all 17 stages on every sheet (15,932 sheets × 17 `{Name}Dur{N}` columns). These are NOT raw template values — they're manually adjusted per-sheet estimates that represent management's best guess at how long each stage will take.

2. **Actual completion timestamps are partially available.** The system records target dates (`{Name}Dt{N}`) and completion flags (`{Name}Done{N}`), but **not actual completion dates per stage**. Actual elapsed time must be inferred from date differences between consecutive stages or from the overall `SheetCompleteDate` minus `SheetStart`.

3. **15,783 completed sheets** (99.1%) with completion dates spanning 2010–2023 provide substantial historical depth for modeling.

4. **Rich segmentation dimensions**: PM (26), Estimator (23), Client (353), SheetType (3), MatlTyp (2,687), Drafter (37), Programmer (21), Foreman (15), plus project-level attributes.

5. **A critical measurement gap**: the system tracks *when stages should finish* (target dates) and *whether they're done* (boolean flags), but not *when they actually finished*. This means per-stage variance analysis requires careful proxy construction.

## 1. Duration Estimates vs Actuals in tblSheetNEW

### What Exists: Per-Stage Duration Fields

Each of the 17 stages has a **planned duration** column in `tblSheetNEW`:

| Stage | Duration Column | Null Rate | Min | Max | Mean | Median |
|-------|----------------|-----------|-----|-----|------|--------|
| 1. Draft | `DraftDur1` | ~1% | -104 | 573 | 7.5 | 5.0 |
| 2. Submit | (no explicit dur) | — | — | — | — | — |
| 3. Approve | `ApprDur2` | ~1% | -60 | 214 | 4.7 | 5.0 |
| 4. Measure | `MeasureDur3` | ~1% | -20 | 262 | 2.3 | 2.0 |
| 5. Correct | `CorrectDur4` | ~1% | -10 | 684 | 2.6 | 2.0 |
| 6. Stock | `StockDur5` | ~1% | -25 | 224 | 2.2 | 2.0 |
| 7. Program | `PgmDur6` | ~1% | -10 | 168 | 1.2 | 1.0 |
| 8. Veneer | `VenDur7` | ~1% | -20 | 112 | 1.5 | 0.0 |
| 9. Mill | `MillDur8` | ~1% | -3 | 112 | 0.7 | 0.0 |
| 10. Saw | `SawDur9` | ~1% | -10 | 112 | 2.4 | 2.0 |
| 11. CNC | `CNCDur10` | ~1% | -10 | 84 | 1.6 | 1.0 |
| 12. Other | `OtherDur11` | ~1% | -10 | 112 | 1.2 | 1.0 |
| 13. Bench | `BenchDur12` | ~1% | -27 | 168 | 3.6 | 2.0 |
| 14. Finish | `FinishDur13` | ~1% | -10 | 112 | 2.3 | 0.0 |
| 15. Assembly | `AssemblyDur14` | ~1% | -10 | 56 | 1.0 | 1.0 |
| 16. Storage | `StorDur` | ~1% | 0 | 84 | 0.6 | 0.0 |
| 17. Field | `FieldDur` | ~1% | -10 | 224 | 6.7 | 5.0 |

**Key observations:**
- **Negative durations exist** (e.g., DraftDur1 = -104). These appear to be manual corrections when schedules are pulled backward — the system allows negative values as schedule adjustments.
- **Zero-dominated distributions** for stages that are frequently skipped (VenDur7 median=0, MillDur8 median=0, FinishDur13 median=0). These correspond to high skip rates.
- **Very low null rates** (~1%) — nearly every sheet has duration values for every stage, even skipped ones (where dur=0 or blank).
- **Heavy right skew** — means are 1.5–3× medians, indicating some sheets have very long allocated durations (likely complex or delayed items).

### What Exists: Target Date Fields

Each stage also has a **target completion date**:

| Stage | Date Column | Population Rate |
|-------|------------|----------------|
| 1. Draft | `DraftDt1` | 49.5% |
| 3. Approve | `ApprDt2` | 48.0% |
| 4. Measure | `MeasureDt3` | 47.7% |
| 5. Correct | `CorrectDt4` | 47.5% |
| 6. Stock | `StockDt5` | 47.4% |
| 7. Program | (via `ToShop`) | 25.3% (planned) / 94.9% (actual `ToShopAct`) |
| 8. Veneer | `VenDt7` | 47.0% |
| 9. Mill | `MillDt8` | 46.9% |
| 10. Saw | `SawDt9` | 46.9% |
| 11. CNC | `CNCDt10` | 46.9% |
| 12. Other | `OthDt11` | 46.8% |
| 13. Bench | `BenchDt12` | 46.8% |
| 14. Finish | `FinishDt13` | 46.7% |
| 15. Assembly | `AssemblyDt` | 46.2% |
| 16. Storage | `StorDt` | 46.2% |
| 17. Field | `FieldDt` | 46.1% |

**Critical finding**: Target dates are only ~47–50% populated. This suggests the scheduling system was adopted **partway through the 15-year data history** — earlier sheets have durations but no target dates.

### What's Missing: Actual Completion Dates Per Stage

The system records:
- `{Name}Done{N}` — boolean flag (-1 = done, 0 = pending)
- `{Name}Started` — boolean flag (-1 = started, 0 = not)

But it does **NOT** record the actual date each stage was completed. The only actual timestamps are:
- `SheetStart` — when the sheet entered the system
- `ToShopAct` — actual release-to-shop date (94.9% populated)
- `SheetCompleteDate` — overall completion date (99.1% populated)
- `Delivered` — delivery date (94.3% populated)

### Constructing Actual Durations: Proxy Approaches

Given the data structure, actual per-stage durations can be approximated via:

**Approach A: Consecutive target date differences (weak proxy)**
```
ActualDraft ≈ ApprDt2 - DraftDt1
ActualAppr  ≈ MeasureDt3 - ApprDt2
...
```
Problem: These are *target* dates, not actual completion dates. They reflect the schedule as set at planning time, not reality.

**Approach B: Aggregate phase durations (stronger)**
The system already computes:
- `EngDur` = total engineering days (stages 1–6 actual, computed in `qrySheetList`)
- `ShopDur` = total shop days (stages 8–14 actual)
- `TotDur` = grand total days

These appear to be computed from actual dates:
```sql
EngDur: IIf([ToShopAct] Is Null, Null, DateDiff("d", [SheetStart], [ToShopAct]))
ShopDur: (computed from ToShopAct to completion or delivery)
TotDur: DateDiff("d", [SheetStart], [SheetCompleteDate])
```

**Approach C: Overall variance (strongest)**
```
PlannedTotal = Sum of all {Name}Dur{N} for non-skipped stages
ActualTotal  = SheetCompleteDate - SheetStart (or TotDur)
Variance     = ActualTotal - PlannedTotal
```

This is the most reliable measure and is available for ~99% of sheets.

**Approach D: Phase-level variance (good)**
```
PlannedEng   = DraftDur1 + ApprDur2 + MeasureDur3 + CorrectDur4 + StockDur5 + PgmDur6
ActualEng    = ToShopAct - SheetStart
EngVariance  = ActualEng - PlannedEng
```

Similarly for shop phase: `ActualShop = SheetCompleteDate - ToShopAct` (approximately).

## 2. Delivery & Completion Estimates vs Actuals

### Available Comparison Pairs

| Planned | Actual | Coverage | Notes |
|---------|--------|----------|-------|
| `TargetComplete` | `SheetCompleteDate` | 28.4% vs 99.1% | Only ~4,500 sheets have both |
| `Delivery` | `Delivered` | 92.1% vs 94.3% | Strong coverage — best pair for delivery variance |
| `ToShop` | `ToShopAct` | 25.3% vs 94.9% | Low planned coverage limits utility |
| `ShtFinish` | `SheetCompleteDate` | (check) | May duplicate TargetComplete |

### Existing Access Queries for Variance

The `qrySheetList` master query already computes delivery variance metrics:

```sql
DeliveryStatusRed:  IIf([Delivery] Is Null, Null, IIf([Delivered] Is Null,
                    IIf(Date()-[Delivery]>0, Date()-[Delivery], Null),
                    IIf([Delivered]-[Delivery]>0, [Delivered]-[Delivery], Null)))

DeliveryStatusBlack: IIf([Delivery] Is Null, Null, IIf([Delivered] Is Null,
                     IIf(Date()-[Delivery]<=0, [Delivery]-Date(), Null),
                     IIf([Delivered]-[Delivery]<=0, [Delivery]-[Delivered], Null)))
```

- **Red** = days LATE (positive = overdue)
- **Black** = days EARLY (positive = ahead of schedule)

Similarly:
- `FinalStatusRed/Black` — variance against `TargetComplete` vs `SheetCompleteDate`
- `OTStatusRed/Black` — variance against OT finish projection (from `qrySheetCompleteDtOT`)

These are already modeling estimation error — the Access system was built with variance tracking in mind.

### Project-Level Estimation (tblBid)

`tblBid` (4,243 rows) contains project-level milestone estimates:

| Field | Description |
|-------|-------------|
| `EstProjectStart` | Estimated project start date |
| `EstReadyForProd` | Estimated ready-for-production date |
| `EstToJobsite` | Estimated delivery to jobsite |
| `MoveIn` | Client move-in date (hard deadline) |

Access queries already compute phase durations from these:
```
tmpEngDays  = EstReadyForProd - EstProjectStart
tmpProdDays = EstToJobsite - EstReadyForProd
tmpFieldDays = MoveIn - EstToJobsite
```

This enables **project-level** estimation variance analysis alongside the sheet-level analysis.

## 3. Granularity of Estimation Data

### Unit of Measurement
- **Duration values are in calendar days** (integer type, stored as whole numbers)
- Target dates include time component but are always `12:00:00 AM` — effectively date-only
- `DateDiff("d", ...)` in Access queries confirms calendar-day arithmetic

### Calendar vs Working Days
- The system uses **calendar days**, not working days. No business-day calendar table was found.
- However, `tblCalendar` (1,643 rows, 2010–2023) exists with a `WorkDay` flag — this could enable calendar-to-working-day conversion if needed.
- `qryCalMonday` generates Monday dates for the Monday Report cycle.

### Historical Depth
- **15,932 sheets** spanning **2010–2023** (13+ years)
- Year distribution of completed sheets:

| Year | Count | Year | Count |
|------|-------|------|-------|
| 2010 | ~200 | 2017 | ~1,350 |
| 2011 | ~750 | 2018 | ~1,200 |
| 2012 | ~900 | 2019 | ~1,250 |
| 2013 | ~850 | 2020 | ~950 |
| 2014 | ~1,000 | 2021 | ~1,250 |
| 2015 | ~1,100 | 2022 | ~1,300 |
| 2016 | ~1,250 | 2023 | ~850 |

- Growth trend from 2010–2017, slight decline 2020 (COVID), recovery 2021–2022.
- 2023 lower count likely represents partial year in the data extract.

### Sample Size Assessment
- **n=15,932** total sheets is excellent for regression modeling
- **n≈7,500** sheets with target dates (post-scheduling-system adoption) for date-based variance
- **n≈14,500** sheets with delivery dates for delivery variance analysis
- Sub-group sizes for segmentation: PM groups average ~600 sheets each, SheetType groups ~7,000+, individual clients range from 1 to hundreds

## 4. Systematic Bias Patterns — Segmentation Dimensions

### Sheet-Level Dimensions (tblSheetNEW)

| Dimension | Column | Distinct Values | Population Rate | Notes |
|-----------|--------|----------------|-----------------|-------|
| **Sheet Type** | `SheetType` | 3 (SK, Sheet, Split) | ~100% | SK=53%, Sheet=46%, Split=0.7% |
| **Material Type** | `MatlTyp` | 2,687 | High | Very high cardinality — needs grouping |
| **Drafter** | `Drftr` | 37 | ~87% | Who drew it |
| **Programmer** | `Pgmr` | 21 | ~68% | Who programmed CNC |
| **Stock Biller** | `Stkblr` | 59 | ~81% | Who did material takeoff |
| **Current Dept** | `CurDept` | 19 | ~100% | Current pipeline position |
| **Hold Status** | `HoldStatus` | 2 | ~100% | Whether sheet is on hold |
| **Building/Floor/Room** | `BldgNo`, `FloorNo`, `RoomNo` | Varies | Varies | Physical location within project |
| **Monday Report No** | `MondayRptNo` | ~18 | ~100% | Pipeline position at last report |

### Skip Pattern as Implicit Complexity Proxy

Skip rates reveal the production profile of each sheet:

| Stage | Skip Rate | Interpretation |
|-------|-----------|---------------|
| Veneer (VenSkip7) | 72.1% | Most items are NOT veneer |
| Mill (MillSkip8) | 60.6% | Many items skip milling |
| Approve (ApprSkip2) | 50.6% | Half skip formal approval |
| Finish (FinishSkip13) | 49.2% | Half skip finishing |
| Other (OthSkip11) | 39.7% | Specialty processes |
| CNC (CNCSkip10) | 27.1% | Most items need CNC |
| Bench (BenchSkip12) | 14.9% | Nearly all go through bench |
| Field (FieldSkip) | 7.4% | Nearly all get installed |
| Storage (StorSkip) | 0.0% | Never skipped (always passes through) |

**Idea**: The number of non-skipped stages per sheet is a natural **complexity proxy**. A sheet with 15 active stages vs 8 has a fundamentally different estimation challenge.

### Project-Level Dimensions (tblProject, joined via ProjectID)

| Dimension | Column | Distinct Values | Notes |
|-----------|--------|----------------|-------|
| **Project Manager** | `PM` | 26 | Links to tblContact |
| **Estimator** | `Estimator` | 23 | Who priced the job |
| **Client** | `Client` | 353 | Repeat clients may show learning effects |
| **Contractor** | `Contractor` | 72 | GC managing the project |
| **Project Type** | (implicit) | — | Inferrable from sheet types within project |
| **Location** | `Location` | 380 | Geographic variation |
| **Foreman** | `Foreman` | 15 | Only 7.5% populated — limited utility |
| **Project Cost** | `TotProjCost` | Continuous | $0–$6.59M, median $4,182, mean $111K |
| **Project Year** | `ProjYear` | ~14 | From start dates |

### Labor Dimensions (tblTimesheet — 289,238 rows)

- 276 employees, 27 process codes, date range 2010–2023
- Could join actual labor hours per stage per sheet for variance analysis
- `tblTimesheet.Process` maps to production stages

### Candidate Bias Hypotheses

1. **PM bias**: Do certain PMs consistently under/overestimate? (n≈600 per PM)
2. **Estimator bias**: Do estimators systematically set optimistic durations? (n≈700 per estimator)
3. **Complexity bias**: Do high-stage-count sheets have proportionally larger variance?
4. **Template bias**: Are certain workflow templates (e.g., "Paneling - Veneer") consistently miscalibrated?
5. **Temporal drift**: Has estimation accuracy improved over 2010–2023?
6. **Client learning**: Does estimation improve for repeat clients?
7. **Stage-specific bias**: Are shop stages (bench, finish) harder to estimate than engineering stages?
8. **Size bias**: Do larger projects ($100K+) have different variance patterns than small ones?
9. **Season/COVID effects**: Did estimation patterns change during 2020?

## 5. Existing Access Queries Analyzing Estimation Accuracy

### Already Computed in qrySheetList

| Field | Formula | What It Measures |
|-------|---------|-----------------|
| `DeliveryStatusRed` | `Delivered - Delivery` (if late) | Days late on delivery |
| `DeliveryStatusBlack` | `Delivery - Delivered` (if early) | Days early on delivery |
| `FinalStatusRed` | `SheetCompleteDate - TargetComplete` (if late) | Days late on completion |
| `FinalStatusBlack` | `TargetComplete - SheetCompleteDate` (if early) | Days early on completion |
| `EngDur` | `DateDiff("d", SheetStart, ToShopAct)` | Actual engineering phase days |
| `ShopDur` | (ToShopAct to completion) | Actual shop phase days |
| `TotDur` | `DateDiff("d", SheetStart, SheetCompleteDate)` | Actual total days |
| `DeptDate` | Target date for current department | Per-stage scheduling |
| `DeptStart` | `DeptDate - duration` | When current dept work should begin |

### Overtime Projection Queries

- `qrySheetCompleteDtOT`: Computes `OTFinishDate` — projected completion if overtime is applied
- `OTStatusRed/OTStatusBlack`: Variance against OT-adjusted schedule
- This suggests management actively models "what if we add OT?" scenarios

### Scheduling Update Query

- `qrySheetschedUpdate`: Complex date-chaining query that recalculates all target dates based on per-sheet durations:
  ```
  DraftDt1 = [SheetStart] + [DraftDur1]
  SubmitDt1 = [DraftDt1] + 0   (submit is same day)
  ApprDt2 = [SubmitDt1] + [ApprDur2]
  MeasureDt3 = [ApprDt2] + [MeasureDur3]
  ...
  ```
  This confirms: **target dates are computed FROM durations**, not independently set. Duration is the primary estimate; dates are derived.

### Field Hour Projection

- `qryFieldProjectionActualHrs`: Buckets actual field hours by week
- Enables comparison of planned field duration vs actual labor hours consumed

## 6. The tblDefaulDurations Template System

### How Templates Work

1. **18 workflow templates** are stored in `tblDefaulDurations`, each named for a product type (e.g., "Pantry - Plam", "Doors - Veneer", "Reception Desk")
2. When a new sheet is created, the user selects a template from a dropdown (`qrySelectDuration`)
3. The template's duration and skip values are **copied** into the sheet's own columns
4. The user (typically PM or scheduler) then **adjusts** the per-sheet values as needed
5. **No foreign key** links the sheet back to its source template — the template origin is lost

### Template Duration Profiles

| Template | Total Days | Active Stages | Heaviest Stage |
|----------|-----------|---------------|----------------|
| Paneling - Veneer | 64 | 17/17 | Drafting (8d), Bench (8d), Finish (8d), Field (10d) |
| Servery Counters PL/SS | 61 | 14/17 | Drafting (10d), Bench (10d), Field (10d) |
| Conf Credenza - Veneer | 56 | 17/17 | Drafting (7d), Bench (7d), Field (8d) |
| Pantry - Veneer | 55 | 17/17 | Drafting (8d), Finish (7d), Field (10d) |
| Reception Desk | 52 | 17/17 | Bench (8d), Finish (5d), Veneer (5d) |
| Banquette | 39 | 17/17 | Drafting (5d), Veneer (5d), Field (5d) |
| Doors - Veneer | 33 | 16/17 | Finish (5d), Approve (5d) |
| Pantry - Plam | 42 | 14/17 | Drafting (8d), Field (8d), Bench (6d) |
| Copy Rooms | 38 | 14/17 | Drafting (5d), Bench (6d), Field (5d) |
| Storage/Coat Closet | 21 | 13/17 | Drafting (3d), Approve (5d), Field (3d) |
| Restroom Vanities | 24 | 14/17 | Approve (5d), Drafting (4d), Field (3d) |
| Primed Base | 21 | 12/17 | Saw (3d), Finish (3d), Field (5d) |
| Doors - Blind PG | 37 | 15/17 | Drafting (5d), Approve (5d), Finish (4d) |
| Elevator Surrounds | 33 | 14/17 | Field (10d), Drafting (5d) |
| Sample - Veneer | 5 | 3/17 | Finish (3d), Veneer (2d) |
| Sample - Paint | 6 | 3/17 | Finish (5d), Saw (1d) |
| Sample Request | 5 | 3/17 | Finish (3d), Veneer (2d) |
| Blocking material | 2 | 3/17 | Stock (1d), Saw (1d) |

### Template vs Actual: Analysis Opportunity

Since templates are copied-then-adjusted, the per-sheet durations reflect a **two-step estimation process**:
1. **Template selection** (initial estimate based on product type)
2. **Manual adjustment** (refinement based on specific project/sheet characteristics)

**Modeling opportunity**: Compare actual durations against (a) the template default and (b) the adjusted per-sheet estimate. This decomposes estimation error into:
- **Template calibration error**: Is the template systematically wrong for this product type?
- **Adjustment error**: Does the manual adjustment improve or worsen the estimate?

Evidence of adjustment: actual `DraftDur1` values range from -104 to 573, while template values only range from 2 to 10. The adjustment step introduces enormous variance.

## 7. Data Quality Assessment

### Date Formats and Types
- All date columns: `M/D/YYYY 12:00:00 AM` format (Access datetime with midnight time)
- Duration columns: integer type, stored as whole numbers in calendar days
- Boolean flags (Done, Skip, Started): string "-1" (true) or "0" (false)

### Null Rates for Key Fields

**High coverage (>90%):**
- `SheetCompleteDate`: 99.1% (15,783 of 15,932)
- `ToShopAct`: 94.9%
- `Delivered`: 94.3%
- `Delivery`: 92.1%
- All `{Name}Dur{N}` fields: ~99%
- All `{Name}Done{N}` fields: ~99%
- All `{Name}Skip{N}` fields: ~99%

**Moderate coverage (25–90%):**
- `DraftDt1` through `FieldDt`: ~47–50% (scheduling system adopted mid-history)
- `Drftr` (drafter): ~87%
- `Stkblr` (stock biller): ~81%
- `Pgmr` (programmer): ~68%

**Low coverage (<25%):**
- `TargetComplete`: 28.4%
- `ToShop` (planned release): 25.3%
- `Foreman`: 7.5% (project-level field)

### Row Counts

| Table | Rows | Relevance |
|-------|------|-----------|
| `tblSheetNEW` | 15,932 | Primary analysis table |
| `tblProject` | ~5,000 | Project-level attributes |
| `tblBid` | 4,243 | Project-level estimates |
| `tblTimesheet` | 289,238 | Actual labor hours |
| `tblDefaulDurations` | 18 | Duration templates |
| `tblDefaultSheetProc` | 38 | Stage definitions |
| `tblCalendar` | 1,643 | Business day calendar |

### Data Issues to Watch

1. **Survivorship bias**: 99.1% of sheets are completed — the incomplete 149 may represent active work or abandoned items. Filter carefully.
2. **Left-censoring**: Pre-2012 data may lack target dates (scheduling system adopted later).
3. **Negative durations**: Appear in all stage duration columns. These are legitimate schedule adjustments, not errors, but need handling in models.
4. **Zero-inflation**: Many stages have duration=0 because they're skipped. Models should condition on `Skip=0` (non-skipped stages only).
5. **Template identity loss**: No FK to source template means template-vs-actual analysis requires heuristic matching (e.g., find sheets where all durations exactly match a template).

## Modeling Recommendations

### Tier 1: High Feasibility, Immediate

1. **Overall duration variance**: `TotDur - PlannedTotal` for all completed sheets. Segment by PM, estimator, year, project size, sheet type. n≈15,000.
2. **Delivery variance**: `Delivered - Delivery` (already computed as DeliveryStatusRed/Black). n≈14,000.
3. **Engineering phase variance**: `(ToShopAct - SheetStart) - PlannedEngDur`. n≈14,000.

### Tier 2: Moderate Feasibility, Requires Construction

4. **Shop phase variance**: `(SheetCompleteDate - ToShopAct) - PlannedShopDur`. Requires careful handling of field and storage phases.
5. **Template calibration analysis**: Match sheets to templates by duration pattern, then compare template defaults vs actuals. Requires fuzzy matching.
6. **Temporal calibration drift**: Has estimation accuracy improved over 2010–2023? Simple regression of |variance| on year.

### Tier 3: Ambitious, Requires Joining Multiple Tables

7. **Labor-adjusted variance**: Join tblTimesheet to get actual hours per stage per sheet. Compare planned duration × expected hours/day vs actual hours.
8. **Hierarchical Bayesian model**: PM-level and estimator-level random effects on estimation bias, with sheet-level covariates (type, complexity, template).
9. **Learning curve analysis**: Does estimation improve for repeat client/product-type combinations?

### Suggested First EDA Notebook

```python
# Notebook outline: estimation_variance_eda.ipynb

# 1. Load tblSheetNEW, compute PlannedTotal = sum of non-skipped Dur columns
# 2. Compute ActualTotal = SheetCompleteDate - SheetStart (calendar days)
# 3. Variance = ActualTotal - PlannedTotal
# 4. Distribution of variance (histogram, heavy tails expected)
# 5. Variance by year (temporal trend)
# 6. Variance by PM (boxplots, 26 PMs)
# 7. Variance by SheetType (SK vs Sheet vs Split)
# 8. Variance by complexity (number of non-skipped stages)
# 9. Delivery variance: Delivered - Delivery
# 10. Scatter: PlannedTotal vs ActualTotal with 45-degree line
# 11. Calibration plot: binned predicted vs actual (are estimates calibrated?)
```

## Code References

- `Tables/tblSheetNEW.csv` — 15,932 sheets with all duration/date/done/skip fields
- `Tables/tblDefaulDurations.csv` — 18 workflow templates
- `Tables/tblDefaultSheetProc.csv` — 38 process/stage definitions
- `metadata/access/queries.csv:2270-2276` — qrySheetList (master scheduling query)
- `metadata/access/queries.csv:2483` — qrySheetschedUpdate (date chaining logic)
- `metadata/access/queries.csv:1900` — qrySelectDuration (template dropdown)
- `metadata/access/queries.csv:884` — qryDupeSheet (copies duration/skip values)
- `catalog/profiles/tblProject.json` — project-level dimension cardinalities
- `catalog/profiles/tblTimesheet.json` — labor data for hours-based analysis
- `catalog/profiles/tblBid.json` — project-level estimates

## Open Questions

1. **When was the scheduling system adopted?** The ~47% target date population rate suggests a cutoff year. Identifying this would define the "clean" analysis window.
2. **Are durations in working days or calendar days?** The queries use `DateDiff("d",...)` (calendar days), and `tblCalendar` exists with `WorkDay` flags, but it's unclear if the *estimates* account for weekends.
3. **Can template origin be recovered?** Without a FK, heuristic matching (exact duration pattern match) might identify which template was used for each sheet.
4. **What fraction of sheets have manually adjusted durations?** If most sheets keep template defaults unchanged, the template calibration analysis becomes primary.
5. **Are there any sheets with actual per-stage completion timestamps?** The `ModDate` field tracks last modification — could this proxy for stage transitions if operators update fields at completion?
