# Phase 1: Foundation + Legal — Research

**Researched:** 2026-05-25
**Domain:** Monorepo scaffolding, PostgreSQL schema design, GDPR compliance, scraping legal strategy
**Confidence:** HIGH (core stack verified against npm registry and official docs)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Use Turborepo + pnpm workspaces as the monorepo tool.
- **D-02:** Three packages in Phase 1: `apps/web` (Next.js 16), `apps/worker` (Fly.io BullMQ), `packages/db` (Drizzle schema + migrations + client). No `packages/shared` yet.
- **D-03:** Drizzle schema in `packages/db/src/schema/`; generated migrations in `packages/db/drizzle/`. Both apps import from `packages/db`.
- **D-04:** CI/CD: GitHub Actions for typecheck/lint/test on every PR. Vercel handles Next.js deploy automatically. Fly.io worker deployed via `flyctl` in a separate GitHub Actions job.
- **D-05:** Docker Compose with `postgres:16`, `redis:7`, `getmeili/meilisearch:v1.x` — fully offline-capable.
- **D-06:** Meilisearch included in Docker Compose from Phase 1 (to avoid config drift when Phase 3 wires search).
- **D-07:** `drizzle-kit push` for local dev; `drizzle-kit migrate` in CI and production (versioned SQL).
- **D-08:** Worker runs locally via `pnpm --filter worker dev` (tsx watch mode). No Dockerfile for local dev.
- **D-09:** `price_history` is append-only — zero UPDATE operations on price columns, ever.
- **D-10:** Supabase Frankfurt (eu-central-1) for GDPR data residency.
- **D-11:** Event ID = URL hash (DATA-04). Stable across scrapes.
- **D-12:** GDPR cascade delete wired in Phase 1 schema (GDPR-04).
- **D-13:** Cookie consent via pre-built library (e.g., vanilla-cookieconsent). Analytics fires only after consent.
- **D-14:** Legal memo = self-authored best-effort (robots.txt analysis + EU Directive 96/9/EC + outreach plan). LT IP lawyer review is pre-Phase 2 prerequisite, not Phase 1 blocker.

### Claude's Discretion
- Cookie consent library selection (vanilla-cookieconsent or similar)
- GitHub Actions workflow YAML structure
- `.env.example` variable naming
- Exact Fly.io app configuration (`fly.toml`)

### Deferred Ideas (OUT OF SCOPE)
- None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DATA-03 | Scrapers operate independently — one failure does not block others | BullMQ Queue-per-scraper isolation pattern; sources table circuit breaker state |
| DATA-04 | Every event has a stable unique ID generated from source URL hash | `crypto.createHash('sha256').update(url).digest('hex').slice(0, 16)` — stable across scrapes |
| DATA-06 | Scrapers respect robots.txt and rate limits | bilietai.lt robots.txt verified: only /search and /beta/ blocked for `*`; tiketa.lt robots.txt verified: `Disallow: /` for `*` — tiketa requires legal memo |
| DATA-07 | Each scraper stores `last_successfully_scraped_at`; source marked degraded on failure | `sources` table with `last_successfully_scraped_at`, `consecutive_failures`, `status` enum |
| GDPR-01 | Privacy policy accessible on all pages | `/privacy` route in Next.js App Router; link in global layout footer |
| GDPR-02 | Cookie consent banner before any tracking | vanilla-cookieconsent v3 with `"use client"` wrapper; analytics scripts gated on `acceptedCategory('analytics')` |
| GDPR-03 | Article 30 processing register created and maintained | Markdown/spreadsheet ROPA document; 8 mandatory controller fields |
| GDPR-04 | Cascade delete — deleting account deletes all associated data | `onDelete: 'cascade'` on all FK references to `users.id` in Drizzle schema |
</phase_requirements>

---

## Summary

Phase 1 establishes the complete technical and legal foundation for Event Sea before a single user signs up or a scraper fires. The work falls into three distinct tracks: (1) monorepo scaffolding (Turborepo 2 + pnpm, three packages, CI/CD), (2) GDPR-compliant PostgreSQL schema with cascade-delete wiring, and (3) a scraping legal strategy document covering robots.txt findings, EU Database Directive analysis, and a partner outreach plan.

The tech stack is fully verified against the npm registry. Next.js is on version 16.2.6 (latest, confirmed stable), Turborepo on 2.9.14, Drizzle ORM on 0.45.2/drizzle-kit 0.31.10, and BullMQ on 5.77.3. All packages have verified source repositories from well-known maintainers (Vercel, drizzle-team, taskforcesh) with no suspicious postinstall scripts.

The most critical legal finding: **tiketa.lt has `Disallow: /` for all crawlers** in its robots.txt. This makes unsolicited scraping legally high-risk and partnership/API-first is the only clean path. bilietai.lt blocks only `/search` and `/beta/` — general event listing pages appear crawlable. Both sources need documented outreach emails before Phase 2 scrapers fire.

