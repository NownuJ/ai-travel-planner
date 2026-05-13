# UI Plan — Pages, Components, User Flow

## Stack
- **Tailwind CSS v4** (already configured)
- **shadcn/ui** — component library; initialize with the New York style variant and CSS variables
- **React state** for client-side trip data; no external state management library for v1
- **sessionStorage** to pass large data (recommendations, itinerary) between pages

## shadcn/ui Initialization

```bash
npx shadcn@latest init
# Choose: New York style, CSS variables: yes
```

## Components to Install

```bash
npx shadcn@latest add button card form input select slider badge tabs dialog alert-dialog alert skeleton avatar dropdown-menu collapsible separator
```

## Page Structure & User Flow

```
/ (Home)
│  → TravelForm submit
↓
/results (Recommendations + Map)
│  → Select a destination → generate itinerary
↓
/itinerary (Day-by-Day Itinerary + Map)
│  → "Save Trip"
│     ├── if guest → SaveTripDialog (prompts login/register)
│     └── if authenticated → POST /api/trips → success toast
↓
/saved (Saved Trips — auth-gated)

/login
/register
```

### State Passing Strategy Between Pages

| From → To | Mechanism | Why |
|---|---|---|
| Home → Results | URL search params (`?city=...&budget=...&days=...&interests=...`) | Small, shareable, survives refresh |
| Results → Itinerary | `sessionStorage` key `"ai_itinerary"` | Itinerary JSON is too large for URL params |
| Results page itself | `sessionStorage` key `"ai_recommendations"` | Avoids re-calling Gemini on page refresh |
| Itinerary → Save | Read `sessionStorage`; POST body | Only runs on user action |

On results page mount: check sessionStorage for cached recommendations matching current search params before calling the AI API. This avoids re-generating on browser back navigation.

---

## Pages

### `/` — Home

**Purpose**: Travel preferences form  
**Server/Client**: Client component (form interactivity)  
**Layout**: Full-screen hero with centered card

**Form Fields**:
| Field | Component | Notes |
|---|---|---|
| Departure city | `Input` | Plain text for v1; autocomplete is stretch goal |
| Budget | `Select` | Options: "Budget", "Mid-range", "Luxury" |
| Trip duration | `Slider` | Range: 3–14 days; display selected value as label |
| Interests | Badge toggle group | Multi-select; at least 1 required; options below |

**Interest options** (8 total): Culture, Food, Nature, Adventure, Relaxation, Nightlife, Shopping, History

**Submit behavior**:
1. Client-side validation (all fields filled, ≥1 interest)
2. Serialize preferences to URL search params
3. `router.push("/results?city=...&budget=...&days=...&interests=...")`

**shadcn/ui**: `Card`, `Form`, `Input`, `Select`, `Slider`, `Badge`, `Button`

---

### `/results` — Destination Recommendations

**Purpose**: AI-generated destination cards + map  
**Server/Client**: Client component (fetches AI data, manages interactive state)  
**Layout**: Two-column on desktop (cards left ~45%, map right ~55%); stacked on mobile (map below cards)

**On mount**:
1. Parse preferences from URL search params (validate; redirect to `/` if invalid)
2. Check `sessionStorage["ai_recommendations"]` — if present and matches current params, skip API call
3. Otherwise: `POST /api/ai/recommend` → store result in state and sessionStorage
4. Render

**State**:
```typescript
const [recommendations, setRecommendations] = useState<Destination[] | null>(null)
const [loading, setLoading] = useState(true)
const [error, setError] = useState<string | null>(null)
const [activeDestinationId, setActiveDestinationId] = useState<string | null>(null)
```

**Components**:
- `RecommendationsList` — renders 3–5 `DestinationCard` components; passes `activeId` and `onSelect`
- `DestinationCard`
  - Name, country, region
  - `whyItMatches` (italic, highlighted)
  - `estimatedBudgetPerDay` + `bestTimeToVisit` as small text
  - `highlights` as `Badge` chips
  - "View Itinerary" `Button`
  - Active state: ring/border highlight when `activeId === card.id`
- `DestinationMap` (dynamic, ssr:false) — markers synced to `activeDestinationId`
- Loading state: 3 `Skeleton` cards + `Skeleton` map block
- Error state: `Alert` with error message + "Try Again" `Button`

