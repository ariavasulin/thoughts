# Duration Regression & Model Evaluation — Implementation Plan

## Overview

Build `notebooks/06_duration_regression.ipynb` to fit and rigorously evaluate a regression model for predicting actual production duration from planned duration and project attributes. Uses the Log-normal error structure established in notebook 05, evaluates with temporal validation, calibration diagnostics, prediction intervals, and segment-level bias checks.

## Current State Analysis

From notebooks 04-05:
- `modeling_df` in `eda/estimation_variance.json`: 14,042 records with `actual_total`, `planned_total`, `PM`, `Estimator`, `SheetType`, `n_active_stages`, `year`
- GoF result: Log-normal on `actual/planned` ratio is the best error structure
- Calibration from OLS: `actual = 1.164 * planned + 15.2`
- 76.2% of sheets are late; median ratio 1.42
- PM matters (79% segment agreement), SheetType stable (100%)
- `Estimator` has NaN values in early records

## Desired End State

A notebook that:
1. Fits a log-linear regression: `log(actual/planned) = β₀ + β₁·complexity + β₂·SheetType + u_PM + ε`
2. Compares against a dumb baseline (median ratio lookup by PM)
3. Evaluates with temporal train/test split (train < 2020, test ≥ 2020)
4. Reports calibration, coverage of prediction intervals, MAE, and segment-level bias
5. Outputs recommended model parameters for operational use

### Verification:
- Notebook runs end-to-end via `jupyter nbconvert --execute`
- `eda/duration_regression.json` exists with model coefficients, metrics, and evaluation results
- `eda/duration_regression.md` exists with >500 chars summarizing findings
- `eda/duration_regression.png` exists with >10KB (multi-panel evaluation figure)

## What We're NOT Doing