**Primary recommendation:** Scaffold monorepo with `pnpm dlx create-turbo@latest`, then create packages manually. Use `uuid().defaultRandom()` (not integer identity) for `events.id` so url-hash-derived IDs can be deterministic strings. Wire all cascade deletes in schema before any migrations run against Supabase.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| GDPR cascade delete | Database / Storage | — | Schema-level constraint; cannot be app-enforced reliably |
| Cookie consent gate | Browser / Client | — | Runs in browser before any tracking script fires |
| Privacy policy page | Frontend Server (SSR) | — | Static page; SSR for SEO and no-JS accessibility |
| Article 30 ROPA | — (document, not code) | — | Markdown/spreadsheet artifact, not a system component |
| BullMQ job queue | API / Backend (Worker) | — | Long-running Node.js process on Fly.io; not serverless |
| Drizzle schema + migrations | Database / Storage | — | Source of truth for all table definitions and constraints |
| Event ID generation | API / Backend (Worker) | — | Hash computed at scrape time; stored as stable text field |
| Source circuit breaker state | Database / Storage | API / Backend | `sources` table tracked by worker; read by future monitoring |
| CI/CD pipeline | — (infra, not runtime tier) | — | GitHub Actions orchestrates build, test, deploy |

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `next` | 16.2.6 | SSR framework, App Router, PWA shell | Locked decision; latest stable on npm [VERIFIED: npm registry] |
| `turbo` | 2.9.14 | Monorepo task orchestration + caching | Locked decision; Vercel-maintained [VERIFIED: npm registry] |
| `drizzle-orm` | 0.45.2 | Type-safe ORM for PostgreSQL | Thin, SQL-close ORM matching CLAUDE.md preference [VERIFIED: npm registry] |
| `drizzle-kit` | 0.31.10 | Schema migration tooling (push/migrate) | Required companion to drizzle-orm [VERIFIED: npm registry] |
| `bullmq` | 5.77.3 | Redis-backed job queue for scraper worker | Locked decision; taskforcesh/bullmq repo [VERIFIED: npm registry] |
| `typescript` | 6.0.3 | Strict TypeScript — required by CLAUDE.md | Latest stable, Microsoft-maintained [VERIFIED: npm registry] |
| `tsx` | 4.22.3 | TypeScript execution for worker dev mode | Enables `pnpm --filter worker dev` without compiled output [VERIFIED: npm registry] |
| `vanilla-cookieconsent` | 3.1.0 | GDPR cookie consent banner (D-13) | Standalone, zero-dependency, orestbida/cookieconsent repo [VERIFIED: npm registry] |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `postgres` | 3.4.9 | PostgreSQL driver (postgres.js) | Recommended by Drizzle docs for serverless/edge [VERIFIED: npm registry] |
| `ioredis` | 5.10.1 | Redis client for BullMQ worker | Required by BullMQ with `maxRetriesPerRequest: null` [VERIFIED: npm registry] |
| `vitest` | 4.1.7 | Unit testing framework | CLAUDE.md specifies Vitest; vitest-dev/vitest repo [VERIFIED: npm registry] |
| `pg` | 8.21.0 | Fallback PostgreSQL driver (node-postgres) | Alternative to postgres.js; use for Supabase connection pooling [VERIFIED: npm registry] |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `postgres` (postgres.js) | `pg` (node-postgres) | postgres.js is faster for serverless; pg has better Supabase transaction pooler support. Use postgres.js for apps/web (Vercel edge), pg for worker (long-running) |
| `vanilla-cookieconsent` | `@consent-manager/core`, `cookiehub` | vanilla-cookieconsent is zero-dependency and free; alternatives add bundle weight or subscription cost |
| integer identity PK | uuid PK for events | uuid allows deterministic ID generation from URL hash at application layer without DB round-trip |

**Installation (root workspace):**
```bash
pnpm add -w turbo typescript
```

**Installation (apps/web):**
```bash
pnpm --filter web add next react react-dom vanilla-cookieconsent
pnpm --filter web add -D @types/react @types/react-dom vitest
```

**Installation (apps/worker):**
```bash
pnpm --filter worker add bullmq ioredis
pnpm --filter worker add -D tsx typescript vitest
```

**Installation (packages/db):**
```bash
pnpm --filter db add drizzle-orm postgres
pnpm --filter db add -D drizzle-kit
```

---

## Package Legitimacy Audit

> slopcheck was run but defaulted to PyPI (Python ecosystem). All packages below were verified on the npm registry manually with `npm view <pkg> version` and `npm view <pkg> repository.url`. No suspicious postinstall scripts found.

| Package | Registry | Source Repo | postinstall | slopcheck | Disposition |
|---------|----------|-------------|-------------|-----------|-------------|
| `turbo` | npm | github.com/vercel/turborepo | none | N/A (npm) | Approved |
| `next` | npm | github.com/vercel/next.js | none | N/A (npm) | Approved |
| `drizzle-orm` | npm | github.com/drizzle-team/drizzle-orm | none | N/A (npm) | Approved |
| `drizzle-kit` | npm | github.com/drizzle-team/drizzle-orm | none | N/A (npm) | Approved |
| `bullmq` | npm | github.com/taskforcesh/bullmq | none | N/A (npm) | Approved |
| `vanilla-cookieconsent` | npm | github.com/orestbida/cookieconsent | none | N/A (npm) | Approved |
| `tsx` | npm | github.com/privatenumber/tsx | none | N/A (npm) | Approved |
| `typescript` | npm | github.com/microsoft/TypeScript | none | N/A (npm) | Approved |
| `vitest` | npm | github.com/vitest-dev/vitest | none | N/A (npm) | Approved |
| `postgres` | npm | (not checked — common package) | none | N/A (npm) | Approved [ASSUMED] |
| `ioredis` | npm | (not checked — common package) | none | N/A (npm) | Approved [ASSUMED] |

**Packages removed due to [SLOP]:** none

**Packages flagged [SUS]:** none

**Note on `@vanilla-cookieconsent/react`:** This scoped package does **not exist** on npm (verified with `npm view @vanilla-cookieconsent/react` returning "not found"). Do not use it. Use `vanilla-cookieconsent` directly with a `"use client"` wrapper component.

*slopcheck could not be run against npm ecosystem (tool defaults to PyPI). All packages verified manually via npm registry as authoritative source.*

---

## Architecture Patterns

### System Architecture Diagram

```
Developer Machine
      │
      ▼
Docker Compose ──────────────────────────────────────┐
  postgres:16   ◄─── drizzle-kit push (schema sync)  │
  redis:7       ◄─── BullMQ Worker (local dev)        │
  meilisearch   ◄─── Phase 3 (search index, not wired │
                      in Phase 1)                      │
                                                       │
pnpm --filter worker dev                               │
  apps/worker (tsx watch) ──► Redis ──► queue skeleton │
                          └──► Postgres (packages/db)  │
                                                       │
pnpm --filter web dev                                  │
  apps/web (Next.js 16) ──► Postgres (packages/db)    │
  /privacy route (static)                              │
  cookie consent banner                                │
                                                       │
GitHub Push ──► GitHub Actions CI                      │
  turbo typecheck lint test ──► pass/fail              │
  flyctl deploy ──────────────► Fly.io (worker)        │
  Vercel integration ─────────► Vercel (web)           │
                                                       │
Prod                                                   │
  Vercel (apps/web) ──► Supabase Frankfurt (postgres) ─┘
  Fly.io (apps/worker) ──► Upstash Redis
```

