# AI Plan — Google Gemini 2.5 Flash

## Stack
- **SDK**: `@google/generative-ai` (official Google AI JS SDK)
- **Model**: `gemini-2.5-flash` (free tier via Google AI Studio)
- **Free tier limits** (verify at integration time — subject to change):
  - 15 requests per minute (RPM)
  - 1,000,000 tokens per minute (TPM)
  - 1,500 requests per day (RPD)
- **Output format**: Structured JSON via `responseMimeType: "application/json"` + `responseSchema`

## Two-Step AI Flow

Each trip generation makes exactly **two sequential AI calls**:

```
[User fills form]
      ↓
POST /api/ai/recommend   →  Gemini call #1  →  3–5 destinations with coordinates
      ↓
[User selects destination]
      ↓
POST /api/ai/itinerary   →  Gemini call #2  →  Day-by-day itinerary with coordinates
```

Coordinates are added server-side via Mapbox geocoding within each route (not a third AI call). See `plan-maps.md`.

## Gemini Client (`lib/gemini.ts`)

```typescript
// Pseudocode

import { GoogleGenerativeAI, SchemaType } from "@google/generative-ai"

const genAI = new GoogleGenerativeAI(process.env.GOOGLE_AI_API_KEY!)

export function getModel() {
  return genAI.getGenerativeModel({
    model: "gemini-2.5-flash",
    systemInstruction: SYSTEM_PROMPT,
  })
}
```

**Singleton pattern**: instantiate `GoogleGenerativeAI` once per module (not per request). Next.js module caching handles this in production.

## System Prompt (shared across both calls)

```
You are an expert travel planner with deep knowledge of global destinations, local culture, 
logistics, and budgeting. You provide accurate, practical, and personalized travel advice.

Always respond with valid JSON that exactly matches the schema provided. 
Do not wrap your response in markdown code blocks. Return raw JSON only.
Use specific, real place names — never placeholder or invented locations.
```

## API Route 1: `POST /api/ai/recommend`

### Request Body
```typescript
{
  departureCity: string      // "New York"
  budget: "budget" | "mid-range" | "luxury"
  durationDays: number       // 3–14
  interests: string[]        // ["culture", "food", "nature"]
}
```

### Prompt Template
```
A traveler departing from {departureCity} is planning a {durationDays}-day trip.
Budget level: {budget} (budget = hostels/street food/free attractions; mid-range = 3-star hotels/casual restaurants; luxury = 5-star/fine dining/premium experiences).
Interests: {interests.join(", ")}.

Recommend exactly 3 to 5 distinct international destinations that are an excellent match for this traveler.
For each destination, explain specifically why it fits their stated interests and budget level.
Vary the recommendations geographically where possible.
```

### Response Schema (passed to Gemini as `responseSchema`)
```typescript
{
  type: SchemaType.OBJECT,
  properties: {
    destinations: {
      type: SchemaType.ARRAY,
      items: {
        type: SchemaType.OBJECT,
        properties: {
          name: { type: SchemaType.STRING },
          country: { type: SchemaType.STRING },
          region: { type: SchemaType.STRING },         // e.g. "Southeast Asia"
          description: { type: SchemaType.STRING },    // 2–3 sentences
          whyItMatches: { type: SchemaType.STRING },   // specific to user's stated interests
          estimatedBudgetPerDay: { type: SchemaType.STRING }, // e.g. "$60–$90 USD"
          bestTimeToVisit: { type: SchemaType.STRING },
          highlights: {
            type: SchemaType.ARRAY,
            items: { type: SchemaType.STRING },        // 3–5 bullet points
          },
        },
        required: ["name", "country", "region", "description", "whyItMatches",
                   "estimatedBudgetPerDay", "bestTimeToVisit", "highlights"],
      },
    },
  },
  required: ["destinations"],
}
```

### Route Handler Logic
1. Parse and validate request body with **Zod** — return 400 on invalid input
2. Build prompt string from validated input
3. Call Gemini with JSON response schema and 30-second timeout
4. Parse response text as JSON
5. Validate parsed JSON shape (basic check — destinations is an array with ≥1 item)
6. Geocode each destination via `lib/mapbox.ts` (`Promise.all`) — add `coordinates: [lng, lat]` to each
7. Return enriched destinations array to client
8. No DB write

## API Route 2: `POST /api/ai/itinerary`

### Request Body
```typescript
{
  destination: string        // "Kyoto, Japan"
  departureCity: string      // "New York"
  budget: "budget" | "mid-range" | "luxury"
  durationDays: number
  interests: string[]
}
```

