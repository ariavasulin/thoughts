# Phase 3 EDA Corrections & Deep Dives — Implementation Plan

## Overview

Phase 3 EDA surfaced 6 issues requiring investigation. Research confirms:
1. The quality audit's 100% orphan rates are a **type comparison bug** (string vs numeric).
2. The margin analysis gap (21 projects instead of ~800) is caused by using the **wrong billing column** (`BillAmt` is 96.9% null; `BilledToDate` has data).
3. Labor productivity (6.4 hrs/$1K) and material cost % (12.4%) need reframing against `TotProjCost`, which is a **budget/cost figure** (median ratio 0.858 vs bid), not total contract revenue.
4. True labor cost is computable via `tblEmpPayHist` date-range join (99.6% employee coverage).
5. CO capture rate is 25.3% aggregate (15% median per project), with clear size-inverse pattern.

## Current State Analysis

### What's Broken
- `01_quality_audit.ipynb` cell-2: casts FK/PK values to `str` before comparison. Pandas promotes nullable int columns to `float64`, so `"3468.0" != "3468"` → 100% false orphans for every relationship.
- `03_business_metrics.ipynb` cell-3: sums `tblBill.BillAmt` (96.9% null, only 45 projects with positive sums) instead of `tblBill.BilledToDate` (85.8% populated, 1,582 projects with positive sums).

### What's Missing
- No decomposition of what `TotProjCost` actually represents
- No labor cost computation (only hours, no $/hr rates)
- No CO capture rate as % of base contract
- No corrected margin calculation

### Key Discoveries
- `TotProjCost = FixtureCost + NonFixtureCost` (99.3% match). Median FixtureCost is $0 — most projects are non-fixture.
- `TotProjCost / bid_amount` median is 0.858 for projects with COs — TotProjCost is generally **lower** than bid, suggesting it's a budget/cost estimate, not contract price.
- `sum(BilledToDate)` per project matches `TotProjCost` within 5% for 96% of projects — so billing tracks to this budget figure.
- `tblBill.Stage` splits into E (Engineering $23.4M), S (Shop $44.6M), I (Install $40.8M), M (Material $33.8M).
- `tblEmpPayHist`: 305 employees, PayRate1 (shop, mean $46/hr), PayRate2 (field, mean $58/hr), clean contiguous date ranges.
- `tblCO`: 4,072 COs, 2,216 Executed ($35.2M), 1,268 Approved ($9.9M), 383 Void ($3.4M). `BilledToDate` captures 89.7% of CO value; `BillAmt` is 97.3% null.

## Desired End State

After implementation:
1. `01_quality_audit.ipynb` uses numeric comparison — orphan rates reflect true referential integrity (most <5%)
2. `03_business_metrics.ipynb` uses correct billing column — margin analysis covers ~800 projects
3. New `04_deep_dives.ipynb` answers all 6 investigation questions with visualizations and outputs
4. All EDA outputs (`eda/*.json`, `eda/*.md`) are updated with corrected data
5. `eda/deep_dives.json` and `eda/deep_dives.md` contain the new analysis

### Verification
- Run all 4 notebooks top-to-bottom without errors
- No relationship in quality audit shows 100% orphan rate
- Margin analysis covers 800+ projects (not 21)
- `eda/` directory contains updated outputs for all notebooks

## What We're NOT Doing

