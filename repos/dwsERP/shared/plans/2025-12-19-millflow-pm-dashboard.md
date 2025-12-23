# MillFlow - AI-Native PM Dashboard Implementation Plan

## Overview

MillFlow is a keyboard-first, AI-native project management interface for millwork shop project managers. This is a **self-contained prototype** within the dwsERP monorepo.

### Key Requirements
- **Self-contained**: Lives in `millflow/` directory with its own Vite/React setup
- **Gruvbox Dark Theme**: Uses exact same Tailwind config as factory-floor
- **Icons over Emojis**: Use Lucide React icons for all visual indicators
- **Keyboard-first**: Superhuman-style aggressive keyboard shortcuts
- **No backend**: All state in memory with mock data

---

## Phase 1: Project Scaffolding

### 1.1 Create Vite + React + TypeScript Project

**Files to create:**
- `millflow/package.json` - Dependencies: react, react-dom, lucide-react, zustand, react-hotkeys-hook, clsx, tailwind-merge, cmdk
- `millflow/tsconfig.json` - Strict TypeScript config
- `millflow/vite.config.ts` - Standard Vite React config
- `millflow/tailwind.config.js` - Copy Gruvbox theme from factory-floor
- `millflow/postcss.config.js` - PostCSS with Tailwind/autoprefixer
- `millflow/index.html` - Entry HTML with JetBrains Mono font
- `millflow/src/main.tsx` - React entry point
- `millflow/src/index.css` - Tailwind imports + base styles

### 1.2 Core Type Definitions

**File: `millflow/src/types/index.ts`**

```typescript
// Status types
export type NodeStatus = 'not_started' | 'ready' | 'in_progress' | 'blocked' | 'done' | 'skipped';
export type BlockerType = 'internal' | 'external' | 'material' | 'approval' | 'site';
export type NodeType = 'shop' | 'external' | 'material' | 'approval' | 'qc' | 'delivery' | 'install';
export type PersonType = 'internal' | 'external' | 'department';

// Core entities
export interface Job { id: string; name: string; client: string; location: string; gc: string; gc_contact: { name: string; phone: string }; priority: 1|2|3|4|5; receiving_hours: string; receiving_days: string[]; status: 'active' | 'completed'; }
export interface Sheet { id: string; job_id: string; name: string; description: string; created_at: string; nodes: Node[]; }
export interface Node { id: string; name: string; type: NodeType; status: NodeStatus; assignees: string[]; blockers: Blocker[]; notes: Note[]; references: Reference[]; due_date?: string; duration_estimate?: number; duration_actual?: number; completed_at?: string; window?: { earliest: string; latest: string }; children: string[]; parents: string[]; suggested_assignees?: string[]; }
export interface Blocker { id: string; type: BlockerType; description: string; owner: string; expected_resolution?: string; created_at: string; resolved_at?: string; }
export interface Note { text: string; author: string; source?: string; created_at: string; }
export interface Reference { type: 'email' | 'document' | 'image' | 'link'; title: string; url: string; }
export interface Person { id: string; name: string; role: string; department: string | null; type: PersonType; load?: number; contact?: string; }

// Dashboard items
export interface DashboardItem { type: string; node_id?: string; sheet?: string; job?: string; title: string; subtitle: string; priority?: 'normal' | 'high'; severity?: 'low' | 'medium' | 'high'; suggestion?: string; owner?: string; waiting_since?: string; status?: string; }
```

---

## Phase 2: State Management

### 2.1 Zustand Store

**File: `millflow/src/store/appStore.ts`**

