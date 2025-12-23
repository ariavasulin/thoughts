---
date: 2025-12-19T16:18:15-08:00
researcher: ariasulin
git_commit: b44e10ada59603fff73defcdafb24f122525e52c
branch: main
repository: dwsERP
topic: "Millflow Add Blocker Popup Implementation"
tags: [research, codebase, millflow, blocker, form, popup, modal]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Millflow Add Blocker Popup Implementation

**Date**: 2025-12-19T16:18:15-08:00
**Researcher**: ariasulin
**Git Commit**: b44e10ada59603fff73defcdafb24f122525e52c
**Branch**: main
**Repository**: dwsERP

## Research Question

Where is the "add blocker" popup in millflow, and what is its current implementation? The goal is to understand the existing structure for a potential redesign involving autofill, magic dates, etc.

## Summary

The blocker form is implemented as a simple inline popup component at `millflow/src/components/node/BlockerForm.tsx`. It is triggered by pressing `b` in the sheet view and renders as a centered modal overlay. The form currently uses basic HTML inputs without autofill, type-ahead, or "magic date" functionality. The DueDatePicker component in the same directory demonstrates a pattern for quick date shortcuts that could serve as reference for adding magic date features.

## Detailed Findings

### BlockerForm Component

**Location**: `millflow/src/components/node/BlockerForm.tsx`

The component is a React functional component wrapped in `memo()` with the following structure:

**Props Interface** (lines 5-8):
```typescript
interface BlockerFormProps {
  onSave: (blocker: { type: BlockerType; description: string; owner: string; expected_resolution?: string }) => void;
  onCancel: () => void;
}
```

**Form Fields**:
1. **Type Selector** (lines 52-67): Toggle buttons for 5 blocker types: `internal`, `external`, `material`, `approval`, `site`
2. **Description** (lines 70-78): Basic text input with placeholder "What's blocking this?"
3. **Owner** (lines 81-88): Basic text input with placeholder "Who owns resolving this?"
4. **Expected Resolution** (lines 91-97): Native HTML `<input type="date">` - no magic dates

**Keyboard Handling** (lines 32-45):
- `Cmd/Ctrl+Enter`: Save blocker
- `Escape`: Cancel and close

**Styling**: Uses Gruvbox theme classes (`bg-gruvbox-bg-1`, `border-gruvbox-bg-3`, etc.)

### How the Form is Triggered

**Keyboard Shortcut** (`millflow/src/hooks/useKeyboard.ts:288-294`):
```typescript
case 'b':
  // Add blocker (in sheet view with node selected)
  if (view === 'sheet' && selectedNodeId) {
    event.preventDefault();
    setActiveActionForm('blocker');
  }
  break;
```

**Rendering in SheetView** (`millflow/src/views/SheetView.tsx:201-208`):
```typescript
{activeActionForm === 'blocker' && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <BlockerForm
      onSave={handleBlockerSave}
      onCancel={handleActionFormCancel}
    />
  </div>
)}
```

### Blocker Type Definition

**Location**: `millflow/src/types/index.ts:49-57`

```typescript
export interface Blocker {
  id: string;
  type: BlockerType;
  description: string;
  owner: string;
  expected_resolution?: string;
  created_at: string;
  resolved_at?: string;
}
```

`BlockerType` is defined at line 3: `'internal' | 'external' | 'material' | 'approval' | 'site'`

### State Management

**Store**: `millflow/src/store/appStore.ts`

- `activeActionForm`: Controls which action form is shown (`'assign' | 'note' | 'blocker' | 'duedate' | null`)
- `addBlocker(sheetId, nodeId, blocker)`: Adds blocker to node (line 744+)
- `setActiveActionForm(form)`: Toggles form visibility (line 303)

### Related Components for Reference

**DueDatePicker** (`millflow/src/components/node/DueDatePicker.tsx`) - Has "magic date" pattern:
```typescript
const quickDates = [
  { label: 'Tomorrow', days: 1 },
  { label: 'In 3 days', days: 3 },
  { label: 'In 1 week', days: 7 },
  { label: 'In 2 weeks', days: 14 },
];
```

**AssigneePicker** (`millflow/src/components/node/AssigneePicker.tsx`) - Has autofill/search pattern:
- Filter input with real-time search
- Keyboard navigation (arrow keys, j/k)
- Suggested items shown first
- Multi-select with checkboxes

**NoteForm** (`millflow/src/components/node/NoteForm.tsx`) - Simple textarea form, similar pattern

## Code References

- `millflow/src/components/node/BlockerForm.tsx` - Main blocker form component
- `millflow/src/types/index.ts:3` - BlockerType definition
- `millflow/src/types/index.ts:49-57` - Blocker interface
- `millflow/src/views/SheetView.tsx:201-208` - BlockerForm rendering
- `millflow/src/hooks/useKeyboard.ts:288-294` - 'b' keyboard shortcut
- `millflow/src/store/appStore.ts:744` - addBlocker action
- `millflow/src/components/node/DueDatePicker.tsx:33-38` - Quick date pattern reference
- `millflow/src/components/node/AssigneePicker.tsx:36-50` - Filter/search pattern reference

## Architecture Documentation

### Modal/Popup Pattern

All action forms in millflow follow a consistent pattern:
1. State controlled via `activeActionForm` in Zustand store
2. Rendered as fixed overlay with semi-transparent backdrop (`bg-black/40`)
3. Component receives `onSave` and `onCancel` callbacks
4. Auto-focus on primary input via `useEffect` + `ref`
5. Keyboard shortcuts: `Cmd+Enter` to save, `Escape` to cancel
6. Fixed width container with Gruvbox styling

### Form Component Directory

All node-related form components live in `millflow/src/components/node/`:
- `BlockerForm.tsx` - Add blocker
- `NoteForm.tsx` - Add note
- `DueDatePicker.tsx` - Set due date
- `AssigneePicker.tsx` - Assign people
- `NodeDetailPanel.tsx` - Side panel for viewing node details

### Current Limitations (for redesign context)

The BlockerForm currently has:
- No autofill/type-ahead for owner field
- No magic date parsing for expected resolution
- No search/filter for blocker types
- No templates or pre-filled descriptions
- No integration with external systems

## Open Questions

- Should the owner field autocomplete from the `people` list in the store?
- What magic date formats should be supported? (e.g., "tomorrow", "next week", "12/25")
- Should there be suggested/template descriptions based on blocker type?
- Should the expected resolution show relative time preview like DueDatePicker?
