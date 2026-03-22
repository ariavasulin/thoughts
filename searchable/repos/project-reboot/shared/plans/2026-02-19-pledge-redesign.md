# Pledge Section Redesign — Implementation Plan

## Overview

Redesign the homepage pledge section with a new single-scroll multi-section pledge form, updated stats counters, and cleaner checkbox UX. All data stays mocked (no backend). The pledge now requires participants to select at least 1 Individual Action, 1 Social Action, and 1 Cultural Action.

## Current State Analysis

The pledge section lives in 4 files inside `components/home/`:

| File | What it does |
|---|---|
| `pledge-section.tsx` | Parent wrapper. Holds `BASE_PLEDGE_COUNT = 1247` in state. Renders Counter, DataViz, PledgeForm. |
| `pledge-counter.tsx` | Animated count-up number + tagline ("people committed to redefining..."). Uses IntersectionObserver. |
| `pledge-form.tsx` | Single flat form with 3 categories (Connection, Reflection, Intentional Use of Time), name/email fields, app multi-select, photo upload link. No validation beyond "at least 1 pledge + name". |
| `data-viz.tsx` | 6 hardcoded stat cards in a grid (apps deleted, in detox, downloaded Clear Space, hobbies, hours saved, hikes). |

All rendered inside `app/page.tsx:101` as `<PledgeSection />`.

### Key Discoveries
- No backend — pledge count is in-memory React state (`pledge-section.tsx:8-11`)
- `react-hook-form` and `zod` are installed but unused — current form uses plain `useState`
- The `PledgeCheckbox` component at `pledge-form.tsx:351-377` renders a native `<input type="checkbox">` with visible checkbox square
- The existing animated counter in `pledge-counter.tsx:9-37` uses an IntersectionObserver-based `useCountUp` hook — this is solid and should be reused
- shadcn/ui has ~40 components in `components/ui/` but the pledge section doesn't use any of them

## Desired End State

A single scrollable pledge form with these sections in order:

1. **Stats banner** — Big animated counter ("___ Students Redefining Our Generation's Relationship with Technology") + grid of 7 action-specific stat counters
2. **Basics section** — Name, Age, School, Email, newsletter opt-in checkbox
3. **Individual Actions** — 3 options (Clearspace challenge, delete apps with multi-select, phone away from bed). Must pick 1+.
4. **Social Actions** — 4 options (send move-off-SM message, call someone, digital detox with friend, phone-free walk/hike). Expandable message templates for the first option. Must pick 1+.
5. **Cultural Action** — Share pledge link with 2+ friends via copy/share button. Required.
6. **Submit button** — Disabled until validation passes (1 individual + 1 social + 1 cultural + required basics fields)
7. **Post-submit confirmation** — "You're in" card with copy-link / native share

### Verification
- `pnpm build` succeeds with no errors
- `pnpm lint` passes
- Visual inspection: form renders correctly on mobile and desktop
- Validation: submit button disabled until 1 action from each category + basics filled
- Counter animates on scroll into view
- Stats grid shows 7 stat cards with animated count-up
- Expandable message templates open/close correctly
- Copy-link button copies URL to clipboard
- Newsletter checkbox present and functional (no backend integration yet)

## What We're NOT Doing

- No backend / API routes / database — all data stays mocked
- No actual Substack newsletter integration for the checkbox (just UI)
- No actual message sending for "Move Off Social Media"
- No verification that users actually shared with 2+ friends
- No email validation beyond HTML `type="email"`
- Not touching any other pages (ethos, events, join)
- Not changing the hero or mission sections
- Not adding new dependencies

## Implementation Approach

Edit the 4 existing files in-place. No new files needed. The changes are:

1. **Phase 1**: Update `data-viz.tsx` with the 7 new stats from the spec
2. **Phase 2**: Rewrite `pledge-form.tsx` with the new multi-section form, clean toggle-style checkboxes, expandable templates, and share CTA
3. **Phase 3**: Update `pledge-counter.tsx` headline and `pledge-section.tsx` layout/ordering

---

## Phase 1: Update Stats Section

### Overview
Replace the 6 current stats with the 7 stats from the spec, each with an animated count-up.

### Changes Required

#### 1. `components/home/data-viz.tsx`

Replace the entire `stats` array and update the component to use animated count-up per stat (reusing the `useCountUp` pattern from `pledge-counter.tsx`, or a simplified inline version).

**New stats array:**

```typescript
const stats = [
  { icon: Smartphone, value: 843, label: "Apps Deleted", sub: "Instagram, TikTok, Snapchat & more", color: "text-[#088188]" },
  { icon: Zap, value: 312, label: "Clearspace Challenges Completed", color: "text-[#1F5D76]" },
  { icon: Timer, value: 187, label: "Digital Detoxes Completed", color: "text-[#088188]" },
  { icon: Moon, value: 529, label: "Phones Moved Away From Bed", color: "text-[#1F5D76]" },
  { icon: MessageCircle, value: 274, label: "Conversations Moved Off Social Media", color: "text-[#088188]" },
  { icon: Phone, value: 391, label: "Calls Made to a Loved One", color: "text-[#1F5D76]" },
  { icon: Footprints, value: 156, label: "Phone-Free Walks & Hikes Planned", color: "text-[#088188]" },
]
```

