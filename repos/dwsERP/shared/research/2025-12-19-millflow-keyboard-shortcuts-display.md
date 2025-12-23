---
date: 2025-12-19T18:03:27-08:00
researcher: ariasulin
git_commit: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
branch: main
repository: dwsERP
topic: "Millflow Keyboard Shortcuts Display - Current Implementation vs Actual Behavior"
tags: [research, codebase, millflow, keyboard, shortcuts, ui]
status: complete
last_updated: 2025-12-19
last_updated_by: ariasulin
---

# Research: Millflow Keyboard Shortcuts Display

**Date**: 2025-12-19T18:03:27-08:00
**Researcher**: ariasulin
**Git Commit**: b5a47c003fdf7cdb9c83da89a2fd88af831e2ea2
**Branch**: main
**Repository**: dwsERP

## Research Question
How does the keyboard shortcuts bar at the bottom of millflow work, and what are the discrepancies between what it displays vs what's actually implemented?

## Summary

The ShortcutBar component (`millflow/src/components/ShortcutBar.tsx`) displays contextual keyboard shortcuts at the bottom of the screen. Currently, it only considers two state variables for context:
1. `view` (dashboard, jobs, job, sheet)
2. `showNodeDetail` (boolean)

However, the actual keyboard handler (`millflow/src/hooks/useKeyboard.ts`) uses **eight different context levels** that determine which shortcuts are active. This mismatch causes the displayed shortcuts to be inaccurate in many situations.

## Detailed Findings

### Current ShortcutBar Implementation

**File**: `millflow/src/components/ShortcutBar.tsx`

The ShortcutBar reads three state values from appStore (line 55):
```typescript
const { view, showNodeDetail, pendingKeySequence } = useAppStore();
```

It then selects shortcuts based on a simple switch (lines 57-71):
1. If `showNodeDetail` → show `nodeDetailShortcuts`
2. Else based on `view`:
   - `dashboard` → `dashboardShortcuts`
   - `jobs` → `jobsShortcuts`
   - `job` → `jobShortcuts`
   - `sheet` → `sheetShortcuts`

### Static Shortcut Definitions in ShortcutBar

#### Dashboard Shortcuts (lines 9-16)
| Key | Label | Actually Implemented? |
|-----|-------|----------------------|
| `j/k` | navigate | Yes |
| `enter` | open | Yes |
| `f` | follow up | **No** - not in useKeyboard |
| `a` | assign | **No** - not in useKeyboard |
| `b` | blocker | **No** - not in useKeyboard |
| `?` | help | Yes |

#### Jobs Shortcuts (lines 18-24)
| Key | Label | Actually Implemented? |
|-----|-------|----------------------|
| `j/k` | navigate | Yes |
| `enter` | open job | Yes |
| `n` | new job | **No** - not in useKeyboard |
| `g h` | go home | Yes |
| `?` | help | Yes |

#### Job Shortcuts (lines 26-32)
| Key | Label | Actually Implemented? |
|-----|-------|----------------------|
| `j/k` | navigate | Yes |
| `enter` | open sheet | Yes |
| `n` | new sheet | **No** - uses `c` key, not `n` |
| `e` | edit job | **No** - not in useKeyboard |
| `esc` | back | Yes |

#### Sheet Shortcuts (lines 34-42)
| Key | Label | Actually Implemented? |
|-----|-------|----------------------|
| `j/k` | nav | Yes |
| `o` | after | Yes |
| `O` | before | Yes |
| `t` | template | Yes |
| `s` | split | Yes |
| `u` | undo | Yes |
| `?` | help | Yes |

#### Node Detail Shortcuts (lines 44-52)
| Key | Label | Actually Implemented? |
|-----|-------|----------------------|
| `a` | assign | Yes |
| `b` | blocker | Yes |
| `n` | note | Yes |
| `r` | reference | **No** - not in useKeyboard |
| `d` | due | **No** - uses `D` (Shift+d), not `d` |
| `f` | follow up | **No** - not in useKeyboard |
| `esc` | back | Yes |

