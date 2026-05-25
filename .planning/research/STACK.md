# Technology Stack

**Project:** EventSea — Lithuanian Events Aggregator
**Researched:** 2026-05-25
**Confidence:** HIGH (all versions verified via npm registry and Context7 official docs)

---

## Recommended Stack

### Core Frontend Framework

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Next.js | 16.x (latest stable) | Full-stack web framework | App Router + RSC gives SSR for SEO (event pages must be crawlable), React 19, Turbopack dev, TypeScript-native config. PWA-first fits the aggregator model better than a native app for v1. |
| React | 19.x | UI runtime | Ships with Next.js 16; concurrent features, Server Components |
| TypeScript | 6.x | Type safety | Full-stack type sharing between tRPC client and server eliminates entire classes of bugs |

**Decision: PWA over React Native for v1.** EventSea is a content aggregator, not a utility requiring native hardware APIs. A Next.js PWA gives: (1) instant indexability — event pages need Google SEO from day one, (2) zero app-store friction for initial user acquisition in LT, (3) calendar `.ics` exports work identically in browser and installed PWA, (4) one codebase for mobile web + desktop B2B dashboard. React Native (Expo) adds build complexity that is unjustified until you have validated user retention requiring push notification-heavy re-engagement. Re-evaluate at 10k MAU.

### PWA Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| @serwist/next | 9.x | Service worker / PWA | Maintained fork of Workbox with active Next.js 15/16 App Router support. `next-pwa` (v5.6) is effectively unmaintained — last release 2022, no App Router support. Serwist is the current community standard. |

### API Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| tRPC | 11.x | Type-safe API between frontend and backend | Eliminates REST boilerplate. With Next.js App Router you get end-to-end type inference from database schema to UI without code generation. Supports both RSC calls and React Query for client-side. Proven in monorepo aggregator patterns. |
| Zod | 4.x | Schema validation + tRPC input parsing | v4 is now stable (released 2025). Validates all scraped event data before DB writes. Shared schemas between scraper, API, and frontend. |

### Backend / Database

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| PostgreSQL (via Supabase) | Supabase v1.26 / pg 15+ | Primary database | Events are highly relational (events → venues → organizers → prices → categories). Postgres full-text search handles Lithuanian accent-aware queries natively. Supabase gives Auth, Realtime, Row Level Security, and Storage in one platform — eliminates managing 4 separate services at v1 scale. |
| Drizzle ORM | 0.45.x | Type-safe query builder | Lightest abstraction that preserves SQL expressiveness. Schema-as-code generates TypeScript types consumed directly by tRPC. Migrations via `drizzle-kit push` are simpler than Prisma's migration engine for a greenfield project. **Not Prisma** — Prisma's query engine is a binary sidecar that complicates serverless cold starts and Docker images. |
| Supabase Realtime | (bundled) | Live event updates, price changes | WebSocket subscriptions over Postgres changes. Delivers real-time price-drop notifications to open browser sessions without a separate WebSocket server. |

### Search

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Meilisearch | 0.58.x (JS client) | Full-text event search | Self-hosted (Fly.io or Railway, ~$5/mo). Typo-tolerance is essential for Lithuanian event name searches ("Vilnius" vs "vilnuis"). Sub-50ms response time. Faceted search for city/date/category filtering built-in. **Not Algolia** — Algolia's pricing ($0.50/1000 searches) is prohibitive at scale for a free consumer app; Meilisearch is identical DX, open source, and $0 once you own the instance. Not Postgres full-text — too slow for faceted typeahead at scale. |

### Job Queue / Background Processing

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| BullMQ | 5.x | Scraping job orchestration, scheduled crawls, price tracking | Redis-backed, cron support via `upsertJobScheduler`, concurrency control, retry logic with exponential backoff. Essential for rate-limiting scraping workers to respect target site rate limits. |
| Inngest | 4.x | Durable event-driven workflows (alerts, calendar syncs, email sends) | Serverless-native, no Redis management for event-driven flows. BullMQ handles high-frequency polling jobs; Inngest handles low-frequency durable workflows (price alert triggers, welcome email sequences). Both are used because they solve different sub-problems. |

### Web Scraping

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Crawlee | 3.x | Scraping orchestration framework | Built-in anti-bot detection (puppeteer-extra-plugin-stealth integration), request queuing, rate limiting, proxy rotation, auto-retry. Production-grade vs writing raw Playwright loops. |
| Playwright | 1.x | Headless browser for JS-heavy targets | Handles bilietai.lt, tiketa.lt, Facebook Events (all require JS execution). Used via Crawlee's `PlaywrightCrawler`. |
| Cheerio | (as needed) | HTML parsing for static pages | Faster than browser for simple HTML targets (RSS feeds, plain HTML event pages). |

