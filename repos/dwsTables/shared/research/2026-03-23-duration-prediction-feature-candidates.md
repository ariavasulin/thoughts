---
date: 2026-03-23T20:53:34Z
researcher: Aria Sulin
git_commit: f1d40a7
branch: main
repository: dwsTables
topic: "What additional features from existing CSV tables could predict duration overruns?"
tags: [research, duration-prediction, feature-engineering, modeling, eda]
status: complete
last_updated: 2026-03-23
last_updated_by: Aria Sulin
---

# Research: Duration Prediction Feature Candidates

**Date**: 2026-03-23T20:53:34Z
**Researcher**: Aria Sulin
**Git Commit**: f1d40a7
**Branch**: main
**Repository**: dwsTables

## Research Question

The current modeling_df (14,042 records) uses SheetID, ProjectID, SheetType, year, n_active_stages, planned_total, actual_total, variance_days, PM, Estimator. A log-linear regression on actual/planned ratio using SheetType + n_active_stages + PM achieved R²≈0. What additional features extractable from the existing 122 CSV tables could predict duration overruns?

## Summary

The current model uses only sheet-level metadata (type, stage count, PM). The database contains rich untapped signal across **financial**, **scope/complexity**, **client/history**, **geographic**, **temporal/backlog**, and **change order** dimensions. The strongest candidates are dollar-denominated features (project cost, per-sheet cost), scope complexity proxies (sheet hours, purchasing line items, process counts), and client/estimator history features — all joinable to sheets via ProjectID.

## Detailed Findings

### Current State (from `eda/duration_regression.json`)

- Training: 8,697 records (2010-2019), Test: 3,912 records (2020-2023)
- Best model: Global Median Ratio (multiply planned by 1.449)
- R² of regression model: 0.0005 — essentially zero
- The features tried (SheetType, n_active_stages, PM) have no linear relationship to log(actual/planned)
- Segment analysis shows SheetType="Sheet" has -30 day bias vs "SK" at -1.2 days — this is confounded with planned duration magnitude

### Table-by-Table Feature Inventory

#### 1. tblProject (1,933 rows) — Primary project-level features

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `TotProjCost` | money | Total project cost — primary dollar feature |
| `FixtureCost` | money | Fixture vs non-fixture split |
| `NonFixtureCost` | money | Fixture vs non-fixture split |
| `TMEstProjCost` | money | T&M estimated cost (alternative estimate) |
| `TotalHrs` | smallint | Estimated total project hours |
| `Estimator` | int FK | Who estimated the project |
| `Client` | int FK | Client company — enables repeat-client features |
| `Contractor` | int FK | GC relationship |
| `Architect` | int FK | Architect relationship |
| `Location` | int FK | Job site location → geography |
| `PM`, `PM2`, `PM3` | int FK | Up to 3 PMs per project |
| `Foreman` | int FK | Shop foreman assigned |
| `RetentionPct` | numeric | Retention percentage |
| `MATSPct` | numeric | MATS percentage |
| `EstProjectStart` | date | Estimated start date → seasonality |
| `EstToJobsite` | date | Estimated delivery to jobsite |
| `EstReadyForProd` | date | Estimated production-ready date |
| `EngEnd`, `ProdEnd` | date | Planned phase end dates |
| `MoveIn` | date | Move-in date |
| `Completed` | bit | Whether project completed |
| `BillRateType` | int | Billing rate type (T&M vs fixed?) |

**Join**: `tblSheetNEW.ProjectID → tblProject.ProjectID`

#### 2. tblSheetNEW (15,939 rows) — Rich per-sheet features already in source

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `SheetHrs` | numeric | Estimated hours for this sheet |
| `ShopHrs` | numeric | Estimated shop hours |
| `MatlTyp` | varchar | Material type (veneer, plam, etc.) |
| `Priority` | varchar | Sheet priority level |
| `HoldStatus` | varchar | Whether sheet was ever on hold |
| `SubmitType` | varchar | Submission type |
| `DraftDur1`–`AssemblyDur14` | numeric | 14 planned stage durations |
| `FieldDur`, `StorDur` | numeric | Field and storage durations |
| `DraftSkip1`–`AssemblySkip14` | bit | Which stages are skipped |
| `Drftr`, `Pgmr`, `Stkblr` | int FK | Assigned drafter, programmer, stockbiller |