### Prompt Template
```
Create a detailed {durationDays}-day itinerary for {destination} for a traveler from {departureCity}.
Budget level: {budget}. Interests: {interests.join(", ")}.

For each day provide a theme, then specific morning, afternoon, and evening activities.
Use real place names (specific landmarks, restaurants, neighborhoods — not generic descriptions).
For meals, suggest specific restaurants or food options appropriate for the budget level.
Include one practical tip per day (transport, timing, booking advice, cultural notes).
```

### Response Schema
```typescript
{
  type: SchemaType.OBJECT,
  properties: {
    destination: { type: SchemaType.STRING },
    days: {
      type: SchemaType.ARRAY,
      items: {
        type: SchemaType.OBJECT,
        properties: {
          dayNumber: { type: SchemaType.INTEGER },
          theme: { type: SchemaType.STRING },
          morning: {
            type: SchemaType.OBJECT,
            properties: {
              activity: { type: SchemaType.STRING },
              location: { type: SchemaType.STRING },   // specific place name for geocoding
              notes: { type: SchemaType.STRING },
            },
            required: ["activity", "location", "notes"],
          },
          afternoon: { /* same shape as morning */ },
          evening: { /* same shape as morning */ },
          meals: {
            type: SchemaType.OBJECT,
            properties: {
              breakfast: { type: SchemaType.STRING },
              lunch: { type: SchemaType.STRING },
              dinner: { type: SchemaType.STRING },
            },
          },
          tip: { type: SchemaType.STRING },
        },
        required: ["dayNumber", "theme", "morning", "afternoon", "evening", "meals", "tip"],
      },
    },
    practicalInfo: {
      type: SchemaType.OBJECT,
      properties: {
        gettingThere: { type: SchemaType.STRING },
        localTransport: { type: SchemaType.STRING },
        currency: { type: SchemaType.STRING },
        language: { type: SchemaType.STRING },
        emergencyTips: { type: SchemaType.STRING },
      },
      required: ["gettingThere", "localTransport", "currency", "language", "emergencyTips"],
    },
  },
  required: ["destination", "days", "practicalInfo"],
}
```

### Route Handler Logic
1. Parse and validate request body with Zod
2. Build prompt
3. Call Gemini with JSON response schema and 30-second timeout
4. Parse and validate response JSON
5. Extract all `location` strings from all days (morning/afternoon/evening = up to 3 × durationDays locations)
6. Batch geocode via `Promise.all` in `lib/mapbox.ts` — add `coordinates` to each activity slot
7. Return enriched itinerary to client
8. No DB write (saving is a separate user action via `/api/trips`)

## Error Handling

| Scenario | HTTP Status | Client Behavior |
|---|---|---|
| Invalid request body (Zod) | 400 | Show form validation errors |
| Gemini returns non-JSON or schema mismatch | 502 | Show "Generation failed, please try again" with retry button |
| Gemini 429 rate limit | 429 + `Retry-After` header | Show "Too many requests — try again in X seconds" |
| Gemini network timeout (>30s) | 504 | Show "Request timed out, please try again" |
| Any unexpected error | 500 | Show generic error with retry button; log full error server-side |

**Retry logic**: one automatic retry on 502/504 with a 2-second delay before showing the error state to the user. Do not auto-retry on 429.

**Server-side logging**: log all non-400 errors with the full error message and timestamp. In v1 this can be `console.error` (Vercel captures these); do not log user input.

## Input Validation (Zod Schemas)

```typescript
// Shared validation in lib/validation.ts (or inline in route handlers)

const RecommendSchema = z.object({
  departureCity: z.string().min(2).max(100),
  budget: z.enum(["budget", "mid-range", "luxury"]),
  durationDays: z.number().int().min(3).max(14),
  interests: z.array(z.string()).min(1).max(8),
})

const ItinerarySchema = z.object({
  destination: z.string().min(2).max(200),
  departureCity: z.string().min(2).max(100),
  budget: z.enum(["budget", "mid-range", "luxury"]),
  durationDays: z.number().int().min(3).max(14),
  interests: z.array(z.string()).min(1).max(8),
})
```

## Client-Side Considerations
- Both routes are called from client components via `fetch`
- Show skeleton/loading UI immediately on submit (Gemini can take 3–15 seconds)
- No streaming for v1 — wait for full JSON response before rendering
- Store results in React state; pass between pages via sessionStorage (see `plan-ui.md`)

## Assumptions
1. Gemini 2.5 Flash free tier (1,500 RPD / 15 RPM) is sufficient for development and low-traffic v1
2. No server-side rate limiting beyond what Gemini enforces natively — add app-level rate limiting if traffic grows
3. `responseSchema` enforcement by Gemini is reliable enough that manual JSON repair is not needed for v1 — a single retry on malformed response is sufficient
4. No streaming for v1; full response before render simplifies JSON parsing
5. Zod is the chosen validation library — add it as a dependency
