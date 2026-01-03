---
date: 2026-01-03T10:54:41+08:00
researcher: claude
git_commit: 2437760ec0a96d85c87f9beb2c72d12950157731
branch: main
repository: Dealtrail
topic: "TraceWriter JSON Export and State Persistence for Vercel Deployment"
tags: [research, tracewriter, export, persistence, vercel, collaboration]
status: complete
last_updated: 2026-01-03
last_updated_by: claude
---

# Research: TraceWriter JSON Export and State Persistence for Vercel Deployment

**Date**: 2026-01-03T10:54:41+08:00
**Researcher**: claude
**Git Commit**: 2437760ec0a96d85c87f9beb2c72d12950157731
**Branch**: main
**Repository**: Dealtrail

## Research Question

What needs to be implemented for JSON export with annotations, and what's required for state persistence when deploying to Vercel for team collaboration (data persists across users, sessions, refreshes)?

## Summary

The TraceWriter app currently has **working import** but **export is a placeholder** and **no persistence exists**. For Vercel team deployment, localStorage won't work - a backend storage solution is required.

### What EXISTS (Implemented)
1. TraceWriter UI with 3-tier hierarchy (Properties → Threads → Emails)
2. MBOX preprocessing to JSON
3. JSON import that parses both fresh and annotated formats
4. In-memory annotation storage
5. emailParser.js with full support for nested property schema

### What's MISSING (Not Implemented)
1. **Export functionality** - `handleExport()` at `App.jsx:212` is a stub
2. **Storage utilities** - no `storage.js` or `exportAnnotations.js` exists
3. **Persistence** - annotations lost on refresh
4. **Export prompt before new import** - no safeguard exists

### For Multi-User Vercel Deployment
localStorage is browser-local and **will not work** for team collaboration. Options:
- **Vercel KV** (Redis) - simple key-value, good for shared state
- **Supabase/Firebase** - real-time sync, user auth
- **Vercel Blob** - file storage for JSON exports
- **Simple JSON API** - minimal backend storing one JSON file

## Detailed Findings

### 1. Export Functionality Gap

**Current State** (`tracewriter/src/App.jsx:212-214`):
```javascript
const handleExport = () => {
  console.log('Export handler - to be implemented');
};
```

**What Needs to Be Implemented** (from plan Phase 4):

Create `tracewriter/src/utils/exportAnnotations.js`:
```javascript
/**
 * Generate annotated export JSON with _annotation_after fields
 */
export function generateExport(properties, annotations, annotator = 'matan') {
  return properties.map(property => ({
    id: property.id,
    subject: property.subject,
    property: property.property,
    thread_count: property.threadCount,
    email_count: property.emailCount,
    threads: property.threads.map(thread => ({
      id: thread.id,
      subject: thread.subject,
      email_count: thread.emailCount,
      emails: thread.emails.map((email, index) => {
        const annotationKey = `${thread.id}:${index}`;
        const annotation = annotations[annotationKey]?.trim() || null;
        const isLastEmail = index === thread.emails.length - 1;

        return {
          id: email.id,
          from: email.from,
          to: email.to,
          date: email.date,
          dateDisplay: email.dateDisplay,
          body: email.body,
          _annotation_after: isLastEmail ? null : annotation,
        };
      }),
    })),
    _metadata: {
      annotated_at: new Date().toISOString(),
      annotator,
    },
  }));
}

/**
 * Download JSON as file
 */
export function downloadJson(data, filename = 'annotated-threads.json') {
  const json = JSON.stringify(data, null, 2);
  const blob = new Blob([json], { type: 'application/json' });
  const url = URL.createObjectURL(blob);

  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}
```

Update `App.jsx` handleExport:
```javascript
import { generateExport, downloadJson } from './utils/exportAnnotations';

const handleExport = () => {
  const exportData = generateExport(properties, annotations);
  const filename = `tracewriter-export-${new Date().toISOString().split('T')[0]}.json`;
  downloadJson(exportData, filename);
};
```

### 2. Export Prompt Before Import

Add confirmation in the file input handler (`App.jsx:269-282`):
```javascript
onChange={(e) => {
  const file = e.target.files[0];
  if (file) {
    // Check if there are existing annotations to save
    const hasAnnotations = Object.keys(annotations).some(k => annotations[k]?.trim());
    if (hasAnnotations) {
      const shouldContinue = window.confirm(
        'You have unsaved annotations. Would you like to export them before importing new data?'
      );
      if (!shouldContinue) {
        e.target.value = '';
        return;
      }
    }
    // ... rest of import logic
  }
}}
```