**Key insight**: The 14 `*Dur` columns ARE the planned durations. The `*Skip` flags define the workflow path. A sheet that skips Veneer+Mill but has long Bench+Finish durations is a fundamentally different product than one that doesn't.

#### 3. tblBid (4,317 rows) — Pre-award estimate data

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `TotProjCost` | money | Bid-stage cost estimate |
| `Confidence` | decimal | Estimator's confidence score (0-1) |
| `TotalHrs` | smallint | Estimated hours at bid stage |
| `DateAdded` → `AwardDate` | dates | Bid-to-award cycle time |
| `Status`, `StatusReason` | varchar | Bid outcome |
| `WPScope` | money | Work package scope amount |
| `IncludeWProj` | bit | Include with project flag |

**Join**: `tblBid.JobNo → tblProject.ProjectID` (awarded bids)

#### 4. tblCO (16,158 rows) — Change Orders

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `COAmt` | money | Change order dollar amount |
| `CODays` | int | Schedule impact in days |
| `ScheduleChange` | varchar | Whether schedule changed |
| `COStatus` | varchar | Approved/Draft/Submitted/Executed |
| `COHold`, `COHoldDept` | varchar/int | Whether CO caused a hold |
| `ProjectID`, `SheetID` | int FK | Links to both project and sheet |

**Key insight**: COs link directly to SheetID. Count of COs per sheet, total CO dollars per project, and whether a CO caused a schedule change are high-signal features. The `CODays` column explicitly captures schedule impact.

#### 5. tblSheetProcess (68,790 rows) — Per-process assignments and estimates

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `Process` | varchar | Process name (Drafting, Programming, etc.) |
| `EstHrs` | numeric | Estimated hours for this process |
| `AvgHrsPerDay` | numeric | Expected throughput rate |
| `EstStart`, `ActStart` | date | Planned vs actual start |
| `EstComplete`, `ActComplete` | date | Planned vs actual completion |
| `AssignedTo` | int FK | Who is assigned |
| `SheetID` | int FK | Direct sheet link |

**Key insight**: ~68K rows for ~16K sheets = ~4.3 processes per sheet. Aggregations: total EstHrs across processes, number of distinct processes, max single-process EstHrs, ratio of EstHrs to AvgHrsPerDay. The gap between EstStart and ActStart for the FIRST process is an early warning signal.

#### 6. tblPurchasing (31,424 rows) — Material purchasing

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `Price` | money | Line item price |
| `Qty` | numeric | Quantity ordered |
| `Sheet` | int FK | Links to sheet |
| `ProjectID` | int FK | Links to project |
| `ReqDate`, `OrderDate`, `ETA`, `ReceiveDate` | dates | Procurement timeline |
| `PurchHold` | bit | Whether purchasing was held |
| `type1`, `type2`, `mfg1`, `mfg2` | varchar | Material/manufacturer types |
| `Vendor` | int FK | Vendor identity |

**Key insight**: 31K rows for 16K sheets = ~2 purchasing items per sheet. Count of purchasing items, total purchase dollar value, number of distinct vendors, and procurement lead time (OrderDate to ReceiveDate) are all computable.

#### 7. tblPO (14,687 rows) — Purchase Orders

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `ShipCost`, `OtherCost` | money | Shipping and other costs |
| `ShipDeadline` | date | Delivery deadline |
| `Project` | int FK | Links to project |
| `TaxPct` | numeric | Tax percentage |

#### 8. tblSheetHoldJournal (750 rows) — Hold Events

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `HoldStart`, `HoldEnd` | dates | Hold duration |
| `HoldDept` | int | Department causing hold |
| `HoldStatus` | varchar | Reason (Design, RFI) |
| `SheetID` | int FK | Direct sheet link |

