# Asset Manager Standalone Web App — Implementation Plan

## Overview

Recreate the Access asset management module (frmAsset, listAssets, frmFindAsset, frmAssetTaskDashboard, frmWorkOrder) as a standalone Vite + React 19 + TypeScript + Tailwind v4 web app. Uses sql.js (SQLite compiled to WASM) for client-side database access with full CRUD. Dark theme matching the existing HTML demo aesthetic.

## Current State Analysis

### Access Form Navigation Graph
```
listAssets (qryAsset) ──DblClick──> frmAsset (qryAsset)
  ├── sfrmAssetVendor (qryAssetVendor)
  │     └── DblClick → frmCompany [out of scope]
  ├── sfrmAssetTasksUpcoming (qryAssetTasksUpcoming)
  │     └── DblClick → frmWorkOrder (qryWorkOrder)
  ├── sfrmAssetTasksCompleted (qryAssetTasksCompleted)
  │     └── DblClick → frmWorkOrder
  ├── Search → frmFindAsset → listAssets [with WHERE]
  └── Task Dashboard → frmAssetTaskDashboard
        ├── sfrmAllAssetTasksUpcoming (qryAllAssetTasksUpcoming)
        └── sfrmAllAssetTasksCompleted (qryAllAssetTasksCompleted)
```

### Key Discoveries

- **Missing tables**: `tblManufacturer` (MfgID, ManufName) and `tblRecurranceIntervals` (RecurranceIntervalID, RecurName, RecurSort) are NOT in the SQL Server export — they were Access-local tables. Must be seeded.
- **DB size**: Full `dws.sqlite` is 81MB. Asset subsystem only needs ~10 tables (~7K rows). A build script will extract a focused `asset.sqlite` (~1MB).
- **Table counts**: tblAssetLibrary=48, tblAssetModel=43, tblAssetType=50, tblAssetVendorLink=12, tblWorkOrders=8, tblShopAreas=11, tblLocation=668, tblCompany=1963, tblContact=5066
- **Access-isms to translate**: `IIf()` → `CASE WHEN`, `Date()` → `date('now')`, `&` concat → `||`, `True`/`False` → `-1`/`0`, `Is Null` → `IS NULL`
- **Recurrence logic** (from VBA): When completing a recurring work order, INSERT a new row with DueDate offset: 1=+1d, 2=+7d, 3=+30d, 4=+90d, 5=+182d, 6=+365d, 7=+1095d, 8=+1825d, 9=+3650d, 10=non-recurring, 11=+730d
- **Existing HTML demo** at `forms/demo/frmAsset.html` — dark theme, Geist font, card layout, color palette: bg `#222222`, card `#2e2e2e`, border `#4e4e4e`, accent `#2680FC`, green `#4ade80`, red `#ef4444`, amber `#fbbf24`, purple `#c4b5fd`
- **CRUD persistence**: sql.js is in-memory only. Must persist to IndexedDB after mutations so changes survive page reload.

### SQL Queries (translated to SQLite dialect)

**qryAsset** (main asset detail + list):
```sql
SELECT a.*, m.AssetModelName, mf.ManufName, sa.ShopAreaName, l.LocationName,
  mf.ManufName || ' ' || m.AssetModelName || ' ' || COALESCE(a.AssetSerialNumber,'') || ' ' || COALESCE(a.AssetNickname,'') || ' ' || COALESCE(a.AssetType,'') AS AssetKeyword
FROM tblAssetLibrary a
INNER JOIN tblAssetModel m ON a.AssetModelID = m.AssetModelID
INNER JOIN tblManufacturer mf ON a.AssetManufacturerID = mf.MfgID
INNER JOIN tblShopAreas sa ON a.ShopAreaID = sa.ShopAreaID
INNER JOIN tblLocation l ON a.LocationID = l.LocationID
ORDER BY a.AssetNickname
```