### Recommended Project Structure

```
eventsea/                          # repo root
├── turbo.json                     # Turborepo task config
├── pnpm-workspace.yaml            # declares apps/* and packages/*
├── package.json                   # root devDeps: turbo, typescript
├── tsconfig.base.json             # shared TS config
├── docker-compose.yml             # postgres:16, redis:7, meilisearch
├── .env.example                   # documented env vars
├── .github/
│   └── workflows/
│       ├── ci.yml                 # typecheck, lint, test on PR
│       └── deploy-worker.yml      # flyctl deploy on main merge
├── apps/
│   ├── web/                       # Next.js 16 App Router
│   │   ├── package.json
│   │   ├── next.config.ts
│   │   ├── app/
│   │   │   ├── layout.tsx         # global layout with cookie consent
│   │   │   ├── page.tsx           # walking skeleton home page
│   │   │   └── privacy/
│   │   │       └── page.tsx       # GDPR-01: /privacy route
│   │   └── components/
│   │       └── CookieConsent.tsx  # "use client" wrapper (GDPR-02)
│   └── worker/
│       ├── package.json
│       ├── fly.toml               # Fly.io config
│       ├── Dockerfile             # for Fly.io deploy
│       └── src/
│           ├── index.ts           # BullMQ worker entry
│           └── queues/
│               └── scraper.ts     # queue skeleton (no scrape logic)
└── packages/
    └── db/
        ├── package.json
        ├── drizzle.config.ts      # points to schema/, out: drizzle/
        ├── src/
        │   ├── index.ts           # exports db client + schema
        │   └── schema/
        │       ├── events.ts
        │       ├── price-history.ts
        │       ├── users.ts
        │       ├── watchlist.ts
        │       ├── alert-log.ts
        │       ├── organizers.ts
        │       └── sources.ts
        └── drizzle/               # generated SQL migration files
```

### Pattern 1: Turborepo v2 turbo.json

`pipeline` key is removed in Turborepo v2. Use `tasks` instead. [VERIFIED: turborepo.dev/docs/reference/configuration]

```json
{
  "$schema": "https://turborepo.dev/schema.json",
  "globalDependencies": ["tsconfig.base.json"],
  "globalEnv": ["NODE_ENV", "DATABASE_URL", "REDIS_URL"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "test": {
      "outputs": ["coverage/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

### Pattern 2: pnpm-workspace.yaml

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### Pattern 3: Drizzle Schema with GDPR Cascade Deletes

Use `uuid().defaultRandom()` for `events.id` so the application can supply a deterministic hash-derived ID. For `users` and internal tables, use integer identity. [VERIFIED: orm.drizzle.team/docs/column-types/pg]

```typescript
// packages/db/src/schema/users.ts
import { pgTable, integer, text, varchar, timestamp, boolean } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  email: varchar({ length: 255 }).notNull().unique(),
  createdAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
  deletedAt: timestamp({ withTimezone: true }),  // soft-delete tombstone
});

// packages/db/src/schema/events.ts
import { pgTable, text, varchar, timestamp, boolean, numeric, index } from 'drizzle-orm/pg-core';

export const events = pgTable('events', {
  id: text().primaryKey(),                          // url_hash — DATA-04
  sourceId: integer().notNull().references(() => sources.id),
  title: text().notNull(),
  startsAt: timestamp({ withTimezone: true }).notNull(),
  endsAt: timestamp({ withTimezone: true }),
  venue: text(),
  city: varchar({ length: 100 }),
  category: varchar({ length: 100 }),
  minPrice: numeric({ precision: 10, scale: 2 }),
  currency: varchar({ length: 3 }).default('EUR'),
  ticketUrl: text().notNull(),
  status: varchar({ length: 50 }).default('active'),  // active | pre_sale | sold_out | stale
  featured: boolean().default(false),
  lastSeenAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
  createdAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
}, (table) => [
  index('events_city_idx').on(table.city),
  index('events_starts_at_idx').on(table.startsAt),
  index('events_source_id_idx').on(table.sourceId),
]);

// packages/db/src/schema/price-history.ts — APPEND ONLY (D-09)
export const priceHistory = pgTable('price_history', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  eventId: text().notNull().references(() => events.id),
  price: numeric({ precision: 10, scale: 2 }).notNull(),
  currency: varchar({ length: 3 }).default('EUR'),
  observedAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
});

// packages/db/src/schema/watchlist.ts — cascade delete on user (GDPR-04)
export const watchlist = pgTable('watchlist', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  userId: integer().notNull().references(() => users.id, { onDelete: 'cascade' }),
  eventId: text().notNull().references(() => events.id),
  watchType: varchar({ length: 50 }).notNull(),  // 'pre_sale' | 'price_drop'
  priceThreshold: numeric({ precision: 10, scale: 2 }),
  createdAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
});

// packages/db/src/schema/alert-log.ts — cascade delete on user (GDPR-04)
export const alertLog = pgTable('alert_log', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  userId: integer().notNull().references(() => users.id, { onDelete: 'cascade' }),
  eventId: text().notNull().references(() => events.id),
  alertType: varchar({ length: 50 }).notNull(),
  sentAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
  channel: varchar({ length: 50 }).notNull(),  // 'email' | 'telegram'
});

// packages/db/src/schema/organizers.ts — separate from users (B2B-01)
export const organizers = pgTable('organizers', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: text().notNull(),
  email: varchar({ length: 255 }).notNull().unique(),
  createdAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
});

// packages/db/src/schema/sources.ts — circuit breaker state (DATA-07)
export const sources = pgTable('sources', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: varchar({ length: 100 }).notNull().unique(),  // 'bilietai.lt' | 'tiketa.lt'
  baseUrl: text().notNull(),
  status: varchar({ length: 50 }).default('active'),  // active | degraded | disabled
  consecutiveFailures: integer().default(0).notNull(),
  lastSuccessfullyScrapedAt: timestamp({ withTimezone: true }),
  createdAt: timestamp({ withTimezone: true }).defaultNow().notNull(),
});
```

### Pattern 4: drizzle.config.ts (packages/db)

```typescript
// packages/db/drizzle.config.ts
import 'dotenv/config';
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/schema/index.ts',  // barrel file exporting all tables
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

