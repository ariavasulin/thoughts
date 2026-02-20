# DataViz Animation Fix + DB Performance — Implementation Plan

## Overview

Fix the count-up animation in `data-viz.tsx` so stats animate only on first scroll-in (not on every cycle), restore the cycling "Apps Deleted" card with per-app breakdowns from real Supabase data, and move the stats aggregation from JS-side row iteration to a single SQL aggregation query for performance.

## Current State Analysis

### Animation (`data-viz.tsx`)
- `useCountUp` hook (line 29) has `target` in its dependency array → every time `target` changes the animation replays from 0
- `AppDeletedCard` (line 120) is currently simplified to a single "Total Apps Deleted" static card — no cycling, no per-app breakdown
- Brand SVG icons (`InstagramIcon`, `TikTokIcon`, `SnapchatIcon`, `XIcon`) exist at lines 155-187 but are unused dead code
- `StatCard` uses `useCountUp` correctly for static stats (target never changes after mount)

### Data fetching (`lib/pledge-stats.ts`)
- Fetches **all rows** with `supabase.from("pledges").select("individual_actions, deleted_apps, social_actions")` then loops in JS
- O(n) transfer + O(n) JS iteration — scales linearly with pledge count
- No per-app breakdown returned — only total `appsDeleted` count
- `deleted_apps` column is `text[]` with values from `["Instagram", "TikTok", "Snapchat", "X"]`

### Caching (`app/page.tsx`)
- ISR with `revalidate = 300` (5 min) — good, stats are cached at the edge
- Pledge API route calls `revalidatePath("/")` on submit — instant cache bust after new pledge

## Desired End State

1. **Animation**: Stats count up from 0 → target on first scroll into view. Subsequent interactions (cycling the Apps Deleted card) show values instantly — no re-animation.

2. **Cycling card**: "Apps Deleted" card cycles through: Total → Instagram → TikTok → Snapchat → X → (loop). First view animates, cycle clicks show values instantly.

3. **DB query**: Single SQL aggregation query replaces fetching all rows. Returns all stats + per-app breakdown in one round trip. O(1) data transfer regardless of pledge count.

4. **Verification**: `pnpm build` passes. Visual inspection confirms animation behavior.

## What We're NOT Doing

- Not adding client-side data fetching or SWR — stats stay server-rendered via ISR
- Not changing the ISR revalidation interval (300s is fine)
- Not adding RLS policies or changing auth strategy
- Not changing the visual design of the cards
- Not touching any other components

## Implementation Approach

Two phases: (1) DB optimization + data shape change, (2) animation fix + cycling card restoration. Phase 1 is backend-only, Phase 2 is frontend-only.

---

## Phase 1: SQL Aggregation + Per-App Breakdown

### Overview
Replace the JS-side row loop with a single SQL aggregation query that returns all stats including per-app counts. This is a pure backend change — the frontend contract changes (new fields on `PledgeStats`) but the component updates happen in Phase 2.

### Changes Required

#### 1. `lib/pledge-stats.ts` — Replace JS aggregation with SQL

**Current**: Fetches all rows, loops in JS.
**New**: Single SQL query using Postgres array functions.

```typescript
import { createServiceClient } from "@/lib/supabase"

export interface AppBreakdown {
  instagram: number
  tiktok: number
  snapchat: number
  x: number
}

export interface PledgeStats {
  totalPledges: number
  appsDeleted: number
  appBreakdown: AppBreakdown
  clearspaceChallenges: number
  digitalDetoxes: number
  phonesFromBed: number
  movedOffSocialMedia: number
  callsMade: number
}

const ZERO_STATS: PledgeStats = {
  totalPledges: 0,
  appsDeleted: 0,
  appBreakdown: { instagram: 0, tiktok: 0, snapchat: 0, x: 0 },
  clearspaceChallenges: 0,
  digitalDetoxes: 0,
  phonesFromBed: 0,
  movedOffSocialMedia: 0,
  callsMade: 0,
}

export async function fetchPledgeStats(): Promise<PledgeStats> {
  const supabase = createServiceClient()

  const { data, error } = await supabase.rpc("get_pledge_stats")

  if (error || !data) {
    console.error("Failed to fetch pledge stats:", error)
    return ZERO_STATS
  }

  return {
    totalPledges: data.total_pledges ?? 0,
    appsDeleted: data.apps_deleted ?? 0,
    appBreakdown: {
      instagram: data.instagram_deleted ?? 0,
      tiktok: data.tiktok_deleted ?? 0,
      snapchat: data.snapchat_deleted ?? 0,
      x: data.x_deleted ?? 0,
    },
    clearspaceChallenges: data.clearspace_challenges ?? 0,
    digitalDetoxes: data.digital_detoxes ?? 0,
    phonesFromBed: data.phones_from_bed ?? 0,
    movedOffSocialMedia: data.moved_off_social_media ?? 0,
    callsMade: data.calls_made ?? 0,
  }
}
```

