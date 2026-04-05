# DWS Agent Harness — Proof of Concept

## Overview

Build a self-learning data analyst agent for the DWS database that lives inside Claude Code as a skill. Inspired by [agno-agi/dash](https://github.com/agno-agi/dash)'s 6-layer context architecture, but adapted to Claude Code's extensibility model (skills, subagents, knowledge files) and using pandas against local CSVs instead of live SQL.

The PoC delivers: a `/dws-analyst` slash command that answers business questions about DWS data by reading consolidated knowledge, writing pandas code, executing it, interpreting results as insights, and saving gotchas for future sessions.

## Current State Analysis

### What exists (strong foundation)
- **128 table profiles** in `catalog/profiles/*.json` — column stats, types, null rates, top values
- **341 inferred relationships** in `catalog/relationships.json` — column-pair joins with confidence + source evidence
- **Table annotations** in `catalog/table_annotations.json` — lifecycle stages, importance tiers (but `description` field is empty)
- **Rich EDA findings** in `eda/*.json` — business metrics, deep dives, quality audit, pipeline trace, duration modeling
- **Helper utilities** in `scripts/eda_helpers.py` — `load_table()`, `TIER_1_TABLES`, `save_eda_output()`
- **122 CSV exports** in `Tables/` — queryable via pandas on Mac

### What's missing (the gaps this plan fills)
1. No agent skill / entry point
2. No consolidated knowledge files optimized for agent context
3. No validated query patterns
4. No learning/gotcha capture mechanism
5. No table descriptions (the `description` field is empty everywhere)

## Desired End State

After this plan:
- `/dws-analyst` skill exists and can answer questions like "What's the average margin by project size?" or "Which PMs have the worst schedule variance?"
- Knowledge directory contains distilled semantic model, business rules, and join map — all markdown, all concise enough to inject into context
- A `knowledge/learnings/` directory exists where the agent saves gotchas it discovers
- A starter set of validated query patterns exists for common questions
- The architecture is extensible: subagents, MCP servers, and richer knowledge can be layered in later

### Verification
- Invoke `/dws-analyst` with 3-5 test questions spanning different domains (revenue, labor, scheduling)
- Agent loads correct tables, writes working pandas code, returns interpreted insights
- Agent saves at least one learning when it encounters a data gotcha

## What We're NOT Doing

- Live SQL Server connection / MCP server
- Filling descriptions for all 128 tables (just Tier 1)
- Eval framework / automated grading
- Vector DB / embeddings (Claude reads files directly)
- Full business logic reverse engineering from Access forms/reports
- Subagent architecture (deferred to Phase 2)

## Implementation Approach

Three phases, each independently testable:

1. **Knowledge distillation** — consolidate catalog + EDA artifacts into agent-ready markdown files
2. **Skill authoring** — write the `.claude/skills/dws-analyst.md` skill with instructions, context injection, and learning mechanism
3. **Smoke test + iterate** — run real questions, capture learnings, tune the skill

---

## Phase 1: Knowledge Distillation

### Overview
Distill the existing catalog and EDA artifacts into three concise markdown files that the skill will reference. These are human-curated summaries, not raw dumps.

### Changes Required:

#### 1. `knowledge/semantic_model.md` — Table descriptions + key columns

Distill from `catalog/profiles/*.json` + `catalog/table_annotations.json` for Tier 1 tables (11 tables).

**Format per table:**
```markdown
### tblProject (1,809 rows) — Core project record
**Stage:** Projects | **Tier:** 1

Key columns:
- `ProjectID` (int, PK) — unique project identifier, range 1860–3990
- `ProjectName` (varchar) — project name, ~94% distinct
- `TotProjCost` (numeric) — total project cost = FixtureCost + NonFixtureCost (see business rules)
- `PM` (int) — project manager, FK to tblContact.ContactID
- `Client` (int) — client company, FK to tblCompany.CompanyID
- `DateAdded` (smalldatetime) — project creation date
- `Completed` (smalldatetime) — completion date, null if active

Data quality:
- 100% populated for ProjectID, ProjectName, TotProjCost
- PM is 2.3% null
- Completed is ~15% null (active/abandoned projects)
```

**Source data:** Read each profile JSON, extract the most useful columns (non-null, high-cardinality, or FK columns). Add human-written descriptions from domain knowledge in the EDA outputs.

Write descriptions for these 11 tables: `tblProject`, `tblTimesheet`, `tblContact`, `tblCompany`, `tblBid`, `tblBill`, `tblBillHistory`, `tblBillHistDetail`, `tblPO`, `tblPurchasing`, `tblSheetNEW`.

#### 2. `knowledge/business_rules.md` — Metrics, rules, gotchas

Consolidate from `eda/business_metrics.json`, `eda/deep_dives.json`, `eda/quality_audit.json`.

**Sections:**
```markdown
## Key Metrics
- **TotProjCost**: = FixtureCost + NonFixtureCost for 80% of projects. Tracks actual billings within 5% for 98.5%. Does NOT include change orders.
- **True Labor Cost**: RegHrs × PayRate1 from tblEmpPayHist (date-range join). $42.79 avg rate. 99.5% timesheet match.
- **Gross Margin**: (TotProjCost + CO_billed - labor_cost - material_cost) / (TotProjCost + CO_billed). Median 58.4%.
...

## Business Rules
- Revenue: $194.7M total, 319 clients. Top 3 = 55.6% (Skyline, Principal, BCCI)
- Estimate accuracy: mean variance -6.1% (slight overestimation), 28.5% under-budget
- Labor productivity: median 6.4 hrs/$1K contract value
- Pipeline: 45.8% of projects flow through all 6 stages
...

## Data Gotchas
- tblPO.POAttn → tblContact.ContactID has 16.76% orphan rate — always LEFT JOIN
- tblBillHistory.ProjectID → tblProject.ProjectID has 1.39% orphan rate — INNER JOIN is safe
- tblSheetNEW has 117 columns but many are >90% null
- tblTimesheet: DeductCd, NoBillHrs, NoBillHrType, SpecialRate, UVCheckNo are 100% null
...
```

#### 3. `knowledge/join_map.md` — Key relationships

Distill from `catalog/relationships.json`. Focus on Tier 1 table relationships only (~50-60 of the 341 total).

**Format:**
```markdown
## tblProject (hub — 68 connections)
| Related Table | This Column | That Column | Join Type | Confidence | Orphan Rate |
|---|---|---|---|---|---|
| tblBid | ProjectID | JobNo | INNER | high | <1% |
| tblTimesheet | ProjectID | Project | LEFT | high | ~2% |
| tblContact | PM | ContactID | LEFT | high | 2.3% null |
| tblCompany | Client | CompanyID | INNER | high | <1% |
| tblBill | ProjectID | ProjectID | INNER | high | <1% |
| tblPO | ProjectID | Project | LEFT | high | ~3% |
...
```

Annotate with orphan rates from `eda/quality_audit.json` where available to guide JOIN type selection.

#### 4. `knowledge/learnings/` — Empty directory with README

```markdown
# Learnings

This directory stores gotchas discovered by the DWS analyst agent during query sessions.
Each file is a markdown note saved when the agent encounters a data quality issue,
type mismatch, or domain-specific quirk not captured in the business rules.

Format: `YYYY-MM-DD-brief-description.md`
```

### Success Criteria:

#### Automated Verification:
- [x] `knowledge/semantic_model.md` exists and contains all 11 Tier 1 tables
- [x] `knowledge/business_rules.md` exists and contains metrics, rules, and gotchas sections
- [x] `knowledge/join_map.md` exists and contains tblProject + at least 5 other Tier 1 table sections
- [x] `knowledge/learnings/` directory exists

#### Manual Verification:
- [ ] Semantic model descriptions are accurate to the domain (not just column stats)
- [ ] Business rules match the validated findings in EDA notebooks
- [ ] Join map orphan rate annotations match quality_audit.json

**Pause here for review before Phase 2.**

---

## Phase 2: Skill Authoring

### Overview
Write the `/dws-analyst` skill that ties everything together: context injection, query instructions, pandas workflow, insight interpretation, and learning capture.

### Changes Required:

#### 1. `.claude/skills/dws-analyst.md`

```markdown
---
name: dws-analyst
description: Self-learning data analyst for the DWS millwork database
user_invocable: true
---

# DWS Data Analyst

You are a data analyst for Design Workshops (DWS), a commercial millwork shop
in Oakland, CA with 20+ years of project data across 128 tables.

## Your Purpose

Answer business questions about DWS operations by querying CSV exports via pandas.
Provide **insights, not just numbers** — contextualize findings, compare to benchmarks,
and flag data quality issues.

## Before Every Query

1. Read `knowledge/semantic_model.md` for table/column descriptions
2. Read `knowledge/join_map.md` for relationship details relevant to your query
3. Read `knowledge/business_rules.md` for metrics definitions and known gotchas
4. Check `knowledge/learnings/` for any previously discovered issues with the tables you'll use

## How to Query

Use pandas via the Bash tool. Always use the project's loader:

\```python
import sys
sys.path.insert(0, 'scripts')
from eda_helpers import load_table, TIER_1_TABLES

# Load tables
proj = load_table('tblProject')
ts = load_table('tblTimesheet')

# Your analysis here...
\```

### Query Rules
- Always inspect dtypes and null counts before analysis
- Use `.merge()` with `how='left'` unless join_map says INNER is safe
- Limit output to top 20 rows for display
- Handle the common gotchas (type mismatches, null columns, orphan FKs)

## After Every Query

1. **Interpret results** — Don't just show a dataframe. Explain what the numbers mean.
   Bad: "Hamilton: 11 wins"
   Good: "Hamilton dominated 2019 with 11 wins out of 21 races (52%), more than double Bottas's 4."

2. **If a query failed or revealed something unexpected** — save a learning:
   Create a file in `knowledge/learnings/YYYY-MM-DD-brief-description.md`:
   \```markdown
   # [Brief title]
   **Discovered**: [date]
   **Tables**: [affected tables]
   **Issue**: [what went wrong]
   **Solution**: [how to handle it]
   \```

3. **If the query pattern is reusable** — mention that it could be saved as a validated pattern.

## Known Tier 1 Tables
tblProject, tblTimesheet, tblContact, tblCompany, tblBid, tblBill,
tblBillHistory, tblBillHistDetail, tblPO, tblPurchasing, tblSheetNEW

## Available EDA Context
For deeper context on any metric, read the relevant file:
- `eda/business_metrics.json` — revenue, labor, material, estimation
- `eda/deep_dives.json` — corrected metrics with full methodology
- `eda/quality_audit.json` — referential integrity audit
- `eda/pipeline_trace.json` — project lifecycle coverage
- `eda/estimation_variance.json` — schedule variance by PM/estimator
```

#### 2. Ensure `.claude/skills/` directory exists

Create the directory and verify the skill is recognized by Claude Code.

### Success Criteria:

#### Automated Verification:
- [x] `.claude/skills/dws-analyst.md` exists with valid frontmatter
- [x] Skill appears in Claude Code's available slash commands

#### Manual Verification:
- [ ] `/dws-analyst` is invocable from Claude Code
- [ ] The skill's instructions are injected into context when invoked

**Pause here for review before Phase 3.**

---

## Phase 3: Smoke Test + Iterate

### Overview
Run 5 test questions through the agent, verify it produces correct insights, and iterate on the skill/knowledge files based on what breaks.

### Test Questions:

1. **Revenue concentration**: "Who are DWS's top 10 clients by total project cost?"
   - Expected: Uses tblProject + tblCompany join, mentions $194.7M total, top 3 = 55.6%
   - Tables: tblProject, tblCompany

2. **Labor productivity**: "What's the average labor hours per $1K of contract value, broken down by project size?"
   - Expected: Uses tblTimesheet + tblProject, mentions median 6.4 hrs/$1K, size buckets
   - Tables: tblProject, tblTimesheet

3. **Schedule variance**: "Which project managers have the worst schedule estimation accuracy?"
   - Expected: Uses tblSheetNEW + tblContact, mentions mean variance 23.2 days
   - Tables: tblSheetNEW, tblProject, tblContact

4. **Margin analysis**: "What's the gross margin distribution for projects over $500K?"
   - Expected: Uses tblProject + tblTimesheet + tblPurchasing, mentions median 58.4%
   - Tables: tblProject, tblTimesheet, tblPurchasing, tblBillHistory

5. **Data exploration**: "What are the most common reasons projects don't have purchasing data?"
   - Expected: Uses pipeline_trace.json context, mentions 34.8% follow bid+timesheet+billing pattern
   - Tables: tblProject, tblPO, tblPurchasing

### Iteration Process:
- Run each question
- Note any failures, incorrect joins, missing context
- Update knowledge files or skill instructions as needed
- If a gotcha is discovered, verify the agent saves it to `knowledge/learnings/`

### Success Criteria:

#### Automated Verification:
- [x] All 5 test queries execute without Python errors
- [x] At least one learning file exists in `knowledge/learnings/`

#### Manual Verification:
- [ ] Results are factually correct (cross-check against EDA notebook outputs)
- [ ] Insights are interpreted, not just raw numbers
- [ ] Agent reads knowledge files before querying
- [ ] Agent uses correct join types per the join map

---

## Future Work (out of scope for PoC)

### Phase 2a: Subagent Architecture
- Extract query execution into `.claude/agents/query-runner.md` subagent
- Main skill dispatches to subagent, keeping context window clean
- Subagent specializes in pandas + data quality checks

### Phase 2b: Validated Query Library
- `knowledge/queries/` directory with reusable pandas patterns
- Agent searches these before writing new code
- Save mechanism from successful queries

### Phase 2c: Enhanced Business Logic
- Reverse engineer Access forms/reports for deeper business rule extraction
- Fill descriptions for Tier 2-3 tables
- Add temporal context (data freshness, seasonal patterns)

### Phase 3: Live SQL + MCP
- MCP server wrapping pyodbc or ODBC connection to SQL Server
- Switch from CSV pandas to live queries
- Runtime schema introspection tool

## References

- Dash repo (inspiration): https://github.com/agno-agi/dash
- Claude Code skills docs: https://code.claude.com/docs/en/skills
- Claude Code subagents: https://code.claude.com/docs/en/sub-agents
- Existing catalog: `catalog/data_dictionary.md`, `catalog/entity_graph.md`
- EDA outputs: `eda/*.json`
