# Event Sea — Research Summary

**Project:** Event Sea — Lithuanian Events Aggregator
**Researched:** 2026-05-25
**Confidence:** HIGH (stack verified via npm/Context7; architecture from well-documented patterns)

---

## Executive Summary

Event Sea is a Skyscanner-for-events for the Lithuanian market. Every existing LT platform (bilietai.lt, tiketa.lt, tiketo.lt) is a transactional ticketing tool — none is a discovery aggregator. That vacuum is the product's entry point and primary moat in v1.

The recommended approach: modular Next.js 16 PWA monorepo with a separate scraping worker process, PostgreSQL via Supabase (Frankfurt/EU), Meilisearch for search, BullMQ for job scheduling, Inngest for durable notification workflows. Runs on ~€30/month infrastructure for MVP.

**Non-negotiable build order:** data ingestion → public catalog → user features → B2B monetization. Any deviation produces a platform with no content or no users.

**Three risks that could kill the project before launch:**
1. Silent scraper failures without a monitoring layer
2. Legal exposure from scraping without documented strategy and source partnerships
3. GDPR non-compliance before user data collection begins

---

## Recommended Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| Next.js | 16.2.6 | Full-stack framework, SSR for SEO, App Router |
| TypeScript | 6.0.3 | End-to-end type safety |
| tRPC | 11.17.0 | Type-safe API layer |
| Zod | 4.4.3 | Schema validation for scraped event data |
| PostgreSQL (Supabase) | pg 15+ | Primary store, Frankfurt EU region (GDPR) |
| Drizzle ORM | 0.45.2 | Lightweight ORM, no binary overhead (vs Prisma) |
| Meilisearch | 0.58.0 | Typo-tolerant faceted search, self-hosted |
| BullMQ | 5.77.3 | Scraping job scheduling + retry |
| Inngest | 4.4.0 | Durable workflows: alert fan-out, email sequences |
| Crawlee + Playwright | 3.16.0 / 1.60.0 | Scraping orchestration + headless browser |
| Cheerio | latest | HTML parsing for static sources (default) |
| Better Auth | 1.6.11 | Auth: Google OAuth, email/password, magic links |
| Resend | 6.12.4 | Transactional email |
| grammY | 1.43.0 | Telegram bot for price alerts |
| ical-generator | 10.2.0 | .ics subscription feed per user |
| Upstash Redis | 1.38.0 | BullMQ broker + API cache (serverless) |
| Stripe | 22.1.1 | B2B organizer subscriptions (SEPA Direct Debit for LT) |
| @serwist/next | 9.5.11 | PWA service worker (next-pwa is dead since 2022) |
| Tailwind CSS | 4.3.0 | Styling |
| shadcn/ui | latest | Component primitives on Radix UI |
| TanStack Query | 5.100.14 | Client-side state |
| TanStack Table | 8.21.3 | B2B analytics tables |

**Infrastructure (MVP ~€30/month):**

| Service | Purpose | Cost |
|---------|---------|------|
| Vercel | Next.js hosting, EU Edge CDN | Free tier |
| Supabase | Postgres + Realtime, Frankfurt GDPR | Free → ~$25/mo |
| Fly.io | Meilisearch + BullMQ workers | ~$7-14/mo |
| Upstash | Redis serverless, eu-west-1 | Free → pay-per-use |
| Cloudflare Images | Image CDN | ~$5/mo |

**Key choices explained:**
- **Better Auth** over NextAuth v5 — Auth.js v5 was beta-quality through 2025; Better Auth is stable, TypeScript-first
- **Drizzle** over Prisma — Prisma's 40MB+ binary causes cold-start latency in Vercel serverless
- **Meilisearch self-hosted** over Algolia — Algolia costs $500/mo at 1M searches; Meilisearch is $7/mo flat
- **iCal .ics subscription** over Google Calendar OAuth — no app review (2-4 weeks), no token expiry, works with all clients
- **@serwist/next** over next-pwa — next-pwa unmaintained since 2022, no App Router support

---

## Table Stakes Features (v1 must-have)

| Feature | Why Non-Negotiable |
|---------|-------------------|
| Browse without registration | Registration walls destroy top-of-funnel |
| City / date / category filters | Primary discovery axis |
| Event detail + ticket redirect | Core value delivery |
| UTM tracking on all outbound links | Revenue instrumentation from day one |
| Mobile-responsive PWA | 70%+ LT users on mobile |
| Keyword search with autocomplete | Meilisearch provides this |
| Price display ("from €X" + freshness) | Users qualify by price before clicking |
| Sold-out indicator | Do not show unavailable as available |
| Multi-source aggregation | This IS the product premise |
| Analytics on every event | Required for B2B analytics product — must instrument from day one |

---

## Key Differentiators

| Feature | Competitive Advantage |
|---------|----------------------|
| Price comparison across sellers | Nobody in LT does this |
| Price history + trend signal | "Prices up 20% vs last week" — unique in LT |
| Price drop alerts (email + Telegram) | Skyscanner for flights applied to events |
| Pre-sale / upcoming events tracking | "Announced, no tickets yet" — kills Early Bird anxiety |
| Telegram notifications | LT Telegram adoption above EU average |
| Wishlist / saved events | No LT competitor has this |
| Calendar integration (.ics export) | One-tap add to any calendar app |
| Scarcity signals | Booking.com pattern, unused in LT events |

**Anti-features (never build):** direct ticket sales, end-user event creation, reviews, subscription paywall for consumers, in-app social network.

---

## Architecture Overview

