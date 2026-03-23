---
date: 2026-03-21T18:00:00-07:00
researcher: ARI
git_commit: f1d40a72137b09b99a231e8846700a79c4e280fe
branch: main
repository: dwsTables
topic: "How PyJobShop flexible job shop scheduling could optimize DWS sheet production"
tags: [research, scheduling, optimization, pyjobshop, production, constraint-programming]
status: complete
last_updated: 2026-03-21
last_updated_by: ARI
---

# Research: PyJobShop for DWS Production Scheduling Optimization

**Date**: 2026-03-21
**Researcher**: ARI
**Git Commit**: f1d40a72137b09b99a231e8846700a79c4e280fe
**Branch**: main
**Repository**: dwsTables

## Research Question

How might we broadly use PyJobShop's flexible job shop / hybrid flow shop model to constrain-optimize scheduling of sheets and production for DWS?

## Summary

DWS production is a **hybrid flow shop with stage-skipping**: ~16 sequential stages (Drafting → Approval → Measure → Correct → Stock → Program → Veneer → Mill → Saw → CNC → Other → Bench → Finish → Assembly → Storage → Field), where each sheet follows a subset of stages determined by its workflow type, and some stages have parallel paths (Veneer ∥ Mill→CNC). PyJobShop directly supports this problem class via its `Model` API — each sheet becomes a **Job**, each stage becomes a **Task** within that job, each physical department/machine becomes a **Machine**, and the stage sequence (with skip logic) defines **precedence constraints**. The library's constraint programming solver can then minimize makespan, tardiness, or flow time across all active sheets.

The DWS database already contains all the data needed to parameterize a PyJobShop model: 15,932 sheets with per-stage planned durations (`tblDefaulDurations`), actual stage completion dates (`tblSheetNEW` date columns), labor hours by process code (`tblTimesheet`), and stage sequence configuration (`tblDefaultSheetProc`). The duration goodness-of-fit analysis (notebook 05) shows stage durations follow log-normal distributions with Gamma shape parameters ranging from α=0.59 (Storage) to α=1.12 (CNC), providing empirical processing time distributions for stochastic scheduling.

## Detailed Findings

### 1. PyJobShop Capabilities

PyJobShop is a Python constraint-programming library for scheduling optimization. Key features relevant to DWS:

| Feature | PyJobShop API | DWS Application |
|---------|--------------|-----------------|
| **Jobs** | `m.add_job()` | Each sheet (SK/Sheet) in `tblSheetNEW` |
| **Tasks** | `m.add_task(job)` | Each production stage for that sheet |
| **Machines** | `m.add_machine()` | Each department/workstation in `tblShopAreas` |
| **Processing modes** | `m.add_mode(task, machine, duration)` | Stage duration from `tblDefaulDurations` or historical fit |
| **Precedence** | `m.add_end_before_start(t1, t2)` | Stage sequence from `tblDefaultSheetProc.ProcOrder` |
| **Setup times** | Sequence-dependent setup times | Changeover between sheet types at a machine |
| **Due dates** | Per-job deadlines | `tblSheetNEW.Delivery` date |
| **Release dates** | Per-job release dates | `tblSheetNEW.ToShop` date |
| **Resources** | Renewable resources with capacity | Labor crew size per department |
| **Objectives** | Makespan, tardiness, flow time | Minimize late deliveries / total shop time |

**Problem type mapping**: DWS is closest to a **Hybrid Flow Shop (HFS)** — jobs flow through sequential stages, each stage has one or more parallel resources. PyJobShop's HFS example directly models this: sequential stages, multiple machines per stage, all jobs follow the same stage order (with DWS extension: stage-skipping per workflow type).

### 2. Concept Mapping: DWS → PyJobShop

#### Jobs = Sheets

Each `tblSheetNEW` row is a **Job**. A typical scheduling window has 100-300 active sheets simultaneously (based on the "Monday Report" that reviews all active production). Key job attributes:

- **Release date**: `ToShop` (when the sheet is released to production)
- **Due date / deadline**: `Delivery` (contractual delivery date to jobsite)
- **Priority**: Computed as slack = `Delivery - sum(remaining stage durations)` — negative slack = overdue
- **Weight**: Could weight by project contract value (`tblProject.TotProjCost`) for revenue-weighted tardiness