```typescript
interface AppState {
  // Navigation
  view: 'dashboard' | 'jobs' | 'job' | 'sheet';
  selectedJobId: string | null;
  selectedSheetId: string | null;
  selectedNodeId: string | null;
  selectedIndex: number; // For list navigation

  // UI State
  showCommandPalette: boolean;
  showNodeDetail: boolean;
  showHelpModal: boolean;
  editingNodeId: string | null;

  // Data
  jobs: Job[];
  sheets: Sheet[];
  people: Person[];
  dashboardState: DashboardState;

  // Actions
  setView: (view: AppState['view']) => void;
  selectJob: (id: string | null) => void;
  selectSheet: (id: string | null) => void;
  selectNode: (id: string | null) => void;
  navigateUp: () => void;
  navigateDown: () => void;
  openSelected: () => void;
  goBack: () => void;
  toggleCommandPalette: () => void;
  toggleNodeDetail: () => void;
  toggleHelp: () => void;

  // Node actions
  addNode: (sheetId: string, afterNodeId: string | null, node: Partial<Node>) => void;
  deleteNode: (sheetId: string, nodeId: string) => void;
  updateNode: (sheetId: string, nodeId: string, updates: Partial<Node>) => void;
  assignNode: (sheetId: string, nodeId: string, personIds: string[]) => void;
  addBlocker: (sheetId: string, nodeId: string, blocker: Partial<Blocker>) => void;
  addNote: (sheetId: string, nodeId: string, note: Partial<Note>) => void;
}
```

---

## Phase 3: Mock Data

### 3.1 Complete Mock Data

**File: `millflow/src/data/mockData.ts`**

Implement the full mock data from the PRD:
- 5 jobs (Marriott Lobby, Chase Bank SF, Salesforce Tower L40, WeWork Oakland, Palo Alto Medical)
- 17 people (shop floor, PM, departments, vendors)
- Multiple sheets per job with complete DAG structures
- Pre-computed dashboard state (your_move, at_risk, waiting_on_others, flowing)

---

## Phase 4: Core Components

### 4.1 Layout Shell

**File: `millflow/src/components/Layout.tsx`**
- Header with app name, date, current user
- Command palette trigger (Cmd+K indicator)
- Main content area
- Footer shortcut bar (contextual based on view)

### 4.2 Shortcut Bar

**File: `millflow/src/components/ShortcutBar.tsx`**
- Displays contextual shortcuts based on current view
- Uses kbd styling for keyboard keys
- Automatically updates when view/selection changes

### 4.3 Command Palette

**File: `millflow/src/components/CommandPalette.tsx`**
- Uses cmdk library for fuzzy search
- Searches: jobs, sheets, nodes, people, actions, navigation
- Keyboard accessible (arrow keys, enter, esc)
- Groups results by type

### 4.4 Help Modal

**File: `millflow/src/components/HelpModal.tsx`**
- Full keyboard shortcut reference
- Organized by category (global, navigation, actions, editing)
- Dismissible with ? or Esc

---

## Phase 5: Dashboard View

### 5.1 Main Dashboard

**File: `millflow/src/views/DashboardView.tsx`**
- Four sections: Your Move, At Risk, Waiting On Others, Flowing
- Each section collapsible with item count
- Items are keyboard-navigable (j/k)
- Selected item highlighted
- Enter opens item detail/navigation

### 5.2 Dashboard Item Component

**File: `millflow/src/components/dashboard/DashboardItem.tsx`**
- Icon based on item type (using Lucide)
- Title and subtitle
- Status indicator (color-coded left border)
- Selected state styling

### 5.3 Dashboard Section

**File: `millflow/src/components/dashboard/DashboardSection.tsx`**
- Section header with title and count
- Collapse/expand functionality
- Contains list of DashboardItem components

---

## Phase 6: Jobs & Job View

### 6.1 Jobs List View

**File: `millflow/src/views/JobsView.tsx`**
- List of all jobs with status summary
- Priority indicators (stars)
- Quick stats (sheets count, blockers, next delivery)
- j/k navigation, Enter to open

### 6.2 Job Detail View

**File: `millflow/src/views/JobView.tsx`**
- Job header (name, client, location, GC contact, receiving info)
- Sheets section with status summaries
- Upcoming deliveries section
- Blockers on this job section
- j/k to navigate sheets, Enter to open sheet

### 6.3 Sheet List Item

**File: `millflow/src/components/job/SheetListItem.tsx`**
- Sheet name and description
- Progress indicator (X/Y nodes)
- Status icon (blocked, in progress, done)
- Due date
- Selected state

---

## Phase 7: Sheet DAG View

### 7.1 Sheet View Container

