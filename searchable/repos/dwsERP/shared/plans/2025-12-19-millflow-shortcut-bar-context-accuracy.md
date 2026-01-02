# ShortcutBar Context Accuracy Implementation Plan

## Overview

Make the ShortcutBar display only shortcuts that actually work in the current context. Currently it only checks `view` and `showNodeDetail`, but the keyboard handler has 8 context levels that determine what shortcuts are active.

## Current State Analysis

**File**: `millflow/src/components/ShortcutBar.tsx`

The ShortcutBar reads only 3 values (line 55):
```typescript
const { view, showNodeDetail, pendingKeySequence } = useAppStore();
```

But `useKeyboard.ts` blocks shortcuts based on these additional states:
- `showCommandPalette` - blocks all except Escape
- `showHelpModal` - blocks all except Escape/?
- `nodeCreationMode` - blocks all except Escape
- `activeActionForm` - blocks all except Escape
- `showTemplatePicker` - blocks all except Escape
- `multiSelectMode` - changes Space/J/dd behavior

### Shortcuts Currently Shown vs Actually Implemented

| Context | Shown | Not Implemented |
|---------|-------|-----------------|
| Dashboard | `f`, `a`, `b` | These don't exist |
| Jobs | `n` (new job) | Doesn't exist |
| Job | `n`, `e` | `n` should be `c`; `e` doesn't exist |
| Node Detail | `r`, `f`, `d` | `r`, `f` don't exist; `d` should be `D` |

## Desired End State

ShortcutBar displays only shortcuts that work RIGHT NOW based on all context states, matching exactly what `useKeyboard.ts` will accept.

### Verification:
1. When command palette is open, bar shows only `esc close`
2. When a form is open, bar shows only `esc close`
3. When in multi-select mode, bar shows multi-select specific shortcuts
4. All displayed shortcuts have working implementations in useKeyboard.ts

## What We're NOT Doing

- NOT implementing new shortcuts (f, a, b, n, e, r)
- NOT changing keyboard handler behavior
- NOT modifying HelpModal (separate task)

## Implementation Approach

Single-phase change to ShortcutBar.tsx:
1. Add all context state reads from appStore
2. Define shortcut arrays that only contain implemented shortcuts
3. Implement context priority logic matching useKeyboard.ts

---

## Phase 1: Update ShortcutBar Context Awareness

### Overview
Rewrite ShortcutBar to read all context states and display only valid shortcuts for the current state.

### Changes Required:

**File**: `millflow/src/components/ShortcutBar.tsx`

#### 1. Update store imports (line 55)

```typescript
const {
  view,
  showNodeDetail,
  pendingKeySequence,
  showCommandPalette,
  showHelpModal,
  nodeCreationMode,
  activeActionForm,
  showTemplatePicker,
  multiSelectMode,
} = useAppStore();
```

#### 2. Replace shortcut arrays with corrected versions (lines 9-52)

**Dashboard shortcuts** - remove `f`, `a`, `b` (not implemented):
```typescript
const dashboardShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'navigate' },
  { key: 'enter', label: 'open' },
  { key: 'g h', label: 'home' },
  { key: '?', label: 'help' },
];
```

**Jobs shortcuts** - remove `n` (not implemented):
```typescript
const jobsShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'navigate' },
  { key: 'enter', label: 'open job' },
  { key: 'g h', label: 'home' },
  { key: '?', label: 'help' },
];
```

**Job shortcuts** - fix `n` to `c`, remove `e`:
```typescript
const jobShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'navigate' },
  { key: 'enter', label: 'open sheet' },
  { key: 'c', label: 'new sheet' },
  { key: 'esc', label: 'back' },
  { key: '?', label: 'help' },
];
```

**Sheet shortcuts** - keep as-is (all implemented):
```typescript
const sheetShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'nav' },
  { key: 'o', label: 'after' },
  { key: 'O', label: 'before' },
  { key: 't', label: 'template' },
  { key: 's', label: 'split' },
  { key: 'u', label: 'undo' },
  { key: '?', label: 'help' },
];
```

**Node detail shortcuts** - remove `r`, `f`; fix `d` to `D`:
```typescript
const nodeDetailShortcuts: Shortcut[] = [
  { key: 'a', label: 'assign' },
  { key: 'b', label: 'blocker' },
  { key: 'n', label: 'note' },
  { key: 'D', label: 'due date' },
  { key: 'esc', label: 'back' },
];
```

#### 3. Add new context-specific shortcut arrays

```typescript
// When any modal/form blocks input
const modalShortcuts: Shortcut[] = [
  { key: 'esc', label: 'close' },
];

// When help modal is open (? also closes it)
const helpModalShortcuts: Shortcut[] = [
  { key: 'esc', label: 'close' },
  { key: '?', label: 'close' },
];

// When in multi-select mode (sheet view)
const multiSelectShortcuts: Shortcut[] = [
  { key: 'j/k', label: 'nav' },
  { key: 'space', label: 'toggle select' },
  { key: 'J', label: 'join' },
  { key: 'd d', label: 'delete' },
  { key: 'v', label: 'exit select' },
  { key: 'esc', label: 'cancel' },
];
```

#### 4. Replace getShortcuts function (lines 57-71)

```typescript
const getShortcuts = (): Shortcut[] => {
  // Priority 1: Command palette open
  if (showCommandPalette) return modalShortcuts;

  // Priority 2: Help modal open
  if (showHelpModal) return helpModalShortcuts;

  // Priority 3: Node creation mode
  if (nodeCreationMode) return modalShortcuts;

  // Priority 4: Action form open
  if (activeActionForm) return modalShortcuts;

  // Priority 5: Template picker open
  if (showTemplatePicker) return modalShortcuts;

  // Priority 6: Multi-select mode (sheet view only)
  if (multiSelectMode && view === 'sheet') return multiSelectShortcuts;

  // Priority 7: Node detail panel
  if (showNodeDetail) return nodeDetailShortcuts;

  // Priority 8: View-based shortcuts
  switch (view) {
    case 'dashboard':
      return dashboardShortcuts;
    case 'jobs':
      return jobsShortcuts;
    case 'job':
      return jobShortcuts;
    case 'sheet':
      return sheetShortcuts;
    default:
      return dashboardShortcuts;
  }
};
```

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd millflow && npm run build`
- [ ] Linting passes: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Open command palette (Cmd+K) - bar shows only `esc close`
- [ ] Open help modal (?) - bar shows `esc close` and `? close`
- [ ] Press `o` to enter node creation mode - bar shows only `esc close`
- [ ] Open a form (a/b/n) - bar shows only `esc close`
- [ ] Open template picker (t) - bar shows only `esc close`
- [ ] Enter multi-select mode (v) - bar shows multi-select shortcuts
- [ ] Navigate views and confirm displayed shortcuts match actual behavior

---

## Testing Strategy

### Manual Testing Steps:
1. Start dev server: `cd millflow && npm run dev`
2. Test each context state transition:
   - Dashboard → press Cmd+K → verify bar shows `esc close`
   - Close palette → verify bar returns to dashboard shortcuts
   - Navigate to sheet → press `v` → verify multi-select shortcuts appear
   - Press `a` → verify bar shows `esc close` (form open)
   - Press Escape → verify bar returns to sheet shortcuts

## References

- Research document: `thoughts/shared/research/2025-12-19-millflow-keyboard-shortcuts-display.md`
- Keyboard handler: `millflow/src/hooks/useKeyboard.ts:68-141` (context priority)
- Store state definitions: `millflow/src/store/appStore.ts:24-32`