- Full Bayesian hierarchical model (that's the next notebook after this validates the structure)
- Feature engineering beyond what's in modeling_df (no new joins or derived features)
- Hyperparameter tuning (this is about evaluating the model structure, not squeezing performance)

---

## Phase 1: Data Prep and Baseline

### Overview
Load data, filter to usable records, create temporal split, establish the "dumb" baseline.

### Cell 1 — Header (markdown)
```markdown
# Notebook 06: Duration Regression & Model Evaluation
Log-linear regression on `actual/planned` ratio with temporal validation.
Uses the Log-normal error structure from notebook 05.
Produces `eda/duration_regression.json`, `eda/duration_regression.md`, and `eda/duration_regression.png`.
```

### Cell 2 — Imports and data loading
```python
import sys
sys.path.insert(0, "../scripts")

import json
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats
import statsmodels.api as sm
import statsmodels.formula.api as smf
from eda_helpers import save_eda_output
from config import ROOT_DIR

eda_dir = ROOT_DIR / "eda"
np.random.seed(42)

# Load modeling_df from estimation variance output
with open(eda_dir / "estimation_variance.json") as f:
    ev_data = json.load(f)
df = pd.DataFrame(ev_data["modeling_df"])
print(f"Raw records: {len(df):,}")

# Convert numeric columns
for col in ["actual_total", "planned_total", "variance_days", "n_active_stages", "year"]:
    df[col] = pd.to_numeric(df[col], errors="coerce")
df["PM"] = df["PM"].astype(str)
df["Estimator"] = df["Estimator"].astype(str)

# Filter to usable: both > 0
df = df[(df["actual_total"] > 0) & (df["planned_total"] > 0)].copy()
df["ratio"] = df["actual_total"] / df["planned_total"]
df["log_ratio"] = np.log(df["ratio"])
print(f"Usable records (both > 0): {len(df):,}")

# --- Temporal split ---
# Train: year < 2020, Test: year >= 2020
# This tests whether the model generalizes to recent production
train = df[df["year"] < 2020].copy()
test = df[df["year"] >= 2020].copy()
print(f"\nTemporal split:")
print(f"  Train (< 2020): {len(train):,} records, years {train['year'].min()}-{train['year'].max()}")
print(f"  Test (>= 2020): {len(test):,} records, years {test['year'].min()}-{test['year'].max()}")
```

### Cell 3 — Dumb baseline: median ratio lookup
```python
# Baseline 1: Global median ratio
global_median = train["ratio"].median()
print(f"=== Baselines ===\n")
print(f"Global median ratio: {global_median:.3f}")
print(f"  → For a 50-day planned job, predict {50 * global_median:.0f} days actual")

# Baseline 2: PM-specific median ratio (lookup table)
pm_medians = train.groupby("PM")["ratio"].median()
pm_counts = train.groupby("PM")["ratio"].count()
# For PMs with < 10 sheets, fall back to global median
pm_lookup = pm_medians.where(pm_counts >= 10, global_median)

print(f"\nPM-specific median ratios (n >= 10 sheets): {(pm_counts >= 10).sum()} PMs")
print(f"  Range: [{pm_lookup.min():.3f}, {pm_lookup.max():.3f}]")

# Evaluate baselines on test set
def evaluate_predictions(actual, predicted, name="Model"):
    """Compute regression evaluation metrics."""
    residuals = actual - predicted
    mae = np.median(np.abs(residuals))  # Median Absolute Error (robust)
    rmse = np.sqrt(np.mean(residuals**2))
    mape = np.median(np.abs(residuals / actual)) * 100
    bias = np.mean(residuals)  # positive = underpredicting
    within_20pct = np.mean(np.abs(residuals / actual) < 0.20) * 100

    print(f"\n  {name}:")
    print(f"    Median Abs Error: {mae:.1f} days")
    print(f"    RMSE:             {rmse:.1f} days")
    print(f"    Median APE:       {mape:.1f}%")
    print(f"    Mean bias:        {bias:+.1f} days ({'underpredicting' if bias > 0 else 'overpredicting'})")
    print(f"    Within 20%:       {within_20pct:.1f}%")

    return {"mae": mae, "rmse": rmse, "mape": mape, "bias": bias, "within_20pct": within_20pct}

print("\n=== Baseline Evaluation on Test Set ===")

# Baseline 1: global median
pred_global = test["planned_total"] * global_median
baseline1_metrics = evaluate_predictions(test["actual_total"].values, pred_global.values, "Global Median Ratio")

# Baseline 2: PM-specific median
pred_pm = test.apply(lambda r: r["planned_total"] * pm_lookup.get(r["PM"], global_median), axis=1)
baseline2_metrics = evaluate_predictions(test["actual_total"].values, pred_pm.values, "PM-Specific Median")
```

### Success Criteria:

#### Automated Verification:
- [ ] Data loads and filters correctly
- [ ] Temporal split produces non-empty train and test sets
- [ ] Baseline metrics are computed without errors

---

## Phase 2: Log-Linear Regression Models

### Overview
Fit three models of increasing complexity: (1) intercept-only on log-ratio, (2) with SheetType + complexity, (3) with PM as random intercept (mixed model). Compare all against baselines.

### Cell 4 — OLS and mixed model fits
```python
print("=== Model Fitting (on training data) ===\n")

# Model 1: Intercept-only on log(ratio)
# This is equivalent to fitting a single Log-normal
model1 = sm.OLS(train["log_ratio"], sm.add_constant(np.ones(len(train)))).fit()
print(f"Model 1 (intercept only):")
print(f"  E[log(ratio)] = {model1.params.iloc[0]:.4f}")
print(f"  σ = {np.sqrt(model1.mse_resid):.4f}")
print(f"  → Predicted median ratio: {np.exp(model1.params.iloc[0]):.3f}")

# Model 2: OLS with SheetType + complexity
# Create design matrix
train_m2 = train.copy()
train_m2["intercept"] = 1

model2 = smf.ols("log_ratio ~ C(SheetType) + n_active_stages", data=train_m2).fit(cov_type="HC1")
print(f"\nModel 2 (SheetType + complexity):")
print(f"  R² = {model2.rsquared:.4f}")
print(f"  Adj R² = {model2.rsquared_adj:.4f}")
print(f"  Residual σ = {np.sqrt(model2.mse_resid):.4f}")
print(f"\n  Coefficients:")
for name, coef, se in zip(model2.params.index, model2.params, model2.bse):
    print(f"    {name:30s}: {coef:+.4f} (SE={se:.4f})")

# Model 3: Mixed model with PM as random intercept
model3 = smf.mixedlm(
    "log_ratio ~ C(SheetType) + n_active_stages",
    data=train_m2,
    groups=train_m2["PM"]
).fit()
print(f"\nModel 3 (Mixed model: PM random intercept):")
print(f"  Log-likelihood = {model3.llf:.1f}")
print(f"  PM random effect std = {float(np.sqrt(model3.cov_re.iloc[0, 0])):.4f}")
print(f"  Residual σ = {float(np.sqrt(model3.scale)):.4f}")
print(f"\n  Fixed effects:")
for name, coef in model3.fe_params.items():
    print(f"    {name:30s}: {coef:+.4f}")
```

### Cell 5 — Model comparison on test set
```python
print("=== Model Evaluation on Test Set ===\n")

# Helper: predict from each model, convert back to days
def predict_days(model, test_df, model_type="ols"):
    """Predict actual days from log-ratio model."""
    if model_type == "ols_intercept":
        log_ratio_pred = model.params.iloc[0]
        pred_ratio = np.exp(log_ratio_pred)
        return test_df["planned_total"] * pred_ratio
    elif model_type == "ols":
        log_ratio_pred = model.predict(test_df)
        # Smearing estimate: E[Y] = exp(Xβ) * E[exp(ε)]
        # For log-normal: E[exp(ε)] = exp(σ²/2)
        sigma2 = model.mse_resid
        pred_ratio = np.exp(log_ratio_pred) * np.exp(sigma2 / 2)
        return test_df["planned_total"] * pred_ratio
    elif model_type == "mixed":
        log_ratio_pred = model.predict(test_df)
        sigma2 = float(model.scale)
        pred_ratio = np.exp(log_ratio_pred) * np.exp(sigma2 / 2)
        return test_df["planned_total"] * pred_ratio

# Predict
pred_m1 = predict_days(model1, test, "ols_intercept")
pred_m2 = predict_days(model2, test, "ols")
pred_m3 = predict_days(model3, test, "mixed")

# Evaluate all models
all_metrics = {}
actual = test["actual_total"].values

all_metrics["Global Median"] = evaluate_predictions(actual, pred_global.values, "Global Median Ratio")
all_metrics["PM Median Lookup"] = evaluate_predictions(actual, pred_pm.values, "PM-Specific Median")
all_metrics["M1: Intercept Only"] = evaluate_predictions(actual, pred_m1.values, "M1: Intercept Only")
all_metrics["M2: SheetType+Complexity"] = evaluate_predictions(actual, pred_m2.values, "M2: SheetType + Complexity")
all_metrics["M3: Mixed (PM RE)"] = evaluate_predictions(actual, pred_m3.values, "M3: Mixed (PM Random Effect)")

# Summary table
print("\n=== Summary Comparison ===\n")
metrics_df = pd.DataFrame(all_metrics).T
metrics_df = metrics_df.round(1)
print(metrics_df.to_string())

# Which model wins?
best_model = metrics_df["mae"].idxmin()
print(f"\nBest model (lowest MAE): {best_model}")
```

### Success Criteria:

#### Automated Verification:
- [ ] All 3 models fit without errors
- [ ] Mixed model converges
- [ ] Test set predictions are computed for all models
- [ ] Summary comparison table is produced

---

## Phase 3: Calibration and Prediction Intervals

### Overview
Check if model predictions are well-calibrated (predicted vs actual) and whether prediction intervals achieve nominal coverage. This is the most important evaluation for operational use — a scheduler needs to trust the intervals.

### Cell 6 — Calibration + coverage analysis
```python
print("=== Calibration & Prediction Interval Analysis ===\n")

# Use Model 2 (or best model) for detailed analysis
# (Mixed model predictions on test require known PM groups — Model 2 is more portable)
best_ols = model2
log_ratio_pred = best_ols.predict(test)
sigma = np.sqrt(best_ols.mse_resid)

# --- Calibration: binned predicted vs actual ratio ---
test_eval = test.copy()
test_eval["pred_log_ratio"] = log_ratio_pred
test_eval["pred_ratio"] = np.exp(log_ratio_pred)
test_eval["pred_days"] = test_eval["planned_total"] * test_eval["pred_ratio"]

# Bin by predicted ratio deciles
test_eval["pred_bin"] = pd.qcut(test_eval["pred_ratio"], q=10, duplicates="drop")
cal_table = test_eval.groupby("pred_bin", observed=True).agg(
    n=("ratio", "count"),
    mean_pred_ratio=("pred_ratio", "mean"),
    mean_actual_ratio=("ratio", "mean"),
    median_pred_ratio=("pred_ratio", "median"),
    median_actual_ratio=("ratio", "median"),
).reset_index()

print("Calibration by predicted ratio decile:")
print(cal_table[["pred_bin", "n", "mean_pred_ratio", "mean_actual_ratio"]].to_string(index=False))

# Calibration slope: regress actual on predicted
cal_slope, cal_intercept, cal_r, _, _ = stats.linregress(
    test_eval["pred_ratio"], test_eval["ratio"]
)
print(f"\nCalibration regression: actual_ratio = {cal_slope:.3f} * pred_ratio + {cal_intercept:.3f}")
print(f"  Perfect calibration: slope=1, intercept=0")
print(f"  R² = {cal_r**2:.4f}")

# --- Prediction intervals ---
# For log-normal: CI on log scale → exponentiate
# P(actual_ratio in [exp(Xβ - z*σ), exp(Xβ + z*σ)]) = 1-α
print(f"\n=== Prediction Interval Coverage ===")
for level, z in [(0.50, 0.674), (0.80, 1.282), (0.90, 1.645), (0.95, 1.960)]:
    lo = np.exp(log_ratio_pred - z * sigma)
    hi = np.exp(log_ratio_pred + z * sigma)
    coverage = np.mean((test_eval["ratio"] >= lo) & (test_eval["ratio"] <= hi)) * 100
    median_width = np.median(hi - lo)
    print(f"  {level*100:.0f}% PI: coverage = {coverage:.1f}% (target: {level*100:.0f}%), "
          f"median width = {median_width:.2f} ratio units")
```

### Success Criteria:

#### Automated Verification:
- [ ] Calibration table is produced with 10 bins
- [ ] Calibration slope is computed
- [ ] Prediction interval coverage is computed for 4 levels
- [ ] Coverage values are within reasonable range (not all 0% or 100%)

---

## Phase 4: Segment-Level Bias and Temporal Stability

### Overview
Check for systematic bias by PM, SheetType, year, and complexity. A model that's great on average but terrible for specific PMs is useless operationally.

### Cell 7 — Segment-level bias analysis
```python
print("=== Segment-Level Bias Analysis (Test Set) ===\n")

test_eval["residual_days"] = test_eval["actual_total"] - test_eval["pred_days"]
test_eval["residual_ratio"] = test_eval["ratio"] - test_eval["pred_ratio"]

def segment_bias(data, group_col, min_n=20):
    """Compute bias metrics by segment."""
    results = []
    for group, gdf in data.groupby(group_col):
        if len(gdf) < min_n:
            continue
        resid = gdf["residual_days"]
        results.append({
            "group": str(group),
            "n": len(gdf),
            "mean_bias_days": round(resid.mean(), 1),
            "median_bias_days": round(resid.median(), 1),
            "mae_days": round(resid.abs().median(), 1),
            "pct_late": round((gdf["ratio"] > gdf["pred_ratio"]).mean() * 100, 1),
        })
    return pd.DataFrame(results).sort_values("mean_bias_days")

# By PM
pm_bias = segment_bias(test_eval, "PM", min_n=20)
print("--- By PM (n >= 20 in test set) ---")
print(pm_bias.to_string(index=False))
print(f"\nPM bias range: [{pm_bias['mean_bias_days'].min():.1f}, {pm_bias['mean_bias_days'].max():.1f}] days")

# By SheetType
type_bias = segment_bias(test_eval, "SheetType", min_n=10)
print(f"\n--- By SheetType (n >= 10 in test set) ---")
print(type_bias.to_string(index=False))

# By year (temporal stability)
year_bias = segment_bias(test_eval, "year", min_n=10)
print(f"\n--- By Year (test period) ---")
print(year_bias.to_string(index=False))

# By complexity
test_eval["complexity_bin"] = pd.cut(test_eval["n_active_stages"], bins=[0, 8, 12, 16, 20], labels=["low(1-8)", "med(9-12)", "high(13-16)", "max(17+)"])
complexity_bias = segment_bias(test_eval, "complexity_bin", min_n=10)
print(f"\n--- By Complexity ---")
print(complexity_bias.to_string(index=False))
```

### Success Criteria:

#### Automated Verification:
- [ ] Bias analysis runs for all 4 segments
- [ ] At least 2 PMs have enough data in test set for evaluation
- [ ] Year-level bias shows whether model degrades over time

---

## Phase 5: Diagnostic Plots

### Overview
6-panel figure covering calibration, residual diagnostics, prediction intervals, and segment bias.

### Cell 8 — Diagnostic figure (6 panels)
```python
fig, axes = plt.subplots(2, 3, figsize=(18, 11))

# 1. Calibration: predicted vs actual ratio (binned)
ax = axes[0, 0]
ax.scatter(cal_table["mean_pred_ratio"], cal_table["mean_actual_ratio"], s=80, color="steelblue", zorder=3)
max_r = max(cal_table["mean_pred_ratio"].max(), cal_table["mean_actual_ratio"].max())
ax.plot([0, max_r*1.1], [0, max_r*1.1], "r--", linewidth=1, label="Perfect calibration")
ax.plot([0, max_r*1.1], [cal_intercept, cal_slope*max_r*1.1 + cal_intercept], "orange",
        linewidth=1.5, label=f"Fit: {cal_slope:.2f}x + {cal_intercept:.2f}")
ax.set_xlabel("Predicted Mean Ratio")
ax.set_ylabel("Actual Mean Ratio")
ax.set_title("Calibration: Predicted vs Actual (Deciles)")
ax.legend(fontsize=8)

# 2. Residual histogram (days)
ax = axes[0, 1]
resid_clipped = test_eval["residual_days"].clip(-200, 500)
ax.hist(resid_clipped, bins=50, color="steelblue", alpha=0.7, edgecolor="white")
ax.axvline(x=0, color="red", linestyle="--", linewidth=1.5, label="Zero")
ax.axvline(x=test_eval["residual_days"].median(), color="orange", linewidth=1.5,
           label=f"Median: {test_eval['residual_days'].median():.0f}d")
ax.set_xlabel("Residual (actual - predicted, days)")
ax.set_ylabel("Count")
ax.set_title("Residual Distribution")
ax.legend(fontsize=8)

# 3. QQ plot of log-ratio residuals (check normality assumption)
ax = axes[0, 2]
resid_log = test_eval["ratio"].apply(np.log) - test_eval["pred_log_ratio"]
sorted_resid = np.sort(resid_log.dropna())
n = len(sorted_resid)
theoretical = stats.norm.ppf(np.arange(1, n+1) / (n+1), loc=0, scale=sigma)
subsample = np.linspace(0, n-1, min(500, n)).astype(int)
ax.scatter(theoretical[subsample], sorted_resid[subsample], s=8, alpha=0.5, color="steelblue")
max_q = max(abs(theoretical[subsample]).max(), abs(sorted_resid[subsample]).max())
ax.plot([-max_q, max_q], [-max_q, max_q], "r--", linewidth=1)
ax.set_xlabel("Theoretical (Normal)")
ax.set_ylabel("Empirical (log-ratio residuals)")
ax.set_title("QQ Plot: Log-Ratio Residuals")

# 4. Prediction interval fan chart (by planned duration)
ax = axes[1, 0]
plan_grid = np.linspace(test_eval["planned_total"].quantile(0.05),
                         test_eval["planned_total"].quantile(0.95), 100)
# Use median predicted log_ratio for the fan
median_log_ratio = float(best_ols.params.iloc[0])  # intercept term (approximate)
for level, z, alpha_val in [(0.50, 0.674, 0.3), (0.80, 1.282, 0.2), (0.95, 1.960, 0.1)]:
    lo = plan_grid * np.exp(median_log_ratio - z * sigma)
    hi = plan_grid * np.exp(median_log_ratio + z * sigma)
    ax.fill_between(plan_grid, lo, hi, alpha=alpha_val, color="steelblue",
                     label=f"{level*100:.0f}% PI")
ax.plot(plan_grid, plan_grid * np.exp(median_log_ratio), "orange", linewidth=1.5, label="Median prediction")
ax.plot(plan_grid, plan_grid, "r--", linewidth=1, label="Perfect (actual=planned)")
# Scatter subsample of test data
sample_idx = np.random.choice(len(test_eval), min(500, len(test_eval)), replace=False)
ax.scatter(test_eval.iloc[sample_idx]["planned_total"],
           test_eval.iloc[sample_idx]["actual_total"],
           s=5, alpha=0.2, color="black", zorder=1)
ax.set_xlabel("Planned Duration (days)")
ax.set_ylabel("Actual Duration (days)")
ax.set_title("Prediction Intervals by Planned Duration")
ax.legend(fontsize=7)
ax.set_xlim(0, plan_grid.max())
ax.set_ylim(0, test_eval["actual_total"].quantile(0.98))

# 5. Bias by PM (bar chart, test set)
ax = axes[1, 1]
if len(pm_bias) > 0:
    pm_bias_sorted = pm_bias.sort_values("mean_bias_days")
    colors = ["steelblue" if b > 0 else "coral" for b in pm_bias_sorted["mean_bias_days"]]
    ax.barh(pm_bias_sorted["group"], pm_bias_sorted["mean_bias_days"], color=colors, alpha=0.7)
    ax.axvline(x=0, color="black", linewidth=1)
    ax.set_xlabel("Mean Bias (days)")
    ax.set_title("Prediction Bias by PM (Test Set)")
    ax.invert_yaxis()

# 6. Bias by year (temporal stability)
ax = axes[1, 2]
if len(year_bias) > 0:
    ax.bar(year_bias["group"].astype(int), year_bias["mean_bias_days"],
           color="steelblue", alpha=0.7)
    ax.axhline(y=0, color="red", linestyle="--", linewidth=1)
    ax.set_xlabel("Year")
    ax.set_ylabel("Mean Bias (days)")
    ax.set_title("Prediction Bias Over Time (Test Set)")

plt.tight_layout()
plt.savefig(str(eda_dir / "duration_regression.png"), dpi=150)
plt.show()
print("Saved eda/duration_regression.png")
```

### Success Criteria:

#### Automated Verification:
- [ ] `eda/duration_regression.png` exists and is >10KB
- [ ] Figure has 6 panels rendered without errors

---

## Phase 6: Assemble Outputs

### Overview
Build JSON and markdown outputs with all model results, metrics, and recommendations.

### Cell 9 — Save outputs
```python
# Assemble output
output = {
    "data_summary": {
        "total_usable": len(df),
        "train_n": len(train),
        "test_n": len(test),
        "train_years": f"{train['year'].min()}-{train['year'].max()}",
        "test_years": f"{test['year'].min()}-{test['year'].max()}",
    },
    "baselines": {
        "global_median_ratio": round(global_median, 3),
        "global_median_metrics": {k: round(v, 2) for k, v in baseline1_metrics.items()},
        "pm_median_metrics": {k: round(v, 2) for k, v in baseline2_metrics.items()},
    },
    "model2_coefficients": {
        name: {"coef": round(float(coef), 4), "se": round(float(se), 4), "p": round(float(p), 4)}
        for name, coef, se, p in zip(model2.params.index, model2.params, model2.bse, model2.pvalues)
    },
    "model2_summary": {
        "r_squared": round(model2.rsquared, 4),
        "adj_r_squared": round(model2.rsquared_adj, 4),
        "residual_sigma": round(np.sqrt(model2.mse_resid), 4),
        "n_predictors": model2.df_model,
    },
    "model3_mixed": {
        "pm_random_effect_std": round(float(np.sqrt(model3.cov_re.iloc[0, 0])), 4),
        "residual_sigma": round(float(np.sqrt(model3.scale)), 4),
        "log_likelihood": round(float(model3.llf), 1),
    },
    "comparison": metrics_df.to_dict("index"),
    "best_model": best_model,
    "calibration": {
        "slope": round(cal_slope, 3),
        "intercept": round(cal_intercept, 3),
        "r_squared": round(cal_r**2, 4),
    },
    "prediction_intervals": {},
    "segment_bias": {
        "by_pm": pm_bias.to_dict("records") if len(pm_bias) > 0 else [],
        "by_sheet_type": type_bias.to_dict("records") if len(type_bias) > 0 else [],
        "by_year": year_bias.to_dict("records") if len(year_bias) > 0 else [],
        "by_complexity": complexity_bias.to_dict("records") if len(complexity_bias) > 0 else [],
    },
}

# Add PI coverage
for level, z in [(0.50, 0.674), (0.80, 1.282), (0.90, 1.645), (0.95, 1.960)]:
    lo = np.exp(log_ratio_pred - z * sigma)
    hi = np.exp(log_ratio_pred + z * sigma)
    coverage = np.mean((test_eval["ratio"] >= lo) & (test_eval["ratio"] <= hi)) * 100
    output["prediction_intervals"][f"{int(level*100)}pct"] = {
        "coverage": round(coverage, 1),
        "target": level * 100,
        "median_width_ratio": round(float(np.median(hi - lo)), 3),
    }

# Markdown report
lines = ["# Duration Regression & Model Evaluation Report\n"]
lines.append("## Summary\n")
lines.append(f"Fit log-linear regression on `log(actual/planned)` using {len(train):,} training records "
             f"(years {train['year'].min()}-{train['year'].max()}), evaluated on {len(test):,} test records "
             f"(years {test['year'].min()}-{test['year'].max()}).\n")

lines.append("## Model Comparison\n")
lines.append("| Model | MAE (days) | RMSE | MAPE | Bias | Within 20% |")
lines.append("|-------|-----------|------|------|------|------------|")
for model_name, m in all_metrics.items():
    lines.append(f"| {model_name} | {m['mae']:.1f} | {m['rmse']:.1f} | {m['mape']:.1f}% | {m['bias']:+.1f} | {m['within_20pct']:.1f}% |")

lines.append(f"\n**Best model (lowest MAE):** {best_model}\n")

lines.append("## Calibration\n")
lines.append(f"- Calibration slope: {cal_slope:.3f} (target: 1.0)")
lines.append(f"- Calibration intercept: {cal_intercept:.3f} (target: 0.0)\n")

lines.append("## Prediction Interval Coverage\n")
lines.append("| Level | Target | Actual Coverage | Width (ratio) |")
lines.append("|-------|--------|-----------------|---------------|")
for level_key, pi in output["prediction_intervals"].items():
    lines.append(f"| {level_key} | {pi['target']:.0f}% | {pi['coverage']:.1f}% | {pi['median_width_ratio']:.3f} |")

lines.append("\n## Segment Bias\n")
if len(pm_bias) > 0:
    lines.append(f"- PM bias range: [{pm_bias['mean_bias_days'].min():.1f}, {pm_bias['mean_bias_days'].max():.1f}] days")
if len(year_bias) > 0:
    lines.append(f"- Year bias range: [{year_bias['mean_bias_days'].min():.1f}, {year_bias['mean_bias_days'].max():.1f}] days")

lines.append("\n## Recommendation\n")
lines.append(f"- **Best model**: {best_model}")
lines.append(f"- **Error structure**: Log-normal (σ = {np.sqrt(model2.mse_resid):.3f} on log scale)")
lines.append(f"- **Operational use**: multiply planned duration by predicted ratio; "
             f"add ±{1.282 * np.sqrt(model2.mse_resid):.2f} on log scale for 80% PI")
lines.append(f"- **Caveat**: temporal split may show drift — check year-level bias before deploying\n")

lines.append("---\n*Generated by notebooks/06_duration_regression.ipynb*")

output["_markdown"] = "\n".join(lines)
save_eda_output(output, "duration_regression", eda_dir)
print(f"Saved eda/duration_regression.json")
print(f"Saved eda/duration_regression.md")
print(f"\nBest model: {best_model}")
```

### Success Criteria:

#### Automated Verification:
- [ ] Notebook runs end-to-end: `source .venv/bin/activate && jupyter nbconvert --execute notebooks/06_duration_regression.ipynb --to notebook --output-dir=notebooks/ --ExecutePreprocessor.kernel_name=dwstables`
- [ ] `eda/duration_regression.json` exists and is valid JSON
- [ ] `eda/duration_regression.md` exists and is >500 chars
- [ ] `eda/duration_regression.png` exists and is >10KB

#### Manual Verification:
- [ ] Calibration plot shows points near diagonal (decile means track well)
- [ ] Prediction interval coverage is within 5% of nominal level (e.g. 80% PI achieves 75-85%)
- [ ] QQ plot of log-ratio residuals is approximately straight (validates Log-normal assumption)
- [ ] No PM has extreme bias (>50 days mean bias) that suggests model is systematically wrong for them
- [ ] Year-level bias doesn't show monotonic drift (would indicate concept drift)
- [ ] Model beats the PM-specific median baseline — if it doesn't, the lookup table is the better "model"

---

## Testing Strategy

### Automated:
- Full notebook execution via `jupyter nbconvert --execute`
- JSON schema validation (all expected keys present)
- Output file size checks

### Manual:
- Visual inspection of calibration and residual diagnostics
- Check whether the model actually improves over the simple baseline
- Assess whether prediction intervals are useful for scheduling (not too wide to be meaningless)
- Cross-check: do the regression coefficients make domain sense?

## References

- Data source: `eda/estimation_variance.json` (`modeling_df` field, 14,042 records)
- GoF analysis: `notebooks/05_duration_goodness_of_fit.ipynb`, `eda/duration_gof.json`
- Previous calibration: notebook 04 (slope=1.164, intercept=15.2)
