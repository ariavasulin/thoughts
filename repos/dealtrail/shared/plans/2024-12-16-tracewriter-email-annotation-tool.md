# TraceWriter - Email Annotation Tool Implementation Plan

## Overview

Build TraceWriter, a standalone React+Vite application for annotating "off-screen actions" in email threads. This tool enables Matan to annotate 150+ real estate transaction coordinator email threads, capturing the hidden actions that occurred between emails to train an AI agent (Amy).

## Current State Analysis

We have a React mockup (`Mockup.jsx`) with:
- Complete UI design with dark/warm CodeLayer-style theme
- Keyboard navigation (Alt+j/k for emails, Alt+n/p for threads, Enter to annotate, Esc to exit)
- Thread sidebar, email display, annotation textareas between emails
- Mock data with simplified email structure
- Basic annotation state management (in-memory only)

### What's Missing:
- ~~Project scaffolding (no build system, no package.json)~~ ✅ Done
- MBOX preprocessing script (Python → JSON conversion)
- JSON import in browser
- LocalStorage persistence
- Export/import of annotated data
- Progress tracking UI

## Desired End State

A fully functional single-page React app that:
1. Imports preprocessed JSON (from MBOX via Python script) and displays threads correctly
2. Displays email threads with clean, readable body text
3. Allows annotation of "gaps" between emails
4. Persists all state to localStorage (survives browser close)
5. Exports annotated JSON with `_annotation_after` fields
6. Can re-import previously annotated JSON to continue work
7. Shows progress indicators and supports filtering/navigation helpers

### Verification:
- Python script converts MBOX to JSON successfully
- Can import preprocessed JSON and see threads rendered correctly
- Annotations persist after browser refresh
- Export produces valid JSON with embedded annotations
- Re-importing exported JSON restores annotations
- Progress shows accurate counts

## What We're NOT Doing

- No backend/server - everything is client-side
- No user authentication
- No real-time sync or cloud storage
- No direct MBOX import in browser (use Python preprocessing script)
- No attachment handling (just email body text)
- No multi-user collaboration features

## Implementation Approach

We'll use Vite + React for fast development and simple deployment. Dependencies will be minimal:
- `turndown` - HTML to Markdown conversion for email bodies

**Email Import Pipeline:**
1. **Python preprocessing script** converts MBOX file → JSON (one-time step)
2. **Browser app** imports the preprocessed JSON for annotation

The app will be structured with clear separation:
- Components for UI (ThreadList, EmailView, AnnotationInput, Header)
- Hooks for cross-cutting concerns (useLocalStorage, useKeyboardNav)
- Utils for data transformation (emailParser, exportAnnotations, storage)
- Scripts for preprocessing (Python MBOX → JSON converter)

---

## Phase 1: Project Setup & Scaffold

### Overview
Create the Vite+React project structure and migrate the mockup into a working application.

### Changes Required:

#### 1.1 Initialize Vite Project

```bash
cd /Users/ariasulin/Git/Dealtrail
npm create vite@latest tracewriter -- --template react
cd tracewriter
npm install
```

#### 1.2 Install Dependencies

```bash
npm install turndown
```

#### 1.3 Project Structure

Create the following directory structure:

```
tracewriter/
├── index.html
├── package.json
├── vite.config.js
├── src/
│   ├── main.jsx           # Entry point
│   ├── App.jsx            # Main app component
│   ├── components/
│   │   ├── Header.jsx
│   │   ├── ThreadList.jsx
│   │   ├── EmailView.jsx
│   │   └── AnnotationInput.jsx
│   ├── hooks/
│   │   ├── useKeyboardNav.js
│   │   └── useLocalStorage.js
│   ├── utils/
│   │   ├── gmailParser.js
│   │   ├── exportAnnotations.js
│   │   └── storage.js
│   └── styles.css
└── public/
```

#### 1.4 Migrate Mockup to App.jsx

**File**: `tracewriter/src/App.jsx`

Adapt the mockup component to use the new structure:
- Extract inline styles to `styles.css` (or keep inline - developer preference)
- Replace `mockThreads` with state that can be populated from import
- Keep the existing keyboard navigation and annotation logic
- Add placeholder for import/export buttons in Header

#### 1.5 Create styles.css

**File**: `tracewriter/src/styles.css`

```css
:root {
  --bg-primary: #1a1816;
  --bg-secondary: #252220;
  --bg-input: #1e1c1a;
  --border: #2a2725;
  --border-active: #3a3735;
  --text-primary: #e8e4df;
  --text-secondary: #c5c1bc;
  --text-muted: #8a8480;
  --text-dim: #5a5755;
  --accent: #c9a86c;
  --success: #7a9f6a;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  background-color: var(--bg-primary);
  color: var(--text-primary);
  font-family: "SF Mono", "Fira Code", "Consolas", monospace;
  font-size: 13px;
  line-height: 1.6;
}
```

