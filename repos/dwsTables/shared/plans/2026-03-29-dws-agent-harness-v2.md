# DWS Agent Harness v2 — Progressive Loading, Self-Learning, Hooks

## Overview

Rewrite the `/dws-analyst` skill from a naive "load everything, hope for the best" prompt into a proper agent harness with progressive context loading, deterministic learning capture, and bundled tooling. Inspired by Dash's 6-layer architecture, adapted to Claude Code's skill/hook system.

The core insight: the current skill loads ~13,600 tokens of knowledge before every query regardless of relevance, has no searchable learning mechanism, and uses zero Claude Code skill features. This plan fixes all three.

## Current State Analysis

### What exists
- **Knowledge base**: 3 markdown files (semantic_model 274L, join_map 223L, business_rules 141L) + 3 learnings
- **Skill**: `.claude/skills/dws-analyst/SKILL.md` — 76 lines, plain prompt, no frontmatter features
- **Helpers**: `scripts/eda_helpers.py` with `load_table()` reading raw CSVs, zero caching
- **Access queries**: 1,055 reconstructed SQL queries in `metadata/access/queries.csv` (untapped)

### What's wrong
1. **Slow**: Reads ALL knowledge (~13.6K tokens) before every query. `load_table()` re-reads CSVs from disk every call. No caching anywhere.
2. **Learnings don't work**: 3 flat files, no tags, no search. Instruction says "check learnings/" but there's no structured retrieval — agent either reads all or misses them.
3. **Missing Dash layers**: No validated query patterns (Layer 3), no Access query search (Layer 4), no runtime introspection (Layer 6). No hooks, no allowed-tools, no dynamic injection.

### Key Discoveries
- `scripts/eda_helpers.py:9-12` — `load_table()` uses `pd.read_csv()` with no caching, reads full file every call
- `.claude/skills/dws-analyst/SKILL.md:1-5` — frontmatter only has name, description, user_invocable. No allowed-tools, no hooks, no dynamic injection
- `knowledge/learnings/*.md` — no YAML frontmatter, no `tables:` tags, unsearchable
- `metadata/access/queries.csv` — 1,055 queries with columns: name, query_type, sql_reconstructed, table_count, field_count. Multi-line SQL strings. Rich source of business logic patterns.
- Existing knowledge files are well-written — the content is good, the delivery mechanism is bad

## Desired End State

After this plan:
- `/dws-analyst` loads a ~500-token table index at invocation, then fetches targeted context (~3-5K tokens for relevant tables only) on demand
- Learnings have YAML frontmatter with `tables:` tags and are searchable by table name via bundled script
- Agent can search 1,055 Access SQL queries for business logic inspiration before writing pandas
- `load_table()` transparently caches to parquet — first call writes cache, subsequent calls are 5-10x faster
- Skill-scoped PostToolUse hook detects query errors and injects learning prompt (silent on success)
- `allowed-tools` eliminates permission dialog stalls

### Verification
- Invoke `/dws-analyst` with 5 test questions
- Agent does NOT read all 3 knowledge files before answering
- Agent calls `context_loader.py` and `search_learnings.py` with specific table names
- Query execution uses parquet cache (visible in Bash output)
- On a failed query, hook injects learning prompt without being asked
- At least one learning is saved with proper frontmatter

## What We're NOT Doing

- Vector DB / embeddings for learning search (grep + frontmatter tags is sufficient at this scale)
- Translating all 1,055 Access queries to pandas (agent translates on demand)
- Subagent architecture for query execution (deferred — inline execution lets the agent iterate)
- `context: fork` on the skill (analyst needs inline iteration, not condensed responses)
- MCP server for anything (CLI scripts are more context-efficient per the harness engineering principles)
- Eval framework / automated grading (premature — build after we have enough query history)

## Implementation Approach

Four phases. Phases 1 and 2 are parallelizable internally (subagent-friendly). Phase 3 depends on Phase 2 scripts existing. Phase 4 is manual.

**Design principles** (from harness engineering research):
- **Deterministic > non-deterministic**: Scripts filter context, not the LLM
- **Silent success, verbose failure**: Hook produces no output on clean queries
- **Instruction budget**: Every token in context should earn its keep
- **Progressive disclosure**: Metadata always loaded, detail loaded on demand
- **Backpressure**: Don't dump 13K tokens and hope — inject what's relevant

---

## Phase 1: Infrastructure

### Overview
Three independent changes: parquet caching, learning frontmatter, query pattern seeds. **All three can be implemented in parallel subagents.**

### 1a. Transparent Parquet Caching

