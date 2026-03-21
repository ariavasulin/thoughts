# Estimation Variance EDA Notebook — Implementation Plan

## Overview

Build `notebooks/04_estimation_variance.ipynb` — an EDA notebook that quantifies schedule estimation error across the 17-stage DWS production pipeline. This is **schedule variance** (planned days vs actual days), complementing the **cost variance** (bid $ vs actual $) already computed in notebook 03.

The notebook is designed as a Bayesian modeling on-ramp: every analysis produces tidy DataFrames with the right grouping structure for hierarchical models (PM-level, estimator-level random effects on bias).

## Current State Analysis

**What exists:**
- `Tables/tblSheetNEW.csv` — 15,932 sheets, 117 columns, with per-stage duration estimates and completion dates
- `Tables/tblDefaulDurations.csv` — 18 workflow templates (copied to sheets at creation, then manually adjusted)
- Notebooks 01–03 already executed (quality audit, pipeline trace, business metrics)
- `scripts/eda_helpers.py` with `load_table()`, `save_eda_output()`, `TIER_1_TABLES`
- `eda/` directory with existing JSON/markdown/PNG outputs

**Key data constraints** (from [research doc](../research/2026-03-21-estimation-variance-calibration-eda.md)):
- No per-stage actual completion timestamps — system records `Done` flags (boolean) but not *when* each stage finished
- Target dates (`{Name}Dt{N}`) only ~47% populated (scheduling system adopted mid-history)
- Duration columns ~99% populated, `SheetCompleteDate` 99.1%, `ToShopAct` 94.9%
- Negative duration values exist (schedule adjustments) — need handling, not filtering

### Key Discoveries:
- Duration estimates are **manually adjusted copies** of templates, not raw template values — actual DraftDur1 ranges [-104, 573] vs template range [2, 10] (`Tables/tblDefaulDurations.csv`)
- Target dates are **computed FROM durations** (confirmed by `qrySheetschedUpdate` in `metadata/access/queries.csv:2483`)
- The Access system already computes delivery variance (DeliveryStatusRed/Black) and completion variance (FinalStatusRed/Black) in `qrySheetList` (`metadata/access/queries.csv:2270`)
- Phase-level actuals are derivable: `EngDur = ToShopAct - SheetStart`, `TotDur = SheetCompleteDate - SheetStart`
- Segmentation dimensions from `tblProject`: PM (26), Estimator (23), Client (353)

## Desired End State

After this plan is complete:

```
eda/
  estimation_variance.json    # Per-sheet variance data, group summaries, calibration metrics
  estimation_variance.md      # Human-readable findings report

notebooks/
  04_estimation_variance.ipynb  # Full EDA notebook (runs end-to-end)
```

The JSON output is specifically structured for downstream Bayesian modeling:
- A tidy DataFrame (saved as records in JSON) with one row per completed sheet containing: `SheetID`, `ProjectID`, `planned_total`, `actual_total`, `variance_days`, `variance_pct`, `planned_eng`, `actual_eng`, `eng_variance`, `n_active_stages`, `sheet_type`, `pm`, `estimator`, `year`, `delivery_variance`
- Group-level summary statistics (by PM, by estimator, by year, by sheet type)
- Calibration metrics (slope/intercept of planned vs actual regression)

Verification: `eda/estimation_variance.json` exists, is valid JSON, contains `"modeling_df"` key with >10,000 records. Notebook runs without errors.

## What We're NOT Doing

