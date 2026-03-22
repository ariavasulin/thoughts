---
date: 2026-01-05T12:08:40Z
researcher: Claude
git_commit: 75aace2c9407c27fda1af3a529b5847a7c102897
branch: main
repository: DWS-Receipts
topic: "Documentation Discrepancy Audit - CLAUDE.md vs Codebase"
tags: [research, documentation, discrepancy-audit, claude-md]
status: complete
last_updated: 2026-01-05
last_updated_by: Claude
---

# Research: Documentation Discrepancy Audit

**Date**: 2026-01-05T12:08:40Z
**Researcher**: Claude
**Git Commit**: 75aace2c9407c27fda1af3a529b5847a7c102897
**Branch**: main
**Repository**: DWS-Receipts

## Research Question

Compare documentation in `/Docs/*.md` and `CLAUDE.md` against the current codebase to identify discrepancies introduced since commit 9c97ecd (Dec 25, 2025).

## Summary

**`/Docs/*.md` documentation is accurate and up-to-date.** The comprehensive docs correctly document all current technologies including BAML, TanStack Query, and the hooks directory structure.

**`CLAUDE.md` contains discrepancies** and should be updated to match the authoritative `/Docs/` documentation:

| Area | CLAUDE.md Says | Codebase Has |
|------|----------------|--------------|
| OCR | Google Cloud Vision API | BAML + GPT-4.1-nano (OpenRouter) |
| State | Not mentioned | TanStack React Query v5 |
| Structure | Missing `hooks/` | `src/hooks/` with 5 custom hooks |
| Structure | Missing `baml_src/`, `baml_client/` | BAML source and generated client |

## Detailed Findings

### 1. OCR Technology Discrepancy

**CLAUDE.md (Line 10)**:
```markdown
- **OCR**: Google Cloud Vision API
```

**Actual Implementation**:
- Uses **BAML v0.214.0** with **GPT-4.1-nano** via **OpenRouter**
- Google Cloud Vision (`@google-cloud/vision`) is in `package.json` but **not used**
- Implementation at `dws-app/src/app/api/receipts/ocr/route.ts`
- BAML definitions at `dws-app/baml_src/receipts.baml`

**Docs/Architecture.md (correct)**:
```markdown
| OCR | BAML + GPT-4.1-nano (OpenRouter) |
```

### 2. State Management Discrepancy

**CLAUDE.md**: No mention of state management library

**Actual Implementation**:
- **TanStack React Query v5.90.12** installed
- **QueryProvider** at `src/components/providers/query-provider.tsx`
- Custom hooks for data fetching with caching
- 5-minute staleTime, 10-minute gcTime configuration

**Docs/Architecture.md (correct)**:
```markdown
| State | TanStack React Query |
```

### 3. Project Structure Discrepancy

**CLAUDE.md (Lines 16-26)**:
```markdown
dws-app/src/
├── app/           # Pages and API routes
├── components/    # React components (ui/ has shadcn components)
└── lib/           # Supabase clients, types, utils
```

**Actual Structure** (missing directories):
```
dws-app/src/
├── app/
├── components/
│   └── providers/     # MISSING from CLAUDE.md
├── hooks/             # MISSING from CLAUDE.md - 5 custom hooks
└── lib/

dws-app/
├── baml_src/          # MISSING from CLAUDE.md - BAML definitions
└── baml_client/       # MISSING from CLAUDE.md - Generated BAML client
```

**Docs/Architecture.md (correct)** shows complete structure including:
- `hooks/` directory
- `providers/` subdirectory
- `baml_src/` and `baml_client/`

### 4. Hooks Directory Contents

The `hooks/` directory contains 5 files not documented in CLAUDE.md:

| Hook File | Purpose |
|-----------|---------|
| `use-mobile.tsx` | Mobile viewport detection |
| `use-admin-prefetch.ts` | Background prefetch for batch-review/users |
| `use-admin-receipts.ts` | Admin receipt CRUD with React Query |
| `use-admin-users.ts` | Admin user management with React Query |
| `use-pending-receipts.ts` | Pending receipts for batch review |

### 5. Accurate Documentation in /Docs/

The following docs are **accurate** and reflect current implementation:

- **Docs/Architecture.md**: Correct tech stack (BAML, TanStack Query, full structure)
- **Docs/Receipts.md**: Correct OCR flow with BAML and GPT-4.1-nano
- **Docs/Components.md**: Lists `providers/query-provider.tsx` and hooks
- **Docs/Admin-Features.md**: References `hooks/use-admin-receipts.ts`

## Code References

- `CLAUDE.md:10` - Incorrect OCR reference
- `CLAUDE.md:16-26` - Incomplete project structure
- `Docs/Architecture.md:18` - Correct OCR reference
- `Docs/Architecture.md:21` - Correct TanStack Query reference
- `Docs/Architecture.md:42-43` - Correct hooks/baml directories
- `dws-app/src/app/api/receipts/ocr/route.ts:3-4` - BAML import
- `dws-app/package.json:25-26` - TanStack Query dependencies
- `dws-app/package.json:12` - BAML dependency

## Recommended CLAUDE.md Updates

### Tech Stack Section (Line 6-10)
```markdown
## Tech Stack

- **Framework**: Next.js 15 (App Router) + React 19 + TypeScript
- **Database/Auth/Storage**: Supabase (Postgres, SMS OTP auth via Twilio, file storage)
- **UI**: shadcn/ui + Tailwind CSS 4 + Lucide icons
- **OCR**: BAML + GPT-4.1-nano (OpenRouter)
- **State**: TanStack React Query
```

### Project Structure Section (Line 12-26)
```markdown
## Project Structure

All application code is in `dws-app/`:

\`\`\`
dws-app/
├── src/
│   ├── app/           # Pages and API routes
│   │   ├── api/       # Backend endpoints (auth, receipts, categories, admin)
│   │   ├── login/     # Phone OTP login
│   │   ├── employee/  # Employee receipt submission
│   │   ├── dashboard/ # Admin receipt management
│   │   └── batch-review/  # Admin batch approval UI
│   ├── components/    # React components
│   │   ├── ui/        # shadcn/ui primitives
│   │   └── providers/ # Context providers (React Query)
│   ├── hooks/         # Custom React hooks (data fetching)
│   └── lib/           # Supabase clients, types, utils
├── baml_src/          # BAML AI definitions
└── baml_client/       # Generated BAML client
\`\`\`
```

## Open Questions

None - discrepancies are clearly identified.

## Related Research

None previously documented.
