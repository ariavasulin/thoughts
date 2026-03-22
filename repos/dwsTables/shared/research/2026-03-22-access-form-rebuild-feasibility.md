---
date: 2026-03-22T10:00:00-07:00
researcher: Ariav Asulin
git_commit: f1d40a7
branch: main
repository: dwsTables
topic: "Feasibility of rebuilding Access pages/forms from the .accdb file"
tags: [research, codebase, access, forms, mdbtools, accdb, UI-extraction]
status: complete
last_updated: 2026-03-22
last_updated_by: Ariav Asulin
---

# Research: Can Access Forms/Pages Be Rebuilt from the .accdb File?

**Date**: 2026-03-22T10:00:00-07:00
**Researcher**: Ariav Asulin
**Git Commit**: f1d40a7
**Branch**: main
**Repository**: dwsTables

## Research Question
If interested in fully rebuilding complete pages from the accdb, would it be possible to scrape data and leverage the existing file to do so?

## Summary

**Short answer: Yes, but only via Access on Windows — not from Mac with current tools.**

The `DWSJA.accdb` file contains 334 forms, ~51 reports, 3 VBA modules, and 1,055 queries. The existing extraction pipeline (`scripts/extract_access_metadata.sh`) captures table data, query SQL, and MSysObjects metadata — but **no form/report designs**. Form designs are stored as proprietary binary blobs inside Access's internal OLE storage streams that no cross-platform tool (mdbtools, Jackcess, pyodbc) can decode.

The only reliable extraction path is `Application.SaveAsText` run from within Access on the Windows machine, which exports complete form definitions (controls, layout, properties, VBA code) to human-readable text files.

## Detailed Findings

### What Currently Exists in the Extracted Metadata

The extraction script (`scripts/extract_access_metadata.sh`) uses mdbtools to produce 7 files in `metadata/access/`. Relevant to forms:

- **`msysobjects.csv`** — Contains a row for every Access object. Forms have `Type = -32768` and `ParentId = -2147483647`. Provides: object name, creation/modification dates, and a binary `LvProp` blob.
- **`queries.csv`** — 1,055 reconstructed queries. Many of these are the RecordSource for forms (e.g., `qryBillHistoryWaiver` is the data source for `frmAcctgWaiverCP`).

The `LvProp` binary blobs in MSysObjects encode some readable strings in wide-character format:
- The form's **RecordSource** query name (e.g., `q r y B i l l H i s t o r y W a i v e r`)
- **Bound field names** (e.g., `B i l l H i s t I D`, `C o m p a n y`, `L o c n`)
- **Subform references** (e.g., `s f r m B i l l W o r k s h e e t`)
- **Theme metadata** (`ThemeResourceName`, `ClusterResourceName`)

### What Is NOT Captured

- Control types (textbox, combobox, button, listbox, etc.)
- Control positions, sizes, colors, fonts, tab order
- Form/report section definitions (header, detail, footer)
- Conditional formatting rules
- Event procedure assignments
- VBA source code (the `LvModule` column is empty for all rows)
- Macro action sequences (no macro objects in the extract)

### Object Inventory from MSysObjects

| Category | Count | Naming Pattern |
|----------|-------|----------------|
| Forms | 334 | `frm*`, `list*`, `sfrm*` |
| Reports | ~51 | `rpt*`, `lbl*` |
| VBA Modules | 3 | `JsonConverter`, `modUpdateSched`, `modUtilities` |
| Queries | 1,055 | `qry*`, `vew*`, `~sq_*` |
| Macros | 0 | (Scripts container exists but empty) |
| Startup form | `frmMenu` | (in MSysDb properties) |

### Why Cross-Platform Tools Cannot Extract Form Designs

Form/report designs are stored as **binary blobs in Access's OLE compound document streams** (MSysAccessStorage). The binary format is:
- Proprietary and undocumented by Microsoft
- Not accessible via ODBC, SQL, or any table-export mechanism
- Not decoded by mdbtools, Jackcess, UCanAccess, pyodbc, or oletools

| Tool | Tables | Queries | Forms/Reports | VBA |
|------|--------|---------|---------------|-----|
| mdbtools (Mac) | Yes | Definitions only | No | No |
| Jackcess (Java) | Yes | Definitions only | No | No |
| pyodbc | Yes* | Execute only | No | No |
| oletools/olevba | N/A | N/A | N/A | No (.accdb unsupported) |

*pyodbc requires Windows Access ODBC driver

### The Extraction Path: Application.SaveAsText on Windows

The only reliable method is to run this VBA from within Access on the Windows machine:

```vba
Dim obj As AccessObject
For Each obj In CurrentProject.AllForms
    Application.SaveAsText acForm, obj.Name, "C:\DWSExport\Form_" & obj.Name & ".txt"
Next
For Each obj In CurrentProject.AllReports
    Application.SaveAsText acReport, obj.Name, "C:\DWSExport\Report_" & obj.Name & ".txt"
Next
For Each obj In CurrentProject.AllModules
    Application.SaveAsText acModule, obj.Name, "C:\DWSExport\Module_" & obj.Name & ".txt"
Next
```

This produces human-readable text files containing:
- Complete control definitions (type, name, position, size, data source, formatting)
- Section layouts (header, detail, footer with dimensions)
- VBA code-behind for each form/report
- All property settings

These text files are reversible via `Application.LoadFromText`, meaning you can reconstruct the forms from the exports.

### Alternative: msaccess-vcs-addin

[msaccess-vcs-addin](https://github.com/joyfullservice/msaccess-vcs-addin) is an Access add-in that automates `SaveAsText` for all objects, converts UCS-2 to UTF-8 for git, and can rebuild an entire database from exported source files. It supports Access 2010–365 and is the most robust option for version-controlling an Access application.

### What You Could Rebuild Without Access

Using only the data currently extracted + the CSV table exports, you could reconstruct:
- **Data layer**: All 1,055 queries (already reconstructed SQL in `queries.csv`)
- **Partial form metadata**: Form names, their RecordSource queries, bound field names, and subform relationships (parseable from LvProp blobs with a custom wide-char decoder)
- **Table data**: All 122 SQL Server table exports in `Tables/`
- **Schema**: Local table DDL in `full_schema.sql`, SQL Server metadata in `metadata/sql_server/`

You could NOT reconstruct: visual layout, control types/positions, VBA logic, event wiring, conditional formatting, or navigation flow.

## Code References

- `scripts/extract_access_metadata.sh` — Main extraction script; uses mdbtools
- `scripts/parse_access_queries.py:76-155` — Reconstructs SQL from MSysQueries attributes
- `metadata/access/msysobjects.csv` — Contains all 334 form names + binary LvProp blobs
- `metadata/access/queries.csv` — 1,055 reconstructed queries (form RecordSources)

## Related Research

- `thoughts/shared/research/2026-03-21-codebase-research-for-claude-md.md` — Codebase overview

## Open Questions

1. **LvProp blob parsing** — A custom Python script could decode the wide-character strings in LvProp to extract form→query→field mappings without needing Access. This would give a partial "form skeleton" (which fields appear on which form, which queries feed which forms). Worth building?
2. **SaveAsText on Windows** — Has this been run yet? If not, a one-time extraction on the Windows machine would unlock complete form analysis on Mac.
3. **Rebuild target platform** — If rebuilding forms, what platform? A modern web UI, Power Apps, or reconstructing within Access itself? The answer determines how much of the original layout detail matters.
