---
date: 2026-03-21T18:00:00-07:00
researcher: ARI
git_commit: (no commits yet - initial repo)
branch: main
repository: dwsTables
topic: "Codebase research for CLAUDE.md creation"
tags: [research, codebase, claude-md, documentation]
status: complete
last_updated: 2026-03-21
last_updated_by: ARI
---

# Research: Codebase Research for CLAUDE.md Creation

**Date**: 2026-03-21T18:00:00-07:00
**Researcher**: ARI
**Git Commit**: (no commits yet)
**Branch**: main
**Repository**: dwsTables

## Research Question
Research this codebase and thoughts to create an excellent CLAUDE.md, applying best practices from the WriteClaudeMD.md reference.

## Summary

Comprehensive research of the dwsTables project to inform CLAUDE.md creation. The project is a data catalog/documentation system for a 128-table SQL Server database (DWSRestore) and its 1,055-query Access frontend (DWSJA.accdb). The workflow spans two platforms (Windows for SQL Server extraction, Mac for Access extraction and analysis) and produces a generated data dictionary with table profiles, relationship graphs, and business annotations.

## Detailed Findings

### Project Purpose (WHY)
- Document and catalog a legacy construction/engineering business management database
- The DWS database has no foreign keys, no extended properties, no stored procedures - all business logic lives in Access queries and forms
- The catalog provides what the database itself lacks: relationship mapping, column profiling, business categorization, and importance tiering

### Technology Stack (WHAT)
- **Python 3** with pandas, jupyter, matplotlib, seaborn, tqdm
- **PowerShell** for SQL Server metadata extraction (Windows-only)
- **Bash + mdbtools** for Access database parsing (Mac)
- **SQL Server**: SQL2016\SQLEXPRESS, database DWSRestore
- **Access**: DWSJA.accdb (123 MB, 1,055 saved queries)

### Workflow (HOW)
Two-phase pipeline:
1. **Phase 1** (metadata extraction): PowerShell on Windows + Bash/mdbtools on Mac
2. **Phase 2** (catalog generation): 5 Python scripts run via `scripts/run_phase2.sh`

### Directory Structure
```
Tables/               — 122 CSV data exports from SQL Server
metadata/sql_server/  — 9 CSV files from extract_metadata.ps1
metadata/access/      — 7 files from extract_access_metadata.sh + parse_access_queries.py
catalog/profiles/     — 118 JSON per-table profiles
catalog/              — relationships.json, entity_graph.md, table_annotations.json, data_dictionary.md
scripts/              — 12 files: config.py, extraction scripts, analysis pipeline
notebooks/            — 1 Jupyter validation notebook
```

### Scripts Pipeline
1. `extract_metadata.ps1` → metadata/sql_server/ (Windows, Invoke-Sqlcmd)
2. `extract_access_metadata.sh` → metadata/access/ (Mac, mdbtools)
3. `parse_access_queries.py` → metadata/access/queries.csv (reconstructs SQL from MSysQueries attributes)
4. `profile_tables.py` → catalog/profiles/*.json (column stats using SQL types as ground truth)
5. `extract_relationships.py` → catalog/relationships.json (341 relationships from views, queries, column names)
6. `generate_entity_graph.py` → catalog/entity_graph.md (relationship graph sorted by connection count)
7. `annotate_tables.py` → catalog/table_annotations.json (lifecycle stages, importance tiers)
8. `assemble_catalog.py` → catalog/data_dictionary.md (final markdown data dictionary)

### Key Design Patterns
- **Multi-source relationship inference**: High confidence from SQL JOINs, medium from column name matching
- **Type-aware profiling**: Uses SQL Server metadata for types (not inferred from CSV values)
- **Lifecycle-based organization**: 13 manually-mapped business stages
- **Importance scoring**: Row count + query references + relationship count → 4-tier system
- **Incremental JSON intermediates**: Each script writes JSON, final assembler reads all to produce markdown

### Notable Technical Details
- MSysObjects uses latin-1 encoding (not UTF-8)
- mdb-queries -q fails for .accdb files
- mdb-queries -L outputs space-separated (not newline-separated)
- No declarative FKs in SQL Server or Access
- Access `dbo_` prefix = linked SQL Server tables; `~sq_` = embedded form queries

### CLAUDE.md Design Principles Applied
Following the WriteClaudeMD.md reference:
- **Less is more**: ~70 lines, focused on universally applicable info
- **WHY/WHAT/HOW**: Covers project purpose, structure, and workflow
- **Progressive disclosure**: Points to specific files (config.py:23-89, annotate_tables.py:136-172) rather than copying content
- **No linter instructions**: No style guides or formatting rules
- **Pointers not copies**: References line numbers instead of duplicating code
- **Gotchas section**: Captures hard-won learnings that prevent repeated mistakes

## Code References
- `scripts/config.py` - Shared paths, skip lists, known empty tables
- `scripts/run_phase2.sh` - Phase 2 orchestrator (5 scripts in sequence)
- `scripts/annotate_tables.py:23-89` - Lifecycle stage mapping (hardcoded)
- `scripts/annotate_tables.py:136-172` - Importance tier computation
- `scripts/extract_relationships.py:22-61` - SQL view JOIN parsing
- `scripts/extract_relationships.py:76-122` - Access query JOIN parsing
- `scripts/extract_relationships.py:125-172` - Column name matching
- `scripts/profile_tables.py:60-128` - Type-aware column profiling
- `scripts/parse_access_queries.py:76-155` - MSysQueries SQL reconstruction

## Architecture Documentation
- Two-platform extraction (Windows PowerShell + Mac Bash/mdbtools)
- CSV files in Tables/ serve as the bridge between platforms
- All catalog/ outputs are version-controlled
- No hack/ directory exists in this project
- No CI/CD or test infrastructure
- Single validation notebook for manual QA

## Historical Context (from thoughts/)
- No project-specific thought documents exist yet
- `thoughts/global/shared/reference/WriteClaudeMD.md` - Reference for CLAUDE.md best practices (used as input for this research)

## Open Questions
- Should the data dictionary be regenerated when table CSVs are updated?
- Are there plans to add table descriptions to `table_annotations.json`?
- Would a Phase 3 (AI-assisted description generation) be useful?
