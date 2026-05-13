# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev       # Start dev server (Turbopack, outputs to .next/dev)
npm run build     # Production build (Turbopack by default)
npm run start     # Start production server
npm run lint      # Run ESLint directly (next lint was removed in v16)
```

No test runner is configured yet.

## Architecture

Next.js 16 App Router project. All routes live under `app/` using file-system routing. The `@/*` alias maps to the project root (e.g., `@/app/components/Foo`).

## Critical: This is Next.js 16 — Breaking Changes from Prior Versions

Your training data likely reflects Next.js 14/15 conventions. The following have changed:

### Async-only Request APIs (Breaking)
`cookies()`, `headers()`, `draftMode()`, `params`, and `searchParams` are **fully async** — synchronous access was removed. Always `await` them:

```tsx
export default async function Page({ params }: PageProps<'/blog/[slug]'>) {
  const { slug } = await params
  const query = await searchParams  // also async
}
```

Run `npx next typegen` to generate `PageProps`, `LayoutProps`, and `RouteContext` type helpers.

### `middleware` → `proxy` (Breaking)
Rename `middleware.ts` to `proxy.ts` and the export from `middleware` to `proxy`. The edge runtime is **not supported** in `proxy` (it's Node.js only). `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`.

### Turbopack is now the default bundler
`next dev` and `next build` both use Turbopack by default — no flag needed. To opt out: `next dev --webpack` / `next build --webpack`. Turbopack config moves from `experimental.turbopack` to top-level `turbopack` in `next.config.ts`. Sass tilde imports (`~package/...`) are not supported — drop the `~`.

### `next lint` removed
Use `eslint` directly (already configured in `package.json` scripts). `next build` no longer runs the linter. ESLint now defaults to Flat Config (`eslint.config.mjs`).

### Caching API changes
- `revalidateTag` now requires a second `cacheLife` profile argument: `revalidateTag('posts', 'max')`
- New `updateTag` (Server Actions only) for immediate cache invalidation
- New `refresh()` to refresh the client router from a Server Action
- `unstable_cacheLife` / `unstable_cacheTag` → `cacheLife` / `cacheTag` (stable)
- `experimental.dynamicIO` → top-level `cacheComponents: true`
- PPR (`experimental.ppr`) removed → use `cacheComponents: true`

### Parallel Routes
All `@slot` directories now require an explicit `default.js` file or builds will fail.

### `next/image` changes
- `next/legacy/image` deprecated → use `next/image`
- `images.domains` deprecated → use `images.remotePatterns`
- Local images with query strings require `images.localPatterns.search` config
- Default `minimumCacheTTL` changed from 60s → 4 hours
- Default `qualities` changed to `[75]` only

### Removed
- `serverRuntimeConfig` / `publicRuntimeConfig` → use `process.env` / `NEXT_PUBLIC_` env vars
- AMP support (`next/amp`, `useAmp`, `amp` config)
- `devIndicators` options: `appIsrStatus`, `buildActivity`, `buildActivityPosition`
- `size` and `First Load JS` metrics from build output

## React 19.2 Features Available
View Transitions, `useEffectEvent`, Activity component. React Compiler is stable but opt-in (`reactCompiler: true` in `next.config.ts`).

## Reference
For any Next.js API questions, read the relevant guide in `node_modules/next/dist/docs/` — this is the authoritative source for v16 behavior.
