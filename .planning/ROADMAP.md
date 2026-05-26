# Event Sea — Project Roadmap

**Project:** Event Sea (Renginių/Įvykių jūra)
**Mode:** mvp (Vertical MVP — each phase delivers end-to-end user value)
**Granularity:** standard
**Requirements mapped:** 51/51
**Generated:** 2026-05-25

---

## Phases

- [ ] **Phase 1: Foundation + Legal** — Project scaffolding, GDPR-compliant schema, scraping legal strategy
- [ ] **Phase 2: Data Ingestion** — bilietai.lt + tiketa.lt scrapers with price history and resilience monitoring
- [ ] **Phase 3: Public Catalog** — Anonymous browse, search, filters, event detail pages, SEO
- [ ] **Phase 4: Auth + User Layer** — Registration, sessions, wishlist, pre-sale + price-drop watch data model
- [ ] **Phase 5: Price Tracking + Alerts** — Price history display, email/Telegram notification delivery
- [ ] **Phase 6: Calendar Integration** — .ics download, personal subscription feed, pre-event email reminders
- [ ] **Phase 7: B2B Dashboard** — Organizer accounts, featured listings, analytics, Stripe payments

---

## Phase Details

### Phase 1: Foundation + Legal
**Mode:** mvp
**Goal:** The project has a GDPR-compliant database schema, documented scraping legal strategy, and infrastructure capable of running the full system — before a single user signup or scrape fires.
**Depends on:** Nothing (first phase)
**Requirements:** DATA-03, DATA-04, DATA-06, DATA-07, GDPR-01, GDPR-02, GDPR-03, GDPR-04
**Success Criteria** (what must be TRUE):
  1. Next.js 16 + TypeScript monorepo runs locally and deploys to Vercel with a passing CI pipeline
  2. PostgreSQL schema exists with `events`, `price_history`, `users`, `watchlist`, `alert_log`, `organizers`, `sources` tables — cascade-delete wiring verified by migration test
  3. Scraper worker skeleton on Fly.io runs BullMQ jobs; each source record tracks `last_successfully_scraped_at` and circuit-breaker state
  4. GDPR Article 30 processing register document created; privacy policy page live at `/privacy`; cookie consent architecture defined
  5. Scraping legal memo written (robots.txt policy per source, EU Directive 96/9/EC analysis, partner outreach plan for bilietai.lt and tiketa.lt)
**Plans:** 5 plans (3 waves)
- [ ] 01-PLAN-monorepo-scaffold.md — Turborepo + pnpm scaffold, Docker Compose, CI/CD, Vitest (Wave 1)
- [ ] 01-PLAN-db-schema.md — Drizzle 7-table schema, URL-hash IDs, cascade delete, [BLOCKING] migration (Wave 2)
- [ ] 01-PLAN-worker-skeleton.md — BullMQ per-source queues, worker boots against Redis (Wave 2)
- [ ] 01-PLAN-legal-gdpr-docs.md — scraping legal memo + Article 30 ROPA (Wave 2)
- [ ] 01-PLAN-web-privacy-consent.md — /privacy route, cookie consent, home DB read (Wave 3)

### Phase 2: Data Ingestion
**Mode:** mvp
**Goal:** The system reliably collects, deduplicates, and stores events from bilietai.lt and tiketa.lt every 1-2 hours, with price history recorded from day one and observable scraper health.
**Depends on:** Phase 1
**Requirements:** DATA-01, DATA-02, DATA-05, DATA-08
**Success Criteria** (what must be TRUE):
  1. bilietai.lt scraper runs on BullMQ cron every 1-2h; new and updated events appear in the `events` table within 2h of source publication
  2. tiketa.lt scraper runs independently — a failure in one scraper does not block or degrade the other
  3. Every price change triggers a new append-only row in `price_history`; zero in-place price updates exist in the schema
  4. Events announced before ticket sale begin ("bilietai dar neparduodami") are ingested with `pre_sale` status flag
  5. Canary assertions check per-source result counts against historical averages; source marked `degraded` after 3 consecutive failures; monitoring dashboard or log observable by developer
**Plans:** TBD

