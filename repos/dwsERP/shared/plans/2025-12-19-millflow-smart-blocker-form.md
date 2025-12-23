# Smart Blocker Form Implementation Plan

## Overview

Replace the current 4-field BlockerForm in millflow with a **single smart input** that parses natural language in real-time. Users can type a sentence like "waiting on material from Acme, arrives friday" and the form automatically extracts type, owner, and resolution date. Includes `/templates` slash command for quick autofill of common blockers.

## Current State Analysis

**Current Location**: `millflow/src/components/node/BlockerForm.tsx`

The existing form has 4 separate fields:
1. Type selector (5 toggle buttons)
2. Description text input
3. Owner text input
4. Expected resolution date picker (native HTML date input)

**Pain Points**:
- Requires 4 separate interactions to add a blocker
- No autofill for owner field
- No magic date parsing
- Slow for quick entry during workflow

**Existing Patterns to Leverage**:
- `AssigneePicker.tsx` - Has fuzzy search and keyboard navigation
- `DueDatePicker.tsx` - Has quick date buttons (Tomorrow, In 3 days, etc.)
- `people` array in store - Available for owner autofill

## Desired End State

A single-input blocker form that:
1. Parses natural language to extract type, owner, and date
2. Shows live preview of parsed data with editable chips
3. Supports `/` slash commands for quick templates
4. Autofills owner from people list with fuzzy matching
5. Parses magic dates like "tomorrow", "friday", "next week"

**Verification**:
- User can type "waiting on material from Acme, arrives friday" and submit with Cmd+Enter
- Form correctly parses: type=material, owner=Acme Hardware Co., date=next Friday
- Typing `/` shows template menu, selecting one autofills the input

## What We're NOT Doing

- No AI/ML integration - simple keyword matching only
- No backend changes - all parsing is client-side
- No changes to Blocker type definition
- No changes to how blockers are stored/displayed elsewhere
- No internationalization of date parsing

## Implementation Approach

Build a new `SmartBlockerForm.tsx` component that replaces the existing `BlockerForm.tsx`. Create parsing utilities in a separate file for testability. Use the existing modal overlay pattern from `SheetView.tsx`.

---

## Phase 1: Parsing Utilities

### Overview
Create utility functions for parsing natural language input into blocker fields.

### Changes Required:

#### 1. Create Parser Module
**File**: `millflow/src/lib/blockerParser.ts` (new file)