**qryAssetVendor** (vendor contacts for an asset):
```sql
SELECT avl.*, c.Company, c.CompanyType, c.CompNotes,
  ct.FirstName, ct.LastName, ct.Mobile, ct.OfficePhone, ct.CompanyID,
  CASE WHEN ct.OfficePhone > '0' THEN ct.OfficePhone ELSE ct.Phone END AS OfficePhoneFormula
FROM tblAssetVendorLink avl
LEFT JOIN tblCompany c ON avl.VendorID = c.CompanyID
LEFT JOIN tblContact ct ON avl.ContactID = ct.ContactID
WHERE avl.AssetID = ?
```

**qryAssetTasksUpcoming** (per-asset upcoming tasks):
```sql
SELECT wo.*,
  CASE WHEN wo.isComplete = -1 THEN 'g'
       WHEN wo.DueDate < date('now') THEN 'r'
       ELSE NULL END AS AlertColor
FROM tblWorkOrders wo
WHERE (wo.isComplete = 0 OR wo.isComplete IS NULL)
  AND wo.AssetID = ?
ORDER BY wo.DueDate
```

**qryAssetTasksCompleted** (per-asset completed tasks):
```sql
SELECT wo.*,
  CASE WHEN wo.isComplete = -1 THEN 'g'
       WHEN wo.DueDate < date('now') THEN 'r'
       ELSE NULL END AS AlertColor
FROM tblWorkOrders wo
WHERE wo.isComplete = -1
  AND wo.AssetID = ?
ORDER BY wo.CompletionDate
```

**qryAllAssetTasksUpcoming** (all assets, 7-day window):
```sql
SELECT mf.ManufName, a.AssetType, m.AssetModelName, sa.ShopAreaName, wo.*,
  CASE WHEN wo.isComplete = -1 THEN 'g'
       WHEN wo.DueDate < date('now') THEN 'r'
       ELSE NULL END AS AlertColor,
  a.AssetNickname
FROM tblWorkOrders wo
INNER JOIN tblAssetLibrary a ON wo.AssetID = a.AssetID
INNER JOIN tblShopAreas sa ON a.ShopAreaID = sa.ShopAreaID
INNER JOIN tblManufacturer mf ON a.AssetManufacturerID = mf.MfgID
INNER JOIN tblAssetModel m ON a.AssetModelID = m.AssetModelID
WHERE wo.DueDate < date('now', '+7 days')
  AND (wo.isComplete = 0 OR wo.isComplete IS NULL)
ORDER BY wo.DueDate
```

**qryAllAssetTasksCompleted** (all assets, completed):
```sql
SELECT wo.*, mf.ManufName, a.AssetType, m.AssetModelName, sa.ShopAreaName,
  CASE WHEN wo.isComplete = -1 THEN 'g'
       WHEN wo.DueDate < date('now') THEN 'r'
       ELSE NULL END AS AlertColor,
  a.AssetNickname
FROM tblWorkOrders wo
LEFT JOIN tblAssetLibrary a ON wo.AssetID = a.AssetID
LEFT JOIN tblShopAreas sa ON a.ShopAreaID = sa.ShopAreaID
LEFT JOIN tblManufacturer mf ON a.AssetManufacturerID = mf.MfgID
LEFT JOIN tblAssetModel m ON a.AssetModelID = m.AssetModelID
WHERE wo.isComplete = -1
ORDER BY wo.DueDate
```

**qryWorkOrder** (single work order detail):
```sql
SELECT wo.*, sa.ShopAreaName, mf.ManufName, m.AssetModelName,
  a.AssetSerialNumber, a.AssetType, a.AssetManufacturerID, a.AssetModelID,
  a.LocationID, a.MaintenenceSummary, a.AssetNickname, l.LocationName
FROM tblWorkOrders wo
INNER JOIN tblAssetLibrary a ON wo.AssetID = a.AssetID
INNER JOIN tblAssetModel m ON a.AssetModelID = m.AssetModelID
INNER JOIN tblShopAreas sa ON a.ShopAreaID = sa.ShopAreaID
INNER JOIN tblManufacturer mf ON a.AssetManufacturerID = mf.MfgID
INNER JOIN tblLocation l ON a.LocationID = l.LocationID
```

