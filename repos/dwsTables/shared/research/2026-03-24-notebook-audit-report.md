---
date: 2026-03-24T10:00:00-07:00
researcher: Claude (Opus 4.6)
git_commit: f1d40a72137b09b99a231e8846700a79c4e280fe
branch: main
repository: dwsTables
topic: "Comprehensive audit of all EDA notebooks: findings, conclusions, assumptions, and flaws"
tags: [research, notebooks, eda, quality-audit, estimation-variance, duration-modeling, business-metrics]
status: complete
last_updated: 2026-03-24
last_updated_by: Claude
---

# Research: Notebook Audit Report

**Date**: 2026-03-24
**Git Commit**: f1d40a72
**Branch**: main

## Research Question

Summarize each notebook's findings, conclusions, assumptions, and potential flaws/mistakes across all 8 EDA notebooks.

---

## Executive Summary

The DWS EDA suite comprises 8 notebooks spanning metadata validation, data quality auditing, pipeline coverage analysis, business KPIs, deep-dive financial analysis, estimation variance analysis, distributional goodness-of-fit testing, and duration regression modeling. Together they form a progressive analytical pipeline: early notebooks validate raw data, middle notebooks compute business metrics, and later notebooks build toward predictive scheduling models.

**Key cross-cutting findings:**
- The database is analytically usable but messy — 128 tables, zero foreign keys, high null rates in most Tier 1 tables
- ~$195M in total project revenue across 1,809 projects; top 5 clients hold 66% of revenue
- Estimation is systematically optimistic: median +5 days late, 75.6% of sheets finish late
- Delivery performance is paradoxically excellent (96.7% on time) — the shop absorbs internal delays
- True gross margin is ~58.9% (labor + material only; overhead excluded)
- Log-normal is the best distributional fit for duration ratios
- **No regression model beats a simple "multiply planned × 1.45" baseline** — the strongest statistical conclusion in the entire suite

**Most critical flaws across all notebooks:**
1. Working-day vs calendar-day unit mismatch likely inflates all variance metrics (NB 04, 05, 06)
2. ~66% of bid-to-actual comparisons show exactly zero variance, suggesting the same field is being compared to itself (NB 03)
3. Boolean string comparison bug silently produces wrong PK/unique index counts (NB 00)
4. Catastrophic mean inflation in small-project buckets from near-zero denominators (NB 03, 04)
5. MAE is actually Median Absolute Error throughout the regression analysis (NB 06)

---

## Notebook-by-Notebook Analysis

---

### NB 00: Metadata Validation (`00_metadata_validation.ipynb`)

**Purpose:** Phase 1 validation checklist confirming extraction completeness before catalog generation. Reads all SQL Server and Access metadata files and prints inventory summaries. Produces no file outputs.

**Key Findings:**
- 128 unique tables in `columns.csv`, 1,726 total columns
- 122 CSV exports in `Tables/`
- `tblTimesheet` is the largest table: 288,977 rows, ~112 MB
- Zero foreign keys, zero stored procedures
- 3 of 9 SQL Server metadata files are intentionally empty (`foreign_keys.csv`, `stored_procs.csv`, `column_extended.csv`)
- 59 Access-local tables, 1,055 Access queries

**Assumptions:**
- `TABLE_NAME` uniqueness equals table count (ignoring schema — `tblSheetHist` is under `DWS\jasulin`, not `dbo`)
- CSV filename stems exactly match SQL Server table names
- Empty FK file is an expected database state, not an extraction error

**Flaws:**
- **Boolean string comparison bug (cell 13):** `indexes_df[indexes_df['is_primary_key'] == True]` compares string `"True"` against Python `True`, always returning empty. PK index count silently reports 0 instead of ~88.
- **Duplicate print label:** Both total-distinct-indexes and unique-filtered-indexes print as `"Unique indexes:"` — output is ambiguous.
- **Wrong column name:** Checks for `sql_definition` but the actual column is `sql_reconstructed` — sample query SQL always prints `"N/A"`.
- **Row count discrepancy not flagged:** `tblTimesheet` shows 2,022,839 in `table_sizes.csv` vs 288,977 in `row_counts.csv` (different extraction times). No cross-check.

---