**Key insight**: Holds directly extend duration. Count of holds per sheet and total hold days are strong features, though they may only be available mid-process (not at prediction time).

#### 9. tblRFI (6,089 rows) — Requests for Information

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `RFIDate`, `Replyby`, `ResolveDate` | dates | RFI lifecycle timing |
| `ProjectID`, `SheetID` | int FK | Links to project and sheet |
| `RFIHold` | bit | Whether RFI caused a hold |
| `Pri` | varchar | Priority level |

**Key insight**: RFI count per project/sheet indicates design ambiguity. Projects with many RFIs likely have scope that's harder to nail down, leading to overruns.

#### 10. tblFS (4,883 rows) — Finish Specifications

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `ProjectID` | int FK | Links to project |
| `FSQuantity` | int | Quantity of finish items |
| `Status` | varchar | Approval status |
| `FinishCode`, `Stain` | varchar | Finish type identifiers |
| `RequestDate`, `ApprovedDate` | dates | Approval cycle time |

**Key insight**: Number of distinct finish specs per project is a complexity proxy. Finish approval delays (RequestDate → ApprovedDate) signal client decision-making speed.

#### 11. tblCompany (2,071 rows) — Client/Company Data

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `CompanyType` | varchar | Company type classification |
| `CoSubtype` | varchar | Subtype |
| `City`, `State`, `Zip` | varchar | Geographic location |
| `Terms` | varchar | Payment terms |

**Join path**: `tblProject.Client → tblContact.CompanyID → tblCompany.CompanyID`

#### 12. tblLocation (696 rows) — Job Site Locations

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `LocationCity`, `LocationState`, `LocationZip` | varchar | Job site geography |

**Join**: `tblProject.Location → tblLocation.LocationID`

#### 13. tblTimesheet (288,977 rows) — Actual Labor Hours

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `RegHrs` | numeric | Regular hours worked |
| `Employee` | int FK | Worker identity |
| `ProcessCd` | varchar | Process code |
| `Project`, `Sheet` | int FK | Links to project and sheet |
| `WorkDate` | date | When work happened |

**Key insight**: This is the ACTUALS table. Total actual hours per sheet vs estimated hours is the core overrun metric. However, this is outcome data — useful for training but not available at prediction time unless lagged.

#### 14. tblTransmittal (8,702 rows) — Document Transmittals

Count of transmittals per project indicates communication overhead and coordination complexity.

#### 15. tblDefaulDurations — Workflow Templates

| WorkflowName | Example |
|---|---|
| "Pantry - Plam" | 8+5+2+2+2+1+0+0+3+3+1+6+0+1 = ~34 days |
| "Doors - Veneer" | 3+5+2+2+2+1+2+2+1+2+1+1+5+1 = ~30 days |

**Key insight**: The workflow template name (if recoverable from MatlTyp or SheetType) encodes the expected production path. Sheets that deviate from the template duration pattern are more likely to overrun.

#### 16. tblMilestone (238 rows) — Project Milestones

Milestone-based billing with `MatlPct`, `EngPct`, `ShopPct`, `FieldPct` breakdowns. Number of milestones per project indicates project phasing complexity.

#### 17. tblProjSubContract (2,201 rows) — Subcontractor Contracts

| Column | Type | Feature Potential |
|--------|------|-------------------|
| `EstimateAmt`, `RevisedEst`, `OrigEst` | money | Subcontract values |
| `MatlValue`, `LaborValue` | money | Material vs labor split |
| `Matlpct`, `ShopPct` | numeric | Phase percentages |

Number of subcontracts per project and total subcontract dollar value indicate external dependency complexity.

---

## Top 15 Candidate Features Ranked by Expected Predictive Value

### Tier 1: High Expected Signal (join directly, strong theoretical basis)

