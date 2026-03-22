# TraceWriter Multi-Thread Property UI Implementation Plan

## Overview

Update TraceWriter to support the new JSON schema where multiple email threads are nested under a single property. This requires a 3-tier hierarchy (Properties → Threads → Emails) with a property-first sidebar, new keyboard navigation, and updated breadcrumbs.

## Current State Analysis

**Existing Implementation (`tracewriter/src/App.jsx`):**
- 2-tier hierarchy: Threads → Emails
- Sidebar shows flat list of threads (lines 279-311)
- State: `threads[]`, `currentThreadIndex`, `currentEmailIndex`
- Keyboard: `Shift+Arrow` switches threads, `Arrow/Tab` navigates emails
- Annotation keys: `{threadId}:{emailIndex}`

**New JSON Schema (`parsed_grouped.json`):**
```
Property
├── id: "prop_7250_franklin"
├── subject: "7250 Franklin"
├── thread_count: 49
├── email_count: 183
└── threads[]
    └── Thread
        ├── id: "prop_7250_franklin_thread_0"
        ├── subject: "Re: Selling Agent Welcome..."
        ├── email_count: 7
        └── emails[]
            └── Email { id, from, to, date, dateDisplay, subject, body }
```

## Desired End State

After implementation:
1. **Sidebar** shows properties with expand/collapse for threads
2. **Header** shows full hierarchy: `properties / 7250 Franklin / TC Introduction...`
3. **Keyboard navigation**:
   - `Cmd+Shift+Arrow` switches properties
   - `Shift+Arrow` switches threads within current property
   - `Arrow/Tab` navigates emails within current thread
4. **Annotations** continue to work, keyed by `{threadId}:{emailIndex}`

### Verification
- Import `parsed_grouped.json` and see 22 properties in sidebar
- Click property to expand and see its threads
- Navigate between properties with `Cmd+Shift+Arrow`
- Navigate between threads with `Shift+Arrow`
- Annotations persist correctly across navigation

## What We're NOT Doing

- Progress tracking (user explicitly deferred this)
- Export functionality (already marked TODO)
- localStorage persistence
- Any backend changes

## Implementation Approach

The changes are contained to two files:
1. `emailParser.js` - Update parser to handle new nested schema
2. `App.jsx` - Update state, UI, and keyboard navigation

We'll work in 3 phases: Parser → State/Navigation → UI.

---

## Phase 1: Update Email Parser

### Overview
Update `parsePreprocessedJson` to handle the new property-nested schema and return properties instead of threads.

### Changes Required:

#### 1. emailParser.js - New Parser Logic
**File**: `tracewriter/src/utils/emailParser.js`

Replace the `parsePreprocessedJson` function to handle properties with nested threads:

```javascript
/**
 * Parse preprocessed JSON (from Python script) into internal format
 * New format: array of properties, each containing threads with emails
 */
export function parsePreprocessedJson(data) {
  const properties = Array.isArray(data) ? data : [data];

  return properties.map(property => ({
    id: property.id,
    subject: property.subject,
    property: property.property,
    threadCount: property.thread_count,
    emailCount: property.email_count,
    threads: property.threads.map(thread => ({
      id: thread.id,
      subject: thread.subject,
      normalizedSubject: thread.normalized_subject,
      emailCount: thread.email_count,
      emails: thread.emails.map((email, index) => ({
        id: email.id || `msg_${index}`,
        from: email.from,
        to: email.to,
        date: email.date,
        dateDisplay: email.dateDisplay || formatDateDisplay(email.date),
        subject: email.subject,
        body: email.body,
      })),
    })),
  }));
}
```

#### 2. emailParser.js - Update Annotation Detection
**File**: `tracewriter/src/utils/emailParser.js`

Update `hasExistingAnnotations` to check within the nested structure:

```javascript
/**
 * Check if imported JSON has pre-existing annotations
 */
export function hasExistingAnnotations(data) {
  const items = Array.isArray(data) ? data : [data];

  // Check if it's the new property-based format
  if (items[0]?.threads) {
    return items.some(property =>
      property.threads?.some(thread =>
        thread.emails?.some(email => '_annotation_after' in email)
      )
    );
  }

  // Legacy flat thread format
  return items.some(thread =>
    thread.emails?.some(email => '_annotation_after' in email) ||
    thread.messages?.some(msg => '_annotation_after' in msg)
  );
}
```

