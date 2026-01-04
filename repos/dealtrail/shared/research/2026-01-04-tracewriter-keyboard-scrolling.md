---
date: 2026-01-04T16:09:53+07:00
researcher: ariasulin
git_commit: 439e2e150308e7e33196c6768ad7c07b9e55b348
branch: main
repository: Dealtrail
topic: "TraceWriter keyboard navigation, context menu, and scrolling behavior"
tags: [research, tracewriter, keyboard-navigation, scrolling, ui]
status: complete
last_updated: 2026-01-04
last_updated_by: ariasulin
---

# Research: TraceWriter Keyboard Navigation, Context Menu, and Scrolling

**Date**: 2026-01-04 16:09:53 +07
**Researcher**: ariasulin
**Git Commit**: 439e2e150308e7e33196c6768ad7c07b9e55b348
**Branch**: main
**Repository**: Dealtrail

## Research Question

Document the current state of TraceWriter's context menu bar, keyboard shortcuts, and scrolling behavior. Analyze futureSelf's scrolling implementation as a reference for how scrolling should work.

## Summary

TraceWriter has keyboard navigation implemented but **lacks auto-scroll functionality** - navigating with arrow keys changes the selected email but does not scroll the view to keep the selected email visible. The footer bar (context menu) displays keyboard hints but is static and working correctly. The futureSelf application demonstrates a hybrid scroll strategy using refs and `scrollIntoView`/`scrollTo` that should be adopted.

## Detailed Findings

### TraceWriter Current Keyboard Navigation

**File**: `tracewriter/src/App.jsx:71-190`

#### Keyboard Bindings (Currently Implemented)

| Key | Action | Line |
|-----|--------|------|
| `Tab` | Next email | 86-96 |
| `Shift+Tab` | Previous email | 77-85 |
| `ArrowDown` | Next email | 157-165 |
| `ArrowUp` | Previous email | 167-175 |
| `Shift+ArrowDown` | Next thread | 131-141 |
| `Shift+ArrowUp` | Previous thread | 143-154 |
| `Cmd+Shift+ArrowDown` | Next property | 102-115 |
| `Cmd+Shift+ArrowUp` | Previous property | 116-129 |
| `Enter` | Edit annotation | 177-181 |
| `Escape` | Exit edit mode | 182-185 |

#### What's Missing

**No auto-scroll on navigation** - The keyboard handler updates `currentEmailIndex` state but never scrolls the view:

```javascript
// App.jsx:157-165 - Only updates state, no scroll
if (e.key === 'ArrowDown') {
  if (currentEmailIndex < totalEmails - 1) {
    const newIndex = currentEmailIndex + 1;
    setCurrentEmailIndex(newIndex);
    // ... no scrollIntoView call
  }
}
```

**No refs for scroll targets** - The email list items have no refs attached:

```javascript
// App.jsx:429-450 - No ref on email container
<div style={{
  marginBottom: '8px',
  padding: '16px',
  backgroundColor: idx === currentEmailIndex ? '#252220' : 'transparent',
  // ...
}}>
```

**No container ref for scroll control** - The main content area has no ref:

```javascript
// App.jsx:428 - No ref on scrollable container
<div style={{ flex: 1, overflowY: 'auto', padding: '20px 32px' }}>
```

### TraceWriter Context Menu / Footer Bar

**File**: `tracewriter/src/App.jsx:516-561`

The footer is a fixed-position bar at the bottom showing keyboard hints and current position:

```javascript
// App.jsx:516-528 - Fixed footer structure
<div style={{
  position: 'fixed',
  bottom: 0,
  left: 0,
  right: 0,
  // ...
}}>
```

#### Current Hint Display (Lines 529-549)

| Hint | Description | Displayed As |
|------|-------------|--------------|
| `↑/↓` | Navigate emails | `emails` |
| `⇧↑/↓` | Navigate threads | `threads` |
| `⌘⇧↑/↓` | Navigate properties | `properties` |
| `Enter` | Edit annotation | `edit` |
| `Esc` | Exit edit mode | `view` |

#### Position Counter (Lines 551-559)

Shows: `1/1 properties · 1/2 threads · email 1/4`

**The footer/context menu itself is working correctly** - it displays the hints and updates the position counter. The issue is not with the footer.

### futureSelf Scrolling Implementation (Reference)

**Key files in `/Users/ariasulin/Git/futureSelf/`:**

- `src/components/NoteList.jsx` - Scroll implementation
- `src/components/Note.jsx` - forwardRef pattern
- `src/hooks/useKeyboardNavigation.js` - Keyboard handling
- `src/styles/components/NoteList.css` - Scroll container CSS

