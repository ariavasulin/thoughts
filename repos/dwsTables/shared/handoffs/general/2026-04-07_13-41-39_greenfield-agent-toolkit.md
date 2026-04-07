---
date: 2026-04-07T13:41:39-07:00
researcher: claude
git_commit: 8b54013162076b4713215bce35e1e845009d70e9
branch: main
repository: dwsTables
topic: "Greenfield DWS Agent Toolkit — Repo Cleanup and CLI Tools"
tags: [greenfield, cli-tools, harness-engineering, context-efficiency]
status: complete
last_updated: 2026-04-07
last_updated_by: claude
type: implementation_strategy
---

# Handoff: Greenfield DWS Agent Toolkit

## Task(s)

### 1. Repo cleanup — COMPLETED
Moved ~12,200 files of bloat from the active repo into `archive/`, leaving a clean PMDBA/ greenfield root. The repo went from thousands of active files to ~815.

### 2. CLI tool suite — COMPLETED (5 tools built and tested)
Built 5 context-efficient CLI search/query tools in `PMDBA/scripts/` that form a progressive disclosure pipeline for the DWS agent.

### 3. SKILL.md + CLAUDE.md rewrite — NOT STARTED
The tools are built and tested but not yet wired into the agent skill definition or the repo-level CLAUDE.md. This is the next step.

## Critical References
- `PMDBA/scripts/` — the 5 CLI tools (the deliverable)
- Articles that informed the design philosophy (read during session):
  - `~/Git/Brain/Current/Inbox/Context-Efficient Backpressure for Coding Agents | HumanLayer Blog.md`
  - `~/Git/Brain/Current/Inbox/Skill Issue Harness Engineering for Coding Agents | HumanLayer Blog (1).md`
  - `~/Git/Brain/Current/Inbox/Harness engineering leveraging Codex in an agent-first world | OpenAI.md`

## Recent changes

All changes are unstaged on `main`. Nothing committed yet.

### Moved to `archive/`:
- `DWS/` → `archive/DWS/app/` (React Vite UI, 10,855 files)
- `scripts/` → `archive/pipeline/scripts/` (18 pipeline scripts, job done)
- `context_compiler/` → `archive/pipeline/context_compiler/` (6 files)
- `metadata/` → `archive/pipeline/metadata/` (raw Access query CSV)
- `PMDBA/forms/` → `archive/pmdba_raw/forms/` (370 Access form defs)
- `PMDBA/reports/` → `archive/pmdba_raw/reports/` (48 report defs)
- `PMDBA/queries/` was initially archived, then **restored** (user wants raw SQL)
- `PMDBA/catalog/` → `archive/pmdba_raw/catalog/` (123 JSON profiles)
- `PMDBA/knowledge/tables/` → `archive/pmdba_raw/tables/` (20 per-table docs — user decided SQLite + access_analysis covers this)
- `PMDBA/knowledge/data_dictionary.md` → `archive/pmdba_raw/` (auto-generated, SQLite answers this)
- `PMDBA/knowledge/entity_graph.md` → `archive/pmdba_raw/` (auto-generated)
- `PMDBA/knowledge/INDEX.md` → `archive/pmdba_raw/` (stale compiler tracking)
- Removed: `PMDBA/knowledge/queries/`, `learnings/`, `flows/` (bloat)
- `AGENTS.md` → `archive/AGENTS.md.old`

### Created:
- `PMDBA/scripts/run_sql.py` — Execute SQL against SQLite, markdown table output
- `PMDBA/scripts/search_queries.py` — Search 796 analyzed queries, returns: name, purpose, raw SQL
- `PMDBA/scripts/get_table_info.py` — SQLite schema + row count + sample rows; `--list` for full inventory
- `PMDBA/scripts/get_query_info.py` — Full analyzed writeup for named queries from access_analysis
- `PMDBA/scripts/get_query_full.py` — Raw SQL from `PMDBA/queries/*.sql`

## Learnings

### Design philosophy (from articles + user feedback)
- **Progressive disclosure is key**: search → summary → full detail → execute. Don't dump context.
- **Sub-agents are context firewalls**: isolate research from execution to stay in the ~75k "smart zone".
- **Silent success, verbose failure**: tools should produce minimal output for the common case.
- **CLI tools > MCP servers**: lighter, composable, already in training data. Agent uses them via Bash.
- **Query-centric > table-centric**: The user explicitly said "thinking from the query perspective is more valuable." The Access queries encode 20+ years of business logic. Table shapes can be derived on-the-fly from SQLite.