#### Tasks = Stage Operations per Sheet

Each sheet has up to 16 stage operations. The skip logic in `tblDefaulDurations` determines which stages apply:

```
Workflow: "Pantry - Plam" (plastic laminate)
  Stages: Draft → Approve → Stock → Saw → CNC → Bench → Finish → Assembly → Field
  Skipped: Measure, Correct, Program, Veneer, Mill, Other, Storage

Workflow: "Doors - Veneer"
  Stages: Draft → Approve → Stock → Program → Veneer → Bench → Finish → Assembly → Field
  Skipped: Measure, Correct, Mill, Saw, CNC, Other, Storage
```

Each non-skipped stage becomes an `m.add_task(job)` with precedence constraints enforcing the stage order.

#### Machines = Departments / Work Areas

From `tblShopAreas` (11 areas) cross-referenced with `tblDefaultSheetProc` (22 stage definitions), the physical resources are:

| PyJobShop Machine | DWS Department | Parallel Capacity |
|-------------------|----------------|-------------------|
| Drafting stations | Engineering office | ~3-5 drafters |
| Veneer Shop | Separate physical area | ~2-4 operators |
| Mill Department | Shop - Mill | ~2-3 machines |
| Saw area | Shop - Main (saw station) | ~1-2 saws |
| CNC machines | Shop - Main (CNC area) | ~1-2 CNCs |
| Bench stations | Shop Work Bench | ~6-10 bench workers |
| Finish Department | Shop - Finish Dept. | ~3-5 finish workers |
| Assembly area | Shop - Main | ~3-5 assemblers |
| Ship Station | Shop - Ship Station | ~1-2 loading bays |

The exact parallel capacity per department is not stored in the database but can be estimated from `tblTimesheet` — count distinct employees working in each `ProcessCd` on a given day.

#### Processing Times from Historical Data

The duration GOF analysis provides empirical distributions per stage:

| Stage | Mean (days) | Gamma α | Gamma scale | Notes |
|-------|-------------|---------|-------------|-------|
| Drafting | 11.6 | 0.848 | 13.6 | Engineering, high variance |
| Approval | 19.7 | 0.946 | 20.8 | External dependency (architect) |
| Measure | 16.9 | 0.697 | 24.2 | Field measurement |
| Corrections | 9.4 | 0.876 | 10.7 | |
| Stock | 7.6 | 0.965 | 7.9 | Cannot reject Exponential |
| Programming | 4.4 | 0.608 | 7.2 | |
| Veneer | 8.3 | 1.026 | 8.1 | Cannot reject Exponential |
| Mill | 5.3 | 0.883 | 6.0 | |
| Saw | 3.6 | 1.070 | 3.4 | |
| CNC | 4.1 | 1.119 | 3.6 | |
| Bench | 6.2 | 1.067 | 5.8 | Largest labor cost stage |
| Finish | 5.6 | 0.945 | 5.9 | |
| Assembly | 4.4 | 0.808 | 5.5 | |
| Storage | 8.6 | 0.592 | 14.6 | Buffer/queue time |
| Field | 14.1 | 0.876 | 16.2 | Installation at jobsite |

For deterministic scheduling, use the **mean duration** or the `tblDefaulDurations` template value per workflow type. For robust/stochastic scheduling, sample from the fitted log-normal distributions.

### 3. Parallel Paths: The Veneer/Mill Fork

The real workflow is not purely sequential — it contains a **fork-join**:

```
                     ┌─ Veneer ─────────────┐
Stock/Program ──────┤                       ├──→ Bench → Finish → Assembly
                     └─ Mill → Saw → CNC ──┘
```

In PyJobShop, this is modeled as:
- `add_end_before_start(stock, veneer)`
- `add_end_before_start(stock, mill)`
- `add_end_before_start(veneer, bench)`
- `add_end_before_start(cnc, bench)`

No constraint between veneer and mill — they run in parallel. The solver handles the resource contention (veneer shop vs mill department are separate physical areas, so they truly parallelize).

