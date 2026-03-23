---
date: 2026-03-21T12:00:00-07:00
researcher: Claude
git_commit: f1d40a7
branch: main
repository: dwsTables
topic: "How profitable is Design Workshops?"
tags: [research, profitability, financial, billing, revenue, labor, margins]
status: complete
last_updated: 2026-03-21
last_updated_by: Claude
---

# Research: How Profitable is Design Workshops?

**Date**: 2026-03-21
**Git Commit**: f1d40a7
**Branch**: main

## Research Question
Generally speaking, how profitable is Design Workshops?

## Summary

Design Workshops is a **$15-22M/year commercial millwork operation** that has shifted from high-volume small jobs to fewer, larger projects over 14+ years. The data reveals a business with strong revenue growth per project, a healthy 43% bid win rate, and meaningful change order capture (~21% of revenue). However, **true net profitability cannot be computed from the available data** because material costs from the MAS90 accounting system are incomplete in the exports. What we can observe are strong revenue proxies and labor productivity metrics.

## Key Financial Metrics

### Revenue Scale

| Metric | Value |
|--------|-------|
| Total contracts (all time) | $194.7M |
| Total invoiced (all time) | $236.0M |
| Total change orders | $48.8M (21% of invoiced) |
| Active backlog | $22.2M across 30 projects |
| Completed projects | 1,779 |
| Bid win rate | 43.5% (1,845 awarded / 4,243 bids) |

### Annual Revenue Trajectory

Invoiced revenue by year (from `tblBillHistory`):

| Year | Invoiced | # Invoices | Avg Invoice |
|------|----------|------------|-------------|
| 2010 | ~$16M | 400+ | ~$34K |
| 2013 | ~$17M | 350 | ~$49K |
| 2016 | ~$22M | 285 | ~$77K |
| 2019 | ~$19M | 225 | ~$84K |
| 2022 | ~$20M | 190 | ~$105K |
| 2023 | ~$18M | 172 | ~$105K |

The company bills roughly **$15-22M per year**, with a clear trend: **fewer invoices but larger dollar amounts** over time, indicating a strategic shift toward bigger projects.

### Project Size Distribution

For 1,779 completed projects:

| Size Tier | # Projects | Total Value | % of Revenue |
|-----------|-----------|-------------|--------------|
| < $10K | 965 (54%) | $2.6M | 1.5% |
| $10K - $50K | 280 (16%) | $6.6M | 3.8% |
| $50K - $100K | 104 (6%) | $7.5M | 4.3% |
| $100K - $250K | 98 (6%) | $15.3M | 8.9% |
| $250K - $500K | 65 (4%) | $23.1M | 13.4% |
| $500K - $1M | 49 (3%) | $33.4M | 19.4% |
| > $1M | 41 (2%) | $84.1M | **48.8%** |

**The top 41 projects (2% of count) account for 49% of all revenue.** This is extremely concentrated - the business lives and dies by its large projects.

### Labor Productivity

| Metric | Value |
|--------|-------|
| Total hours tracked | 1,501,709 |
| Peak year (2015) | 145,287 hours |
| Recent year (2023) | 88,370 hours |
| Median revenue/labor hour | **$166/hr** |
| 25th percentile | $127/hr |
| 75th percentile | $238/hr |

Revenue per labor hour has been **relatively stable at $140-200/hr** across years, suggesting consistent labor pricing discipline.

### Labor Cost Estimation

Using burden rates from `tblBurden`:

| Period | Burden Rate |
|--------|-------------|
| 2000-2012 | $44.00/hr |
| 2013 - Jun 2022 | $45.00/hr |
| Jun 2022 - present | $61.05/hr |

**Note**: The burden rate is the fully-loaded cost (wages + benefits + overhead) on top of the weighted average pay rate. The actual labor cost per hour = `WeightAvgPayRate + BurdenRate`. Without the weighted average pay rates from `tblLaborBillRates`, we can only estimate a range.

If the blended labor cost is roughly **$80-$110/hr** (burden + base wage), then:
- At $166/hr median revenue and ~$95/hr labor cost, **labor gross margin is roughly 40-45%**
- But this ignores materials, subcontractors, and overhead

### Material/Purchasing Costs