| Rank | Feature | Source Table(s) | Join Key | Rationale |
|------|---------|----------------|----------|-----------|
| **1** | **Project total cost (`TotProjCost`)** | tblProject | ProjectID | Dollar magnitude is the strongest proxy for scope. A $2M project behaves differently than a $50K project. Use log-transform. |
| **2** | **Per-sheet estimated hours (`SheetHrs` + `ShopHrs`)** | tblSheetNEW | Already in source | Direct measure of expected effort per sheet. Sheets with more hours have more opportunity for variance. |
| **3** | **Number of change orders per project** | tblCO → count by ProjectID | ProjectID | COs indicate scope instability. High CO count = higher overrun probability. ~16K COs across ~1.9K projects = ~8.4 per project. |
| **4** | **Total CO dollar amount per project** | tblCO → sum(COAmt) by ProjectID | ProjectID | Dollar magnitude of scope changes. |
| **5** | **Material type (`MatlTyp`)** | tblSheetNEW | Already in source | Veneer vs plam vs solid wood have fundamentally different production paths and risk profiles. |
| **6** | **Total purchasing line items per sheet** | tblPurchasing → count by Sheet | Sheet FK | Number of materials to procure = supply chain complexity. More items = more chances for delay. |

### Tier 2: Medium-High Expected Signal (aggregation required)

| Rank | Feature | Source Table(s) | Join Key | Rationale |
|------|---------|----------------|----------|-----------|
| **7** | **Sheets per project (project scope)** | tblSheetNEW → count by ProjectID | ProjectID | More sheets = larger project = more coordination overhead. A project with 50 sheets has different dynamics than one with 3. |
| **8** | **Stage skip pattern (binary vector)** | tblSheetNEW Skip columns | Already in source | Which of 14 stages are skipped defines the workflow. Encode as count of active stages or as categorical workflow type. |
| **9** | **Total process EstHrs per sheet** | tblSheetProcess → sum(EstHrs) by SheetID | SheetID | More granular than SheetHrs — captures planned effort across all assigned processes. |
| **10** | **Client repeat-project count** | tblProject → count by Client | Client FK | Repeat clients have established expectations and workflows. First-time clients introduce uncertainty. |
| **11** | **Estimator historical overrun rate** | tblProject + modeling_df → avg(actual/planned) by Estimator | Estimator FK | Some estimators systematically under- or over-estimate. Use lagged historical average (only past projects). |

### Tier 3: Medium Expected Signal (more complex extraction)

| Rank | Feature | Source Table(s) | Join Key | Rationale |
|------|---------|----------------|----------|-----------|
| **12** | **Seasonality (month/quarter of SheetStart)** | tblSheetNEW.SheetStart | Already in source | Shop throughput varies seasonally (holiday slowdowns, summer staffing). |
| **13** | **Shop backlog at sheet start** | tblSheetNEW → count of active sheets at SheetStart date | Cross-join on dates | When the shop is overloaded, everything takes longer. Count sheets where SheetStart ≤ date ≤ ActualComplete. |
| **14** | **RFI count per project** | tblRFI → count by ProjectID | ProjectID | Design ambiguity proxy. More RFIs = less clear scope = more rework. ~6K RFIs across projects. |
| **15** | **Bid confidence score** | tblBid.Confidence | tblBid.JobNo → ProjectID | Estimator's self-assessed confidence. Low confidence = higher risk of overrun. Only available for awarded bids. |

### Honorable Mentions (worth exploring)

| Feature | Source | Rationale |
|---------|--------|-----------|
| Purchasing lead time (OrderDate→ReceiveDate) | tblPurchasing | Long lead times for materials can cascade into production delays |
| Number of distinct vendors per project | tblPurchasing → count distinct Vendor | Supply chain fragmentation |
| Finish spec count per project | tblFS → count by ProjectID | Finish complexity |
| Subcontract count & dollar total | tblProjSubContract | External dependency risk |
| Hold count and total hold days per sheet | tblSheetHoldJournal | Direct measure of disruption (but may not be available at prediction time) |
| Location (city/state) | tblProject → tblLocation | Geographic effects (travel time, permitting, inspection requirements) |
| PM × Client interaction | tblProject | Some PM-client combinations may systematically over/underrun |
| Transmittal count per project | tblTransmittal | Communication overhead proxy |
| Bid-to-award cycle time | tblBid (AwardDate - DateAdded) | Long sales cycles may indicate complex/uncertain scope |
| Planned duration deviation from template | tblSheetNEW durations vs tblDefaulDurations | Sheets whose planned durations deviate from the standard template for their MatlTyp may be unusual/risky |