**"View Itinerary" flow**:
1. Store selected destination in sessionStorage
2. `router.push("/itinerary?city=...&budget=...&days=...&interests=...&destination=Kyoto%2C+Japan")`

**shadcn/ui**: `Card`, `CardHeader`, `CardContent`, `CardFooter`, `Badge`, `Button`, `Skeleton`, `Alert`

---

### `/itinerary` — Day-by-Day Itinerary

**Purpose**: Full itinerary for selected destination + activity map  
**Server/Client**: Client component  
**Layout**: Two-column on desktop (itinerary left, map right sticky); stacked on mobile

**On mount**:
1. Parse params (same preferences + destination name)
2. Check `sessionStorage["ai_itinerary"]` — use if present and matching
3. Otherwise: `POST /api/ai/itinerary` → store in state and sessionStorage
4. Render

**State**:
```typescript
const [itinerary, setItinerary] = useState<Itinerary | null>(null)
const [loading, setLoading] = useState(true)
const [error, setError] = useState<string | null>(null)
const [activeDay, setActiveDay] = useState<number | null>(null)  // null = all days
const [activeMarkerId, setActiveMarkerId] = useState<string | null>(null)
```

**Components**:

`ItineraryView`
- Header: destination name, trip duration, `Badge` for budget level
- "Save Trip" button (top-right corner — always visible)
- Day tabs: `Tabs` component with "All" + "Day 1" … "Day N" triggers
- Renders `ItineraryDay` for each visible day

`ItineraryDay`
- Day number + theme as heading
- Three activity blocks (Morning / Afternoon / Evening), each with: activity name, location (small), notes
- Meals section: breakfast / lunch / dinner suggestions
- Tip block: `Alert` styled with info icon
- Clicking an activity sets `activeMarkerId` (syncs with map)

`PracticalInfo` (collapsible section below the days)
- Getting There, Local Transport, Currency, Language, Emergency Tips
- Uses `Collapsible` — collapsed by default

`DestinationMap` (dynamic, ssr:false)
- Activity markers color-coded by day number
- Day filter controlled by `activeDay` state
- `activeMarkerId` causes map to fly to that marker

**"Save Trip" behavior**:
- If authenticated: call `POST /api/trips` with `{ title, preferences, destination, itinerary }` → show success `Alert`/toast
- If guest: open `Dialog` — "Sign in to save your trip" with Login and Register buttons; after auth redirect back with `?save=1` param to trigger auto-save
- `title` generated on client: `"{durationDays} Days in {destination}"` (e.g., "7 Days in Kyoto, Japan")

**shadcn/ui**: `Tabs`, `Card`, `Badge`, `Button`, `Dialog`, `DialogContent`, `Collapsible`, `Alert`, `Skeleton`, `Separator`

---

### `/saved` — Saved Trips

**Purpose**: List of authenticated user's saved trips  
**Server/Client**: Server component (auth check server-side; data fetch server-side)  
**Auth**: `auth()` check at top; `redirect("/login")` if no session

**Data fetch**:
```typescript
const trips = await fetch("/api/trips", { headers: { Cookie: ... } })
// Or use Prisma directly in a server component — call prisma.trip.findMany()
```

**Components**:
- Page heading + "Plan a New Trip" `Button` (links to `/`)
- Grid of `SavedTripCard` components
- Empty state: friendly message + "Plan Your First Trip" CTA

`SavedTripCard`
- Trip title
- Destination name + country (from `trip.destination` JSON)
- Duration (from `trip.preferences.durationDays`)
- Date saved (formatted)
- "View" `Button` → navigates to `/itinerary` with data loaded from the saved record
- "Delete" `Button` → opens `AlertDialog` for confirmation → `DELETE /api/trips/[id]`

**Loading the saved itinerary**: When "View" is clicked, write the saved `destination` and `itinerary` JSON back to sessionStorage (under the matching keys), then navigate to `/itinerary` with the appropriate params. The itinerary page will find the data in sessionStorage and skip the AI API call.

**shadcn/ui**: `Card`, `CardHeader`, `CardContent`, `CardFooter`, `Button`, `AlertDialog`, `AlertDialogContent`, `AlertDialogTrigger`, `Skeleton`