Local dev push: `pnpm --filter db drizzle-kit push`
Prod migration: `pnpm --filter db drizzle-kit generate && pnpm --filter db drizzle-kit migrate`

### Pattern 5: Docker Compose (local dev)

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: eventsea
      POSTGRES_PASSWORD: eventsea
      POSTGRES_DB: eventsea
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U eventsea -d eventsea"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  meilisearch:
    image: getmeili/meilisearch:v1.13.3
    environment:
      MEILI_MASTER_KEY: local-dev-master-key
      MEILI_ENV: development
    ports:
      - "7700:7700"
    volumes:
      - meilisearch_data:/meili_data

volumes:
  postgres_data:
  redis_data:
  meilisearch_data:
```

**Meilisearch version note:** `v1.x` was specified in decisions. Latest stable release tag must be verified at Docker Hub before pinning. `v1.13.3` is a placeholder — confirm latest v1.x at hub.docker.com/r/getmeili/meilisearch/tags. [ASSUMED]

### Pattern 6: BullMQ v5 Worker Skeleton

```typescript
// apps/worker/src/index.ts
import IORedis from 'ioredis';
import { Worker, Queue } from 'bullmq';

const connection = new IORedis(process.env.REDIS_URL!, {
  maxRetriesPerRequest: null,  // required for BullMQ
});

// One queue per source (DATA-03: independent failure domains)
const bilietaiQueue = new Queue('scraper:bilietai', { connection });
const tiketaQueue = new Queue('scraper:tiketa', { connection });

const worker = new Worker(
  'scraper:bilietai',
  async (job) => {
    // Phase 2 fills this in. Phase 1 skeleton only.
    console.log(`[Phase 1 skeleton] Job received: ${job.name}`);
  },
  { connection }
);

worker.on('failed', (job, err) => {
  console.error(`Job ${job?.id} failed:`, err.message);
  // Phase 2: update sources.consecutive_failures, mark degraded after 3
});

console.log('Worker skeleton running. BullMQ connected to Redis.');
```

### Pattern 7: vanilla-cookieconsent v3 with Next.js App Router

The `"use client"` directive is required because cookieconsent manipulates the DOM. [CITED: cookieconsent.orestbida.com/essential/getting-started.html]

```typescript
// apps/web/components/CookieConsent.tsx
'use client';

import { useEffect } from 'react';
import 'vanilla-cookieconsent/dist/cookieconsent.css';
import CookieConsent from 'vanilla-cookieconsent';

export default function CookieConsentBanner() {
  useEffect(() => {
    CookieConsent.run({
      categories: {
        necessary: {
          enabled: true,
          readOnly: true,
        },
        analytics: {}, // user opt-in required
      },
      language: {
        default: 'lt',
        translations: {
          lt: {
            consentModal: {
              title: 'Naudojame slapukus',
              description: 'Naudojame būtinus slapukus sistemos veikimui ir analytics slapukus svetainės tobulinimui.',
              acceptAllBtn: 'Sutinku su visais',
              acceptNecessaryBtn: 'Tik būtini',
              showPreferencesBtn: 'Nustatymai',
            },
            preferencesModal: {
              title: 'Slapukų nustatymai',
              acceptAllBtn: 'Sutinku su visais',
              acceptNecessaryBtn: 'Tik būtini',
              savePreferencesBtn: 'Išsaugoti',
              sections: [
                {
                  title: 'Būtini slapukai',
                  description: 'Reikalingi svetainės veikimui.',
                  linkedCategory: 'necessary',
                },
                {
                  title: 'Analytics slapukai',
                  description: 'Padeda mums suprasti, kaip naudotojai naudoja svetainę.',
                  linkedCategory: 'analytics',
                },
              ],
            },
          },
        },
      },
      onConsent: () => {
        if (CookieConsent.acceptedCategory('analytics')) {
          // Fire analytics initialisation here (Phase 3)
        }
      },
      onChange: () => {
        if (CookieConsent.acceptedCategory('analytics')) {
          // Re-enable analytics if user updates consent
        }
      },
    });
  }, []);

  return null;
}

// apps/web/app/layout.tsx — import this in global layout
// import CookieConsentBanner from '@/components/CookieConsent';
// <CookieConsentBanner /> in body before closing tag
```

### Pattern 8: GitHub Actions CI (Turborepo-aware)

[VERIFIED: turborepo.dev/docs/guides/ci-vendors/github-actions]

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize]

jobs:
  ci:
    name: Typecheck, Lint, Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - uses: pnpm/action-setup@v3
        with:
          version: 9  # pin to match local pnpm version

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - run: pnpm turbo typecheck lint test
```

```yaml
# .github/workflows/deploy-worker.yml
name: Deploy Worker
on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy to Fly.io
    runs-on: ubuntu-latest
    concurrency: deploy-worker
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only --config apps/worker/fly.toml
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

### Pattern 9: Fly.io fly.toml (worker, no HTTP)

[CITED: fly.io/docs/reference/configuration/]

```toml
# apps/worker/fly.toml
app = "eventsea-worker"
primary_region = "fra"  # Frankfurt — near Supabase Frankfurt

[build]
dockerfile = "Dockerfile"

[processes]
worker = "node dist/index.js"

[[vm]]
size = "shared-cpu-1x"
memory = "512mb"
processes = ["worker"]
```

Worker Dockerfile skeleton:
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build
CMD ["node", "dist/index.js"]
```

### Pattern 10: Event ID Generation (DATA-04)

```typescript
// packages/db/src/utils/event-id.ts
import { createHash } from 'crypto';

export function generateEventId(sourceUrl: string): string {
  return createHash('sha256').update(sourceUrl).digest('hex').slice(0, 32);
}
```

### Anti-Patterns to Avoid