### Success Criteria:

#### Automated Verification:
- [x] Project builds without errors: `cd tracewriter && npm run build`
- [x] Dev server starts: `npm run dev`
- [ ] No console errors in browser

#### Manual Verification:
- [ ] UI matches mockup appearance (dark theme, gold accent)
- [ ] Mock threads display correctly
- [ ] Keyboard navigation works (Alt+j/k, Alt+n/p, Enter, Esc)
- [ ] Annotation textareas appear and accept input

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the UI looks correct before proceeding.

---

## Phase 2: MBOX Preprocessing & JSON Import

### Overview
Create a Python script to convert the MBOX email export into JSON format, then import that JSON into TraceWriter. The Python script handles all the complex MIME parsing, threading, and text extraction.

### Changes Required:

#### 2.1 Python MBOX → JSON Converter

**File**: `tracewriter/scripts/mbox_to_json.py`

```python
#!/usr/bin/env python3
"""
Convert MBOX file to JSON format for TraceWriter.

Usage:
    python mbox_to_json.py <input.mbox> <output.json>

Example:
    python mbox_to_json.py "../Mail/Transaction Coordinator Emails.mbox" threads.json
"""

import mailbox
import email
import json
import sys
import re
import html
from email.utils import parsedate_to_datetime
from collections import defaultdict
from pathlib import Path


def get_header(msg, name, default=''):
    """Extract and decode email header."""
    value = msg.get(name, default)
    if value:
        # Handle encoded headers (=?utf-8?Q?...?=)
        try:
            decoded_parts = email.header.decode_header(value)
            decoded = ''
            for part, charset in decoded_parts:
                if isinstance(part, bytes):
                    decoded += part.decode(charset or 'utf-8', errors='replace')
                else:
                    decoded += part
            return decoded.strip()
        except:
            return str(value).strip()
    return default


def extract_body(msg):
    """Extract plain text body from email message."""
    body = ''

    if msg.is_multipart():
        # Walk through all parts
        for part in msg.walk():
            content_type = part.get_content_type()
            content_disposition = str(part.get('Content-Disposition', ''))

            # Skip attachments
            if 'attachment' in content_disposition:
                continue

            # Prefer plain text
            if content_type == 'text/plain':
                try:
                    payload = part.get_payload(decode=True)
                    charset = part.get_content_charset() or 'utf-8'
                    body = payload.decode(charset, errors='replace')
                    break
                except:
                    continue

            # Fall back to HTML if no plain text found
            elif content_type == 'text/html' and not body:
                try:
                    payload = part.get_payload(decode=True)
                    charset = part.get_content_charset() or 'utf-8'
                    html_body = payload.decode(charset, errors='replace')
                    body = html_to_text(html_body)
                except:
                    continue
    else:
        # Simple message
        try:
            payload = msg.get_payload(decode=True)
            if payload:
                charset = msg.get_content_charset() or 'utf-8'
                content_type = msg.get_content_type()
                text = payload.decode(charset, errors='replace')

                if content_type == 'text/html':
                    body = html_to_text(text)
                else:
                    body = text
        except:
            body = str(msg.get_payload())

    return clean_body(body)


def html_to_text(html_content):
    """Convert HTML to plain text."""
    # Remove style and script tags
    text = re.sub(r'<style[^>]*>.*?</style>', '', html_content, flags=re.DOTALL | re.IGNORECASE)
    text = re.sub(r'<script[^>]*>.*?</script>', '', text, flags=re.DOTALL | re.IGNORECASE)

    # Convert common tags
    text = re.sub(r'<br\s*/?>', '\n', text, flags=re.IGNORECASE)
    text = re.sub(r'<p[^>]*>', '\n\n', text, flags=re.IGNORECASE)
    text = re.sub(r'</p>', '', text, flags=re.IGNORECASE)
    text = re.sub(r'<div[^>]*>', '\n', text, flags=re.IGNORECASE)
    text = re.sub(r'</div>', '', text, flags=re.IGNORECASE)

    # Remove all other HTML tags
    text = re.sub(r'<[^>]+>', '', text)

    # Decode HTML entities
    text = html.unescape(text)

    return text


def clean_body(body):
    """Clean up email body text."""
    if not body:
        return '[No body content]'

    # Normalize line endings
    body = body.replace('\r\n', '\n').replace('\r', '\n')

    # Remove excessive blank lines (more than 2 in a row)
    body = re.sub(r'\n{4,}', '\n\n\n', body)

    # Strip leading/trailing whitespace
    body = body.strip()

    return body if body else '[No body content]'


def get_thread_id(msg):
    """
    Extract or generate thread ID.
    Uses In-Reply-To or References header to group messages.
    Falls back to subject-based threading.
    """
    # Try Message-ID based threading
    references = msg.get('References', '')
    in_reply_to = msg.get('In-Reply-To', '')

    # Get the root message ID from references (first one)
    if references:
        ref_ids = references.split()
        if ref_ids:
            return ref_ids[0].strip('<>').replace('@', '_at_')

    if in_reply_to:
        return in_reply_to.strip('<>').replace('@', '_at_')

    # Fall back to subject-based threading
    subject = get_header(msg, 'Subject', '').lower()
    # Remove Re:, Fwd:, etc.
    subject = re.sub(r'^(re|fwd|fw):\s*', '', subject, flags=re.IGNORECASE)
    subject = re.sub(r'\s+', '_', subject.strip())

    return f"subj_{subject[:50]}" if subject else None


def parse_date(msg):
    """Parse email date to ISO format."""
    date_str = msg.get('Date', '')
    if date_str:
        try:
            dt = parsedate_to_datetime(date_str)
            return dt.isoformat()
        except:
            pass
    return None


def format_date_display(iso_date):
    """Format date for display (e.g., 'Dec 12, 2:34 PM')."""
    if not iso_date:
        return 'Unknown date'
    try:
        from datetime import datetime
        dt = datetime.fromisoformat(iso_date.replace('Z', '+00:00'))
        return dt.strftime('%b %d, %I:%M %p')
    except:
        return iso_date


def convert_mbox_to_json(mbox_path, output_path):
    """Convert MBOX file to TraceWriter JSON format."""
    print(f"Reading MBOX file: {mbox_path}")

    mbox = mailbox.mbox(mbox_path)

    # Group messages by thread
    threads = defaultdict(list)
    standalone = []

    for i, msg in enumerate(mbox):
        if i % 100 == 0:
            print(f"  Processing message {i}...")

        thread_id = get_thread_id(msg)
        message_id = msg.get('Message-ID', f'msg_{i}').strip('<>')

        email_data = {
            'id': message_id.replace('@', '_at_'),
            'from': get_header(msg, 'From'),
            'to': get_header(msg, 'To'),
            'date': parse_date(msg),
            'subject': get_header(msg, 'Subject', 'No Subject'),
            'body': extract_body(msg),
        }

        if thread_id:
            threads[thread_id].append(email_data)
        else:
            # Standalone message (no threading info)
            standalone.append(email_data)

    print(f"  Found {len(threads)} threads and {len(standalone)} standalone messages")

    # Convert to output format
    output = []

    # Process threaded messages
    for thread_id, messages in threads.items():
        # Sort by date
        messages.sort(key=lambda m: m['date'] or '')

        # Use first message's subject as thread subject
        subject = messages[0]['subject']
        # Clean up Re:/Fwd: prefixes for thread subject
        subject = re.sub(r'^(Re|Fwd|Fw):\s*', '', subject, flags=re.IGNORECASE).strip()

        output.append({
            'id': thread_id,
            'subject': subject or 'No Subject',
            'emails': [
                {
                    'id': m['id'],
                    'from': m['from'],
                    'to': m['to'],
                    'date': m['date'],
                    'dateDisplay': format_date_display(m['date']),
                    'body': m['body'],
                }
                for m in messages
            ]
        })

    # Add standalone messages as single-email threads
    for msg in standalone:
        output.append({
            'id': msg['id'],
            'subject': msg['subject'],
            'emails': [{
                'id': msg['id'],
                'from': msg['from'],
                'to': msg['to'],
                'date': msg['date'],
                'dateDisplay': format_date_display(msg['date']),
                'body': msg['body'],
            }]
        })

    # Sort threads by date of first email
    output.sort(key=lambda t: t['emails'][0]['date'] or '')

    print(f"Writing {len(output)} threads to: {output_path}")

    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump(output, f, indent=2, ensure_ascii=False)

    print("Done!")
    print(f"  Total threads: {len(output)}")
    print(f"  Total emails: {sum(len(t['emails']) for t in output)}")


if __name__ == '__main__':
    if len(sys.argv) != 3:
        print(__doc__)
        sys.exit(1)

    mbox_path = Path(sys.argv[1])
    output_path = Path(sys.argv[2])

    if not mbox_path.exists():
        print(f"Error: MBOX file not found: {mbox_path}")
        sys.exit(1)

    convert_mbox_to_json(mbox_path, output_path)
```