#### Pattern 1: Ref Setup

```javascript
// NoteList.jsx:14-15
const noteRefs = useRef([]);        // Array for list items
const containerRef = useRef(null);   // Container for scrollTo

// NoteList.jsx:52 - Container ref
<div className="note-list__notes" ref={containerRef}>

// NoteList.jsx:56 - Item refs via callback
ref={el => noteRefs.current[index] = el}
```

#### Pattern 2: forwardRef for Child Components

```javascript
// Note.jsx:11-22
export const Note = forwardRef(function Note({ ... }, ref) {
  return (
    <div ref={ref} className="note">
      {/* content */}
    </div>
  );
});
```

#### Pattern 3: Hybrid Scroll Strategy

```javascript
// NoteList.jsx:18-37
useEffect(() => {
  if (noteRefs.current[selectedNoteIndex]) {
    const isFirst = selectedNoteIndex === 0;
    const isLast = selectedNoteIndex === thread.notes.length - 1;

    if (isFirst && containerRef.current) {
      // First item: scroll to absolute top (includes padding)
      containerRef.current.scrollTo({ top: 0, behavior: 'smooth' });
    } else if (isLast && containerRef.current) {
      // Last item: scroll to absolute bottom
      containerRef.current.scrollTo({ top: containerRef.current.scrollHeight, behavior: 'smooth' });
    } else {
      // Middle items: minimal scroll movement
      noteRefs.current[selectedNoteIndex].scrollIntoView({
        behavior: 'smooth',
        block: 'nearest',
      });
    }
  }
}, [selectedNoteIndex, thread]);
```

**Why hybrid approach?**
- `scrollIntoView({ block: 'nearest' })` doesn't account for container padding
- `scrollTo({ top: 0 })` and `scrollTo({ top: scrollHeight })` reach the absolute edges
- Middle items use `scrollIntoView` for minimal movement

#### Pattern 4: CSS Scroll Container

```css
/* NoteList.css:1-27 */
.note-list {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  min-height: 0;  /* Critical for flex scrolling */
}

.note-list__notes {
  flex: 1;
  overflow-y: auto;
  padding: var(--spacing-md) var(--spacing-xl);
  min-height: 0;  /* Critical for flex scrolling */
}
```

## Code References

**TraceWriter (current - missing scroll):**
- `tracewriter/src/App.jsx:71-190` - Keyboard handler with no scroll
- `tracewriter/src/App.jsx:428` - Scroll container with no ref
- `tracewriter/src/App.jsx:429-450` - Email items with no refs
- `tracewriter/src/App.jsx:516-561` - Footer/context menu (working)

**futureSelf (reference - working scroll):**
- `/Users/ariasulin/Git/futureSelf/src/components/NoteList.jsx:14-37` - Ref setup and scroll effect
- `/Users/ariasulin/Git/futureSelf/src/components/Note.jsx:11-22` - forwardRef pattern
- `/Users/ariasulin/Git/futureSelf/src/hooks/useKeyboardNavigation.js:97-134` - Arrow key handling
- `/Users/ariasulin/Git/futureSelf/src/styles/components/NoteList.css:22-27` - Scroll CSS

## Architecture Documentation

### TraceWriter Current Structure

```
App.jsx
├── State: currentEmailIndex, currentThreadIndex, currentPropertyIndex
├── Keyboard Handler: useEffect with window.addEventListener('keydown')
├── Sidebar: Property/Thread list (has overflow-y: auto)
└── Main Content: Email list (has overflow-y: auto, NO refs)
```

### Required Changes for Scroll Support

1. **Add refs**: `emailRefs = useRef([])` and `containerRef = useRef(null)`
2. **Attach refs**: Add callback ref to email items, container ref to main content div
3. **Add scroll effect**: useEffect that triggers on `currentEmailIndex` changes
4. **Use hybrid strategy**: scrollTo for first/last, scrollIntoView for middle

## Historical Context (from thoughts/)

- `thoughts/shared/plans/2026-01-03-tracewriter-vercel-deployment.md` - Recent deployment plan, mentions keyboard-first navigation as key feature but doesn't address scroll behavior
- `thoughts/shared/plans/2024-12-16-tracewriter-email-annotation-tool.md` - Original implementation spec (referenced in CLAUDE.md)

## Open Questions

1. Should the sidebar (property/thread list) also auto-scroll when navigating with keyboard?
2. Should `Tab`/`Shift+Tab` navigation also trigger scroll, or only arrow keys?
3. Should there be visual feedback when scroll occurs (brief highlight animation)?
