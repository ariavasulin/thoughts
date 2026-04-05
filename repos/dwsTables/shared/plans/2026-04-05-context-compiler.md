# DWS Context Compiler — Implementation Plan

## Overview

A ralph-wiggum-style autonomous agent loop that continuously builds the semantic knowledge layer for the DWS database. Each iteration: pick a task from a shared TODO, investigate the data, write findings to `knowledge/`, discover new tasks, commit, exit. The loop restarts with a fresh context window.

**Upstream of**: `/dws-analyst` skill (the consumer — already has progressive loading via `context_loader.py`, `search_learnings.py`, `search_access_queries.py`)

**Inspired by**:
- **Ralph Wiggum**: Bash loop, declarative prompt, small independent context windows. The loop is 5 lines — the prompt is where the value lives.
- **Autoresearch**: Single metric to optimize, git as memory, immutable evaluation harness. Each iteration builds on accumulated wins.
- **OpenAI Data Agent**: 6-layer context. Access queries = Layer 3 (code-level enrichment). Self-improving knowledge = Layer 5 (learning memory).
- **Harness Engineering**: Constraint + back-pressure. Confine the mutation surface, verify with an immutable harness.

## Current State

### Already built (the consumer side)
- `/dws-analyst` skill with progressive context loading, hooks, learnings
- `knowledge/semantic_model.md` — Tier 1 table profiles (11 tables, deep)
- `knowledge/business_rules.md` — metrics and formulas
- `knowledge/join_map.md` — Tier 1 join patterns with orphan rates
- `knowledge/learnings/` — 5 data gotcha files with YAML frontmatter
- `knowledge/queries/` — 5 validated pandas patterns
- `catalog/` — 122 table profiles (JSON), 341 relationships, annotations

### The gap (what the compiler fills)
- **111 undocumented tables**: Tier 2 (36), Tier 3 (62), Tier 4 (13) have no semantic descriptions
- **702 unanalyzed user queries**: Access queries encode institutional knowledge — untapped
- **15 lifecycle stages undocumented**: No process flow documentation
- **No cross-table column glossary**: Same column names appear across tables with different meanings
- **No master index**: No progressive-disclosure entry point for the analyst

## Desired End State

After the compiler runs to completion:
1. Every non-empty, non-backup table has a knowledge file documenting its business purpose, key columns, relationships, Access query usage, and gotchas
2. All 702 user-created Access queries are categorized by business function and analyzed
3. Each lifecycle stage has a flow document showing how data moves through it
4. A column glossary maps shared column names to their meanings across tables
5. An INDEX.md serves as the progressive-disclosure entry point
6. The `/dws-analyst` skill can answer any question a human power user could

### Verification
- `verify.py` reports >80% table coverage, >60% query coverage, >80% flow coverage
- 5 test questions that previously required human knowledge are answered correctly by `/dws-analyst`

## What We're NOT Doing