### Tool design decisions
- `search_queries.py` output was iterated: started with name+tables+purpose+SQL, then dropped `Tables:` line because the raw SQL already shows the tables. Final format: name, purpose, SQL.
- The `tables/` knowledge docs were archived because they're incomplete (20/128) and long. SQLite `PRAGMA table_info` + sample rows covers schema; `access_analysis/` covers business meaning.
- `search_queries.py` still uses the scoring system from the old `search_knowledge.py` (term frequency with caps). Works well for relevance ranking.

### Repo structure
- `PMDBA/knowledge/access_analysis/` is 16 markdown files, each covering a business category (billing, labor, engineering, etc.) with `### queryName` sections. Each section has `**Tables**:`, `**Purpose**:`, `**Key logic**:`, `**Business rules**:` fields. 796 queries total.
- `PMDBA/queries/` has 798 raw `.sql` files (2 extra — likely duplicates or renamed). Filenames match query names.
- SQLite at `Tables/.cache/dws.sqlite` has 130 entries (128 tables + `dbo_` views + `__import_manifest`). 77MB.
- The old skill scripts in `.agents/skills/dws-analyst/scripts/` and `.claude/skills/dws-analyst/scripts/` are now stale — they reference paths that no longer exist (`knowledge/learnings/`, `knowledge/tables/`). The new tools in `PMDBA/scripts/` replace them entirely.

## Artifacts

- `PMDBA/scripts/run_sql.py` — new
- `PMDBA/scripts/search_queries.py` — new
- `PMDBA/scripts/get_table_info.py` — new
- `PMDBA/scripts/get_query_info.py` — new
- `PMDBA/scripts/get_query_full.py` — new

## Action Items & Next Steps

1. **Rewrite `.agents/skills/dws-analyst/SKILL.md`** — Wire up the 5 new tools in PMDBA/scripts/. Replace old script references. Keep it under 60 lines (per harness engineering best practices). Include `!python3 ... get_table_info.py --list` as the on-start orientation (lightweight table inventory).

2. **Sync `.claude/skills/dws-analyst/`** — Delete old scripts, point to PMDBA/scripts/ or copy new ones in. Eliminate the two-copy divergence problem.

3. **Rewrite `CLAUDE.md`** — Update to reflect the cleaned repo structure. Remove references to old paths. Keep it map-style (pointers, not content).

4. **Consider a `check_query_output.py` hook** — The old skill had a PostToolUse hook that prompted the agent to save learnings on error. Could be worth keeping in some form.

5. **Consider adding `knowledge/semantic_model.md`, `glossary.md`, `join_map.md` as read-on-demand references** — These are compact (229-273 lines each) and contain cross-table patterns the agent can't derive from SQLite alone. The SKILL.md could mention them as "read these if you need cross-table context."

6. **Commit the changes** — Everything is unstaged on main.

## Other Notes

### Current PMDBA/ layout
```
PMDBA/
├── knowledge/
│   ├── access_analysis/   16 category files (796 analyzed queries)
│   ├── semantic_model.md  Tier 1 table reference (273 lines)
│   ├── glossary.md        Column naming patterns (229 lines)
│   └── join_map.md        Stable key joins (222 lines)
├── queries/               798 raw .sql files
└── scripts/               5 CLI tools
```

### Tool usage flow
```
search_queries.py   →  discover queries (compact: name, purpose, SQL)
get_query_info.py   →  deep dive (full business logic writeup)
get_query_full.py   →  just the raw SQL
get_table_info.py   →  schema + sample rows from SQLite
run_sql.py          →  execute SQL
```

### Archive organization
```
archive/
├── DWS/app/               React UI
├── pipeline/
│   ├── scripts/           Data extraction scripts
│   ├── context_compiler/  Knowledge builder loop
│   └── metadata/          Raw Access query CSV
├── pmdba_raw/
│   ├── catalog/           JSON profiles
│   ├── forms/             Access form definitions
│   ├── reports/           Access report definitions
│   ├── tables/            Per-table knowledge docs
│   ├── queries/           (empty — restored to PMDBA/)
│   ├── Test.md            Old skill manifest draft
│   ├── data_dictionary.md Auto-generated
│   ├── entity_graph.md    Auto-generated
│   └── INDEX.md           Stale compiler tracking
└── AGENTS.md.old
```