---

### `/login` and `/register`

**Layout**: Full-screen centered card  
**Server/Client**: Client component (form state, error handling)

`/login`:
- Email + password `Input` fields
- "Sign In" `Button` → calls NextAuth `signIn("credentials", { email, password })`
- "Continue with Google" `Button` → `signIn("google")`
- Error display: read `?error=` search param → show `Alert`
- Link to `/register`

`/register`:
- Name, email, password `Input` fields (password min 8 chars — validate client + server)
- "Create Account" `Button` → `POST /api/register` → on success `signIn("credentials", ...)`
- Error display via `Alert` (email already in use, password too short, etc.)
- Link to `/login`

**shadcn/ui**: `Card`, `CardHeader`, `CardContent`, `Form`, `Input`, `Button`, `Alert`, `Separator`

---

## Navbar (`components/layout/Navbar.tsx`)

**Server component** — calls `auth()` to read session server-side

| State | Left | Right |
|---|---|---|
| Unauthenticated | Logo / "AI Travel Planner" (links to `/`) | "Log in" + "Sign up" `Button` |
| Authenticated | Logo (links to `/`) | `Avatar` + `DropdownMenu` |

`DropdownMenu` items (authenticated):
- "My Saved Trips" → `/saved`
- `Separator`
- "Sign Out" → calls NextAuth `signOut()`

**shadcn/ui**: `Button`, `Avatar`, `AvatarFallback`, `AvatarImage`, `DropdownMenu`, `DropdownMenuContent`, `DropdownMenuItem`, `DropdownMenuSeparator`

---

## Shared TypeScript Types (`types/index.ts`)

```typescript
export type BudgetTier = "budget" | "mid-range" | "luxury"

export interface TravelPreferences {
  departureCity: string
  budget: BudgetTier
  durationDays: number
  interests: string[]
}

export interface Destination {
  name: string
  country: string
  region: string
  description: string
  whyItMatches: string
  estimatedBudgetPerDay: string
  bestTimeToVisit: string
  highlights: string[]
  coordinates?: [number, number]   // [lng, lat] — added server-side after geocoding
}

export interface ActivitySlot {
  activity: string
  location: string
  notes: string
  coordinates?: [number, number]   // [lng, lat] — added server-side after geocoding
}

export interface ItineraryDay {
  dayNumber: number
  theme: string
  morning: ActivitySlot
  afternoon: ActivitySlot
  evening: ActivitySlot
  meals: {
    breakfast?: string
    lunch?: string
    dinner?: string
  }
  tip: string
}

export interface PracticalInfo {
  gettingThere: string
  localTransport: string
  currency: string
  language: string
  emergencyTips: string
}

export interface Itinerary {
  destination: string
  days: ItineraryDay[]
  practicalInfo: PracticalInfo
}

export interface SavedTrip {
  id: string
  title: string
  preferences: TravelPreferences
  destination: Destination
  itinerary: Itinerary
  createdAt: string
}
```

---

## `hooks/useSessionStorage.ts`

```typescript
// A thin typed wrapper around sessionStorage.getItem/setItem
// Returns [value, setValue] — similar to useState but backed by sessionStorage
// Handles JSON parse errors gracefully (returns null, clears bad data)
```

---

## Responsive Design Notes
- Mobile-first breakpoints; use Tailwind `md:` and `lg:` prefixes
- Map on results/itinerary: `h-64` below cards on mobile; sticky `h-full` sidebar on `lg:`
- Interest badge grid: wraps naturally on small screens
- Navbar: collapses to hamburger menu on mobile (stretch goal for v1; simple stacked links acceptable)

## Assumptions
1. shadcn/ui New York variant with CSS variables — all components use `cn()` utility from `lib/utils.ts` (generated by shadcn init)
2. Dark mode is out of scope for v1; light mode only
3. No animation library (Framer Motion etc.) — Tailwind `transition-*` and `animate-*` utilities only
4. Toasts/notifications for save success use a `Alert` component inline, not a toast system — add `sonner` or `react-hot-toast` if the UX feels lacking during implementation
5. The departure city input is plain text for v1; Mapbox Places typeahead autocomplete is a stretch goal
