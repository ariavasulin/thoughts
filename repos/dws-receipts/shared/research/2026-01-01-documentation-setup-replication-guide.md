---
date: 2026-01-01T02:07:54-08:00
researcher: ariasulin
git_commit: 75aace2c9407c27fda1af3a529b5847a7c102897
branch: main
repository: DWS-Receipts
topic: "Documentation Setup for Replication in Other Projects"
tags: [research, documentation, claude-code, configuration, templates]
status: complete
last_updated: 2026-01-01
last_updated_by: ariasulin
---

# Research: Documentation Setup for Replication in Other Projects

**Date**: 2026-01-01T02:07:54-08:00
**Researcher**: ariasulin
**Git Commit**: 75aace2c9407c27fda1af3a529b5847a7c102897
**Branch**: main
**Repository**: DWS-Receipts

## Research Question

What documentation, configuration, tech stack, and style patterns are used in this project that can be replicated in another project?

## Summary

This project uses a comprehensive documentation and AI-assistant configuration system centered around:
1. **CLAUDE.md** - Main project context file for Claude Code
2. **.claude/ directory** - Claude Code settings, custom agents, and slash commands
3. **thoughts/ directory** - Structured knowledge management via HumanLayer
4. **Development configs** - Next.js 15, React 19, TypeScript, Tailwind CSS 4, shadcn/ui
5. **Linear integration** - `.linear.toml` for project management

---

## Part 1: Claude Code Configuration

### 1.1 CLAUDE.md (Project Root)

The main context file that Claude Code reads automatically. Structure:

```markdown
# Project Name

Brief description of the project.

## Tech Stack
- Bulleted list of technologies

## Project Structure
- Directory tree with descriptions

## Development
- Key commands (dev, build, lint)

## Key Patterns
- Important conventions and patterns

## External Integrations
- Linear, Supabase, etc.

## Detailed Documentation
- Links to other docs (.cursorrules, PRD, types)
```

**Key principle**: Keep CLAUDE.md concise (~50 lines). Reference detailed docs rather than duplicating.

### 1.2 .claude/ Directory Structure

```
.claude/
├── settings.json          # Project-level Claude settings
├── settings.local.json    # User-specific settings (gitignored)
├── fullProjectContext.md  # Extended documentation (optional)
├── agents/                # Custom sub-agent definitions
│   ├── codebase-locator.md
│   ├── codebase-analyzer.md
│   ├── codebase-pattern-finder.md
│   ├── thoughts-locator.md
│   ├── thoughts-analyzer.md
│   ├── web-search-researcher.md
│   └── claude-desktop-researcher.md
└── commands/              # Custom slash commands
    ├── create_plan.md
    ├── implement_plan.md
    ├── research_codebase.md
    ├── commit.md
    ├── describe_pr.md
    └── [many more...]
```

### 1.3 Settings Files

**settings.json** (project-level, committed):
```json
{
  "permissions": {
    "allow": [
      "Bash(./hack/spec_metadata.sh)",
      "Bash(hack/spec_metadata.sh)"
    ]
  },
  "enableAllProjectMcpServers": false,
  "env": {
    "MAX_THINKING_TOKENS": "32000"
  }
}
```

**settings.local.json** (user-specific, gitignored):
```json
{
  "permissions": {
    "allow": ["Bash(cat:*)"]
  },
  "enabledMcpjsonServers": ["linear"],
  "enableAllProjectMcpServers": true
}
```

### 1.4 Custom Agent Format

Agents are markdown files with YAML frontmatter:

```markdown
---
name: agent-name
description: Short description for Task tool
tools: Grep, Glob, LS, Read  # Available tools
model: sonnet  # or opus, haiku
color: yellow  # optional
---

You are a specialist at [purpose].

## Core Responsibilities
1. [Responsibility 1]
2. [Responsibility 2]

## Output Format
[Expected output structure]

## Important Guidelines
- [Guideline 1]
- [Guideline 2]
```

### 1.5 Custom Slash Command Format

Commands are markdown files with YAML frontmatter:

```markdown
---
description: Short description for command menu
model: opus  # optional, defaults to current
---

# Command Name

Instructions for what the command does when invoked.

## Process:
1. Step 1
2. Step 2
3. Step 3

## Important:
- Key consideration
- Another consideration
```

