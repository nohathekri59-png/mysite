# SmartServe Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   └── api-server/         # Express API server
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts (single workspace package)
│   └── src/                # Individual .ts scripts, run via `pnpm --filter @workspace/scripts run <script>`
├── pnpm-workspace.yaml     # pnpm workspace (artifacts/*, lib/*, lib/integrations/*, scripts)
├── tsconfig.base.json      # Shared TS options (composite, bundler resolution, es2022)
├── tsconfig.json           # Root TS project references
└── package.json            # Root package with hoisted devDeps
```

## TypeScript & Composite Projects

Every package extends `tsconfig.base.json` which sets `composite: true`. The root `tsconfig.json` lists all packages as project references. This means:

- **Always typecheck from the root** — run `pnpm run typecheck` (which runs `tsc --build --emitDeclarationOnly`). This builds the full dependency graph so that cross-package imports resolve correctly. Running `tsc` inside a single package will fail if its dependencies haven't been built yet.
- **`emitDeclarationOnly`** — we only emit `.d.ts` files during typecheck; actual JS bundling is handled by esbuild/tsx/vite...etc, not `tsc`.
- **Project references** — when package A depends on package B, A's `tsconfig.json` must list B in its `references` array. `tsc --build` uses this to determine build order and skip up-to-date packages.

## Root Scripts

- `pnpm run build` — runs `typecheck` first, then recursively runs `build` in all packages that define it
- `pnpm run typecheck` — runs `tsc --build --emitDeclarationOnly` using project references

## Packages

### `artifacts/api-server` (`@workspace/api-server`)

Express 5 API server. Routes live in `src/routes/` and use `@workspace/api-zod` for request and response validation and `@workspace/db` for persistence.

- Entry: `src/index.ts` — reads `PORT`, starts Express
- App setup: `src/app.ts` — mounts CORS, JSON/urlencoded parsing, routes at `/api`
- Routes: `src/routes/index.ts` mounts sub-routers; `src/routes/health.ts` exposes `GET /health` (full path: `/api/health`)
- Depends on: `@workspace/db`, `@workspace/api-zod`
- `pnpm --filter @workspace/api-server run dev` — run the dev server
- `pnpm --filter @workspace/api-server run build` — production esbuild bundle (`dist/index.cjs`)
- Build bundles an allowlist of deps (express, cors, pg, drizzle-orm, zod, etc.) and externalizes the rest

### `lib/db` (`@workspace/db`)

Database layer using Drizzle ORM with PostgreSQL. Exports a Drizzle client instance and schema models.

- `src/index.ts` — creates a `Pool` + Drizzle instance, exports schema
- `src/schema/index.ts` — barrel re-export of all models
- `src/schema/<modelname>.ts` — table definitions with `drizzle-zod` insert schemas (no models definitions exist right now)
- `drizzle.config.ts` — Drizzle Kit config (requires `DATABASE_URL`, automatically provided by Replit)
- Exports: `.` (pool, db, schema), `./schema` (schema only)

Production migrations are handled by Replit when publishing. In development, we just use `pnpm --filter @workspace/db run push`, and we fallback to `pnpm --filter @workspace/db run push-force`.

### `lib/api-spec` (`@workspace/api-spec`)

Owns the OpenAPI 3.1 spec (`openapi.yaml`) and the Orval config (`orval.config.ts`). Running codegen produces output into two sibling packages:

1. `lib/api-client-react/src/generated/` — React Query hooks + fetch client
2. `lib/api-zod/src/generated/` — Zod schemas

Run codegen: `pnpm --filter @workspace/api-spec run codegen`

### `lib/api-zod` (`@workspace/api-zod`)

Generated Zod schemas from the OpenAPI spec (e.g. `HealthCheckResponse`). Used by `api-server` for response validation.

### `lib/api-client-react` (`@workspace/api-client-react`)

Generated React Query hooks and fetch client from the OpenAPI spec (e.g. `useHealthCheck`, `healthCheck`).

### `scripts` (`@workspace/scripts`)

Utility scripts package. Each script is a `.ts` file in `src/` with a corresponding npm script in `package.json`. Run scripts via `pnpm --filter @workspace/scripts run <script>`. Scripts can import any workspace package (e.g., `@workspace/db`) by adding it as a dependency in `scripts/package.json`.

### `artifacts/smartserve` (`@workspace/smartserve`)

React + Vite PWA — a luxury "Browse-First" presentation app for SmartServe digital services agency.

- **Theme**: Deep charcoal/black background (`hsl 0 0% 4%`) with premium gold accents `#d4af37` (`hsl 46 65% 52%`). Configured in `src/index.css` via CSS variables.
- **Fonts**: Outfit (sans), Playfair Display (display headings), Tajawal (Arabic). Loaded from Google Fonts.
- **PWA**: `public/manifest.webmanifest` + `public/sw.js` (network-first for navigation, cache-first for assets). Service worker is registered in `src/main.tsx` only in production (`import.meta.env.PROD`) so dev HMR is unaffected. Apple/Android meta tags live in `index.html`.
- **Bilingual (EN ↔ AR with RTL)**: `src/lib/i18n.tsx` exposes `<LanguageProvider>` (wraps the app in `App.tsx`), the `useT()` hook for inline `t("English", "العربية")` translations, `useLang()` for reading `lang` / `dir`, and `<LangToggle />` mounted in the header. The provider sets `document.documentElement.lang` and `dir` and persists the choice in `localStorage` as `smartserve_lang`. All page text — services, pricing tiers, industries, FAQs, testimonials, footer — is translated via the same hook. Floating WhatsApp button auto-mirrors to the bottom-left in RTL.
- **Architecture**: Single-page browse-first design in `src/pages/Home.tsx`:
  - Hero with system-status indicator, countdown timer, "Browse Services" + "Download Project Profile" CTAs
  - Services grid (5 cards): Educational Suite, Smart Accounting (tiered), Enterprise Platform, Digital Menu — Basic, Interactive Menu
  - `<ServiceModal>` (mini-dashboard) — description, optional pricing tiers (for Smart Accounting), industry dropdown (Schools/Hospitals/Clinics/Cafes/Retail) that dynamically updates the features list, gold-glow trust badges (24/7 Support, Free Trial), Quick FAQ accordion, dual CTAs: "اطلب الآن · Get Started" + "تواصل واتساب · WhatsApp"
  - `<RegistrationModal>` opens from Get Started — uses `useSignUp` from `@workspace/api-client-react` + react-hook-form + zod. After successful signup, displays the `<PaymentInfo>` block with WhatsApp, InstaPay (`01276387146`), and Vodafone Cash (`01002081026`) numbers + copy buttons.
  - Trust strip + Success Stories testimonials section + shared footer.
  - `<FloatingWhatsApp>` — sticky bottom-right green button (always visible) that opens `wa.me/201276387146`.
- **PDF generation**: `jspdf` package — `downloadProjectProfile()` generates the Project Profile PDF on-demand client-side.
- **Footer**: `src/components/SharedFooter.tsx` renders `All Rights Reserved © 2026 — إعداد نهى ذكري` on every page.
- **Auth**: Optional. Browsing is open; signup is only required at "Get Started". JWT stored in `localStorage` as `smartserve_token`.
- **Countdown timer**: Persists 7-day window in `localStorage` as `smartserve_offer_end` so it stays consistent across reloads.