#### 3. emailParser.js - Update Annotated Export Parser
**File**: `tracewriter/src/utils/emailParser.js`

Update `parseAnnotatedExport` to handle the nested structure:

```javascript
/**
 * Parse pre-annotated JSON format (our export format)
 */
export function parseAnnotatedExport(data) {
  const items = Array.isArray(data) ? data : [data];
  const annotations = {};

  // Check if it's the new property-based format
  if (items[0]?.threads) {
    const properties = items.map(property => ({
      id: property.id,
      subject: property.subject,
      property: property.property,
      threadCount: property.thread_count || property.threadCount,
      emailCount: property.email_count || property.emailCount,
      threads: property.threads.map(thread => {
        const emails = thread.emails.map((email, index) => {
          if (email._annotation_after) {
            annotations[`${thread.id}:${index}`] = email._annotation_after;
          }
          return {
            id: email.id,
            from: email.from,
            to: email.to,
            date: email.date,
            dateDisplay: email.dateDisplay || formatDateDisplay(email.date),
            subject: email.subject,
            body: email.body,
          };
        });
        return {
          id: thread.id,
          subject: thread.subject,
          normalizedSubject: thread.normalized_subject || thread.normalizedSubject,
          emailCount: thread.email_count || thread.emailCount,
          emails,
        };
      }),
    }));
    return { properties, annotations };
  }

  // Legacy flat thread format - return as single property
  const parsedThreads = [];
  for (const thread of items) {
    const messages = thread.emails || thread.messages || [];
    const emails = messages.map((msg, index) => {
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
    parsedThreads.push({ id: thread.id, subject: thread.subject, emails });
  }

  // Wrap legacy format in a single property
  const properties = [{
    id: 'legacy_import',
    subject: 'Imported Threads',
    property: 'imported',
    threadCount: parsedThreads.length,
    emailCount: parsedThreads.reduce((sum, t) => sum + t.emails.length, 0),
    threads: parsedThreads,
  }];

  return { properties, annotations };
}
```

### Success Criteria:

#### Automated Verification:
- [x] No TypeScript/lint errors: `cd tracewriter && npm run lint`
- [x] App starts without errors: `cd tracewriter && npm run dev`

#### Manual Verification:
- [ ] Import `parsed_grouped.json` - no console errors
- [ ] Data loads (check React DevTools or console.log)

---

## Phase 2: Update State and Keyboard Navigation

### Overview
Change state from `threads[]` to `properties[]` with 3-level indices, and add `Cmd+Shift+Arrow` navigation.

### Changes Required:

#### 1. App.jsx - Update State Variables
**File**: `tracewriter/src/App.jsx`

Replace the state declarations (lines 37-44):

```javascript
// Create mock data in new format
const mockProperties = [
  {
    id: 'prop_mock',
    subject: 'Sample Property',
    property: 'sample property',
    threadCount: 2,
    emailCount: 5,
    threads: [
      {
        id: 'prop_mock_thread_0',
        subject: '123 Oak Street - Disclosure Documents',
        emailCount: 4,
        emails: [
          { id: 1, from: 'buyer@email.com', dateDisplay: 'Dec 12, 2:34 PM', body: 'Hi Matan, we reviewed the inspection report and have some concerns about the roof.' },
          { id: 2, from: 'matan@theagency.com', dateDisplay: 'Dec 12, 3:15 PM', body: 'Of course! I\'ll get that over to you right away.' },
          { id: 3, from: 'matan@theagency.com', dateDisplay: 'Dec 12, 3:45 PM', body: 'Here\'s the signed disclosure. Let me know if you have any questions.' },
          { id: 4, from: 'buyer@email.com', dateDisplay: 'Dec 12, 5:02 PM', body: 'Thanks! Can we get a copy of that roof inspection report as well?' },
        ]
      },
      {
        id: 'prop_mock_thread_1',
        subject: '123 Oak Street - Escrow Timeline',
        emailCount: 2,
        emails: [
          { id: 1, from: 'title@titleco.com', dateDisplay: 'Dec 11, 10:00 AM', body: 'We need the amended purchase agreement to proceed with escrow.' },
          { id: 2, from: 'matan@theagency.com', dateDisplay: 'Dec 11, 11:30 AM', body: 'Understood. I\'ll coordinate with both parties.' },
        ]
      }
    ]
  }
];

export default function App() {
  const [properties, setProperties] = useState(mockProperties);
  const [currentPropertyIndex, setCurrentPropertyIndex] = useState(0);
  const [currentThreadIndex, setCurrentThreadIndex] = useState(0);
  const [currentEmailIndex, setCurrentEmailIndex] = useState(0);
  const [annotations, setAnnotations] = useState({});
  const [focusedAnnotation, setFocusedAnnotation] = useState(null);
  const [expandedProperties, setExpandedProperties] = useState(new Set([mockProperties[0]?.id]));

  const currentProperty = properties[currentPropertyIndex] || { subject: 'No properties', threads: [] };
  const currentThread = currentProperty.threads[currentThreadIndex] || { subject: 'No threads', emails: [] };
  const totalProperties = properties.length;
  const totalThreads = currentProperty.threads.length;
  const totalEmails = currentThread.emails.length;
```

