# Architecture Patterns: Events Aggregator Platform

**Domain:** Scraping-heavy data aggregation with price tracking and notifications
**Project:** Event Sea — Lithuanian events aggregator
**Researched:** 2026-05-25
**Overall confidence:** HIGH (patterns verified via Context7 docs + well-established domain)

---

## Recommended Architecture

**Verdict: Modular monorepo, deployed as a monolith initially, with clean internal service boundaries that allow extraction later.**

Do NOT start with microservices. Lithuania market at 50K active users is a monolith-scale problem. Microservices add operational overhead without benefit at this scale. Clean internal module boundaries (scraper module, notification module, etc.) give the flexibility to extract later without forcing distributed systems complexity upfront.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SOURCES                         │
│  bilietai.lt  │  tiketa.lt  │  Facebook Events  │  Organizer   │
│  (HTML/API)   │  (HTML/API) │  (Graph API)      │  websites    │
└──────┬────────┴──────┬──────┴────────┬───────────┴──────┬───────┘
       │               │               │                   │
       ▼               ▼               ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SCRAPER LAYER                               │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐  │
│  │ HTTP Scrapers│  │ JS Scrapers │  │   API Adapters        │  │
│  │  (Cheerio)  │  │ (Playwright)│  │  (REST/GraphQL fetch) │  │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬────────────┘  │
│         └────────────────┼─────────────────────┘               │
│                          ▼                                      │
│             ┌────────────────────────┐                         │
│             │  Source Adapters       │                         │
│             │  (normalize raw →      │                         │
│             │   canonical format)    │                         │
│             └────────────┬───────────┘                         │
└──────────────────────────┼──────────────────────────────────────┘
                           │ enqueues jobs
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     JOB QUEUE (BullMQ + Redis)                  │
│                                                                 │
│  scrape:schedule  →  scrape:fetch  →  ingest:normalize          │
│                                            │                    │
│                             ┌──────────────┘                    │
│                             ▼                                   │
│                   ingest:deduplicate  →  ingest:persist         │
│                                              │                  │
│                             ┌────────────────┘                  │
│                             ▼                                   │
│                   price:compare  →  alert:notify                │
└──────────────────────────────────────────────────────────────────┘
                           │ reads/writes
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DATABASE LAYER                              │
│                                                                 │
│  PostgreSQL (primary store)          Redis (cache + queues)     │
│  - events table                      - scrape job queue         │
│  - price_history table               - alert job queue          │
│  - sources table                     - session cache            │
│  - users table                       - rate limit counters      │
│  - alerts table                      - dedup fingerprints       │
│  - watchlist table                                              │
│  - organizer table                                              │
└──────────────────────────────────────────────────────────────────┘
                           │ queries
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API LAYER (Next.js)                         │
│                                                                 │
│  Public REST API              Server Actions (internal)         │
│  GET /api/events              - trigger manual scrape           │
│  GET /api/events/:id          - manage watchlist                │
│  GET /api/events/:id/prices   - update user preferences         │
│  POST /api/auth/*                                               │
│  GET /api/feed (personalized)                                   │
│  GET /api/calendar/:userId.ics  (iCal subscription)             │
│  POST /api/b2b/*                                                │
└──────────────────────────────────────────────────────────────────┘
                           │ serves
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FRONTEND (Next.js PWA)                      │
│                                                                 │
│  Public catalog       User feed         B2B dashboard           │
│  Event detail         Watchlist         Organizer analytics      │
│  Price chart          Alerts prefs      Featured listing mgmt   │
└──────────────────────────────────────────────────────────────────┘
                           │ sends to
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     NOTIFICATION DELIVERY                       │
│                                                                 │
│  Email (Nodemailer/Resend)         Telegram (Telegraf bot)      │
│  - price drop alerts               - price drop alerts          │
│  - new event notifications         - event reminders            │
│  - event reminders                                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## Component Boundaries

| Component | Responsibility | Communicates With | Technology |
|-----------|---------------|-------------------|------------|
| **Scraper: HTTP** | Fetch static HTML pages, parse with Cheerio | Source Adapters | Cheerio + node fetch |
| **Scraper: JS** | Render JavaScript-heavy pages (SPAs) | Source Adapters | Playwright (headless) |
| **Scraper: API** | Call official REST/GraphQL endpoints | Source Adapters | Fetch + source-specific auth |
| **Source Adapters** | Normalize raw scraped data into canonical EventData shape | Job Queue | TypeScript interfaces |
| **Job Queue** | Schedule, retry, prioritize scrape and notification jobs | Scraper, Ingestor, Notifier | BullMQ + Redis |
| **Ingestor** | Deduplicate events, detect price changes, persist to DB | Database, Job Queue | PostgreSQL + Prisma |
| **Price Tracker** | Compare new price snapshot to last known price, emit change events | Database, Alert Queue | SQL queries + BullMQ |
| **API Layer** | Serve frontend, handle auth, expose REST + iCal endpoints | Database, Job Queue | Next.js App Router |
| **Frontend** | User catalog, personalized feed, B2B dashboard, PWA shell | API Layer | Next.js + React |
| **Notifier** | Consume alert queue, send email/Telegram | Nodemailer/Telegraf, Alert Queue | BullMQ Worker |
| **Calendar Service** | Generate per-user .ics subscription feeds | Database | ical-generator |
| **Auth** | Sessions, OAuth, JWT | Database, API Layer | NextAuth.js |
| **Human Curation UI** | Admin interface for reviewing scraped events, fixing bad data | Database, API Layer | Internal Next.js route |

---

## Data Flow

### Scrape Pipeline

```
1. SCHEDULE
   BullMQ cron job fires every 1-4h per source
   e.g. { pattern: '0 */2 * * *', data: { source: 'bilietai.lt' } }

