# ARI-83: Remove rounded corners from diff view

**Status:** In Review
**Assignee:** Ariav Asulin
**Created:** 2026-01-14T09:56:36.851Z
**URL:** https://linear.app/ariav/issue/ARI-83/remove-rounded-corners-from-diff-view
**Branch:** ari/ari-83-remove-rounded-corners-from-diff-view

## Description

## Overview

Remove rounded corners from the diff view containers to create a cleaner, more code-editor-like appearance.

## Affected Components

1. **DiffApprovalOverlay.svelte** (line 225)
   * Main diff container has `rounded-lg` class
2. **BlockDetailModal.svelte** (line 269)
   * Diff container has `rounded-lg` class

## Implementation

Simple Tailwind class removal - change `rounded-lg` to remove border-radius from diff containers.

## Size

XS - Two line changes

## Links

- [Implementation Plan](https://github.com/humanlayer/YouLab/blob/main/thoughts/shared/plans/2026-01-14-ARI-83-remove-rounded-corners-diff-view.md)

## Comments

### Ariav Asulin (2026-01-14T09:57:24.475Z)

**Implementation Plan Created**

Plan: `thoughts/shared/plans/2026-01-14-ARI-83-remove-rounded-corners-diff-view.md`

XS task - two line changes to remove `rounded-lg` Tailwind classes from:
1. `DiffApprovalOverlay.svelte:225`
2. `BlockDetailModal.svelte:269`