### Phase 3: Public Catalog
**Mode:** mvp
**Goal:** Any visitor can open Event Sea and immediately see all current events with filters, search, and direct ticket links — without creating an account.
**Depends on:** Phase 2
**Requirements:** CAT-01, CAT-02, CAT-03, CAT-04, CAT-05, CAT-06, CAT-07, CAT-08, CAT-09, CAT-10, CAT-11
**Success Criteria** (what must be TRUE):
  1. Visitor lands on homepage and sees a browsable event list with no registration prompt; city / date / category filters work without login
  2. Every event has a dedicated page showing name, date, venue, price ("nuo €X, prieš 2 val."), description, and a "Pirkti bilietus" button with UTM tracking that redirects to the source seller
  3. Sold-out events are visibly labeled "Išparduota" and not surfaced as available in default listings
  4. Keyword search with Lithuanian diacritic support and autocomplete (Meilisearch) returns relevant results within 200ms
  5. Event pages render schema.org Event structured data; view, save, and click-through events are instrumented and flowing to the analytics store
**Plans:** TBD
**UI hint**: yes

### Phase 4: Auth + User Layer
**Mode:** mvp
**Goal:** Users can create an account and build a personal watchlist of events, establishing the data model for all future personalization and notification features.
**Depends on:** Phase 3
**Requirements:** AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, WATCH-01, WATCH-02, WATCH-03, WATCH-04
**Success Criteria** (what must be TRUE):
  1. User can register and log in via Google OAuth or passwordless magic link; session persists up to 30 days across browser restarts
  2. User can log out from any page; session is invalidated server-side
  3. User can delete their account — all personal data (profile, watchlist, alert preferences) is cascade-deleted; action is irreversible and confirmed in UI
  4. Logged-in user can save events to wishlist, view their saved list, and remove events from it
  5. Logged-in user can mark a pre-sale event as "sekti" (watch for ticket release) and mark an event for price-drop watching; both watches are persisted in `watchlist` with type flag — no notifications sent yet (that is Phase 5)
**Plans:** TBD
**UI hint**: yes

### Phase 5: Price Tracking + Alerts
**Mode:** mvp
**Goal:** Users see price history on every event page and receive timely, non-spammy email and Telegram notifications when watched events go on sale or prices drop.
**Depends on:** Phase 4
**Requirements:** PRICE-01, PRICE-02, PRICE-03, PRICE-04, NOTIFY-01, NOTIFY-02, NOTIFY-03, NOTIFY-04, NOTIFY-05, NOTIFY-06
**Success Criteria** (what must be TRUE):
  1. Every event detail page shows a price history chart or textual trend ("kaina krito 15% per 2 sav.") derived from `price_history` rows collected since Phase 2
  2. User can set a percentage price-drop threshold on a watched event; when the threshold is crossed, an alert fires within the next scrape cycle
  3. Email alerts are delivered via Resend with a working one-click unsubscribe link; Telegram alerts are delivered via grammY bot
  4. User can independently enable/disable email and Telegram per notification type (price drop, pre-sale open, event reminder) through a preferences screen
  5. No user receives the same alert twice within 24h for the same event; no user receives more than 3 notifications per day across all types combined
**Plans:** TBD
**UI hint**: yes

### Phase 6: Calendar Integration
**Mode:** mvp
**Goal:** Users can add any event to their calendar app with one tap, and logged-in users get a live-sync feed that keeps their wishlist in sync with their calendar automatically.
**Depends on:** Phase 4
**Requirements:** CAL-01, CAL-02, CAL-03, CAL-04
**Success Criteria** (what must be TRUE):
  1. Any visitor can click "Pridėti į kalendorių" on an event page and download a valid .ics file that imports correctly into Google Calendar, Apple Calendar, and Outlook
  2. Logged-in user has a personal .ics subscription URL; adding it to any calendar app auto-syncs their wishlist — new saves appear, removed saves disappear
  3. All iCal entries use `Europe/Vilnius` IANA timezone; events display at the correct local time in all calendar clients
  4. Logged-in user can opt in to receive an email reminder 24h and 1h before each wishlisted event; reminders are sent via the notification pipeline built in Phase 5
**Plans:** TBD

### Phase 7: B2B Dashboard
**Mode:** mvp
**Goal:** Event organizers can claim their events, enhance their listings, pay for featured placement, and see how their events perform — providing the project's first revenue stream.
**Depends on:** Phase 3 (traffic + analytics data), Phase 4 (auth foundation)
**Requirements:** B2B-01, B2B-02, B2B-03, B2B-04, B2B-05
**Success Criteria** (what must be TRUE):
  1. Organizer can register a separate organizer account (distinct from consumer account) and access a dashboard route
  2. Organizer can search the catalog and "claim" their events; claimed events are linked to their organizer profile pending any verification step
  3. Organizer can add or edit photos, description, and supplementary info on their claimed events; changes are visibly reflected on public event pages
  4. Organizer can pay for a featured listing (top-of-catalog placement) via Stripe with SEPA Direct Debit; payment creates a timed boost record
  5. Organizer dashboard displays view count, wishlist-save count, and outbound ticket-click count for each of their events, drawn from the analytics instrumented in Phase 3