```typescript
import type { BlockerType } from '@/types';
import type { Person } from '@/types';

// Blocker type detection from keywords
const TYPE_KEYWORDS: Record<BlockerType, string[]> = {
  material: ['material', 'materials', 'supply', 'supplies', 'parts', 'hardware', 'delivery', 'shipment', 'order'],
  approval: ['approval', 'approve', 'sign off', 'signoff', 'confirm', 'permission', 'authorize'],
  external: ['vendor', 'supplier', 'contractor', 'external', 'third party', 'outside'],
  site: ['site', 'location', 'access', 'gc', 'general contractor', 'field'],
  internal: ['internal', 'shop', 'team', 'staff', 'capacity'],
};

export function detectBlockerType(text: string): BlockerType {
  const lower = text.toLowerCase();

  for (const [type, keywords] of Object.entries(TYPE_KEYWORDS)) {
    if (keywords.some(kw => lower.includes(kw))) {
      return type as BlockerType;
    }
  }

  return 'internal'; // default
}

// Magic date parsing
export function parseMagicDate(text: string): string | undefined {
  const lower = text.toLowerCase();
  const today = new Date();

  // Relative dates
  if (lower.includes('tomorrow')) {
    return addDays(today, 1);
  }
  if (lower.includes('next week')) {
    return addDays(today, 7);
  }
  if (/in (\d+) days?/.test(lower)) {
    const match = lower.match(/in (\d+) days?/);
    if (match) return addDays(today, parseInt(match[1]));
  }

  // Day names (find next occurrence)
  const days = ['sunday', 'monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday'];
  for (let i = 0; i < days.length; i++) {
    if (lower.includes(days[i])) {
      return getNextDayOfWeek(today, i);
    }
  }

  // End of week/month
  if (lower.includes('end of week') || lower.includes('eow')) {
    return getNextDayOfWeek(today, 5); // Friday
  }
  if (lower.includes('end of month') || lower.includes('eom')) {
    return getEndOfMonth(today);
  }

  return undefined;
}

// Owner matching with fuzzy search
export function findOwnerMatch(text: string, people: Person[]): Person | undefined {
  const lower = text.toLowerCase();

  // Look for "from X" or "by X" or "owner: X" patterns
  const patterns = [
    /from\s+([a-z\s]+?)(?:,|$|\s+(?:by|should|will|arrives?))/i,
    /by\s+([a-z\s]+?)(?:,|$|\s+(?:should|will|arrives?))/i,
    /owner[:\s]+([a-z\s]+?)(?:,|$)/i,
    /waiting on\s+([a-z\s]+?)(?:,|$|\s+(?:for|to))/i,
  ];

  for (const pattern of patterns) {
    const match = text.match(pattern);
    if (match) {
      const name = match[1].trim().toLowerCase();
      // Fuzzy match against people list
      const found = people.find(p =>
        p.name.toLowerCase().includes(name) ||
        name.includes(p.name.toLowerCase().split(' ')[0])
      );
      if (found) return found;
    }
  }

  // Also check if any person name appears in the text
  for (const person of people) {
    const firstName = person.name.split(' ')[0].toLowerCase();
    const lastName = person.name.split(' ').slice(-1)[0].toLowerCase();
    if (lower.includes(firstName) || lower.includes(lastName) ||
        lower.includes(person.name.toLowerCase())) {
      return person;
    }
  }

  return undefined;
}

// Helper functions
function addDays(date: Date, days: number): string {
  const result = new Date(date);
  result.setDate(result.getDate() + days);
  return result.toISOString().split('T')[0];
}

function getNextDayOfWeek(date: Date, dayOfWeek: number): string {
  const result = new Date(date);
  const currentDay = result.getDay();
  const daysUntil = (dayOfWeek - currentDay + 7) % 7 || 7;
  result.setDate(result.getDate() + daysUntil);
  return result.toISOString().split('T')[0];
}

function getEndOfMonth(date: Date): string {
  const result = new Date(date.getFullYear(), date.getMonth() + 1, 0);
  return result.toISOString().split('T')[0];
}

// Full parse result
export interface ParsedBlocker {
  type: BlockerType;
  description: string;
  owner?: Person;
  ownerText: string;
  expectedResolution?: string;
  dateText?: string;
}

export function parseBlockerInput(text: string, people: Person[]): ParsedBlocker {
  return {
    type: detectBlockerType(text),
    description: text.trim(),
    owner: findOwnerMatch(text, people),
    ownerText: findOwnerMatch(text, people)?.name || '',
    expectedResolution: parseMagicDate(text),
    dateText: extractDateText(text),
  };
}

function extractDateText(text: string): string | undefined {
  const patterns = [
    /(tomorrow)/i,
    /(next week)/i,
    /(in \d+ days?)/i,
    /(monday|tuesday|wednesday|thursday|friday|saturday|sunday)/i,
    /(end of week|eow)/i,
    /(end of month|eom)/i,
  ];

  for (const pattern of patterns) {
    const match = text.match(pattern);
    if (match) return match[1];
  }
  return undefined;
}
```

### Success Criteria:

#### Automated Verification:
- [x] File exists: `millflow/src/lib/blockerParser.ts`
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [x] N/A for this phase (utilities only)

---

## Phase 2: Blocker Templates

### Overview
Define quick templates that users can access via `/` slash command.

### Changes Required:

#### 1. Create Templates Data
**File**: `millflow/src/data/blockerTemplates.ts` (new file)