### Missing Contexts from ShortcutBar

The keyboard handler checks these additional contexts that ShortcutBar does NOT consider:

#### 1. Command Palette Open (`showCommandPalette`)
When the command palette is open (Cmd+K):
- **Only Escape works** - all other shortcuts are blocked
- ShortcutBar shows normal view shortcuts (incorrect)

#### 2. Help Modal Open (`showHelpModal`)
When help modal is displayed (?):
- **Only Escape and ? work** - all other shortcuts are blocked
- ShortcutBar shows normal view shortcuts (incorrect)

#### 3. Node Creation Mode (`nodeCreationMode`)
When user presses `o` or `O` to insert a node:
- **Only Escape works** - waiting for node type selection
- ShortcutBar shows normal sheet shortcuts (incorrect)

#### 4. Active Action Form (`activeActionForm`)
When assign/note/blocker/duedate/newsheet form is open:
- **Only Escape works** - form captures keyboard input
- ShortcutBar shows normal shortcuts (incorrect)

#### 5. Template Picker Open (`showTemplatePicker`)
When template picker is displayed (t key):
- **Only Escape works** - waiting for template selection
- ShortcutBar shows normal sheet shortcuts (incorrect)

#### 6. Multi-Select Mode (`multiSelectMode`)
When multi-select is active (v key):
- **Space** toggles node selection (not status)
- **J** (Shift+j) joins selected nodes
- **d d** deletes multiple nodes
- ShortcutBar shows normal sheet shortcuts (missing multi-select specific actions)

#### 7. Pending Key Sequence (`pendingKeySequence`)
When first key of sequence pressed (g, d):
- Shows "g..." or "d..." waiting indicator
- **Correctly implemented** - ShortcutBar does show this state (lines 78-83)

### Complete Actual Keyboard Shortcuts (from useKeyboard.ts)

#### Global (Always Work)
| Key | Action | Implementation Line |
|-----|--------|-------------------|
| `Cmd+K` / `Ctrl+K` | Toggle command palette | 74-78 |

#### Key Sequences (500ms timeout)
| Sequence | Action | View | Line |
|----------|--------|------|------|
| `g h` | Go home (dashboard) | Any | 156-160 |
| `g j` | Go to jobs | Any | 165-169 |
| `g g` | Jump to top | Any | 173-177 |
| `g d` | Go to deliveries | Any | 193-197 |
| `d d` | Delete node(s) | Sheet | 179-189 |

#### Single-Key Navigation
| Key | Action | View | Line |
|-----|--------|------|------|
| `j` / `ArrowDown` | Navigate down | Any | 210-214 |
| `k` / `ArrowUp` | Navigate up | Any | 216-220 |
| `h` / `ArrowLeft` | Navigate left (parallel) | Sheet | 253-258 |
| `l` / `ArrowRight` | Navigate right (parallel) | Sheet | 260-267 |
| `G` (Shift+g) | Jump to bottom | Any | 233-238 |
| `Enter` | Open selected / toggle detail | Any | 222-231 |
| `?` | Toggle help modal | Any | 90-94 |
| `Tab` | Next actionable item | Sheet | 377-397 |
| `Shift+Tab` | Previous actionable item | Sheet | 377-397 |

#### Node Operations (Sheet View Only)
| Key | Action | Requires | Line |
|-----|--------|----------|------|
| `o` | Insert node after | selectedNodeId | 269-274 |
| `O` | Insert node before | selectedNodeId | 276-283 |
| `s` | Split (create parallel) | selectedNodeId | 285-291 |
| `v` | Toggle multi-select mode | - | 293-298 |
| `J` (Shift+j) | Join nodes | multiSelectMode | 300-311 |
| `a` | Assign person | selectedNodeId | 313-318 |
| `n` | Add note | selectedNodeId | 320-326 |
| `b` | Add blocker | selectedNodeId | 328-334 |
| `D` (Shift+d) | Set due date | selectedNodeId | 336-342 |
| `t` | Template picker | selectedNodeId | 360-367 |
| `Space` | Toggle status / selection | selectedNodeId | 240-251 |