**File**: `scripts/eda_helpers.py`
**Changes**: Modify `load_table()` to check for a `.parquet` cache before reading CSV. Write cache on first load.

```python
def load_table(table_name: str) -> pd.DataFrame:
    """Load a table with transparent parquet caching."""
    csv_path = TABLES_DIR / f"{table_name}.csv"
    cache_path = TABLES_DIR / ".cache" / f"{table_name}.parquet"

    # Use cache if it exists and is newer than CSV
    if cache_path.exists() and cache_path.stat().st_mtime >= csv_path.stat().st_mtime:
        return pd.read_parquet(cache_path)

    # Read CSV, write cache, return
    df = pd.read_csv(csv_path, keep_default_na=False, na_values=[""], low_memory=False)
    cache_path.parent.mkdir(exist_ok=True)
    df.to_parquet(cache_path, index=False)
    return df
```

**Also**: Add `Tables/.cache/` to `.gitignore`.

**Why parquet**: tblTimesheet (289K rows) takes ~2s from CSV, ~0.2s from parquet. The analyst agent often loads 2-3 tables per query.

### 1b. Learning Frontmatter Migration

**Files**: All 3 files in `knowledge/learnings/` (excluding README.md)

Add YAML frontmatter with `tables` list to each existing learning. This makes them searchable by `search_learnings.py` (Phase 2b).

**Example** (`knowledge/learnings/2026-03-29-hrs-per-k-outliers.md`):
```markdown
---
tables: [tblProject, tblTimesheet]
tags: [outliers, metric, hours-per-k]
---
# Extreme outliers in hours-per-$1K metric for small projects
... (existing content unchanged)
```

**Do the same for**:
- `2026-03-29-purchasing-line-total-not-stored.md` — tables: [tblPurchasing]
- `2026-03-29-gc-company-duplicates.md` — tables: [tblProject, tblCompany]

**Also update** `knowledge/learnings/README.md` to document the frontmatter format:
```markdown
# Learnings

Data gotchas discovered by the DWS analyst agent during query sessions.

## Format

Each file has YAML frontmatter with:
- `tables`: list of table names this learning applies to
- `tags`: list of topic tags for keyword search

File naming: `YYYY-MM-DD-brief-description.md`
```

### 1c. Seed Query Patterns

**Directory**: `knowledge/queries/`

Create 5 validated pandas patterns from the existing EDA work. Each file is a self-contained pandas snippet with metadata frontmatter.

**Format per file**:
```markdown
---
name: revenue-by-client
tables: [tblProject, tblCompany]
tags: [revenue, client, concentration]
question: Who are the top clients by total project cost?
---
# Revenue by Client

​```python
import sys; sys.path.insert(0, 'scripts')
from eda_helpers import load_table

proj = load_table('tblProject')
co = load_table('tblCompany')

# Filter to projects with real cost
proj_valid = proj[proj['TotProjCost'] > 0]

# Join to get company names
merged = proj_valid.merge(co[['CompanyID', 'Company']], left_on='Client', right_on='CompanyID', how='inner')

# Aggregate
client_rev = (merged.groupby('Company')['TotProjCost']
              .agg(['sum', 'count'])
              .rename(columns={'sum': 'total_revenue', 'count': 'project_count'})
              .sort_values('total_revenue', ascending=False))

print(f"Total: ${client_rev['total_revenue'].sum():,.0f} across {len(client_rev)} clients")
print(f"\nTop 10:")
print(client_rev.head(10).to_string())
​```

## Notes
- Total revenue: $194.7M across 319 clients
- Top 3 = 55.6% (Skyline, Principal, BCCI)
- Filter TotProjCost > 0 to exclude placeholder projects
```

**Create these 5 patterns**:
1. `revenue-by-client.md` — tblProject + tblCompany
2. `labor-hours-by-project.md` — tblTimesheet + tblProject
3. `material-cost-by-project.md` — tblPurchasing + tblProject (using Price * Qty)
4. `gross-margin.md` — tblProject + tblTimesheet + tblPurchasing + tblBillHistory
5. `schedule-variance.md` — tblSheetNEW + tblProject (estimated vs actual dates)

**Important**: Each pattern should incorporate relevant learnings. For example, `material-cost-by-project.md` must use `Price * Qty` (not a stored `line_total` column). The `labor-hours-by-project.md` pattern should note the outlier issue for small projects.

### Success Criteria