```typescript
import type { BlockerType } from '@/types';

export interface BlockerTemplate {
  id: string;
  label: string;
  shortcut: string; // what user types after /
  type: BlockerType;
  descriptionTemplate: string;
  ownerPlaceholder?: string;
}

export const blockerTemplates: BlockerTemplate[] = [
  {
    id: 'material-waiting',
    label: 'Waiting on material',
    shortcut: 'material',
    type: 'material',
    descriptionTemplate: 'Waiting on material from {owner}',
    ownerPlaceholder: 'vendor',
  },
  {
    id: 'material-delayed',
    label: 'Material delayed',
    shortcut: 'delayed',
    type: 'material',
    descriptionTemplate: 'Material delivery delayed - {owner}',
    ownerPlaceholder: 'vendor',
  },
  {
    id: 'approval-needed',
    label: 'Need approval',
    shortcut: 'approval',
    type: 'approval',
    descriptionTemplate: 'Waiting for approval from {owner}',
    ownerPlaceholder: 'PM or client',
  },
  {
    id: 'site-access',
    label: 'Site not ready',
    shortcut: 'site',
    type: 'site',
    descriptionTemplate: 'Site access not confirmed - waiting on {owner}',
    ownerPlaceholder: 'GC',
  },
  {
    id: 'gc-confirm',
    label: 'GC confirmation needed',
    shortcut: 'gc',
    type: 'site',
    descriptionTemplate: 'Waiting for GC confirmation - {owner}',
    ownerPlaceholder: 'GC contact',
  },
  {
    id: 'vendor-delay',
    label: 'Vendor unreliable',
    shortcut: 'vendor',
    type: 'external',
    descriptionTemplate: 'Vendor delayed/unreliable - {owner}',
    ownerPlaceholder: 'vendor name',
  },
  {
    id: 'capacity',
    label: 'Shop capacity',
    shortcut: 'capacity',
    type: 'internal',
    descriptionTemplate: 'Shop at capacity - need to reschedule',
  },
  {
    id: 'rework',
    label: 'Rework needed',
    shortcut: 'rework',
    type: 'internal',
    descriptionTemplate: 'Rework required - {reason}',
  },
];

export function findTemplateByShortcut(shortcut: string): BlockerTemplate | undefined {
  const lower = shortcut.toLowerCase();
  return blockerTemplates.find(t =>
    t.shortcut.toLowerCase().startsWith(lower) ||
    t.label.toLowerCase().includes(lower)
  );
}

export function filterTemplates(query: string): BlockerTemplate[] {
  if (!query) return blockerTemplates;
  const lower = query.toLowerCase();
  return blockerTemplates.filter(t =>
    t.shortcut.toLowerCase().includes(lower) ||
    t.label.toLowerCase().includes(lower)
  );
}
```

### Success Criteria:

#### Automated Verification:
- [x] File exists: `millflow/src/data/blockerTemplates.ts`
- [x] TypeScript compiles: `cd millflow && npm run build`

#### Manual Verification:
- [x] N/A for this phase (data only)

---

## Phase 3: Smart Blocker Form Component

### Overview
Build the new SmartBlockerForm component with live parsing, template menu, and preview.

### Changes Required:

#### 1. Create Smart Blocker Form
**File**: `millflow/src/components/node/SmartBlockerForm.tsx` (new file)

