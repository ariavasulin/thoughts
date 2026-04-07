# Clean Hex/Binary from Access Form & Report Exports

## Overview

Strip hex/binary blobs and convert encoding in the 416 Access form/report text exports under `PMDBA/reports/` (48 files) and `PMDBA/forms/` (368 files). These files are UTF-16LE with CRLF line endings and contain ~76% hex blob content that is meaningless for analysis.

## Current State Analysis

**Files**: 416 `.txt` files exported from Access via VBA (`scripts/export_access_forms.vba`)
- `PMDBA/reports/` — 48 files (rpt*, lbl*)
- `PMDBA/forms/` — 368 files (frm*, sfrm*, list*, pop*, pu*, _*)
- `PMDBA/forms/demo/` — 2 HTML files (untouched)

**Encoding**: All UTF-16LE with BOM (`ff fe`), CRLF line endings

**Hex sections** (property = Begin / 0x... / End pattern):
- `GUID` — per-object GUIDs, appear at every nesting level (form, controls)
- `RecSrcDt` — record source timestamp
- `NameMap` — internal field name mapping blob (largest section)
- `PrtMip` — printer margin/info settings
- `PrtDevMode` — printer device mode binary
- `PrtDevNames` — printer device name binary
- `PrtDevModeW` — wide version of PrtDevMode
- `PrtDevNamesW` — wide version of PrtDevNames

**Preserved content** (structural blocks that are NOT hex):
- `Begin Form`, `Begin Report`, `Begin Section`, `Begin Label`, `Begin TextBox`, `Begin ComboBox`, `Begin CommandButton`, `Begin Subform`, `Begin Tab`, `Begin Image`, `Begin Rectangle`, `Begin CheckBox`, `Begin FormHeader`, etc.
- `OnClickEmMacro = Begin` (contains VBA macro text, not hex)
- All property lines (Name, RecordSource, ControlSource, OnOpen, Width, etc.)
- Version/VersionRequired lines

**Also removed**: `Checksum` line (checksum of hex data, meaningless without it)

## Desired End State

Every `.txt` file in `PMDBA/reports/` and `PMDBA/forms/` is:
1. **UTF-8** (no BOM), **LF** line endings
2. **No hex blob sections** — all `{property} = Begin` / `0x...` / `End` blocks removed
3. **No Checksum line**
4. **All meaningful content preserved** — properties, structural blocks, event procedures, macro text blocks
5. File names unchanged (`.txt`)
6. `PMDBA/forms/demo/` untouched

### Verification:
- `file PMDBA/reports/*.txt` → all report "ASCII text" or "UTF-8 Unicode text"
- `grep -r '0x[0-9a-f]' PMDBA/reports/ PMDBA/forms/` → zero matches
- `grep -rl 'Checksum' PMDBA/reports/ PMDBA/forms/` → zero matches
- `grep -rl 'RecordSource' PMDBA/reports/ PMDBA/forms/` → same count as before (meaningful content preserved)
- Spot-check: structural Begin/End blocks intact, event procedures preserved

## What We're NOT Doing

- Not renaming files or changing extensions
- Not modifying `PMDBA/forms/demo/` HTML files
- Not stripping Version/VersionRequired lines
- Not modifying the VBA export script (`scripts/export_access_forms.vba`)
- Not changing any files outside `PMDBA/reports/` and `PMDBA/forms/`

## Implementation — Single Phase

### Script: `scripts/clean_access_exports.py`

```python
#!/usr/bin/env python3
"""Strip hex/binary blobs from Access form/report text exports.

Converts UTF-16LE → UTF-8, strips GUID/NameMap/PrtMip/PrtDevMode/PrtDevNames/
PrtDevModeW/PrtDevNamesW/RecSrcDt hex blocks and Checksum lines.
"""

import sys
from pathlib import Path

PMDBA = Path(__file__).resolve().parent.parent / "PMDBA"


def clean_lines(lines: list[str]) -> list[str]:
    """Remove hex Begin/End blocks and Checksum lines."""
    result = []
    i = 0
    while i < len(lines):
        line = lines[i]
        stripped = line.strip()

        # Skip Checksum line
        if stripped.startswith("Checksum"):
            i += 1
            continue

        # Check for hex blob: "{property} = Begin" where content is 0x...
        if stripped.endswith("= Begin"):
            # Look ahead to determine if this is a hex block
            j = i + 1
            is_hex = True
            while j < len(lines):
                bline = lines[j].strip()
                if bline == "End":
                    j += 1
                    break
                if not bline.startswith("0x") and bline not in ("", ","):
                    is_hex = False
                    break
                j += 1

            if is_hex:
                # Skip entire hex block (Begin line + hex lines + End line)
                i = j
                continue

        result.append(line)
        i += 1

    return result


def process_file(path: Path) -> tuple[int, int]:
    """Process one file. Returns (original_lines, cleaned_lines)."""
    text = path.read_bytes().decode("utf-16-le").lstrip("\ufeff")
    lines = text.splitlines()
    original = len(lines)

    cleaned = clean_lines(lines)

    # Write back as UTF-8 with LF
    path.write_text("\n".join(cleaned) + "\n", encoding="utf-8")
    return original, len(cleaned)


def main():
    dirs = [PMDBA / "reports", PMDBA / "forms"]
    total_orig = 0
    total_clean = 0
    file_count = 0

    for d in dirs:
        for f in sorted(d.glob("*.txt")):
            orig, clean = process_file(f)
            total_orig += orig
            total_clean += clean
            file_count += 1
            print(f"  {f.relative_to(PMDBA)}: {orig} → {clean} lines")

    removed = total_orig - total_clean
    pct = removed * 100 // total_orig if total_orig else 0
    print(f"\n{file_count} files processed")
    print(f"{total_orig:,} → {total_clean:,} lines ({removed:,} removed, {pct}%)")


if __name__ == "__main__":
    main()
```

### Success Criteria

#### Automated Verification:
- [ ] Script runs without errors: `python3 scripts/clean_access_exports.py`
- [ ] No hex content remains: `grep -r '0x[0-9a-f]' PMDBA/reports/ PMDBA/forms/*.txt` → 0 matches
- [ ] No Checksum lines remain: `grep -rl 'Checksum' PMDBA/reports/ PMDBA/forms/` → 0 matches
- [ ] All files now UTF-8: `file PMDBA/reports/*.txt PMDBA/forms/*.txt | grep -v 'UTF-8\|ASCII\|text'` → 0 matches
- [ ] RecordSource lines preserved: `grep -rl 'RecordSource' PMDBA/reports/ PMDBA/forms/` → same count as original
- [ ] Version lines preserved: `grep -c 'Version' PMDBA/reports/rptBidClient.txt` → 2 (Version + VersionRequired)
- [ ] Structural blocks intact: `grep -c 'Begin Form\|Begin Report\|Begin Section\|Begin Label\|Begin TextBox' PMDBA/forms/frmBid.txt` → same as original
- [ ] demo/ untouched: `file PMDBA/forms/demo/*` → HTML files unchanged

#### Manual Verification:
- [ ] Spot-check 2-3 cleaned files to confirm readability and no lost meaningful content