**Lookup queries for dropdowns:**
```sql
-- qryselectassettype
SELECT assetType FROM tblAssetType ORDER BY assetType

-- qrySelectAssetLocation (hardcoded DWS locations)
SELECT LocationID, LocationName FROM tblLocation WHERE LocationID IN (538, 1762)

-- Shop areas
SELECT ShopAreaID, ShopAreaName FROM tblShopAreas

-- Manufacturers
SELECT MfgID, ManufName FROM tblManufacturer

-- Models
SELECT AssetModelID, AssetModelName FROM tblAssetModel

-- Recurrence intervals
SELECT * FROM tblRecurranceIntervals ORDER BY RecurSort

-- Vendor contacts for a company
SELECT ContactID, FirstName || ' ' || LastName AS name, CompanyID
FROM tblContact WHERE CompanyID = ? AND FirstName || ' ' || LastName > ' A '
ORDER BY FirstName || ' ' || LastName

-- Vendor companies
SELECT CompanyID, Company FROM tblCompany WHERE CompanyType = 'Vendor' ORDER BY Company
```

## Desired End State

A standalone web app at `asset-manager/` that:
- Loads a focused SQLite database client-side via sql.js WASM
- Renders 5 views matching the Access asset management forms
- Supports full CRUD (add/edit/delete assets, work orders, vendor links)
- Persists changes to IndexedDB across page reloads
- Runs on `localhost:5200` via Vite dev server
- Exports modified DB as downloadable `.sqlite` file

### Verification:
- `cd asset-manager && npm run dev` starts the app
- `npm run build` produces a production build
- `npm run typecheck` passes with zero errors
- All 5 views render with real data from the SQLite database
- CRUD operations persist across page reload

## What We're NOT Doing

- File system integration (J: drive folder management from VBA)
- Print/report layout
- IT hardware/software asset pages (qryHardware, qryLicense)
- Integration with the existing DWS Pulse React app
- Authentication/multi-user
- Server-side persistence (all client-side)
- The frmCompany form (vendor detail — out of scope)

## Implementation Approach

Single standalone Vite project. React for UI with Tailwind v4 for styling. sql.js for SQLite WASM. No router library — use useState for view switching (matches existing DWS app pattern). A Python build script creates the focused `asset.sqlite` from `dws.sqlite`.

---

## Phase 1: Project Scaffolding & Data Layer

### Overview
Set up the Vite project, install dependencies, create the SQLite provider, build the asset-focused database, and implement all query/mutation functions.

### Changes Required:

#### 1. Python build script
**File**: `scripts/build_asset_db.py`
**Purpose**: Extract asset-related tables from dws.sqlite into a small asset.sqlite, create missing tblManufacturer and tblRecurranceIntervals with seed data.

Tables to extract: tblAssetLibrary, tblAssetModel, tblAssetType, tblAssetVendorLink, tblWorkOrders, tblShopAreas, tblLocation, tblCompany, tblContact

Seed tables to create:
- `tblManufacturer` (MfgID INTEGER PK, ManufName TEXT) — populate MfgID from distinct AssetManufacturerID values in tblAssetLibrary, ManufName defaults to 'Manufacturer {MfgID}' (user can rename via CRUD)
- `tblRecurranceIntervals` (RecurranceIntervalID INTEGER PK, RecurName TEXT, RecurSort INTEGER, DaysOffset INTEGER) — hardcoded from VBA: 1=Daily/1, 2=Weekly/7, 3=Monthly/30, 4=Quarterly/90, 5=Semi-Annual/182, 6=Annual/365, 7=3-Year/1095, 8=5-Year/1825, 9=10-Year/3650, 10=Non-Recurring/0, 11=2-Year/730

