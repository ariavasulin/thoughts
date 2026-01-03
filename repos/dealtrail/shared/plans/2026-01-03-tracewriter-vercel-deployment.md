# TraceWriter Vercel Deployment with Team Collaboration

## Overview

Deploy TraceWriter to Vercel with shared state persistence using Vercel KV, enabling team collaboration (up to 3 users) with auto-save, export functionality, and refresh-based sync.

## Current State Analysis

**What EXISTS:**
- TraceWriter React 19 + Vite 7 app with 3-tier hierarchy (Properties → Threads → Emails)
- Working JSON import from preprocessed MBOX (`src/utils/emailParser.js`)
- Keyboard-first navigation in `App.jsx`
- In-memory annotation storage (`App.jsx:43`)

**What's MISSING:**
- Export functionality - `handleExport()` at `App.jsx:212` is a stub
- Persistence layer - annotations lost on refresh
- Backend API - no serverless functions
- Vercel deployment configuration

### Key Discoveries:
- `tracewriter/src/App.jsx:212-214` - Placeholder export handler
- `tracewriter/src/App.jsx:43` - Annotations state (in-memory only)
- `tracewriter/src/utils/emailParser.js` - Full parser supporting nested property schema
- No `storage.js` or `exportAnnotations.js` exists

## Desired End State

A deployed Vercel application where:
1. Team members can access TraceWriter via a shared URL
2. Annotations auto-save to Vercel KV (debounced 2s)
3. State loads from KV on page load
4. Export button downloads annotated JSON
5. Multiple users can work (not simultaneously on same gap) with refresh to sync

### Verification:
- App deploys successfully to Vercel
- Import JSON, add annotation, refresh → annotation persists
- Open in different browser → see same annotations
- Export produces valid JSON with `_annotation_after` fields
- Re-import exported JSON restores annotations

## What We're NOT Doing

- Real-time sync (no WebSockets/Supabase)
- User authentication
- Conflict resolution UI (last-write-wins is acceptable)
- Per-user annotation tracking
- Optimistic UI updates with rollback

## Implementation Approach

Use Vercel's serverless functions with Vercel KV for a simple key-value persistence layer. Store the entire app state (properties + annotations) as a single JSON blob. Frontend auto-saves on annotation changes with debouncing.

---

## Phase 1: Export Functionality

### Overview
Implement the missing export handler to download annotated JSON files. This is a prerequisite for users to save their work locally.

### Changes Required:

#### 1.1 Create Export Utility

**File**: `tracewriter/src/utils/exportAnnotations.js`