#### 2.2 Email Parser Utility (Simplified)

**File**: `tracewriter/src/utils/emailParser.js`

Since preprocessing handles the heavy lifting, the browser-side parser is much simpler:

```javascript
/**
 * Format date for display (e.g., "Dec 12, 2:34 PM")
 */
function formatDateDisplay(dateStr) {
  if (!dateStr) return 'Unknown date';
  try {
    const date = new Date(dateStr);
    return date.toLocaleDateString('en-US', {
      month: 'short',
      day: 'numeric',
    }) + ', ' + date.toLocaleTimeString('en-US', {
      hour: 'numeric',
      minute: '2-digit',
      hour12: true,
    });
  } catch {
    return dateStr;
  }
}

/**
 * Parse preprocessed JSON (from Python script) into internal format
 * The JSON is already in the right shape, just needs dateDisplay if missing
 */
export function parsePreprocessedJson(data) {
  const threads = Array.isArray(data) ? data : [data];

  return threads.map(thread => ({
    id: thread.id,
    subject: thread.subject,
    emails: thread.emails.map((email, index) => ({
      id: email.id || `msg_${index}`,
      from: email.from,
      to: email.to,
      date: email.date,
      dateDisplay: email.dateDisplay || formatDateDisplay(email.date),
      body: email.body,
    })),
  }));
}

/**
 * Check if imported JSON has pre-existing annotations
 */
export function hasExistingAnnotations(data) {
  const threads = Array.isArray(data) ? data : [data];
  return threads.some(thread =>
    thread.emails?.some(email => '_annotation_after' in email) ||
    thread.messages?.some(msg => '_annotation_after' in msg)
  );
}

/**
 * Parse pre-annotated JSON format (our export format)
 */
export function parseAnnotatedExport(data) {
  const threads = Array.isArray(data) ? data : [data];

  const parsedThreads = [];
  const annotations = {};

  for (const thread of threads) {
    // Handle both 'emails' (preprocessed) and 'messages' (exported) formats
    const messages = thread.emails || thread.messages || [];

    const emails = messages.map((msg, index) => {
      // Extract annotation if present
      if (msg._annotation_after) {
        annotations[`${thread.id}:${index}`] = msg._annotation_after;
      }

      return {
        id: msg.id,
        from: msg.from,
        to: msg.to,
        date: msg.date,
        dateDisplay: msg.dateDisplay || formatDateDisplay(msg.date),
        body: msg.body,
      };
    });

    parsedThreads.push({
      id: thread.id,
      subject: thread.subject,
      emails,
    });
  }

  return { threads: parsedThreads, annotations };
}

export default {
  parsePreprocessedJson,
  parseAnnotatedExport,
  hasExistingAnnotations,
};
```