## Feature Engineering Notes

### Join Paths
```
tblSheetNEW.SheetID ← primary key (your modeling unit)
tblSheetNEW.ProjectID → tblProject.ProjectID → tblBid.JobNo
                                              → tblCompany (via Client FK)
                                              → tblLocation (via Location FK)
tblSheetNEW.SheetID → tblCO.SheetID (change orders per sheet)
tblSheetNEW.SheetID → tblSheetProcess.SheetID (process details)
tblSheetNEW.SheetID → tblPurchasing.Sheet (purchasing items)
tblSheetNEW.SheetID → tblSheetHoldJournal.SheetID (holds)
tblSheetNEW.ProjectID → tblRFI.ProjectID (RFIs per project)
tblSheetNEW.ProjectID → tblTransmittal.ProjectID
tblSheetNEW.ProjectID → tblProjSubContract (via tblProjSub.ProjID)
```

### Temporal Leakage Warning
Features must be computed using ONLY information available at prediction time (typically when the sheet is created / durations are planned). Features derived from:
- `tblCO` — COs may arrive after sheet start. Use only COs with `CODate < SheetStart` or aggregate at project level from prior phases.
- `tblSheetHoldJournal` — Holds happen during production. Only useful if predicting mid-stream.
- `tblTimesheet` — Actuals. Never available at prediction time.
- `tblPurchasing.ReceiveDate` — Outcome data. Use `OrderDate` and `ETA` instead.
- Client/estimator history features must use ONLY projects completed BEFORE the current sheet's start date.

### Recommended First Pass
Start with features 1-6 (Tier 1) since they require minimal aggregation and have the strongest theoretical basis. Project cost alone may explain more variance than all current features combined, given the enormous range of project sizes in a commercial millwork shop.

## Code References
- `Tables/tblProject.csv` — 1,933 projects with cost and date fields
- `Tables/tblSheetNEW.csv` — 15,939 sheets with 109 columns including all stage durations
- `Tables/tblCO.csv` — 16,158 change orders with dollar amounts and schedule impact
- `Tables/tblSheetProcess.csv` — 68,790 process assignments with estimated hours
- `Tables/tblPurchasing.csv` — 31,424 purchasing line items with prices and dates
- `Tables/tblBid.csv` — 4,317 bids with confidence scores and cost estimates
- `Tables/tblRFI.csv` — 6,089 RFIs linked to projects and sheets
- `Tables/tblSheetHoldJournal.csv` — 750 hold events with start/end dates
- `Tables/tblFS.csv` — 4,883 finish specifications
- `Tables/tblDefaulDurations.csv` — Workflow templates with standard stage durations
- `eda/duration_regression.json` — Current model results (R²=0.0005)

## Open Questions

1. **What is `MatlTyp` cardinality?** — Need to check distinct values in tblSheetNEW.MatlTyp to understand how many material categories exist and whether they map to tblDefaulDurations.WorkflowName.
2. **Can we recover which workflow template was used per sheet?** — If MatlTyp maps to WorkflowName, we can compute planned vs template deviation.
3. **How many sheets have non-null SheetHrs?** — This column may be sparsely populated.
4. **What's the Confidence column fill rate in tblBid?** — May be too sparse to use.
5. **Are COs systematically pre- or post-sheet-start?** — Determines whether CO count is a valid prediction-time feature or only mid-stream.

---
*Generated during codebase research session, 2026-03-23*