```javascript
/**
 * Generate annotated export JSON with _annotation_after fields
 */
export function generateExport(properties, annotations, annotator = 'team') {
  return properties.map(property => ({
    id: property.id,
    subject: property.subject,
    property: property.property,
    thread_count: property.threadCount,
    email_count: property.emailCount,
    threads: property.threads.map(thread => ({
      id: thread.id,
      subject: thread.subject,
      normalized_subject: thread.normalizedSubject,
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
          subject: email.subject,
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

#### 1.2 Update App.jsx Export Handler

**File**: `tracewriter/src/App.jsx`

Add import at top:
```javascript
import { generateExport, downloadJson } from './utils/exportAnnotations';
```

Replace the placeholder `handleExport` function (line 212-214):
```javascript
const handleExport = () => {
  const exportData = generateExport(properties, annotations);
  const filename = `tracewriter-export-${new Date().toISOString().split('T')[0]}.json`;
  downloadJson(exportData, filename);
};
```

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `cd tracewriter && npm run build`
- [x] No TypeScript/ESLint errors: `npm run lint`

#### Manual Verification:
- [x] Export button downloads a JSON file
- [x] Downloaded JSON has correct structure with `_annotation_after` fields
- [x] Last email in each thread has `_annotation_after: null`
- [x] `_metadata` block present with timestamp
- [x] Re-importing exported JSON restores annotations

**Implementation Note**: Test export, inspect the JSON, then re-import to verify round-trip works before proceeding.

---

## Phase 2: Vercel Project Setup

### Overview
Initialize Vercel project configuration and set up Vercel KV database.

### Changes Required:

#### 2.1 Create Vercel Configuration

**File**: `tracewriter/vercel.json`

```json
{
  "framework": "vite",
  "buildCommand": "npm run build",
  "outputDirectory": "dist",
  "rewrites": [
    { "source": "/api/(.*)", "destination": "/api/$1" }
  ]
}
```

#### 2.2 Install Vercel KV Package

```bash
cd tracewriter
npm install @vercel/kv
```

#### 2.3 Create Environment Variables Template

**File**: `tracewriter/.env.example`

```bash
# Vercel KV credentials (auto-populated when you link KV in Vercel dashboard)
KV_REST_API_URL=
KV_REST_API_TOKEN=
KV_REST_API_READ_ONLY_TOKEN=
```

#### 2.4 Update .gitignore

Add to `tracewriter/.gitignore`:
```
.env
.env.local
.vercel
```

### Vercel Dashboard Setup (Manual Steps)

1. Go to [vercel.com](https://vercel.com) and create new project
2. Import the `Dealtrail` repository
3. Set root directory to `tracewriter`
4. Go to Storage tab → Create Database → KV
5. Name it `tracewriter-state`
6. Connect it to your project (this auto-populates env vars)

### Success Criteria:

#### Automated Verification:
- [x] `npm install` succeeds with @vercel/kv added
- [x] Build succeeds: `npm run build`

#### Manual Verification:
- [x] Vercel project created and linked to repo
- [x] KV database created in Vercel dashboard
- [x] Environment variables visible in Vercel project settings

**Implementation Note**: Complete Vercel dashboard setup before proceeding to Phase 3.

---

## Phase 3: API Routes for State Persistence

### Overview
Create Vercel serverless functions to get/set application state in KV.

### Changes Required:

#### 3.1 Create API Directory Structure

```
tracewriter/
├── api/
│   └── state.js      # Serverless function for state get/set
```

#### 3.2 State API Endpoint

**File**: `tracewriter/api/state.js`

```javascript
import { kv } from '@vercel/kv';

const STATE_KEY = 'tracewriter:state';

export default async function handler(req, res) {
  // Set CORS headers for local development
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') {
    return res.status(200).end();
  }

  try {
    if (req.method === 'GET') {
      const state = await kv.get(STATE_KEY);
      return res.status(200).json(state || { properties: [], annotations: {} });
    }

    if (req.method === 'POST') {
      const { properties, annotations } = req.body;

      if (!properties || !annotations) {
        return res.status(400).json({ error: 'Missing properties or annotations' });
      }

      await kv.set(STATE_KEY, { properties, annotations });
      return res.status(200).json({ success: true, savedAt: new Date().toISOString() });
    }

    return res.status(405).json({ error: 'Method not allowed' });
  } catch (error) {
    console.error('State API error:', error);
    return res.status(500).json({ error: 'Internal server error' });
  }
}
```

#### 3.3 Update Vite Config for API Proxy (Development)

**File**: `tracewriter/vite.config.js`

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vite.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
})
```

**Note**: For local development with KV, you'll need to use `vercel dev` instead of `npm run dev` to have access to the serverless functions and KV.

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `npm run build`
- [x] API file has no syntax errors

#### Manual Verification:
- [x] Deploy to Vercel preview
- [x] `GET /api/state` returns `{ properties: [], annotations: {} }`
- [x] `POST /api/state` with body saves and returns success
- [x] `GET /api/state` after POST returns saved data

**Implementation Note**: Test the API endpoints in Vercel preview deployment before proceeding to frontend integration.

---

## Phase 4: Frontend Persistence Integration

### Overview
Connect the React app to the API for auto-save and load-on-mount functionality.

### Changes Required:

#### 4.1 Create Storage Hook

**File**: `tracewriter/src/hooks/useCloudStorage.js`