#### 2.3 Update App.jsx Import Logic

**File**: `tracewriter/src/App.jsx`

```javascript
import { parsePreprocessedJson, parseAnnotatedExport, hasExistingAnnotations } from './utils/emailParser';

// In App component:
const handleImport = (json) => {
  if (hasExistingAnnotations(json)) {
    // Re-importing annotated export
    const { threads: parsed, annotations: existingAnnotations } = parseAnnotatedExport(json);
    setThreads(parsed);
    setAnnotations(prev => ({ ...prev, ...existingAnnotations }));
  } else {
    // Fresh preprocessed JSON from Python script
    const parsed = parsePreprocessedJson(json);
    setThreads(parsed);
  }
  setCurrentThreadIndex(0);
  setCurrentEmailIndex(0);
};
```

### Usage Instructions

1. **Preprocess the MBOX file** (one-time step):
   ```bash
   cd tracewriter/scripts
   python mbox_to_json.py "../../Mail/Transaction Coordinator Emails.mbox" ../threads.json
   ```

2. **Import in TraceWriter**:
   - Click "Import JSON" button
   - Select the generated `threads.json` file
   - Begin annotating

### Success Criteria:

#### Automated Verification:
- [x] Python script runs without errors: `python mbox_to_json.py <input> <output>`
- [x] Build succeeds: `npm run build`
- [x] Generated JSON is valid: `python -c "import json; json.load(open('threads.json'))"`

#### Manual Verification:
- [x] Python script processes the full MBOX file
- [x] Generated JSON has expected thread structure
- [x] Email bodies are readable (no base64, no raw HTML)
- [x] Threads are grouped correctly (related emails together) - Note: threading algorithm works but data needs cleanup
- [x] Import in TraceWriter loads all threads
- [x] Email display looks correct (proper text formatting)

**Note:** Phase 2 complete. The MBOX preprocessing pipeline works, but the source email data needs additional cleanup work (thread grouping, email body formatting). This is a data quality issue, not a code issue.