#### 2. App.jsx - Update Keyboard Navigation
**File**: `tracewriter/src/App.jsx`

Replace the keyboard handler useEffect (lines 51-149):

```javascript
  // Handle keyboard navigation
  useEffect(() => {
    const handleKeyDown = (e) => {
      const inEditMode = focusedAnnotation !== null;

      if (e.key === 'Tab') {
        e.preventDefault();
        if (e.shiftKey) {
          // Shift+Tab: previous email
          if (currentEmailIndex > 0) {
            const newIndex = currentEmailIndex - 1;
            setCurrentEmailIndex(newIndex);
            if (inEditMode) {
              setFocusedAnnotation(getAnnotationKey(currentThread.id, newIndex));
            }
          }
        } else {
          // Tab: next email
          if (currentEmailIndex < totalEmails - 1) {
            const newIndex = currentEmailIndex + 1;
            setCurrentEmailIndex(newIndex);
            if (inEditMode && newIndex < totalEmails - 1) {
              setFocusedAnnotation(getAnnotationKey(currentThread.id, newIndex));
            } else if (inEditMode) {
              setFocusedAnnotation(null);
            }
          }
        }
      } else if (e.key === 'ArrowDown' || e.key === 'ArrowUp') {
        e.preventDefault();

        if (e.metaKey && e.shiftKey) {
          // Cmd+Shift+Arrow: switch properties
          if (e.key === 'ArrowDown') {
            if (currentPropertyIndex < totalProperties - 1) {
              const nextProperty = properties[currentPropertyIndex + 1];
              setCurrentPropertyIndex(i => i + 1);
              setCurrentThreadIndex(0);
              setCurrentEmailIndex(0);
              setExpandedProperties(prev => new Set([...prev, nextProperty.id]));
              if (inEditMode && nextProperty.threads[0]?.emails.length > 1) {
                setFocusedAnnotation(getAnnotationKey(nextProperty.threads[0].id, 0));
              } else {
                setFocusedAnnotation(null);
              }
            }
          } else {
            if (currentPropertyIndex > 0) {
              const prevProperty = properties[currentPropertyIndex - 1];
              setCurrentPropertyIndex(i => i - 1);
              setCurrentThreadIndex(0);
              setCurrentEmailIndex(0);
              setExpandedProperties(prev => new Set([...prev, prevProperty.id]));
              if (inEditMode && prevProperty.threads[0]?.emails.length > 1) {
                setFocusedAnnotation(getAnnotationKey(prevProperty.threads[0].id, 0));
              } else {
                setFocusedAnnotation(null);
              }
            }
          }
        } else if (e.shiftKey) {
          // Shift+Arrow: switch threads within property
          if (e.key === 'ArrowDown') {
            if (currentThreadIndex < totalThreads - 1) {
              const nextThread = currentProperty.threads[currentThreadIndex + 1];
              setCurrentThreadIndex(i => i + 1);
              setCurrentEmailIndex(0);
              if (inEditMode && nextThread.emails.length > 1) {
                setFocusedAnnotation(getAnnotationKey(nextThread.id, 0));
              } else {
                setFocusedAnnotation(null);
              }
            }
          } else {
            if (currentThreadIndex > 0) {
              const prevThread = currentProperty.threads[currentThreadIndex - 1];
              setCurrentThreadIndex(i => i - 1);
              setCurrentEmailIndex(0);
              if (inEditMode && prevThread.emails.length > 1) {
                setFocusedAnnotation(getAnnotationKey(prevThread.id, 0));
              } else {
                setFocusedAnnotation(null);
              }
            }
          }
        } else {
          // Arrow without modifiers: navigate emails
          if (e.key === 'ArrowDown') {
            if (currentEmailIndex < totalEmails - 1) {
              const newIndex = currentEmailIndex + 1;
              setCurrentEmailIndex(newIndex);
              if (inEditMode && newIndex < totalEmails - 1) {
                setFocusedAnnotation(getAnnotationKey(currentThread.id, newIndex));
              } else if (inEditMode) {
                setFocusedAnnotation(null);
              }
            }
          } else {
            if (currentEmailIndex > 0) {
              const newIndex = currentEmailIndex - 1;
              setCurrentEmailIndex(newIndex);
              if (inEditMode) {
                setFocusedAnnotation(getAnnotationKey(currentThread.id, newIndex));
              }
            }
          }
        }
      } else if (e.key === 'Enter' && !inEditMode && !e.shiftKey) {
        e.preventDefault();
        if (currentEmailIndex < totalEmails - 1) {
          setFocusedAnnotation(getAnnotationKey(currentThread.id, currentEmailIndex));
        }
      } else if (e.key === 'Escape') {
        setFocusedAnnotation(null);
        document.activeElement.blur();
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [currentEmailIndex, currentThreadIndex, currentPropertyIndex, totalEmails, totalThreads, totalProperties, currentThread.id, currentProperty, properties, focusedAnnotation]);
```