- **Using `pipeline` key in turbo.json:** Turborepo v2 renamed it to `tasks`. Using `pipeline` is treated as an unknown key and silently ignored, causing tasks to not run.
- **Importing `@vanilla-cookieconsent/react`:** This package does not exist on npm. Use `vanilla-cookieconsent` directly with a `"use client"` component.
- **Using synchronous `cookies()`, `headers()`, `params` in Next.js 16:** These APIs are fully async in v16. Always `await` them. The codemod `npx @next/codemod@canary upgrade latest` handles migration.
- **Using `middleware.ts` naming:** Deprecated in Next.js 16; rename to `proxy.ts` (though `middleware.ts` still works for edge runtime).
- **UPDATE on price_history:** Architecture constraint D-09 forbids it. Only INSERT allowed.
- **Missing `default.js` for parallel routes:** Next.js 16 requires explicit `default.js` in all parallel route slots — builds fail without them.
- **Meilisearch wired in Phase 1 app code:** D-06 requires Meilisearch in Docker Compose now, but no application code should index or query it until Phase 3.
- **BullMQ with `enableReadyCheck: true` (ioredis default):** BullMQ documentation recommends `maxRetriesPerRequest: null`. Upstash TLS connections need `tls: {}`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Job queue with Redis | Custom pub/sub queue | `bullmq` | Retry logic, dead letter queues, job dedup, cron scheduling — all edge cases |
| Cookie consent UI | Custom banner + consent storage | `vanilla-cookieconsent` | IAB TCF compliance, a11y, persistent consent cookie, per-category API |
| DB migrations | Custom SQL runner scripts | `drizzle-kit migrate` | Migration hash tracking, idempotency, rollback safety |
| GDPR cascade delete | Application-layer delete cascade | Postgres `ON DELETE CASCADE` | Race conditions and missed joins will cause GDPR violations; DB constraint is atomic |
| Event ID deduplication | Random UUID on insert | URL-hash derived text PK | Random ID breaks idempotent scraping; same event would insert multiple rows |
| TypeScript monorepo task caching | Shell scripts | `turbo` | Turborepo handles task graph, parallel execution, and cache invalidation correctly |

**Key insight:** The job queue and cascade delete patterns have subtle failure modes that cost weeks to debug; the ecosystem tools encode years of production learnings.

---

## Runtime State Inventory

> Greenfield project — no existing runtime state. Explicitly verified by checking that no services, databases, or configs exist yet.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None — database does not exist yet | None |
| Live service config | None — no running services | None |
| OS-registered state | None — no task scheduler or daemon entries | None |
| Secrets/env vars | None — `.env.local` does not exist yet | Create from `.env.example` |
| Build artifacts | None — no prior builds | None |

---

## Common Pitfalls

### Pitfall 1: turbo.json `pipeline` vs `tasks` key
**What goes wrong:** Developer uses v1 `pipeline` key in turbo.json. Turborepo v2 silently ignores it. All `turbo run` commands succeed but run 0 tasks — CI appears green while doing nothing.
**Why it happens:** Docs for older Turborepo versions are everywhere; the key rename happened in v2.
**How to avoid:** Use `"tasks"` key. Add `"$schema": "https://turborepo.dev/schema.json"` — editor will flag unknown keys.
**Warning signs:** `turbo run build` completes in milliseconds with 0 tasks cached.

### Pitfall 2: Next.js 16 async APIs in new project code
**What goes wrong:** New code written with synchronous `params`, `cookies()`, or `headers()` — compiles fine, crashes at runtime with "Error: Route used `cookies` which is not supported".
**Why it happens:** Next.js 16 fully removed synchronous access that v15 still allowed.
**How to avoid:** Always `await params`, `await cookies()`, `await headers()` in Server Components and Route Handlers. Use `export default async function Page({ params }:...)`.
**Warning signs:** `Error: Route ... used a synchronous API` in terminal.

### Pitfall 3: drizzle-kit push in CI destroys prod schema
**What goes wrong:** `drizzle-kit push` is used in CI/Vercel build commands. It drops and recreates columns that differ, potentially destroying production data.
**Why it happens:** `push` is designed for local dev only — it does not generate migration files and cannot roll back.
**How to avoid:** CI and production MUST use `drizzle-kit generate` + `drizzle-kit migrate`. Only use `push` against Docker Compose local postgres.
**Warning signs:** CI workflow calls `drizzle-kit push` with a `DATABASE_URL` pointing to Supabase.

### Pitfall 4: BullMQ connection without `maxRetriesPerRequest: null`
**What goes wrong:** BullMQ worker throws `ReplyError: LOADING Redis is loading the dataset in memory` and crashes on startup.
**Why it happens:** ioredis default `maxRetriesPerRequest` causes immediate failure on blocked commands. BullMQ uses blocking Redis commands that require unlimited retries.
**How to avoid:** Always create ioredis with `{ maxRetriesPerRequest: null }`.
**Warning signs:** Worker starts and immediately exits with ioredis errors.

### Pitfall 5: Cookie consent fires analytics before consent is granted
**What goes wrong:** Analytics script is imported at module level or in `_app.tsx` without checking consent. GDPR violation on first page load.
**Why it happens:** Standard Google Analytics or PostHog snippets load unconditionally.
**How to avoid:** Only initialise analytics inside `CookieConsent.onConsent` callback after `acceptedCategory('analytics')` returns true.
**Warning signs:** Browser DevTools shows analytics network requests before user clicks "Accept".

### Pitfall 6: tiketa.lt scraping without partnership agreement
**What goes wrong:** tiketa.lt robots.txt has `Disallow: /` for all crawlers. Scraping without consent violates their explicit machine-readable policy.
**Why it happens:** Developer sees event data in browser and assumes it's scrapable.
**How to avoid:** Legal memo must document this. Phase 2 cannot start tiketa.lt scraper without either (a) partnership/API agreement or (b) explicit legal opinion that the `Disallow: /` robots.txt directive does not create a legal bar under Lithuanian law.
**Warning signs:** tiketa.lt scraper in Phase 2 plan without a legal gate.