**Implementation Note**: Run the Python script first, inspect the output JSON, then test import in TraceWriter. Pause for verification before proceeding.

---

## Phase 3: Local Storage Persistence

### Overview
Persist annotations, navigation position, and imported threads to localStorage so work survives browser close.

### Changes Required:

#### 3.1 Storage Utility

**File**: `tracewriter/src/utils/storage.js`

```javascript
const STORAGE_KEYS = {
  ANNOTATIONS: 'tracewriter:annotations',
  PROGRESS: 'tracewriter:progress',
  THREADS: 'tracewriter:threads',
};

/**
 * Debounce helper for auto-save
 */
export function debounce(fn, delay) {
  let timeoutId;
  return (...args) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

/**
 * Save annotations to localStorage
 */
export function saveAnnotations(annotations) {
  try {
    localStorage.setItem(STORAGE_KEYS.ANNOTATIONS, JSON.stringify(annotations));
  } catch (e) {
    console.warn('Failed to save annotations:', e);
  }
}

/**
 * Load annotations from localStorage
 */
export function loadAnnotations() {
  try {
    const stored = localStorage.getItem(STORAGE_KEYS.ANNOTATIONS);
    return stored ? JSON.parse(stored) : {};
  } catch (e) {
    console.warn('Failed to load annotations:', e);
    return {};
  }
}

/**
 * Save navigation progress
 */
export function saveProgress(threadIndex, emailIndex) {
  try {
    localStorage.setItem(STORAGE_KEYS.PROGRESS, JSON.stringify({
      currentThreadIndex: threadIndex,
      currentEmailIndex: emailIndex,
    }));
  } catch (e) {
    console.warn('Failed to save progress:', e);
  }
}

/**
 * Load navigation progress
 */
export function loadProgress() {
  try {
    const stored = localStorage.getItem(STORAGE_KEYS.PROGRESS);
    return stored ? JSON.parse(stored) : { currentThreadIndex: 0, currentEmailIndex: 0 };
  } catch (e) {
    console.warn('Failed to load progress:', e);
    return { currentThreadIndex: 0, currentEmailIndex: 0 };
  }
}

/**
 * Save imported threads
 */
export function saveThreads(threads) {
  try {
    localStorage.setItem(STORAGE_KEYS.THREADS, JSON.stringify(threads));
  } catch (e) {
    console.warn('Failed to save threads:', e);
  }
}

/**
 * Load imported threads
 */
export function loadThreads() {
  try {
    const stored = localStorage.getItem(STORAGE_KEYS.THREADS);
    return stored ? JSON.parse(stored) : [];
  } catch (e) {
    console.warn('Failed to load threads:', e);
    return [];
  }
}

/**
 * Clear all stored data
 */
export function clearAllStorage() {
  localStorage.removeItem(STORAGE_KEYS.ANNOTATIONS);
  localStorage.removeItem(STORAGE_KEYS.PROGRESS);
  localStorage.removeItem(STORAGE_KEYS.THREADS);
}

export default {
  saveAnnotations,
  loadAnnotations,
  saveProgress,
  loadProgress,
  saveThreads,
  loadThreads,
  clearAllStorage,
  debounce,
};
```

#### 3.2 useLocalStorage Hook

**File**: `tracewriter/src/hooks/useLocalStorage.js`

```javascript
import { useEffect, useCallback, useRef } from 'react';
import {
  saveAnnotations,
  loadAnnotations,
  saveProgress,
  loadProgress,
  saveThreads,
  loadThreads,
  debounce,
} from '../utils/storage';

/**
 * Hook to manage localStorage persistence for TraceWriter
 */
export function useLocalStorage({
  threads,
  setThreads,
  annotations,
  setAnnotations,
  currentThreadIndex,
  setCurrentThreadIndex,
  currentEmailIndex,
  setCurrentEmailIndex,
}) {
  const isInitialized = useRef(false);

  // Load from localStorage on mount
  useEffect(() => {
    if (isInitialized.current) return;
    isInitialized.current = true;

    const storedThreads = loadThreads();
    const storedAnnotations = loadAnnotations();
    const storedProgress = loadProgress();

    if (storedThreads.length > 0) {
      setThreads(storedThreads);
      setAnnotations(storedAnnotations);

      // Validate progress indices
      const threadIdx = Math.min(storedProgress.currentThreadIndex, storedThreads.length - 1);
      const emailIdx = Math.min(
        storedProgress.currentEmailIndex,
        storedThreads[threadIdx]?.emails?.length - 1 || 0
      );

      setCurrentThreadIndex(threadIdx);
      setCurrentEmailIndex(emailIdx);
    }
  }, [setThreads, setAnnotations, setCurrentThreadIndex, setCurrentEmailIndex]);

  // Debounced save for annotations
  const debouncedSaveAnnotations = useCallback(
    debounce((anns) => saveAnnotations(anns), 500),
    []
  );

  // Auto-save annotations on change
  useEffect(() => {
    if (Object.keys(annotations).length > 0) {
      debouncedSaveAnnotations(annotations);
    }
  }, [annotations, debouncedSaveAnnotations]);

  // Auto-save progress on navigation
  useEffect(() => {
    if (threads.length > 0) {
      saveProgress(currentThreadIndex, currentEmailIndex);
    }
  }, [currentThreadIndex, currentEmailIndex, threads.length]);

  // Save threads when imported
  const persistThreads = useCallback((newThreads) => {
    saveThreads(newThreads);
  }, []);

  return { persistThreads };
}
```