**Proxy strategy:** Crawlee + residential proxies (BrightData or Webshare) for targets with bot detection. Static scrapes (open APIs, RSS) run bare. Start with direct requests and add proxies only when blocked.

### Authentication

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Better Auth | 1.6.x | User auth + B2B organizer sessions | Framework-agnostic, TypeScript-first, ships with Google OAuth (critical for Lithuanian users who expect Google login), email+password, magic links. Active development (v1.6 released 2026). **Not NextAuth/Auth.js** — Auth.js v5 is still in beta-quality; Better Auth has a cleaner API and more active maintenance. **Not Supabase Auth** — Supabase Auth forces you into their JWT flow; Better Auth integrates with your own Postgres via Drizzle. |

### Notifications

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Resend | 6.x | Transactional email (price alerts, confirmations) | Developer-first email API. React Email templates render server-side. 100 emails/day free tier adequate for MVP validation. Better deliverability than SendGrid for cold start. |
| grammY | 1.43.x | Telegram bot for price alerts | Lithuanian users skew heavily toward Telegram. grammY is the modern, TypeScript-native Telegram bot library (not the legacy `node-telegram-bot-api` which is effectively unmaintained). Webhook-based, works in serverless. |
| Web Push (native browser API) | — | PWA push notifications | Expo push is not needed (no native app). Use web-push npm package for VAPID-based PWA notifications. |

### Calendar Integration

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| ical-generator | 10.x | Generate .ics files for Google/Apple/Outlook export | Server-side `.ics` generation. One `.ics` URL per event. Users click "Add to Calendar" → browser downloads file → adds to system calendar. No OAuth required for this flow. **Not Google Calendar API OAuth** — forcing Google login to add a single event is conversion-killing friction; `.ics` download works universally. |

### Caching / Rate Limiting

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Upstash Redis | 1.38.x | Serverless Redis for caching + BullMQ broker | HTTP-based Redis that works in serverless/edge without TCP persistent connections. Used as BullMQ broker AND for API response caching (event lists, search results). Single instance handles both workloads. |

### Payments (B2B tier)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Stripe | 22.x | B2B featured listing subscriptions | Standard. Stripe supports EUR/LT billing natively. Use Stripe Checkout to avoid PCI scope on your own servers. |

### UI / Styling

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Tailwind CSS | 4.x | Utility-first styling | v4 ships with Oxide engine (Rust-based, 5x faster builds). Dark mode by default aligns with EventSea's design direction. CSS-first config (no `tailwind.config.js` needed). |
| shadcn/ui | latest | Headless component primitives | Copy-paste components built on Radix UI. Not a dependency — you own the code. Accessible, dark-mode-ready, works with Tailwind 4. EventSea needs a design language, not a component library that fights customization. |
| Lucide React | 1.16.x | Icon set | Consistent with shadcn/ui's default icon set. Tree-shakeable. |
| TanStack Table | 8.x | B2B analytics data tables | Feature-complete headless table for the organizer dashboard. Sort, filter, paginate — all without opinionated styles. |

### Data Fetching (Client-side)

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| TanStack Query | 5.x | Client-side server state, caching, optimistic updates | Used through tRPC's React Query adapter. Handles stale-while-revalidate for the event feed, infinite scroll, and wishlist optimistic updates. v5 is stable (v5.100 as of 2026). |

### Observability

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Sentry | latest | Error tracking, performance monitoring | Next.js 15+ has native `onRequestError` instrumentation hook — Sentry integrates directly. Free tier (5k errors/mo) sufficient for MVP. |

---

## Infrastructure / Hosting