```typescript
import { memo, useState, useRef, useEffect, useMemo } from 'react';
import { AlertCircle, Calendar, User, Tag, Zap } from 'lucide-react';
import { useAppStore } from '@/store/appStore';
import { parseBlockerInput, type ParsedBlocker } from '@/lib/blockerParser';
import { blockerTemplates, filterTemplates, type BlockerTemplate } from '@/data/blockerTemplates';
import { formatDate, cn } from '@/lib/styles';
import type { BlockerType } from '@/types';

interface SmartBlockerFormProps {
  onSave: (blocker: { type: BlockerType; description: string; owner: string; expected_resolution?: string }) => void;
  onCancel: () => void;
}

const typeColors: Record<BlockerType, string> = {
  internal: 'bg-gruvbox-purple-bright/20 text-gruvbox-purple-bright',
  external: 'bg-gruvbox-blue-bright/20 text-gruvbox-blue-bright',
  material: 'bg-gruvbox-orange-bright/20 text-gruvbox-orange-bright',
  approval: 'bg-gruvbox-yellow-bright/20 text-gruvbox-yellow-bright',
  site: 'bg-gruvbox-aqua-bright/20 text-gruvbox-aqua-bright',
};

export const SmartBlockerForm = memo(function SmartBlockerForm({
  onSave,
  onCancel,
}: SmartBlockerFormProps) {
  const { people } = useAppStore();
  const [input, setInput] = useState('');
  const [showTemplates, setShowTemplates] = useState(false);
  const [templateFilter, setTemplateFilter] = useState('');
  const [selectedTemplateIndex, setSelectedTemplateIndex] = useState(0);
  const [manualOverrides, setManualOverrides] = useState<Partial<ParsedBlocker>>({});
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  // Parse input in real-time
  const parsed = useMemo(() => {
    const result = parseBlockerInput(input, people);
    return { ...result, ...manualOverrides };
  }, [input, people, manualOverrides]);

  // Filter templates when in template mode
  const filteredTemplates = useMemo(() => {
    return filterTemplates(templateFilter);
  }, [templateFilter]);

  // Detect slash command
  useEffect(() => {
    if (input.startsWith('/')) {
      setShowTemplates(true);
      setTemplateFilter(input.slice(1));
      setSelectedTemplateIndex(0);
    } else {
      setShowTemplates(false);
      setTemplateFilter('');
    }
  }, [input]);

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (showTemplates) {
      // Template navigation
      if (e.key === 'ArrowDown') {
        e.preventDefault();
        setSelectedTemplateIndex(i => Math.min(i + 1, filteredTemplates.length - 1));
      } else if (e.key === 'ArrowUp') {
        e.preventDefault();
        setSelectedTemplateIndex(i => Math.max(i - 1, 0));
      } else if (e.key === 'Enter' || e.key === 'Tab') {
        e.preventDefault();
        if (filteredTemplates[selectedTemplateIndex]) {
          applyTemplate(filteredTemplates[selectedTemplateIndex]);
        }
      } else if (e.key === 'Escape') {
        e.preventDefault();
        setShowTemplates(false);
        setInput('');
      }
      return;
    }

    // Normal input mode
    if (e.key === 'Enter' && (e.metaKey || e.ctrlKey) && input.trim()) {
      e.preventDefault();
      handleSave();
    } else if (e.key === 'Escape') {
      e.preventDefault();
      onCancel();
    }
  };

  const applyTemplate = (template: BlockerTemplate) => {
    setInput(template.descriptionTemplate.replace('{owner}', '').replace('{reason}', ''));
    setManualOverrides({ type: template.type });
    setShowTemplates(false);
    // Focus and move cursor to end
    setTimeout(() => {
      inputRef.current?.focus();
      inputRef.current?.setSelectionRange(input.length, input.length);
    }, 0);
  };

  const handleSave = () => {
    onSave({
      type: parsed.type,
      description: parsed.description,
      owner: parsed.owner?.name || parsed.ownerText || 'Unknown',
      expected_resolution: parsed.expectedResolution,
    });
  };

  const clearOverride = (field: keyof ParsedBlocker) => {
    const newOverrides = { ...manualOverrides };
    delete newOverrides[field];
    setManualOverrides(newOverrides);
  };

  return (
    <div className="bg-gruvbox-bg-1 border border-gruvbox-bg-3 rounded-lg shadow-lg w-[480px] overflow-hidden">
      {/* Header */}
      <div className="px-4 py-3 border-b border-gruvbox-bg-2 flex items-center gap-2">
        <AlertCircle size={16} className="text-gruvbox-red-bright" />
        <span className="text-sm font-medium text-gruvbox-fg">Add Blocker</span>
        <span className="text-xs text-gruvbox-fg-4 ml-auto">
          Type <kbd className="kbd">/</kbd> for templates
        </span>
      </div>

      {/* Smart Input */}
      <div className="p-4">
        <input
          ref={inputRef}
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Describe the blocker... (e.g., 'waiting on material from Acme, arrives friday')"
          className="w-full bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded-lg px-4 py-3 text-sm text-gruvbox-fg placeholder:text-gruvbox-fg-4 focus:outline-none focus:border-gruvbox-red-bright"
        />

        {/* Template Dropdown */}
        {showTemplates && (
          <div className="mt-2 bg-gruvbox-bg-2 border border-gruvbox-bg-3 rounded-lg overflow-hidden">
            <div className="px-3 py-2 text-xs text-gruvbox-fg-4 border-b border-gruvbox-bg-3 flex items-center gap-2">
              <Zap size={12} />
              Quick Templates
            </div>
            <div className="max-h-48 overflow-auto">
              {filteredTemplates.map((template, idx) => (
                <button
                  key={template.id}
                  onClick={() => applyTemplate(template)}
                  className={cn(
                    'w-full px-3 py-2 text-left flex items-center gap-3 transition-colors',
                    idx === selectedTemplateIndex ? 'bg-gruvbox-bg-3' : 'hover:bg-gruvbox-bg-1'
                  )}
                >
                  <span className={cn('px-2 py-0.5 rounded text-xs', typeColors[template.type])}>
                    {template.type}
                  </span>
                  <span className="text-sm text-gruvbox-fg">{template.label}</span>
                  <span className="text-xs text-gruvbox-fg-4 ml-auto">/{template.shortcut}</span>
                </button>
              ))}
              {filteredTemplates.length === 0 && (
                <div className="px-3 py-4 text-sm text-gruvbox-fg-4 text-center">
                  No matching templates
                </div>
              )}
            </div>
          </div>
        )}

        {/* Live Preview */}
        {input && !showTemplates && (
          <div className="mt-4 space-y-2">
            <div className="text-xs text-gruvbox-fg-4 uppercase">Parsed Result</div>

            <div className="flex flex-wrap gap-2">
              {/* Type Chip */}
              <button
                onClick={() => {
                  // Cycle through types on click
                  const types: BlockerType[] = ['internal', 'external', 'material', 'approval', 'site'];
                  const currentIdx = types.indexOf(parsed.type);
                  const nextType = types[(currentIdx + 1) % types.length];
                  setManualOverrides({ ...manualOverrides, type: nextType });
                }}
                className={cn(
                  'flex items-center gap-1.5 px-2 py-1 rounded text-xs transition-colors',
                  typeColors[parsed.type],
                  'hover:opacity-80 cursor-pointer'
                )}
              >
                <Tag size={12} />
                {parsed.type}
              </button>

              {/* Owner Chip */}
              {parsed.owner && (
                <div className="flex items-center gap-1.5 px-2 py-1 rounded text-xs bg-gruvbox-bg-3 text-gruvbox-fg">
                  <User size={12} />
                  {parsed.owner.name}
                  <button
                    onClick={() => clearOverride('owner')}
                    className="ml-1 text-gruvbox-fg-4 hover:text-gruvbox-fg"
                  >
                    ×
                  </button>
                </div>
              )}

              {/* Date Chip */}
              {parsed.expectedResolution && (
                <div className="flex items-center gap-1.5 px-2 py-1 rounded text-xs bg-gruvbox-green-bright/20 text-gruvbox-green-bright">
                  <Calendar size={12} />
                  {formatDate(parsed.expectedResolution)}
                  {parsed.dateText && (
                    <span className="text-gruvbox-green-bright/60">({parsed.dateText})</span>
                  )}
                </div>
              )}
            </div>

            {/* Description Preview */}
            <div className="text-sm text-gruvbox-fg bg-gruvbox-bg-2 rounded px-3 py-2">
              {parsed.description}
            </div>
          </div>
        )}
      </div>

      {/* Footer */}
      <div className="px-4 py-3 border-t border-gruvbox-bg-2 flex items-center justify-between">
        <div className="text-xs text-gruvbox-fg-4">
          {showTemplates ? (
            <span><kbd className="kbd">↑↓</kbd> navigate <kbd className="kbd">Enter</kbd> select</span>
          ) : (
            <span>Magic dates: tomorrow, friday, next week, in 3 days</span>
          )}
        </div>
        <div className="flex items-center gap-3 text-xs text-gruvbox-fg-4">
          <span><kbd className="kbd">Cmd+Enter</kbd> save</span>
          <span><kbd className="kbd">Esc</kbd> cancel</span>
        </div>
      </div>
    </div>
  );
});

export default SmartBlockerForm;
```

