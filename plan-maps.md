# Maps Plan — Mapbox

## Stack
- **Package**: `react-map-gl` v8+ (React wrapper around Mapbox GL JS)
- **Geocoding**: Mapbox Geocoding API v6 — called **server-side only** from `lib/mapbox.ts`
- **Token**: `NEXT_PUBLIC_MAPBOX_TOKEN` — this is a public token (Mapbox design intent); restrict it in the Mapbox dashboard to your production domain

## Why `react-map-gl` Over Raw `mapbox-gl`
- Declarative React API for `<Map>`, `<Marker>`, `<Popup>` — no imperative DOM manipulation
- Handles resize/cleanup lifecycle automatically
- Works well with Next.js dynamic imports
- Maintained by Vis.gl (Urban Computing Foundation)

## Map Views

### View 1: Recommendations Map (`/results`)
- Renders after Gemini returns destination recommendations
- One marker per recommended destination (3–5 markers)
- Map auto-fits viewport bounds to show all markers on load
- Clicking a marker highlights the corresponding destination card (via shared `activeId` state)
- Clicking a destination card causes the map to fly to that marker (`flyTo`)
- Marker color: single accent color (no differentiation needed — all are candidate destinations)
- Popup on marker click: destination name + country

### View 2: Itinerary Map (`/itinerary`)
- Renders after itinerary is generated
- One marker per activity location across all days (up to 3 × durationDays markers)
- Markers are **numbered and color-coded by day** (e.g., Day 1 = blue, Day 2 = green, …)
- Day filter: tab/button row ("All Days" | "Day 1" | "Day 2" | …) — toggling filters visible markers
- Clicking a marker shows a popup: day number, time of day (morning/afternoon/evening), activity name
- Some activities may share the same location (e.g., dinner near a morning landmark) — deduplicate by `[lng, lat]` before rendering; show combined popup

## Geocoding Flow

### Why Server-Side Only
Mapbox geocoding is called exclusively from the Next.js API routes, not from the browser. Reasons:
1. Avoids an extra client round-trip (results are enriched before the response is sent)
2. The Mapbox token is in `NEXT_PUBLIC_*` so it is technically exposed, but keeping geocoding server-side avoids client network tab exposure and keeps the logic centralized

### Recommendations Geocoding
- Done inside `POST /api/ai/recommend` after Gemini returns destinations
- Query per destination: `"{destination.name}, {destination.country}"` (e.g., "Kyoto, Japan")
- Run all geocoding requests in parallel: `Promise.all(destinations.map(geocode))`
- Each destination gets a `coordinates: [lng, lat]` field added before the response is sent to the client

### Itinerary Geocoding
- Done inside `POST /api/ai/itinerary` after Gemini returns the itinerary
- Extract all `location` strings from every activity slot (morning/afternoon/evening across all days)
- Run in parallel: `Promise.all(locations.map(geocode))`
- Each activity slot gets `coordinates: [lng, lat]` added if geocoding succeeds
- If a location fails to geocode, set `coordinates: null` — that activity is shown in the itinerary without a map marker (graceful degradation, no error thrown)

### `lib/mapbox.ts` — Geocoding Helper

```typescript
// Pseudocode

const GEOCODING_BASE = "https://api.mapbox.com/search/geocode/v6/forward"

export async function forwardGeocode(
  query: string
): Promise<[lng: number, lat: number] | null> {
  const url = new URL(GEOCODING_BASE)
  url.searchParams.set("q", query)
  url.searchParams.set("access_token", process.env.NEXT_PUBLIC_MAPBOX_TOKEN!)
  url.searchParams.set("limit", "1")

  const res = await fetch(url.toString(), { next: { revalidate: 3600 } })
  // Cache geocoding results for 1 hour using Next.js fetch cache
  // (same location name queried multiple times in one session returns instantly)

  if (!res.ok) return null

  const data = await res.json()
  const feature = data.features?.[0]
  if (!feature) return null

  return [feature.geometry.coordinates[0], feature.geometry.coordinates[1]]
}
```

**Caching note**: using `next: { revalidate: 3600 }` on the geocoding fetch leverages Next.js's built-in data cache. The same location string won't hit the Mapbox API more than once per hour, staying well within the 100,000/month free tier limit.

## `DestinationMap` Component

```
components/map/DestinationMap.tsx
```

### Interface
```typescript
interface MarkerData {
  id: string
  lng: number
  lat: number
  label: string        // Display name shown in popup
  color?: string       // CSS color string; defaults to accent blue
  dayNumber?: number   // For itinerary map — used to color-code and filter
}

interface DestinationMapProps {
  markers: MarkerData[]
  activeMarkerId?: string
  onMarkerClick?: (id: string) => void
  filterDay?: number | null   // null = show all; number = show only that day
  className?: string
}
```

### Implementation Notes
- `'use client'` directive required
- **Always dynamically imported** — never import directly (Mapbox GL uses browser APIs)
- Map style: `"mapbox://styles/mapbox/light-v11"` (clean, lets markers stand out)
- Use `fitBounds` from `react-map-gl/maplibre` or WebMercatorViewport to auto-fit all markers on load
- `flyTo` when `activeMarkerId` changes (smooth pan/zoom to that marker)
- Marker click calls `onMarkerClick(marker.id)` — parent manages `activeMarkerId` state

### Dynamic Import Pattern

```typescript
// In results/page.tsx and itinerary/page.tsx:
import dynamic from "next/dynamic"

const DestinationMap = dynamic(
  () => import("@/components/map/DestinationMap"),
  {
    ssr: false,
    loading: () => (
      <div className="h-full min-h-64 bg-muted animate-pulse rounded-lg" />
    ),
  }
)
```

## Free Tier Usage Estimates

| Resource | Free Limit | Estimated v1 Usage |
|---|---|---|
| Map loads | 50,000/month | 1 per page view with a map (2 pages × visitors) |
| Geocoding requests | 100,000/month | ~8 per trip generation (5 destinations + 3 per itinerary day avg) |

Both limits are comfortably within range for a hobby/v1 app with ≤ a few hundred users.

## Mapbox Dashboard Setup (Before Development)
1. Create a Mapbox account at mapbox.com
2. Create a public access token restricted to your domains (localhost:3000 for dev, production domain)
3. Enable "Styles API" and "Geocoding API" scopes on the token
4. Set `NEXT_PUBLIC_MAPBOX_TOKEN` in `.env.local`

## Assumptions
1. `react-map-gl` v8 is compatible with Next.js 16 App Router — verify at integration time; check for any peer dependency conflicts with React 19
2. Mapbox Geocoding API v6 endpoint is stable — check release notes if errors occur
3. Activities with `coordinates: null` are excluded from the map without breaking the itinerary UI
4. No custom Mapbox Studio map styles for v1 — use built-in `light-v11`
5. Typeahead/autocomplete for the departure city input field is out of scope for v1 (plain text input is used)
