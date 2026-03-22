# Supabase Persistence for Pledge Form — Implementation Plan

**Status**: Implemented (Phases 2-6)
**Date**: 2026-02-19
**Depends on**: pledge redesign (committed as `a43a99b`)

---

## Goal

Replace the mocked client-side pledge data with real Supabase persistence. One table, one API route to insert, one server-side fetch for aggregated stats.

---

## Current State

- `pledge-section.tsx` — `"use client"`, holds `pledgeCount` in `useState(1247)` (hardcoded)
- `pledge-form.tsx` — `"use client"`, collects all form fields, calls `onSubmit?.()` which just increments the local counter
- `data-viz.tsx` — `"use client"`, renders 6 stats from a hardcoded `stats` array with animated count-up
- `pledge-counter.tsx` — `"use client"`, receives `count` prop, animated count-up display
- `app/page.tsx` — async server component, renders `<PledgeSection />` (no props passed)
- No API routes exist (`app/api/` doesn't exist)
- No Supabase dependency installed

---

## Architecture

```
Browser (pledge-form.tsx)
  │
  │ POST /api/pledges  { name, email, age?, school?, ... }
  ▼
app/api/pledges/route.ts
  │  uses service role key (bypasses RLS)
  │  validates + inserts into `pledges` table
  ▼
Supabase Postgres — `pledges` table

app/page.tsx (server component)
  │  fetches aggregate stats at render time (ISR, revalidate: 300)
  │  passes stats as props to PledgeSection
  ▼
PledgeSection → PledgeCounter (total count)
             → DataViz (per-action counts)
```

### Why this architecture:
- **Service role key for writes**: No RLS policy needed for inserts, no auth complexity
- **Server-side reads in page.tsx**: Stats fetched at build/ISR time, not on every client render — fast, cacheable, no key exposed to browser
- **No `@supabase/ssr` needed**: We have no auth. Direct `@supabase/supabase-js` with service role key server-side is sufficient
- **Single dependency**: `@supabase/supabase-js` only

---

## Phase 1: Supabase Setup (manual, outside code)

### 1a. Create Supabase project
- Go to [supabase.com/dashboard](https://supabase.com/dashboard) → New Project
- Note the **Project URL** and **Service Role Key** from Settings → API

### 1b. Create the `pledges` table

Run this SQL in the Supabase SQL Editor:

```sql
CREATE TABLE pledges (
  id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  created_at    TIMESTAMPTZ DEFAULT now() NOT NULL,

  -- Basics
  name          TEXT NOT NULL,
  email         TEXT NOT NULL,
  age           INT,
  school        TEXT,
  newsletter    BOOLEAN DEFAULT false,

  -- Individual actions (stored as text arrays)
  individual_actions  TEXT[] DEFAULT '{}',
  deleted_apps        TEXT[] DEFAULT '{}',

  -- Social actions
  social_actions      TEXT[] DEFAULT '{}',

  -- Cultural action
  shared_link         BOOLEAN DEFAULT false
);

-- Enable RLS (good practice even if we only use service role for writes)
ALTER TABLE pledges ENABLE ROW LEVEL SECURITY;

-- Allow public SELECT (for anon key reads, optional — we'll use service role for reads too)
-- CREATE POLICY "public_read" ON pledges FOR SELECT TO public USING (true);

-- Index for fast aggregation
CREATE INDEX idx_pledges_created_at ON pledges (created_at DESC);
```

**Column mapping from the form:**

| Form field | Column | Type | Notes |
|---|---|---|---|
| name | `name` | TEXT | required |
| email | `email` | TEXT | required |
| age | `age` | INT | optional |
| school | `school` | TEXT | optional |
| newsletter opt-in | `newsletter` | BOOLEAN | default false |
| individualActions[] | `individual_actions` | TEXT[] | e.g. `{"clearspace","delete_app","phone_away"}` |
| selectedApps[] | `deleted_apps` | TEXT[] | e.g. `{"Instagram","TikTok"}` |
| socialActions[] | `social_actions` | TEXT[] | e.g. `{"move_off_sm","call_someone"}` |
| culturalDone | `shared_link` | BOOLEAN | always true at submission |

### 1c. Environment variables

Add to `.env.local` (and Vercel dashboard):

```bash
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...
```

Note: We do NOT need `NEXT_PUBLIC_SUPABASE_ANON_KEY` since all Supabase access is server-side only.

---

## Phase 2: Supabase Client Library

### 2a. Install dependency

```bash
pnpm add @supabase/supabase-js
```

### 2b. Create server-only Supabase client

**New file: `lib/supabase.ts`**

```typescript
import { createClient } from "@supabase/supabase-js"

// Server-only Supabase client using service role key.
// NEVER import this file from client components.
export function createServiceClient() {
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY

  if (!url || !key) {
    throw new Error("Missing Supabase environment variables")
  }

  return createClient(url, key, {
    auth: {
      persistSession: false,
      autoRefreshToken: false,
      detectSessionInUrl: false,
    },
  })
}
```

One file, one export. No browser client needed since we never talk to Supabase from the client — the form POSTs to our own API route instead.

---

## Phase 3: API Route for Pledge Submission

**New file: `app/api/pledges/route.ts`**

```typescript
import { NextResponse } from "next/server"
import { createServiceClient } from "@/lib/supabase"

export async function POST(request: Request) {
  try {
    const body = await request.json()

    // Validate required fields
    const name = typeof body.name === "string" ? body.name.trim() : ""
    const email = typeof body.email === "string" ? body.email.trim() : ""

    if (!name || !email) {
      return NextResponse.json(
        { error: "Name and email are required" },
        { status: 400 }
      )
    }

    const supabase = createServiceClient()

    const { error } = await supabase.from("pledges").insert({
      name,
      email,
      age: typeof body.age === "number" ? body.age : null,
      school: typeof body.school === "string" ? body.school.trim() || null : null,
      newsletter: Boolean(body.newsletter),
      individual_actions: Array.isArray(body.individualActions) ? body.individualActions : [],
      deleted_apps: Array.isArray(body.deletedApps) ? body.deletedApps : [],
      social_actions: Array.isArray(body.socialActions) ? body.socialActions : [],
      shared_link: Boolean(body.sharedLink),
    })

    if (error) {
      console.error("Supabase insert error:", error)
      return NextResponse.json(
        { error: "Failed to save pledge" },
        { status: 500 }
      )
    }

    return NextResponse.json({ ok: true }, { status: 201 })
  } catch {
    return NextResponse.json(
      { error: "Invalid request" },
      { status: 400 }
    )
  }
}
```

**Key decisions:**
- Validate + sanitize server-side (don't trust client data)
- Return minimal response (`{ ok: true }`) — the form doesn't need the inserted row back
- No rate limiting in v1 (can add later if needed)

---

## Phase 4: Server-Side Stats Fetching

### 4a. Stats aggregation query

**New file: `lib/pledge-stats.ts`**

```typescript
import { createServiceClient } from "@/lib/supabase"

export interface PledgeStats {
  totalPledges: number
  appsDeleted: number
  clearspaceChallenges: number
  digitalDetoxes: number
  phonesFromBed: number
  movedOffSocialMedia: number
  callsMade: number
}

export async function fetchPledgeStats(): Promise<PledgeStats> {
  const supabase = createServiceClient()

  // Single query: get all pledges, aggregate in JS
  // For a small-to-medium table this is fine; switch to a Postgres function if it grows large
  const { data, error } = await supabase
    .from("pledges")
    .select("individual_actions, deleted_apps, social_actions")

  if (error || !data) {
    console.error("Failed to fetch pledge stats:", error)
    return {
      totalPledges: 0,
      appsDeleted: 0,
      clearspaceChallenges: 0,
      digitalDetoxes: 0,
      phonesFromBed: 0,
      movedOffSocialMedia: 0,
      callsMade: 0,
    }
  }

  let appsDeleted = 0
  let clearspaceChallenges = 0
  let phonesFromBed = 0
  let digitalDetoxes = 0
  let movedOffSocialMedia = 0
  let callsMade = 0

  for (const row of data) {
    const ind: string[] = row.individual_actions ?? []
    const soc: string[] = row.social_actions ?? []
    const apps: string[] = row.deleted_apps ?? []

    if (ind.includes("clearspace")) clearspaceChallenges++
    if (ind.includes("delete_app")) appsDeleted += apps.length || 1
    if (ind.includes("phone_away")) phonesFromBed++
    if (soc.includes("move_off_sm")) movedOffSocialMedia++
    if (soc.includes("call_someone")) callsMade++
    if (soc.includes("detox_friend")) digitalDetoxes++
  }

  return {
    totalPledges: data.length,
    appsDeleted,
    clearspaceChallenges,
    digitalDetoxes,
    phonesFromBed,
    movedOffSocialMedia,
    callsMade,
  }
}
```

**Why aggregate in JS instead of SQL:**
- Postgres array containment queries (`@>`) work but are harder to type-check
- For < 10k rows, pulling 3 columns and looping is fast enough
- Easier to understand and maintain
- Can migrate to a Postgres function or materialized view later

### 4b. Wire stats into `app/page.tsx`

Modify the server component to fetch stats and pass them down:

```typescript
// app/page.tsx
import { fetchPledgeStats } from "@/lib/pledge-stats"

export default async function Home() {
  const articles = await fetchFeed()
  const stats = await fetchPledgeStats()

  return (
    <>
      <HeroSection />
      <MissionSection />
      <PledgeSection initialStats={stats} />
      <StayConnectedSection articles={articles} />
    </>
  )
}
```

**Caching**: Next.js will cache this fetch via ISR. Add `revalidate` to the fetch or use `export const revalidate = 300` on the page to refresh stats every 5 minutes.

---

## Phase 5: Update Components

### 5a. `pledge-section.tsx` — Accept stats props, pass to children

```diff
+ import { PledgeStats } from "@/lib/pledge-stats"

- const BASE_PLEDGE_COUNT = 1247

+ interface PledgeSectionProps {
+   initialStats: PledgeStats
+ }

- export function PledgeSection() {
-   const [pledgeCount, setPledgeCount] = useState(BASE_PLEDGE_COUNT)
+ export function PledgeSection({ initialStats }: PledgeSectionProps) {
+   const [pledgeCount, setPledgeCount] = useState(initialStats.totalPledges)

    // ... rest stays the same, but pass stats to DataViz:
-   <DataViz />
+   <DataViz stats={initialStats} />
```

### 5b. `data-viz.tsx` — Accept real stats instead of hardcoded array

Change from hardcoded `stats` array to accepting a `PledgeStats` prop and mapping it to the display format. The animated count-up logic stays the same — just swap the source values.

### 5c. `pledge-form.tsx` — POST to API route on submit

Replace the `handleSubmit` function:

```diff
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (!canSubmit) return
+
+   try {
+     const res = await fetch("/api/pledges", {
+       method: "POST",
+       headers: { "Content-Type": "application/json" },
+       body: JSON.stringify({
+         name: name.trim(),
+         email: email.trim(),
+         age: age ? parseInt(age, 10) : null,
+         school: school.trim() || null,
+         newsletter: newsletterOptIn,
+         individualActions,
+         deletedApps: selectedApps,
+         socialActions,
+         sharedLink: culturalDone,
+       }),
+     })
+
+     if (!res.ok) throw new Error("Failed to submit")
+   } catch (err) {
+     console.error("Pledge submission failed:", err)
+     // Still show success UI — we don't want to block the user experience
+     // The pledge can be retried or we accept best-effort
+   }
+
    setSubmitted(true)
    onSubmit?.()
  }
```

**Key decision**: Fire-and-forget with graceful degradation. If the API call fails, we still show the success screen. The pledge counter will be slightly off until the next ISR revalidation, but the user experience isn't blocked.

### 5d. `pledge-counter.tsx` — No changes needed

It already accepts a `count` prop. It will now receive the real count from Supabase instead of a hardcoded value.

---

## Phase 6: ISR Revalidation

Add to `app/page.tsx`:

```typescript
export const revalidate = 300 // re-fetch stats every 5 minutes
```

This means:
- First visitor after deploy sees stats from build time
- Stats refresh every 5 minutes on subsequent visits
- No stale data beyond 5 minutes

Optionally, add on-demand revalidation from the API route after a successful insert:

```typescript
import { revalidatePath } from "next/cache"

// After successful insert:
revalidatePath("/")
```

This gives instant stat updates after each pledge submission.

---

## New Files Summary

| File | Purpose |
|---|---|
| `lib/supabase.ts` | Server-only Supabase client (service role) |
| `lib/pledge-stats.ts` | `fetchPledgeStats()` — aggregation query |
| `app/api/pledges/route.ts` | POST endpoint for pledge inserts |
| `.env.local` | Supabase URL + service role key |

## Modified Files Summary

| File | Change |
|---|---|
| `app/page.tsx` | Import + call `fetchPledgeStats()`, pass to `PledgeSection` |
| `components/home/pledge-section.tsx` | Accept `initialStats` prop, pass to children |
| `components/home/data-viz.tsx` | Accept `PledgeStats` prop instead of hardcoded array |
| `components/home/pledge-form.tsx` | POST to `/api/pledges` in `handleSubmit` |
| `package.json` | Add `@supabase/supabase-js` |

## Files NOT Changed

| File | Why |
|---|---|
| `pledge-counter.tsx` | Already accepts `count` prop, works as-is |

---

## Verification Checklist

After implementation:

- [x] `.env.local` has `NEXT_PUBLIC_SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`
- [x] `pnpm build` succeeds with no type errors
- [x] `pnpm dev` → fill out pledge form → check Supabase dashboard for new row
- [x] Refresh page → pledge counter reflects new total
- [x] Stats grid shows real aggregated values
- [x] Form still works if Supabase is unreachable (graceful degradation)
- [ ] Service role key is NOT exposed in client bundle (check browser network tab)
- [ ] Vercel environment variables configured before deploy

---

## Decisions (Resolved)

1. **Duplicate emails**: Allowed — no unique constraint on `email`. Users can pledge multiple times.
2. **Rate limiting**: None for v1.
3. **Seed data**: Start from 0. No seeding.
