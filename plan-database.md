# Database Plan — Neon + Prisma

## Database Choice: Neon (Serverless PostgreSQL)

### Why Neon
- **Free tier**: 0.5 GB storage, 190 compute hours/month, up to 10 branches — sufficient for v1 and early users
- **Serverless**: scales to zero; no idle compute cost; built-in connection pooler (Neon Proxy) prevents connection limit exhaustion in Next.js serverless functions
- **PostgreSQL**: relational model fits structured trip data and NextAuth's required tables; mature ecosystem
- **Prisma adapter**: `@auth/prisma-adapter` is the most mature and best-documented NextAuth adapter; generated types match the schema exactly
- **Prisma ORM**: strong TypeScript support, migration tooling, readable schema DSL

### Alternatives Considered
| Option | Reason Not Chosen |
|---|---|
| Supabase | Adds BaaS overhead (auth, storage, realtime) that isn't needed; more complex setup for a focused v1 |
| PlanetScale | MySQL-based; no longer has a free tier |
| Turso (SQLite/libSQL) | Less mature Prisma support; `@auth/prisma-adapter` compatibility not guaranteed |
| MongoDB Atlas | NoSQL; overkill for structured relational data; NextAuth adapter exists but less battle-tested |

## Packages
- `prisma` (devDependency) — CLI and schema compiler
- `@prisma/client` — generated runtime client
- `@auth/prisma-adapter` — bridges NextAuth v5 to Prisma models

## Prisma Schema (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")   // Neon pooled URL — used at runtime
  directUrl = env("DIRECT_URL")     // Neon direct URL — used by prisma migrate only
}

// ── NextAuth Required Models ─────────────────────────────────────────────────

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}

// ── Application Models ───────────────────────────────────────────────────────

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  emailVerified DateTime?
  image         String?
  password      String?   // Null for OAuth-only users
  createdAt     DateTime  @default(now())

  accounts Account[]
  sessions Session[]
  trips    Trip[]
}

model Trip {
  id          String   @id @default(cuid())
  userId      String
  title       String   // e.g. "7 Days in Kyoto, Japan"
  preferences Json     // TravelPreferences object
  destination Json     // Selected Destination object (includes coordinates)
  itinerary   Json     // Full Itinerary object (includes coordinates on activity slots)
  createdAt   DateTime @default(now())

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}
```

### Why `Json` Columns for Trip Data
Storing `preferences`, `destination`, and `itinerary` as JSON rather than normalized relational tables keeps the schema simple for v1. The AI output shape may evolve; JSON columns avoid migrations every time the Gemini prompt output changes. TypeScript types in `types/index.ts` enforce shape at the application layer. Normalization can be introduced later if query patterns demand it (e.g., filtering trips by destination country).

## Prisma Client Singleton (`lib/prisma.ts`)

```typescript
import { PrismaClient } from "@prisma/client"

const globalForPrisma = global as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error"] : ["error"],
  })

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma
```

This singleton pattern prevents multiple `PrismaClient` instances during Next.js hot reload in development, which would exhaust the database connection pool.

## Neon Connection Strings

Neon provides two connection strings per project:

| Variable | URL Type | Used By |
|---|---|---|
| `DATABASE_URL` | Pooled (via Neon Proxy) | Runtime — all Prisma queries in production |
| `DIRECT_URL` | Direct (no pooler) | `prisma migrate deploy` and `prisma db push` only |

**Why two URLs?** Neon's connection pooler doesn't support the extended query protocol that Prisma migrations use. The `directUrl` bypasses the pooler for migrations only.

## Trip API Routes

All trip routes require an authenticated session. Check at the top of each handler using `auth()` from NextAuth v5.

### `GET /api/trips`
```typescript
// Returns all trips for the authenticated user, newest first
const trips = await prisma.trip.findMany({
  where: { userId: session.user.id },
  orderBy: { createdAt: "desc" },
  select: {
    id: true,
    title: true,
    destination: true,   // include for display on the saved trips page
    createdAt: true,
    // Omit itinerary (large) from list view — fetch on demand
  },
})
```

### `POST /api/trips`
```typescript
// Body: { title, preferences, destination, itinerary }
// Validate with Zod before writing
const trip = await prisma.trip.create({
  data: {
    userId: session.user.id,
    title,
    preferences,
    destination,
    itinerary,
  },
})
// Return: { id: trip.id }
```

### `GET /api/trips/[id]`
```typescript
const trip = await prisma.trip.findUnique({ where: { id } })
if (!trip || trip.userId !== session.user.id) return 404
// Return full trip including itinerary
```

### `DELETE /api/trips/[id]`
```typescript
const trip = await prisma.trip.findUnique({ where: { id }, select: { userId: true } })
if (!trip || trip.userId !== session.user.id) return 403
await prisma.trip.delete({ where: { id } })
// Return 204 No Content
```

## Migration Workflow

| Environment | Command |
|---|---|
| Development (iterate fast) | `npx prisma db push` — pushes schema without migration files |
| Production deploy | `npx prisma migrate deploy` — runs pending migration files |
| Generate initial migration | `npx prisma migrate dev --name init` |

**First-time setup**:
1. Create Neon project → copy both connection strings to `.env.local`
2. `npx prisma generate` — generates the Prisma client
3. `npx prisma db push` — creates tables in the Neon database

## Assumptions
1. `prisma db push` is used during development; proper migration files (`prisma migrate dev`) are created before any production deploy
2. No soft deletes — hard delete is acceptable for trip records in v1
3. No pagination on the saved trips list in v1 — `findMany` returns all trips; add pagination if a user accumulates many trips
4. The `Trip.title` is generated client-side before the save API call (e.g., `"7 Days in Kyoto, Japan"`) — it is not AI-generated
