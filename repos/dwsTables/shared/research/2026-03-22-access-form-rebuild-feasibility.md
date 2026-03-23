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

## What the SaveAsText Export Format Looks Like

The format is a hierarchical text file using `Begin`/`End` blocks with indentation. Example from a real form export:

```
Version =20
VersionRequired =20
Begin Form
    PopUp = NotDefault
    RecordSelectors = NotDefault
    AutoCenter = NotDefault
    Width =9360
    Caption ="MSAccessVCS"
    OnLoad ="[Event Procedure]"
    RecSrcDt = Begin
        0x79e78b777268e540
    End
    Begin
        Begin Label
            BackStyle =0
            FontSize =11
            FontName ="Calibri"
        End
        Begin CommandButton
            FontSize =11
            FontWeight =400
            Shape =1
        End
        Begin Section
            Height =6480
            BackColor =15130848
            Name ="Detail"
            Begin
                Begin CommandButton
                    Left =1800
                    Top =960
                    Width =900
                    Height =840
                    Name ="cmdExport"
                    Caption ="Export All Source"
                    OnClick ="[Event Procedure]"
                End
            End
        End
    End
End
```

Key format rules:
- `Begin <Type>` / `End` delimiters for nested blocks
- `<Property> =<Value>` pairs (only non-default values are written)
- Binary data (GUIDs, NameMaps, images) encoded as hex in `Begin`/`End` blocks
- Event procedures referenced as `"[Event Procedure]"` — actual VBA follows in a `CodeBehindForm` section
- Control positions in **twips** (1 twip = 1/1440 inch ≈ 1/15 pixel)
- Control type defaults declared first, then instances inside `Begin Section` blocks

## What Other People Have Done

### Version Control / Full Database Rebuild (Well-Established)

Rebuilding Access databases from SaveAsText exports is a **well-established practice**:

- **[msaccess-vcs-addin](https://github.com/joyfullservice/msaccess-vcs-addin)**: The most mature tool. Exports all objects into a `{db}.src/` folder structure:
  ```
  forms/frmMyForm.bas       # Form design (SaveAsText)
  forms/frmMyForm.cls       # VBA code-behind
  reports/rptMyReport.bas   # Report design
  queries/qryMyQuery.bas    # Query SQL
  modules/modMyModule.bas   # VBA modules
  tbldefs/tblMyTable.xml    # Table definitions (XML)
  dbs-properties.json       # Database properties
  vbe-references.json       # VBA library references
  ```
  Supports "Build from Source" — creating a complete `.accdb` from exported text files. Converts UCS-2 to UTF-8 for git. Supports Access 2010–365.

- **[decompose-msaccess](https://github.com/toddmowen/decompose-msaccess/blob/master/decompose-msaccess.vbs)**: Older VBScript approach using `.ADF` (forms), `.ADR` (reports), `.ADQ` (queries) extensions.

- **[OASIS-SVN](https://accessexperts.com/blog/2019/07/02/using-oasis-svn-and-git-for-access-source-code-control/)**: Commercial COM add-in that decomposes and recomposes databases.

- **Manual rebuild**: [RipTutorial guide](https://riptutorial.com/ms-access/example/26375/rebuild-the-entire-database) documents using `Application.LoadFromText` to reconstruct all objects. Tables require separate handling (XML/CSV import).

### Automated Access → Web Migrations (Commercial Only)

No open-source project converts Access forms to web UIs. Only commercial tools exist:

| Tool | Output | Approach | Link |
|------|--------|----------|------|
| **GAPVelocity AI Migrator** | ASP.NET Core + KendoUI + C# | "Hybrid AI + deterministic semantic pattern matching" from live .accdb | [gapvelocity.ai](https://www.gapvelocity.ai/migrate/ms-access) |
| **Antrow Software** | HTML5 + ASP.NET + JavaScript | 620+ databases migrated | [antrow.com](https://antrow.com/) |
| **fecher accessPORTER** | Wisej.NET browser app | Replaces Access presentation layer | [fecher.net](https://www.fecher.net/our-services/access-migration/) |
| **FMS Total Access Analyzer** | Documentation only | Generates form dictionary/control reports | [fmsinc.com](https://www.fmsinc.com/MicrosoftAccess/Documentation/) |

All of these work from the **live `.accdb` file via COM automation**, not from SaveAsText exports.

### Microsoft's Own Migration Path

- **[Access to Power Apps + Dataverse](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/migrate-access-to-dataverse)**: GA tool that migrates Access tables → Dataverse + generates Power Apps. Handles data and basic table structure but **does not convert forms** — you rebuild UI manually in Power Apps.
- **[SSMA for Access](https://learn.microsoft.com/en-us/sql/ssma/access/sql-server-migration-assistant-for-access-accesstosql)**: Migrates data to SQL Server. No form conversion.

### SaveAsText Parsers (None Exist Standalone)

No standalone parser for the SaveAsText format exists in Python, JavaScript, or any other language:

- **[Rubberduck VBA](https://github.com/rubberduck-vba/Rubberduck/issues/1647)** (C#): Has internal SaveAsText parsing for extracting control names for IntelliSense, but it's embedded in a large C# codebase — not a reusable library.
- The format is structurally simple (recursive `Begin`/`End` grammar with `Key = Value` properties) and would be straightforward to parse with a stack-based approach.

### Binary Format Reverse Engineering

The Access binary format has been **partially** reverse-engineered, but only for the data layer:

| Resource | Covers | Does NOT Cover |
|----------|--------|----------------|
| [Unofficial MDB Guide](http://jabakobob.net/mdb/) | Page structure, tables, indexes | Forms, reports, VBA |
| [mdbtools HACKING.md](https://github.com/mdbtools/mdbtools/blob/dev/HACKING.md) | Byte-level Jet3/Jet4 format | Form/report blobs |
| [Jackcess source code](https://github.com/jahlborn/jackcess) | Most complete accdb knowledge | Form deserialization |
| [Library of Congress](https://www.loc.gov/preservation/digital/formats/fdd/fdd000463.shtml) | Format identification | Internal structure |

**Nobody has reverse-engineered the form/report binary blob format** stored in MSysAccessStorage. Microsoft has never published a specification. The mdbtools FAQ explicitly lists form extraction as an aspirational future goal.

### COM Automation via Python (Windows Only)

On Windows, `pywin32` can automate Access via COM to extract everything programmatically:

```python
import win32com.client
app = win32com.client.Dispatch("Access.Application")
app.OpenCurrentDatabase(r"C:\path\to\DWSJA.accdb")
for form in app.CurrentProject.AllForms:
    app.SaveAsText(2, form.Name, f"C:\\export\\{form.Name}.txt")  # 2 = acForm
app.Quit()
```

This is equivalent to the VBA approach but scriptable from Python. References: [Python win32com docs](https://mail.python.org/pipermail/python-win32/2006-February/004236.html), [Practical Business Python](https://pbpython.com/windows-com.html).

## Practical Paths Forward for DWS

### Path 1: Full Extraction (Recommended First Step)
Run `SaveAsText` or install [msaccess-vcs-addin](https://github.com/joyfullservice/msaccess-vcs-addin) on the Windows machine. One-time export of all 334 forms, 51 reports, and 3 modules. Copy to Mac. This gives you complete, parseable text definitions of every UI component.

### Path 2: Build a SaveAsText Parser (Novel but Feasible)
No one has published a standalone Python parser for the format. The grammar is simple enough to build one — recursive `Begin`/`End` blocks, `Key = Value` properties, hex-encoded binary sections. This would let you:
- Map every form → its RecordSource query → the underlying SQL Server tables
- Inventory all controls (type, data binding, position)
- Generate documentation or a web UI scaffold

### Path 3: LvProp Blob Decoder (No Windows Needed)
Parse the wide-character strings already in `msysobjects.csv` to extract form→query→field mappings. Gets you a "form skeleton" without needing to touch the Windows machine.

### Path 4: Commercial Migration
If the end goal is replacing Access entirely, tools like GAPVelocity or Antrow handle the full conversion. These are substantial investments but produce working web applications.

## Related Research

- `thoughts/shared/research/2026-03-21-codebase-research-for-claude-md.md` — Codebase overview

## Open Questions

1. **SaveAsText on Windows** — Has this been run yet? If not, a one-time extraction on the Windows machine would unlock complete form analysis on Mac.
2. **Parser scope** — If building a SaveAsText parser, what's the target output? Documentation? A web UI scaffold? A data lineage graph?
3. **Rebuild target platform** — If rebuilding forms, what platform? React, Power Apps, or staying in Access? Determines how much layout detail matters.
4. **VBA complexity** — The 3 modules (`JsonConverter`, `modUpdateSched`, `modUtilities`) and form code-behind may contain significant business logic. How much VBA exists and how critical is it?
