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
4. **Docs/ directory** - User-facing documentation site using Docsify
5. **Development configs** - Next.js 15, React 19, TypeScript, Tailwind CSS 4, shadcn/ui
6. **Linear integration** - `.linear.toml` for project management

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

### User Documentation (Docsify)
1. `Docs/index.html` - Customize name, theme colors
2. `Docs/_sidebar.md` - Update navigation structure
3. `Docs/.nojekyll` - Copy as-is (empty file)
4. `Docs/README.md` - Project homepage
5. `Docs/*.md` - Create pages matching sidebar

### Directory Structure to Create
```
project/
├── .claude/
│   ├── agents/
│   └── commands/
├── Docs/                  # Docsify documentation site
│   ├── index.html
│   ├── _sidebar.md
│   ├── .nojekyll
│   └── screenshots/
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

## Part 7: User-Facing Documentation (Docsify)

### 7.1 Directory Structure

```
Docs/
├── index.html         # Docsify entry point with config
├── _sidebar.md        # Navigation sidebar
├── .nojekyll          # Prevents GitHub Pages Jekyll processing
├── README.md          # Homepage/landing page
├── Architecture.md    # System overview
├── Authentication.md  # Auth documentation
├── Receipts.md        # Receipt workflow
├── Database.md        # Supabase setup
├── API.md             # REST endpoints
├── Admin-Features.md  # Admin user guide
├── Employee-Features.md # Employee user guide
├── Components.md      # UI component library
├── Configuration.md   # Environment setup
├── screenshots/       # UI screenshots
│   ├── admin-*.png
│   └── mobile-*.png
└── Archive/           # Historical docs
```

### 7.2 Docsify Configuration (index.html)

```javascript
window.$docsify = {
  name: 'Project Name',
  repo: '',
  loadSidebar: true,
  subMaxLevel: 3,
  auto2top: true,
  search: {
    placeholder: 'Search docs...',
    noData: 'No results',
    depth: 3
  },
  copyCode: {
    buttonText: 'Copy',
    successText: 'Copied!'
  },
  pagination: {
    crossChapter: true,
    crossChapterText: true
  },
  // Wikilink support [[Page]] → [Page](Page.md)
  plugins: [
    function(hook) {
      hook.beforeEach(function(content) {
        content = content.replace(/\[\[([^\]|]+)\|([^\]]+)\]\]/g, '[$2]($1.md)');
        content = content.replace(/\[\[([^\]]+)\]\]/g, '[$1]($1.md)');
        return content;
      });
    }
  ]
}
```

### 7.3 Theme: Gruvbox Dark Terminal

Custom CSS using Gruvbox color palette with JetBrains Mono font:

```css
:root {
  /* Gruvbox palette */
  --gruvbox-bg-hard: #1d2021;
  --gruvbox-bg: #282828;
  --gruvbox-fg: #ebdbb2;

  /* App accent color */
  --dws-blue: #60a5fa;

  /* Typography */
  --base-font-family: 'JetBrains Mono', monospace;
}
```

### 7.4 Plugins Used

```html
<!-- Core -->
<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/docsify.min.js"></script>

<!-- Plugins -->
<script src="https://cdn.jsdelivr.net/npm/docsify@4/lib/plugins/search.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/docsify-copy-code@2"></script>
<script src="https://cdn.jsdelivr.net/npm/docsify-pagination@2/dist/docsify-pagination.min.js"></script>

<!-- Syntax highlighting -->
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-typescript.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/prismjs@1/components/prism-bash.min.js"></script>
```

### 7.5 Sidebar Structure (_sidebar.md)

```markdown
- **Getting Started**
  - [Overview](README.md)
  - [Architecture](Architecture.md)
  - [Configuration](Configuration.md)

- **Core Features**
  - [Authentication](Authentication.md)
  - [Receipts](Receipts.md)

- **User Guides**
  - [Employee Features](Employee-Features.md)
  - [Admin Features](Admin-Features.md)

- **Reference**
  - [API](API.md)
  - [Database](Database.md)
  - [Components](Components.md)
```

### 7.6 Page Template Pattern

```markdown
# Page Title

[[README|← Back to Index]]

## Overview

Brief description...

## Key Concept 1

| Column | Description |
|--------|-------------|
| Item 1 | Details |

## Key Concept 2

\`\`\`code
example
\`\`\`

## Related Pages

- [[OtherPage]] - Description
- [[AnotherPage]] - Description
```

### 7.7 Wikilink Syntax

Docsify is configured to support Obsidian-style wikilinks:
- `[[PageName]]` → Links to PageName.md with "PageName" text
- `[[PageName|Custom Text]]` → Links to PageName.md with "Custom Text"

### 7.8 Hosting

- **GitHub Pages**: Just push `Docs/` to repo, enable Pages on `/Docs` folder
- `.nojekyll` file prevents Jekyll processing (required for `_sidebar.md`)

---

## Key Patterns Summary

1. **CLAUDE.md is concise** - ~50 lines, references detailed docs
2. **Agents are specialists** - Each has one job, clear output format
3. **Commands are workflows** - Multi-step processes with checkpoints
4. **thoughts/ is structured** - Date prefixes, ticket references, clear categories
5. **Docs/ uses Docsify** - Gruvbox theme, wikilinks, search + copy code plugins
6. **Tailwind 4 uses CSS** - No config file, @theme in globals.css
7. **shadcn/ui new-york style** - Consistent component styling
8. **Scripts in hack/** - Helper utilities, whitelisted in settings.json
9. **Linear integration** - .linear.toml + custom CLI tools

## Code References

- `.claude/settings.json` - Project Claude settings
- `.claude/agents/codebase-locator.md` - Example agent definition
- `.claude/commands/create_plan.md` - Example command definition
- `Docs/index.html` - Docsify configuration with Gruvbox theme
- `Docs/_sidebar.md` - Documentation navigation
- `Docs/README.md` - Documentation homepage with changelog
- `Docs/Architecture.md` - Example documentation page
- `dws-app/package.json` - Dependencies and scripts
- `dws-app/tsconfig.json` - TypeScript configuration
- `dws-app/eslint.config.mjs` - ESLint flat config
- `dws-app/src/app/globals.css` - Tailwind 4 theme
- `dws-app/components.json` - shadcn/ui configuration
- `hack/spec_metadata.sh` - Metadata collection script