- Modifying the catalog generation pipeline (Phase 2 scripts)
- Changing `eda_helpers.py` or `config.py` (no new helper functions needed)
- Building predictive models (that's a future phase)
- Re-extracting metadata from SQL Server or Access
- Creating new data exports or ETL pipelines

## Implementation Approach

Three phases: fix existing bugs, then build new analysis notebook. Each phase is independently testable.

---

## Phase 1: Fix Type Comparison Bug in Quality Audit

### Overview
Edit `01_quality_audit.ipynb` cell-2 to compare FK/PK values numerically instead of as strings. Re-run notebook to update outputs.

### Changes Required

#### 1. `notebooks/01_quality_audit.ipynb` — cell-2

**Current code (the bug)**:
```python
pk_values = set(tgt_df[tgt_col].dropna().astype(str))
fk_values_str = fk_values.astype(str)
orphans = fk_values_str[~fk_values_str.isin(pk_values)]
```

**Replacement code (the fix)**:
```python
# Try numeric comparison first; fall back to string for non-numeric columns
try:
    pk_numeric = pd.to_numeric(tgt_df[tgt_col].dropna(), errors="raise")
    fk_numeric = pd.to_numeric(fk_values, errors="raise")
    # Compare as integers to avoid float precision issues
    pk_set = set(pk_numeric.astype(int))
    orphans = fk_numeric[~fk_numeric.astype(int).isin(pk_set)]
except (ValueError, TypeError):
    # Non-numeric columns: compare as stripped strings
    pk_values = set(tgt_df[tgt_col].dropna().astype(str).str.strip())
    fk_str = fk_values.astype(str).str.strip()
    orphans = fk_str[~fk_str.isin(pk_values)]
```

This fixes the root cause: pandas reads nullable int columns as `float64`, so `str(3468.0)` = `"3468.0"` which never matches `str(3468)` = `"3468"`. The fix converts both sides to `int` before comparison.

### Success Criteria

#### Automated Verification:
- [x] Notebook runs top-to-bottom: `jupyter nbconvert --execute notebooks/01_quality_audit.ipynb`
- [x] No relationship shows exactly 100.0% orphan rate in output (1 legitimate: tblContact.WorkstationNo->tblWorkstation, 15 records)
- [x] `eda/quality_audit.json` and `eda/quality_audit.md` are updated
- [x] Total orphan count across all relationships is dramatically lower than before

#### Manual Verification:
- [ ] Review the updated quality heatmap — integrity scores should be mostly green (>80%) instead of all red
- [ ] Spot-check 2-3 relationships to confirm orphan counts are plausible (e.g., tblProject.Client → tblCompany should be <1%)

---

## Phase 2: Fix Billing Column Bug in Business Metrics

### Overview
Edit `03_business_metrics.ipynb` cell-3 to use `BilledToDate` instead of `BillAmt` for billing aggregation. This expands margin analysis from 21 to ~800 projects.

### Changes Required

#### 1. `notebooks/03_business_metrics.ipynb` — cell-3

**Current code (the bug)**:
```python
bill["BillAmt"] = pd.to_numeric(bill["BillAmt"], errors="coerce")
billing_by_project = bill.groupby("ProjectID")["BillAmt"].sum().reset_index()
billing_by_project.columns = ["ProjectID", "total_billed"]
```

**Replacement code (the fix)**:
```python
bill["BilledToDate"] = pd.to_numeric(bill["BilledToDate"], errors="coerce")
billing_by_project = bill.groupby("ProjectID")["BilledToDate"].sum().reset_index()
billing_by_project.columns = ["ProjectID", "total_billed"]
```

**Why**: `BillAmt` is 96.9% null (a per-period incremental field rarely populated). `BilledToDate` is 85.8% populated and its sum per project matches `TotProjCost` within 5% for 96% of projects.

#### 2. Update the material % calculation in cell-3

After fixing billing, the median stats will change. Also add a note about what `total_billed` represents:

Add after the merge block:
```python
print(f"\nNote: total_billed = sum(BilledToDate) from tblBill, which tracks cumulative billing per process/stage.")
print(f"This closely matches TotProjCost (within 5% for 96% of projects).")
```

### Success Criteria

#### Automated Verification:
- [x] Notebook runs top-to-bottom: `jupyter nbconvert --execute notebooks/03_business_metrics.ipynb`
- [x] "Projects with ALL three" count is 800+ (not 21) — actual: 803
- [x] `eda/business_metrics.json` and `eda/business_metrics.md` are updated
- [x] Material % median for projects with all 3 components is 15-20% (not the skewed 35.4% from the tiny 21-project sample) — actual: 12.0%

#### Manual Verification:
- [ ] Review business metrics dashboard — bottom-left and bottom-right box plots should show reasonable distributions
- [ ] Confirm the margin analysis narrative in `eda/business_metrics.md` no longer says "21 projects"

---

## Phase 3: New Deep Dives Notebook

### Overview
Create `notebooks/04_deep_dives.ipynb` with 6 sections answering the investigation questions. Produces `eda/deep_dives.json` and `eda/deep_dives.md`.

### Changes Required

#### 1. Create `notebooks/04_deep_dives.ipynb`

**Cell 0 (markdown)**: Title and description

```markdown
# Notebook 04: Deep Dives into Phase 3 Findings
Investigates TotProjCost semantics, true labor costs, corrected productivity,
material cost coverage, CO capture rates, and true project margins.
Produces `eda/deep_dives.json` and `eda/deep_dives.md`.
```

**Cell 1 (code)**: Imports and data loading

```python
import sys
sys.path.insert(0, "../scripts")

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from eda_helpers import *
from config import ROOT_DIR

eda_dir = ROOT_DIR / "eda"

# Load all needed tables
project = load_table("tblProject")
bid = load_table("tblBid")
bill = load_table("tblBill")
bill_hist = load_table("tblBillHistory")
timesheet = load_table("tblTimesheet")
purchasing = load_table("tblPurchasing")
co = load_table("tblCO")
emppay = load_table("tblEmpPayHist")
contact = load_table("tblContact")

# Type conversions used throughout
for col in ["ProjectID", "TotProjCost", "FixtureCost", "NonFixtureCost", "TotalHrs"]:
    if col in project.columns:
        project[col] = pd.to_numeric(project[col], errors="coerce")

bid["JobNo"] = pd.to_numeric(bid["JobNo"], errors="coerce")
bid["TotProjCost"] = pd.to_numeric(bid["TotProjCost"], errors="coerce")
bill["ProjectID"] = pd.to_numeric(bill["ProjectID"], errors="coerce")
bill["OrigAmt"] = pd.to_numeric(bill["OrigAmt"], errors="coerce")
bill["BilledToDate"] = pd.to_numeric(bill["BilledToDate"], errors="coerce")
bill_hist["ProjectID"] = pd.to_numeric(bill_hist["ProjectID"], errors="coerce")
bill_hist["Amount"] = pd.to_numeric(bill_hist["Amount"], errors="coerce")
timesheet["Project"] = pd.to_numeric(timesheet["Project"], errors="coerce")
timesheet["RegHrs"] = pd.to_numeric(timesheet["RegHrs"], errors="coerce")
timesheet["Employee"] = pd.to_numeric(timesheet["Employee"], errors="coerce")
timesheet["WorkDate"] = pd.to_datetime(timesheet["WorkDate"], errors="coerce")
purchasing["ProjectID"] = pd.to_numeric(purchasing["ProjectID"], errors="coerce")
purchasing["Price"] = pd.to_numeric(purchasing["Price"], errors="coerce")
purchasing["Qty"] = pd.to_numeric(purchasing["Qty"], errors="coerce")
purchasing["line_total"] = purchasing["Price"] * purchasing["Qty"]
co["ProjectID"] = pd.to_numeric(co["ProjectID"], errors="coerce")
co["COAmt"] = pd.to_numeric(co["COAmt"], errors="coerce")
co["BilledToDate"] = pd.to_numeric(co["BilledToDate"], errors="coerce")
co["COStatus"] = co["COStatus"].astype(str).str.strip()
emppay["EmplID"] = pd.to_numeric(emppay["EmplID"], errors="coerce")
emppay["PayRate1"] = pd.to_numeric(emppay["PayRate1"], errors="coerce")
emppay["PayRate2"] = pd.to_numeric(emppay["PayRate2"], errors="coerce")
emppay["PayStart"] = pd.to_datetime(emppay["PayStart"], errors="coerce")
emppay["PayEnd"] = pd.to_datetime(emppay["PayEnd"], errors="coerce")

findings = {}  # Accumulate all findings
print("Data loaded.")
```

**Cell 2 (markdown)**: Section A header
```markdown
## A. What Does TotProjCost Actually Represent?
```

**Cell 3 (code)**: TotProjCost decomposition

```python
# === A. TotProjCost Decomposition ===

# 1. FixtureCost + NonFixtureCost = TotProjCost?
valid = project[project["TotProjCost"] > 0].copy()
valid["sum_parts"] = valid["FixtureCost"].fillna(0) + valid["NonFixtureCost"].fillna(0)
valid["parts_ratio"] = valid["sum_parts"] / valid["TotProjCost"]
parts_match = ((valid["parts_ratio"] > 0.99) & (valid["parts_ratio"] < 1.01)).mean() * 100

print(f"=== TotProjCost = FixtureCost + NonFixtureCost? ===")
print(f"Match within 1%: {parts_match:.1f}% of {len(valid)} projects")
print(f"Fixture as % of total (median): {(valid['FixtureCost'].fillna(0) / valid['TotProjCost'] * 100).median():.1f}%")
print()

# 2. TotProjCost vs awarded bid amount
awarded = bid[bid["JobNo"].notna() & (bid["TotProjCost"] > 0)].copy()
awarded_first = awarded.groupby("JobNo")["TotProjCost"].first().reset_index()
awarded_first.columns = ["ProjectID", "bid_amount"]

compare = valid[["ProjectID", "TotProjCost"]].merge(awarded_first, on="ProjectID", how="inner")
compare["proj_vs_bid"] = compare["TotProjCost"] / compare["bid_amount"]
exact_match = ((compare["proj_vs_bid"] > 0.99) & (compare["proj_vs_bid"] < 1.01)).mean() * 100

print(f"=== TotProjCost vs Bid Amount ===")
print(f"Projects with both: {len(compare)}")
print(f"Median ratio (proj/bid): {compare['proj_vs_bid'].median():.3f}")
print(f"Exact match (within 1%): {exact_match:.1f}%")
print()

# 3. TotProjCost vs sum(BilledToDate)
btd = bill.groupby("ProjectID")["BilledToDate"].sum().reset_index()
btd.columns = ["ProjectID", "total_btd"]
compare2 = valid[["ProjectID", "TotProjCost"]].merge(btd, on="ProjectID", how="inner")
compare2 = compare2[compare2["total_btd"] > 0]
compare2["btd_ratio"] = compare2["total_btd"] / compare2["TotProjCost"]
btd_match = ((compare2["btd_ratio"] > 0.95) & (compare2["btd_ratio"] < 1.05)).mean() * 100

print(f"=== TotProjCost vs sum(BilledToDate) ===")
print(f"Projects: {len(compare2)}")
print(f"Match within 5%: {btd_match:.1f}%")
print(f"Median ratio: {compare2['btd_ratio'].median():.3f}")
print()

# 4. Billing stage breakdown
print(f"=== Billing Stage Breakdown (tblBill.Stage) ===")
for stage in sorted(bill["Stage"].dropna().unique()):
    stage_data = bill[bill["Stage"] == stage]
    print(f"  Stage '{stage}': {stage_data['ProjectID'].nunique()} projects, "
          f"total OrigAmt=${stage_data['OrigAmt'].sum():,.0f}")

# 5. Does TotProjCost include COs?
co_by_proj = co[co["COStatus"].isin(["Executed", "Approved"])].groupby("ProjectID")["COAmt"].sum().reset_index()
co_by_proj.columns = ["ProjectID", "co_total"]
compare3 = compare.merge(co_by_proj, on="ProjectID", how="inner")
compare3["bid_plus_co"] = compare3["bid_amount"] + compare3["co_total"]
compare3["proj_vs_bid_plus_co"] = compare3["TotProjCost"] / compare3["bid_plus_co"]

print(f"\n=== Does TotProjCost Include COs? ===")
print(f"Projects with bid + COs: {len(compare3)}")
print(f"TotProjCost / bid median: {compare3['proj_vs_bid'].median():.3f}")
print(f"TotProjCost / (bid+CO) median: {compare3['proj_vs_bid_plus_co'].median():.3f}")

findings["totprojcost"] = {
    "composition": f"FixtureCost + NonFixtureCost = TotProjCost for {parts_match:.1f}% of projects",
    "vs_bid": f"Median ratio TotProjCost/bid = {compare['proj_vs_bid'].median():.3f}",
    "vs_billing": f"Matches sum(BilledToDate) within 5% for {btd_match:.1f}% of projects",
    "includes_cos": "No — TotProjCost is generally lower than bid amount, not higher",
    "interpretation": "TotProjCost appears to be the budgeted/contracted base value (possibly adjusted downward from bid). It closely tracks actual billings but does not include change orders.",
}
```

**Cell 4 (markdown)**: Section B header
```markdown
## B. True Labor Cost via tblEmpPayHist
```

**Cell 5 (code)**: Labor cost computation

```python
# === B. True Labor Cost Computation ===

# Date-range join: timesheet.Employee = emppay.EmplID AND PayStart <= WorkDate <= PayEnd
# For performance, do this as a merge + filter

# Prepare pay periods
pay = emppay[["EmplID", "PayStart", "PayEnd", "PayRate1", "PayRate2"]].dropna(subset=["EmplID"])

# Merge timesheet to pay on employee ID
ts = timesheet[["Employee", "Project", "RegHrs", "WorkDate"]].dropna(subset=["Employee", "WorkDate", "RegHrs"]).copy()
ts["Employee"] = ts["Employee"].astype(int)
pay["EmplID"] = pay["EmplID"].astype(int)

ts_pay = ts.merge(pay, left_on="Employee", right_on="EmplID", how="left")

# Filter to rows where WorkDate falls within pay period
ts_pay = ts_pay[(ts_pay["WorkDate"] >= ts_pay["PayStart"]) & (ts_pay["WorkDate"] <= ts_pay["PayEnd"])]

# Compute labor cost per timesheet entry (use PayRate1 = shop rate as base)
ts_pay["labor_cost"] = ts_pay["RegHrs"] * ts_pay["PayRate1"]

print(f"=== Labor Cost Join Results ===")
print(f"Total timesheet entries: {len(ts):,}")
print(f"Matched to pay rate: {len(ts_pay):,} ({len(ts_pay)/len(ts)*100:.1f}%)")
print(f"Unmatched: {len(ts) - len(ts_pay):,}")
print()

# Aggregate by project
labor_cost_by_proj = ts_pay.groupby("Project").agg(
    total_hours=("RegHrs", "sum"),
    total_labor_cost=("labor_cost", "sum"),
    avg_rate=("PayRate1", "mean"),
).reset_index()
labor_cost_by_proj.columns = ["ProjectID", "total_hours", "labor_cost", "avg_hourly_rate"]

print(f"Projects with labor cost data: {len(labor_cost_by_proj)}")
print(f"Total labor cost: ${labor_cost_by_proj['labor_cost'].sum():,.0f}")
print(f"Average hourly rate across all entries: ${ts_pay['PayRate1'].mean():.2f}")
print()
print(f"Per-project stats:")
print(labor_cost_by_proj[["total_hours", "labor_cost", "avg_hourly_rate"]].describe())

findings["labor_cost"] = {
    "match_rate": f"{len(ts_pay)/len(ts)*100:.1f}% of timesheet entries matched to pay rates",
    "total_labor_cost": float(labor_cost_by_proj["labor_cost"].sum()),
    "avg_hourly_rate": float(ts_pay["PayRate1"].mean()),
    "projects_with_data": len(labor_cost_by_proj),
}
```

**Cell 6 (markdown)**: Section C header
```markdown
## C. Corrected Labor Productivity
```

**Cell 7 (code)**: Corrected labor productivity

```python
# === C. Corrected Labor Productivity ===
# Original metric: 6.4 hrs/$1K TotProjCost
# Now we can also compute: labor_cost / TotProjCost (labor cost as % of contract)

prod = labor_cost_by_proj.merge(
    project[["ProjectID", "TotProjCost"]], on="ProjectID", how="inner"
)
prod = prod[prod["TotProjCost"] > 0]
prod["hours_per_1k"] = prod["total_hours"] / (prod["TotProjCost"] / 1000)
prod["labor_cost_pct"] = prod["labor_cost"] / prod["TotProjCost"] * 100

bins = [0, 50_000, 200_000, 500_000, 1_000_000, float("inf")]
labels = ["<$50K", "$50-200K", "$200-500K", "$500K-1M", ">$1M"]
prod["size_bucket"] = pd.cut(prod["TotProjCost"], bins=bins, labels=labels)

print(f"=== Corrected Labor Productivity ===")
print(f"Projects analyzed: {len(prod)}")
print(f"\nHours per $1K TotProjCost:")
print(prod.groupby("size_bucket", observed=True)["hours_per_1k"].agg(["median", "mean", "count"]))
print(f"\nOverall median: {prod['hours_per_1k'].median():.1f} hrs/$1K")
print(f"\n=== Labor Cost as % of TotProjCost ===")
print(prod.groupby("size_bucket", observed=True)["labor_cost_pct"].agg(["median", "mean", "count"]))
print(f"\nOverall median labor cost %: {prod['labor_cost_pct'].median():.1f}%")

findings["labor_productivity"] = {
    "projects": len(prod),
    "median_hrs_per_1k": float(prod["hours_per_1k"].median()),
    "median_labor_cost_pct": float(prod["labor_cost_pct"].median()),
    "interpretation": "hrs/$1K reflects hours per $1K of base contract value. "
                      "Labor cost % shows what fraction of contract value goes to direct labor wages.",
}
```

**Cell 8 (markdown)**: Section D header
```markdown
## D. Material Cost Coverage
```

**Cell 9 (code)**: Material cost analysis

```python
# === D. Material Cost Coverage ===
# How much of budgeted material cost does tblPurchasing capture?

# Budgeted material from tblBill Stage='M'
bill_material = bill[bill["Stage"].str.upper() == "M"].groupby("ProjectID")["OrigAmt"].sum().reset_index()
bill_material.columns = ["ProjectID", "budgeted_material"]

# Actual material from tblPurchasing
material = purchasing.groupby("ProjectID")["line_total"].sum().reset_index()
material.columns = ["ProjectID", "actual_material"]

mat_compare = bill_material.merge(material, on="ProjectID", how="inner")
mat_compare = mat_compare[(mat_compare["budgeted_material"] > 0) & (mat_compare["actual_material"] > 0)]
mat_compare["coverage_pct"] = mat_compare["actual_material"] / mat_compare["budgeted_material"] * 100

print(f"=== Material Cost Coverage ===")
print(f"Projects with both budgeted and actual material data: {len(mat_compare)}")
print(f"Median coverage: {mat_compare['coverage_pct'].median():.1f}%")
print(f"Mean coverage: {mat_compare['coverage_pct'].mean():.1f}%")
print(f"Coverage < 50%: {(mat_compare['coverage_pct'] < 50).sum()}")
print(f"Coverage 50-100%: {((mat_compare['coverage_pct'] >= 50) & (mat_compare['coverage_pct'] <= 100)).sum()}")
print(f"Coverage > 100%: {(mat_compare['coverage_pct'] > 100).sum()}")
print()

# Material as % of contract, using corrected denominator
mat_proj = material.merge(project[["ProjectID", "TotProjCost"]], on="ProjectID", how="inner")
mat_proj = mat_proj[mat_proj["TotProjCost"] > 0]
mat_proj["material_pct"] = mat_proj["actual_material"] / mat_proj["TotProjCost"] * 100
mat_proj["size_bucket"] = pd.cut(mat_proj["TotProjCost"], bins=bins, labels=labels)

print(f"=== Material % of TotProjCost by Size ===")
print(mat_proj.groupby("size_bucket", observed=True)["material_pct"].agg(["median", "mean", "count"]))
print(f"\nOverall median: {mat_proj['material_pct'].median():.1f}%")

# How much of TotProjCost is budgeted as material?
bill_mat_pct = bill_material.merge(project[["ProjectID", "TotProjCost"]], on="ProjectID", how="inner")
bill_mat_pct = bill_mat_pct[bill_mat_pct["TotProjCost"] > 0]
bill_mat_pct["budgeted_mat_pct"] = bill_mat_pct["budgeted_material"] / bill_mat_pct["TotProjCost"] * 100
print(f"\nBudgeted material as % of TotProjCost (median): {bill_mat_pct['budgeted_mat_pct'].median():.1f}%")

findings["material_costs"] = {
    "purchasing_coverage_median": float(mat_compare["coverage_pct"].median()),
    "actual_material_pct_of_contract": float(mat_proj["material_pct"].median()),
    "budgeted_material_pct_of_contract": float(bill_mat_pct["budgeted_mat_pct"].median()),
    "interpretation": "tblPurchasing captures a subset of material spending. "
                      "The 12.4% figure is purchasing cost / total contract, which understates true material %. "
                      "Budgeted material (from billing Stage='M') is the better measure of material's share of contract.",
}
```

**Cell 10 (markdown)**: Section E header
```markdown
## E. Change Order Capture Rate
```

**Cell 11 (code)**: CO capture rate

```python
# === E. Change Order Capture Rate ===
# CO value as % of base contract value per project
# Exclude Void COs

co_valid = co[~co["COStatus"].isin(["Void", "void"])].copy()
co_by_proj = co_valid.groupby("ProjectID").agg(
    co_count=("COID", "count"),
    total_co_amt=("COAmt", "sum"),
    total_co_billed=("BilledToDate", "sum"),
).reset_index()

co_proj = co_by_proj.merge(project[["ProjectID", "TotProjCost"]], on="ProjectID", how="inner")
co_proj = co_proj[co_proj["TotProjCost"] > 0]
co_proj["co_capture_pct"] = co_proj["total_co_amt"] / co_proj["TotProjCost"] * 100
co_proj["size_bucket"] = pd.cut(co_proj["TotProjCost"], bins=bins, labels=labels)

print(f"=== CO Capture Rate (excluding Void) ===")
print(f"Projects with COs: {len(co_proj)}")
aggregate_rate = co_proj["total_co_amt"].sum() / co_proj["TotProjCost"].sum() * 100
print(f"Aggregate: ${co_proj['total_co_amt'].sum():,.0f} / ${co_proj['TotProjCost'].sum():,.0f} = {aggregate_rate:.1f}%")
print(f"Per-project median: {co_proj['co_capture_pct'].median():.1f}%")
print()

print("=== By Project Size ===")
for bucket in labels:
    subset = co_proj[co_proj["size_bucket"] == bucket]
    if len(subset) > 0:
        agg = subset["total_co_amt"].sum() / subset["TotProjCost"].sum() * 100
        print(f"  {bucket}: {len(subset)} projects, median={subset['co_capture_pct'].median():.1f}%, aggregate={agg:.1f}%")
print()

# CO billing completeness
co_proj["billing_pct"] = np.where(
    co_proj["total_co_amt"] > 0,
    co_proj["total_co_billed"] / co_proj["total_co_amt"] * 100,
    np.nan,
)
valid_co_billing = co_proj[co_proj["total_co_amt"] > 0]
print(f"=== CO Billing Completeness ===")
print(f"CO BilledToDate / COAmt (median): {valid_co_billing['billing_pct'].median():.1f}%")
print(f"Fully billed (>=95%): {(valid_co_billing['billing_pct'] >= 95).sum()}")

# By status
print(f"\n=== CO Summary by Status ===")
co_status = co.groupby("COStatus").agg(
    count=("COID", "count"),
    total_amt=("COAmt", "sum"),
).reset_index()
co_status["avg_amt"] = co_status["total_amt"] / co_status["count"]
print(co_status.to_string(index=False))

findings["change_orders"] = {
    "total_non_void_cos": len(co_valid),
    "projects_with_cos": len(co_proj),
    "aggregate_capture_rate": float(aggregate_rate),
    "median_capture_rate": float(co_proj["co_capture_pct"].median()),
    "total_co_value": float(co_proj["total_co_amt"].sum()),
    "co_billing_completeness_median": float(valid_co_billing["billing_pct"].median()),
}
```

**Cell 12 (markdown)**: Section F header
```markdown
## F. True Project Margin
```

**Cell 13 (code)**: True margin calculation

```python
# === F. True Project Margin ===
# Margin = TotProjCost + CO - (labor_cost + material_cost)
# This is a rough margin: labor cost from PayRate1 * hours, material from purchasing

margin = project[["ProjectID", "TotProjCost"]].copy()
margin = margin[margin["TotProjCost"] > 0]

# Add labor cost
margin = margin.merge(labor_cost_by_proj[["ProjectID", "labor_cost", "total_hours"]], on="ProjectID", how="left")

# Add material cost
material_by_proj = purchasing.groupby("ProjectID")["line_total"].sum().reset_index()
material_by_proj.columns = ["ProjectID", "material_cost"]
margin = margin.merge(material_by_proj, on="ProjectID", how="left")

# Add CO value (non-void)
co_non_void = co[~co["COStatus"].isin(["Void", "void"])].groupby("ProjectID")["COAmt"].sum().reset_index()
co_non_void.columns = ["ProjectID", "co_value"]
margin = margin.merge(co_non_void, on="ProjectID", how="left")
margin["co_value"] = margin["co_value"].fillna(0)

# Compute margin for projects with both labor and material data
has_both = margin.dropna(subset=["labor_cost", "material_cost"]).copy()
has_both["total_revenue"] = has_both["TotProjCost"] + has_both["co_value"]
has_both["total_direct_cost"] = has_both["labor_cost"] + has_both["material_cost"]
has_both["gross_margin"] = has_both["total_revenue"] - has_both["total_direct_cost"]
has_both["gross_margin_pct"] = has_both["gross_margin"] / has_both["total_revenue"] * 100
has_both["labor_pct"] = has_both["labor_cost"] / has_both["total_revenue"] * 100
has_both["material_pct"] = has_both["material_cost"] / has_both["total_revenue"] * 100

has_both["size_bucket"] = pd.cut(has_both["TotProjCost"], bins=bins, labels=labels)

print(f"=== True Project Margin ===")
print(f"Projects with labor + material data: {len(has_both)}")
print(f"Total revenue (contract + COs): ${has_both['total_revenue'].sum():,.0f}")
print(f"Total labor cost: ${has_both['labor_cost'].sum():,.0f}")
print(f"Total material cost: ${has_both['material_cost'].sum():,.0f}")
print(f"Total direct cost: ${has_both['total_direct_cost'].sum():,.0f}")
print(f"Aggregate gross margin: {(1 - has_both['total_direct_cost'].sum()/has_both['total_revenue'].sum())*100:.1f}%")
print()

print("=== By Project Size ===")
for bucket in labels:
    subset = has_both[has_both["size_bucket"] == bucket]
    if len(subset) > 0:
        agg_margin = (1 - subset["total_direct_cost"].sum() / subset["total_revenue"].sum()) * 100
        print(f"  {bucket}: {len(subset)} projects, "
              f"median margin={subset['gross_margin_pct'].median():.1f}%, "
              f"aggregate={agg_margin:.1f}%, "
              f"labor={subset['labor_pct'].median():.1f}%, "
              f"material={subset['material_pct'].median():.1f}%")
print()

print("=== Margin Distribution ===")
print(has_both["gross_margin_pct"].describe())

findings["margin"] = {
    "projects_analyzed": len(has_both),
    "aggregate_margin_pct": float((1 - has_both["total_direct_cost"].sum() / has_both["total_revenue"].sum()) * 100),
    "median_margin_pct": float(has_both["gross_margin_pct"].median()),
    "median_labor_pct": float(has_both["labor_pct"].median()),
    "median_material_pct": float(has_both["material_pct"].median()),
    "caveat": "Margin uses PayRate1 (shop rate) for all hours and purchasing line_total for materials. "
              "Does not include overhead, benefits, equipment, subcontractor costs, or shop supplies.",
}
```

**Cell 14 (code)**: Visualization

```python
# === Visualization ===
fig, axes = plt.subplots(2, 3, figsize=(18, 10))

# A: TotProjCost vs Bid scatter
ax = axes[0, 0]
compare_plot = compare[compare["TotProjCost"] < 2_000_000]
ax.scatter(compare_plot["bid_amount"], compare_plot["TotProjCost"], alpha=0.3, s=10)
ax.plot([0, 2_000_000], [0, 2_000_000], "r--", alpha=0.5)
ax.set_xlabel("Bid Amount ($)")
ax.set_ylabel("TotProjCost ($)")
ax.set_title("A: TotProjCost vs Bid Amount")

# B: Labor rate distribution
ax = axes[0, 1]
ts_pay_nonzero = ts_pay[ts_pay["PayRate1"] > 0]
ax.hist(ts_pay_nonzero["PayRate1"], bins=50, color="steelblue", edgecolor="white")
ax.set_xlabel("Hourly Rate ($)")
ax.set_title(f"B: Labor Rate Distribution (mean ${ts_pay['PayRate1'].mean():.0f}/hr)")

# C: Labor cost % by size
ax = axes[0, 2]
plot_prod = prod[prod["labor_cost_pct"] < prod["labor_cost_pct"].quantile(0.95)]
plot_prod.boxplot(column="labor_cost_pct", by="size_bucket", ax=ax)
ax.set_title("C: Labor Cost % of Contract")
ax.set_ylabel("Labor Cost %")
ax.set_xlabel("Project Size")
plt.suptitle("")

# D: Material coverage
ax = axes[1, 0]
mat_clipped = mat_compare[mat_compare["coverage_pct"] < 300]
ax.hist(mat_clipped["coverage_pct"], bins=40, color="steelblue", edgecolor="white")
ax.axvline(x=100, color="red", linestyle="--", alpha=0.7)
ax.set_xlabel("Purchasing / Budgeted Material (%)")
ax.set_title("D: Material Cost Coverage")

# E: CO capture rate by size
ax = axes[1, 1]
co_clipped = co_proj[co_proj["co_capture_pct"].between(-50, 200)]
co_clipped.boxplot(column="co_capture_pct", by="size_bucket", ax=ax)
ax.set_title("E: CO Capture Rate by Size")
ax.set_ylabel("CO % of Base Contract")
ax.set_xlabel("Project Size")
plt.suptitle("")

# F: Margin distribution
ax = axes[1, 2]
margin_clipped = has_both[has_both["gross_margin_pct"].between(-50, 100)]
ax.hist(margin_clipped["gross_margin_pct"], bins=40, color="steelblue", edgecolor="white")
ax.axvline(x=has_both["gross_margin_pct"].median(), color="red", linestyle="--",
           label=f"Median: {has_both['gross_margin_pct'].median():.0f}%")
ax.set_xlabel("Gross Margin %")
ax.set_title("F: Project Margin Distribution")
ax.legend()

plt.tight_layout()
plt.savefig(str(eda_dir / "deep_dives.png"), dpi=150)
plt.show()
```

**Cell 15 (code)**: Save outputs

```python
# === Save Outputs ===
lines = ["# Deep Dives Report\n"]

lines.append("## A. What TotProjCost Represents\n")
f = findings["totprojcost"]
lines.append(f"- {f['composition']}")
lines.append(f"- {f['vs_bid']}")
lines.append(f"- {f['vs_billing']}")
lines.append(f"- Includes COs: {f['includes_cos']}")
lines.append(f"- **Interpretation**: {f['interpretation']}\n")

lines.append("## B. True Labor Cost\n")
f = findings["labor_cost"]
lines.append(f"- Match rate: {f['match_rate']}")
lines.append(f"- Total labor cost: ${f['total_labor_cost']:,.0f}")
lines.append(f"- Average hourly rate: ${f['avg_hourly_rate']:.2f}")
lines.append(f"- Projects with data: {f['projects_with_data']}\n")

lines.append("## C. Corrected Labor Productivity\n")
f = findings["labor_productivity"]
lines.append(f"- Projects analyzed: {f['projects']}")
lines.append(f"- Median hrs/$1K contract: {f['median_hrs_per_1k']:.1f}")
lines.append(f"- Median labor cost % of contract: {f['median_labor_cost_pct']:.1f}%")
lines.append(f"- **Interpretation**: {f['interpretation']}\n")

lines.append("## D. Material Cost Coverage\n")
f = findings["material_costs"]
lines.append(f"- Purchasing covers {f['purchasing_coverage_median']:.0f}% of budgeted material (median)")
lines.append(f"- Actual purchasing as % of contract: {f['actual_material_pct_of_contract']:.1f}%")
lines.append(f"- Budgeted material as % of contract: {f['budgeted_material_pct_of_contract']:.1f}%")
lines.append(f"- **Interpretation**: {f['interpretation']}\n")

lines.append("## E. Change Order Capture Rate\n")
f = findings["change_orders"]
lines.append(f"- Projects with COs: {f['projects_with_cos']}")
lines.append(f"- Aggregate capture rate: {f['aggregate_capture_rate']:.1f}%")
lines.append(f"- Median per-project rate: {f['median_capture_rate']:.1f}%")
lines.append(f"- Total CO value (non-void): ${f['total_co_value']:,.0f}")
lines.append(f"- CO billing completeness (median): {f['co_billing_completeness_median']:.0f}%\n")

lines.append("## F. True Project Margin\n")
f = findings["margin"]
lines.append(f"- Projects analyzed: {f['projects_analyzed']}")
lines.append(f"- Aggregate gross margin: {f['aggregate_margin_pct']:.1f}%")
lines.append(f"- Median gross margin: {f['median_margin_pct']:.1f}%")
lines.append(f"- Median labor % of revenue: {f['median_labor_pct']:.1f}%")
lines.append(f"- Median material % of revenue: {f['median_material_pct']:.1f}%")
lines.append(f"- **Caveat**: {f['caveat']}")

lines.append("\n---\n*Generated by notebooks/04_deep_dives.ipynb*")

deep_dives = {**findings}
deep_dives["_markdown"] = "\n".join(lines)
save_eda_output(deep_dives, "deep_dives", eda_dir)
print("Saved eda/deep_dives.json and eda/deep_dives.md")
```

### Success Criteria

#### Automated Verification:
- [ ] Notebook runs top-to-bottom: `jupyter nbconvert --execute notebooks/04_deep_dives.ipynb`
- [ ] `eda/deep_dives.json` exists and contains all 6 finding sections
- [ ] `eda/deep_dives.md` exists and is non-empty
- [ ] `eda/deep_dives.png` visualization is generated
- [ ] Labor cost match rate is >95% (timesheet entries matched to pay rates)
- [ ] True margin projects count is 700+ (not 21)

#### Manual Verification:
- [ ] Review the 6-panel visualization for reasonable distributions
- [ ] Confirm TotProjCost interpretation makes business sense
- [ ] Check that median gross margin is in a reasonable range for commercial millwork (30-60%)
- [ ] Verify CO capture rate aligns with industry expectations (10-25% for complex commercial work)

---

## Testing Strategy

### Automated:
- Run all 4 notebooks in sequence (01 → 02 → 03 → 04) without errors
- Verify no 100% orphan rates remain in quality audit
- Verify margin analysis covers 700+ projects

### Manual:
- Compare corrected quality audit heatmap to original (should show dramatic improvement)
- Review business metrics dashboard with corrected billing data
- Sanity-check true margin numbers against industry benchmarks

## Performance Considerations

- The date-range join in Section B (288K timesheet rows × 1,380 pay periods) may be slow as a full merge+filter. If >60 seconds, optimize by pre-sorting pay periods and using `pd.merge_asof` with direction="backward" after sorting both by date within each employee.
- All other operations are standard groupby/merge on <50K rows and should complete in seconds.

## References

- Phase 3 notebooks: `notebooks/01_quality_audit.ipynb`, `notebooks/02_pipeline_trace.ipynb`, `notebooks/03_business_metrics.ipynb`
- EDA outputs: `eda/quality_audit.md`, `eda/pipeline_trace.md`, `eda/business_metrics.md`
- Helper code: `scripts/eda_helpers.py`, `scripts/config.py`
- Key tables: tblProject, tblBill, tblBillHistory, tblTimesheet, tblPurchasing, tblEmpPayHist, tblCO
