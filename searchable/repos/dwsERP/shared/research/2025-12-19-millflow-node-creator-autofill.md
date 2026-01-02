---
date: 2025-12-19T16:17:50-08:00
researcher: ariasulin
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "Millflow Node Creator Popup with Template Name Autofill"
tags: [research, codebase, millflow, node-creator, autofill, templates]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Millflow Node Creator Popup with Template Name Autofill

**Date**: 2025-12-19T16:17:50-08:00
**Researcher**: ariasulin
**Git Commit**: b44e10ada59603fff73defcdafb24f122525e52c
**Branch**: main
**Repository**: dwsERP

## Research Question
Where is the "add new node" popup in millflow, and how can it be redesigned to use autofill for template names (borrowing from factory-floor's node names)?

## Summary

The millflow project has **two components** for adding nodes:

1. **NodeCreator** (`millflow/src/components/dag/NodeCreator.tsx`) - The main popup for creating single nodes with a text input and type selector
2. **TemplatePicker** (`millflow/src/components/dag/TemplatePicker.tsx`) - A separate popup for selecting predefined multi-node templates

The user wants to add **autofill/autocomplete** to the NodeCreator's name input field, using template names borrowed from the factory-floor project's `OPERATION_TYPES` array.

## Detailed Findings

### NodeCreator Component

**Location**: `millflow/src/components/dag/NodeCreator.tsx`

**Current Implementation**:
- Props: `onSave(name, type)`, `onCancel()`, `position ('before' | 'after')`
- State: `name` (string), `type` (NodeType), `showTypeSelector` (boolean)
- Input: Plain text input with no autocomplete (line 70-78)
- Type selector: Grid of 7 buttons, toggled with `/` key (lines 88-108)

**Available Node Types** (line 13):
```typescript
const nodeTypes: NodeType[] = ['shop', 'external', 'material', 'approval', 'qc', 'delivery', 'install'];
```

**Current Input Field** (lines 70-78):
```typescript
<input
  ref={inputRef}
  type="text"
  value={name}
  onChange={(e) => setName(e.target.value)}
  onKeyDown={handleKeyDown}
  placeholder="Node name..."
  className="flex-1 bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded px-3 py-2 text-sm..."
/>
```

**Keyboard Shortcuts**:
- `Enter` - save (if name not empty)
- `Esc` - cancel
- `/` - toggle type selector

### TemplatePicker Component

**Location**: `millflow/src/components/dag/TemplatePicker.tsx`

**Current Implementation**:
- Displays predefined templates from `millflow/src/data/templates.ts`
- Keyboard navigable with `j/k` (vim-style) and arrow keys
- Shows template name, description, and node count
- Selects with `Enter`, cancels with `Esc`

**Predefined Templates** (from `templates.ts`):
1. Standard Shop Flow (CNC → Assembly → Finishing → QC)
2. Delivery + Install
3. Approval Gate
4. Material Wait
5. External Vendor
6. QC + Rework

### SheetView Integration

**Location**: `millflow/src/views/SheetView.tsx`

Both popups are rendered as overlays:
- NodeCreator: lines 170-178, triggered by `nodeCreationMode` state
- TemplatePicker: lines 220-228, triggered by `showTemplatePicker` state

```typescript
{nodeCreationMode && selectedNodeId && (
  <div className="fixed inset-0 z-30 flex items-center justify-center bg-black/40">
    <NodeCreator
      position={nodeCreationMode}
      onSave={handleNodeCreate}
      onCancel={handleNodeCreateCancel}
    />
  </div>
)}
```

### Factory-Floor Template Names (Source for Autofill)

**Location**: `factory-floor/src/App.tsx:24-40`

```typescript
const OPERATION_TYPES = [
    // Shop operations
    { type: 'Drafting', label: 'Drafting' },
    { type: 'CNC', label: 'CNC' },
    { type: 'EdgeBanding', label: 'Edge Banding' },
    { type: 'Veneer', label: 'Veneer' },
    { type: 'Milling', label: 'Milling' },
    { type: 'Finishing', label: 'Finishing' },
    { type: 'Bench', label: 'Bench' },
    // External
    { type: 'Outsourced', label: 'Outsourced' },
    // Logistics operations
    { type: 'Staging', label: 'Staging' },
    { type: 'Delivery', label: 'Delivery' },
    { type: 'FieldInstall', label: 'Field Install' },
    { type: 'PunchList', label: 'Punch List' },
];
```

These 12 operation labels are the suggested source for autofill suggestions in millflow.

### Type Differences Between Projects

| Millflow NodeType | Factory-Floor Operation Types |
|-------------------|-------------------------------|
| shop              | Drafting, CNC, EdgeBanding, Veneer, Milling, Finishing, Bench |
| external          | Outsourced |
| material          | (no direct equivalent) |
| approval          | (no direct equivalent) |
| qc                | (no direct equivalent) |
| delivery          | Staging, Delivery |
| install           | FieldInstall, PunchList |

## Code References

- `millflow/src/components/dag/NodeCreator.tsx:1-120` - Main node creation popup
- `millflow/src/components/dag/NodeCreator.tsx:70-78` - Text input field (target for autofill)
- `millflow/src/components/dag/TemplatePicker.tsx:1-77` - Template selection popup
- `millflow/src/data/templates.ts:1-74` - Template definitions
- `millflow/src/views/SheetView.tsx:170-178` - NodeCreator overlay rendering
- `millflow/src/views/SheetView.tsx:220-228` - TemplatePicker overlay rendering
- `millflow/src/types/index.ts:4` - NodeType definition
- `factory-floor/src/App.tsx:24-40` - OPERATION_TYPES array (source for autofill suggestions)

## Architecture Documentation

### Current UI Pattern
- Full-screen overlay with `fixed inset-0 z-30 bg-black/40`
- Centered card with rounded corners and shadow
- Keyboard-first interaction (Enter/Esc/arrow keys)
- Gruvbox color theme throughout

### State Management
- `nodeCreationMode` in Zustand store controls NodeCreator visibility
- `showTemplatePicker` in Zustand store controls TemplatePicker visibility
- Both set via actions in `millflow/src/store/appStore.ts`

### Existing Autocomplete in Codebase
No autocomplete/combobox implementations exist in either project. Both use:
- Native HTML `<select>` elements (factory-floor)
- Custom button grids for type selection (millflow)
- The `cmdk` package is installed in millflow for the CommandPalette but not used for node creation

## Related Files

| File | Purpose |
|------|---------|
| `millflow/src/components/dag/NodeCreator.tsx` | Node creation popup (primary target) |
| `millflow/src/components/dag/TemplatePicker.tsx` | Template selection popup |
| `millflow/src/data/templates.ts` | Template definitions |
| `millflow/src/views/SheetView.tsx` | Parent view integrating both popups |
| `millflow/src/store/appStore.ts` | State management for popup visibility |
| `millflow/src/lib/icons.ts` | Node type icons and labels |
| `millflow/src/types/index.ts` | Type definitions |
| `factory-floor/src/App.tsx` | Source of OPERATION_TYPES for autofill |

## Open Questions

1. Should the autofill use exact factory-floor names or adapt them to millflow's type system?
2. Should selecting an autofill suggestion auto-set the node type based on the name?
3. Should the NodeCreator and TemplatePicker be combined into a single unified component?
4. Should the autofill suggestions be stored in a shared data file or hardcoded?