**Structure:** Modular monorepo, monolith deployment initially, separate worker process for scraping.

**Why separate worker?** Next.js web server on Vercel must not run Playwright — headless browser memory pressure affects API response times. Worker runs on Fly.io.

**Key architectural decision — append-only price_history:**
Every price write is a new row. Never UPDATE a price in-place. The price history feature (core differentiator) is architecturally impossible to accidentally break.

**Scrape pipeline:**
```
BullMQ cron (every 1-2h per source)
  → Fetch worker (Cheerio or Playwright per source)
  → Source Adapter (normalize → Zod validate → canonical EventData)
  → Deduplicator (sha256 fingerprint → DB upsert ON CONFLICT source_id)
  → Price Tracker (compare to latest price_history row, INSERT if changed)
  → Alert Queue (Inngest fan-out to watching users)
  → Notifier (Resend email + grammY Telegram, rate-limited, deduped via alert_log)
```

**Core database tables:** `events` (catalog, never delete), `price_history` (append-only), `users`, `watchlist`, `alert_log` (dedup + anti-spam), `organizers`, `sources` (per-source config + cron schedule).

---

## Critical Risks (Top 5)

### C1: Scraper fragility without monitoring — silent data rot
**What:** A source redesign breaks a scraper silently; events stop appearing; users see stale data.
**Prevention:** Every scraper emits `last_successfully_scraped_at`; canary assertions check result count ±30% of historical average; circuit breaker marks source as degraded after 3 failures.
**Phase:** Build monitoring alongside first scraper (Phase 2), not after.

### C2: Legal exposure from scraping without strategy
**What:** EU Directive 96/9/EC protects structured collections like bilietai.lt's event database.
**Prevention:** Approach bilietai.lt and tiketa.lt as B2B partners before scraping at scale. Document legal rationale per source. LT IP lawyer consultation before public launch.
**Phase:** Phase 1 prerequisite; external consultation required.

### C3: GDPR before first user signup
**What:** Right-to-erasure cascade, Article 30 processing register, per-feature legal basis must precede user data collection. Retrofitting is expensive. VDAI fines up to €20M.
**Prevention:** Privacy policy, consent architecture, cascade delete in initial schema.
**Phase:** Phase 1 architecture.

### C4: Notification spam destroys the re-engagement channel
**What:** A blocked Telegram bot cannot be unblocked. A spam-flagged email domain cannot be un-blacklisted.
**Prevention:** Explicit opt-in per notification type; min 10% price drop threshold; daily frequency caps per user; separate transactional and marketing email domains.
**Phase:** Notification data model in Phase 4 (before first notification in Phase 5).

### C5: Price staleness displayed as current
**What:** User arrives at bilietai.lt expecting EventSea's price, finds higher price — trust destroyed. This is the core differentiator feature.
**Prevention:** Display "Price as of Xh ago" on every card; frame as "from €X" not definitive; separate discovery scrape (daily) from price scrape (every 2-4h for events within 30 days).
**Phase:** Phase 2 (separate cadences from day one).

---

## Recommended Build Order (8 Phases)

| Phase | Name | Rationale | Research Needed? |
|-------|------|-----------|-----------------|
| 1 | Foundation + Legal | Irreversible schema + GDPR + scraping legal strategy | No (legal = external lawyer) |
| 2 | Data Ingestion | Nothing is possible without event data | **YES** — live site analysis per source |
| 3 | Public Catalog | First user-facing surface; anonymous browse is table stakes | No (minor Meilisearch LT test) |
| 4 | Auth + User Layer | Identity unlocks wishlist, alerts, calendar | No |
| 5 | Price Tracking + Alerts | Primary differentiator surfaces | **YES** — Inngest fan-out + grammY prod setup |
| 6 | Calendar Integration | .ics subscription; passive retention, no OAuth complexity | No |
| 7 | B2B Dashboard | Only viable after traffic exists; analytics data collecting since Phase 3 | **YES** — LT B2B VAT/PVM obligations |
| 8 | Source Expansion | tiketo.lt, Eventbrite, Facebook Events, organizer sites | **YES** — per-source audit |

**Phase ordering rationale:**
- Data before UI — catalog (Phase 3) cannot exist without ingestion (Phase 2)
- Traffic before B2B — organizer analytics only valuable when real users exist
- Legal before users — GDPR and scraping strategy precede first user signup
- Notification data model before notifications — frequency caps and opt-in in Phase 4, actual sends in Phase 5
- Price data collection starts in Phase 2 — 3 phases of history accumulate before users see it in Phase 5

---

## Open Questions (require validation)

| Gap | How to Address |
|-----|---------------|
| bilietai.lt HTML structure (static vs SPA?) | Phase 2: live inspection to determine Cheerio vs Playwright |
| tiketa.lt ToS scraping clause | Pre-Phase 2: legal review before first scraper |
| LT Telegram adoption (quantitative) | Validate with LT market data; critical for notification strategy |
| Facebook Events Graph API access | Begin app review 4-6 weeks before Phase 8 |
| LT B2B VAT/PVM obligations | Phase 7 pre-work: Lithuanian accountant consultation |
| Meilisearch Lithuanian diacritics | Phase 3: test ė, ą, š, ž, ų, ū, č, į handling |
| Google Events rich snippet eligibility for LT | Phase 3: verify schema.org Event markup renders |
| LT IP lawyer for scraping strategy | Pre-Phase 2: external consultation required |

---

*Research synthesized: 2026-05-25*
*Phases: 8 | Ready for roadmap: yes*