### Pitfall 7: Price history rows accidentally updated
**What goes wrong:** ORM query accidentally uses `.update()` on `price_history` rows. Destroys audit trail.
**Why it happens:** Developer assumes "update if exists, insert if new" is the right pattern.
**How to avoid:** Never write UPDATE on `price_history`. Always INSERT a new row. Add a comment `/* APPEND ONLY — see D-09 */` in the schema file.
**Warning signs:** Any Drizzle `.update()` call targeting `priceHistory` table.

### Pitfall 8: Turborepo caches test runs with side effects
**What goes wrong:** Vitest tests that interact with the database are cached by Turborepo. Subsequent runs serve cached "pass" result even when DB state has changed.
**Why it happens:** Turborepo caches based on input file hashes, not runtime state.
**How to avoid:** DB integration tests must NOT be in the `test` turbo task's cacheable scope. Use a separate `test:integration` task with `"cache": false`.
**Warning signs:** Test run shows "FULL TURBO" but the database hasn't been seeded.

---

## Code Examples

### Walking Skeleton DB Read in Next.js 16

```typescript
// apps/web/app/page.tsx
import { db } from '@eventsea/db';
import { events } from '@eventsea/db/schema';

export default async function HomePage() {
  const recentEvents = await db.select().from(events).limit(5);
  return (
    <main>
      <h1>Event Sea</h1>
      <p>Events in DB: {recentEvents.length}</p>
    </main>
  );
}
```

### packages/db index.ts export shape

```typescript
// packages/db/src/index.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema/index.js';

const client = postgres(process.env.DATABASE_URL!);
export const db = drizzle(client, { schema });
export * from './schema/index.js';
```

### packages/db package.json (internal workspace package)

```json
{
  "name": "@eventsea/db",
  "version": "0.0.1",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "scripts": {
    "db:push": "drizzle-kit push",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate"
  }
}
```

Root and app packages reference it with: `"@eventsea/db": "workspace:*"`

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `pipeline` in turbo.json | `tasks` in turbo.json | Turborepo v2 (2024) | Old config silently ignored |
| Sync `cookies()`, `params` in Next.js | Fully async APIs | Next.js 16 (Oct 2025) | Sync access throws at runtime |
| `experimental.turbopack` in next.config | Top-level `turbopack` | Next.js 16 | Old location still works but deprecated |
| `middleware.ts` | `proxy.ts` (nodejs runtime) | Next.js 16 | `middleware.ts` deprecated; edge only |
| `serial` / `bigserial` PostgreSQL columns | `generatedAlwaysAsIdentity()` | PostgreSQL 10+ / Drizzle 2024 | Identity columns are SQL standard |
| `pg` as default Drizzle driver | `postgres` (postgres.js) for edge | 2024 | postgres.js is faster for connection pooling |
| `next lint` command | ESLint CLI directly | Next.js 16 | `next lint` removed; use `eslint .` |

**Deprecated/outdated:**
- `pages/` directory: Not removed but App Router is the default and required for all Phase 1 code.
- `next/legacy/image`: Deprecated in Next.js 16. Use `next/image` only.
- `images.domains` config: Deprecated. Use `images.remotePatterns`.
- `serverRuntimeConfig` / `publicRuntimeConfig`: Removed in Next.js 16. Use env vars.

---

## Scraping Legal Analysis

### bilietai.lt robots.txt (verified 2026-05-25)

```
User-agent: *
Disallow: /search
Disallow: /beta/

User-agent: dotbot
Disallow: /
[... specific SEO bots blocked ...]

Sitemap: https://www.bilietai.lt/sitemap/index.xml
```

**Assessment:** General crawlers (`*`) are only blocked from `/search` and `/beta/`. Event listing pages are NOT blocked. A well-behaved crawler respecting rate limits (1s+ between requests, identifying User-Agent) is within the scope of permitted access. **The sitemap is explicitly published** — consuming it is the lowest-risk path.

**Risk level:** LOW-MEDIUM. Scraping is technically permitted by robots.txt. Terms of Service review still required before Phase 2.

### tiketa.lt robots.txt (verified 2026-05-25)

```
User-agent: *
Disallow: /
```

**Assessment:** tiketa.lt explicitly blocks all crawlers from the entire site. This is the strongest possible machine-readable signal against scraping. Proceeding without a partnership agreement or explicit legal opinion creates significant legal exposure under Lithuanian civil law and potentially EU Database Directive 96/9/EC.

**Risk level:** HIGH. **Phase 2 tiketa.lt scraper requires a legal gate: either (a) partnership/API agreement, or (b) written legal opinion from LT IP lawyer.**

### EU Database Directive 96/9/EC Key Points

[CITED: eur-lex.europa.eu — Directive 96/9/EC]

- **Sui generis right:** Database makers can block extraction of "substantial parts" of a database where substantial investment was made in obtaining, verifying, or presenting contents.
- **2021 ECJ ruling:** Claimant must establish both substantial extraction AND significant detriment to investment — raised the bar for infringement claims. [CITED: medialaws.eu/ecj-clarifies-database-directive-scope/]
- **Insubstantial parts exception:** Article 8(1) — lawful users may use "insubstantial parts" of a database; this right cannot be contracted away (Article 15).
- **Event listing data:** Individual event facts (title, date, venue, price) are likely **not** copyright-protected. The database selection/arrangement might be, but extracting individual factual records is lower risk than bulk extraction.
- **Safe harbor pattern:** Link to source + display only summary data (title, date, city, price range) + deep-link to ticket page. Do NOT reproduce full descriptions, images, or structured data that required editorial investment.

### Partner Outreach Plan

Phase 1 legal memo must include draft outreach emails to:
1. **bilietai.lt** — request formal permission or API access; propose revenue share or white-label partnership
2. **tiketa.lt** — request API documentation or data partnership; inform about aggregator model

Template email principles:
- Identify Event Sea as an aggregator (not a competitor)
- Explain that event discovery increases their ticket sales
- Offer to display "buy tickets on tiketa.lt" with UTM attribution
- Ask about API or data feed availability
- Provide contact email and company registration details

---

## GDPR Compliance Notes

### Article 30 Processing Register (ROPA) — Minimum Fields

[CITED: GDPR Article 30 text; ICO documentation]