2. FETCH
   Worker picks up job, selects scraper type (HTTP vs JS vs API)
   HTTP sources: Cheerio fetch → raw HTML → $.extract({ ... })
   JS sources:   Playwright headless → page.goto() → page.evaluate()
   API sources:  fetch(endpoint, { headers: { Authorization: apiKey } })

3. NORMALIZE
   Source Adapter maps raw fields → canonical EventData:
   {
     sourceId: string,         // e.g. "bilietai:12345"
     title: string,
     startAt: Date,
     endAt: Date | null,
     venue: string,
     city: string,
     category: string,
     price: { min: number, max: number, currency: 'EUR' } | null,
     ticketUrl: string,
     imageUrl: string | null,
     description: string | null,
     scrapedAt: Date
   }

4. DEDUPLICATE
   Compute fingerprint: sha256(title + date + venue + city)
   Check Redis for fingerprint (TTL: 24h) → skip if seen
   Fall back to DB lookup for older events

5. PERSIST
   Upsert into events table (ON CONFLICT on sourceId)
   If price changed: INSERT into price_history, enqueue alert job

6. ALERT CHECK
   price:compare worker queries users who watch this event
   For each user with price alert enabled: enqueue notification job

7. NOTIFY
   Notifier worker sends email via Nodemailer or Telegram via Telegraf
```

### User Feed Flow

```
User opens app
  → GET /api/feed?city=vilnius&page=1
  → Query: SELECT events WHERE city = ? AND startAt > NOW()
            ORDER BY personalisation_score DESC, startAt ASC
  → personalisation_score = weighted match against user.preferences (categories, venues)
  → Response: paginated list with prices, ticketUrl, isSaved
```

### Price Tracking Flow

```
Scrape run completes → new price found
  → SELECT last price_history entry for this event
  → IF price changed:
      INSERT INTO price_history (eventId, price, recordedAt, changeType)
      changeType = 'INCREASE' | 'DECREASE' | 'SALE_STARTED' | 'SOLD_OUT'
      Enqueue alert:price_change job with { eventId, oldPrice, newPrice }
  → IF price unchanged: no-op (no duplicate history rows)