| Service | Purpose | Why |
|---------|---------|-----|
| Vercel | Next.js hosting | Zero-config deployment, Edge Network CDN in Europe (Frankfurt), Next.js-native. Free tier for MVP. Scales automatically. |
| Supabase (managed) | Postgres + Auth + Realtime + Storage | Managed Postgres in eu-central-1 (Frankfurt). GDPR-compliant EU hosting critical for LT market. |
| Fly.io | Meilisearch + BullMQ workers (long-running) | Persistent VMs for services that can't run serverless. Meilisearch needs a volume. BullMQ workers need persistent Redis connection. ~$7/mo for a shared-cpu-1x 256MB instance. |
| Upstash Redis | Redis (BullMQ broker + cache) | Serverless Redis, eu-west-1 region. No server to manage. |
| Cloudflare Images | Event image optimization/CDN | Scraped images re-hosted through Cloudflare. $5/mo covers ~100k images. Avoids hotlinking DMCA exposure. |

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Frontend | Next.js 16 (PWA) | Expo (React Native + Web) | Expo adds native build pipeline complexity (EAS Build costs, app store reviews, two codebases in practice). PWA is sufficient for v1 aggregator UX. Re-evaluate post-PMF. |
| Frontend | Next.js 16 | Remix 2 | Next.js has stronger RSC ecosystem, better SEO primitives (metadata API), larger hiring pool in LT. Remix is excellent but Next.js is the safer enterprise choice for a content-heavy site. |
| Database | Supabase (Postgres) | PlanetScale (MySQL) | PlanetScale killed its free tier. MySQL lacks native full-text search quality for Lithuanian. Postgres GIN indexes + `unaccent` extension handles LT characters correctly. |
| ORM | Drizzle | Prisma | Prisma query engine binary is 40MB+ and causes cold start latency. Drizzle is 500KB and has zero runtime overhead. For a data-heavy scraping app, Drizzle's raw SQL escape hatches are frequently needed. |
| Auth | Better Auth | Supabase Auth | Supabase Auth couples you to their JWT flow and their user table schema. Better Auth stores users in YOUR Postgres via Drizzle — one less moving part. |
| Auth | Better Auth | NextAuth/Auth.js v5 | Auth.js v5 remained in beta-quality through 2025. Better Auth v1.6 is production-stable with more complete plugin ecosystem. |
| Search | Meilisearch | Algolia | Algolia search pricing ($0.50/1k searches) becomes $500/mo at 1M searches — unsustainable for a free consumer product. Meilisearch self-hosted is $7/mo flat on Fly.io. |
| Search | Meilisearch | Postgres FTS | Postgres full-text is adequate for simple search but lacks instant typo-tolerant autocomplete and faceted filtering performance needed for sub-50ms event typeahead. |
| Queue | BullMQ + Inngest | Temporal | Temporal requires a server cluster (complex ops). BullMQ + Inngest covers all EventSea's queue needs at 1/10th the operational cost. |
| Queue | BullMQ + Inngest | AWS SQS | SQS lacks cron scheduling and job retry visibility. BullMQ has a UI (Bull Board) for monitoring scraping jobs. |
| Scraping | Crawlee | Scrapy (Python) | Introduces a second language stack. Crawlee is TypeScript-native, integrates with the same repo as the backend. Crawlee is production-grade with better stealth support than raw Playwright. |
| Email | Resend | SendGrid | SendGrid's free tier requires credit card verification. Resend has cleaner API, React Email template support, and better EU deliverability out of the box. |
| Hosting | Vercel | Railway | Vercel has deeper Next.js integration (ISR, Edge Functions, automatic image optimization). Railway is a valid fallback if Vercel costs grow. |
| Styling | Tailwind 4 | CSS Modules | Tailwind 4 with shadcn/ui provides a complete design system. CSS Modules require building that system from scratch. |
| Calendar | ical-generator (.ics) | Google Calendar API (OAuth) | OAuth flow for adding a single event is a conversion killer. `.ics` download works universally across Google, Apple, Outlook without any account linking. |

---

## Installation

```bash
# Initialize Next.js 16
npx create-next-app@latest eventsea --typescript --tailwind --app --src-dir

# Core framework
npm install next react react-dom

# API layer
npm install @trpc/server @trpc/client @trpc/react-query @trpc/next
npm install zod

# ORM + Database
npm install drizzle-orm @supabase/supabase-js
npm install -D drizzle-kit

# Auth
npm install better-auth

# Queue + Background jobs
npm install bullmq inngest

# Scraping
npm install crawlee playwright
npm install playwright-extra puppeteer-extra-plugin-stealth

# Search
npm install meilisearch

# Notifications
npm install resend grammy web-push

# Calendar
npm install ical-generator

# Caching
npm install @upstash/redis

# Payments
npm install stripe

# Data fetching (client)
npm install @tanstack/react-query

# UI
npm install @tanstack/react-table lucide-react
# shadcn/ui via CLI (not npm install)
npx shadcn@latest init

# PWA
npm install @serwist/next

# Tailwind 4
npm install tailwindcss @tailwindcss/postcss

# Dev tooling
npm install -D typescript @types/node @types/react @types/react-dom
```

---

## Version Pinning Strategy

Pin major versions in `package.json`. Critical versions verified 2026-05-25:

| Package | Verified Version |
|---------|-----------------|
| next | 16.2.6 |
| expo (if added later) | 56.0.4 |
| drizzle-orm | 0.45.2 |
| drizzle-kit | 0.31.10 |
| bullmq | 5.77.3 |
| better-auth | 1.6.11 |
| @trpc/server | 11.17.0 |
| crawlee | 3.16.0 |
| meilisearch | 0.58.0 |
| resend | 6.12.4 |
| ical-generator | 10.2.0 |
| zod | 4.4.3 |
| @tanstack/react-query | 5.100.14 |
| @tanstack/react-table | 8.21.3 |
| inngest | 4.4.0 |
| stripe | 22.1.1 |
| grammy | 1.43.0 |
| @upstash/redis | 1.38.0 |
| playwright | 1.60.0 |
| @serwist/next | 9.5.11 |
| tailwindcss | 4.3.0 |
| lucide-react | 1.16.0 |
| typescript | 6.0.3 |
| @supabase/supabase-js | 2.106.2 |

---

## What NOT to Use

| Technology | Reason |
|------------|--------|
| Prisma ORM | 40MB+ query engine binary, cold start latency in serverless, complex migration engine — Drizzle is superior for this use case |
| next-pwa | Unmaintained since 2022, no App Router support — use @serwist/next |
| Auth.js (NextAuth) v5 | Beta-quality through 2025, incomplete plugin ecosystem — use Better Auth |
| Supabase Auth | Couples user management to Supabase's JWT flow; use Better Auth + your own Postgres |
| Algolia | Cost prohibitive ($0.50/1k searches) for free consumer product — use Meilisearch self-hosted |
| Scrapy / Python scraping | Introduces second language stack in a TypeScript monorepo — use Crawlee |
| GraphQL | Adds resolver boilerplate and code generation steps that tRPC solves more elegantly for a TypeScript-only stack |
| Expo / React Native | Premature for v1 aggregator; app store friction hurts early-stage user acquisition in LT market |
| Temporal | Over-engineered for EventSea's job complexity; BullMQ + Inngest covers all needs without a server cluster |
| SendGrid / Mailgun | SendGrid requires credit card upfront; Mailgun has poor EU deliverability — use Resend |
| Firebase / Firestore | NoSQL is wrong data model for relational event data; Firebase Auth is proprietary lock-in |
| PlanetScale | Killed free tier; MySQL lacks quality full-text search for Lithuanian content |
| node-telegram-bot-api | Last release 2022, effectively unmaintained — use grammY |

---

## Confidence Assessment

| Area | Confidence | Basis |
|------|------------|-------|
| Next.js 16 as framework | HIGH | Confirmed stable via nextjs.org official blog + npm registry |
| PWA over React Native | HIGH | Well-established tradeoff; content aggregators consistently choose web-first |
| Drizzle over Prisma | HIGH | Context7 official Drizzle docs + known Prisma serverless limitations |
| Better Auth over NextAuth | HIGH | npm registry (Better Auth v1.6.11 stable, Auth.js v5 still RC-era) |
| Meilisearch over Algolia | HIGH | Pricing verified from Algolia public pricing page; Meilisearch open source |
| BullMQ for queues | HIGH | Context7 official BullMQ docs confirm cron scheduling + job scheduler API |
| Crawlee for scraping | HIGH | Context7 official Crawlee docs + stealth plugin confirmed |
| Serwist over next-pwa | HIGH | npm last-publish dates confirm next-pwa unmaintained; Serwist Context7 confirmed |
| Inngest for durable jobs | MEDIUM | Context7 docs confirm API; production scale for LT market unverified |
| Meilisearch Lithuanian support | MEDIUM | Typo tolerance confirmed; Lithuanian-specific accent handling needs phase validation |
| Supabase EU region | MEDIUM | Frankfurt region confirmed in Supabase docs; GDPR compliance adequate for LT |
| Telegram penetration in LT | MEDIUM | Based on known Baltic market patterns; should be validated in discovery phase |

---

## Sources

- Next.js 15 official blog: https://nextjs.org/blog/next-15 (published October 2024)
- Context7: `/vercel/next.js` — App Router, RSC, Server Actions
- Context7: `/taskforcesh/bullmq` — cron scheduling, job scheduler API
- Context7: `/websites/crawlee_dev_js` — Playwright crawler, stealth plugin
- Context7: `/drizzle-team/drizzle-orm-docs` — schema, migrations, Postgres config
- Context7: `/websites/supabase` — Realtime RLS, edge functions, queue consumption
- Context7: `/better-auth/better-auth` — Google OAuth, social providers, v1.6
- Context7: `/trpc/trpc` — v11 Next.js App Router route handler integration
- Context7: `/sebbo2002/ical-generator` — iCal calendar event generation and serving
- Context7: `/meilisearch/documentation` — typo tolerance, faceted search
- Context7: `/inngest/inngest-js` — cron triggers, durable step functions
- Context7: `/websites/serwist_pages_dev` — Next.js 15 PWA service worker config
- Context7: `/colinhacks/zod` — v4 schema validation
- npm registry (verified 2026-05-25): all version numbers above