### NB 01: Data Quality Audit (`01_quality_audit.ipynb`)

**Purpose:** Structured quality audit of the 11 Tier 1 tables measuring referential integrity (FK orphans), column completeness (null patterns), temporal coverage, and duplicate detection. Outputs a quality heatmap.

**Key Findings:**
- 52 of 103 candidate FK relationships checked; 41 have orphans
- `tblPO` has the worst completeness: 9.1% (56 of 66 columns >50% null)
- `tblContact` completeness: 7.4%; `tblProject`: 15.5%
- Zero PK duplicates and zero full-row duplicates across all 11 tables
- Temporal coverage spans 2000–2024 for most tables; `tblBid` starts at 1900-01-03 (sentinel value)
- `tblCompany` is the only table with 100% referential integrity

**Assumptions:**
- Every JOIN in a SQL view constitutes an expected referential constraint
- Profile-derived `null_pct` is accurate (no drift from CSV export time)
- Only the first PK column is checked for duplicates (composite PKs missed)
- 50% non-null threshold for "usable" date columns is appropriate
- All nulls represent missing data (vs. intentionally optional fields)

**Flaws:**
- **51 of 103 FK relationships silently skipped** — no log of which ones or why
- **Integrity score is unweighted across FK links:** A single broken lookup link (e.g., `tblContact.WorkstationNo → tblWorkstation` at 100% orphan rate with only 15 rows) drags down the table's score equally to a core operational join
- **`tblTimesheet.Project → tblLaborRateCard` is a category error** — a rate card lookup is not a referential parent of every timesheet; 96.94% orphan rate inflates tblTimesheet's failure score
- **`tmpWorkProjChart` is a temp table** — checking FK integrity against it is meaningless
- **`tblBid` date range starts at 1900** — a well-known Access/SQL Server unset-date sentinel, reported as real data
- **`annotations` loaded but never used**

---

### NB 02: Pipeline Trace (`02_pipeline_trace.ipynb`)

**Purpose:** Measures how consistently projects flow through 6 lifecycle stages (Bid → PO → Purchasing → Timesheet → Billing → BillHistory) by set-intersection joins against `tblProject.ProjectID`.

**Key Findings:**
- 828 projects (45.8%) have all 6 stages present
- 1,481 projects (81.9%) have both timesheet + billing ("core for margin analysis")
- Bid coverage: 99.9%; Timesheet: 83.7%; PO/Purchasing: ~51%
- Dominant pattern: full pipeline (45.8%), followed by bid+timesheet+billing+billhist without PO/purchasing (34.8%)

**Assumptions:**
- `tblBid.JobNo` (not `tblBid.Project`) is the correct column for linking bids to projects
- The 6-stage model is exhaustive (excludes engineering tables, `tblSheetNEW`, `tblContact`)
- Set-membership presence = meaningful activity (no check for non-zero amounts/hours)
- `tblBill.BilledToDate` represents invoicing activity (may actually be billing schedule/process setup)

**Flaws:**
- **`tblBid.JobNo` is 56.5% null** — 2,399 of 4,243 bids silently dropped by `.dropna()`. The 99.9% bid coverage figure counts only explicitly linked bids, not all bids.
- **Sample timelines only show 2010 projects** — non-representative artifact of set iteration order
- **"Core for margin" doesn't verify non-zero amounts** — a project with $0 billing and 0-hour timesheet still counts
- **Pipeline order not validated temporally** — bid after timesheet would still count as "full pipeline"

---

### NB 03: Business Metrics (`03_business_metrics.ipynb`)

**Purpose:** Computes 6 operational KPIs: revenue concentration, margin proxies, estimate-to-actual variance, labor productivity, material cost ratios, and change order behavior.

**Key Findings:**
- Total project revenue (`TotProjCost`): **$194.7M** across 1,626 projects with non-zero cost
- Top 5 clients: **66.0%** of revenue; single largest (Skyline Construction SF): **28.89%**
- Estimate-to-actual: mean variance **−6.1%**, median **0.0%**, only 4.9% over budget
- Labor productivity: median **6.4 hrs/$1K** revenue (stable across project sizes)
- Material as % of project cost: median **12.4%**
- Change orders: 4,072 COs across 639 projects, total value **$48.8M**

