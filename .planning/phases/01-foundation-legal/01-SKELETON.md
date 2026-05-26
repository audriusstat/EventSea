# Walking Skeleton — Event Sea

**Phase:** 1
**Generated:** 2026-05-26

## Capability Proven End-to-End

A visitor can load the `/privacy` page served by Next.js 16, that page reads the live `events` table through `@eventsea/db` (proving the Postgres schema is migrated and queryable), the BullMQ worker boots against Redis and accepts a dummy job, and GitHub Actions CI runs `turbo typecheck lint test` green — proving scaffold + routing + real DB write + real UI page + queue + CI/CD all work together before any feature code exists.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Framework | Next.js 16.2.6 + TypeScript 6 (strict) | D-01/D-02 locked; latest stable, App Router required for all Phase 1+ code. All dynamic APIs (`params`, `cookies()`, `headers()`) are async in v16 — always await. |
| Monorepo | Turborepo 2.9.14 + pnpm 9 workspaces | D-01 locked. `turbo.json` MUST use `tasks` key (v2), NOT `pipeline` — `pipeline` silently runs 0 tasks. Vercel auto-detects the Turborepo root. |
| Data layer | Postgres 16 + Drizzle ORM 0.45.2 / drizzle-kit 0.31.10 + postgres.js 3.4.9 | D-03 locked; thin SQL-close ORM per CLAUDE.md. Schema in `packages/db/src/schema/`, migrations in `packages/db/drizzle/`. `events.id` is `text()` (URL-hash-derived, DATA-04), not integer identity. |
| Queue | BullMQ 5.77.3 + ioredis 5.10.1 + Redis 7 | D-02 locked. ioredis MUST be created with `maxRetriesPerRequest: null` or worker crashes on boot (BullMQ blocking commands). One Queue per source for independent failure domains (DATA-03). |
| Migration strategy | `drizzle-kit push` local dev only; `drizzle-kit generate` + `drizzle-kit migrate` in CI/prod | D-07 locked. `push` drops/recreates differing columns — NEVER point it at Supabase. CI uses versioned `migrate` against the test DB. |
| Deployment | Vercel (apps/web, auto-deploy on main) + Fly.io fra region (apps/worker, flyctl in GitHub Actions) | D-04/D-06 locked. Worker is a long-running Node process incompatible with serverless (Playwright memory pressure, Phase 2). Fly.io fra is near Supabase Frankfurt. |
| Data residency | Supabase Frankfurt (eu-central-1) prod; Docker Compose Postgres local | D-10 locked. GDPR data residency in EEA — no transfer outside EU. |
| Local dev services | Docker Compose: postgres:16, redis:7, getmeili/meilisearch:v1.x | D-05/D-06 locked. Meilisearch present from Phase 1 to avoid config drift, but NO app code touches it until Phase 3. |
| Directory layout | `apps/web`, `apps/worker`, `packages/db` | D-02 locked. No `packages/shared` yet — added only when real shared non-DB code exists. Both apps import `@eventsea/db` via `workspace:*`. |
| Cookie consent | vanilla-cookieconsent 3.1.0 with a `"use client"` wrapper | D-13 locked. `@vanilla-cookieconsent/react` does NOT exist on npm — use the base package directly. Analytics fires only inside `onConsent` after `acceptedCategory('analytics')`. |
| Test framework | Vitest 4.1.7, per-package configs | CLAUDE.md specifies Vitest. DB integration tests use a non-cacheable `test:integration` turbo task to avoid stale cached passes. |

## Stack Touched in Phase 1
- [ ] Project scaffold (Turborepo + pnpm, ESLint, Prettier, TypeScript strict)
- [ ] Routing — `/privacy` route live in `apps/web`
- [ ] Database — `drizzle-kit migrate` creates events + price_history + users + watchlist + alert_log + organizers + sources tables
- [ ] UI — `/privacy` page renders static privacy policy text (SSR, no JS required); home page reads `events` table
- [ ] Deployment — GitHub Actions CI passes (`turbo typecheck lint test`); Vercel preview deploy works; Fly.io worker deploy workflow defined
- [ ] Queue — BullMQ worker starts, connects to Redis, accepts a dummy job

## Out of Scope (Deferred)
- User authentication (Phase 4) — `users` table exists but no auth flow
- Actual scraper logic (Phase 2) — worker is a skeleton; tiketa.lt scraper BLOCKED on legal gate
- Event catalog UI, search, filters (Phase 3) — only `/privacy` and a minimal home page
- Meilisearch indexing/querying (Phase 3) — present in Docker Compose only
- Price history display (Phase 5)
- Notification sends (Phase 5) — `alert_log` table exists, no delivery
- B2B organizer dashboard (Phase 7) — `organizers` table exists, no UI
- LT IP lawyer review of legal memo (pre-Phase 2 prerequisite, not a Phase 1 blocker)

## Subsequent Slice Plan
- Phase 2: Events appear in the database from real bilietai.lt + tiketa.lt scrapers (tiketa gated on legal clearance)
- Phase 3: Any visitor can browse all events in a public catalog with search and filters
- Phase 4: Users register, log in, and build a watchlist
- Phase 5: Users see price history and receive email/Telegram alerts
- Phase 6: Users add events to their calendar via .ics
- Phase 7: Organizers claim events, pay for featured listings, see analytics
