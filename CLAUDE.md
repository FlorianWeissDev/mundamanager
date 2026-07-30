# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**Munda Manager** — a gang and campaign management web app for the Necromunda tabletop game
(gang rosters, fighters, equipment, vehicles, campaigns, battle sessions, territories).
Production: https://www.mundamanager.com

Single Next.js application backed by Supabase. There is no separate backend service: business
logic lives in Next.js Server Actions and Route Handlers, plus Postgres RPC functions, triggers,
and RLS policies in Supabase.

## Stack

| Concern | Choice |
|---|---|
| Framework | Next.js 16 (App Router, RSC), React 19 |
| Language | TypeScript (`strict: true`), `@/*` → repo root |
| Database / Auth / Storage / Realtime | Supabase (Postgres 17) |
| Styling | Tailwind CSS v4 (`@import "tailwindcss"` + `@config`), shadcn-style components |
| Client data fetching | TanStack Query v5 (admin/customise screens, lookup data) |
| Editor / maps / DnD | TipTap, Leaflet + Geoman, dnd-kit |
| Toasts | `sonner` |
| Hosting / CI | Vercel, deployed from GitHub Actions |

Node `>= 20.20.2` (`.nvmrc` pins `20.20.2`).

## Commands

```bash
npm install
npm run dev      # next dev --turbopack
npm run build    # next build — this is also the type check (tsconfig has noEmit)
npm run start
npm run lint     # eslint . (flat config, eslint-config-next/core-web-vitals)
npm run clean
```

**There is no test suite.** Verification = `npm run lint` and `npm run build`. Do not claim tests
pass; say what you actually ran.

Environment: copy `.env.example` → `.env.local`. Supabase URL/anon key are required for the app to
boot; Turnstile, Discord, and `UNSUBSCRIBE_SECRET` gate specific features.
`SUPABASE_SERVICE_ROLE_KEY` is only needed for the code paths using `createServiceRoleClient()`.

Local database: `supabase start` / `supabase db reset` (Supabase CLI + Docker). See the README —
migrations are **deliberately disabled locally** (`[db.migrations] enabled = false`); fresh local
databases are built from `supabase/schema/schema.public.sql` + bootstrap + `seed.sql` via
`[db.seed].sql_paths`.

## Repository layout

```
app/
  layout.tsx, page.tsx, footer.tsx, not-found.tsx, sitemap.ts, globals.css
  @breadcrumb/          Parallel route rendering the breadcrumb bar per page (mirrors route tree)
  <route>/page.tsx      Server Components: auth → fetch → render a client component
  api/**/route.ts       ~67 Route Handlers (client fetch / TanStack Query targets)
  actions/**            Server Actions ('use server') — all writes go through here
  actions/logs/**       Gang activity-log writers used by the actions above
  lib/**                Cached server-side read functions (unstable_cache + cache tags)
  lib/shared/           gang-data.ts, fighter-data.ts — the two big granular data layers
  providers/            React Query provider
components/            Feature-grouped React components; components/ui = primitives
hooks/                 Client hooks (use-*)
utils/                 Cross-cutting logic: cache-tags, auth, permissions, domain rules
types/                 Shared TypeScript interfaces
supabase/              config.toml, migrations/, functions/, edge-functions/, schema/, bootstrap/,
                       webhooks/, seed.sql
proxy.ts               Next 16 proxy (formerly middleware.ts) — auth gate
.github/workflows/     Vercel deploy/preview, Supabase function + edge-function deploy, schema snapshot
```

## Core architecture

### 1. Reads: Server Component → cached fetcher → client component props

Route `page.tsx` files are async Server Components. The pattern (see `app/gang/[id]/page.tsx`):

