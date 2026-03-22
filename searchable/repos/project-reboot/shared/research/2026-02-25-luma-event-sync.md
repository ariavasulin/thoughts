---
date: 2026-02-26T03:26:32+0000
researcher: ariasulin
git_commit: ec7be8f
branch: main
repository: project-reboot
topic: "Luma event sync integration — can we auto-pull events like Substack?"
tags: [research, codebase, luma, events, integration, substack, rss]
status: complete
last_updated: 2026-02-25
last_updated_by: ariasulin
---

# Research: Luma Event Sync Integration

**Date**: 2026-02-26T03:26:32+0000
**Researcher**: ariasulin
**Git Commit**: ec7be8f
**Branch**: main
**Repository**: project-reboot

## Research Question

Can we auto-pull events from Luma and sync them with the website, similar to the existing Substack RSS sync?

## Summary

**Yes, this is very feasible.** Luma has an official REST API (`GET /v1/calendar/list-events`) that returns structured JSON for all events on your calendar. This maps cleanly onto the existing Substack pattern: fetch server-side in a page component, parse, and pass as props to `EventCards`. The main prerequisite is a **Luma Plus subscription** for API access and an API key from your calendar settings. A free alternative is embedding Luma's calendar widget via iframe, but that sacrifices the custom card UI.

## Detailed Findings

### 1. Current Substack Integration (the pattern to replicate)

**File**: `app/page.tsx:81-93`

The Substack integration works as follows:
- `fetchFeed()` fetches `https://rebootwithucberkeley.substack.com/feed` (RSS/XML)
- Uses `next: { revalidate: 3600 }` on the fetch (1-hour cache)
- Regex-based XML parsing extracts `title`, `link`, `pubDate`, `description`, `imageUrl`
- The page itself sets `export const revalidate = 300` (5-minute ISR)
- Results passed as props to `<StayConnectedSection articles={articles} />`
- Graceful degradation: returns `[]` on any error

### 2. Current Events Implementation (what would change)

**File**: `app/events/page.tsx` — Static server component, renders `<EventCards>` and `<PhotoGallery>`

**File**: `components/events/event-cards.tsx:3-34` — Events are **hardcoded** in an `upcomingEvents` array:
```ts
const upcomingEvents = [
  {
    title: "Sunrise Hike & Reflection",
    date: "March 8, 2026",
    time: "6:30 AM",
    location: "Runyon Canyon, Los Angeles",
    description: "...",
    cta: "Sign Up",
    href: "#",
  },
  // ... 2 more
]
```

Each event card displays: date/time, title, location (with MapPin icon), description, and a CTA link. The component accepts a `variant: "dark" | "light"` prop. It also has an empty-state when `upcomingEvents.length === 0`.

**File**: `components/events/photo-gallery.tsx` — Past event photos, also hardcoded. This would remain unchanged (past events are static photos, not pulled from Luma).

### 3. Luma API Capabilities

**Base URL**: `https://public-api.luma.com`

**Key endpoint**: `GET /v1/calendar/list-events`
- Auth: `x-luma-api-key: YOUR_API_KEY` header
- Requirement: **Luma Plus subscription** on the calendar
- API key generated in Calendar > Settings > Developer > API Keys

**Query parameters for list-events**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `after` | ISO 8601 | Filter events after this timestamp |
| `before` | ISO 8601 | Filter events before this timestamp |
| `pagination_limit` | number | Max results per page |
| `sort_column` | enum | Only `start_at` |
| `sort_direction` | enum | `asc` or `desc` |
| `pagination_cursor` | string | For next page |

**Response shape**:
```json
{
  "entries": [
    {
      "event": {
        "name": "Event Title",
        "description": "Plain text",
        "description_md": "**Markdown**",
        "start_at": "2026-03-15T18:00:00.000Z",
        "end_at": "2026-03-15T20:00:00.000Z",
        "timezone": "America/Los_Angeles",
        "cover_url": "https://images.luma.com/...",
        "url": "https://lu.ma/event-slug",
        "geo_address_json": {
          "description": "Venue Name",
          "full_address": "123 Main St, Berkeley, CA",
          "city_state": "Berkeley, CA"
        },
        "meeting_url": "https://zoom.us/j/..."
      },
      "tags": [{ "name": "tag-name" }]
    }
  ],
  "has_more": false,
  "next_cursor": null
}
```

**Rate limits**:
- GET: 500 requests / 5 minutes per calendar
- POST: 100 requests / 5 minutes per calendar
- Exceeding → HTTP 429, blocked 1 minute