### 4. What You Could Optimize

**Objective 1: Minimize total tardiness** — `sum(max(0, completion_i - delivery_i))` across all sheets. This directly addresses the estimation variance problem (mean 23.2 days late).

**Objective 2: Minimize makespan** — total time to complete all active sheets. Useful for batch planning (e.g., "when can we clear the current backlog?").

**Objective 3: Minimize weighted tardiness** — weight each sheet's tardiness by contract value. A $500K project sheet being late matters more than a $20K project sheet.

**Objective 4: Level shop loading** — minimize max resource utilization across departments to avoid bottlenecks (Bench/Assembly at $102.67/hr billing rate is likely the constraint).

### 5. Data Pipeline: Database → PyJobShop Model

```python
from pyjobshop import Model
import pandas as pd

# 1. Load active sheets
sheets = pd.read_sql("""
    SELECT s.SheetID, s.ProjectID, s.Sheet, s.SheetType,
           s.ToShop, s.Delivery, s.CurDept,
           d.WorkflowName,
           d.DraftDur1, d.ApprDur2, ..., d.FieldDur,
           d.DraftSkip1, d.ApprSkip2, ..., d.FieldSkip
    FROM tblSheetNEW s
    JOIN tblDefaulDurations d ON s.SheetType = d.WorkflowName  -- or manual mapping
    WHERE s.SheetCompleteDate IS NULL
      AND s.HoldStatus IS NULL
""", conn)

# 2. Load stage sequence
stages = pd.read_csv('metadata/sql_server/tblDefaultSheetProc.csv')
stage_order = stages.sort_values('ProcOrder')

# 3. Build model
m = Model()

# Machines: one per department (or multiple for parallel capacity)
dept_machines = {}
for dept in ['Drafting', 'Veneer', 'Mill', 'Saw', 'CNC',
             'Bench', 'Finish', 'Assembly', 'Field']:
    n_parallel = estimate_capacity(dept)  # from tblTimesheet analysis
    dept_machines[dept] = [
        m.add_machine(name=f"{dept}_{i}")
        for i in range(n_parallel)
    ]

# Jobs and tasks
for _, sheet in sheets.iterrows():
    job = m.add_job(
        name=f"Sheet {sheet.SheetID}",
        release_date=sheet.ToShop,  # converted to integer days
        deadline=sheet.Delivery,     # converted to integer days
    )

    prev_task = None
    for stage_name, dur_col, skip_col in STAGE_COLUMNS:
        if sheet[skip_col] == -1:
            continue  # skip this stage

        duration = int(sheet[dur_col]) if sheet[dur_col] > 0 else DEFAULT_DURATIONS[stage_name]
        task = m.add_task(job, name=f"{sheet.SheetID}_{stage_name}")

        # Each machine in this department can process this task
        for machine in dept_machines[stage_name]:
            m.add_mode(task, machine, duration)

        # Precedence: sequential within a job
        if prev_task is not None:
            m.add_end_before_start(prev_task, task)
        prev_task = task

# 4. Solve
result = m.solve(time_limit=60, display=True)

# 5. Visualize
from pyjobshop.plot import plot_machine_gantt
plot_machine_gantt(result.best, m.data(), plot_labels=True)
```

### 6. Advanced Extensions

**Setup times**: When a CNC machine switches between sheet types (e.g., from a door to a countertop), there's changeover time. PyJobShop supports sequence-dependent setup times via `m.add_setup_time(machine, task_i, task_j, duration)`. This could be estimated from gaps in `tblTimesheet` entries on the same machine.

**Consumable resources**: Materials from `tblPurchasing` — a sheet can't start Bench if its materials haven't arrived (`ReceiveDate IS NULL`). Model as a precedence constraint: material-arrival task must complete before Bench starts.

**Rolling horizon**: Don't schedule all 15,932 sheets at once. Use a 4-8 week rolling window of active sheets (those with `ToShop` within the window and `SheetCompleteDate IS NULL`). Re-solve weekly as new sheets enter and completed sheets exit.