| Metric | Value |
|--------|-------|
| Purchase records | 31,416 items |
| PO records | 14,687 |
| MAS90 invoice detail | 60,014 line items, $60.8M total |
| Purchasing as % of contract | Median 10.3%, rising to ~20% recently |

The purchasing-to-contract ratio has roughly **doubled from ~8-10% (2011-2013) to ~19-20% (2019-2023)**, suggesting increasing material cost pressure or a shift in project mix toward more material-intensive work.

### Change Order Performance

| Status | Count | Total Value | Avg Value |
|--------|-------|-------------|-----------|
| Executed | 2,216 | $35.2M | $15,900 |
| Approved | 1,267 | $9.9M | $7,800 |
| Void | 367 | $3.4M | - |
| Submitted | 131 | $503K | - |

Change orders represent **~21% of total invoiced revenue** - a healthy capture rate for the construction/millwork industry. This suggests the estimating team prices base contracts competitively and captures additional scope effectively.

### Billing Completeness

For 657 completed projects over $10K:
- Total contract value: $236.1M
- Total invoiced: $227.6M
- **Under-billed by ~$8.4M (3.6%)**

This 3.6% under-billing gap could represent retention not yet released, disputed amounts, write-offs, or projects where final billing was never completed. It's a meaningful number that could impact profitability.

### Top Projects

| Project | Contract | Hours | Rev/Hr |
|---------|----------|-------|--------|
| Thoma Bravo | $7.67M | 33,628 | $228 |
| Asana TI | $6.59M | 30,126 | $216 |
| The Battery | $6.56M | 43,914 | $149 |
| Stripe HQ | $5.96M | 24,280 | $245 |
| Carbonview Beach House | $5.94M | 49,063 | $121 |

Note the wide variance in revenue/hour: Stripe HQ at $245/hr vs Carbonview at $121/hr - a 2x spread that likely maps to very different margin outcomes.

## What We Can and Cannot Determine

### CAN determine from this data:
- Revenue scale and trajectory (~$15-22M/year)
- Project mix and concentration risk (top 2% = 49% of revenue)
- Labor hours and productivity trends
- Change order capture rate (~21%)
- Billing completeness (96.4%)
- Bid win rate (43.5%)
- Revenue per labor hour ($166 median)

### CANNOT determine without additional data:
- **True gross margin** - requires joining MAS90 material costs with labor costs against revenue
- **Net profit** - requires SG&A overhead, rent, equipment, insurance, etc. (not in this database)
- **Margin by project type** - no project category/type classification exists beyond T&M vs fixed-price
- **Estimate accuracy** - requires comparing original `TotProjCost` bid amounts to final actual costs
- **Cash flow / DSO** - payment date tracking exists but is 68% null

### Rough profitability estimate:
For a **$20M/year millwork operation** in the Bay Area with ~100 employees:
- Labor cost (burden-loaded): possibly $8-12M
- Materials/subcontractors: possibly $6-10M (based on purchasing data trends)
- Gross margin: likely **15-30%** ($3-6M)
- Net margin: likely **5-15%** after SG&A

This is consistent with typical commercial millwork/specialty contractor margins, which generally range from 5-15% net.

## Data Sources
- `Tables/tblProject.csv` - 1,809 projects, contract values
- `Tables/tblBillHistory.csv` - 11,618 invoice records
- `Tables/tblCO.csv` - 4,072 change orders
- `Tables/tblTimesheet.csv` - 288,977 timesheet entries
- `Tables/tblPurchasing.csv` - 31,416 purchase records
- `Tables/tblPO.csv` - 14,687 purchase orders
- `Tables/tblBurden.csv` - 3 burden rate periods
- `Tables/tblBid.csv` - 4,243 bids
- `Tables/tmpMas90InvHistDetail.csv` - 60,014 accounting line items
- `metadata/access/queries.csv` - 1,055 Access queries with financial logic
- `catalog/table_annotations.json` - lifecycle stage classifications
- `catalog/entity_graph.md` - 68 relationships from tblProject hub

## Open Questions
1. What are the actual weighted average pay rates to compute true labor cost?
2. Can MAS90 material costs be joined at the project level for gross margin?
3. What drives the 2x variance in revenue/hour across large projects?
4. Why has purchasing as % of contract doubled since 2013?
5. What accounts for the $8.4M under-billing gap on completed projects?