#### 3. App.jsx - Update Import Handler
**File**: `tracewriter/src/App.jsx`

Update the `handleImport` function:

```javascript
  // Handle JSON import (from preprocessed MBOX or annotated export)
  const handleImport = (json) => {
    if (hasExistingAnnotations(json)) {
      // Re-importing annotated export
      const { properties: parsed, annotations: existingAnnotations } = parseAnnotatedExport(json);
      setProperties(parsed);
      setAnnotations(prev => ({ ...prev, ...existingAnnotations }));
    } else {
      // Fresh preprocessed JSON from Python script
      const parsed = parsePreprocessedJson(json);
      setProperties(parsed);
    }
    setCurrentPropertyIndex(0);
    setCurrentThreadIndex(0);
    setCurrentEmailIndex(0);
    setFocusedAnnotation(null);
    // Expand first property by default
    if (json.length > 0) {
      setExpandedProperties(new Set([json[0]?.id || 'prop_0']));
    }
  };
```

### Success Criteria:

#### Automated Verification:
- [x] No lint errors: `cd tracewriter && npm run lint`
- [x] App starts: `cd tracewriter && npm run dev`

#### Manual Verification:
- [ ] `Cmd+Shift+Down` moves to next property
- [ ] `Cmd+Shift+Up` moves to previous property
- [ ] `Shift+Down` moves to next thread within property
- [ ] `Shift+Up` moves to previous thread within property
- [ ] Arrow keys still navigate emails

---

## Phase 3: Update UI Components

### Overview
Update the sidebar to show properties with expandable threads, and update header breadcrumbs.

### Changes Required:

#### 1. App.jsx - Update Header Breadcrumbs
**File**: `tracewriter/src/App.jsx`

Replace the header section (lines 200-276):