#### 2. Supabase Migration — Create `get_pledge_stats` RPC function

Apply via Supabase MCP `apply_migration`:

```sql
CREATE OR REPLACE FUNCTION get_pledge_stats()
RETURNS JSON
LANGUAGE sql
STABLE
AS $$
  SELECT json_build_object(
    'total_pledges', count(*),
    'apps_deleted', coalesce(sum(array_length(deleted_apps, 1)) FILTER (WHERE 'delete_app' = ANY(individual_actions)), 0),
    'instagram_deleted', coalesce(count(*) FILTER (WHERE 'Instagram' = ANY(deleted_apps)), 0),
    'tiktok_deleted', coalesce(count(*) FILTER (WHERE 'TikTok' = ANY(deleted_apps)), 0),
    'snapchat_deleted', coalesce(count(*) FILTER (WHERE 'Snapchat' = ANY(deleted_apps)), 0),
    'x_deleted', coalesce(count(*) FILTER (WHERE 'X' = ANY(deleted_apps)), 0),
    'clearspace_challenges', coalesce(count(*) FILTER (WHERE 'clearspace' = ANY(individual_actions)), 0),
    'digital_detoxes', coalesce(count(*) FILTER (WHERE 'detox_friend' = ANY(social_actions)), 0),
    'phones_from_bed', coalesce(count(*) FILTER (WHERE 'phone_away' = ANY(individual_actions)), 0),
    'moved_off_social_media', coalesce(count(*) FILTER (WHERE 'move_off_sm' = ANY(social_actions)), 0),
    'calls_made', coalesce(count(*) FILTER (WHERE 'call_someone' = ANY(social_actions)), 0)
  )
  FROM pledges
$$;
```

**Why a Postgres function instead of inline SQL via supabase-js:**
- Single round trip, single table scan
- `STABLE` marking allows Postgres query planner to cache within a transaction
- Clean RPC interface: `supabase.rpc("get_pledge_stats")` — no raw SQL in app code
- Easy to extend later (add new stats without changing client code)

**Why `FILTER (WHERE ...)` instead of `SUM(CASE ...)`:**
- Postgres-idiomatic, more readable
- Same performance, slightly cleaner

#### 3. Optional: GIN index on array columns

If the table grows past ~10k rows, add indexes:

```sql
CREATE INDEX IF NOT EXISTS idx_pledges_individual_actions ON pledges USING GIN (individual_actions);
CREATE INDEX IF NOT EXISTS idx_pledges_social_actions ON pledges USING GIN (social_actions);
CREATE INDEX IF NOT EXISTS idx_pledges_deleted_apps ON pledges USING GIN (deleted_apps);
```

For now, skip this — the table is small and a sequential scan on the whole table is fast. The ISR cache means the query runs at most once every 5 minutes anyway. Add indexes later if needed.

### Success Criteria

#### Automated Verification:
- [x] Migration applies cleanly via Supabase MCP
- [x] `supabase.rpc("get_pledge_stats")` returns correct JSON shape when tested via MCP `execute_sql`
- [x] `pnpm build` succeeds
- [x] Insert test rows, verify aggregation matches expected values, then clean up

#### Manual Verification:
- [ ] Homepage still renders with correct stats after switching to RPC

---

## Phase 2: Animation Fix + Cycling Card

### Overview
Fix `useCountUp` to animate only once, then restore the cycling "Apps Deleted" card using real per-app breakdown data from Phase 1.

### Changes Required

#### 1. Fix `useCountUp` hook — animate only on first trigger

**Current behavior**: Re-animates from 0 whenever `target` changes.
**New behavior**: Animates from 0 → target on first trigger. After that, returns `target` directly (no animation). This means cycling the Apps Deleted card shows values instantly.

```typescript
function useCountUp(target: number, trigger: boolean, duration = 1500) {
  const [current, setCurrent] = useState(0)
  const rafId = useRef<number>(0)
  const hasAnimated = useRef(false)

  useEffect(() => {
    if (!trigger || hasAnimated.current) {
      if (hasAnimated.current) setCurrent(target)
      return
    }

    hasAnimated.current = true
    const start = performance.now()

    const animate = (now: number) => {
      const elapsed = now - start
      const progress = Math.min(elapsed / duration, 1)
      const eased = 1 - Math.pow(1 - progress, 3)
      setCurrent(Math.floor(eased * target))
      if (progress < 1) {
        rafId.current = requestAnimationFrame(animate)
      }
    }

    rafId.current = requestAnimationFrame(animate)
    return () => cancelAnimationFrame(rafId.current)
  }, [target, trigger, duration])

  return current
}
```

