# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

SaaS website builder (Wix/WordPress.com-like) for local businesses. Core flow: signup → create site → pick industry template → editor → publish to subdomain. Full spec (Spanish): `instruccion_codex_website_builder.md`.

## Commands

- `npm run dev` — dev server (Turbopack), http://localhost:3000
- `npm run build` — production build
- `npm run lint` — ESLint
- No test framework set up yet.

## MVP scope (do not exceed)

- Single-page landing sites only (`/` per site), Hero/About/Services/Gallery/Contact sections.
- Components: text, image, button, divider, socialLinks, hours, map.
- Grid + reorder only — **no free XY drag placement** (dnd-kit sortable, sections + children within sections).
- Draft/published versioning only, no full history.

## Architecture

Next.js 15 App Router + TypeScript + Tailwind v4, Zustand editor state, dnd-kit, Zod, Supabase.

### Dual data mode (key design)

`src/lib/data/mode.ts`: `DATA_MODE=supabase` + Supabase env vars → Supabase; otherwise **local mode** (default). `getRepo()` in `src/lib/data/repo.ts` returns the singleton `Repo` implementation:
- `src/lib/data/local/` — JSON file store at `data/db.json` (gitignored), for dev without Supabase. Local auth: custom password hash + session cookie (`src/lib/auth/`), secret in `data/auth-secret`.
- `src/lib/data/supabase/` — Supabase Postgres + Auth. Schema/RLS in `supabase/migrations/0001_init.sql`.

`Repo` interface in `src/lib/data/types.ts` is the contract — any data feature goes through it, implemented in both backends. `src/lib/auth/service.ts` branches on mode the same way.

### Host-based public routing

`src/middleware.ts`: requests to root domain (`NEXT_PUBLIC_ROOT_DOMAIN`, default `localhost:3000`) serve the app; `sub.ROOT` or a custom domain rewrites to `/public/[siteKey]` (`src/app/public/[siteKey]/page.tsx`), which resolves the published version via `resolvePublished(siteKey)` and renders with `src/components/render/`. In-memory cache with 60s TTL in `src/lib/publicCache.ts`, invalidated on publish.

### Editor document (PageDoc)

`src/lib/doc/types.ts` — `{ schemaVersion, theme, sections[] }`. Sections: 12-col grid (mobile 4-col), `children[]` components. Nodes carry `layout.colSpan` per breakpoint + `props`. **Bump `SCHEMA_VERSION` and add a migration in `normalizeDoc` (`schema.ts`) on any shape change.** Zod validation in `schema.ts`; presets/templates in `presets.ts` / `templates.ts`.

### Editor state

`src/components/editor/store.ts` — Zustand store: doc, selection, breakpoint, undo/redo (snapshot history, limit 50), save state. All doc mutations go through `mutateDoc` (pushes history). UI: `EditorShell.tsx` (page shell), `Toolbar.tsx`, `Palette.tsx` (component picker), `Canvas.tsx` (dnd-kit sortable grid), `Inspector.tsx` (props panel). `useAutosave.ts` debounces draft saves; `useEditorHotkeys.ts` wires undo/redo/save shortcuts. Route: `src/app/app/editor/[siteId]/page.tsx`.

### Server actions

`src/lib/actions/` — auth, sites (create/save draft/publish/delete), assets. Mutations happen here, validated with Zod, ownership-checked.

### Publish flow

`Repo.publish(siteId, ownerUserId)` copies draft into new `page_versions` row with `is_published=true`, flips prior versions false, invalidates public cache.

### Uploads

`src/app/api/uploads/[...path]/route.ts` serves local-mode uploads; Supabase mode uses Storage under `sites/{site_id}/...`.

## Multi-tenant security (must hold)

- Every repo method takes/validates `ownerUserId` or site ownership; Supabase side also enforced by RLS.
- No free-form HTML rendering — text renders as text. Map component: only Google Maps embed URLs become iframes.
- Rate limiting on auth (`src/lib/auth/ratelimit.ts`).

## Constraints

- No features outside MVP scope. Prefer simple, robust implementations.
- Path alias: `@/*` → `src/*`.