#### Automated Verification:
- [x] `load_table('tblProject')` creates `Tables/.cache/tblProject.parquet` on first call
- [x] Second call to `load_table('tblProject')` reads from cache (verify with timing)
- [x] `Tables/.cache/` is in `.gitignore`
- [x] All 3 learnings have YAML frontmatter with `tables:` field
- [x] `knowledge/queries/` contains 5 `.md` files with valid frontmatter
- [x] All 5 query pattern code blocks execute without error

#### Manual Verification:
- [ ] Query patterns produce correct results matching EDA notebook outputs
- [ ] Learning frontmatter tags accurately reflect the tables involved

**Implementation note**: These 3 tasks (1a, 1b, 1c) are independent. Implement in parallel subagents. 1a touches `scripts/eda_helpers.py` + `.gitignore`. 1b touches `knowledge/learnings/*.md`. 1c creates new files in `knowledge/queries/`.

---

## Phase 2: Bundled Scripts

### Overview
Three Python scripts that live inside the skill directory. They are the deterministic layer — the agent calls them instead of reading/filtering files itself. **All three can be implemented in parallel subagents.**

### Key Context for Implementers

Before implementing, read these files:
- `scripts/eda_helpers.py` — existing helpers, import patterns
- `scripts/config.py` — path constants (ROOT_DIR, TABLES_DIR, etc.)
- `knowledge/semantic_model.md` — section headers start with `### tblTableName`
- `knowledge/join_map.md` — section headers start with `## tblTableName`
- `knowledge/business_rules.md` — 141 lines total, organized by topic not table

### 2a. `context_loader.py` — Targeted Context Retrieval

**File**: `.claude/skills/dws-analyst/scripts/context_loader.py`