**Stochastic durations**: Use the log-normal fits from the GOF analysis. Run N Monte Carlo scenarios with sampled durations, optimize the expected tardiness. PyJobShop doesn't natively support stochastic scheduling, but you can:
1. Solve deterministically with mean durations
2. Evaluate the schedule under sampled durations
3. Add buffer time to high-variance stages (Drafting α=0.85, Storage α=0.59)

**Integration with MillFlow**: The optimized schedule (start/end dates per task) could feed directly into MillFlow's `ScheduleView` component, replacing the current manual sequencing. The Gantt output from PyJobShop maps to MillFlow's timeline view.

### 7. Estimation Variance Connection

The existing estimation variance analysis (actual = 1.164 × planned + 15.2, R²=0.702) provides calibrated duration estimates. Instead of using raw `tblDefaulDurations` values (which are systematically optimistic), use:

```python
calibrated_duration = 1.164 * planned_duration + 15.2 / n_stages
```

Or better: use the per-PM random intercepts from the hierarchical model (once built) to adjust durations based on which PM is running the project.

## Architecture Documentation

### Data Sources for Model Parameterization

| Parameter | Source Table | Column(s) |
|-----------|-------------|-----------|
| Sheet identity | `tblSheetNEW` | `SheetID`, `ProjectID`, `SheetType` |
| Release date | `tblSheetNEW` | `ToShop` |
| Due date | `tblSheetNEW` | `Delivery` |
| Stage sequence | `tblDefaultSheetProc` | `ProcOrder`, `SheetProcess` |
| Skip logic | `tblDefaulDurations` | `*Skip*` columns per workflow type |
| Planned durations | `tblDefaulDurations` | `*Dur*` columns per workflow type |
| Actual durations | `tblSheetNEW` | Date columns (`DraftDt1`, `CNCDt10`, etc.) |
| Labor capacity | `tblTimesheet` | `ProcessCd`, `Employee`, `WorkDate` |
| Department map | `tblShopAreas` | `ShopAreaName` |
| Process codes | `tblProcCd` | `Process` (16-category rollup) |
| Material readiness | `tblPurchasing` | `ReceiveDate`, `ETA` |
| Contract value | `tblProject` | `TotProjCost` (for weighted objectives) |

### Key Quantitative Facts

- 15,932 total sheets, ~100-300 active at any time
- 16 production stages, 18 workflow types with different skip patterns
- 288,977 timesheet entries across 276 employees (capacity estimation)
- Mean estimation variance: +23.2 days (systematically optimistic)
- Stage durations: log-normal distributed, mean 3.6 days (Saw) to 19.7 days (Approval)
- Pooled Gamma α = 0.721 across stages (right-skewed, long-tailed)

## Historical Context (from thoughts/)

- `thoughts/repos/dwsERP/shared/research/2025-12-22-millflow-product-scope.md` — MillFlow frontend prototype for shop scheduling, not yet connected to real data or optimization
- `thoughts/repos/dwsERP/shared/plans/2025-12-23-shop-schedule-integration.md` — Plan for integrating schedule data into MillFlow
- `thoughts/repos/dwsERP/shared/research/2025-12-23-ARI-42-time-estimates-timeline-integration.md` — Research on connecting time estimates to timeline views
- `thoughts/repos/dwsTables/shared/research/2026-03-21-shop-schedule-production-workflow.md` — Ground truth production workflow from Access DB analysis

## Open Questions

1. **Department capacity**: How many parallel workers/machines per department? Must be estimated from `tblTimesheet` (max concurrent workers per `ProcessCd` per day)
2. **Approval stage**: Is this an external dependency (architect review) or internal? If external, it should be modeled as a fixed delay rather than a machine-scheduled task
3. **Setup times**: Are there measurable changeover costs when switching between sheet types at a given machine? Would need to analyze `tblTimesheet` gaps
4. **Partial completion**: Can a sheet be partially complete at a stage and moved? Or must each stage fully complete before the next starts?
5. **Sheet batching**: Are sheets from the same project batched together for shipping? If so, project-level constraints (all sheets complete before any ship) add coupling
6. **CurDept accuracy**: Is `tblSheetNEW.CurDept` reliably updated? This determines the initial state for warm-starting the optimizer

---
*Generated by research session 2026-03-21*