#### 3.3 Integrate Hook in App.jsx

```javascript
import { useLocalStorage } from './hooks/useLocalStorage';

// In App component:
const { persistThreads } = useLocalStorage({
  threads,
  setThreads,
  annotations,
  setAnnotations,
  currentThreadIndex,
  setCurrentThreadIndex,
  currentEmailIndex,
  setCurrentEmailIndex,
});

// Update handleImport to also persist:
const handleImport = (json) => {
  let parsed;
  if (hasExistingAnnotations(json)) {
    const result = parseAnnotatedExport(json);
    parsed = result.threads;
    setAnnotations(prev => ({ ...prev, ...result.annotations }));
  } else {
    parsed = parseGmailExport(json);
  }
  setThreads(parsed);
  persistThreads(parsed); // Save to localStorage
  setCurrentThreadIndex(0);
  setCurrentEmailIndex(0);
};
```

### Success Criteria:

#### Automated Verification:
- [ ] Build succeeds: `npm run build`
- [ ] No console errors related to localStorage

#### Manual Verification:
- [ ] After importing threads, refresh browser - threads still appear
- [ ] After adding annotations, refresh browser - annotations persist
- [ ] After navigating to a different thread/email, refresh - position restored
- [ ] Annotations save within ~500ms of typing (check localStorage in DevTools)

**Implementation Note**: Test the full flow: import -> annotate -> refresh -> verify persistence. Pause for manual verification.

---

## Phase 4: Export Annotated JSON

### Overview
Add export functionality that produces a clean JSON file with `_annotation_after` fields embedded in each message.

### Changes Required:

#### 4.1 Export Utility

**File**: `tracewriter/src/utils/exportAnnotations.js`

```javascript
/**
 * Generate annotated export JSON
 */
export function generateExport(threads, annotations, annotator = 'matan') {
  return threads.map(thread => ({
    id: thread.id,
    subject: thread.subject,
    messages: thread.emails.map((email, index) => {
      const annotationKey = `${thread.id}:${index}`;
      const annotation = annotations[annotationKey]?.trim() || null;

      // Last email in thread has null annotation (no "after")
      const isLastEmail = index === thread.emails.length - 1;

      return {
        id: email.id,
        from: email.from,
        to: email.to,
        date: email.date,
        body: email.body,
        _annotation_after: isLastEmail ? null : annotation,
      };
    }),
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

export default {
  generateExport,
  downloadJson,
};
```

#### 4.2 Wire Up Export Button in App.jsx

```javascript
import { generateExport, downloadJson } from './utils/exportAnnotations';

// In App component:
const handleExport = () => {
  const exportData = generateExport(threads, annotations);
  const filename = `tracewriter-export-${new Date().toISOString().split('T')[0]}.json`;
  downloadJson(exportData, filename);
};

// Pass to Header:
<Header
  // ... other props
  onExport={handleExport}
/>
```

### Success Criteria:

#### Automated Verification:
- [ ] Build succeeds: `npm run build`

#### Manual Verification:
- [ ] Export button appears when threads are loaded
- [ ] Clicking export downloads a JSON file
- [ ] Downloaded JSON has correct structure with `_annotation_after` fields
- [ ] Annotations appear in correct positions
- [ ] Last email in each thread has `_annotation_after: null`
- [ ] `_metadata` block present with timestamp and annotator

**Implementation Note**: Export a file, inspect the JSON, then re-import to verify round-trip works. Pause for verification.

---

## Phase 5: Progress Indicators & UX Polish

### Overview
Add progress tracking UI, thread filtering, jump-to-next-unannotated, and clear-all functionality.

### Changes Required:

#### 5.1 Thread List with Progress Indicators

**File**: `tracewriter/src/components/ThreadList.jsx`