- Modifying the analyst skill or its scripts (that's the consumer — already done)
- Building vector search or embeddings (grep + structure is sufficient at this scale)
- Live SQL Server queries (working from CSV exports)
- Building a UI or dashboard
- Automated quality scoring of knowledge content (human review is the quality gate)

---

## Artifact Space — Exactly What the Agents Produce

### Constrained mutation surface

Agents can ONLY modify files in these locations:

```
knowledge/                          ← KNOWLEDGE OUTPUT
├── INDEX.md                        ← NEW: master table of contents
├── semantic_model.md               ← EXISTING: Tier 1 (read-only for compiler)
├── business_rules.md               ← EXISTING: metrics (append-only)
├── join_map.md                     ← EXISTING: joins (append-only)
├── learnings/                      ← EXISTING: append new files only
├── queries/                        ← EXISTING: append new files only
├── tables/                         ← NEW: per-table documentation
│   ├── tblSheetProcess.md
│   ├── tblShipTagItems.md
│   └── ... (one file per table)
├── access_analysis/                ← NEW: Access query analysis
│   ├── overview.md                 ← query inventory + categorization
│   ├── estimating.md               ← estimating/bidding queries
│   ├── billing.md                  ← billing/invoicing queries
│   ├── labor.md                    ← labor/timesheet queries
│   ├── purchasing.md               ← purchasing/PO queries
│   ├── engineering.md              ← engineering/scheduling queries
│   ├── field.md                    ← field work queries
│   ├── project_management.md       ← project/job queries
│   └── other.md                    ← uncategorized queries
├── flows/                          ← NEW: lifecycle process docs
│   ├── estimate_to_contract.md
│   ├── contract_to_engineering.md
│   ├── engineering_to_production.md
│   ├── production_to_field.md
│   ├── billing_and_closeout.md
│   └── purchasing_flow.md
└── glossary.md                     ← NEW: cross-table column dictionary

context_compiler/TODO.md            ← TASK LIST (read + write)
```

### What agents CANNOT modify

```
context_compiler/PROMPT.md          ← immutable declarative spec
context_compiler/verify.py          ← immutable evaluation harness
context_compiler/run.sh             ← immutable loop script
catalog/                            ← read-only reference data
metadata/                           ← read-only reference data
Tables/                             ← read-only source data
scripts/                            ← read-only helper code
.claude/skills/dws-analyst/         ← read-only (consumer side)
```

### File Templates

#### Per-table file (`knowledge/tables/{tblName}.md`)

```markdown
# {tblName}

**Stage:** {lifecycle_stage} | **Tier:** {tier} | **Rows:** {row_count} | **Query Refs:** {query_references}

## Business Purpose

{1-3 sentences: What this table records and WHY it exists in the DWS workflow. Not just "stores data about X" — explain the business context.}

## Key Columns

| Column | Type | Business Meaning | FK Target | Null % | Notes |
|--------|------|-----------------|-----------|--------|-------|
| {col} | {type} | {what it means in business terms} | {table.column or —} | {X%} | {gotchas} |

## Relationships

| Related Table | This→That | Join Type | Orphan Rate | Confidence | Notes |
|---------------|-----------|-----------|-------------|------------|-------|
| {table} | {col}→{col} | INNER/LEFT | {X%} | high/medium | {context} |

## Access Query Usage

{Which Access queries reference this table. List the most important ones with what they reveal about how DWS uses this data.}

- **{queryName}**: {what it does, what business question it answers}
- ...

If no Access queries reference this table, note that explicitly.

## Data Quality

{Known issues: empty columns, encoding quirks, suspicious patterns, columns that look like they should be FKs but aren't.}

## Workflow Context

{Where this table sits in the DWS lifecycle. What comes before/after it. How it connects to the broader business process.}
```

#### Access query analysis file (`knowledge/access_analysis/{function}.md`)

```markdown
# {Business Function} Queries

## Overview

{How many queries in this category. What they do as a group. What this tells us about how DWS uses the data for this function.}

## Key Queries

| Query Name | Tables | Purpose | Key Logic |
|------------|--------|---------|-----------|
| {name} | {tables joined} | {business question} | {notable WHERE/GROUP BY/calculation} |

## Business Rules Revealed

{What these queries teach us about DWS business rules that aren't obvious from the schema alone. Specific filter conditions, calculation patterns, join logic that encodes institutional knowledge.}

## Discovered Relationships

{Joins found in queries that aren't in catalog/relationships.json. These are high-confidence relationship discoveries.}
```

#### Lifecycle flow file (`knowledge/flows/{flow_name}.md`)

```markdown
# {Stage A} → {Stage B}

## Tables Involved

| Table | Role in This Flow |
|-------|-------------------|
| {table} | {how it participates} |

## How Data Moves

{Step by step, what happens as work moves through this stage. Which columns track the transition. What triggers the transition.}

## Key Columns Tracking This Flow

| Table.Column | What It Signals |
|-------------|----------------|
| {table.col} | {when this is set, what it means} |

## Common Access Queries

{Queries that span this transition — they show how DWS staff track this flow.}
```

#### Column glossary (`knowledge/glossary.md`)

```markdown
# DWS Column Glossary

Cross-table column reference. Same column name can mean different things in different tables.

## ID Columns

| Column Name | Found In | Meaning | FK Target |
|-------------|----------|---------|-----------|
| ProjectID | tblProject, tblTimesheet, tblBill, ... | Project identifier | tblProject.ProjectID |
| ContactID | tblContact | Person identifier | — (is PK) |
| Employee | tblTimesheet | Worker identity | tblContact.ContactID |
| PM | tblProject | Project manager | tblContact.ContactID |
| ... | ... | ... | ... |

## Code Columns

| Column Name | Found In | Meaning | Lookup Table |
|-------------|----------|---------|-------------|
| ProcessCd | tblTimesheet | Department/process code | tblProcCdNew.ProcCd |
| EarningCd | tblTimesheet | Pay type (01=reg, 02=OT) | tblEarningCd |
| ... | ... | ... | ... |

## Date Columns

| Column Name | Found In | Meaning |
|-------------|----------|---------|
| DateAdded | tblProject | Project creation date |
| WorkDate | tblTimesheet | Date of labor entry |
| ... | ... | ... |
```

#### INDEX.md (`knowledge/INDEX.md`)

```markdown
# DWS Knowledge Base Index

Quick reference for what's documented. Updated by the context compiler.

## Coverage
- Tables: X/122 documented (Y%)
- Access queries: X/702 analyzed (Y%)
- Lifecycle flows: X/6 documented (Y%)

## Table Documentation

### Tier 1 (Core — 11 tables)
All documented in `semantic_model.md`.

### Tier 2 (Important — 36 tables)
| Table | File | Stage | Status |
|-------|------|-------|--------|
| tblSheetProcess | tables/tblSheetProcess.md | Engineering | ✓ |
| ... | ... | ... | pending |

### Tier 3 (Supporting — 62 tables)
...

## Access Query Analysis
| Function | File | Queries | Status |
|----------|------|---------|--------|
| Estimating | access_analysis/estimating.md | ~X queries | ✓ |
| ... | ... | ... | pending |

## Lifecycle Flows
| Flow | File | Status |
|------|------|--------|
| Estimate → Contract | flows/estimate_to_contract.md | ✓ |
| ... | ... | pending |

## Other References
- Column glossary: `glossary.md`
- Business rules: `business_rules.md`
- Join patterns: `join_map.md`
- Validated query patterns: `queries/`
- Data gotchas: `learnings/`
```

---

## Harness Architecture

Four files in `context_compiler/`. The PROMPT is the agent. The TODO is the backlog. The verify is the scorecard. The run is the loop.

```
context_compiler/
├── PROMPT.md       ← declarative spec (what the agent IS and what done looks like)
├── TODO.md         ← shared task list (agents pick, complete, discover tasks)
├── verify.py       ← immutable scorecard (coverage + format checks)
└── run.sh          ← while : ; do claude -p PROMPT.md ; done
```

### Harness flow per iteration

```
┌─ run.sh restarts claude with fresh context ─┐
│                                              │
│  1. Agent runs verify.py → reads scorecard   │
│  2. Agent reads TODO.md → picks top task     │
│  3. Agent investigates:                      │
│     - Read table CSV from Tables/            │
│     - Read profile from catalog/profiles/    │
│     - Search Access queries in metadata/     │
│     - Check catalog/relationships.json       │
│     - Cross-ref existing knowledge/ docs     │
│  4. Agent writes to knowledge/               │
│  5. Agent updates TODO.md:                   │
│     - Mark task done                         │
│     - Add discovered tasks                   │
│  6. Agent updates INDEX.md                   │
│  7. Agent commits                            │
│  8. Agent exits → loop restarts              │
│                                              │
└──────────────────────────────────────────────┘
```

### Context budget per iteration

Each iteration should consume roughly:
- PROMPT.md: ~2,000 tokens (loaded once)
- verify.py output: ~500 tokens
- TODO.md: ~1,000 tokens
- Table CSV sample: ~2,000 tokens (head of CSV, not full file)
- Profile JSON: ~1,000 tokens
- Access query search results: ~2,000 tokens
- Existing knowledge cross-reference: ~1,000 tokens
- **Total investigation context: ~10K tokens**
- Writing + TODO update + commit: ~3K tokens
- **Total per iteration: ~13K tokens** — well within the smart zone

---

## Phase 1: Build the Harness

Create the 4 files in `context_compiler/` and seed `knowledge/INDEX.md`.

### 1a. PROMPT.md

The declarative specification. This is the most important file — it's the "soul" of the compiler.

**File**: `context_compiler/PROMPT.md`

Key design principles (from harness engineering research):
- **Declarative, not imperative**: Describe what done looks like, not step-by-step instructions
- **Constrained mutation surface**: Explicitly list what you can and cannot modify
- **Progressive disclosure**: Point to files, don't dump content
- **One task per iteration**: Keep context windows small and focused

```markdown
# DWS Context Compiler

You are building the semantic knowledge layer for a Design Workshops (DWS) operations agent. Your goal: make a future AI agent as capable as a human power user of the DWS database.

DWS is a commercial millwork shop in Oakland, CA. Its SQL Server database (128 tables) + Access frontend (1,055 queries) runs the entire business: estimating, projects, engineering, purchasing, labor tracking, field work, billing, and closeout. The database has NO foreign keys — all relationships are implicit in the application logic and Access queries.

## What Done Looks Like

Every table has a documented business purpose, key columns, relationships, and data quality notes. Every Access query cluster is analyzed for the business rules it encodes. Every lifecycle stage has a flow document showing how data moves through it. A column glossary maps shared names across tables. An INDEX.md tracks coverage.

## Your Workspace

### You CAN modify:
- `knowledge/tables/*.md` — per-table documentation (one file per table)
- `knowledge/access_analysis/*.md` — Access query analysis by business function
- `knowledge/flows/*.md` — lifecycle process documentation
- `knowledge/glossary.md` — cross-table column dictionary
- `knowledge/INDEX.md` — master index (update coverage counts)
- `knowledge/business_rules.md` — append newly discovered rules
- `knowledge/learnings/` — append new data gotcha files
- `context_compiler/TODO.md` — the shared task list

### You CANNOT modify:
- `context_compiler/PROMPT.md` (this file)
- `context_compiler/verify.py`
- `context_compiler/run.sh`
- `knowledge/semantic_model.md` (Tier 1 reference — already complete)
- `knowledge/join_map.md` (Tier 1 joins — already complete)
- `knowledge/queries/` (validated pandas patterns — already complete)
- Anything in `catalog/`, `metadata/`, `Tables/`, `scripts/`, `.claude/`

### You READ from:
- `Tables/{tblName}.csv` — raw table data (read sparingly — sample, don't load all)
- `catalog/profiles/{tblName}.json` — column stats, null rates, value distributions
- `catalog/relationships.json` — 341 extracted relationships with confidence levels
- `catalog/table_annotations.json` — lifecycle stage, tier, row counts per table
- `metadata/access/queries.csv` — 1,055 Access queries (columns: name, query_type, sql_reconstructed, table_count, field_count)
- `knowledge/` — existing documentation to cross-reference and build on

## How To Work

1. **Run the scorecard**: `python3 context_compiler/verify.py` — see what's covered and what's missing
2. **Read the TODO**: `context_compiler/TODO.md` — pick the top uncompleted task
3. **Investigate ONE thing thoroughly**:
   - Read the table profile: `catalog/profiles/{tblName}.json`
   - Sample the actual data: read first 30 lines of `Tables/{tblName}.csv`
   - Check relationships: grep for the table in `catalog/relationships.json`
   - Search Access queries: grep `metadata/access/queries.csv` for the table name (queries use `dbo_{tblName}` prefix)
   - Read related existing knowledge for cross-referencing
4. **Write your findings** to the appropriate file in `knowledge/` following the templates
5. **Update TODO.md**: mark your task `[x]`, add any new tasks you discovered
6. **Update INDEX.md**: update the relevant coverage entry
7. **Git commit** with message: `compiler: {what you documented}`
8. **Stop.** Do not pick up another task. The loop will restart you fresh.

## Quality Standards

- **Business meaning over technical description**: "Records which carpenter is assigned to which field work order" not "Stores integer FK relationships between contacts and work orders"
- **Cite your sources**: "(from Access query qryBidSummary)" or "(inferred from column name pattern)" or "(confirmed by data: 98% non-null)"
- **Flag uncertainty**: If you're not sure about a column's meaning, say so explicitly rather than guessing
- **Depth over breadth**: A thorough analysis of one table beats shallow descriptions of five
- **When surprised, add a task**: If you discover something unexpected, add it to TODO.md for a future iteration

## Access Query Conventions
- `dbo_` prefix in SQL = linked SQL Server table (e.g., `dbo_tblProject` = `tblProject`)
- `qry` prefix = user-created business query (most valuable)
- `vew` prefix = view-like query
- `~sq_` prefix = form-embedded subquery (less useful, skip unless specifically investigating forms)
- Queries are in `metadata/access/queries.csv` with reconstructed SQL

## Table Tier Reference
- **Tier 1** (11 tables): Core operational tables. ALREADY DOCUMENTED in `knowledge/semantic_model.md`. Do not re-document these — reference them.
- **Tier 2** (36 tables): Important supporting tables. HIGH PRIORITY for documentation.
- **Tier 3** (62 tables): Reference/lookup tables. MEDIUM PRIORITY.
- **Tier 4** (13 tables): Low-activity tables. LOW PRIORITY — document only if they reveal something interesting.
```

### 1b. verify.py

The immutable evaluation harness. Produces a scorecard showing coverage gaps. The agent runs this at the start of each iteration.

**File**: `context_compiler/verify.py`

```python
#!/usr/bin/env python3
"""DWS Context Compiler — Coverage Scorecard.

Immutable evaluation harness. Do not modify.
Measures how much of the DWS database is semantically documented.
"""
import json
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
KNOWLEDGE = ROOT / "knowledge"
CATALOG = ROOT / "catalog"


def count_table_coverage():
    """Count documented tables vs total."""
    annotations = json.load(open(CATALOG / "table_annotations.json"))
    total = len(annotations)

    # Tier 1 tables documented in semantic_model.md
    tier1_tables = {
        "tblProject", "tblTimesheet", "tblContact", "tblCompany",
        "tblBid", "tblBill", "tblBillHistory", "tblBillHistDetail",
        "tblPO", "tblPurchasing", "tblSheetNEW",
    }

    # Per-table files in knowledge/tables/
    tables_dir = KNOWLEDGE / "tables"
    per_table_docs = set()
    if tables_dir.exists():
        for f in tables_dir.glob("*.md"):
            per_table_docs.add(f.stem)

    documented = tier1_tables | per_table_docs

    # By tier
    tiers = {}
    for name, info in annotations.items():
        tier = info["importance_tier"]
        tiers.setdefault(tier, {"total": 0, "documented": 0})
        tiers[tier]["total"] += 1
        if name in documented:
            tiers[tier]["documented"] += 1

    return {
        "total": total,
        "documented": len(documented & set(annotations.keys())),
        "pct": round(100 * len(documented & set(annotations.keys())) / total, 1),
        "by_tier": {
            t: {**v, "pct": round(100 * v["documented"] / v["total"], 1) if v["total"] else 0}
            for t, v in sorted(tiers.items())
        },
        "undocumented_tier2": sorted([
            name for name, info in annotations.items()
            if info["importance_tier"] == 2 and name not in documented
        ]),
    }


def count_query_coverage():
    """Count analyzed Access query groups."""
    analysis_dir = KNOWLEDGE / "access_analysis"
    analyzed_files = []
    if analysis_dir.exists():
        analyzed_files = [f.stem for f in analysis_dir.glob("*.md")]
    return {
        "total_user_queries": 702,
        "analysis_files": len(analyzed_files),
        "files": analyzed_files,
    }


def count_flow_coverage():
    """Count documented lifecycle flows."""
    target_flows = [
        "estimate_to_contract",
        "contract_to_engineering",
        "engineering_to_production",
        "production_to_field",
        "billing_and_closeout",
        "purchasing_flow",
    ]
    flows_dir = KNOWLEDGE / "flows"
    documented = []
    if flows_dir.exists():
        documented = [f.stem for f in flows_dir.glob("*.md")]
    return {
        "total": len(target_flows),
        "documented": len(set(documented) & set(target_flows)),
        "pct": round(100 * len(set(documented) & set(target_flows)) / len(target_flows), 1),
        "missing": sorted(set(target_flows) - set(documented)),
    }


def check_glossary():
    """Check if glossary exists."""
    return (KNOWLEDGE / "glossary.md").exists()


def check_index():
    """Check if INDEX.md exists."""
    return (KNOWLEDGE / "INDEX.md").exists()


def main():
    tables = count_table_coverage()
    queries = count_query_coverage()
    flows = count_flow_coverage()
    has_glossary = check_glossary()
    has_index = check_index()

    # Composite score: weighted average
    table_score = tables["pct"]
    query_score = min(100, queries["analysis_files"] * 12.5)  # 8 files = 100%
    flow_score = flows["pct"]
    glossary_score = 100 if has_glossary else 0
    index_score = 100 if has_index else 0

    composite = round(
        0.40 * table_score +
        0.25 * query_score +
        0.20 * flow_score +
        0.10 * glossary_score +
        0.05 * index_score,
        1
    )

    print("=" * 60)
    print("DWS CONTEXT COMPILER — SCORECARD")
    print("=" * 60)
    print()
    print(f"COMPOSITE SCORE: {composite}%")
    print()
    print(f"Tables:   {tables['documented']}/{tables['total']} ({tables['pct']}%)")
    for tier, info in tables["by_tier"].items():
        print(f"  Tier {tier}: {info['documented']}/{info['total']} ({info['pct']}%)")
    print()
    print(f"Queries:  {queries['analysis_files']}/8 analysis files")
    print(f"Flows:    {flows['documented']}/{flows['total']} ({flows['pct']}%)")
    print(f"Glossary: {'✓' if has_glossary else '✗'}")
    print(f"Index:    {'✓' if has_index else '✗'}")
    print()

    # Highlight biggest gaps
    print("BIGGEST GAPS:")
    if not has_index:
        print("  → No INDEX.md — create it first")
    if tables["undocumented_tier2"]:
        print(f"  → {len(tables['undocumented_tier2'])} Tier 2 tables undocumented: {', '.join(tables['undocumented_tier2'][:5])}...")
    if queries["analysis_files"] == 0:
        print("  → No Access query analysis files yet")
    if flows["missing"]:
        print(f"  → {len(flows['missing'])} lifecycle flows missing: {', '.join(flows['missing'][:3])}...")
    if not has_glossary:
        print("  → No column glossary")
    print()


if __name__ == "__main__":
    main()
```

### 1c. TODO.md

The shared task list. Seeded with ~40 tasks across 4 priority levels. Agents pick from the top, mark done, and add discoveries at the bottom.

**File**: `context_compiler/TODO.md`

Seed content:

```markdown
# DWS Context Compiler — Task List

Run `python3 context_compiler/verify.py` for current coverage scorecard.

---

## In Progress


## Ready

### P0: Foundation (do these first)
- [ ] Create `knowledge/INDEX.md` — master table of contents with coverage tracking
- [ ] Create `knowledge/access_analysis/overview.md` — categorize all 702 user-created Access queries by business function (estimating, billing, labor, purchasing, engineering, field, project mgmt, reporting, other). Just the inventory and counts, not deep analysis yet.
- [ ] Create `knowledge/glossary.md` — start with ID columns (ProjectID, ContactID, CompanyID, SheetID, etc.) documenting where each appears and what it means in each context

### P1: Access Query Analysis (highest leverage — institutional knowledge)
- [ ] Analyze estimating/bidding Access queries → `knowledge/access_analysis/estimating.md`. Search queries.csv for queries touching dbo_tblBid, dbo_tblProject with estimate-related names (qryBid*, qryEst*, qryAward*)
- [ ] Analyze billing/invoicing Access queries → `knowledge/access_analysis/billing.md`. Search for queries touching dbo_tblBill*, dbo_tblBillHist*
- [ ] Analyze labor/timesheet Access queries → `knowledge/access_analysis/labor.md`. Search for queries touching dbo_tblTimesheet, dbo_tblEmpWeek, with names like qryTime*, qryLabor*, qryHours*
- [ ] Analyze purchasing/PO Access queries → `knowledge/access_analysis/purchasing.md`. Search for queries touching dbo_tblPurchasing, dbo_tblPO, dbo_tblMaterials
- [ ] Analyze engineering/scheduling Access queries → `knowledge/access_analysis/engineering.md`. Search for queries touching dbo_tblSheetNEW, dbo_tblSheetProcess
- [ ] Analyze field work Access queries → `knowledge/access_analysis/field.md`. Search for queries touching dbo_tblFieldWorkOrder, dbo_tblShipTag*
- [ ] Analyze project management Access queries → `knowledge/access_analysis/project_management.md`. Search for queries touching dbo_tblProject with names like qryProj*, qryJob*, qryPM*
- [ ] Analyze remaining/cross-cutting Access queries → `knowledge/access_analysis/other.md`. Anything not covered above.

### P2: Tier 2 Table Semantics (36 tables — high value)
- [ ] Document tblSheetProcess (Engineering, 68,790 rows) → `knowledge/tables/tblSheetProcess.md`
- [ ] Document tblShipTagItems (Field, 64,994 rows) → `knowledge/tables/tblShipTagItems.md`
- [ ] Document tmpMas90InvHistDetail (Mas90 Integration, 60,014 rows) → `knowledge/tables/tmpMas90InvHistDetail.md`
- [ ] Document tblTimesheetEntry (Labor, 17,850 rows) → `knowledge/tables/tblTimesheetEntry.md`
- [ ] Document tblShipTag (Field, 15,314 rows) → `knowledge/tables/tblShipTag.md`
- [ ] Document tblTransDrawings (Transmittals & RFIs, 14,338 rows) → `knowledge/tables/tblTransDrawings.md`
- [ ] Document tblCalendar (Labor, 8,057 rows) → `knowledge/tables/tblCalendar.md`
- [ ] Document tblTransmittal (Transmittals & RFIs, 7,246 rows) → `knowledge/tables/tblTransmittal.md`
- [ ] Document remaining Tier 2 tables (28 more — add individual tasks as you discover them)

### P3: Lifecycle Flows
- [ ] Document Estimate → Contract flow → `knowledge/flows/estimate_to_contract.md`
- [ ] Document Contract → Engineering flow → `knowledge/flows/contract_to_engineering.md`
- [ ] Document Engineering → Production flow → `knowledge/flows/engineering_to_production.md`
- [ ] Document Production → Field flow → `knowledge/flows/production_to_field.md`
- [ ] Document Billing & Closeout flow → `knowledge/flows/billing_and_closeout.md`
- [ ] Document Purchasing flow → `knowledge/flows/purchasing_flow.md`

### P4: Tier 3 Table Semantics (62 tables — batch by stage)
- [ ] Document Labor reference tables: tblCalMo, tblWeek, tblNoBillTimeCd, tblEmpWeek
- [ ] Document HR tables: tblNameChg, tblVacationRequest, tmpEmployeeMas90, tblNameChgCodes
- [ ] Document Engineering tables: tblSheetType, tblItemType, tblOtherItems, tblOtherItemOptions
- [ ] Document remaining Tier 3 tables (add individual tasks as you discover them)

### P5: Glossary Expansion
- [ ] Add date columns to glossary — map all date/datetime columns across tables
- [ ] Add code columns to glossary — ProcessCd, EarningCd, DeptCd, CompanyType, etc.
- [ ] Add status/flag columns to glossary — Archived, Certified, TM, InStock, etc.

## Done


## Discovered
(agents add new tasks here as they find them during investigation)

```

### 1d. run.sh

The ralph-wiggum bash loop.

**File**: `context_compiler/run.sh`

```bash
#!/usr/bin/env bash
#
# DWS Context Compiler — ralph wiggum style
#
# Usage:
#   ./context_compiler/run.sh                  # default: sonnet, 30 turns
#   COMPILER_MODEL=opus ./context_compiler/run.sh   # use opus
#   COMPILER_TURNS=50 ./context_compiler/run.sh     # more turns per iteration
#
set -euo pipefail
cd "$(dirname "$0")/.."

MODEL="${COMPILER_MODEL:-sonnet}"
TURNS="${COMPILER_TURNS:-30}"
SLEEP="${COMPILER_SLEEP:-5}"

echo "DWS Context Compiler"
echo "  model=$MODEL  max_turns=$TURNS  sleep=${SLEEP}s"
echo "  Press Ctrl+C to stop"
echo ""

ITER=0
while : ; do
    ITER=$((ITER + 1))
    echo "--- iteration $ITER | $(date '+%H:%M:%S') ---"

    claude -p "$(cat context_compiler/PROMPT.md)" \
        --model "$MODEL" \
        --max-turns "$TURNS" \
        --dangerously-skip-permissions \
        2>&1 | tail -5

    echo ""
    sleep "$SLEEP"
done
```

### 1e. Seed INDEX.md

**File**: `knowledge/INDEX.md`

Create the initial index with all entries marked as pending. The compiler updates this as it works.

---

## Phase 2: Run It

Once the harness files exist:

```bash
chmod +x context_compiler/run.sh
./context_compiler/run.sh
```

Monitor progress:
- `python3 context_compiler/verify.py` — check the scorecard
- `git log --oneline | head -20` — see what the compiler has committed
- Read `context_compiler/TODO.md` — see what's done and what's been discovered

### Expected iteration cadence
- **Sonnet**: ~3 min per iteration, ~20 iterations/hour
- **Opus**: ~8 min per iteration, ~7 iterations/hour
- **Full coverage estimate**: ~60-80 iterations for Tier 2 tables + query analysis + flows. At sonnet speed, ~3-4 hours unattended.

### When to stop
- `verify.py` composite score > 80%
- Or you're satisfied with the coverage
- Or you notice quality degrading (review the last few commits)

### Human checkpoints
After every ~20 iterations, review:
1. Quality of the last 5 knowledge files written
2. Any "Discovered" tasks in TODO.md that need triage
3. Whether the agent is going in circles or making progress

---

## Success Criteria

### Automated Verification
- [x] `context_compiler/PROMPT.md` exists and is <3,000 tokens
- [x] `context_compiler/verify.py` runs and produces scorecard
- [x] `context_compiler/TODO.md` exists with seeded tasks
- [x] `context_compiler/run.sh` is executable
- [x] `knowledge/INDEX.md` exists
- [ ] After 10 iterations: verify.py shows >15% table coverage (up from 9%)
- [ ] After 10 iterations: at least 1 access_analysis file exists
- [ ] After 10 iterations: git log shows 10+ `compiler:` commits

### Manual Verification
- [ ] Knowledge files follow the templates (business meaning, not just technical description)
- [ ] Access query analysis reveals actual business rules not obvious from schema
- [ ] TODO.md has "Discovered" tasks showing the compiler found new things to investigate
- [ ] No hallucinated column names or relationships (spot-check against profiles)

---

## Implementation Strategy

This is a single-phase build. All 5 files can be created in one session:

1. Create `context_compiler/` directory
2. Write `PROMPT.md` (from 1a above)
3. Write `verify.py` (from 1b above)
4. Write `TODO.md` (from 1c above)
5. Write `run.sh` (from 1d above)
6. Create `knowledge/INDEX.md` (from 1e above)
7. Create empty directories: `knowledge/tables/`, `knowledge/access_analysis/`, `knowledge/flows/`
8. Run `python3 context_compiler/verify.py` to verify baseline scorecard
9. Commit: `Add context compiler harness`
10. Run `./context_compiler/run.sh` and observe first iteration

**All files are specified in full above. No codebase exploration needed. Just write them.**

---

## References

- Ralph Wiggum: https://ghuntley.com/ralph/
- Autoresearch: https://github.com/karpathy/autoresearch
- OpenAI Data Agent: https://openai.com/index/inside-our-in-house-data-agent/
- Harness Engineering: https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents
- DWS Agent Harness v2 plan: `thoughts/shared/plans/2026-03-29-dws-agent-harness-v2.md`
- Existing dws-analyst skill: `.claude/skills/dws-analyst/SKILL.md`