### 4. Field Mapping: Luma → EventCards

| EventCards field | Luma API field | Notes |
|------------------|----------------|-------|
| `title` | `event.name` | Direct map |
| `date` | `event.start_at` | Parse ISO → format to "March 8, 2026" |
| `time` | `event.start_at` | Parse ISO → format to "6:30 AM" using `event.timezone` |
| `location` | `event.geo_address_json.description` or `.city_state` | Use venue name or fallback to city |
| `description` | `event.description` | Plain text version |
| `cta` | — | Default to "RSVP" or "Sign Up" |
| `href` | `event.url` | Direct link to lu.ma event page |
| (new) `imageUrl` | `event.cover_url` | Could add cover images to cards |

### 5. Integration Options

**Option A — Luma API (recommended, matches Substack pattern)**:
- Add `LUMA_API_KEY` to `.env.local` (server-only, no `NEXT_PUBLIC_` prefix)
- Create `lib/luma.ts` with a `fetchLumaEvents()` function
- Fetch from `https://public-api.luma.com/v1/calendar/list-events?after={now}&sort_column=start_at&sort_direction=asc`
- Convert `events/page.tsx` from static to async server component (like `app/page.tsx`)
- Pass fetched events as props to `EventCards` instead of using hardcoded array
- Use ISR revalidation (e.g., `revalidate = 300` or `3600`)
- Graceful degradation: fall back to empty state on error (already handled in EventCards)
- Requires: **Luma Plus subscription**

**Option B — Luma Calendar Embed (free, no API key)**:
- Embed URL: `https://lu.ma/embed/calendar/{calendar-id}/events`
- Renders in an iframe with Luma's default styling
- No control over card design — loses the custom EventCards UI
- Simplest to implement but least integrated

**Option C — Hybrid (API + embed fallback)**:
- Use API when available, fall back to embed iframe if API fails

### 6. No RSS/iCal Feed Available

Unlike Substack, Luma does **not** expose an RSS or iCal subscription feed for calendars. The API is the only programmatic option for fetching structured event data. Individual events have "Add to Calendar" buttons (one-time `.ics` downloads), but there is no ongoing feed.

## Code References

- `app/page.tsx:81-93` — `fetchFeed()` function (Substack RSS pattern to replicate)
- `app/page.tsx:95` — `export const revalidate = 300` (ISR config)
- `app/events/page.tsx:11-51` — Events page layout (would become async)
- `components/events/event-cards.tsx:3-34` — Hardcoded `upcomingEvents` array (would be replaced by props)
- `components/events/event-cards.tsx:36-38` — `EventCardsProps` interface (would add events prop)
- `components/events/event-cards.tsx:43-58` — Empty state handler (already exists)
- `components/events/photo-gallery.tsx` — Past photos (no change needed)
- `lib/supabase.ts` — Server-only pattern to follow for Luma client

## Architecture Documentation

The integration follows the established server-side data fetching pattern:
1. Server component fetches external data with ISR caching
2. Data parsed/transformed in the server component or a `lib/` helper
3. Passed as props to presentational components
4. Graceful degradation on fetch failure

This is identical to the Substack flow and consistent with the project's server-only external access convention (no client-side API calls to third parties).

## Open Questions

1. **Does Project Reboot have a Luma Plus subscription?** Required for API access.
2. **What is the Luma calendar slug/ID?** Needed for generating the API key and/or embed URL.
3. **Should past events also be pulled from Luma?** Currently `PhotoGallery` uses hardcoded photos. Luma's API returns past events too — could supplement or replace.
4. **Should cover images be added to event cards?** Luma provides `cover_url` — the current cards don't have images but could.
5. **What CTA text should dynamically-fetched events use?** Current hardcoded events use different CTAs ("Sign Up", "Learn More", "RSVP"). Could default to "RSVP" for all Luma events since they link to the lu.ma registration page.

## External Sources

- [Luma API - Getting Started](https://docs.luma.com/reference/getting-started-with-your-api)
- [Luma API - List Events](https://docs.luma.com/reference/get_v1-calendar-list-events)
- [Luma API - Rate Limits](https://docs.luma.com/reference/rate-limits)
- [Luma Help - API Overview](https://help.luma.com/p/luma-api)
- [Luma Help - Embed Widget](https://help.luma.com/p/helpart-sGi8hj5CaMnMQWi/embed-our-checkout-registration-button-on-your-website)