---

## Part 2: thoughts/ Directory (HumanLayer)

### 2.1 Directory Structure

```
thoughts/
├── CLAUDE.md              # Explains thoughts directory usage
├── searchable/            # Hard links for search (auto-managed)
│   ├── shared/            # Project-specific, team-shared
│   │   ├── handoffs/      # Work transfer documents
│   │   │   ├── general/   # General handoffs
│   │   │   └── ARI-XX/    # Ticket-specific handoffs
│   │   ├── plans/         # Implementation plans
│   │   ├── research/      # Investigation documents
│   │   └── tickets/       # Linear ticket notes
│   └── global/            # Cross-repository notes
│       └── shared/
│           ├── reference/ # Reusable reference docs
│           └── research/  # Cross-project research
```

### 2.2 Naming Conventions

**Handoff files**: `YYYY-MM-DD_HH-MM-SS_[TICKET-ID]_description.md`
- Example: `2025-12-17_13-38-00_ARI-11_receipt-editing-research.md`

**Plan files**: `YYYY-MM-DD-[TICKET-ID]-description.md`
- Example: `2025-12-17-ARI-15-baml-vlm-receipt-parsing.md`

**Research files**: `YYYY-MM-DD-description.md`
- Example: `2025-12-30-batch-approval-freeze-bug.md`

**Reference files**: `descriptive-name.md` (no date prefix)
- Example: `write-claude-md.md`

### 2.3 Research Document Template

```markdown
---
date: [ISO datetime with timezone]
researcher: [username]
git_commit: [commit hash]
branch: [branch name]
repository: [repo name]
topic: "[Research topic]"
tags: [research, component-names]
status: complete
last_updated: [YYYY-MM-DD]
last_updated_by: [username]
---

# Research: [Topic]

**Date**: [datetime]
**Researcher**: [name]
**Git Commit**: [hash]
**Branch**: [branch]
**Repository**: [repo]

## Research Question
[Original query]

## Summary
[High-level findings]

## Detailed Findings

### [Component 1]
- Finding with `file.ts:123` reference

### [Component 2]
...

## Code References
- `path/to/file.py:123` - Description

## Historical Context
[Links to related thoughts/ documents]

## Open Questions
[Areas needing further investigation]
```

---

## Part 3: Development Configuration

### 3.1 Package.json Key Patterns

```json
{
  "scripts": {
    "dev": "next dev --turbopack -H 0.0.0.0",
    "build": "NEXT_ESLINT_DISABLED=1 next build",
    "lint": "next lint"
  }
}
```

**Core dependencies pattern**:
- Framework: `next`, `react`, `react-dom`
- Backend: `@supabase/ssr`, `@supabase/supabase-js`
- UI: Radix primitives, `class-variance-authority`, `clsx`, `tailwind-merge`
- Icons: `lucide-react`
- Notifications: `sonner`

### 3.2 TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "jsx": "preserve",
    "noEmit": true,
    "incremental": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### 3.3 ESLint Configuration (Flat Config)

```javascript
// eslint.config.mjs
import { FlatCompat } from "@eslint/eslintrc";
import { dirname } from "path";
import { fileURLToPath } from "url";

const __dirname = dirname(fileURLToPath(import.meta.url));
const compat = new FlatCompat({ baseDirectory: __dirname });

export default [...compat.extends("next/core-web-vitals", "next/typescript")];
```

### 3.4 Tailwind CSS 4 Configuration

No `tailwind.config.ts` - configuration in CSS:

```css
/* globals.css */
@import "tailwindcss";
@import "tw-animate-css";

@custom-variant dark (&:is(.dark *));

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  /* ... other tokens */
  --font-sans: var(--font-geist-sans);
  --radius-lg: var(--radius);
}

:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
  /* ... light theme colors in OKLCH */
}

.dark {
  --background: oklch(0.145 0 0);
  /* ... dark theme colors */
}
```

### 3.5 shadcn/ui Configuration

```json
// components.json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/app/globals.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils",
    "ui": "@/components/ui",
    "lib": "@/lib",
    "hooks": "@/hooks"
  },
  "iconLibrary": "lucide"
}
```

