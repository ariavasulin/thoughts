# Documentation Styling Enhancements Plan

## Overview

Enhance the appearance of the YouLab Docsify documentation without changing any content. Fix visual issues with table backgrounds and sidebar title alignment, plus add polish for a more cohesive dark theme.

## Current State Analysis

The documentation uses Docsify with a Gruvbox dark theme. Two specific issues were identified:

1. **YouLab title misalignment** - The `.app-name-link` title lacks consistent left padding with the menu items below it
2. **Table white backgrounds** - Table cells (`td`) have no explicit background, causing them to appear white/transparent instead of matching the dark theme

### Key Discoveries:
- CSS is inline in `docs/index.html:19-388`
- Uses Gruvbox dark palette with custom CSS variables
- Table `th` has background (`var(--bg1)`) but `td` does not
- Sidebar title uses `.app-name-link` class but lacks alignment styles

## Desired End State

After implementation:
- Tables have consistent dark backgrounds matching the theme
- YouLab sidebar title is properly aligned with menu items
- Overall appearance is polished and cohesive

### Verification:
- Visual inspection confirms tables have dark backgrounds
- YouLab title appears centered/aligned with menu structure
- No content changes made to any `.md` files

## What We're NOT Doing

- Changing any documentation content (`.md` files)
- Adding new pages or navigation items
- Changing the color scheme or fonts
- Adding new plugins or JavaScript

## Implementation Approach

Single phase of CSS-only changes to `docs/index.html`.

## Phase 1: CSS Styling Fixes

### Overview
Fix table backgrounds, improve sidebar title alignment, and add polish.

### Changes Required:

#### 1. Fix Table Cell Backgrounds
**File**: `docs/index.html`
**Lines**: ~213-216 (after `td` padding rule)

Add explicit background color to table cells:

```css
.markdown-section td {
  padding: 0.75rem 1rem;
  border-bottom: 1px solid var(--bg1);
  background-color: var(--bg);  /* Add this - matches content area */
}
```

#### 2. Fix Table Row Hover State
**File**: `docs/index.html`
**Lines**: ~218-220

The hover state is fine but could be more subtle:

```css
.markdown-section tr:hover td {
  background-color: var(--bg-soft);
}
```

No change needed - existing style is good.

#### 3. Improve Sidebar Title Alignment
**File**: `docs/index.html`
**Lines**: ~114-119 (`.app-name-link`)

Update to align with menu items:

```css
.app-name-link {
  font-family: var(--code-font-family);
  font-weight: 700;
  font-size: 1.3rem;
  color: var(--accent) !important;
  display: block;
  padding: 0.5rem 0.8rem;
  margin-bottom: 0.5rem;
}
```

Changes:
- Increase font size slightly (1.2rem → 1.3rem) for prominence
- Keep accent color as requested
- Add `display: block` and padding to match menu item alignment
- Add margin-bottom for spacing from search/nav

#### 4. Add Table Header Styling Enhancement
**File**: `docs/index.html`
**Lines**: ~204-210

Enhance table header for better visual separation:

```css
.markdown-section th {
  background-color: var(--bg1);
  color: var(--fg1);
  font-weight: 600;
  text-align: left;
  padding: 0.75rem 1rem;
  border-bottom: 2px solid var(--bg2);
}
```

No change needed - existing style is good.

#### 5. Alternating Row Colors
**Status**: Skipped per user request - keeping tables uniform.

### Success Criteria:

#### Automated Verification:
- [ ] HTML file is valid (can be loaded in browser)
- [ ] No syntax errors in CSS

#### Manual Verification:
- [ ] Tables display with dark backgrounds (no white)
- [ ] YouLab title aligns with menu items below it
- [ ] Hover states on table rows work correctly
- [ ] Overall appearance is polished and consistent
- [ ] No content changes in any markdown files

---

## Summary of Changes

All changes are in `docs/index.html` CSS section:

| Issue | Fix | Lines |
|-------|-----|-------|
| Table white background | Add `background-color: var(--bg)` to `td` | ~216-220 |
| Title misalignment | Add padding and display:block to `.app-name-link` | ~114-122 |

Total: ~6 lines of CSS changes.

## Implementation Status: COMPLETE
