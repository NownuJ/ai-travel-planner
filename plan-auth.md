# Auth Plan — NextAuth.js

## Stack
- **NextAuth.js v5** (Auth.js) — `next-auth@^5`
- **Prisma Adapter** — `@auth/prisma-adapter`
- **Providers**: Credentials (email + bcrypt password) + Google OAuth (optional, design for it but can ship without)
- **Session strategy**: JWT — stateless, no DB query on every request, works well with Neon serverless

## File Layout

```
auth.ts                                    # NextAuth v5 root config — providers, adapter, callbacks
app/api/auth/[...nextauth]/route.ts        # Exports { GET, POST } from auth.ts
app/api/register/route.ts                  # Custom POST endpoint for new user registration
proxy.ts                                   # Route protection (see Next.js 16 note below)
lib/auth.ts                               # Re-export of auth() for use in server components/routes
```

**Why two auth files?** NextAuth v5 expects `auth.ts` at the project root. `lib/auth.ts` is a thin re-export to keep imports consistent with the rest of `lib/`.

## Guest vs Authenticated Flow

### Guest User
- No account required — the full form → recommendations → itinerary flow is always available
- Trip data is held in **client-side React state** and **sessionStorage** between pages
- Nothing is written to the database
- "Save Trip" button is visible but triggers a `Dialog` prompting login/register
- After logging in (redirect back or in-dialog), the itinerary data is read from sessionStorage and saved

### Authenticated User
- Same full flow as guest
- "Save Trip" button directly calls `POST /api/trips` (no login prompt)
- `/saved` page is accessible and shows all saved trips
- Session is available server-side via `auth()` in server components and route handlers

## NextAuth v5 Root Config (`auth.ts`)

```typescript
// Pseudocode — not executable

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: "jwt" },
  providers: [
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        // 1. Find user by email via prisma.user.findUnique
        // 2. If no user or no password (OAuth user), return null
        // 3. bcryptjs.compare(credentials.password, user.password)
        // 4. Return { id, email, name } or null
      },
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      // On first sign-in, user object is present — attach id to token
      if (user) token.id = user.id
      return token
    },
    session({ session, token }) {
      // Attach user.id from token to session object
      if (token.id) session.user.id = token.id as string
      return session
    },
  },
  pages: {
    signIn: "/login",
    error: "/login",   // Auth errors redirect here with ?error= param
  },
})
```

## Route Handler (`app/api/auth/[...nextauth]/route.ts`)

```typescript
import { handlers } from "@/auth"
export const { GET, POST } = handlers
```

## Registration Flow (Custom — Not NextAuth)

NextAuth does not handle registration. Use a custom endpoint.

**`POST /api/register`**:
1. Validate body: `{ name, email, password }` with Zod
2. Check if email already exists (`prisma.user.findUnique`)
3. Hash password: `bcryptjs.hash(password, 12)`
4. Create user: `prisma.user.create({ data: { name, email, password: hashedPassword } })`
5. Return `{ success: true }` — client then calls `signIn("credentials", { email, password })`

**Password requirements**: minimum 8 characters (validate client + server side).

## Password Handling

- Library: `bcryptjs` (pure JS — no native bindings, safe in Neon/Vercel serverless)
- Salt rounds: 12
- `password` column is **nullable** in the User model to accommodate OAuth-only users
- OAuth users who try to use Credentials login are rejected in `authorize()` with a clear error

## Route Protection (Next.js 16 Breaking Change — Important)

In **Next.js 16**, `middleware.ts` is deprecated and renamed to `proxy.ts`. The proxy file runs in the **Node.js runtime only** — the Edge runtime is not supported. NextAuth v5's built-in `auth` middleware helper historically ran on Edge.

**Recommended approach for this project**:

1. **`proxy.ts`** (Node.js runtime): Minimal route matching — redirect unauthenticated users away from `/saved` to `/login`. Use the NextAuth v5 `auth()` function.

```typescript
// proxy.ts
import { auth } from "@/auth"
import { NextResponse } from "next/server"

export default auth((req) => {
  const isLoggedIn = !!req.auth
  const isOnSaved = req.nextUrl.pathname.startsWith("/saved")

  if (isOnSaved && !isLoggedIn) {
    return NextResponse.redirect(new URL("/login", req.url))
  }
})

export const config = {
  matcher: ["/saved/:path*"],
}
```

2. **Server component auth check** (belt-and-suspenders): Each protected page also calls `auth()` server-side and redirects if no session — don't rely solely on proxy.

```typescript
// app/(main)/saved/page.tsx
import { auth } from "@/auth"
import { redirect } from "next/navigation"

export default async function SavedPage() {
  const session = await auth()
  if (!session) redirect("/login")
  // ...
}
```

3. **API route auth check**: All `/api/trips/*` routes check session at the top of the handler.

**Assumption**: NextAuth v5 `auth()` is compatible with the `proxy.ts` Node.js runtime in Next.js 16. Verify this during integration — if not compatible, fall back to option 2+3 only (no proxy-level protection).

## Session Shape

After the `session` callback runs, the session object has:
```typescript
session.user = {
  id: string        // DB cuid — added via jwt + session callbacks
  name: string | null
  email: string
  image: string | null
}
```

TypeScript: extend the `Session` and `JWT` types via NextAuth module augmentation in `types/next-auth.d.ts`.

## Auth Pages

### `/login`
- Email + password fields (`Form`, `Input`, `Button`)
- "Continue with Google" button (calls `signIn("google")`)
- Error display: read `?error=` search param, show `Alert`
- Link to `/register`
- On success: redirect to `/` (or back to the page that triggered the login prompt)

### `/register`
- Name, email, password fields
- Calls `POST /api/register`, then `signIn("credentials", { email, password })`
- Error display via `Alert`
- Link to `/login`

## Out of Scope for v1
- Email verification
- Password reset / forgot password
- "Remember me" / extended session duration
- Account linking (OAuth + Credentials for same email)