- Change from `"use client"` (implicit) — add `"use client"` directive and `useState`/`useEffect`/`useRef` for animated count-up
- Each stat card value should animate from 0 to its target when scrolled into view (use IntersectionObserver like the pledge counter does)
- Keep the existing grid layout (2 cols mobile, 3 cols desktop) but add a 4th column option for 7 items: `grid-cols-2 md:grid-cols-3 lg:grid-cols-4` with the 7th item spanning or centered

### Success Criteria

#### Automated Verification:
- [x] `pnpm build` succeeds
- [x] `pnpm lint` passes (no eslint config in project — build is the gate)

#### Manual Verification:
- [ ] 7 stat cards visible in the pledge section
- [ ] Each stat animates from 0 to target on scroll
- [ ] Grid looks good on mobile (2 cols) and desktop (3-4 cols)
- [ ] Icons and colors match the design palette

---

## Phase 2: Rebuild Pledge Form

### Overview
The largest change. Replace the flat checkbox form with a multi-section scrollable form using clean toggle-style selections (no visible checkbox squares — just background/border color change on selection).

### Changes Required

#### 1. `components/home/pledge-form.tsx` — Full rewrite

**New form structure:**

```
Basics Section
  - Name (required)
  - Age (number input)
  - School (text input)
  - Email (required, type="email")
  - Newsletter opt-in toggle

Individual Actions Section (must select 1+)
  - Start the Project Reboot Challenge (hosted by Clearspace)
  - Delete an app → reveals app multi-select (Instagram / TikTok / Snapchat / X / Other)
  - Move your phone out of arm's reach while sleeping

Social Actions Section (must select 1+)
  - Send the "Move Off Social Media" message → expandable templates
  - Call someone you appreciate and tell them why they matter
  - Do a digital detox with a friend
  - Plan a phone-free walk or hike with someone

Cultural Action Section (required)
  - Share the pledge link with 2+ friends
  - Copy link button + native Web Share API fallback

Submit Button
  - Disabled state when validation fails
  - Shows which categories still need selections
```

**Key design decisions:**

**Clean toggle checkboxes (no checkbox square):**
Replace the current `PledgeCheckbox` component. Instead of `<input type="checkbox">` with a visible square, use a `<button>` or clickable `<div>` that changes background/border color when selected:

```typescript
function PledgeOption({
  selected,
  onToggle,
  children,
}: {
  selected: boolean
  onToggle: () => void
  children: React.ReactNode
}) {
  return (
    <button
      type="button"
      onClick={onToggle}
      className={`w-full rounded-xl border p-4 text-left text-sm font-medium transition-all ${
        selected
          ? "border-[#088188] bg-[#088188]/15 text-foreground"
          : "border-border bg-transparent text-muted-foreground hover:bg-secondary/20"
      }`}
    >
      {children}
    </button>
  )
}
```

No hidden `<input>` needed — this is client-only with no form submission to a server.

**Expandable message templates:**
For the "Send the Move Off Social Media message" social action, add a collapsible section below it when selected:

```typescript
{selectedSocial.includes("move_off_sm") && (
  <div className="ml-4 mt-2 flex flex-col gap-2">
    <ExpandableTemplate title="Group Chat Move-Off Script">
      <p>Hey everyone! I've decided to take a break from [app]...</p>
      <CopyButton text="..." />
    </ExpandableTemplate>
    <ExpandableTemplate title="1:1 Move to iMessage Script">
      <p>Hey! I'm moving off [app] for a while...</p>
      <CopyButton text="..." />
    </ExpandableTemplate>
  </div>
)}
```

**App multi-select for "Delete an app":**
Keep the existing pill-style multi-select pattern from the current form — it works well. Reduce the app list to match spec: Instagram, TikTok, Snapchat, X, Other (with custom input).

**Cultural action — share link:**
```typescript
<div className="flex gap-3">
  <button onClick={copyPledgeLink} className="...">
    {copied ? "Copied!" : "Copy Pledge Link"}
  </button>
  {typeof navigator !== "undefined" && "share" in navigator && (
    <button onClick={() => navigator.share({ url: pledgeUrl, title: "..." })} className="...">
      Share
    </button>
  )}
</div>
```

**Validation logic:**
```typescript
const hasIndividual = individualActions.length > 0
const hasSocial = socialActions.length > 0
const hasCultural = culturalDone // boolean — user clicked copy/share
const basicsComplete = name.trim() !== "" && email.trim() !== ""
const canSubmit = hasIndividual && hasSocial && hasCultural && basicsComplete
```