Key change: `hasAnimated` ref tracks whether the initial animation has played. After that, `setCurrent(target)` is called directly — no `requestAnimationFrame` loop, no visual delay.

#### 2. Restore cycling `AppDeletedCard`

Replace the static `AppDeletedCard` with a cycling version that uses `appBreakdown` from `PledgeStats`:

```typescript
function AppDeletedCard({ visible, stats }: { visible: boolean; stats: PledgeStats }) {
  const breakdown = useMemo(() => [
    { label: "Apps Deleted", value: stats.appsDeleted, icon: Smartphone },
    { label: "Instagram Deleted", value: stats.appBreakdown.instagram, icon: InstagramIcon },
    { label: "TikTok Deleted", value: stats.appBreakdown.tiktok, icon: TikTokIcon },
    { label: "Snapchat Deleted", value: stats.appBreakdown.snapchat, icon: SnapchatIcon },
    { label: "X Deleted", value: stats.appBreakdown.x, icon: XIcon },
  ], [stats.appsDeleted, stats.appBreakdown])

  const [activeIndex, setActiveIndex] = useState(0)
  const active = breakdown[activeIndex]

  const count = useCountUp(active.value, visible)

  const cycle = useCallback(() => {
    setActiveIndex((prev) => (prev + 1) % breakdown.length)
  }, [breakdown.length])

  return (
    <button
      type="button"
      onClick={cycle}
      className="flex flex-col items-center gap-3 rounded-xl border border-[#1a5560] bg-[#0d4550] p-6 text-center transition-all hover:shadow-md hover:shadow-[#088188]/5 hover:border-[#088188]/40 cursor-pointer"
    >
      <active.icon className="h-6 w-6 text-[#088188]" />
      <span className="text-3xl font-extrabold text-[#088188]">
        {formatNumber(count)}
      </span>
      <span className="text-sm text-[#7cb8bd]">{active.label}</span>
      <span className="text-xs text-[#7cb8bd]/40">tap to cycle</span>
    </button>
  )
}
```

**How the animation + cycling interaction works:**
1. Card scrolls into view → `visible` becomes `true` → `useCountUp` animates 0 → total (1500ms ease-out)
2. User clicks → `activeIndex` changes → `active.value` changes → `target` changes in `useCountUp`
3. But `hasAnimated.current` is `true`, so `useCountUp` skips the animation and does `setCurrent(target)` directly
4. Value appears instantly. No janky re-animation.

#### 3. Update `DataViz` to pass full stats to `AppDeletedCard`

```diff
- <AppDeletedCard visible={visible} totalAppsDeleted={pledgeStats.appsDeleted} />
+ <AppDeletedCard visible={visible} stats={pledgeStats} />
```

#### 4. Re-add `useCallback` import

The cycling card needs `useCallback` which was previously removed. Add it back to the import.

#### 5. Clean up: remove unused `import type { SVGProps }` if it was removed

The brand icons use `SVGProps<SVGSVGElement>` — ensure it's imported from React.

### Success Criteria

#### Automated Verification:
- [x] `pnpm build` succeeds

#### Manual Verification:
- [ ] Stats animate from 0 → target on first scroll into view
- [ ] Clicking "Apps Deleted" card cycles: Total → Instagram → TikTok → Snapchat → X → Total
- [ ] Cycling shows new value **instantly** (no count-up animation on cycle)
- [ ] Initial animation is smooth (cubic ease-out, ~1.5s)
- [ ] No layout shift when cycling (card stays same size)
- [ ] "tap to cycle" hint visible below the label

---

## Performance Summary

| Concern | Before | After |
|---|---|---|
| DB query | `SELECT *` → transfer all rows → JS loop | Single `get_pledge_stats()` RPC → 1 JSON object |
| Data transfer | O(n) rows | O(1) fixed-size JSON |
| JS processing | O(n) loop on every page render | Zero — Postgres does all aggregation |
| Animation | Re-animates from 0 on every target change | Animates once, then instant updates |
| Caching | ISR 300s ✓ | ISR 300s ✓ (unchanged) |
| RAF loops | 6 independent loops on scroll-in | 6 independent loops on scroll-in (unchanged — acceptable, each is short-lived) |

## References

- Current implementation: `components/home/data-viz.tsx`, `lib/pledge-stats.ts`
- Supabase table schema: `pledges` with `deleted_apps text[]`, `individual_actions text[]`, `social_actions text[]`
- ISR config: `app/page.tsx:95` — `revalidate = 300`
- Pledge form app values: `components/home/pledge-form.tsx:17` — `["Instagram", "TikTok", "Snapchat", "X"]`