- Fitting Bayesian models (that's the next notebook — this one produces the clean dataset and EDA)
- Per-stage variance analysis (no actual stage completion timestamps available — only phase-level)
- Template matching/recovery (no FK, would require fuzzy matching — separate effort)
- Working-day corrections (calendar vs business days — the estimates are in calendar days per `DateDiff("d",...)`)
- Modifying `eda_helpers.py` (current helpers are sufficient)

## Implementation Approach

Single notebook with 9 cells following the established pattern from notebooks 01–03:
1. Load data and define stage mappings
2. Construct the core variance dataset (planned vs actual for every completed sheet)
3. Overall variance distribution + summary stats
4. Phase-level variance (engineering vs shop)
5. Delivery variance
6. Segmentation analysis (PM, estimator, year, sheet type, complexity)
7. Calibration analysis (is planned a good predictor of actual?)
8. Visualizations
9. Assemble and save outputs (JSON + markdown)

---

## Phase 1: Build the Notebook

### Overview
Create `notebooks/04_estimation_variance.ipynb` with all 9 cells. Produces `eda/estimation_variance.json` and `eda/estimation_variance.md`.

### Changes Required

#### 1. Notebook: `notebooks/04_estimation_variance.ipynb`

**Cell 0 — Markdown header**:
```markdown
# Notebook 04: Estimation Variance Analysis
Quantifies schedule estimation error across the 17-stage production pipeline.
Compares planned durations vs actual elapsed time at overall, phase, and delivery levels.
Produces `eda/estimation_variance.json` and `eda/estimation_variance.md`.

**Key question**: Are DWS duration estimates systematically biased, and does bias vary by PM, estimator, sheet type, or project complexity?
```

**Cell 1 — Setup & Stage Definitions**:
```python
import sys
sys.path.insert(0, "../scripts")

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats
from eda_helpers import *
from config import ROOT_DIR

eda_dir = ROOT_DIR / "eda"

# Load primary tables
sheets = load_table("tblSheetNEW")
project = load_table("tblProject")
templates = load_table("tblDefaulDurations")

print(f"Sheets: {len(sheets):,}")
print(f"Projects: {len(project):,}")
print(f"Templates: {len(templates):,}")

# Stage definitions: (duration_col, skip_col, date_col_if_exists)
# Order matches the production pipeline
STAGES = [
    ("DraftDur1",     "DraftSkip1",    "DraftDt1"),
    ("ApprDur2",      "ApprSkip2",     "ApprDt2"),
    ("MeasureDur3",   "MeasureSkip3",  "MeasureDt3"),
    ("CorrectDur4",   "CorrectSkip4",  "CorrectDt4"),
    ("StockDur5",     "StockSkip5",    "StockDt5"),
    ("PgmDur6",       "PgmSkip6",     None),       # No PgmDt — uses ToShop
    ("VenDur7",       "VenSkip7",     "VenDt7"),
    ("MillDur8",      "MillSkip8",    "MillDt8"),
    ("SawDur9",       "SawSkip9",     "SawDt9"),
    ("CNCDur10",      "CNCSkip10",   "CNCDt10"),
    ("OtherDur11",    "OthSkip11",   "OthDt11"),
    ("BenchDur12",    "BenchSkip12", "BenchDt12"),
    ("FinishDur13",   "FinishSkip13","FinishDt13"),
    ("AssemblyDur14", "AssemblySkip14","AssemblyDt"),
    ("StorDur",       "StorSkip",    "StorDt"),
    ("FieldDur",      "FieldSkip",   "FieldDt"),
]

# Engineering stages (pre-shop): Draft through Program (indices 0-5)
ENG_STAGES = STAGES[:6]
# Shop stages: Veneer through Assembly (indices 6-13)
SHOP_STAGES = STAGES[6:14]
# Post-shop: Storage + Field (indices 14-15)
POST_STAGES = STAGES[14:]

DUR_COLS = [s[0] for s in STAGES]
SKIP_COLS = [s[1] for s in STAGES]
```

**Cell 2 — Construct Core Variance Dataset**:
```python
# Filter to completed sheets with dates
df = sheets.copy()

# Parse dates
for col in ["SheetStart", "SheetCompleteDate", "ToShopAct", "Delivery", "Delivered"]:
    df[col] = pd.to_datetime(df[col], errors="coerce")

# Convert duration and skip columns to numeric
for col in DUR_COLS:
    df[col] = pd.to_numeric(df[col], errors="coerce").fillna(0)
for col in SKIP_COLS:
    df[col] = pd.to_numeric(df[col], errors="coerce").fillna(0)

# Filter: completed sheets with start and end dates
completed = df[
    df["SheetCompleteDate"].notna() &
    df["SheetStart"].notna()
].copy()
print(f"Completed sheets with start+end dates: {len(completed):,}")

# --- Planned durations ---
# For each sheet, sum duration columns where skip != -1 (not skipped)
# Note: skip=-1 means "skip this stage", 0 means "don't skip"
for dur_col, skip_col, _ in STAGES:
    col_name = f"_adj_{dur_col}"
    completed[col_name] = np.where(
        completed[skip_col] == -1,  # skipped
        0,
        completed[dur_col]
    )

adj_dur_cols = [f"_adj_{s[0]}" for s in STAGES]
eng_adj_cols = [f"_adj_{s[0]}" for s in ENG_STAGES]
shop_adj_cols = [f"_adj_{s[0]}" for s in SHOP_STAGES]

completed["planned_total"] = completed[adj_dur_cols].sum(axis=1)
completed["planned_eng"] = completed[eng_adj_cols].sum(axis=1)
completed["planned_shop"] = completed[shop_adj_cols].sum(axis=1)

# --- Actual durations ---
completed["actual_total"] = (completed["SheetCompleteDate"] - completed["SheetStart"]).dt.days
completed["actual_eng"] = np.where(
    completed["ToShopAct"].notna(),
    (completed["ToShopAct"] - completed["SheetStart"]).dt.days,
    np.nan
)
completed["actual_shop"] = np.where(
    completed["ToShopAct"].notna() & completed["SheetCompleteDate"].notna(),
    (completed["SheetCompleteDate"] - completed["ToShopAct"]).dt.days,
    np.nan
)

# --- Variance ---
completed["variance_days"] = completed["actual_total"] - completed["planned_total"]
completed["variance_pct"] = np.where(
    completed["planned_total"] > 0,
    (completed["variance_days"] / completed["planned_total"] * 100),
    np.nan
)
completed["eng_variance"] = completed["actual_eng"] - completed["planned_eng"]
completed["shop_variance"] = completed["actual_shop"] - completed["planned_shop"]

# --- Complexity proxy ---
completed["n_active_stages"] = (completed[SKIP_COLS].ne(-1)).sum(axis=1)

# --- Year ---
completed["year"] = completed["SheetCompleteDate"].dt.year

# --- Delivery variance ---
completed["delivery_variance"] = np.where(
    completed["Delivered"].notna() & completed["Delivery"].notna(),
    (completed["Delivered"] - completed["Delivery"]).dt.days,
    np.nan
)

print(f"\nPlanned total (days): median={completed['planned_total'].median():.0f}, "
      f"mean={completed['planned_total'].mean():.1f}")
print(f"Actual total (days):  median={completed['actual_total'].median():.0f}, "
      f"mean={completed['actual_total'].mean():.1f}")
print(f"Variance (days):      median={completed['variance_days'].median():.0f}, "
      f"mean={completed['variance_days'].mean():.1f}")
print(f"Active stages:        median={completed['n_active_stages'].median():.0f}")
print(f"Sheets with eng phase data: {completed['actual_eng'].notna().sum():,}")
print(f"Sheets with delivery variance: {completed['delivery_variance'].notna().sum():,}")
```

**Cell 3 — Overall Variance Distribution**:
```python
# Overall summary statistics
var = completed["variance_days"].dropna()

print("=== Overall Schedule Variance (Actual - Planned, calendar days) ===")
print(f"  n = {len(var):,}")
print(f"  Mean:   {var.mean():.1f} days")
print(f"  Median: {var.median():.0f} days")
print(f"  Std:    {var.std():.1f} days")
print(f"  IQR:    [{var.quantile(0.25):.0f}, {var.quantile(0.75):.0f}]")
print(f"  Range:  [{var.min():.0f}, {var.max():.0f}]")
print(f"\n  On time or early (variance <= 0): {(var <= 0).sum()} ({(var <= 0).mean()*100:.1f}%)")
print(f"  Late by 1-10 days:  {((var > 0) & (var <= 10)).sum()} ({((var > 0) & (var <= 10)).mean()*100:.1f}%)")
print(f"  Late by 11-30 days: {((var > 10) & (var <= 30)).sum()} ({((var > 10) & (var <= 30)).mean()*100:.1f}%)")
print(f"  Late by 30+ days:   {(var > 30).sum()} ({(var > 30).mean()*100:.1f}%)")

# Skewness and kurtosis — important for choosing priors
print(f"\n  Skewness: {var.skew():.2f}")
print(f"  Kurtosis: {var.kurtosis():.2f}")
print(f"  (Heavy tails suggest t-distribution likelihood, not Gaussian)")
```

**Cell 4 — Phase-Level Variance (Engineering vs Shop)**:
```python
has_phases = completed[completed["actual_eng"].notna()].copy()
print(f"=== Phase-Level Variance (n={len(has_phases):,} sheets with ToShopAct) ===\n")

for phase, planned_col, actual_col, var_col in [
    ("Engineering", "planned_eng", "actual_eng", "eng_variance"),
    ("Shop", "planned_shop", "actual_shop", "shop_variance"),
]:
    v = has_phases[var_col].dropna()
    p = has_phases[planned_col]
    a = has_phases[actual_col].dropna()
    print(f"  {phase} phase:")
    print(f"    Planned: median={p.median():.0f}d, mean={p.mean():.1f}d")
    print(f"    Actual:  median={a.median():.0f}d, mean={a.mean():.1f}d")
    print(f"    Variance: median={v.median():.0f}d, mean={v.mean():.1f}d, std={v.std():.1f}d")
    print(f"    On time or early: {(v <= 0).mean()*100:.1f}%")
    print()

# Correlation: does eng variance predict shop variance?
both = has_phases[["eng_variance", "shop_variance"]].dropna()
if len(both) > 10:
    r, p_val = stats.pearsonr(both["eng_variance"], both["shop_variance"])
    print(f"  Eng-Shop variance correlation: r={r:.3f}, p={p_val:.2e}")
    print(f"  (Positive r means eng delays cascade into shop delays)")
```

**Cell 5 — Delivery Variance**:
```python
has_delivery = completed[completed["delivery_variance"].notna()].copy()
dv = has_delivery["delivery_variance"]

print(f"=== Delivery Variance (Delivered - Delivery target, n={len(has_delivery):,}) ===")
print(f"  Mean:   {dv.mean():.1f} days")
print(f"  Median: {dv.median():.0f} days")
print(f"  Std:    {dv.std():.1f} days")
print(f"  On time or early: {(dv <= 0).sum()} ({(dv <= 0).mean()*100:.1f}%)")
print(f"  Late (>0 days):   {(dv > 0).sum()} ({(dv > 0).mean()*100:.1f}%)")
print(f"  Late by 7+ days:  {(dv > 7).sum()} ({(dv > 7).mean()*100:.1f}%)")
```

**Cell 6 — Segmentation Analysis**:
```python
# Join project-level attributes
project["ProjectID"] = pd.to_numeric(project["ProjectID"], errors="coerce")
completed["ProjectID"] = pd.to_numeric(completed["ProjectID"], errors="coerce")

seg = completed.merge(
    project[["ProjectID", "PM", "Estimator", "Client", "TotProjCost"]].rename(
        columns={"TotProjCost": "proj_cost"}
    ),
    on="ProjectID", how="left"
)
seg["proj_cost"] = pd.to_numeric(seg["proj_cost"], errors="coerce")

def group_summary(data, group_col, min_n=20):
    """Compute variance summary by group, filtering to groups with min_n sheets."""
    g = data.groupby(group_col)["variance_days"].agg(["count", "mean", "median", "std"])
    g = g[g["count"] >= min_n].sort_values("mean")
    g.columns = ["n", "mean_var", "median_var", "std_var"]
    return g.round(1)

# --- By PM ---
pm_stats = group_summary(seg, "PM")
print(f"=== Variance by PM (n >= 20 sheets) ===")
print(f"PMs with enough data: {len(pm_stats)}")
print(pm_stats.to_string())
print(f"\nRange of PM mean variance: [{pm_stats['mean_var'].min():.1f}, {pm_stats['mean_var'].max():.1f}] days")

# --- By Estimator ---
est_stats = group_summary(seg, "Estimator")
print(f"\n=== Variance by Estimator (n >= 20 sheets) ===")
print(f"Estimators with enough data: {len(est_stats)}")
print(est_stats.to_string())

# --- By Year ---
year_stats = group_summary(seg, "year")
print(f"\n=== Variance by Year ===")
print(year_stats.to_string())

# --- By SheetType ---
type_stats = group_summary(seg, "SheetType", min_n=10)
print(f"\n=== Variance by SheetType ===")
print(type_stats.to_string())

# --- By Complexity (n_active_stages) ---
complexity_stats = group_summary(seg, "n_active_stages", min_n=30)
print(f"\n=== Variance by Complexity (active stage count) ===")
print(complexity_stats.to_string())
```

**Cell 7 — Calibration Analysis**:
```python
# Is planned_total a good predictor of actual_total?
# Perfect calibration: slope=1, intercept=0

cal = completed[
    (completed["planned_total"] > 0) &
    (completed["actual_total"] > 0) &
    (completed["actual_total"] < completed["actual_total"].quantile(0.99))  # trim extreme outliers
].copy()

slope, intercept, r_value, p_value, std_err = stats.linregress(
    cal["planned_total"], cal["actual_total"]
)

print(f"=== Calibration: Planned vs Actual Total Duration ===")
print(f"  n = {len(cal):,}")
print(f"  Regression: actual = {slope:.3f} * planned + {intercept:.1f}")
print(f"  R² = {r_value**2:.3f}")
print(f"  Slope SE = {std_err:.4f}")
print(f"\n  Interpretation:")
if slope > 1.05:
    print(f"  Slope > 1: estimates are systematically OPTIMISTIC (actual grows faster than planned)")
elif slope < 0.95:
    print(f"  Slope < 1: estimates are systematically PESSIMISTIC (actual grows slower than planned)")
else:
    print(f"  Slope ≈ 1: estimates are reasonably calibrated on average")
if intercept > 5:
    print(f"  Intercept = {intercept:.1f}: fixed overhead of ~{intercept:.0f} days not captured in estimates")

# Binned calibration (for calibration plot)
cal["planned_bin"] = pd.cut(cal["planned_total"], bins=10)
cal_binned = cal.groupby("planned_bin", observed=True)["actual_total"].agg(["mean", "median", "count"])
print(f"\n  Binned calibration (planned range -> actual mean):")
print(cal_binned.to_string())
```

**Cell 8 — Visualizations**:
```python
fig, axes = plt.subplots(2, 3, figsize=(18, 11))

# 1. Overall variance histogram
ax = axes[0, 0]
var_clipped = completed["variance_days"].clip(-100, 200)
ax.hist(var_clipped, bins=50, color="steelblue", edgecolor="white", alpha=0.8)
ax.axvline(x=0, color="red", linestyle="--", linewidth=1.5, label="On time")
ax.axvline(x=completed["variance_days"].median(), color="orange", linestyle="-",
           linewidth=1.5, label=f"Median: {completed['variance_days'].median():.0f}d")
ax.set_xlabel("Variance (days)")
ax.set_ylabel("Count")
ax.set_title("Overall Schedule Variance Distribution")
ax.legend(fontsize=8)

# 2. Planned vs Actual scatter with calibration line
ax = axes[0, 1]
sample = cal.sample(min(2000, len(cal)), random_state=42)
ax.scatter(sample["planned_total"], sample["actual_total"], alpha=0.15, s=8, color="steelblue")
x_range = np.array([0, cal["planned_total"].quantile(0.99)])
ax.plot(x_range, x_range, "r--", linewidth=1, label="Perfect calibration")
ax.plot(x_range, slope * x_range + intercept, "orange", linewidth=1.5,
        label=f"Fit: {slope:.2f}x + {intercept:.0f}")
ax.set_xlabel("Planned Total (days)")
ax.set_ylabel("Actual Total (days)")
ax.set_title("Calibration: Planned vs Actual")
ax.legend(fontsize=8)
ax.set_aspect("equal")

# 3. Variance by year (trend)
ax = axes[0, 2]
year_data = year_stats.reset_index()
ax.bar(year_data["year"], year_data["mean_var"], color="steelblue", alpha=0.7)
ax.axhline(y=0, color="red", linestyle="--", linewidth=1)
ax.set_xlabel("Year")
ax.set_ylabel("Mean Variance (days)")
ax.set_title("Estimation Bias Over Time")

# 4. Variance by PM (boxplot, top 10 by volume)
ax = axes[1, 0]
top_pms = seg.groupby("PM")["variance_days"].count().nlargest(10).index
pm_data = seg[seg["PM"].isin(top_pms)]
pm_order = pm_data.groupby("PM")["variance_days"].median().sort_values().index
pm_data["PM_str"] = pm_data["PM"].astype(str)
# Use pandas boxplot grouped by PM
pm_pivot = [group["variance_days"].clip(-100, 200).values for _, group in pm_data.groupby("PM")]
pm_labels = [str(pm) for pm in pm_data.groupby("PM").groups.keys()]
ax.boxplot(pm_pivot, labels=pm_labels, vert=True)
ax.axhline(y=0, color="red", linestyle="--", linewidth=1)
ax.set_xlabel("PM ID")
ax.set_ylabel("Variance (days)")
ax.set_title("Variance by PM (Top 10 by Volume)")
ax.tick_params(axis="x", rotation=45)

# 5. Eng vs Shop variance scatter
ax = axes[1, 1]
if len(both) > 0:
    ax.scatter(both["eng_variance"].clip(-100, 200),
               both["shop_variance"].clip(-100, 200),
               alpha=0.1, s=8, color="steelblue")
    ax.axhline(y=0, color="gray", linestyle="--", alpha=0.5)
    ax.axvline(x=0, color="gray", linestyle="--", alpha=0.5)
    ax.set_xlabel("Engineering Variance (days)")
    ax.set_ylabel("Shop Variance (days)")
    ax.set_title(f"Eng vs Shop Variance (r={r:.2f})")

# 6. Variance by complexity
ax = axes[1, 2]
complexity_data = complexity_stats.reset_index()
ax.bar(complexity_data["n_active_stages"], complexity_data["mean_var"], color="steelblue", alpha=0.7)
ax.axhline(y=0, color="red", linestyle="--", linewidth=1)
ax.set_xlabel("Number of Active Stages")
ax.set_ylabel("Mean Variance (days)")
ax.set_title("Variance by Complexity")

plt.tight_layout()
plt.savefig(str(eda_dir / "estimation_variance.png"), dpi=150)
plt.show()
```

**Cell 9 — Assemble and Save Output**:
```python
# Build the modeling-ready DataFrame
modeling_cols = [
    "SheetID", "ProjectID", "SheetType", "year", "n_active_stages",
    "planned_total", "actual_total", "variance_days", "variance_pct",
    "planned_eng", "actual_eng", "eng_variance",
    "planned_shop", "actual_shop", "shop_variance",
    "delivery_variance",
]
modeling_df = seg[modeling_cols + ["PM", "Estimator"]].copy()
modeling_records = modeling_df.to_dict("records")

# Build output
output = {
    "summary": {
        "total_sheets": len(sheets),
        "completed_sheets": len(completed),
        "mean_variance_days": round(float(completed["variance_days"].mean()), 1),
        "median_variance_days": round(float(completed["variance_days"].median()), 1),
        "std_variance_days": round(float(completed["variance_days"].std()), 1),
        "pct_on_time_or_early": round(float((completed["variance_days"] <= 0).mean() * 100), 1),
        "pct_late_30_plus": round(float((completed["variance_days"] > 30).mean() * 100), 1),
        "skewness": round(float(completed["variance_days"].skew()), 2),
        "kurtosis": round(float(completed["variance_days"].kurtosis()), 2),
    },
    "calibration": {
        "slope": round(slope, 3),
        "intercept": round(intercept, 1),
        "r_squared": round(r_value**2, 3),
        "interpretation": "optimistic" if slope > 1.05 else "pessimistic" if slope < 0.95 else "calibrated",
    },
    "by_pm": pm_stats.reset_index().to_dict("records") if len(pm_stats) > 0 else [],
    "by_estimator": est_stats.reset_index().to_dict("records") if len(est_stats) > 0 else [],
    "by_year": year_stats.reset_index().to_dict("records"),
    "by_sheet_type": type_stats.reset_index().to_dict("records"),
    "by_complexity": complexity_stats.reset_index().to_dict("records"),
    "modeling_df": modeling_records,
}

# Markdown report
lines = ["# Estimation Variance Report\n"]
s = output["summary"]
lines.append(f"**Completed sheets analyzed**: {s['completed_sheets']:,}")
lines.append(f"**Mean variance**: {s['mean_variance_days']} days")
lines.append(f"**Median variance**: {s['median_variance_days']} days")
lines.append(f"**On time or early**: {s['pct_on_time_or_early']}%")
lines.append(f"**Late by 30+ days**: {s['pct_late_30_plus']}%\n")

c = output["calibration"]
lines.append("## Calibration\n")
lines.append(f"- Regression: actual = {c['slope']} * planned + {c['intercept']}")
lines.append(f"- R² = {c['r_squared']}")
lines.append(f"- Interpretation: estimates are **{c['interpretation']}**\n")

lines.append("## Variance by Year\n")
lines.append("| Year | n | Mean Var | Median Var | Std |")
lines.append("|------|---|----------|------------|-----|")
for row in output["by_year"]:
    lines.append(f"| {row['year']} | {row['n']:.0f} | {row['mean_var']:.1f} | {row['median_var']:.1f} | {row['std_var']:.1f} |")

lines.append("\n## Variance by PM\n")
lines.append("| PM | n | Mean Var | Median Var | Std |")
lines.append("|----|---|----------|------------|-----|")
for row in output["by_pm"]:
    lines.append(f"| {row['PM']} | {row['n']:.0f} | {row['mean_var']:.1f} | {row['median_var']:.1f} | {row['std_var']:.1f} |")

lines.append("\n## Variance by Estimator\n")
lines.append("| Estimator | n | Mean Var | Median Var | Std |")
lines.append("|-----------|---|----------|------------|-----|")
for row in output["by_estimator"]:
    lines.append(f"| {row['Estimator']} | {row['n']:.0f} | {row['mean_var']:.1f} | {row['median_var']:.1f} | {row['std_var']:.1f} |")

lines.append("\n## Next Steps: Bayesian Modeling\n")
lines.append("The `modeling_df` in `estimation_variance.json` is structured for hierarchical modeling:")
lines.append("- **Outcome**: `variance_days` or `log(actual_total / planned_total)`")
lines.append("- **Group-level effects**: PM (random intercept), Estimator (random intercept)")
lines.append("- **Sheet-level covariates**: `n_active_stages`, `SheetType`, `year`, `planned_total`")
lines.append("- **Likelihood**: Student-t (heavy tails observed) or log-normal")
lines.append("- **Framework**: PyMC or numpyro")

lines.append("\n---\n*Generated by notebooks/04_estimation_variance.ipynb*")

output["_markdown"] = "\n".join(lines)
save_eda_output(output, "estimation_variance", eda_dir)
print(f"Saved eda/estimation_variance.json ({len(modeling_records):,} sheet records)")
print(f"Saved eda/estimation_variance.md")
```

### Success Criteria

#### Automated Verification:
- [ ] Notebook runs end-to-end: `cd /Users/ariasulin/Git/dwsTables && source .venv/bin/activate && jupyter nbconvert --execute notebooks/04_estimation_variance.ipynb --to notebook --output-dir=notebooks/`
- [ ] `eda/estimation_variance.json` exists and is valid JSON: `python -c "import json; d=json.load(open('eda/estimation_variance.json')); print(f'Records: {len(d[\"modeling_df\"]):,}')"`
- [ ] `eda/estimation_variance.md` exists and is >500 chars
- [ ] `eda/estimation_variance.png` exists and is >10KB
- [ ] `scipy` is available (needed for `stats.linregress`): `python -c "from scipy import stats; print('ok')"`

#### Manual Verification:
- [ ] Variance distribution is plausible (centered near 0, heavy right tail for late deliveries)
- [ ] Calibration slope is between 0.5 and 2.0 (sanity check)
- [ ] PM/Estimator variance ranges make domain sense (individual biases of -30 to +60 days typical for millwork)
- [ ] Year trend is sensible (no sudden jumps unless there's a known cause like COVID 2020)
- [ ] `modeling_df` contains >10,000 records with no all-null columns

---

## Dependency: scipy

The notebook uses `scipy.stats.linregress` and `scipy.stats.pearsonr`. Check if scipy is installed:

```bash
source /Users/ariasulin/Git/dwsTables/.venv/bin/activate && python -c "import scipy; print(scipy.__version__)"
```

If not installed: `pip install scipy` and add to `requirements.txt`.

---

## Bayesian Modeling On-Ramp

The `modeling_df` output from this notebook is specifically designed to feed into a follow-up notebook (`05_bayesian_estimation_model.ipynb`) with this structure:

```python
# Sketch of the hierarchical model (notebook 05, future work)
import pymc as pm

with pm.Model() as estimation_model:
    # Hyperpriors
    mu_global = pm.Normal("mu_global", mu=0, sigma=30)
    sigma_global = pm.HalfNormal("sigma_global", sigma=20)

    # PM-level random intercepts
    n_pms = len(pm_ids)
    sigma_pm = pm.HalfNormal("sigma_pm", sigma=15)
    pm_offset = pm.Normal("pm_offset", mu=0, sigma=1, shape=n_pms)
    pm_effect = pm.Deterministic("pm_effect", sigma_pm * pm_offset)

    # Estimator-level random intercepts
    n_est = len(est_ids)
    sigma_est = pm.HalfNormal("sigma_est", sigma=15)
    est_offset = pm.Normal("est_offset", mu=0, sigma=1, shape=n_est)
    est_effect = pm.Deterministic("est_effect", sigma_est * est_offset)

    # Sheet-level covariates
    beta_complexity = pm.Normal("beta_complexity", mu=0, sigma=5)
    beta_planned = pm.Normal("beta_planned", mu=0, sigma=1)

    # Mean model
    mu = (mu_global
          + pm_effect[pm_idx]
          + est_effect[est_idx]
          + beta_complexity * complexity
          + beta_planned * planned_total)

    # Student-t likelihood (heavy tails)
    nu = pm.Gamma("nu", alpha=2, beta=0.1)
    sigma_obs = pm.HalfNormal("sigma_obs", sigma=30)
    y = pm.StudentT("y", nu=nu, mu=mu, sigma=sigma_obs, observed=variance_days)
```

Key modeling decisions that this EDA informs:
1. **Likelihood family**: Check skewness/kurtosis in Cell 3 — heavy tails → Student-t
2. **Group sizes**: PM and Estimator `n` values determine whether partial pooling is useful
3. **Covariate scaling**: Use planned_total and n_active_stages stats for prior calibration
4. **Interaction terms**: Eng-Shop correlation (Cell 4) determines whether phase-level modeling adds value

## References

- Research document: `thoughts/shared/research/2026-03-21-estimation-variance-calibration-eda.md`
- Production workflow: `thoughts/shared/research/2026-03-21-shop-schedule-production-workflow.md`
- Existing notebook pattern: `notebooks/03_business_metrics.ipynb`
- Stage definitions: `Tables/tblDefaultSheetProc.csv`
- Data: `Tables/tblSheetNEW.csv` (15,932 rows, 117 columns)
- Templates: `Tables/tblDefaulDurations.csv` (18 rows)