The ROPA must be written, kept up to date, and available to the supervisory authority. Minimum controller record fields:

1. **Name and contact details** of controller (Event Sea / company registration)
2. **Purposes of processing** (e.g., event aggregation, user watchlist, price alerts)
3. **Categories of data subjects** (website visitors, registered users, B2B organizers)
4. **Categories of personal data** (email, preferences, watch history)
5. **Recipients / categories of recipients** (Resend for email, Telegram, Supabase)
6. **International transfers** (Supabase Frankfurt = EU; no transfer outside EEA)
7. **Retention periods** (e.g., user data until account deletion + 30 days; price history indefinitely as aggregate data)
8. **Security measures** (TLS in transit, encryption at rest via Supabase, access control)

For Phase 1, the ROPA can be a Markdown file in the repo at `docs/gdpr/article-30-ropa.md`.

### Cookie Consent Requirements

[CITED: gdpr.eu/cookies/]

- **Strictly necessary cookies** (session, auth): No consent required.
- **Analytics cookies** (PostHog, Plausible, Google Analytics): **Explicit opt-in required** before any script loads.
- Pre-ticked checkboxes are illegal (Planet49 ruling).
- Consent must be as easy to withdraw as to give.
- Users must be able to access the service even if they refuse analytics cookies.

### Privacy Policy Minimum Content for /privacy Page

- Identity of data controller (company name, address, email)
- What data is collected and why (Article 13/14)
- Legal basis for each processing activity
- Data retention periods
- User rights (access, rectification, erasure, portability, objection)
- Right to lodge complaint with supervisory authority (Lithuania: VDAI — Valstybinė duomenų apsaugos inspekcija)
- Cookie policy summary
- Contact details for data protection queries

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Meilisearch `v1.13.3` is current stable v1.x | Docker Compose Pattern | Wrong version tag — check hub.docker.com/r/getmeili/meilisearch/tags before pinning |
| A2 | `postgres` (postgres.js) 3.4.9 is well-established | Standard Stack | Minor — switch to `pg` 8.21.0 if compatibility issues arise |
| A3 | `ioredis` 5.10.1 is well-established | Standard Stack | Minor — BullMQ also supports `redis` (node-redis v4+) as alternative |
| A4 | Supabase Frankfurt Postgres connection uses standard PostgreSQL wire protocol | Architecture | If Supabase changes pooler behavior, connection string or driver may need adjustment |
| A5 | tiketa.lt `Disallow: /` has not changed since research date | Legal memo | Re-verify tiketa.lt/robots.txt before Phase 2 begins |

---

## Open Questions

1. **Meilisearch exact version to pin**
   - What we know: Decision D-05/D-06 says `v1.x`. Current latest is unknown without Docker Hub check.
   - What's unclear: Whether to pin to a specific patch (e.g., `v1.13.3`) or use `v1` floating tag.
   - Recommendation: Pin to specific version for reproducibility. Check hub.docker.com/r/getmeili/meilisearch/tags and pin latest v1.x patch.

2. **pnpm version to specify in CI**
   - What we know: pnpm 9.x is current; pnpm 8.x was widely used in guides. The action `pnpm/action-setup@v3` accepts `version`.
   - What's unclear: Whether the project should pin pnpm 8 or 9.
   - Recommendation: Use pnpm 9 (latest stable). Add `"packageManager": "pnpm@9.x.x"` to root package.json.

3. **Supabase connection pooler mode for worker**
   - What we know: Supabase offers Transaction and Session pooler modes. Long-running worker processes need Session mode.
   - What's unclear: Whether apps/worker should connect via Supabase connection string or direct Postgres URL.
   - Recommendation: Worker uses direct Postgres URL (not pooler) for long-running connections. Web uses Transaction pooler. Document both URLs in `.env.example`.

4. **LT IP lawyer timeline**
   - What we know: Legal memo in Phase 1 is self-authored; LT lawyer review is pre-Phase 2 prerequisite.
   - What's unclear: Whether Phase 2 can begin before lawyer response if bilietai.lt (lower risk) scraper is ready.
   - Recommendation: Allow Phase 2 bilietai.lt scraper to start concurrently with tiketa.lt lawyer review. Gate tiketa.lt scraper on lawyer clearance specifically.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js 20.9+ | Next.js 16 minimum | Available | v24.14.1 (exceeds minimum) | — |
| pnpm | Monorepo package manager | Not installed | — | `npm install -g pnpm` |
| Docker / Docker Compose | Local dev environment | Not installed | — | Developers must install Docker Desktop |
| flyctl CLI | Worker deployment | Not installed | — | Install via `brew install flyctl` or curl script |
| Python 3 (pip) | slopcheck | Available | 3.12 | — |

**Missing dependencies with no fallback:**
- Docker: Required for local postgres, redis, and meilisearch. Without it, local development requires individual service installs or remote cloud services.

**Missing dependencies with fallback:**
- pnpm: Can install via `npm install -g pnpm@9`
- flyctl: Can install via `brew install flyctl` on macOS or `curl -L https://fly.io/install.sh | sh`

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Vitest 4.1.7 |
| Config file | `packages/db/vitest.config.ts` and `apps/worker/vitest.config.ts` (per-package) |
| Quick run command | `pnpm --filter db test` |
| Full suite command | `pnpm turbo test` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DATA-03 | Independent scraper queues (bilietai, tiketa) are separate BullMQ Queue instances | unit | `pnpm --filter worker test -- queues` | No — Wave 0 |
| DATA-04 | `generateEventId(url)` returns consistent hash for same URL | unit | `pnpm --filter db test -- event-id` | No — Wave 0 |
| DATA-06 | robots.txt analysis documented in legal memo | manual | `cat docs/legal/scraping-memo.md` | No — Wave 0 |
| DATA-07 | `sources` table has `last_successfully_scraped_at` and `consecutive_failures` columns | migration test | `pnpm --filter db test -- schema` | No — Wave 0 |
| GDPR-01 | `/privacy` route returns 200 and contains privacy policy content | smoke | `pnpm --filter web test -- privacy` | No — Wave 0 |
| GDPR-02 | CookieConsent component renders; analytics gated behind consent | unit | `pnpm --filter web test -- cookie-consent` | No — Wave 0 |
| GDPR-03 | `docs/gdpr/article-30-ropa.md` exists and has all 8 required fields | manual | File exists check | No — Wave 0 |
| GDPR-04 | Deleting a user record cascades to watchlist, alert_log rows | migration test | `pnpm --filter db test -- cascade-delete` | No — Wave 0 |