**Section headers:** Each action category gets a labeled header with:
- Category name (e.g., "Individual Actions")
- Brief description
- A subtle validation indicator (e.g., a small checkmark icon when at least 1 is selected, or a "pick at least one" hint if empty on attempted submit)

**State management:** Keep using plain `useState`. The form has ~8 pieces of state:
- `name`, `age`, `school`, `email` (strings)
- `newsletterOptIn` (boolean)
- `individualActions` (string[])
- `socialActions` (string[])
- `culturalDone` (boolean — set true when user copies/shares the link)
- `selectedApps` (string[]) — for the "delete an app" sub-selection
- `submitted` (boolean)

**Post-submit confirmation:**
Update the existing confirmation card. Keep the "You're in." headline and share buttons. Add a summary of what the user pledged to do.

### Success Criteria

#### Automated Verification:
- [x] `pnpm build` succeeds
- [x] `pnpm lint` passes (no eslint config in project — build is the gate)

#### Manual Verification:
- [ ] Basics section renders with all 5 fields (name, age, school, email, newsletter checkbox)
- [ ] Individual Actions section shows 3 options with clean toggle style (no checkbox square)
- [ ] Selecting "Delete an app" reveals app multi-select (Instagram, TikTok, Snapchat, X, Other)
- [ ] Social Actions section shows 4 options
- [ ] Selecting "Send the Move Off Social Media message" reveals expandable templates
- [ ] Templates expand/collapse and have copy buttons
- [ ] Cultural Action section shows share/copy link
- [ ] Submit button disabled until 1 individual + 1 social + cultural + name + email filled
- [ ] Validation hints visible when categories are incomplete
- [ ] Post-submit shows confirmation with pledge summary
- [ ] All toggles feel snappy with smooth color transitions
- [ ] Form looks good on mobile and desktop

**Implementation Note**: After completing this phase, pause for manual verification on mobile and desktop before proceeding.

---

## Phase 3: Update Counter & Section Layout

### Overview
Update the headline text to Option A, and adjust the parent `pledge-section.tsx` layout to match the new ordering: Counter at top, Stats below it, then the Form.

### Changes Required

#### 1. `components/home/pledge-counter.tsx`

Change the tagline text from:
```
"people committed to redefining our generation's relationship with technology"
```
to:
```
"Students Redefining Our Generation's Relationship with Technology"
```

Capitalize to match the spec's title-case style.

#### 2. `components/home/pledge-section.tsx`

Update the section intro copy and ensure ordering is:
1. Section intro (heading + paragraph)
2. PledgeCounter
3. DataViz (stats grid)
4. PledgeForm

The current ordering is already this, so this is mainly about updating copy. Change:
- Heading: keep "Join the Movement" or update if desired
- Paragraph: refresh to match the new pledge framing

Pass the `onSubmit` callback to PledgeForm so it increments the counter.

### Success Criteria

#### Automated Verification:
- [x] `pnpm build` succeeds
- [x] `pnpm lint` passes (no eslint config in project — build is the gate)

#### Manual Verification:
- [ ] Counter headline reads "Students Redefining Our Generation's Relationship with Technology"
- [ ] Counter still animates on scroll
- [ ] Section flows well: intro → counter → stats → form
- [ ] Submitting the form increments the counter

---

## Testing Strategy

### Automated:
- `pnpm build` — ensures no TypeScript/build errors
- `pnpm lint` — ensures no linting issues

### Manual Testing Steps:
1. Load homepage, scroll to pledge section
2. Verify counter animates from 0 to 1247
3. Verify all 7 stat cards animate
4. Fill out basics (name, age, school, email)
5. Toggle newsletter checkbox
6. Select 1+ individual actions — verify clean toggle style (no checkbox square)
7. Select "Delete an app" — verify app pills appear
8. Select 1+ social actions
9. Select "Send Move Off SM message" — verify templates expand
10. Copy a template — verify clipboard works
11. Click "Copy Pledge Link" in cultural section
12. Verify submit button becomes enabled
13. Submit — verify confirmation card appears with pledge summary
14. Verify counter incremented to 1248
15. Test on mobile viewport — verify responsive layout
16. Test share button on mobile (native share sheet should appear)

## Performance Considerations

- No new dependencies added
- IntersectionObserver-based animations are already lightweight
- The form is client-side only — no network requests
- All 7 stat animations should use a single shared IntersectionObserver rather than 7 individual ones

## References

- Current implementation: `components/home/pledge-section.tsx`, `pledge-form.tsx`, `pledge-counter.tsx`, `data-viz.tsx`
- Design tokens: `app/globals.css:40-74` (color palette)
- Existing animation pattern: `pledge-counter.tsx:9-37` (useCountUp hook)