**Plans:** TBD
**UI hint**: yes

---

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation + Legal | 0/? | Not started | - |
| 2. Data Ingestion | 0/? | Not started | - |
| 3. Public Catalog | 0/? | Not started | - |
| 4. Auth + User Layer | 0/? | Not started | - |
| 5. Price Tracking + Alerts | 0/? | Not started | - |
| 6. Calendar Integration | 0/? | Not started | - |
| 7. B2B Dashboard | 0/? | Not started | - |

---

## Coverage Map

| Requirement | Phase | Category |
|-------------|-------|----------|
| DATA-01 | Phase 2 | Data Ingestion |
| DATA-02 | Phase 2 | Data Ingestion |
| DATA-03 | Phase 1 | Foundation + Legal |
| DATA-04 | Phase 1 | Foundation + Legal |
| DATA-05 | Phase 2 | Data Ingestion |
| DATA-06 | Phase 1 | Foundation + Legal |
| DATA-07 | Phase 1 | Foundation + Legal |
| DATA-08 | Phase 2 | Data Ingestion |
| CAT-01 | Phase 3 | Public Catalog |
| CAT-02 | Phase 3 | Public Catalog |
| CAT-03 | Phase 3 | Public Catalog |
| CAT-04 | Phase 3 | Public Catalog |
| CAT-05 | Phase 3 | Public Catalog |
| CAT-06 | Phase 3 | Public Catalog |
| CAT-07 | Phase 3 | Public Catalog |
| CAT-08 | Phase 3 | Public Catalog |
| CAT-09 | Phase 3 | Public Catalog |
| CAT-10 | Phase 3 | Public Catalog |
| CAT-11 | Phase 3 | Public Catalog |
| AUTH-01 | Phase 4 | Auth + User Layer |
| AUTH-02 | Phase 4 | Auth + User Layer |
| AUTH-03 | Phase 4 | Auth + User Layer |
| AUTH-04 | Phase 4 | Auth + User Layer |
| AUTH-05 | Phase 4 | Auth + User Layer |
| WATCH-01 | Phase 4 | Auth + User Layer |
| WATCH-02 | Phase 4 | Auth + User Layer |
| WATCH-03 | Phase 4 | Auth + User Layer |
| WATCH-04 | Phase 4 | Auth + User Layer |
| PRICE-01 | Phase 5 | Price Tracking + Alerts |
| PRICE-02 | Phase 5 | Price Tracking + Alerts |
| PRICE-03 | Phase 5 | Price Tracking + Alerts |
| PRICE-04 | Phase 5 | Price Tracking + Alerts |
| NOTIFY-01 | Phase 5 | Price Tracking + Alerts |
| NOTIFY-02 | Phase 5 | Price Tracking + Alerts |
| NOTIFY-03 | Phase 5 | Price Tracking + Alerts |
| NOTIFY-04 | Phase 5 | Price Tracking + Alerts |
| NOTIFY-05 | Phase 5 | Price Tracking + Alerts |
| NOTIFY-06 | Phase 5 | Price Tracking + Alerts |
| CAL-01 | Phase 6 | Calendar Integration |
| CAL-02 | Phase 6 | Calendar Integration |
| CAL-03 | Phase 6 | Calendar Integration |
| CAL-04 | Phase 6 | Calendar Integration |
| B2B-01 | Phase 7 | B2B Dashboard |
| B2B-02 | Phase 7 | B2B Dashboard |
| B2B-03 | Phase 7 | B2B Dashboard |
| B2B-04 | Phase 7 | B2B Dashboard |
| B2B-05 | Phase 7 | B2B Dashboard |
| GDPR-01 | Phase 1 | Foundation + Legal |
| GDPR-02 | Phase 1 | Foundation + Legal |
| GDPR-03 | Phase 1 | Foundation + Legal |
| GDPR-04 | Phase 1 | Foundation + Legal |

**Total mapped: 51/51**

> Note: REQUIREMENTS.md states "50 total" but lists 51 distinct IDs (DATA 8 + CAT 11 + AUTH 5 + WATCH 4 + PRICE 4 + NOTIFY 6 + CAL 4 + B2B 5 + GDPR 4 = 51). All 51 are mapped.

---

*Roadmap generated: 2026-05-25*
*Next: `/gsd:plan-phase 1`*