### Success Criteria:

#### Automated Verification:
- [x] File exists: `millflow/src/components/node/SmartBlockerForm.tsx`
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Component renders without errors
- [ ] Basic input works

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Integration

### Overview
Replace the old BlockerForm with SmartBlockerForm in SheetView.

### Changes Required:

#### 1. Update SheetView Import
**File**: `millflow/src/views/SheetView.tsx`

Replace the import:
```typescript
// Change this:
import BlockerForm from '@/components/node/BlockerForm';

// To this:
import SmartBlockerForm from '@/components/node/SmartBlockerForm';
```

#### 2. Update SheetView Render
**File**: `millflow/src/views/SheetView.tsx`

Replace the BlockerForm render (around line 201-208):
```typescript
// Change this:
{activeActionForm === 'blocker' && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <BlockerForm
      onSave={handleBlockerSave}
      onCancel={handleActionFormCancel}
    />
  </div>
)}

// To this:
{activeActionForm === 'blocker' && (
  <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
    <SmartBlockerForm
      onSave={handleBlockerSave}
      onCancel={handleActionFormCancel}
    />
  </div>
)}
```

### Success Criteria:

#### Automated Verification:
- [x] TypeScript compiles: `cd millflow && npm run build`
- [x] No lint errors: `cd millflow && npm run lint`
- [ ] Dev server runs: `cd millflow && npm run dev`