```javascript
import { useEffect, useCallback, useRef } from 'react';

/**
 * Debounce helper
 */
function debounce(fn, delay) {
  let timeoutId;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

/**
 * Hook to manage cloud persistence for TraceWriter
 */
export function useCloudStorage({
  properties,
  setProperties,
  annotations,
  setAnnotations,
  setCurrentPropertyIndex,
  setCurrentThreadIndex,
  setCurrentEmailIndex,
  setExpandedProperties,
}) {
  const isInitialized = useRef(false);
  const isSaving = useRef(false);

  // Load state from API on mount
  useEffect(() => {
    if (isInitialized.current) return;
    isInitialized.current = true;

    async function loadState() {
      try {
        const response = await fetch('/api/state');
        if (!response.ok) throw new Error('Failed to load state');

        const state = await response.json();

        if (state.properties?.length > 0) {
          setProperties(state.properties);
          setAnnotations(state.annotations || {});
          setCurrentPropertyIndex(0);
          setCurrentThreadIndex(0);
          setCurrentEmailIndex(0);
          // Expand first property
          setExpandedProperties(new Set([state.properties[0]?.id]));
          console.log('Loaded state from cloud:', {
            properties: state.properties.length,
            annotations: Object.keys(state.annotations || {}).length,
          });
        }
      } catch (error) {
        console.warn('Failed to load state from cloud:', error);
      }
    }

    loadState();
  }, [setProperties, setAnnotations, setCurrentPropertyIndex, setCurrentThreadIndex, setCurrentEmailIndex, setExpandedProperties]);

  // Save state to API
  const saveState = useCallback(async (props, anns) => {
    if (isSaving.current) return;
    if (!props?.length) return;

    isSaving.current = true;
    try {
      const response = await fetch('/api/state', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ properties: props, annotations: anns }),
      });

      if (!response.ok) throw new Error('Failed to save state');

      const result = await response.json();
      console.log('Saved state to cloud:', result.savedAt);
    } catch (error) {
      console.error('Failed to save state to cloud:', error);
    } finally {
      isSaving.current = false;
    }
  }, []);

  // Debounced save (2 seconds)
  const debouncedSave = useCallback(
    debounce((props, anns) => saveState(props, anns), 2000),
    [saveState]
  );

  // Auto-save when annotations change
  useEffect(() => {
    if (properties.length > 0) {
      debouncedSave(properties, annotations);
    }
  }, [annotations, properties, debouncedSave]);

  // Return manual save function for import
  return { saveState };
}
```

#### 4.2 Update App.jsx to Use Cloud Storage

**File**: `tracewriter/src/App.jsx`

Add import at top:
```javascript
import { useCloudStorage } from './hooks/useCloudStorage';
```

Add the hook after state declarations (around line 45):
```javascript
const { saveState } = useCloudStorage({
  properties,
  setProperties,
  annotations,
  setAnnotations,
  setCurrentPropertyIndex,
  setCurrentThreadIndex,
  setCurrentEmailIndex,
  setExpandedProperties,
});
```

Update `handleImport` to save immediately after import (around line 191):
```javascript
const handleImport = (json) => {
  let parsedProperties;
  if (hasExistingAnnotations(json)) {
    const { properties: parsed, annotations: existingAnnotations } = parseAnnotatedExport(json);
    parsedProperties = parsed;
    setProperties(parsed);
    setAnnotations(prev => ({ ...prev, ...existingAnnotations }));
    // Save immediately after import
    saveState(parsed, { ...annotations, ...existingAnnotations });
  } else {
    parsedProperties = parsePreprocessedJson(json);
    setProperties(parsedProperties);
    // Save immediately after import
    saveState(parsedProperties, annotations);
  }
  setCurrentPropertyIndex(0);
  setCurrentThreadIndex(0);
  setCurrentEmailIndex(0);
  setFocusedAnnotation(null);
  if (parsedProperties.length > 0) {
    setExpandedProperties(new Set([parsedProperties[0]?.id]));
  }
};
```

#### 4.3 Add Save Status Indicator (Optional Enhancement)

Add a small indicator in the footer to show save status. Update the footer section in App.jsx:

```javascript
// Add state for save status
const [saveStatus, setSaveStatus] = useState('');

// In the footer, add after TRACEWRITER:
<span style={{ marginLeft: '12px', color: '#5a5755' }}>
  {saveStatus}
</span>
```

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `npm run build`
- [ ] No console errors on load

#### Manual Verification:
- [ ] Open app → loads any existing state from KV
- [ ] Import JSON → state saved to KV (check console logs)
- [ ] Add annotation → auto-saves after 2s (check console logs)
- [ ] Refresh page → annotations persist
- [ ] Open in different browser/incognito → see same annotations
- [ ] Import prompt warns about overwriting (if annotations exist)

**Implementation Note**: Test the full flow: import → annotate → refresh → verify. Then test in a different browser to confirm shared state works.

---

## Phase 5: Deploy & Final Testing

### Overview
Deploy to Vercel production and verify team collaboration works.

### Deployment Steps:

1. **Push to GitHub**:
   ```bash
   git add .
   git commit -m "Add Vercel deployment with KV persistence"
   git push
   ```

2. **Vercel Auto-deploys**: Vercel will automatically deploy on push

3. **Verify Production URL**: Check the deployment at your Vercel URL

### Success Criteria:

#### Automated Verification:
- [ ] Vercel build succeeds
- [ ] No build errors in Vercel dashboard

#### Manual Verification:
- [ ] Production URL loads the app
- [ ] Import JSON works
- [ ] Annotations save (wait 2s, refresh)
- [ ] Export downloads correct JSON
- [ ] Different team member can access and see annotations
- [ ] Different team member's annotations appear after refresh

### Team Collaboration Testing:

1. **User A**: Import JSON, add annotation to first gap
2. **User B**: Open same URL, refresh, verify annotation visible
3. **User B**: Add annotation to second gap
4. **User A**: Refresh, verify both annotations visible
5. **Both**: Export, verify JSON contains all annotations

---

## Testing Strategy

### Unit Tests (Skip for v1):
- Export generates correct structure
- Debounce works correctly

### Manual Testing Checklist:

1. **Fresh Start Flow**:
   - [ ] Open app with empty KV state
   - [ ] Import preprocessed JSON
   - [ ] Add annotations
   - [ ] Refresh → annotations persist
   - [ ] Export → correct JSON structure

2. **Resume Flow**:
   - [ ] Open app with existing KV state
   - [ ] Annotations load automatically
   - [ ] Continue annotating
   - [ ] Export includes all annotations

3. **Team Collaboration Flow**:
   - [ ] User A imports and annotates
   - [ ] User B opens URL, refreshes, sees annotations
   - [ ] User B adds annotations
   - [ ] User A refreshes, sees all annotations

4. **Edge Cases**:
   - [ ] Rapid typing (debounce prevents excessive saves)
   - [ ] Network error during save (console warning, no crash)
   - [ ] Import over existing data (replaces state)

---

## File Structure Summary

```
tracewriter/
├── api/
│   └── state.js                    # NEW: Vercel serverless function
├── src/
│   ├── App.jsx                     # MODIFIED: Add cloud storage hook
│   ├── hooks/
│   │   └── useCloudStorage.js      # NEW: Cloud persistence hook
│   └── utils/
│       ├── emailParser.js          # EXISTING: No changes
│       └── exportAnnotations.js    # NEW: Export utilities
├── vercel.json                     # NEW: Vercel configuration
├── .env.example                    # NEW: Environment template
└── package.json                    # MODIFIED: Add @vercel/kv
```

## Dependencies

Add to `package.json`:
```json
{
  "dependencies": {
    "@vercel/kv": "^2.0.0"
  }
}
```

## References

- Research: `thoughts/shared/research/2026-01-03-tracewriter-export-persistence.md`
- Original plan: `thoughts/shared/plans/2024-12-16-tracewriter-email-annotation-tool.md`
- Vercel KV docs: https://vercel.com/docs/storage/vercel-kv
- Vercel KV pricing: https://vercel.com/docs/storage/vercel-kv/usage-and-pricing