#### 2. Vite project scaffold
**Directory**: `asset-manager/`

```
asset-manager/
  package.json          — react 19, sql.js, tailwindcss v4
  vite.config.ts        — react plugin, tailwind vite plugin, port 5200
  tsconfig.json
  index.html
  public/
    asset.sqlite        — copied from scripts output
  src/
    main.tsx
    index.css           — tailwind import, Geist font, dark theme base
    App.tsx             — view state machine, tab navigation
    types.ts            — all TypeScript interfaces
    db/
      provider.tsx      — React context providing sql.js Database instance
      queries.ts        — all SELECT query functions
      mutations.ts      — all INSERT/UPDATE/DELETE functions
      persist.ts        — IndexedDB save/load for database persistence
    components/
      AssetList.tsx
      AssetDetail.tsx
      AssetSearch.tsx
      TaskDashboard.tsx
      WorkOrderDetail.tsx
      shared/
        RecordNav.tsx    — record navigation bar
        FieldInput.tsx   — styled input/select/textarea
        DataTable.tsx    — reusable sortable table
        StatusPill.tsx   — status badge component
```

#### 3. SQLite Provider (`src/db/provider.tsx`)
- Initialize sql.js with WASM from CDN
- On mount: try load from IndexedDB first, fall back to fetch `public/asset.sqlite`
- Expose `db` instance via React context
- After each mutation, persist full DB to IndexedDB
- Provide `exportDB()` function for downloading `.sqlite` file

#### 4. Query functions (`src/db/queries.ts`)
All translated SQL queries from the "SQL Queries" section above, typed with return interfaces.

#### 5. Mutation functions (`src/db/mutations.ts`)
- `insertAsset(fields)` → INSERT into tblAssetLibrary, return new AssetID
- `updateAsset(id, fields)` → UPDATE tblAssetLibrary
- `deleteAsset(id)` → DELETE from tblAssetLibrary + cascade tblAssetVendorLink + tblWorkOrders
- `insertWorkOrder(fields)` → INSERT into tblWorkOrders
- `updateWorkOrder(id, fields)` → UPDATE tblWorkOrders
- `completeWorkOrder(id)` → UPDATE isComplete=-1, CompletionDate=now; if recurring, INSERT new work order with offset DueDate
- `insertVendorLink(fields)` → INSERT into tblAssetVendorLink
- `deleteVendorLink(id)` → DELETE from tblAssetVendorLink
- `insertManufacturer(name)` → INSERT into tblManufacturer
- `insertModel(name)` → INSERT into tblAssetModel
- `insertAssetType(name)` → INSERT into tblAssetType

### Success Criteria:

#### Automated Verification:
- [ ] `cd scripts && python build_asset_db.py` creates `asset-manager/public/asset.sqlite`
- [ ] `cd asset-manager && npm install` succeeds
- [ ] `npm run typecheck` passes
- [ ] `npm run dev` starts on port 5200
- [ ] Browser loads and displays "Asset Management" header with data from SQLite

#### Manual Verification:
- [ ] Database loads in <2 seconds
- [ ] Console shows no errors

**Implementation Note**: After completing this phase and all automated verification passes, pause for confirmation before proceeding.

---

## Phase 2: Asset List View (listAssets)

### Overview
Sortable table of all assets with clickable rows, column header sort, and navigation to asset detail.

### Changes Required:

#### 1. AssetList component
**File**: `src/components/AssetList.tsx`

Columns (matching Access listAssets): ID, Type, Department, Manufacturer, Model, Location, Nickname

Features:
- Column header click → sort (matching Access sort keys: e.g., Manufacturer sorts by ManufName, AssetModelName, AssetNickname)
- Row click → navigate to AssetDetail for that AssetID
- "Add Asset" button → navigate to AssetDetail in add mode
- Uses qryAsset query
- Accepts optional `whereFilter` prop from search view

### Success Criteria:

#### Automated Verification:
- [ ] `npm run typecheck` passes