```ts
const supabase = await createClient();                 // utils/supabase/server.ts
const user = await getAuthenticatedUser(supabase);     // throws → redirect(signInPath(...))
const gangBasic = await getGangBasic(id, supabase);    // existence check first
if (!gangBasic) notFound();
if (!canView) forbidden();
const [a, b, c] = await Promise.all([...granular fetchers...]);
return <GangPageContent initialGangData={...} userPermissions={...} />;
```

Fetchers live in `app/lib/**` and each wraps its query in `unstable_cache` with an explicit tag and
`revalidate: false` (global reference data uses a TTL instead):

```ts
export const getGangBasic = async (gangId: string, supabase: any) =>
  unstable_cache(
    async () => { /* supabase query */ },
    [`gang-basic-${gangId}`],
    { tags: [CACHE_TAGS.BASE_GANG_BASIC(gangId)], revalidate: false }
  )();
```

Keep fetchers **granular** (one concern per function) so invalidation stays surgical. Fetch them in
parallel at the page level.

### 2. Writes: Server Actions

`app/actions/*.ts` start with `'use server'` and follow a consistent shape:

```ts
export async function doThing(params: Params): Promise<{ success: boolean; data?: T; error?: string }> {
  const supabase = await createClient();
  try {
    const user = await getAuthenticatedUser(supabase);
    // ...mutate...
    // ...update gang financials if money/rating changed...
    // ...write an activity log...
    // ...invalidate cache tags...
    return { success: true, data };
  } catch (error) {
    return { success: false, error: error instanceof Error ? error.message : 'Unknown error' };
  }
}
```

Actions **return** results rather than throwing to the client; callers surface errors via `toast`.
Client components typically apply the returned values to local state instead of calling
`router.refresh()` (only two `router.refresh()` calls exist in the whole codebase).

### 3. Cache invalidation — `utils/cache-tags.ts`

This file is the heart of data freshness and the easiest thing to get wrong. Tags are hierarchical
and named `CATEGORY_ENTITY_SCOPE`:

- `BASE_*` — raw entities (`BASE_GANG_CREDITS`, `BASE_FIGHTER_EQUIPMENT`, …)
- `COMPUTED_*` — derived values (`COMPUTED_GANG_RATING`, `COMPUTED_FIGHTER_TOTAL_COST`)
- `COMPOSITE_*` — multi-entity aggregates
- `USER_*` — per-user collections (`USER_GANGS`, `USER_CUSTOM_*`)
- `SHARED_*` — same data rendered on multiple pages (`SHARED_GANG_RATING`)
- `GLOBAL_*` — reference/statistics data
- `CHECK_PERMISSION(userId, gangId)`

Rules:
- **Never call `revalidateTag` with a hand-written string.** Use `CACHE_TAGS.*`.
- Prefer the composed helpers (`invalidateEquipmentPurchase`, `invalidateFighterAddition`,
  `invalidateGangFinancials`, `invalidateFighterDataWithFinancials`, …) over ad-hoc tag lists — they
  encode which tags a given user-facing operation actually dirties.
- New cached fetcher ⇒ add a tag ⇒ wire it into every action that can change that data.

### 4. Route Handlers (`app/api/**`)

Used by client components (often through TanStack Query) for lookups, admin CRUD, search, and
webhooks. `proxy.ts` **does not** run on `/api`, so every handler authenticates itself:

```ts
const supabase = await createClient();
const userId = await getUserIdFromClaims(supabase);
if (!userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
```

### 5. Auth and permissions