Takes table names as arguments. Returns only the relevant sections from semantic_model.md, join_map.md, and all of business_rules.md (it's only 141 lines — always worth including).

**Usage**:
```bash
python3 .claude/skills/dws-analyst/scripts/context_loader.py tblProject tblTimesheet
```

**Output**: Markdown to stdout containing:
1. Relevant sections from `knowledge/semantic_model.md` (sections start with `### tblTableName`)
2. Relevant sections from `knowledge/join_map.md` (sections start with `## tblTableName`, plus always include "## Common Join Patterns")
3. Full contents of `knowledge/business_rules.md`

**Implementation approach**:
```python
#!/usr/bin/env python3
"""Extract targeted context for specific tables from knowledge files."""
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parents[4]  # .claude/skills/dws-analyst/scripts/ -> project root
KNOWLEDGE = ROOT / "knowledge"


def extract_sections(filepath: Path, table_names: set, header_prefix: str) -> str:
    """Extract sections matching table names from a markdown file.

    Sections are delimited by lines starting with header_prefix (e.g. '### ' or '## ').
    A section is included if any table name appears in its header line.
    """
    lines = filepath.read_text().splitlines()
    result = []
    capturing = False

    for line in lines:
        if line.startswith(header_prefix):
            # Check if this section header matches any requested table
            capturing = any(t in line for t in table_names)
            # Always include Common Join Patterns from join_map
            if "Common Join Patterns" in line:
                capturing = True
        if capturing:
            result.append(line)

    return "\n".join(result)


def main():
    if len(sys.argv) < 2:
        print("Usage: context_loader.py <table1> [table2] ...", file=sys.stderr)
        sys.exit(1)

    tables = set(sys.argv[1:])

    # Semantic model: sections start with "### tblName"
    sem = extract_sections(KNOWLEDGE / "semantic_model.md", tables, "### ")
    if sem:
        print("# Relevant Table Descriptions\n")
        print(sem)
        print()

    # Join map: sections start with "## tblName"
    jm = extract_sections(KNOWLEDGE / "join_map.md", tables, "## ")
    if jm:
        print("# Relevant Join Map\n")
        print(jm)
        print()

    # Business rules: always include in full (141 lines, always relevant)
    br = (KNOWLEDGE / "business_rules.md").read_text()
    print(br)


if __name__ == "__main__":
    main()
```

**Note on ROOT path**: The script lives at `.claude/skills/dws-analyst/scripts/context_loader.py`. That's 4 levels deep from project root. Verify with: `Path(__file__).resolve().parents[4]` should equal the project root containing `knowledge/`. Add a sanity check and fall back to searching upward for a directory containing `knowledge/` if the hard-coded depth is wrong.

### 2b. `search_learnings.py` — Learning Search

**File**: `.claude/skills/dws-analyst/scripts/search_learnings.py`

Searches learnings by table name or keyword. Returns matching learnings with full content.

**Usage**:
```bash
python3 .claude/skills/dws-analyst/scripts/search_learnings.py tblProject tblTimesheet
python3 .claude/skills/dws-analyst/scripts/search_learnings.py --keyword outlier
```

**Implementation approach**:
```python
#!/usr/bin/env python3
"""Search learnings by table name or keyword."""
import sys
import re
from pathlib import Path

ROOT = Path(__file__).resolve().parents[4]
LEARNINGS_DIR = ROOT / "knowledge" / "learnings"


def parse_frontmatter(filepath: Path) -> tuple[dict, str]:
    """Parse YAML frontmatter and return (metadata, body)."""
    text = filepath.read_text()
    if not text.startswith("---"):
        return {}, text

    # Find closing ---
    end = text.index("---", 3)
    fm_text = text[3:end].strip()
    body = text[end + 3:].strip()

    # Simple YAML parsing for our known fields
    meta = {}
    for line in fm_text.splitlines():
        if ":" in line:
            key, val = line.split(":", 1)
            key = key.strip()
            val = val.strip()
            if val.startswith("[") and val.endswith("]"):
                meta[key] = [v.strip().strip("'\"") for v in val[1:-1].split(",")]
            else:
                meta[key] = val
    return meta, body


def main():
    args = sys.argv[1:]
    if not args:
        print("Usage: search_learnings.py <table1> [table2] ... [--keyword <kw>]", file=sys.stderr)
        sys.exit(1)

    keyword = None
    tables = []
    i = 0
    while i < len(args):
        if args[i] == "--keyword" and i + 1 < len(args):
            keyword = args[i + 1].lower()
            i += 2
        else:
            tables.append(args[i])
            i += 1

    tables_set = set(tables)
    matches = []

    for f in sorted(LEARNINGS_DIR.glob("*.md")):
        if f.name == "README.md":
            continue
        meta, body = parse_frontmatter(f)
        file_tables = set(meta.get("tables", []))
        file_tags = [t.lower() for t in meta.get("tags", [])]

        # Match by table name overlap
        if tables_set and tables_set & file_tables:
            matches.append((f.name, body))
            continue

        # Match by keyword in tags or body
        if keyword and (keyword in " ".join(file_tags) or keyword in body.lower()):
            matches.append((f.name, body))

    if not matches:
        print("No relevant learnings found.")
    else:
        print(f"Found {len(matches)} relevant learning(s):\n")
        for name, body in matches:
            print(f"--- {name} ---")
            print(body)
            print()


if __name__ == "__main__":
    main()
```

### 2c. `search_access_queries.py` — Institutional Knowledge Search

**File**: `.claude/skills/dws-analyst/scripts/search_access_queries.py`

Searches the 1,055 Access SQL queries by table name or keyword. Returns matching query names and their reconstructed SQL. This is the "institutional knowledge" layer — the agent reads SQL to understand business logic intent, then writes the pandas equivalent.

**Usage**:
```bash
python3 .claude/skills/dws-analyst/scripts/search_access_queries.py tblProject tblBid
python3 .claude/skills/dws-analyst/scripts/search_access_queries.py --keyword "margin"
```

**Implementation approach**:
```python
#!/usr/bin/env python3
"""Search Access DB queries for business logic patterns."""
import csv
import sys
from pathlib import Path

ROOT = Path(__file__).resolve().parents[4]
QUERIES_CSV = ROOT / "metadata" / "access" / "queries.csv"


def main():
    args = sys.argv[1:]
    if not args:
        print("Usage: search_access_queries.py <table_or_keyword> [...]", file=sys.stderr)
        print("  Tables: searches SQL for dbo_<table> references", file=sys.stderr)
        print("  --keyword <kw>: searches query names and SQL text", file=sys.stderr)
        sys.exit(1)

    keyword = None
    tables = []
    i = 0
    while i < len(args):
        if args[i] == "--keyword" and i + 1 < len(args):
            keyword = args[i + 1].lower()
            i += 2
        else:
            tables.append(args[i])
            i += 1

    # Build search patterns: Access queries use dbo_ prefix for linked SQL Server tables
    table_patterns = [f"dbo_{t}" for t in tables]

    matches = []
    with open(QUERIES_CSV, encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            name = row["name"]
            sql = row["sql_reconstructed"]

            # Skip embedded subform queries (not user-created)
            if name.startswith("~sq_"):
                continue

            # Match by table reference in SQL
            if table_patterns and any(p.lower() in sql.lower() for p in table_patterns):
                matches.append(row)
                continue

            # Match by keyword in name or SQL
            if keyword and (keyword in name.lower() or keyword in sql.lower()):
                matches.append(row)

    # Limit output to top 15 most relevant (by table count — more tables = more complex/interesting)
    matches.sort(key=lambda r: int(r.get("table_count", 0)), reverse=True)
    matches = matches[:15]

    if not matches:
        print("No matching Access queries found.")
    else:
        print(f"Found {len(matches)} relevant Access queries (showing top 15):\n")
        for row in matches:
            print(f"### {row['name']} ({row['table_count']} tables, {row['field_count']} fields)")
            print(f"```sql\n{row['sql_reconstructed']}\n```\n")


if __name__ == "__main__":
    main()
```

**Important**: The queries.csv uses `dbo_` prefix for SQL Server linked tables (e.g., `dbo_tblProject`). The search script must prepend `dbo_` to table names when searching SQL text. Also skip `~sq_` prefix queries (embedded subform queries, not user-created business logic).

### 2d. `table_index.py` — Compact Index for Dynamic Injection

**File**: `.claude/skills/dws-analyst/scripts/table_index.py`

Generates a compact table index (~500 tokens) for injection at skill load time via `!`backtick.

**Usage**:
```bash
python3 .claude/skills/dws-analyst/scripts/table_index.py
```

**Output**: A compact markdown table like:
```
# DWS Table Index (11 Tier 1 Tables)
| Table | Rows | Description | Key Joins |
|---|---|---|---|
| tblProject | 1,809 | Core project record — costs, dates, assignments | PM→Contact, Client→Company |
| tblTimesheet | 288,977 | Labor time entries per employee per day | Employee→Contact, Project→Project |
| tblContact | 5,066 | People — employees, clients, vendors, architects | CompanyID→Company |
| tblCompany | 1,963 | Companies — clients, vendors, GCs, architects | — |
| tblBid | 4,243 | Bids/estimates, awarded link to Project | JobNo→Project, Estimator→Contact |
| tblBill | 3,770 | Billing process lines per project | ProjectID→Project |
| tblBillHistory | 11,618 | Individual invoices sent to clients | ProjectID→Project |
| tblBillHistDetail | 19,947 | Invoice line items | BillHistID→BillHistory, BillID→Bill |
| tblPO | 14,687 | Purchase orders to vendors | Project→Project, POTo→Company |
| tblPurchasing | 31,416 | PO line items (Price * Qty for cost) | ProjectID→Project, Vendor→Company |
| tblSheetNEW | 15,932 | Engineering sheets/work items in pipeline | ProjectID→Project |

122 CSV tables in Tables/. 11 above are Tier 1 (most queried). Use context_loader.py for detail.
```

**Implementation**: Parse the first few lines of `knowledge/semantic_model.md` to extract table name, row count, and description for each `### tblName` section header. Hard-code the key joins from the section content. Keep it dead simple — this is a static index.

### Success Criteria

#### Automated Verification:
- [x] `python3 .claude/skills/dws-analyst/scripts/context_loader.py tblProject tblTimesheet` returns semantic model + join map sections for just those 2 tables + full business rules
- [x] `python3 .claude/skills/dws-analyst/scripts/context_loader.py tblProject` does NOT include tblTimesheet sections
- [x] `python3 .claude/skills/dws-analyst/scripts/search_learnings.py tblPurchasing` returns the "line_total not stored" learning
- [x] `python3 .claude/skills/dws-analyst/scripts/search_learnings.py tblBid` returns "No relevant learnings found."
- [x] `python3 .claude/skills/dws-analyst/scripts/search_access_queries.py tblProject tblBid` returns queries containing `dbo_tblProject` and `dbo_tblBid`
- [x] `python3 .claude/skills/dws-analyst/scripts/search_access_queries.py --keyword "margin"` — no margin queries in Access DB (valid result)
- [x] `python3 .claude/skills/dws-analyst/scripts/table_index.py` outputs a compact markdown table with all 11 Tier 1 tables

#### Manual Verification:
- [ ] context_loader output is concise — only requested tables, not the full files
- [ ] Access query search results are useful for understanding business logic

**Implementation note**: All 4 scripts are independent. Implement in parallel subagents. Each script only needs to read the files listed in its description — no codebase exploration needed.

---

## Phase 3: Skill Rewrite

### Overview
Replace the current SKILL.md with a new version that uses dynamic injection, progressive loading, hooks, and bundled scripts. Also create the hook script.

### Key Context for Implementers

Before implementing, read these files:
- Current skill: `.claude/skills/dws-analyst/SKILL.md` (76 lines)
- Phase 2 scripts: all 4 `.claude/skills/dws-analyst/scripts/*.py` (must exist from Phase 2)
- `scripts/eda_helpers.py` — the `load_table()` function and `TIER_1_TABLES` constant

### 3a. Hook Script — `check_query_output.py`

**File**: `.claude/skills/dws-analyst/scripts/check_query_output.py`

A PostToolUse hook that checks Bash output for Python errors or data quality warnings. Silent on success (no output, exit 0). On error patterns, outputs a learning prompt (stdout, exit 0).

**Receives on stdin** (from Claude Code hook system):
```json
{
  "session_id": "...",
  "tool_name": "Bash",
  "tool_input": {"command": "python3 ..."},
  "tool_output": "..."
}
```

**Implementation**:
```python
#!/usr/bin/env python3
"""PostToolUse hook: detect query errors, prompt learning capture."""
import json
import sys
import re

ERROR_PATTERNS = [
    r"Traceback \(most recent call last\)",
    r"KeyError:",
    r"ValueError:",
    r"TypeError:",
    r"MergeError:",
    r"ParserError:",
    r"FileNotFoundError:",
    r"Empty DataFrame",
    r"0 rows",
]

WARNING_PATTERNS = [
    r"SettingWithCopyWarning",
    r"FutureWarning",
    r"DtypeWarning",
]


def main():
    try:
        data = json.load(sys.stdin)
    except (json.JSONDecodeError, EOFError):
        sys.exit(0)

    command = data.get("tool_input", {}).get("command", "")
    output = data.get("tool_output", "")

    # Only check python3 commands
    if "python3" not in command and "python" not in command:
        sys.exit(0)

    # Check for error patterns
    errors = [p for p in ERROR_PATTERNS if re.search(p, output)]
    warnings = [p for p in WARNING_PATTERNS if re.search(p, output)]

    if errors:
        print("LEARNING PROMPT: The query above produced an error. "
              "After diagnosing and fixing, save a learning to "
              "knowledge/learnings/ with YAML frontmatter (tables: [...], tags: [...]) "
              "so this mistake is never repeated.")
    elif warnings:
        print("NOTE: Query produced warnings. Consider whether this indicates "
              "a data quality issue worth saving as a learning.")
    # else: silent success — no output

    sys.exit(0)


if __name__ == "__main__":
    main()
```

### 3b. New SKILL.md

**File**: `.claude/skills/dws-analyst/SKILL.md`

Replace the current file entirely:

```markdown
---
name: dws-analyst
description: Answers business questions about DWS millwork operations by querying 128-table database via pandas. Loads targeted context, searches institutional knowledge, captures learnings.
user-invocable: true
allowed-tools: Bash(python3 *), Read, Grep, Glob, Write
hooks:
  PostToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "python3 ${CLAUDE_SKILL_DIR}/scripts/check_query_output.py"
---

# DWS Data Analyst

You are a data analyst for Design Workshops (DWS), a commercial millwork shop in Oakland, CA with 20+ years of data across 128 tables (122 CSVs in `Tables/`).

## Table Index

!`python3 ${CLAUDE_SKILL_DIR}/scripts/table_index.py`

## Your Workflow

### Step 1: Identify tables needed
Read the user's question. Identify which Tier 1 tables are relevant from the index above.

### Step 2: Load targeted context
Run these scripts with only the tables you need:

```bash
# Get table descriptions, join map, and business rules for relevant tables
python3 ${CLAUDE_SKILL_DIR}/scripts/context_loader.py <table1> <table2> ...

# Check if any learnings exist for these tables
python3 ${CLAUDE_SKILL_DIR}/scripts/search_learnings.py <table1> <table2> ...
```

### Step 3: Check for existing patterns
Before writing new code, check if a validated pattern exists:

```bash
# Check validated pandas patterns
ls knowledge/queries/

# Search Access SQL queries for business logic inspiration
python3 ${CLAUDE_SKILL_DIR}/scripts/search_access_queries.py <table1> <table2> ...
# or by keyword:
python3 ${CLAUDE_SKILL_DIR}/scripts/search_access_queries.py --keyword "margin"
```

Access queries are SQL (not pandas) but show how DWS staff actually use the data — use them for join logic and business intent.

### Step 4: Write and execute query
Use pandas via Bash. Always use the project's loader:

```python
import sys; sys.path.insert(0, 'scripts')
from eda_helpers import load_table

df = load_table('tblProject')  # cached to parquet after first load
```

**Query rules**:
- Inspect dtypes and null counts before analysis
- Follow join types from context_loader output (LEFT vs INNER)
- Limit display output to top 20 rows
- Compute `Price * Qty` for tblPurchasing cost (no stored line_total column)
- Use median (not mean) for ratio metrics with small-project outliers

### Step 5: Interpret results
Don't just show a dataframe. Explain what the numbers mean for DWS's business.

Bad: "Skyline: $56.2M"
Good: "Skyline Construction is DWS's largest client at $56.2M (29% of total revenue), nearly double the next largest (Principal at $26.1M). This concentration represents significant key-account risk."

### Step 6: Save learnings (when warranted)
If you discovered a data gotcha, type mismatch, or non-obvious join pattern — save it:

```bash
cat > knowledge/learnings/$(date +%Y-%m-%d)-brief-description.md << 'LEARNING'
---
tables: [tblTableName1, tblTableName2]
tags: [relevant, topic, tags]
---
# Brief title

**Discovered**: YYYY-MM-DD
**Tables**: affected tables
**Issue**: what went wrong or was unexpected
**Solution**: how to handle it correctly
LEARNING
```

Only save if genuinely novel — check existing learnings first. The PostToolUse hook will remind you if a query errors out.

## Available EDA Context
For deeper analysis beyond the knowledge base, read the relevant EDA output:
- `eda/business_metrics.json` — revenue, labor, material, estimation
- `eda/deep_dives.json` — corrected metrics with full methodology
- `eda/quality_audit.json` — referential integrity audit
- `eda/estimation_variance.json` — schedule variance by PM/estimator
```

### 3c. Verify Skill Registration

After writing the new SKILL.md, verify:
1. The skill appears in Claude Code's available slash commands
2. The `!`backtick dynamic injection executes `table_index.py` at load time
3. The hook is registered (visible in skill metadata)

### Success Criteria

#### Automated Verification:
- [x] `.claude/skills/dws-analyst/SKILL.md` has valid frontmatter with `allowed-tools` and `hooks`
- [x] `.claude/skills/dws-analyst/scripts/check_query_output.py` exists and is executable
- [x] The hook script returns exit 0 with no output for clean Python output
- [x] The hook script returns exit 0 with learning prompt for Python tracebacks
- [ ] `table_index.py` output appears when skill is invoked (test with `/dws-analyst test`)

#### Manual Verification:
- [ ] `/dws-analyst` is invocable and shows the table index inline
- [ ] Agent calls context_loader with specific tables, not reading full knowledge files
- [ ] Hook fires after a Bash(python3) call and injects learning prompt on error

**Implementation note**: Phase 3 depends on Phase 2 scripts existing. Implement 3a and 3b together (they're closely coupled). Read the Phase 2 scripts to understand their interfaces before writing the skill instructions.

---

## Phase 4: Smoke Test + Iterate

### Overview
Run 5 test questions. Measure context efficiency, learning capture, and answer quality. This is manual — run each question, observe behavior, iterate on skill/scripts.

### Test Questions

1. **Revenue concentration**: "Who are DWS's top 10 clients by total project cost?"
   - **Expected behavior**: Agent identifies tblProject + tblCompany → calls context_loader with those tables → checks query patterns → finds `revenue-by-client.md` → adapts or uses directly
   - **Context loaded**: ~3K tokens (2 table descriptions + 2 join map sections + business rules)

2. **Labor productivity**: "What's the average labor hours per $1K of contract value by project size?"
   - **Expected behavior**: Agent calls search_learnings for tblProject + tblTimesheet → finds hrs-per-k outlier learning → uses median instead of mean
   - **Learning integration test**: Does the agent actually apply the outlier learning?

3. **Material analysis**: "What are the top 10 vendors by total purchasing volume?"
   - **Expected behavior**: Agent calls context_loader for tblPurchasing + tblCompany → computes Price * Qty (not a stored column, per learning)
   - **Learning integration test**: Does the agent compute line total correctly?

4. **Cross-domain**: "What's the gross margin distribution for projects over $500K?"
   - **Expected behavior**: Agent needs 4+ tables → calls context_loader with all of them → checks query patterns → finds `gross-margin.md`
   - **Context scaling test**: Does targeted loading still work with 4 tables?

5. **Exploration**: "What are the most common Access queries involving tblBid? What business questions do they answer?"
   - **Expected behavior**: Agent calls search_access_queries for tblBid → reads SQL patterns → interprets business intent
   - **Institutional knowledge test**: Does the agent use Access queries as a reference layer?

### What to Measure

For each question, observe:
- [ ] Agent calls context_loader (not reading full knowledge files)
- [ ] Agent calls search_learnings (not reading all learning files)
- [ ] Agent checks query patterns before writing new code
- [ ] Total context consumed before first query execution
- [ ] Query executes successfully on first attempt
- [ ] Results are interpreted with business context (not just raw numbers)
- [ ] Hook fires correctly on any error (silent on success)

### Iteration Process
After running all 5 questions:
1. Note any failures or unexpected behavior
2. Update scripts or skill instructions as needed
3. If a new data gotcha is discovered, verify it gets saved with proper frontmatter
4. If context_loader returns too much or too little, adjust section extraction logic

### Success Criteria

#### Automated Verification:
- [x] All 5 test queries execute without Python errors
- [ ] At least one new learning is saved with proper YAML frontmatter during testing
- [ ] search_learnings returns the new learning when queried by table name

#### Manual Verification:
- [ ] Agent uses targeted context loading (not reading full knowledge files)
- [ ] Agent applies existing learnings when relevant (e.g., median for hrs/$1K, Price * Qty for material cost)
- [ ] Results are factually correct (cross-check against EDA notebook outputs)
- [ ] Insights are interpreted contextually, not just raw numbers
- [ ] Access query search provides useful business logic context

---

## Implementation Strategy for Agents

This plan is designed for parallel implementation. Here's the recommended agent dispatch:

### Dispatch 1 (Phase 1 — 3 parallel subagents):
- **Agent 1a**: Modify `scripts/eda_helpers.py` for parquet caching + update `.gitignore`
- **Agent 1b**: Add YAML frontmatter to 3 learning files + update README
- **Agent 1c**: Create 5 query pattern files in `knowledge/queries/`

### Dispatch 2 (Phase 2 — 4 parallel subagents):
- **Agent 2a**: Create `context_loader.py`
- **Agent 2b**: Create `search_learnings.py`
- **Agent 2c**: Create `search_access_queries.py`
- **Agent 2d**: Create `table_index.py`

### Dispatch 3 (Phase 3 — sequential, 1 agent):
- Create hook script, rewrite SKILL.md, verify registration
- Must read Phase 2 scripts first to understand interfaces

### Dispatch 4 (Phase 4 — manual):
- Run smoke tests interactively

**Each agent only needs to read the files listed in its section.** The plan includes enough code specifics that agents should not need to explore the codebase beyond the listed files.

---

## Architecture: Before vs After

### Before (current)
```
User question
  → Agent reads semantic_model.md (274 lines, 5.5K tokens)
  → Agent reads join_map.md (223 lines, 4.5K tokens)
  → Agent reads business_rules.md (141 lines, 2.8K tokens)
  → Agent reads all learnings (38 lines, 700 tokens)
  = ~13,600 tokens loaded before any work
  → Agent writes pandas query
  → Agent hopes it remembers to save learnings
```

### After (this plan)
```
User question
  → table_index.py injects compact index (500 tokens, via !backtick at load time)
  → Agent identifies relevant tables from question
  → context_loader.py returns targeted sections (2-4K tokens for 2-3 tables)
  → search_learnings.py returns relevant learnings only (0-500 tokens)
  → Agent checks knowledge/queries/ for existing patterns
  → Agent optionally searches Access queries for business logic
  → Agent writes pandas query (with parquet-cached data)
  → PostToolUse hook detects errors → injects learning prompt
  → Agent saves learning with proper frontmatter tags
  = ~3-5K tokens of targeted, high-relevance context
```

**Context reduction**: ~13.6K → ~3-5K tokens (60-75% reduction), with higher relevance per token.

---

## Future Work (out of scope)

- **Subagent query runner**: Extract query execution into a `context: fork` subagent for context isolation
- **Query pattern auto-save**: Hook or instruction to save successful query patterns automatically
- **Eval framework**: Automated grading of query accuracy against known-good answers
- **Live SQL via MCP**: Replace CSV/pandas with live SQL Server queries
- **Learning deduplication**: Script to merge/deduplicate learnings as they accumulate

## References

- Dash repo (inspiration): https://github.com/agno-agi/dash
- Harness engineering principles: https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents
- Context-efficient backpressure: https://www.humanlayer.dev/blog/context-efficient-backpressure
- Claude Code skills docs: https://code.claude.com/docs/en/skills
- Claude Code hooks docs: https://code.claude.com/docs/en/hooks
- Previous plan: `thoughts/shared/plans/2026-03-29-dws-agent-harness.md`
- Existing skill: `.claude/skills/dws-analyst/SKILL.md`