**File: `millflow/src/views/SheetView.tsx`**
- Header with breadcrumb (← Job Name)
- Sheet title and description
- Scrollable DAG area
- Contextual shortcut bar for DAG editing

### 7.2 DAG Renderer

**File: `millflow/src/components/dag/DAGRenderer.tsx`**
- Linearizes DAG for stack layout
- Handles parallel branches (shows side-by-side)
- Draws connection lines between nodes
- Manages node selection and focus

### 7.3 DAG Node Component

**File: `millflow/src/components/dag/DAGNode.tsx`**
- Node card with type-based styling
- Status indicator (left border color)
- Icon based on node type (Lucide icons)
- Assignee display
- Due date / completion date
- Blocker indicator
- Selected state with blue outline

### 7.4 Parallel Branch Container

**File: `millflow/src/components/dag/ParallelBranch.tsx`**
- Renders nodes that occur in parallel
- Horizontal layout with connecting lines
- Handles h/l navigation between branches

---

## Phase 8: Node Detail Panel

### 8.1 Node Detail Overlay

**File: `millflow/src/components/node/NodeDetailPanel.tsx`**
- Slide-in panel from right (or modal)
- Full node information display
- Status, due date, duration, priority
- Assigned section with suggestions
- Blockers list
- Notes list (chronological)
- References list
- History timeline
- Action shortcuts at bottom

### 8.2 Assignee Picker

**File: `millflow/src/components/node/AssigneePicker.tsx`**
- Dropdown/popover for assignment
- Shows suggested first, then all people
- Filterable by typing
- Multi-select with comma
- Shows person's current load

### 8.3 Blocker Form

**File: `millflow/src/components/node/BlockerForm.tsx`**
- Inline form for adding blockers
- Type selector (dropdown)
- Description input
- Owner input
- Expected resolution date picker

### 8.4 Note Form

**File: `millflow/src/components/node/NoteForm.tsx`**
- Inline textarea for adding notes
- Auto-attributes to current user
- Timestamps automatically

---

## Phase 9: Keyboard Handling

### 9.1 Keyboard Manager

**File: `millflow/src/hooks/useKeyboard.ts`**

Implement layered keyboard handling:

**Global (always active):**
- `Cmd+K` / `Ctrl+K`: Toggle command palette
- `g h`: Go home (dashboard)
- `g j`: Go to jobs
- `?`: Toggle help
- `Esc`: Back/close/cancel

**View-level (dashboard, jobs, job, sheet):**
- `j` / `↓`: Move down
- `k` / `↑`: Move up
- `Enter`: Open selected
- `Esc`: Go back
- `g g`: Jump to top
- `G`: Jump to bottom
- `Tab`: Next actionable item
- `Shift+Tab`: Previous actionable item

**Sheet DAG view:**
- `h` / `←`: Move left (parallel branch)
- `l` / `→`: Move right (parallel branch)
- `o`: Insert node after
- `O`: Insert node before
- `s`: Split (create parallel branch)
- `J`: Join (set convergence)
- `d d`: Delete node
- `z`: Toggle collapse

**Node actions (when node selected):**
- `a`: Assign
- `b`: Add blocker
- `n`: Add note
- `d`: Set due date
- `Space`: Toggle status
- `Enter`: Open detail panel

### 9.2 Sequence Key Handler

**File: `millflow/src/hooks/useKeySequence.ts`**
- Handles multi-key sequences (g h, g j, d d, etc.)
- Timeout for sequence reset (500ms)
- Visual indicator for pending sequence

---

## Phase 10: DAG Editing

### 10.1 Node Creation

**File: `millflow/src/components/dag/NodeCreator.tsx`**
- Inline node creation UI
- Auto-focus name input
- Type selector with quick keys (/cnc, /assy, etc.)
- Enter to save, Esc to cancel
- Inserts into DAG structure properly

### 10.2 Template System

**File: `millflow/src/data/templates.ts`**
- Predefined node sequences
- Standard shop workflow template
- Insert template at cursor position

---

## Phase 11: UI Polish

### 11.1 Icons Mapping

**File: `millflow/src/lib/icons.ts`**