### 3. State Persistence Options for Vercel

#### Option A: Vercel KV (Recommended for Simplicity)
- Redis-compatible key-value store
- Simple API: `kv.set()`, `kv.get()`
- Store state as single JSON blob
- ~$0.20/100K requests

**Implementation**:
```javascript
// api/state.js (Vercel serverless function)
import { kv } from '@vercel/kv';

export async function GET() {
  const state = await kv.get('tracewriter-state');
  return Response.json(state || { properties: [], annotations: {} });
}

export async function POST(request) {
  const body = await request.json();
  await kv.set('tracewriter-state', body);
  return Response.json({ success: true });
}
```

**Frontend integration**:
```javascript
// Load on mount
useEffect(() => {
  fetch('/api/state').then(r => r.json()).then(state => {
    if (state.properties?.length > 0) {
      setProperties(state.properties);
      setAnnotations(state.annotations || {});
    }
  });
}, []);

// Auto-save annotations (debounced)
useEffect(() => {
  const save = debounce(() => {
    fetch('/api/state', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ properties, annotations })
    });
  }, 2000);
  save();
}, [annotations, properties]);
```

#### Option B: Supabase (Best for Real-Time Sync)
- PostgreSQL database with real-time subscriptions
- Multiple users see changes instantly
- Built-in auth if needed
- Free tier available

#### Option C: localStorage + Manual Export/Import
- Simplest to implement (use existing plan Phase 3 code)
- Each user has their own local state
- Not truly collaborative - users must export/share JSON files
- Good for "one person annotates at a time" workflow

### 4. Files to Create

| File | Purpose | Priority |
|------|---------|----------|
| `src/utils/exportAnnotations.js` | Export JSON with annotations | High |
| `src/utils/storage.js` | Persistence utilities | High |
| `src/hooks/useLocalStorage.js` | Auto-save hook (if localStorage) | Medium |
| `api/state.js` | Vercel KV endpoint (if backend) | High for collab |

### 5. Existing Plan Reference

The implementation plan at `thoughts/shared/plans/2024-12-16-tracewriter-email-annotation-tool.md` contains:
- **Phase 3** (lines 645-887): localStorage persistence code
- **Phase 4** (lines 905-1001): Export functionality code

These phases were designed for single-user localStorage but the code structure is reusable. The storage.js and exportAnnotations.js can be adapted for any backend.

## Code References

- `tracewriter/src/App.jsx:212-214` - Placeholder export handler
- `tracewriter/src/App.jsx:43` - Annotations state (in-memory only)
- `tracewriter/src/App.jsx:269-282` - Import handler (no export prompt)
- `tracewriter/src/utils/emailParser.js` - Full parser (works correctly)
- `thoughts/shared/plans/2024-12-16-tracewriter-email-annotation-tool.md:645-1001` - Phase 3 & 4 implementation specs

## Architecture Documentation

**Current Data Flow**:
```
MBOX → Python script → JSON file → Import button → parsePreprocessedJson() → properties state
                                                                            ↓
                                                 annotations state ← user edits gaps
                                                       ↓
                                              (LOST ON REFRESH - no persistence)
```

**Desired Data Flow (with Vercel KV)**:
```
MBOX → Python script → JSON file → Import → parse → properties + annotations
                                      ↓                    ↓
                                 Vercel KV ←←←←←←← auto-save (debounced)
                                      ↓
                        Load on mount ← other users/sessions
```

## Implementation Priority

1. **Export functionality** (blocking - users can't save work)
2. **Export prompt before import** (safety feature)
3. **Backend persistence** (required for team collaboration)
4. **Progress tracking UI** (nice-to-have, Phase 5 of plan)

## Open Questions

1. **Single JSON or user-specific?** - Should all team members edit the same annotation set, or have individual copies?
2. **Conflict resolution** - If two people annotate the same gap, which wins?
3. **Auth required?** - Private link implies no auth, but might want basic protection

## Related Research

- `thoughts/shared/plans/2024-12-16-tracewriter-email-annotation-tool.md` - Full implementation spec
- `thoughts/shared/plans/2026-01-03-tracewriter-multi-thread-property-ui.md` - Recent UI hierarchy changes
- `thoughts/shared/handoffs/general/2025-12-18_20-14-07_tracewriter-property-grouping.md` - Data cleanup context
