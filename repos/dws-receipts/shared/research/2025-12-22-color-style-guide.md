---
date: 2025-12-22T11:09:54-08:00
researcher: ariasulin
git_commit: a82a08c74be48153d3eb86ba41722e5182f5eae7
branch: test-branch
repository: DWS-Receipts
topic: "Color and Style Guide / Theme Breakdown"
tags: [research, codebase, theming, tailwind, shadcn-ui, colors]
status: complete
last_updated: 2025-12-22
last_updated_by: ariasulin
---

# Research: Color and Style Guide / Theme Breakdown

**Date**: 2025-12-22T11:09:54-08:00
**Researcher**: ariasulin
**Git Commit**: a82a08c74be48153d3eb86ba41722e5182f5eae7
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question
Concise breakdown of the color and style guide for this project (the theme).

## Summary

The project uses **Tailwind CSS v4** with an inline theme configuration in CSS, following the **shadcn/ui "new-york" style**. Colors are defined in **OKLCH color space** for perceptual uniformity. The app employs a **dark-first design** with hardcoded hex values for the custom UI alongside semantic CSS variables for shadcn/ui components.

---

## Core Technology Stack

| Component | Technology |
|-----------|------------|
| CSS Framework | Tailwind CSS v4 (inline config via `@theme`) |
| Component Library | shadcn/ui (new-york style) |
| Icons | Lucide |
| Fonts | Geist Sans + Geist Mono |
| Color Space | OKLCH |

---

## Custom Hex Color Palette (Dark Theme)

The application uses a consistent dark gray scale with a blue accent:

### Background Hierarchy
```
#222222  →  Page backgrounds (darkest)
#2e2e2e  →  Card/modal backgrounds
#333333  →  Container/table backgrounds
#3e3e3e  →  Input field backgrounds
#444444  →  Header backgrounds, borders
#4e4e4e  →  Secondary borders
#555555  →  Hover states (lightest)
```

### Primary Brand Color
```
#2680FC  →  Primary blue (CTAs, active states)
#1a6fd8  →  Primary blue hover
```

### Text Colors
```
text-white      →  Primary text
text-gray-300   →  Secondary text
text-gray-400   →  Labels, placeholders
text-gray-500   →  Muted text
text-blue-400   →  Links, action icons
```

---

## Status Badge Colors

Receipt statuses use semi-transparent backgrounds with light text:

| Status | Background | Text | Border |
|--------|------------|------|--------|
| Pending | `bg-yellow-500/30` | `text-yellow-300` | `border-yellow-500/30` |
| Approved | `bg-green-500/30` | `text-green-300` | `border-green-500/30` |
| Rejected | `bg-red-500/30` | `text-red-300` | `border-red-500/30` |
| Reimbursed | `bg-blue-500/30` | `text-blue-300` | `border-blue-500/30` |

---

## Action Button Colors

| Action | Base | Hover | Text |
|--------|------|-------|------|
| Primary (Submit) | `bg-[#2680FC]` | `hover:bg-[#1a6fd8]` | `text-white` |
| Approve | `bg-green-600` | `hover:bg-green-700` | `text-white` |
| Reject/Delete | `bg-red-600` | `hover:bg-red-700` | `text-white` |
| Logout | `bg-red-500` | `hover:bg-red-600` | `text-white` |
| Ghost | `bg-[#333333]` | `hover:bg-[#444444]` | `text-white` |

---

## CSS Variables (shadcn/ui)

Defined in `dws-app/src/app/globals.css` using OKLCH color space:

### Light Mode (:root)
```css
--background: oklch(1 0 0);           /* White */
--foreground: oklch(0.145 0 0);       /* Near black */
--primary: oklch(0.205 0 0);          /* Dark gray */
--primary-foreground: oklch(0.985 0 0); /* Off-white */
--destructive: oklch(0.577 0.245 27.325); /* Red */
--border: oklch(0.922 0 0);           /* Light gray */
--radius: 0.625rem;                   /* 10px base */
```

### Dark Mode (.dark)
```css
--background: oklch(0.145 0 0);       /* Near black */
--foreground: oklch(0.985 0 0);       /* Off-white */
--primary: oklch(0.922 0 0);          /* Light gray */
--primary-foreground: oklch(0.205 0 0); /* Dark gray */
--destructive: oklch(0.704 0.191 22.216); /* Lighter red */
--border: oklch(1 0 0 / 10%);         /* White 10% opacity */
```

---

## Typography

### Font Families
- **Sans-serif**: Geist Sans (`--font-geist-sans`)
- **Monospace**: Geist Mono (`--font-geist-mono`)

Applied via CSS variables:
```css
--font-sans: var(--font-geist-sans);
--font-mono: var(--font-geist-mono);
```

---

## Border Radius Scale

Base radius: `0.625rem` (10px)

| Token | Size |
|-------|------|
| `--radius-sm` | 6px |
| `--radius-md` | 8px |
| `--radius-lg` | 10px |
| `--radius-xl` | 14px |

---

## Chart Colors

### Light Mode
```
--chart-1: oklch(0.646 0.222 41.116)   → Orange/yellow
--chart-2: oklch(0.6 0.118 184.704)    → Cyan
--chart-3: oklch(0.398 0.07 227.392)   → Dark blue
--chart-4: oklch(0.828 0.189 84.429)   → Yellow/green
--chart-5: oklch(0.769 0.188 70.08)    → Yellow
```

### Dark Mode
```
--chart-1: oklch(0.488 0.243 264.376)  → Purple
--chart-2: oklch(0.696 0.17 162.48)    → Cyan
--chart-3: oklch(0.769 0.188 70.08)    → Yellow
--chart-4: oklch(0.627 0.265 303.9)    → Magenta
--chart-5: oklch(0.645 0.246 16.439)   → Orange/red
```

---

## Code References

- `dws-app/src/app/globals.css:6-44` - Tailwind v4 `@theme inline` configuration
- `dws-app/src/app/globals.css:46-79` - Light mode CSS variables
- `dws-app/src/app/globals.css:81-113` - Dark mode CSS variables
- `dws-app/src/app/layout.tsx:7-15` - Font loading
- `dws-app/components.json` - shadcn/ui configuration (new-york style)
- `dws-app/src/components/receipt-table.tsx:174-190` - Status badge implementation
- `dws-app/src/components/ui/button.tsx:12-22` - Button variant definitions
- `dws-app/src/lib/utils.ts` - `cn()` class merge utility

---

## Configuration Files

| File | Purpose |
|------|---------|
| `dws-app/components.json` | shadcn/ui config (style: "new-york", baseColor: "neutral") |
| `dws-app/postcss.config.mjs` | PostCSS with `@tailwindcss/postcss` |
| `dws-app/src/app/globals.css` | All theme definitions |
| `dws-app/src/lib/utils.ts` | Tailwind class merging utility |

---

## Key Patterns

1. **Dark-first design**: UI built primarily for dark backgrounds using hardcoded hex values
2. **Dual color systems**: Custom hex palette for app UI + CSS variables for shadcn components
3. **OKLCH color space**: All CSS variables use OKLCH for perceptual consistency
4. **Semi-transparent status badges**: 30% opacity backgrounds with light text
5. **Consistent hover pattern**: Each background has a lighter hover state (+1 step in gray scale)