Map node types and statuses to Lucide icons:
```typescript
import { Wrench, Globe, Package, Hand, CheckCircle, Truck, Hammer, Circle, CircleDot, Clock, Lock, Check, XCircle } from 'lucide-react';

export const nodeTypeIcons = {
  shop: Wrench,
  external: Globe,
  material: Package,
  approval: Hand,
  qc: CheckCircle,
  delivery: Truck,
  install: Hammer,
};

export const statusIcons = {
  not_started: Circle,
  ready: CircleDot,
  in_progress: Clock,
  blocked: Lock,
  done: Check,
  skipped: XCircle,
};
```

### 11.2 Styling Utilities

**File: `millflow/src/lib/styles.ts`**
- Status color mapping (using Gruvbox)
- Node type background colors
- Common component classes

---

## Phase 12: Integration & Testing

### 12.1 App Entry Point

**File: `millflow/src/App.tsx`**
- Initialize store with mock data
- Set up keyboard listeners
- Render Layout with current view
- Command palette overlay
- Help modal overlay

### 12.2 View Router

Simple view switching based on store state:
- `dashboard` → DashboardView
- `jobs` → JobsView
- `job` → JobView
- `sheet` → SheetView

---

## File Structure

```
millflow/
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
├── postcss.config.js
├── index.html
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── index.css
    ├── types/
    │   └── index.ts
    ├── store/
    │   └── appStore.ts
    ├── data/
    │   ├── mockData.ts
    │   └── templates.ts
    ├── lib/
    │   ├── icons.ts
    │   └── styles.ts
    ├── hooks/
    │   ├── useKeyboard.ts
    │   └── useKeySequence.ts
    ├── components/
    │   ├── Layout.tsx
    │   ├── ShortcutBar.tsx
    │   ├── CommandPalette.tsx
    │   ├── HelpModal.tsx
    │   ├── dashboard/
    │   │   ├── DashboardSection.tsx
    │   │   └── DashboardItem.tsx
    │   ├── job/
    │   │   └── SheetListItem.tsx
    │   ├── dag/
    │   │   ├── DAGRenderer.tsx
    │   │   ├── DAGNode.tsx
    │   │   ├── ParallelBranch.tsx
    │   │   └── NodeCreator.tsx
    │   └── node/
    │       ├── NodeDetailPanel.tsx
    │       ├── AssigneePicker.tsx
    │       ├── BlockerForm.tsx
    │       └── NoteForm.tsx
    └── views/
        ├── DashboardView.tsx
        ├── JobsView.tsx
        ├── JobView.tsx
        └── SheetView.tsx
```

---

## Success Criteria

### Automated Verification
- `cd millflow && npm run build` completes without errors
- `cd millflow && npm run lint` passes
- TypeScript strict mode with no errors

### Manual Verification
- Navigate entire app with keyboard only (no mouse required)
- j/k navigation works in all list views
- Enter opens items, Esc goes back
- Cmd+K opens command palette with fuzzy search
- ? shows help modal with all shortcuts
- g h goes to dashboard, g j goes to jobs
- In sheet view, h/l navigates parallel branches
- Node detail panel shows full information
- Assignment dropdown works with filtering
- Visual design matches Gruvbox dark theme
- All icons render (no emoji fallbacks)
- Shortcut bar updates contextually

---

## Implementation Order

1. **Phase 1**: Project scaffolding (Vite, Tailwind, types)
2. **Phase 2**: Store setup with mock data
3. **Phase 3**: Complete mock data implementation
4. **Phase 4**: Layout shell, shortcut bar, help modal
5. **Phase 5**: Dashboard view with sections
6. **Phase 6**: Jobs list and job detail views
7. **Phase 7**: Sheet DAG view with node rendering
8. **Phase 8**: Node detail panel
9. **Phase 9**: Full keyboard handling
10. **Phase 10**: DAG editing (node creation)
11. **Phase 11**: Command palette integration
12. **Phase 12**: Polish and verification

---

## Dependencies

```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "zustand": "^5.0.0",
    "cmdk": "^1.0.0",
    "lucide-react": "^0.460.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.0.0"
  }
}
```

Note: Using cmdk for command palette instead of react-hotkeys-hook. Keyboard handling will be custom with native event listeners for more control over sequences.