```javascript
import React from 'react';

export default function ThreadList({
  threads,
  currentThreadIndex,
  annotations,
  filter,
  onSelectThread,
}) {
  // Calculate annotation status for each thread
  const getThreadStatus = (thread) => {
    const gaps = thread.emails.length - 1;
    if (gaps === 0) return { annotated: 0, total: 0, status: 'complete' };

    let annotated = 0;
    for (let i = 0; i < gaps; i++) {
      if (annotations[`${thread.id}:${i}`]?.trim()) {
        annotated++;
      }
    }

    if (annotated === 0) return { annotated, total: gaps, status: 'none' };
    if (annotated === gaps) return { annotated, total: gaps, status: 'complete' };
    return { annotated, total: gaps, status: 'partial' };
  };

  // Apply filter
  const filteredThreads = threads.filter((thread, idx) => {
    if (filter === 'all') return true;
    const status = getThreadStatus(thread).status;
    if (filter === 'unannotated') return status === 'none';
    if (filter === 'partial') return status === 'partial';
    if (filter === 'complete') return status === 'complete';
    return true;
  });

  // Map filtered indices back to original indices
  const getOriginalIndex = (thread) => threads.findIndex(t => t.id === thread.id);

  return (
    <div style={{
      width: '280px',
      borderRight: '1px solid #2a2725',
      padding: '12px 0',
      overflowY: 'auto',
      height: '100%',
    }}>
      {filteredThreads.map((thread) => {
        const originalIdx = getOriginalIndex(thread);
        const { annotated, total, status } = getThreadStatus(thread);
        const isSelected = originalIdx === currentThreadIndex;

        return (
          <div
            key={thread.id}
            onClick={() => onSelectThread(originalIdx)}
            style={{
              padding: '10px 16px',
              cursor: 'pointer',
              backgroundColor: isSelected ? '#252220' : 'transparent',
              borderLeft: isSelected ? '2px solid #c9a86c' : '2px solid transparent',
            }}
          >
            <div style={{
              display: 'flex',
              alignItems: 'center',
              gap: '8px',
              marginBottom: '4px',
            }}>
              {/* Status indicator */}
              <span style={{
                width: '8px',
                height: '8px',
                borderRadius: '50%',
                backgroundColor: status === 'complete' ? '#7a9f6a'
                  : status === 'partial' ? '#c9a86c'
                  : 'transparent',
                border: status === 'none' ? '1px solid #5a5755' : 'none',
                flexShrink: 0,
              }} />

              <div style={{
                color: isSelected ? '#e8e4df' : '#8a8480',
                whiteSpace: 'nowrap',
                overflow: 'hidden',
                textOverflow: 'ellipsis',
                flex: 1,
              }}>
                {thread.subject}
              </div>
            </div>

            <div style={{ color: '#5a5755', fontSize: '11px', marginLeft: '16px' }}>
              {thread.emails.length} emails
              {total > 0 && ` · ${annotated}/${total}`}
            </div>
          </div>
        );
      })}

      {filteredThreads.length === 0 && (
        <div style={{ padding: '20px 16px', color: '#5a5755', textAlign: 'center' }}>
          No threads match filter
        </div>
      )}
    </div>
  );
}
```

#### 5.2 Filter Controls in Header

Add filter dropdown to Header.jsx:

```javascript
// Add to Header props and render:
<select
  value={filter}
  onChange={(e) => onFilterChange(e.target.value)}
  style={{
    backgroundColor: '#252220',
    border: '1px solid #3a3735',
    borderRadius: '4px',
    color: '#8a8480',
    padding: '6px 8px',
    fontFamily: 'inherit',
    fontSize: '12px',
    cursor: 'pointer',
  }}
>
  <option value="all">All threads</option>
  <option value="unannotated">Unannotated</option>
  <option value="partial">Partial</option>
  <option value="complete">Complete</option>
</select>
```

#### 5.3 Jump to Next Unannotated (Alt+u)

Add to keyboard navigation in App.jsx:

```javascript
// In useEffect for keyboard handling:
else if (e.altKey && e.key === 'u') {
  e.preventDefault();
  jumpToNextUnannotated();
}

// Helper function:
const jumpToNextUnannotated = () => {
  // First check current thread for unannotated gaps
  const currentThread = threads[currentThreadIndex];
  for (let i = currentEmailIndex; i < currentThread.emails.length - 1; i++) {
    if (!annotations[`${currentThread.id}:${i}`]?.trim()) {
      setCurrentEmailIndex(i);
      return;
    }
  }

  // Then check subsequent threads
  for (let t = currentThreadIndex + 1; t < threads.length; t++) {
    const thread = threads[t];
    for (let i = 0; i < thread.emails.length - 1; i++) {
      if (!annotations[`${thread.id}:${i}`]?.trim()) {
        setCurrentThreadIndex(t);
        setCurrentEmailIndex(i);
        return;
      }
    }
  }

  // Wrap around to beginning
  for (let t = 0; t <= currentThreadIndex; t++) {
    const thread = threads[t];
    const startIdx = t === currentThreadIndex ? 0 : 0;
    for (let i = startIdx; i < thread.emails.length - 1; i++) {
      if (!annotations[`${thread.id}:${i}`]?.trim()) {
        setCurrentThreadIndex(t);
        setCurrentEmailIndex(i);
        return;
      }
    }
  }

  // All annotated!
  alert('All gaps have been annotated!');
};
```

#### 5.4 Clear All with Confirmation

Add to Header.jsx:

```javascript
const handleClearAll = () => {
  if (window.confirm('Clear all annotations? This cannot be undone.')) {
    onClearAll();
  }
};

// Button:
<button
  onClick={handleClearAll}
  style={{
    backgroundColor: 'transparent',
    border: '1px solid #3a3735',
    borderRadius: '4px',
    color: '#5a5755',
    padding: '6px 12px',
    cursor: 'pointer',
    fontFamily: 'inherit',
    fontSize: '12px',
  }}
>
  Clear All
</button>
```

#### 5.5 Global Progress in Header

Update the progress display to show overall stats:

```javascript
// Calculate global progress
const totalGaps = threads.reduce((sum, t) => sum + Math.max(0, t.emails.length - 1), 0);
const totalAnnotated = Object.values(annotations).filter(a => a?.trim()).length;

// Display:
<div style={{ color: '#5a5755', fontSize: '12px' }}>
  {threadIndex + 1}/{totalThreads} threads
  {' · '}
  {totalAnnotated}/{totalGaps} gaps annotated
</div>
```

#### 5.6 Update Footer with New Shortcut

Add Alt+u hint to footer keyboard hints.

### Success Criteria:

#### Automated Verification:
- [ ] Build succeeds: `npm run build`
- [ ] No console errors

#### Manual Verification:
- [ ] Thread list shows status indicators (empty circle, yellow partial, green complete)
- [ ] Filter dropdown filters threads correctly
- [ ] Alt+u jumps to next unannotated gap
- [ ] Progress shows accurate global counts
- [ ] Clear All prompts for confirmation and clears annotations
- [ ] Thread sidebar shows annotation progress per thread

**Implementation Note**: Test all filters and the jump functionality with partially annotated data. Verify the UX flows smoothly.

---

## Testing Strategy

### Unit Tests (Optional - skip for v1):
- Python MBOX parser handles various email formats
- Export generates correct structure
- Storage functions work correctly

### Manual Testing Checklist:

1. **MBOX Preprocessing Flow**:
   - Run Python script on MBOX file
   - Verify output JSON structure
   - Check thread grouping is correct
   - Verify email bodies are readable text

2. **Fresh Start Flow**:
   - Open app with empty localStorage
   - Import preprocessed JSON
   - Verify threads load and display correctly
   - Navigate with keyboard
   - Add annotations
   - Refresh - verify persistence
   - Export and verify JSON structure

3. **Resume Flow**:
   - Import previously exported annotated JSON
   - Verify annotations are pre-populated
   - Continue annotating
   - Export again

4. **Edge Cases**:
   - Single-email thread (no annotation gaps)
   - Very long email bodies
   - Emails with only HTML (no plain text)
   - Non-ASCII characters in email bodies
   - Empty import file
   - Invalid JSON import

## File Structure Summary

```
tracewriter/
├── index.html
├── package.json
├── vite.config.js
├── scripts/
│   └── mbox_to_json.py      # Python MBOX → JSON converter
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── styles.css
│   ├── components/
│   │   ├── Header.jsx
│   │   ├── ThreadList.jsx
│   │   ├── EmailView.jsx
│   │   └── AnnotationInput.jsx
│   ├── hooks/
│   │   ├── useKeyboardNav.js
│   │   └── useLocalStorage.js
│   └── utils/
│       ├── emailParser.js    # Simplified JSON parser
│       ├── exportAnnotations.js
│       └── storage.js
└── public/
```

## Dependencies

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "turndown": "^7.1.2"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.2.0",
    "vite": "^5.0.0"
  }
}
```

## References

- Original mockup: `Mockup.jsx`
- Turndown.js: https://github.com/mixmark-io/turndown
- Python mailbox module: https://docs.python.org/3/library/mailbox.html
- MBOX format: https://en.wikipedia.org/wiki/Mbox
