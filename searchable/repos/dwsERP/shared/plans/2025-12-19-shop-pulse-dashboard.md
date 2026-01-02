# Shop Pulse Dashboard Implementation Plan

## Overview
Implement the "Shop Pulse" Executive Dashboard for the Millwork Shop Scheduling System. This read-only view serves as an "Air Traffic Control" center for the CEO and Project Managers, visualizing the real-time flow of 200+ active sheets across the shop floor. It complements the "Factory Floor" DAG editor by providing macro-level visibility into bottlenecks, critical paths, and future load.

## Current State Analysis
- **Existing App**: A "Factory Floor" DAG editor (React + React Flow) exists in `src/App.tsx`.
- **Missing**: A high-level dashboard view.
- **Goal**: Demonstrate the full value proposition (Edit + Monitor) to stakeholders.

## Desired End State
- **Navigation**: A seamless switch between "Factory Floor" (Editor) and "Shop Pulse" (Dashboard).
- **Visuals**: A "Swimlane Flow" visualization showing sheets moving through stations (Drafting, CNC, Veneer, Assembly).
- **Metrics**: A "Weather Forecast" load chart and a "Critical Alerts" panel.
- **Aesthetic**: Consistent **Gruvbox Dark** theme with "Slick & Industrial" feel.

## Implementation Approach
1.  **Architecture**: Single Page Application with client-side routing (or simple state-based view switching for prototype).
2.  **Visuals**: Custom React components for Swimlanes; `recharts` for graphs; `framer-motion` for fluid particle effects.
3.  **Data**: Rich mock data generator to simulate realistic shop conditions (bottlenecks, urgent jobs).

## Phase 1: Dashboard Scaffolding & Routing
### Overview
Create the dashboard container and enable navigation between the Editor and the Dashboard.

### Changes Required
#### 1. Navigation & Routing
**File**: `src/App.tsx`
**Changes**:
- Introduce state `currentView: 'editor' | 'dashboard'`.
- Add a top-level `NavBar` or `ViewSwitcher`.
- Conditionally render `<Editor />` (refactored current App content) or `<Dashboard />`.

#### 2. Dashboard Shell
**File**: `src/Dashboard.tsx`
**Changes**:
- Create basic layout: Header, Main Content Area (Grid system).

### Success Criteria
#### Automated Verification:
- [ ] App compiles without errors
- [ ] Navigation state switches correctly

#### Manual Verification:
- [ ] User can toggle between "Factory Floor" and "Shop Pulse" views
- [ ] Both views render correctly within the Gruvbox theme

## Phase 2: The "Swimlane Flow" Visual
### Overview
The core visualization showing jobs flowing through stations.

### Changes Required
#### 1. Mock Data Generator
**File**: `src/data/mockShopData.ts`
**Changes**:
- Generate 50+ jobs with diverse attributes: `id`, `name`, `station`, `urgency` ('critical', 'high', 'normal', 'low'), `status`.

#### 2. Components
**File**: `src/components/dashboard/StationLane.tsx`
- Horizontal or vertical track representing a station.
- Shows capacity indicators (e.g., "CNC: 110% Load").

**File**: `src/components/dashboard/JobChip.tsx`
- Small, color-coded particle representing a sheet.
- `bg-gruvbox-red` for critical items.
- Tooltip details on hover.

**File**: `src/Dashboard.tsx`
- Assemble lanes into the main view.

### Success Criteria
#### Automated Verification:
- [ ] Mock data generator produces valid objects
- [ ] Components mount without crashing

#### Manual Verification:
- [ ] "Red" jobs pop out visually against the dark background
- [ ] Bottlenecks are visually apparent (e.g., a pile-up in the CNC lane)

## Phase 3: The "Weather Forecast" & Metrics
### Overview
Add analytical charts and alerts to answer "What's coming?".

### Changes Required
#### 1. Dependencies
- Install `recharts`.

#### 2. Load Forecast Chart
**File**: `src/components/dashboard/LoadChart.tsx`
- Bar chart showing scheduled hours vs capacity for the next 5 days.
- Use Gruvbox palette for bars.

#### 3. Alerts Panel
**File**: `src/components/dashboard/AlertsPanel.tsx`
- List of top 3 critical issues (e.g., "Job #402 blocked at Veneer").

### Success Criteria
#### Automated Verification:
- [ ] `recharts` installed and building

#### Manual Verification:
- [ ] Charts are legible and match the theme
- [ ] Alerts panel clearly communicates priority issues

## Phase 4: Polish & "The Sell"
### Overview
Add interactions and animations to make the prototype feel "alive" and impressive.

### Changes Required
#### 1. Dependencies
- Install `framer-motion`.

#### 2. Animations
- Animate `JobChip` entry (slide-in).
- Add "pulse" effect to critical items.
- Smooth transitions between dashboard sections.

### Success Criteria
#### Manual Verification:
- [x] UI feels responsive and fluid
- [x] "Critical" items have a subtle pulse or attention-grabbing effect