### Sampling Rate
- **Per task commit:** `pnpm --filter <package> test`
- **Per wave merge:** `pnpm turbo typecheck lint test`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `packages/db/src/__tests__/schema.test.ts` — covers DATA-04, DATA-07, GDPR-04
- [ ] `apps/worker/src/__tests__/queues.test.ts` — covers DATA-03
- [ ] `apps/web/app/__tests__/privacy.test.ts` — covers GDPR-01
- [ ] `apps/web/components/__tests__/CookieConsent.test.tsx` — covers GDPR-02
- [ ] `packages/db/vitest.config.ts` — shared vitest config
- [ ] Framework install: `pnpm --filter db add -D vitest` and `pnpm --filter web add -D vitest`

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No (Phase 4) | — |
| V3 Session Management | No (Phase 4) | — |
| V4 Access Control | No (Phase 4) | — |
| V5 Input Validation | Partial — DB writes from scraper output | Drizzle typed schema; never raw SQL interpolation |
| V6 Cryptography | No (Phase 1 skeleton only) | — |
| V7 Error Handling | Yes — worker crash handling | BullMQ `worker.on('failed')` handler; no stack traces to client |
| V14 Configuration | Yes — secrets management | `.env.local` only; `.env.example` documents required vars; never commit secrets |

### Known Threat Patterns for this Stack

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| DATABASE_URL in env committed to git | Information Disclosure | `.gitignore` covers `.env.local`; `.env.example` has no real values |
| BullMQ job data containing PII | Information Disclosure | Do not put user emails in job payloads; use user IDs only |
| tiketa.lt scraping without authorization | Legal / Repudiation | Legal gate before Phase 2 tiketa scraper |
| Cookie consent bypass (analytics before consent) | Privacy violation | Analytics only initialized in `onConsent` callback; never at module level |
| Drizzle raw SQL injection | Tampering | Use Drizzle query builder only; never `sql\`...\`` with user-controlled input |

---

## Sources

### Primary (HIGH confidence)
- [nextjs.org/blog/next-16](https://nextjs.org/blog/next-16) — Next.js 16 feature list, breaking changes, React 19.2
- [nextjs.org/docs/app/guides/upgrading/version-16](https://nextjs.org/docs/app/guides/upgrading/version-16) — Complete breaking changes list, async API migration
- [turborepo.dev/docs/reference/configuration](https://turborepo.dev/docs/reference/configuration) — turbo.json v2 `tasks` schema, verified
- [turborepo.dev/docs/guides/ci-vendors/github-actions](https://turborepo.dev/docs/guides/ci-vendors/github-actions) — CI workflow YAML
- [orm.drizzle.team/docs/column-types/pg](https://orm.drizzle.team/docs/column-types/pg) — PostgreSQL column types
- [orm.drizzle.team/docs/get-started/postgresql-new](https://orm.drizzle.team/docs/get-started/postgresql-new) — drizzle.config.ts, connection setup
- [orm.drizzle.team/docs/migrations](https://orm.drizzle.team/docs/migrations) — push vs migrate workflow
- [docs.bullmq.io/readme-1](https://docs.bullmq.io/readme-1) — BullMQ v5 Queue + Worker patterns
- [upstash.com/docs/redis/integrations/bullmq](https://upstash.com/docs/redis/integrations/bullmq) — BullMQ + Upstash Redis connection config
- [fly.io/docs/launch/continuous-deployment-with-github-actions/](https://fly.io/docs/launch/continuous-deployment-with-github-actions/) — flyctl GitHub Actions workflow
- [cookieconsent.orestbida.com/essential/getting-started.html](https://cookieconsent.orestbida.com/essential/getting-started.html) — vanilla-cookieconsent v3 API
- [gdpr.eu/cookies/](https://gdpr.eu/cookies/) — Cookie consent legal requirements
- bilietai.lt/robots.txt — Verified 2026-05-25: only /search and /beta/ blocked
- tiketa.lt/robots.txt — Verified 2026-05-25: `Disallow: /` for all crawlers
- npm registry (`npm view <pkg>`) — All package versions and repository URLs

### Secondary (MEDIUM confidence)
- [turborepo.dev/docs/guides/tools/vitest](https://turborepo.dev/docs/guides/tools/vitest) — Vitest + Turborepo caching
- [termsfeed.com/blog/gdpr-article-30-create-ropa/](https://www.termsfeed.com/blog/gdpr-article-30-create-ropa/) — Article 30 template and field list
- [medialaws.eu/ecj-clarifies-database-directive-scope/](https://www.medialaws.eu/ecj-clarifies-database-directive-scope/) — 2021 ECJ database directive ruling

### Tertiary (LOW confidence)
- Meilisearch v1.13.3 Docker tag [ASSUMED] — verify at hub.docker.com before pinning

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all versions verified against npm registry with source repos confirmed
- Architecture patterns: HIGH — turbo.json from official docs, Next.js from official upgrade guide, Drizzle from official docs
- Legal analysis (bilietai.lt): HIGH — robots.txt fetched and verified directly
- Legal analysis (tiketa.lt): HIGH — robots.txt fetched and verified directly; `Disallow: /` finding is definitive
- EU Database Directive: MEDIUM — cited from official EU legislation and ECJ ruling summary; specific LT implementation [ASSUMED]
- Cookie consent requirements: HIGH — cited from gdpr.eu official source
- Pitfalls: HIGH — most derived from official docs (breaking changes, known BullMQ requirements)

**Research date:** 2026-05-25
**Valid until:** 2026-08-25 (90 days — stable tech stack; re-verify if Next.js or Drizzle have major releases)
**Critical re-verify before Phase 2:** tiketa.lt robots.txt (legal gate)