### 3.6 Next.js Configuration

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  serverExternalPackages: ['@boundaryml/baml'],
  images: {
    remotePatterns: [{
      protocol: 'https',
      hostname: 'your-supabase.supabase.co',
      pathname: '/storage/v1/object/public/**'
    }]
  },
  typescript: { ignoreBuildErrors: true },
  eslint: { ignoreDuringBuilds: true }
};
```

### 3.7 Vercel Deployment

```json
// vercel.json
{
  "env": { "NEXT_ESLINT_DISABLED": "1" },
  "installCommand": "npm install --legacy-peer-deps"
}
```

---

## Part 4: Linear Integration

### 4.1 .linear.toml

```toml
# .linear.toml
workspace = "your-workspace"
team_id = "TEAM"
issue_sort = "manual"
```

### 4.2 Linear CLI (hack/linear/)

Custom TypeScript CLI for Linear operations:
- `list-issues` - Show assigned issues
- `get-issue [ID]` - View issue details
- `add-comment "message"` - Add comments
- `fetch-images ID` - Download attachments

---

## Part 5: Helper Scripts (hack/)

### 5.1 Essential Scripts

**spec_metadata.sh** - Collects metadata for documentation:
```bash
#!/bin/bash
set -euo pipefail
echo "Current Date/Time (TZ): $(date '+%Y-%m-%d %H:%M:%S %Z')"
echo "Current Git Commit Hash: $(git rev-parse HEAD)"
echo "Current Branch Name: $(git rev-parse --abbrev-ref HEAD)"
echo "Repository Name: $(basename "$(pwd)")"
echo "Timestamp For Filename: $(date '+%Y-%m-%d_%H-%M-%S')"
```

**create_worktree.sh** - Git worktree management with thoughts initialization

**cleanup_worktree.sh** - Worktree cleanup with thoughts cleanup

---

## Part 6: Files to Copy for New Project

### Required Files
1. `CLAUDE.md` - Customize for your project
2. `.claude/settings.json` - Adjust permissions
3. `.claude/agents/*.md` - Copy all, customize if needed
4. `.claude/commands/*.md` - Copy all, customize if needed
5. `.linear.toml` - Update workspace/team

### Development Config Files
1. `package.json` - Update name, adjust dependencies
2. `tsconfig.json` - Usually copy as-is
3. `eslint.config.mjs` - Usually copy as-is
4. `postcss.config.mjs` - Copy as-is for Tailwind 4
5. `components.json` - Adjust paths if different structure
6. `next.config.ts` - Adjust for your needs
7. `vercel.json` - If deploying to Vercel
8. `src/app/globals.css` - Copy Tailwind 4 theme setup

### Directory Structure to Create
```
project/
├── .claude/
│   ├── agents/
│   └── commands/
├── hack/
│   └── spec_metadata.sh
├── thoughts/
│   └── shared/
│       ├── handoffs/
│       ├── plans/
│       └── research/
├── src/
│   ├── app/
│   ├── components/
│   │   └── ui/
│   ├── lib/
│   └── hooks/
└── [config files]
```

---

## Key Patterns Summary

1. **CLAUDE.md is concise** - ~50 lines, references detailed docs
2. **Agents are specialists** - Each has one job, clear output format
3. **Commands are workflows** - Multi-step processes with checkpoints
4. **thoughts/ is structured** - Date prefixes, ticket references, clear categories
5. **Tailwind 4 uses CSS** - No config file, @theme in globals.css
6. **shadcn/ui new-york style** - Consistent component styling
7. **Scripts in hack/** - Helper utilities, whitelisted in settings.json
8. **Linear integration** - .linear.toml + custom CLI tools

## Code References

- `.claude/settings.json` - Project Claude settings
- `.claude/agents/codebase-locator.md` - Example agent definition
- `.claude/commands/create_plan.md` - Example command definition
- `dws-app/package.json` - Dependencies and scripts
- `dws-app/tsconfig.json` - TypeScript configuration
- `dws-app/eslint.config.mjs` - ESLint flat config
- `dws-app/src/app/globals.css` - Tailwind 4 theme
- `dws-app/components.json` - shadcn/ui configuration
- `hack/spec_metadata.sh` - Metadata collection script