- `proxy.ts` (Next 16's renamed middleware) gates page routes: redirects signed-in users away from
  `/sign-in`/`/sign-up`, allows a fixed `publicPaths` list, and otherwise redirects to
  `/sign-in?next=…`. It skips Server Action requests (`Next-Action` header) and `/api`.
- `utils/auth.ts` is the only auth entry point: `getClaims`, `getAuthenticatedUser` (throws),
  `getUserIdFromClaims` (nullable), `checkAdmin`, plus `safePath`/`signInPath` redirect sanitizers.
  These read `supabase.auth.getClaims()` — a **local JWT read, no network round trip**. Do not
  reintroduce `supabase.auth.getUser()` in hot paths. A Postgres `custom_access_token_hook` injects
  a `user_profile` claim.
- `utils/user-permissions.ts` wraps the `check_permission` RPC and derives permission objects:
  `deriveGangPermissions` (owner / admin / campaign arbitrator ⇒ `canEdit`, `canDelete`) and
  `deriveCampaignPermissions` (OWNER > ARBITRATOR > MEMBER role ladder). Pages compute permissions
  server-side and pass a `UserPermissions` object down as props.
- Supabase RLS is the real security boundary; UI permission objects are for affordances. Server-side
  checks still matter — don't rely on the client hiding a button.

Supabase clients:
| Use | Import |
|---|---|
| Server Components / Actions / Route Handlers | `createClient()` from `@/utils/supabase/server` (`server-only`) |
| Elevated, RLS-bypassing server work | `createServiceRoleClient()` from the same file |
| Browser | `createClient()` from `@/utils/supabase/client` |

## Database workflow (`supabase/`)

Read `supabase/README.md` before touching anything here. Summary of how each piece reaches
production:

| Piece | Location | Applied by |
|---|---|---|
| Schema deltas | `migrations/*.sql` | Applied to the remote; **not** replayed locally |
| RPC / trigger / helper functions | `functions/*.sql` | `deploy_supabase_functions.yml` — psql-applies only files changed in the push to `main` |
| Edge functions (Deno) | `edge-functions/*/index.ts` | `deploy_supabase_edge_functions.yml`; registered in `config.toml` |
| Public schema snapshot | `schema/schema.public.sql` | Regenerated daily by `supabase_schema_snapshot.yml` (commits `chore(db): update Supabase schema snapshot`) |
| `private` schema, grants, auth trigger | `bootstrap/*.sql` | Local seed paths; manual/migration on remote |
| Reference game data | `seed.sql` | Local seed only |
| Database webhooks | `webhooks/*.sql` | **Manually** — they embed a header secret and are stripped from snapshots |

Notes:
- `functions/*.sql` are **not** auto-synced from the Supabase dashboard. If a function changes in the
  dashboard, mirror it here or the next CI deploy will silently revert it.
- Migration filenames are `YYYYMMDDHHMMSS_description.sql`.
- The `private` schema (`is_admin`, `is_arb`) holds SECURITY DEFINER helpers referenced by ~200 RLS
  policies; it must exist before the public schema loads.
- Never commit real secrets into `webhooks/` or workflow files.

## Domain model — things to get right

**Fighter effects** (`utils/effect-modifiers.ts`, `types/fighter-effect.ts`). Every stat modification
— injuries, advancements, bionics, cyberteknika, gene-smithing, rig-glitches, power-boosts,
augmentations, equipment, skills, vehicle damages, user tweaks — is a `fighter_effects` row with
`fighter_effect_modifiers` children, grouped by category. `calculateAdjustedStats()` applies them
over base stats (supports `add` and `set`); `applyWeaponModifiers()` handles equipment-on-equipment
weapon profile changes.

> ⚠️ `components/fighter/fighter-details-card.tsx` and `components/gang/fighter-card.tsx` each build
> their own `fighterData.effects` object before calling `calculateAdjustedStats()`. **A new effect
> category must be added to both**, or stats will differ between the fighter page and the gang
> roster. Also update `getFighterEffects()` in `app/lib/shared/fighter-data.ts` and the types.

**Gang financials** (`utils/gang-rating-and-wealth.ts`). Credits, rating, and wealth move together
through `updateGangFinancials({ ratingDelta, creditsDelta, stashValueDelta, applyToRating })`, which
returns old/new values for logging. Don't update `gangs.credits`/`rating`/`wealth` directly.

**Fighter status** (`utils/fighter-status.ts`). `countsTowardRating()` is the single source of truth:
killed / retired / enslaved / captured fighters do **not** count; recovery and starved do.
`isStatusIncompatible()` drives status-button disabling.

**Activity logs** (`app/actions/logs/**` plus Postgres triggers). Gang actions write human-readable
log lines (`"Fighter 'Ganger' bought Lasgun for 15 credits. New gang rating: 280"`). Use the existing
`log*` helpers so descriptions stay consistent and duplicates are suppressed.

**Custom / user content** (`app/actions/customise/`, `app/lib/customise/`, `components/customise/`).
Users author their own equipment, skills, fighter types, gang types, trading posts, and collections;
these shadow the global tables everywhere (queries usually fetch both and merge, tagging custom rows).
Invalidate with `invalidateUserCustom*` / `invalidateAllUserCustomContent`.

Other notable subsystems: exotic beasts (`utils/exotic-beasts.ts` — beasts are fighters owned by
fighters, so owner caches need invalidating too), battle sessions (with Supabase Realtime via
`hooks/use-battle-session-realtime.ts`), campaign maps (Leaflet + hex grid), notifications
(Realtime + email outbox → edge function), Discord integration, and Patreon sync.

## Conventions

- **Files** are kebab-case (`fighter-details-card.tsx`, `move-to-stash.ts`). Components are usually
  default exports; utilities are named exports.
- **British spelling in domain/feature names**: `customise`, `favourite`, `colour`, `armour`. Match
  the surrounding code — the database columns use these spellings.
- **Imports** use the `@/` alias from the repo root. `cn()` comes from `@/app/lib/utils` (note:
  `components.json` still says `@/lib/utils`; that alias is stale — follow the imports, not the file).
- **UI**: reuse `components/ui/*` primitives — especially `Modal` (with its `onConfirm` returning
  `false` to keep the modal open), `Button`, `List`, `Combobox`. Icons come from `react-icons`.
  Theming is class-based dark mode over HSL CSS variables; use the semantic tokens
  (`bg-card`, `text-muted-foreground`) rather than raw colours.
- **Feedback** is `toast` from `sonner`; there is no custom toast wrapper.
- **Client components** declare `'use client'` and receive server-fetched data as props. Keep data
  fetching on the server; use TanStack Query only for genuinely client-driven data (admin panels,
  search, lookups). The query client defaults to `staleTime: 5m`, `refetchOnWindowFocus: false`.
- `console.*` is stripped from production builds — don't rely on it for anything user-facing, but
  `console.error` in catch blocks is the established pattern.

## Working in this repo

1. Branch off `main`; the designated branch for this session is set by the harness.
2. Commit messages are short, imperative, sentence-case subjects
   (`Fix misaligned Fighter's List radio button on mobile`). Conventional-commit prefixes
   (`fix(battle-session): …`, `chore(db): …`, `refactor: …`) are common but not mandatory.
3. Before finishing: `npm run lint` and `npm run build`.
4. `git push -u origin <branch>`. Only open a PR when explicitly asked. Vercel previews are built by
   `preview.yml` for non-draft PRs from repo members; `main` deploys to production.

### Checklist when adding a feature that touches gang/fighter data

- [ ] Read path: granular fetcher in `app/lib/**` wrapped in `unstable_cache` with a `CACHE_TAGS` tag
- [ ] Write path: Server Action in `app/actions/**` returning `{ success, data?, error? }`
- [ ] Financials updated through `updateGangFinancials` if credits/rating/wealth move
- [ ] Activity log written via `app/actions/logs/**`
- [ ] Cache invalidated via the matching `invalidate*` helper (every affected page, including the
      owner's caches for exotic beasts and the gang list)
- [ ] Permissions checked server-side, not just hidden in the UI
- [ ] Types added/updated in `types/**`
- [ ] Any schema change added as a `supabase/migrations/*.sql` file, and any RPC change mirrored into
      `supabase/functions/*.sql`
