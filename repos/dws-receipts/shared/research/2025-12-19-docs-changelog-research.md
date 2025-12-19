---
date: 2025-12-19T12:00:00-08:00
researcher: Claude
git_commit: 39459f208a391979774c4b5d32696c9b7fcd2afa
branch: test-branch
repository: DWS-Receipts
topic: "Changelog for Docs Homepage"
tags: [research, documentation, changelog]
status: complete
last_updated: 2025-12-19
last_updated_by: Claude
---

# Research: Changelog for Docs Homepage

**Date**: 2025-12-19
**Researcher**: Claude
**Git Commit**: 39459f208a391979774c4b5d32696c9b7fcd2afa
**Branch**: test-branch
**Repository**: DWS-Receipts

## Research Question

Analyze recent commits to understand project updates and determine where to add a changelog in the documentation homepage.

## Summary

The project has 50+ commits spanning from May 31, 2025 (prototype) to December 19, 2025 (current). The documentation uses **Docsify** with `README.md` as the homepage. A changelog section should be added at the bottom of `Docs/README.md`, just before the "Last updated" line.

## Detailed Findings

### Documentation Structure

The docs use Docsify with:
- `Docs/index.html` - Main entry point with Gruvbox terminal theme
- `Docs/README.md` - Homepage content (what users see first)
- `Docs/_sidebar.md` - Navigation menu

The README.md currently ends with a "Key Flows" section and a "Last updated" note. A changelog fits naturally between these.

### Commit History Analysis

**December 2025 (Major Updates)**
- Comprehensive documentation added
- Background prefetching for admin pages
- Security fix: Next.js upgrade for CVE-2025-66478
- TanStack Query caching for instant tab switching
- Mobile improvements: Drawer modals, better layout
- Auto-submit receipts when OCR extracts all fields
- Admin user management UI (ARI-15)
- OCR upgrade: BAML + GPT-4o-mini VLM (replaced Google Vision)
- Receipt editing for pending submissions

**November 2025**
- Phone number display in admin dashboard

**August 2025**
- Feedback form
- Preferred name support
- Export formatting improvements

**July 2025**
- Mobile file upload enhancements
- Server-side image preprocessing with Sharp

**June 2025**
- Bulk reimbursement flow
- Security improvements (credentials cleanup)

**May 2025 (Initial Launch)**
- Prototype integration
- Receipt dashboard
- OCR date/amount extraction
- Build and deployment fixes

### Recommended Changelog Location

Add a new `## Changelog` section in `Docs/README.md` at line 43, just before the `---` separator and "Last updated" note.

## Proposed Changelog Format

```markdown
## Changelog

### December 2025
- **OCR Upgrade**: Switched from Google Vision to BAML + GPT-4.1-nano for faster, more accurate extraction
- **Auto-Submit**: Receipts now auto-submit when OCR extracts all required fields
- **Admin User Management**: Full-page UI for managing employees (ARI-15)
- **Performance**: TanStack Query caching for instant tab switching
- **Mobile UX**: Drawer modals and improved layout
- **Receipt Editing**: Edit pending submissions before approval
- **Security**: Next.js 15.3.6 upgrade (CVE-2025-66478)
- **Documentation**: Comprehensive project docs

### November 2025
- Phone number display in admin dashboard

### August 2025
- Feedback form for employee submissions
- Preferred name support in user profiles
- Payroll-formatted CSV export

### July 2025
- Mobile file upload improvements
- Server-side image preprocessing for OCR

### June 2025
- Bulk reimbursement workflow
- Security hardening

### May 2025
- Initial release
- Receipt dashboard and batch review
- OCR integration
- SMS OTP authentication
```

## Code References

- `Docs/README.md` - Homepage content, add changelog at line 43
- `Docs/_sidebar.md` - Sidebar navigation (no changes needed)
- `Docs/index.html` - Docsify configuration (no changes needed)

## Architecture Documentation

Docsify renders README.md as the homepage. The `[[Page]]` wikilink syntax is converted to regular markdown links via a custom plugin in index.html.