#### Job View Only
| Key | Action | Requires | Line |
|-----|--------|----------|------|
| `c` | Create new sheet | selectedJobId | 369-374 |

#### History
| Key | Action | Line |
|-----|--------|------|
| `u` | Undo (not with Cmd/Ctrl) | 345-350 |
| `Ctrl+r` | Redo | 352-359 |

#### Escape Key (Hierarchical)
1. Close command palette (if open)
2. Close help modal (if open)
3. Cancel node creation mode (if active)
4. Close action form (if active)
5. Close template picker (if open)
6. Exit multi-select mode (if active)
7. Navigate back (`goBack()`)

### HelpModal Shortcuts (for comparison)

**File**: `millflow/src/components/HelpModal.tsx`

The HelpModal shows a comprehensive list organized into categories (lines 10-67). Notable discrepancies with actual implementation:

| Shown in Help | Actually Implemented? |
|--------------|----------------------|
| `A` - Reassign | **No** |
| `B` - View all blockers | **No** |
| `u` - Unblock (resolve) | **No** - `u` is undo |
| `N` - View all notes | **No** |
| `r` - Add reference | **No** |
| `d` - Set due date | **No** - uses `D` (Shift+d) |
| `D` - Set duration | **No** |
| `f` - Follow up (AI) | **No** |
| `h` - View history | **No** - `h` navigates left |

## Code References

- `millflow/src/components/ShortcutBar.tsx:1-95` - Shortcut bar component
- `millflow/src/components/HelpModal.tsx:1-132` - Help modal with all shortcuts
- `millflow/src/hooks/useKeyboard.ts:1-458` - Actual keyboard handler
- `millflow/src/store/appStore.ts:21-135` - State definitions affecting context

## Architecture Documentation

### Context Priority Hierarchy in useKeyboard.ts

The keyboard handler evaluates context in this priority order:

```
1. Input field focused → Block all shortcuts
2. showCommandPalette → Only Escape works
3. showHelpModal → Only Escape/? work
4. nodeCreationMode → Only Escape works
5. activeActionForm → Only Escape works
6. showTemplatePicker → Only Escape works
7. Escape key special handling → Hierarchical close/back
8. Key sequences (g/d prefix) → Multi-key commands
9. View-specific shortcuts → Based on current view
```

### State Variables Affecting Context

From `appStore.ts`:

| State Variable | Type | Purpose |
|---------------|------|---------|
| `view` | `'dashboard' \| 'jobs' \| 'job' \| 'sheet' \| 'deliveries'` | Current view |
| `showCommandPalette` | `boolean` | Command palette visibility |
| `showNodeDetail` | `boolean` | Node detail panel visibility |
| `showHelpModal` | `boolean` | Help modal visibility |
| `showTemplatePicker` | `boolean` | Template picker visibility |
| `nodeCreationMode` | `'before' \| 'after' \| null` | Node insertion mode |
| `activeActionForm` | `'assign' \| 'note' \| 'blocker' \| 'duedate' \| 'newsheet' \| null` | Active form |
| `multiSelectMode` | `boolean` | Multi-node selection mode |
| `pendingKeySequence` | `string` | Partial key sequence |

## Open Questions

1. Should the ShortcutBar show context-specific shortcuts for modal states (command palette, help, forms)?
2. Should multi-select mode have its own shortcut set showing available multi-node operations?
3. Should unimplemented shortcuts (f, r, A, B, etc.) be added to keyboard handler or removed from display?
4. Should the help modal be generated from a single source of truth to avoid discrepancies?