```

### Calendar Sync Flow

```
User enables calendar sync
  → Server generates unique subscription token
  → Returns URL: /api/calendar/{token}.ics
  → User adds URL to Google/Apple/Outlook (subscribe, not import)
  → Calendar client polls URL every 1-24h (controlled by TTL header)
  → Server generates live .ics from user's watchlist using ical-generator
  → No push needed — pull model handles sync automatically
```

---

## Key Architectural Decisions

### 1. Monolith over microservices

**Decision:** Single Next.js application + separate worker process, sharing one database.
**Why:** 50K users is monolith scale. Microservices multiply operational cost: multiple services to deploy, monitor, debug, and secure. Clean module boundaries within a monolith allow extraction if needed at 500K+ users. Ship faster.
**Alternative rejected:** Microservices (scraper service, notification service, API service) — premature at this scale.

### 2. Polling over webhooks for scraping

**Decision:** BullMQ cron jobs poll sources every 1-4 hours.
**Why:** Bilietai.lt, Tiketa.lt, and organizer websites do not offer webhooks. Polling is the only option. Facebook Events Graph API requires app review; design the adapter to use polling until API access is granted.
**Polling intervals by source type:**
- High-value sources (bilietai.lt, tiketa.lt): every 1-2h
- Organizer websites: every 4-6h
- Facebook Events: every 4h (rate limit aware)

### 3. BullMQ FlowProducer for pipeline stages

**Decision:** Each scrape run is a parent job with child jobs per pipeline stage (fetch → normalize → persist → alert).
**Why:** Stage failures isolate cleanly. Failed normalization does not block the fetch result from retrying. Exponential backoff prevents hammering a slow source. Bull Dashboard provides visibility.
**Pattern:**
```typescript
await flowProducer.add({
  name: 'scrape-bilietai',
  queueName: 'scrape:schedule',
  children: [
    { name: 'fetch', queueName: 'scrape:fetch', data: { url, sourceId } },
  ],
});
```

### 4. Append-only price_history table

**Decision:** Never update a price record. Always INSERT a new row with `recordedAt`.
**Why:** Price history is the core differentiator. Users want to see the price trajectory (price went from €25 → €35 → €45 as event date approached). Append-only is also safe under concurrent scrapes.
**Anti-pattern rejected:** Updating `events.price` in-place — loses history entirely.

### 5. iCal subscription (pull) over Google Calendar API (push)

**Decision:** Serve a live .ics URL per user. Let calendar clients subscribe to it.
**Why:** Works with Google Calendar, Apple Calendar, and Outlook without OAuth from the user. Zero maintenance — the calendar client handles re-fetching. Google Calendar API push requires more complex OAuth flows and per-user refresh tokens.
**Downside:** Calendar sync delay is up to 24h on some clients (Google refreshes every 24h). Acceptable for events use case.

### 6. Separate worker process for scraping

**Decision:** BullMQ workers run in a separate Node.js process from the Next.js app.
**Why:** Playwright (headless browser) is memory-heavy. Keeping it in the same process as the web server risks memory pressure affecting response times. On deployment: Next.js app on Vercel/Railway, workers on a separate always-on instance (Railway, Fly.io, or dedicated VPS).

### 7. Cheerio for static sources, Playwright only for JS-heavy sources

**Decision:** Use Cheerio by default. Escalate to Playwright only when JavaScript rendering is required.
**Why:** Playwright launches a full headless browser — expensive in memory and time. Bilietai.lt and Tiketa.lt likely serve server-rendered HTML (verify per source). Playwright is reserved for SPAs or sources that load event data via client-side JS.
**Classification per source (verify during implementation):**
- bilietai.lt: static HTML → Cheerio
- tiketa.lt: check — likely Cheerio
- Organizer WordPress sites: Cheerio
- Facebook Events: API (Graph API) or skip
- Eventbrite Lithuania: REST API preferred

### 8. robots.txt compliance architecture

**Decision:** Build a RobotsChecker service that reads and caches robots.txt per domain. All scrapers check this before crawling.
**Why:** PROJECT.md explicitly requires legal/ethical scraping. Ignoring robots.txt creates legal risk (Lithuanian data protection law, BDAR/GDPR). Also damages relationship with sources we may want to partner with.
**Pattern:** Cache robots.txt responses for 24h in Redis. Log every blocked path. Respect crawl-delay directives.

---

## Database Schema Hints

### Core Tables (PostgreSQL)

```sql
-- Canonical event record
CREATE TABLE events (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_id     VARCHAR(255) UNIQUE NOT NULL,  -- e.g. "bilietai:12345"
  source        VARCHAR(50) NOT NULL,           -- e.g. "bilietai.lt"
  title         VARCHAR(500) NOT NULL,
  start_at      TIMESTAMPTZ NOT NULL,
  end_at        TIMESTAMPTZ,
  venue         VARCHAR(255),
  city          VARCHAR(100) NOT NULL,          -- vilnius, kaunas, klaipeda
  category      VARCHAR(100),                   -- concert, theater, sport, etc.
  ticket_url    TEXT NOT NULL,
  image_url     TEXT,
  description   TEXT,
  status        VARCHAR(50) DEFAULT 'active',   -- active, cancelled, sold_out
  is_featured   BOOLEAN DEFAULT FALSE,
  scraped_at    TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON events (city, start_at);
CREATE INDEX ON events (category, start_at);
CREATE INDEX ON events (source, scraped_at);

-- Price history — append-only, never UPDATE
CREATE TABLE price_history (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_id     UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  price_min    NUMERIC(10,2),
  price_max    NUMERIC(10,2),
  currency     CHAR(3) DEFAULT 'EUR',
  change_type  VARCHAR(50),  -- INITIAL, INCREASE, DECREASE, SALE_STARTED, SOLD_OUT
  recorded_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ON price_history (event_id, recorded_at DESC);

-- Registered users
CREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           VARCHAR(255) UNIQUE NOT NULL,
  telegram_chat_id BIGINT,
  city_preference VARCHAR(100),              -- default city
  categories      TEXT[],                   -- ['concert', 'theater']
  calendar_token  UUID UNIQUE DEFAULT gen_random_uuid(),
  notify_email    BOOLEAN DEFAULT TRUE,
  notify_telegram BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- User watchlist (saved events)
CREATE TABLE watchlist (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  event_id       UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  alert_on_price BOOLEAN DEFAULT TRUE,
  alert_on_date  BOOLEAN DEFAULT TRUE,
  added_at       TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(user_id, event_id)
);

-- Price change alerts sent (prevents duplicates)
CREATE TABLE alert_log (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id),
  event_id   UUID NOT NULL REFERENCES events(id),
  type       VARCHAR(50),     -- PRICE_CHANGE, EVENT_REMINDER, NEW_EVENT
  channel    VARCHAR(50),     -- email, telegram
  sent_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON alert_log (user_id, event_id, type, sent_at DESC);

-- Event organizer accounts (B2B)
CREATE TABLE organizers (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID NOT NULL REFERENCES users(id),
  company_name VARCHAR(255) NOT NULL,
  plan         VARCHAR(50) DEFAULT 'free',    -- free, basic, pro
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Scraping sources configuration
CREATE TABLE sources (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          VARCHAR(100) UNIQUE NOT NULL,  -- bilietai.lt
  base_url      TEXT NOT NULL,
  scraper_type  VARCHAR(50) NOT NULL,          -- cheerio, playwright, api
  schedule_cron VARCHAR(50) NOT NULL,          -- e.g. "0 */2 * * *"
  is_active     BOOLEAN DEFAULT TRUE,
  last_scraped  TIMESTAMPTZ,
  config        JSONB                          -- source-specific adapter config
);
```

### Prisma Schema Pattern

Use Prisma ORM with PostgreSQL. The `price_history` relation is one-to-many from `Event`. `Watchlist` is a many-to-many join with extra fields (alert flags). Do NOT use implicit many-to-many Prisma relations for watchlist — the extra columns require an explicit join model.

---

## Build Order (Dependency Graph)

Build in this exact sequence. Each phase unblocks the next.

```
Phase 1: Foundation
  ├── PostgreSQL schema + Prisma migrations
  ├── Redis setup
  ├── BullMQ queue infrastructure (empty workers, no logic yet)
  └── Next.js app shell (no pages yet)
  UNBLOCKS: everything else

Phase 2: Data Ingestion (Core Value)
  ├── HTTP scraper (Cheerio) for one static source (e.g. bilietai.lt)
  ├── Source Adapter for that source (canonical EventData)
  ├── Ingestor: upsert to events table
  ├── Deduplication logic
  └── Manual trigger endpoint (POST /api/admin/scrape/:source)
  UNBLOCKS: public catalog, price tracking

Phase 3: Public Catalog (User Value)
  ├── GET /api/events (filter by city, date, category)
  ├── GET /api/events/:id
  ├── Frontend: event listing page (no auth required)
  └── Frontend: event detail page
  UNBLOCKS: user testing, SEO, B2B demo

Phase 4: Price Tracking
  ├── price_history writes in Ingestor
  ├── Price change detection logic
  ├── Price chart component on event detail page
  └── GET /api/events/:id/prices
  UNBLOCKS: alerts, B2B analytics

Phase 5: Auth + Watchlist
  ├── NextAuth.js (email magic link or Google OAuth)
  ├── Watchlist API + frontend
  └── User preferences (city, categories)
  UNBLOCKS: personalized feed, alerts, calendar

Phase 6: Notifications
  ├── Alert queue workers
  ├── Nodemailer email sending
  ├── Telegraf bot setup + /start command
  └── Alert dedup via alert_log table
  UNBLOCKS: retention, engagement

Phase 7: Calendar Sync
  ├── ical-generator: per-user .ics feed endpoint
  ├── Calendar token in users table
  └── Frontend: "Add to Calendar" flow
  UNBLOCKS: passive retention

Phase 8: More Sources
  ├── Additional scrapers (tiketa.lt, organizer sites)
  ├── Playwright adapter for JS-heavy sources
  └── robots.txt compliance checker
  UNBLOCKS: content completeness

Phase 9: B2B Dashboard
  ├── Organizer auth + account model
  ├── Featured listing management
  ├── Analytics (views, clicks per event)
  └── Payment integration (Stripe or Paysera for LT market)
  UNBLOCKS: monetization
```

**Critical path:** Phase 1 → 2 → 3 is the absolute minimum to prove value. Everything else layers on top.

---

## Scraping Architecture: Legal and Ethical Layer

This deserves its own section because PROJECT.md explicitly flags it as a constraint.

### robots.txt Compliance

```typescript
// Every scraper MUST call this before fetching
class RobotsService {
  async isAllowed(sourceUrl: string, path: string): Promise<boolean>
  async getCrawlDelay(sourceUrl: string): Promise<number | null>
}
// Cached in Redis for 24h per domain
```

### Rate Limiting

Each source has a configurable `min_request_interval_ms` in the `sources.config` JSONB field. Workers enforce this via Redis rate limit counters. Default: 2-5 seconds between requests.

### Fingerprinting and Caching

Never re-scrape a page that has not changed. Use HTTP `ETag` / `Last-Modified` headers where available. Store last-seen ETags in `sources` or Redis.

### Legal Risk Mitigation by Source Type

| Source | Approach | Risk Level |
|--------|----------|------------|
| bilietai.lt HTML | Respect robots.txt, crawl-delay, no login-required pages | Low |
| tiketa.lt HTML | Same as above; check ToS for scraping clauses | Low-Medium |
| Facebook Events | Use Graph API (requires app review) not HTML scraping | Medium without API |
| Organizer websites | Public HTML only; attribute source; link back to original | Low |
| Eventbrite | Official API preferred | Low |

**Recommended:** Before scraping any source, document: (1) did you read robots.txt, (2) does ToS explicitly forbid scraping, (3) are you only collecting publicly visible data. Keep this log per source in the `sources.config` JSONB field.

---

## Scalability Considerations

| Concern | At 50K users (target) | At 500K users | Notes |
|---------|----------------------|---------------|-------|
| Web serving | Single Next.js instance | Horizontal scale or Vercel | Stateless, easy to scale |
| Scrape workers | 1 worker process, ~10 concurrent jobs | Multiple worker instances | BullMQ supports distributed workers |
| Database | Single PostgreSQL instance | Read replicas | price_history grows fast; archive old rows |
| Redis | Single instance | Redis Cluster | Low memory at this scale |
| Notifications | Synchronous queue drain | Batch processing | Alert dedup prevents fan-out explosion |
| price_history | ~100 events × 5 scrapes/day × 365 days = ~183K rows/year | Manageable | Partition by year if needed |

At 50K users with ~5,000 events and 4 sources scraped every 2h, this is entirely within single-server territory. A €20/month VPS handles the workers; Next.js on Vercel handles the web layer.

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Storing scraped data directly in the events table without deduplication
**What goes wrong:** The same concert appears 3 times because it was scraped on 3 different runs. Users see duplicates. Price history has conflicting records.
**Prevention:** Always compute `sourceId` (source:external_id), use UPSERT with ON CONFLICT on sourceId.

### Anti-Pattern 2: Running Playwright for every source
**What goes wrong:** Memory pressure, slow scrapes, high resource cost.
**Prevention:** Default to Cheerio. Audit each source — most Lithuanian event sites render HTML server-side. Playwright is a last resort.

### Anti-Pattern 3: Updating the current price in-place
**What goes wrong:** Price history is lost. The "price trajectory" feature — the core differentiator — cannot be implemented.
**Prevention:** Append-only `price_history` table. The current price is always the latest row per event.

### Anti-Pattern 4: Coupling the web server to scraping jobs
**What goes wrong:** A memory leak in Playwright crashes the web server. A slow scrape blocks incoming requests.
**Prevention:** Separate worker process. Web server only enqueues jobs; workers execute them.

### Anti-Pattern 5: Building B2B dashboard before public catalog has data
**What goes wrong:** Organizers see an empty platform. Demo fails. No sales.
**Prevention:** Build order is explicit: data first (Phase 2), catalog second (Phase 3), B2B last (Phase 9). B2B's value proposition depends on user traffic existing.

### Anti-Pattern 6: Calendar sync via Google Calendar API OAuth per user
**What goes wrong:** Each user must grant OAuth consent. You must store and refresh tokens. Token expiry breaks calendar sync silently.
**Prevention:** Serve a static .ics subscription URL. Calendar clients poll it. Zero token management. Works with all calendar apps, not just Google.

---

## Sources

- BullMQ documentation (Context7: /taskforcesh/bullmq) — job queues, FlowProducer, cron scheduling, retry patterns
- Playwright documentation (Context7: /microsoft/playwright) — headless scraping, concurrent sessions
- Cheerio documentation (Context7: /cheeriojs/cheerio) — HTML parsing, extract() API
- Prisma documentation (Context7: /prisma/web) — schema design, relations
- ical-generator documentation (Context7: /sebbo2002/ical-generator) — iCal subscription feeds
- Telegraf documentation (Context7: /websites/telegraf_js) — Telegram bot notifications
- Redis documentation (Context7: /redis/docs) — pub/sub, keyspace notifications, rate limiting
- PostgreSQL documentation (Context7: /websites/postgresql) — schema patterns
- Project constraints: /Users/audrius/BEARING/EventSea/.planning/PROJECT.md