#### Manual Verification:
- [ ] Press `b` in sheet view opens new SmartBlockerForm
- [ ] Typing text shows live preview with parsed type/owner/date
- [ ] Typing `/` opens template menu
- [ ] Selecting template autofills input
- [ ] `Cmd+Enter` saves blocker
- [ ] `Esc` closes form
- [ ] Saved blocker appears on node

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 5: Polish & Edge Cases

### Overview
Handle edge cases and polish the UX.

### Changes Required:

#### 1. Add formatDate to styles lib (if not exists)
**File**: `millflow/src/lib/styles.ts`

Ensure this function exists:
```typescript
export function formatDate(dateStr: string): string {
  const date = new Date(dateStr);
  return date.toLocaleDateString('en-US', {
    month: 'short',
    day: 'numeric',
    year: date.getFullYear() !== new Date().getFullYear() ? 'numeric' : undefined
  });
}
```

#### 2. Handle click-outside to close template menu
**File**: `millflow/src/components/node/SmartBlockerForm.tsx`

Add click handler to close template dropdown when clicking outside.

#### 3. Add keyboard hint animation
When input is empty, show subtle animation hinting at `/` templates.

### Success Criteria:

#### Automated Verification:
- [ ] TypeScript compiles: `cd millflow && npm run build`
- [ ] No lint errors: `cd millflow && npm run lint`

#### Manual Verification:
- [ ] Clicking outside template menu closes it
- [ ] Empty state shows helpful hints
- [ ] All keyboard shortcuts work
- [ ] Form is responsive and feels fast

---

## Testing Strategy

### Manual Testing Steps:
1. Navigate to sheet view, select a node
2. Press `b` to open blocker form
3. Type "waiting on material from Acme, arrives friday"
4. Verify preview shows: type=material, owner=Acme Hardware Co., date=next Friday
5. Press `Cmd+Enter` to save
6. Verify blocker appears on node
7. Open form again, type `/`
8. Verify template menu appears
9. Use arrow keys to navigate, Enter to select
10. Verify template fills input correctly

### Edge Cases to Test:
- Empty input (should not save)
- Input with no detectable type (defaults to internal)
- Input with no owner match
- Input with no date
- Very long description
- Special characters in input

## Performance Considerations

- Parsing is done with useMemo to avoid recalculating on every render
- Template filtering is debounced implicitly by React state updates
- No expensive operations - all string matching is simple

## References

- Current implementation: `millflow/src/components/node/BlockerForm.tsx`
- Research document: `thoughts/shared/research/2025-12-19-millflow-blocker-form.md`
- Similar patterns: `millflow/src/components/node/AssigneePicker.tsx` (fuzzy search)
- Similar patterns: `millflow/src/components/node/DueDatePicker.tsx` (quick dates)
