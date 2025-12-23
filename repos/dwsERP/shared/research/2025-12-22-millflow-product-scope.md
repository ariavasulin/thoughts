---
date: 2025-12-22T11:38:41-08:00
researcher: ariasulin
git_commit: 56c02369e91bfd15c379d9b7df2524340d80e7a2
branch: main
repository: dwsERP
topic: "MillFlow Product Scope - Features Built vs. Not Built"
tags: [research, product, scope, features, millflow]
status: complete
last_updated: 2025-12-22
last_updated_by: ariasulin
---

# Research: MillFlow Product Scope - Features Built vs. Not Built

**Date**: 2025-12-22T11:38:41-08:00
**Researcher**: ariasulin
**Git Commit**: 56c02369e91bfd15c379d9b7df2524340d80e7a2
**Branch**: main
**Repository**: dwsERP

## Research Question
Comprehensive state of affairs focusing on product specs and features - what has been built and what has not been built. A current scope document.

## Executive Summary

MillFlow is a keyboard-first PM dashboard for millwork production that models manufacturing workflows as DAGs (directed acyclic graphs). The application is **frontend-only with sophisticated mock data** - no backend, persistence, or AI integration exists yet.

**Core Value Proposition**: PMs manage ~200 sheets at any time. The insight is that manufacturing workflows are DAGs (diverge through parallel operations, reconverge at assembly), not linear stages. The PM's job is "DAG orchestration and unblocking."

---

## Features BUILT

### 1. Views / Pages

| View | Status | Description |
|------|--------|-------------|
| **Dashboard** | Complete | Home screen with 4 priority queues (Your Move, At Risk, Waiting On Others, Flowing) |
| **Jobs List** | Complete | Master list with live stats (progress %, blocker count, sheet count) |
| **Job Detail** | Complete | Job metadata, sheet list, upcoming deliveries, blockers across sheets |
| **Sheet/DAG Editor** | Complete | Full DAG visualization with keyboard navigation, node detail sidebar |
| **Deliveries** | Partial | Display-only calendar of upcoming deliveries; no drill-down navigation |

### 2. DAG Editing Capabilities

| Feature | Status | Description |
|---------|--------|-------------|
| **Node insertion (before/after)** | Complete | `o j` / `o k` inserts nodes in sequence |
| **Parallel branches (split)** | Complete | `o l` / `o h` creates diverging branches |
| **Branch convergence (join)** | Complete | Visual mode `v` + select nodes + `J` joins to current |
| **Node deletion** | Complete | `dd` removes node and reconnects parents to children |
| **Node reordering** | Complete | `M k` / `M j` swaps node position within branch |
| **Workflow templates** | Complete | `t` opens template picker with 6 pre-built workflows |
| **Smart node creator** | Complete | Fuzzy search across 22 millwork operations with autocomplete |
| **Undo/Redo** | Complete | `u` / `Ctrl+r` with history snapshots |

### 3. Node Management

| Feature | Status | Description |
|---------|--------|-------------|
| **Node detail sidebar** | Complete | Resizable panel showing all node data |
| **Assignee management** | Complete | `a` opens picker with AI suggestions and load balancing |
| **Blocker creation** | Complete | `b` opens smart form with NLP parsing and templates |
| **Blocker resolution** | Complete | Click checkmark on active blocker to resolve |
| **Note management** | Complete | `n` adds notes; inline edit/delete in detail panel |
| **Due date picker** | Complete | `Shift+D` with quick buttons (tomorrow, 3 days, 1 week) |
| **Status toggling** | Complete | `Space` cycles through statuses |
| **Multi-select operations** | Complete | `v` enters visual mode for bulk assign/delete |

### 4. Keyboard-First UX

| Feature | Status | Description |
|---------|--------|-------------|
| **Vim navigation** | Complete | `j/k/h/l` for up/down/left/right |
| **Multi-key sequences** | Complete | `gg`, `dd`, `gh`, `gj`, `gd`, `o j`, `m a`, `' a` |
| **Command palette** | Complete | `Cmd+K` fuzzy search across jobs, sheets, people, actions |
| **Help modal** | Complete | `?` shows all keyboard shortcuts |
| **Shortcut bar** | Complete | Context-aware footer showing available actions |
| **Navigation marks** | Complete | `m {a-z}` sets mark, `' {a-z}` jumps to mark |
| **Tab navigation** | Complete | `Tab`/`Shift+Tab` jumps between actionable items |

### 5. Dashboard / PM Workflow

| Feature | Status | Description |
|---------|--------|-------------|
| **Your Move queue** | Complete | Items requiring PM action (assign, approve, respond, QC) |
| **At Risk queue** | Complete | Items with blockers/delays, includes AI suggestions |
| **Waiting On Others queue** | Complete | External dependencies with staleness tracking |
| **Flowing indicator** | Complete | Positive reinforcement showing smooth work |
| **Smart blocker parser** | Complete | NLP extraction of type, owner, date from natural language |
| **Blocker templates** | Complete | 8 slash-command templates (`/material`, `/approval`, etc.) |

### 6. Data Model