**Assumptions:**
- `TotProjCost` is the revenue proxy (not actual billing records)
- `tblBid.JobNo` non-null = awarded bid
- `Price × Qty` in `tblPurchasing` = material cost (no distinction from subcontractor POs)
- All purchasing rows included regardless of PO status (open, cancelled, etc.)

**Flaws:**
- **~66% of projects show exactly zero estimate-to-actual variance** — strongly suggests `tblBid.TotProjCost` and `tblProject.TotProjCost` are the same value written to both tables at award, not independently tracked estimate vs. actual
- **Catastrophic mean inflation in `<$50K` buckets:** mean labor productivity = 1,921,088 hrs/$1K; mean material % = 244,673% (tiny denominators). Stored in JSON without flagging.
- **Duplicate client entities:** Hathaway Dinwiddie appears as two separate companies (combined $20.2M / 10.37% would rank #2). No deduplication.
- **`BilledToDate` double-counting risk:** If cumulative per billing stage, summing across stages overstates billing
- **4 tables loaded but never used** (`tblBillHistory`, `tblBillHistDetail`, `tblPO`, `tblContact`)
- **No temporal dimension** — all metrics are lifetime aggregates with no year/cohort/trend analysis
- **NaN vendor in top 10 spend** ($417K unmatched vendor ID)

---

### NB 04a: Deep Dives (`04_deep_dives.ipynb`)

**Purpose:** Six targeted deep-dive analyses clarifying TotProjCost semantics, computing true labor cost using pay rate history, corrected labor productivity, material coverage, CO capture rates, and an approximation of gross project margin.

**Key Findings:**
- `FixtureCost + NonFixtureCost = TotProjCost` within 1% for 80% of projects; most projects are entirely NonFixtureCost
- `TotProjCost` matches `sum(BilledToDate)` within 5% for 98.5% of projects — confirming it as base contract value
- `TotProjCost` does NOT include change orders: `TotProjCost / bid` median = 0.885 for CO projects
- True labor cost (using `tblEmpPayHist` rate history): **$66.1M** total, avg rate **$42.79/hr**, 99.5% of timesheet entries matched to a pay period
- Corrected labor productivity: median **6.4 hrs/$1K**, labor cost as % of revenue: median **29.2%**
- Material coverage (purchasing vs budgeted): median **79.4%** (only 291 projects have both data sources)
- CO capture: aggregate **23.7%** of base contract value; median per-project **12.3%**; CO billing completeness median **100%**
- **True gross margin: aggregate 58.9%, median 58.4%** (813 projects with both labor + purchasing data)
  - Caveat: excludes overhead, benefits, equipment, subcontractors

**Assumptions:**
- `PayRate1` = shop rate (ignoring `PayRate2`, which may be overtime)
- Only `RegHrs` used (no overtime column)
- `tblBill.Stage='M'` identifies budgeted material line items
- Non-void CO statuses include Draft (negative $111K total)
- Revenue = TotProjCost + non-void CO amount

**Flaws:**
- **Same `<$50K` mean explosion** as NB 03 (1.9M hrs/$1K, 244,673% material)
- **First-row bid selection is non-deterministic:** `groupby("JobNo")["TotProjCost"].first()` takes arbitrary row order, not sorted by date/revision
- **Selection bias in margin:** Only 813 of 1,566+ labor projects have purchasing data — larger/more labor-intensive projects overrepresented (90.7% of labor cost from 51.9% of projects)
- **Negative `purchasing["line_total"]` possible** from credit memos — not filtered
- **`bill_hist` and `contact` loaded but never used**

---

### NB 04b: Estimation Variance (`04_estimation_variance.ipynb`)

**Purpose:** Analyzes schedule estimation bias on production sheets — whether planned durations systematically differ from actual elapsed time, segmented by year, PM, estimator, sheet type, and complexity.

**Key Findings:**
- 14,042 completed sheets analyzed (of 15,932 total)
- **Mean schedule variance: +23.2 days late** (median: +5 days)
- 24.4% on time or early; 25.4% late by 30+ days
- **Calibration regression: actual = 1.164 × planned + 15.2** (R² = 0.702)
  - Estimates systematically optimistic (slope > 1); ~15 days fixed overhead not captured
- **Shop phase is where lateness originates:** only 9.3% on time vs 56.7% for engineering
- **Delivery performance: 96.7% on time or early** — shop absorbs internal delays without missing customer commitments
- 2011 is a severe outlier: mean 109.8 days late (4–10× other years)
- Eng-shop variance correlation: weak (r=0.169)

**Assumptions:**
- "Completed" = has both `SheetStart` and `SheetCompleteDate` (no status field check)
- Calendar days = planned days (THIS IS THE BIGGEST RISK — see flaws)
- `skip_col == -1` is the only skip sentinel
- `ToShopAct` is the engineering/shop phase boundary
- `Delivery` = customer-promised, `Delivered` = actual delivery

**Flaws:**
- **CRITICAL: Working-day vs calendar-day unit mismatch.** Planned total median = 7d, actual median = 18d (2.6× ratio). Consistent with duration columns being working days (5-day weeks) while actual elapsed is calendar days. If so, all variance metrics are systematically inflated.
- **`tblDefaulDurations` loaded (18 rows) but never used** — natural comparison target for planned vs template durations
- **2011 anomaly (mean 109.8d) noted but not investigated**
- **POST_STAGES (Storage + Field) excluded from phase-level analysis** — eng + shop variance ≠ total variance
- **PM boxplot ordering bug:** `pm_order` (sorted by median) is computed but not applied; boxplot uses default numeric key order
- **Date parsing without format** raises UserWarnings; ambiguous dates may be silently misinterpreted
- **NaN PM/Estimator silently excluded** from group summaries

---

### NB 05: Duration Goodness-of-Fit (`05_duration_goodness_of_fit.ipynb`)

**Purpose:** Tests which probability distribution (Exponential, Gamma, Log-normal, Weibull) best describes production cycle durations and duration ratios. Designed to select a likelihood for a future Bayesian hierarchical model.

**Key Findings:**
- **Log-normal wins decisively** on AIC, BIC, and held-out log-likelihood for both raw durations and ratios
- Raw duration Log-normal: s=1.835, scale=18.58; R² improvement over Exponential: massive
- Ratio Log-normal: s=0.768, scale=1.613
- All distributions rejected by KS test (all p ≈ 0) — no distribution fits perfectly
- LRT rejects Exponential in favor of Gamma (Λ=4,598.5 for durations)
- Bootstrap 95% CI on Log-normal vs Weibull LL difference excludes 0 → statistically significant
- Per-stage Gamma α̂ range: [0.592, 1.119]; std=0.150 → FULL_POOL verdict (single α across all stages)
- Segmented stability: 79% of PMs and 100% of SheetTypes agree Log-normal is best

**Assumptions:**
- `actual_total` inherited from NB 04 without re-verification
- All distributions fit with `loc=0` (no minimum cycle time)
- FULL_POOL threshold (std < 0.2) has no stated statistical derivation
- Train/test comment says "stratified by year" but code is simple random

**Flaws:**
- **"Stratified by year" comment is false** — code is `rng.random(len(dur)) < 0.8`, a simple random split
- **`dur` and `ratio` arrays split independently** — different records may be in train vs test across the two analyses
- **KS tests run on training data, not test data** — D statistics are optimistic
- **Segment KS re-fits within each segment** — `overall_best_D` is actually "Log-normal refitted to this segment," not "global fit evaluated on segment"
- **Pooling threshold is arbitrary** — no formal heterogeneity test; Storage (α=0.592) and CNC (α=1.119) differ by 0.527 despite FULL_POOL verdict
- **Per-stage analysis uses raw `tblSheetNEW` durations**, not the same `actual_total` construction as the sheet-level analysis
- **Bootstrap uses shared RNG state** — results change if cells rerun out of order

---

### NB 06: Duration Regression (`06_duration_regression.ipynb`)

**Purpose:** Fits log-linear regression models predicting actual/planned duration ratio with temporal validation (train: pre-2020, test: 2020–2023). Compares OLS and mixed-effects models against naive baselines.

**Key Findings:**
- **Global Median Ratio (planned × 1.449) is the best predictor** — MedAE 1.9 days, 55% within 20%
- All regression models perform worse than the naive baseline
- Model 2 (SheetType + complexity): R² = 0.0005 — structured predictors add essentially nothing
- Model 3 (PM random intercept): **did not converge** — results used anyway
- Calibration slope = 3.275 (target: 1.0) — severe miscalibration
- Prediction intervals massively over-cover: 50% nominal PI achieves 91.2% actual coverage
- Systematic negative bias across all segments: models overpredict duration

**Key conclusion:** "A simple multiplier of 1.45× planned duration outperforms every statistical model tested."

**Assumptions:**
- Log-normality of ratio assumed from NB 05 (not re-tested)
- Temporal split (pre/post 2020) assumes stationarity
- Model 3 test predictions use fixed effects only (PM BLUPs discarded)
- `year` from completion date, not start date

**Flaws:**
- **MAE is actually Median AE throughout** — `np.median(np.abs(residuals))` labeled as MAE in all outputs. The "best model" selection and all comparison tables use this mislabeled metric.
- **Model 3 did not converge** — two `ConvergenceWarning` messages in output; parameters and predictions used without flagging
- **OLS on log_ratio estimates the mean, not the median** — comparing `exp(mean of log)` = 1.712 against a baseline of `median ratio` = 1.449 is an apples-to-oranges comparison of central tendencies
- **PI construction uses only residual sigma** — missing the `X(X'X)⁻¹X'` term for parameter uncertainty (though the severe over-coverage suggests a different underlying issue)
- **Fan chart uses intercept only** — ignores SheetType and complexity effects (though they are near-zero)
- **`pct_late` is model-relative**, not absolute (actual > predicted, not actual > planned)

---

## Cross-Notebook Issues

### 1. The Calendar-Day vs Working-Day Question
Notebooks 04, 05, and 06 all build on planned stage durations that may be in working days while comparing against calendar-day actuals. This is the single most consequential unresolved assumption in the entire analytical suite. If confirmed, all variance metrics, distributional parameters, and regression coefficients would need recalibration (roughly: multiply planned by 1.4).

### 2. The `TotProjCost` Identity Problem
NB 03 finds 66% of bid-to-actual comparisons show exactly zero variance. NB 04a confirms `TotProjCost` closely tracks billing and is the base contract value. The estimate-to-actual analysis in NB 03 is likely comparing a field to itself, making the variance metric unreliable for assessing bidding accuracy.

### 3. The `<$50K` Denominator Explosion
Notebooks 03 and 04a both report astronomically inflated means for the smallest project bucket (1.9M hrs/$1K, 244K% material cost). These values are serialized to JSON output files without any flagging or capping.

### 4. Progressive Data Loss
The analytical pipeline progressively filters records: 15,932 sheets → 14,042 completed → 13,624 positive actual → 12,664 positive both → 12,609 with PM → 8,697 train / 3,912 test. Each filter is individually justified but the cumulative effect means the regression models operate on 55% of the original population.

### 5. Loaded-But-Unused Tables
Multiple tables are loaded and never used across notebooks: `tblBillHistory`, `tblBillHistDetail`, `tblPO`, `tblContact` (NB 03), `tblDefaulDurations` (NB 04b), `bill_hist`, `contact` (NB 04a), `annotations` (NB 01). These indicate planned analyses that were never completed.

---

## Code References

- `notebooks/00_metadata_validation.ipynb` — metadata checklist
- `notebooks/01_quality_audit.ipynb` — quality heatmap and FK integrity
- `notebooks/02_pipeline_trace.ipynb` — lifecycle stage coverage
- `notebooks/03_business_metrics.ipynb` — business KPIs
- `notebooks/04_deep_dives.ipynb` — TotProjCost semantics, true labor cost, margin
- `notebooks/04_estimation_variance.ipynb` — schedule variance analysis
- `notebooks/05_duration_goodness_of_fit.ipynb` — distributional testing
- `notebooks/06_duration_regression.ipynb` — predictive modeling
- `scripts/eda_helpers.py` — shared loading/saving utilities
- `scripts/config.py` — path configuration
- `eda/*.json` — structured outputs from each notebook
- `eda/*.md` — human-readable reports
- `eda/*.png` — visualizations
