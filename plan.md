# AI Travel Planner — Master Plan

## Project Summary
AI Travel Planner is a Next.js web app where users enter travel preferences (departure city, budget, duration, interests) and receive AI-generated destination recommendations and a full day-by-day itinerary powered by Google Gemini 2.5 Flash. Users can browse as guests with no account required, or create an account to save and revisit past trip plans across sessions.

## Major Features
| Feature | Description |
|---|---|
| Travel Preferences Form | Departure city, budget tier, trip duration, and multi-select interest tags |
| AI Destination Recommendations | 3–5 Gemini-generated destinations with rationale, estimated cost, and highlights |
| Interactive Map | Mapbox map showing recommended destinations or itinerary activity locations as markers |
| Day-by-Day Itinerary | Full itinerary generated after user selects a destination; morning/afternoon/evening + meals |
| Guest Mode | Full app flow without an account; trip data lives in sessionStorage only |
| User Accounts | Email+password registration and login via NextAuth.js (Google OAuth optional) |
| Saved Trips | Authenticated users can save, view, and delete past trip plans |

## Sub-Plans
- [plan-auth.md](./plan-auth.md) — NextAuth.js setup, guest vs authenticated flow, session handling, route protection
- [plan-ai.md](./plan-ai.md) — Gemini 2.5 Flash integration, prompt design, API routes, error handling
- [plan-maps.md](./plan-maps.md) — Mapbox integration, geocoding flow, map components
- [plan-database.md](./plan-database.md) — Database choice (Neon + Prisma), schema, NextAuth adapter
- [plan-ui.md](./plan-ui.md) — Page structure, user flow, components, shadcn/ui usage

## Proposed Folder Structure

```
ai-travel-planner/
├── app/
│   ├── (auth)/
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (main)/
│   │   ├── layout.tsx              # Navbar + footer wrapper
│   │   ├── page.tsx                # Home: travel preferences form
│   │   ├── results/
│   │   │   └── page.tsx            # Destination recommendations + map
│   │   ├── itinerary/
│   │   │   └── page.tsx            # Day-by-day itinerary + map
│   │   └── saved/
│   │       └── page.tsx            # Saved trips list (auth-gated)
│   ├── api/
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts        # NextAuth v5 handler
│   │   ├── register/
│   │   │   └── route.ts            # Custom registration endpoint
│   │   ├── ai/
│   │   │   ├── recommend/
│   │   │   │   └── route.ts        # POST: generate destination recommendations
│   │   │   └── itinerary/
│   │   │       └── route.ts        # POST: generate day-by-day itinerary
│   │   └── trips/
│   │       ├── route.ts            # GET (list), POST (save)
│   │       └── [id]/
│   │           └── route.ts        # GET (single), DELETE
│   ├── layout.tsx                  # Root layout: fonts, providers, globals
│   └── globals.css
├── components/
│   ├── ui/                         # shadcn/ui generated components (do not edit manually)
│   ├── forms/
│   │   └── TravelForm.tsx
│   ├── map/
│   │   └── DestinationMap.tsx      # Client-only; always dynamically imported (ssr: false)
│   ├── recommendations/
│   │   ├── DestinationCard.tsx
│   │   └── RecommendationsList.tsx
│   ├── itinerary/
│   │   ├── ItineraryDay.tsx
│   │   ├── ItineraryView.tsx
│   │   └── PracticalInfo.tsx
│   ├── trips/
│   │   └── SavedTripCard.tsx
│   └── layout/
│       ├── Navbar.tsx
│       └── Footer.tsx
├── lib/
│   ├── auth.ts                     # NextAuth v5 config (providers, adapter, callbacks)
│   ├── prisma.ts                   # Prisma client singleton
│   ├── gemini.ts                   # Gemini client + prompt builders
│   └── mapbox.ts                   # Server-side geocoding helper
├── types/
│   └── index.ts                    # TravelPreferences, Destination, Itinerary, etc.
├── hooks/
│   └── useSessionStorage.ts        # Helper for reading/writing trip state to sessionStorage
├── prisma/
│   └── schema.prisma
├── public/
├── auth.ts                         # NextAuth v5 root export (used by route handler + server components)
├── proxy.ts                        # Next.js 16 route protection (replaces middleware.ts)
└── ...config files (next.config.ts, tsconfig.json, postcss.config.mjs, eslint.config.mjs)
```

## Environment Variables

```bash
# NextAuth
NEXTAUTH_SECRET=
NEXTAUTH_URL=

# Google OAuth (optional provider)
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Database — Neon PostgreSQL (see plan-database.md)
DATABASE_URL=          # Neon pooled connection string (used at runtime)
DIRECT_URL=            # Neon direct connection string (used by Prisma migrations only)

# Google AI Studio — Gemini 2.5 Flash
GOOGLE_AI_API_KEY=

# Mapbox (public — safe to expose to client)
NEXT_PUBLIC_MAPBOX_TOKEN=
```

## Definition of Done — v1

- [ ] User can submit travel preferences and receive 3–5 AI-generated destination recommendations
- [ ] User can select a destination and receive a full day-by-day itinerary
- [ ] Mapbox map renders on the results page (destination markers) and itinerary page (activity markers)
- [ ] Guest users can complete the full form → recommendations → itinerary flow without creating an account
- [ ] Guest trip data is not persisted to the database; it exists in sessionStorage only
- [ ] Users can register with email + password and log in
- [ ] Authenticated users can save a trip and view it on the `/saved` page
- [ ] Authenticated users can delete a saved trip
- [ ] AI API failures return a user-facing error message with a retry option (no blank/broken states)
- [ ] No API keys or secrets are exposed in the client bundle (only `NEXT_PUBLIC_MAPBOX_TOKEN` is public)
- [ ] App deploys to Vercel without build errors