```javascript
      {/* Header */}
      <div style={{
        borderBottom: '1px solid #2a2725',
        padding: '12px 20px',
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center'
      }}>
        <div style={{ display: 'flex', alignItems: 'center', gap: '12px' }}>
          <span style={{ color: '#8a8480' }}>properties</span>
          <span style={{ color: '#4a4745' }}>/</span>
          <span style={{ color: '#c9a86c' }}>{currentProperty.subject}</span>
          <span style={{ color: '#4a4745' }}>/</span>
          <span style={{
            color: '#8a8480',
            maxWidth: '400px',
            overflow: 'hidden',
            textOverflow: 'ellipsis',
            whiteSpace: 'nowrap'
          }}>
            {currentThread.subject}
          </span>
        </div>
        <div style={{ display: 'flex', alignItems: 'center', gap: '16px' }}>
          <button
            onClick={() => document.getElementById('file-input').click()}
            style={{
              backgroundColor: '#252220',
              border: '1px solid #3a3735',
              borderRadius: '4px',
              color: '#c9a86c',
              padding: '6px 12px',
              cursor: 'pointer',
              fontFamily: 'inherit',
              fontSize: '12px',
            }}
          >
            Import JSON
          </button>
          <input
            id="file-input"
            type="file"
            accept=".json"
            style={{ display: 'none' }}
            onChange={(e) => {
              const file = e.target.files[0];
              if (file) {
                file.text().then(text => {
                  try {
                    const json = JSON.parse(text);
                    handleImport(json);
                  } catch (err) {
                    alert(`Failed to parse JSON: ${err.message}`);
                  }
                });
              }
              e.target.value = '';
            }}
          />
          {totalProperties > 0 && (
            <button
              onClick={handleExport}
              style={{
                backgroundColor: '#252220',
                border: '1px solid #3a3735',
                borderRadius: '4px',
                color: '#8a8480',
                padding: '6px 12px',
                cursor: 'pointer',
                fontFamily: 'inherit',
                fontSize: '12px',
              }}
            >
              Export
            </button>
          )}
        </div>
      </div>
```

#### 2. App.jsx - Update Sidebar with Property/Thread Hierarchy
**File**: `tracewriter/src/App.jsx`

Replace the sidebar section (lines 278-311):

```javascript
        {/* Property/Thread list sidebar */}
        <div style={{
          width: '300px',
          borderRight: '1px solid #2a2725',
          padding: '12px 0',
          overflowY: 'auto'
        }}>
          {properties.map((property, propIdx) => (
            <div key={property.id}>
              {/* Property header */}
              <div
                onClick={() => {
                  setExpandedProperties(prev => {
                    const next = new Set(prev);
                    if (next.has(property.id)) {
                      next.delete(property.id);
                    } else {
                      next.add(property.id);
                    }
                    return next;
                  });
                  if (propIdx !== currentPropertyIndex) {
                    setCurrentPropertyIndex(propIdx);
                    setCurrentThreadIndex(0);
                    setCurrentEmailIndex(0);
                    setFocusedAnnotation(null);
                  }
                }}
                style={{
                  padding: '10px 16px',
                  cursor: 'pointer',
                  backgroundColor: propIdx === currentPropertyIndex ? '#252220' : 'transparent',
                  borderLeft: propIdx === currentPropertyIndex ? '2px solid #c9a86c' : '2px solid transparent',
                  display: 'flex',
                  alignItems: 'center',
                  gap: '8px',
                }}
              >
                <span style={{
                  color: '#5a5755',
                  fontSize: '10px',
                  width: '12px',
                }}>
                  {expandedProperties.has(property.id) ? '▼' : '▶'}
                </span>
                <div style={{ flex: 1, minWidth: 0 }}>
                  <div style={{
                    color: propIdx === currentPropertyIndex ? '#c9a86c' : '#e8e4df',
                    fontWeight: propIdx === currentPropertyIndex ? '500' : '400',
                    whiteSpace: 'nowrap',
                    overflow: 'hidden',
                    textOverflow: 'ellipsis'
                  }}>
                    {property.subject}
                  </div>
                  <div style={{ color: '#5a5755', fontSize: '11px' }}>
                    {property.threadCount} threads · {property.emailCount} emails
                  </div>
                </div>
              </div>

              {/* Threads within property */}
              {expandedProperties.has(property.id) && (
                <div style={{ marginLeft: '20px' }}>
                  {property.threads.map((thread, threadIdx) => (
                    <div
                      key={thread.id}
                      onClick={() => {
                        setCurrentPropertyIndex(propIdx);
                        setCurrentThreadIndex(threadIdx);
                        setCurrentEmailIndex(0);
                        setFocusedAnnotation(null);
                      }}
                      style={{
                        padding: '8px 16px',
                        cursor: 'pointer',
                        backgroundColor: propIdx === currentPropertyIndex && threadIdx === currentThreadIndex ? '#1e1c1a' : 'transparent',
                        borderLeft: propIdx === currentPropertyIndex && threadIdx === currentThreadIndex ? '2px solid #7a9f6a' : '2px solid transparent',
                      }}
                    >
                      <div style={{
                        color: propIdx === currentPropertyIndex && threadIdx === currentThreadIndex ? '#e8e4df' : '#8a8480',
                        fontSize: '12px',
                        whiteSpace: 'nowrap',
                        overflow: 'hidden',
                        textOverflow: 'ellipsis'
                      }}>
                        {thread.subject}
                      </div>
                      <div style={{ color: '#5a5755', fontSize: '10px' }}>
                        {thread.emailCount || thread.emails?.length || 0} emails
                      </div>
                    </div>
                  ))}
                </div>
              )}
            </div>
          ))}
        </div>
```