#### Manual Verification:
- [ ] Table shows all 48 assets with correct data
- [ ] Clicking column headers sorts correctly
- [ ] Clicking a row opens asset detail
- [ ] "Add Asset" opens blank detail form

---

## Phase 3: Asset Detail View (frmAsset)

### Overview
Main asset form with dropdowns, notes, and three embedded subform tables (vendor contacts, upcoming tasks, completed tasks).

### Changes Required:

#### 1. AssetDetail component
**File**: `src/components/AssetDetail.tsx`

Top section — Asset Details card:
- Manufacturer dropdown (tblManufacturer, with "+" add button)
- Model dropdown (tblAssetModel, with "+" add button)
- Type dropdown (tblAssetType, with "+" add button)
- Location dropdown (qrySelectAssetLocation — IDs 538, 1762)
- Department dropdown (tblShopAreas)
- Nickname text input
- Serial # text input
- Purchase Date text input
- Retire Date text input
- Asset ID badge (read-only)

Middle section — Notes card:
- MaintenenceSummary textarea

Bottom section — three subform tables:

**Vendor Contacts table** (sfrmAssetVendor):
- Columns: Company, Contact, Mobile, Office, Category, Notes
- "Add Contact" button → inline add row with company dropdown (tblCompany) and contact dropdown (filtered by selected company)
- Delete button per row

**Maintenance Needed table** (sfrmAssetTasksUpcoming):
- Columns: Done (checkbox), Task, Due (with color-coded status pill), Recurrence
- Checkbox click → completes task, triggers recurrence spawn if applicable
- "Add Task" button → inline add with description, due date, recurrence dropdown
- Row click → navigate to WorkOrderDetail

**Services Completed table** (sfrmAssetTasksCompleted):
- Columns: Done (checked, disabled), Task, Completed, Recurrence
- Row click → navigate to WorkOrderDetail