| Entity | Status | Description |
|--------|--------|-------------|
| **Jobs** | Complete | Full model with client, location, GC, priority, receiving info |
| **Sheets** | Complete | Full model with nodes array forming DAG |
| **Nodes** | Complete | Full model with status, type, assignees, blockers, notes, references, due dates, windows |
| **People** | Complete | Internal/external/department types with load tracking |
| **Blockers** | Complete | 5 types with owner, resolution tracking |
| **Notes** | Complete | With author, source (manual/AI/email), timestamps |

---

## Features NOT BUILT

### 1. Backend / Infrastructure

| Feature | Status | Notes |
|---------|--------|-------|
| **Backend API** | Not Built | Frontend-only, no server |
| **Database** | Not Built | Data in memory, resets on refresh |
| **Authentication** | Not Built | No login/users |
| **Real-time sync** | Not Built | No WebSocket/SSE |
| **Deployment** | Not Built | No production infrastructure |

### 2. AI Integration

| Feature | Status | Notes |
|---------|--------|-------|
| **Email monitoring** | Not Built | Would auto-move items to "Your Move" |
| **Staleness detection** | Not Built | Would auto-mark items as stale |
| **Shop floor integration** | Not Built | Would auto-advance DAG nodes |
| **Pattern learning** | Not Built | Would pre-suggest assignees |
| **AI-generated notes** | Mock Only | Mock data has "from AI" notes, no real AI |

### 3. Views / Features

| Feature | Status | Notes |
|---------|--------|-------|
| **Deliveries drill-down** | Not Built | Can view list but not navigate to nodes |
| **Shop Pulse dashboard** | Not Built | "Air Traffic Control" executive view for 200+ sheets |
| **Scheduling optimization** | Not Built | PyJobShop/OR-Tools integration for sequencing |
| **Constraint-based assignment** | Not Built | Boolean logic for worker assignment rules |

### 4. Data Operations

| Feature | Status | Notes |
|---------|--------|-------|
| **Data persistence** | Not Built | All changes lost on refresh |
| **Sheet archiving** | Not Built | Archive action exists but no persistence |
| **Job creation** | Not Built | Jobs are mock data only |
| **Person management** | Not Built | People are mock data only |

### 5. External Integrations

| Feature | Status | Notes |
|---------|--------|-------|
| **Email integration** | Not Built | References show email links but no fetch |
| **Document storage** | Not Built | References are mock URLs |
| **Calendar sync** | Not Built | Due dates don't sync externally |
| **Notification system** | Not Built | No alerts/push notifications |

---

## Technical Stack

| Layer | Technology | Status |
|-------|------------|--------|
| **Framework** | React 19 | Production |
| **Language** | TypeScript (strict) | Production |
| **Build** | Vite | Production |
| **Styling** | Tailwind + Gruvbox theme | Production |
| **State** | Zustand (single store, 1200+ lines) | Production |
| **DAG Layout** | Custom BFS algorithm | Production |
| **Backend** | None | Not started |

---

## Mock Data Scope

The application ships with realistic mock data for demonstration:

- **4 Jobs**: Marriott Lobby, Chase SF, Apple Park Cafe, WeWork Oakland
- **~15 Sheets**: Distributed across jobs with varying complexity
- **~100 Nodes**: Various types (shop, external, material, approval, QC, delivery, install)
- **~20 People**: Mix of internal staff, external vendors, departments
- **Dashboard Items**: Pre-populated Your Move, At Risk, Waiting On Others queues

---

## File Structure

```
millflow/src/
├── views/           # 5 page-level components
├── components/      # 15+ UI components
│   ├── dag/         # DAG rendering (5 files)
│   ├── node/        # Node management (7 files)
│   ├── dashboard/   # Dashboard sections (2 files)
│   └── job/         # Job components (1 file)
├── store/           # Zustand store (1 file, 1200+ lines)
├── hooks/           # Keyboard handler (1 file, 450+ lines)
├── lib/             # Utilities (4 files)
├── types/           # TypeScript interfaces (1 file)
└── data/            # Mock data and templates (4 files)
```

---

## Summary: What Works End-to-End

A PM can currently:

1. **Open the app** → See prioritized dashboard of work
2. **Navigate with keyboard** → Never touch the mouse
3. **Drill into any job/sheet** → Full DAG visualization
4. **Edit DAGs** → Insert, delete, split, join, reorder nodes
5. **Manage nodes** → Assign people, add blockers/notes, set dates
6. **Use templates** → Insert pre-built workflows
7. **Use marks** → Bookmark and jump between nodes
8. **Multi-select** → Batch operations on multiple nodes
9. **Undo mistakes** → Full history with undo/redo

**What's missing**: Saving anything. All changes are lost on page refresh. The app is a complete frontend prototype awaiting backend integration.

---

## Related Documents

- `thoughts/shared/plans/2025-12-19-millflow-remaining-features.md` - Implementation phases
- `thoughts/shared/plans/2025-12-19-shop-pulse-dashboard.md` - Executive dashboard spec
- `thoughts/shared/plans/2025-12-19-assignment-constraint-system.md` - Constraint assignment spec

## Code References

- `millflow/src/views/` - All view components
- `millflow/src/store/appStore.ts` - Complete state management
- `millflow/src/hooks/useKeyboard.ts` - All keyboard shortcuts
- `millflow/src/data/mockData.ts` - Mock data structure