#### 3. App.jsx - Update Footer Keyboard Hints
**File**: `tracewriter/src/App.jsx`

Replace the footer section (lines 402-437):

```javascript
      {/* Footer / keyboard hints */}
      <div style={{
        position: 'fixed',
        bottom: 0,
        left: 0,
        right: 0,
        backgroundColor: '#1a1816',
        borderTop: '1px solid #2a2725',
        padding: '10px 20px',
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center'
      }}>
        <div style={{ display: 'flex', gap: '20px', flexWrap: 'wrap' }}>
          <span style={{ color: '#5a5755' }}>
            <span style={{ color: '#8a8480', backgroundColor: '#252220', padding: '2px 6px', borderRadius: '3px', marginRight: '6px' }}>↑/↓</span>
            emails
          </span>
          <span style={{ color: '#5a5755' }}>
            <span style={{ color: '#8a8480', backgroundColor: '#252220', padding: '2px 6px', borderRadius: '3px', marginRight: '6px' }}>⇧↑/↓</span>
            threads
          </span>
          <span style={{ color: '#5a5755' }}>
            <span style={{ color: '#8a8480', backgroundColor: '#252220', padding: '2px 6px', borderRadius: '3px', marginRight: '6px' }}>⌘⇧↑/↓</span>
            properties
          </span>
          <span style={{ color: '#5a5755' }}>
            <span style={{ color: '#8a8480', backgroundColor: '#252220', padding: '2px 6px', borderRadius: '3px', marginRight: '6px' }}>Enter</span>
            edit
          </span>
          <span style={{ color: '#5a5755' }}>
            <span style={{ color: '#8a8480', backgroundColor: '#252220', padding: '2px 6px', borderRadius: '3px', marginRight: '6px' }}>Esc</span>
            view
          </span>
        </div>
        <div style={{ color: '#5a5755', fontSize: '12px' }}>
          <span style={{ color: '#c9a86c' }}>TRACEWRITER</span>
          <span style={{ marginLeft: '12px' }}>
            {currentPropertyIndex + 1}/{totalProperties} properties
            {' · '}
            {currentThreadIndex + 1}/{totalThreads} threads
            {' · '}
            email {currentEmailIndex + 1}/{totalEmails}
          </span>
        </div>
      </div>
```

### Success Criteria:

#### Automated Verification:
- [x] No lint errors: `cd tracewriter && npm run lint`
- [x] App builds successfully: `cd tracewriter && npm run build`

#### Manual Verification:
- [ ] Sidebar shows properties with expand/collapse arrows
- [ ] Clicking property expands to show threads
- [ ] Clicking thread selects it and shows emails in main area
- [ ] Header shows `properties / [Property Name] / [Thread Subject]`
- [ ] Footer shows updated keyboard hints with symbols
- [ ] Footer shows property/thread/email counts

---

## Testing Strategy

### After All Phases Complete:

1. **Import Test**: Load `parsed_grouped.json` (22 properties, ~1400 threads)
2. **Navigation Test**:
   - Use `Cmd+Shift+Arrow` to cycle through all properties
   - Use `Shift+Arrow` to cycle through threads
   - Use `Arrow` to navigate emails
3. **Annotation Test**:
   - Press `Enter` to edit an annotation gap
   - Type annotation text
   - Navigate away and back - annotation should persist
4. **Edge Cases**:
   - First/last property boundary (shouldn't wrap or error)
   - First/last thread boundary within property
   - Single-email threads (no annotation gaps)
   - Empty properties (if any exist)

---

## References

- Current implementation: `tracewriter/src/App.jsx`
- Parser utility: `tracewriter/src/utils/emailParser.js`
- New data format: `parsed_grouped.json`
- Previous handoff: `thoughts/shared/handoffs/general/2025-12-18_20-14-07_tracewriter-property-grouping.md`