Record navigation bar at bottom (matching Access): |◄ ◄ [record#] of [total] ► ►| + search box

Save/revert: Auto-save on field blur (matching Access behavior) or explicit Save button.

### Success Criteria:

#### Automated Verification:
- [ ] `npm run typecheck` passes

#### Manual Verification:
- [ ] All dropdowns populate with real data
- [ ] Editing fields and blurring saves to SQLite
- [ ] Adding a vendor contact creates a row in tblAssetVendorLink
- [ ] Completing a recurring task creates a new work order with correct next due date
- [ ] Record nav navigates between all 48 assets
- [ ] "+" buttons on dropdowns open inline add for manufacturer/model/type
- [ ] Changes persist after page reload (IndexedDB)

---

## Phase 4: Search View (frmFindAsset)

### Overview
Filter form with 10+ criteria that builds a compound WHERE clause and passes it to the asset list.

### Changes Required:

#### 1. AssetSearch component
**File**: `src/components/AssetSearch.tsx`

Filter fields (matching frmFindAsset):
- Type dropdown (qryselectassettype)
- Warehouse dropdown (qrySelectAssetLocation)
- Serial # text input
- Department dropdown (tblShopAreas)
- Manufacturer dropdown (tblManufacturer)
- Model dropdown (tblAssetModel)
- Nickname text input
- Keyword text input (searches AssetKeyword concatenation)
- Date Added range (from/to)
- Active checkbox / Retired checkbox (RemoveDate IS NULL / IS NOT NULL)

"Search" button → builds WHERE string → sets filter on AssetList
"Clear" button → resets all filters

### Success Criteria:

#### Automated Verification:
- [ ] `npm run typecheck` passes

#### Manual Verification:
- [ ] Each filter field correctly narrows the asset list
- [ ] Multiple filters combine with AND
- [ ] Active/Retired toggle works (RemoveDate null vs not null)
- [ ] Keyword search matches across manufacturer, model, serial, nickname, type
- [ ] Clear resets to full list

---

## Phase 5: Task Dashboard (frmAssetTaskDashboard)

### Overview
All-assets view showing upcoming and completed maintenance tasks with enriched asset context.

### Changes Required:

#### 1. TaskDashboard component
**File**: `src/components/TaskDashboard.tsx`

Two side-by-side tables:

**Upcoming Tasks** (qryAllAssetTasksUpcoming):
- Columns: Asset (nickname), Manufacturer, Model, Department, Task, Due Date (color-coded), Recurrence
- "Done" checkbox → complete task with recurrence spawn
- Row click → navigate to WorkOrderDetail
- Color coding: red if overdue, amber if due within 7 days

**Completed Tasks** (qryAllAssetTasksCompleted):
- Columns: Asset (nickname), Manufacturer, Model, Department, Task, Due Date, Recurrence
- All rows green (completed)
- Row click → navigate to WorkOrderDetail

### Success Criteria:

#### Automated Verification:
- [ ] `npm run typecheck` passes

#### Manual Verification:
- [ ] Upcoming tasks show only incomplete work orders due within 7 days
- [ ] Completed tasks show all finished work orders across all assets
- [ ] Completing a task from this view works and refreshes both tables
- [ ] Clicking a row navigates to work order detail

---

## Phase 6: Work Order Detail (frmWorkOrder)

### Overview
Individual work order form with full asset context and recurrence completion logic.

### Changes Required:

#### 1. WorkOrderDetail component
**File**: `src/components/WorkOrderDetail.tsx`

Asset context section (read-only from asset):
- Manufacturer, Model, Serial #, Type, Nickname, Department, Location

Work order fields:
- Description (WorkOrderDescription) — text input
- Long Description (WorkOrderDescriptionLong) — textarea
- Due Date — date input
- Completion Date — date input (auto-set when completing)
- Resolution Details — textarea
- Is Complete — checkbox
- Recurrence Interval — dropdown (tblRecurranceIntervals)

Completion logic (matching VBA Check62_Click):
- When isComplete checked → set CompletionDate to today
- If RecurranceInterval is not 10 (Non-Recurring) and DueDate > 2000-01-01:
  - Calculate newDueDate based on interval offset
  - INSERT new tblWorkOrders row with same AssetID, description, recurrence, new due date

"Go to Asset" button → navigate to AssetDetail for this work order's AssetID

### Success Criteria:

#### Automated Verification:
- [ ] `npm run typecheck` passes
- [ ] `npm run build` succeeds with no errors

#### Manual Verification:
- [ ] Work order form shows correct asset context
- [ ] Completing a recurring work order creates the next occurrence
- [ ] "Go to Asset" navigates correctly
- [ ] All fields save to SQLite and persist

---

## Testing Strategy

### Automated:
- TypeScript strict mode catches type errors
- `npm run build` catches build issues
- All queries tested against real SQLite data on load

### Manual Testing Steps:
1. Load app → verify all 48 assets appear in list
2. Click an asset → verify detail form populates correctly
3. Edit nickname → verify saves and persists after reload
4. Add a vendor contact → verify appears in table
5. Complete a recurring task → verify new task spawns with correct date
6. Use search filters → verify list narrows correctly
7. Check task dashboard → verify upcoming/completed split
8. Export database → verify downloadable file

## Performance Considerations

- Asset-focused SQLite (~1MB) loads in <500ms vs full 81MB database
- sql.js WASM loaded from CDN (sql.js CDN or bundled)
- IndexedDB persistence adds ~50ms per mutation
- All queries are indexed on PKs (from build script)

## References

- Existing HTML demo: `forms/demo/frmAsset.html`
- Access form definitions: `forms/frmAsset.txt`, `forms/listAssets.txt`, etc.
- Query analysis: `knowledge/access_analysis/assets_maintenance.md`
- Access SQL queries: `context_compiler/queries/qry*.sql`
- SQLite build script: `scripts/build_sqlite.py`
- Catalog profiles: `catalog/profiles/tblAsset*.json`
