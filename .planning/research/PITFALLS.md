# Domain Pitfalls: Event Sea

**Domain:** Events aggregator platform (Lithuania market)
**Researched:** 2026-05-25
**Confidence note:** WebSearch and WebFetch were unavailable. All findings are from training data
(cutoff Aug 2025) based on documented failure patterns from similar aggregator platforms,
GDPR enforcement actions, Google Calendar API documentation, and scraping case law. Confidence
levels reflect this constraint.

---

## Critical Pitfalls

These mistakes cause rewrites, legal exposure, or complete feature failure.

---

### Pitfall C1: Scraping Brittle by Design — No Resilience Layer

**What goes wrong:** Scrapers are written directly against current HTML structure of
bilietai.lt, tiketa.lt, Facebook Events. A single site redesign silently breaks ingestion.
Events stop appearing. Users see stale data but don't know it. The catalog rots invisibly.

**Why it happens:** Teams write scrapers fast against current structure, skip a monitoring
layer, then discover breakage days later when a user complains or an event was cancelled but
still shows as available.

**Consequences:**
- Sold-out or cancelled events appear as available, destroying user trust
- Price data goes stale; users arrive at ticket site to find different price
- Staff scramble to fix scrapers during high-traffic events (concerts, festivals)
- No single source of truth about when data was last successfully scraped

**Prevention:**
- Every scraper must emit a `last_successfully_scraped_at` timestamp — surface this on event
  cards when data is > 24h old
- Canary checks: after each scraper run, assert that result count is within expected range
  (> 0, within ±30% of historical average). Alert on anomaly, do NOT silently succeed
- Schema validation layer between raw scrape and database write — if schema breaks, fail loudly
- Circuit breaker per source: if a source fails 3 consecutive runs, mark source as degraded
  and notify admin rather than writing partial/garbage data
- Separate scraper health dashboard from main app — ops concern not user concern

**Warning signs:**
- Event count for a source drops suddenly in analytics
- Price on event card doesn't match ticket site price
- Users report "event already happened" but it still shows in catalog

**Phase to address:** Phase 1 (data ingestion) — scraper monitoring must be built alongside
the first scraper, not added later.

---

### Pitfall C2: Scraping Legal Exposure — Terms of Service and Database Rights

**What goes wrong:** Teams assume public web = freely scrapeable. In the EU, database rights
(Directive 96/9/EC) are separate from copyright and specifically protect structured
collections like event listings. bilietai.lt and tiketa.lt databases are almost certainly
protected. Scraping them at scale for commercial purposes creates legal exposure even if
robots.txt doesn't explicitly forbid it.

**Why it happens:** Developers read "public data" and treat it as permission. Lawyers are not
consulted until a cease-and-desist arrives.

**Consequences:**
- C&D from bilietai.lt or tiketa.lt shuts down the primary data source mid-product
- If ignored, litigation — EU database rights cases have been won by plaintiffs
- Bilateral negotiations become adversarial instead of partnership-driven
- Facebook explicitly prohibits scraping in its ToS; automated collection = ban risk

**Prevention:**
- For bilietai.lt / tiketa.lt: approach them as prospective B2B partners before or during
  MVP phase. Frame as traffic referral (you send them buyers, you want their feed). Ask for
  an official data feed or affiliate program. Many ticket platforms have this and prefer it
  over being scraped.
- For Facebook Events: use the official Graph API (Events endpoint). It is rate-limited and
  requires app review for extended permissions, but it is the only legally clean path.
  Facebook aggressively blocks scrapers and has sued scrapers in the past.
- robots.txt is not legally binding but disregarding it weakens your position if sued
- Add attribution and direct links to source — this is the "aggregator as traffic driver"
  argument that strengthens your position

**Warning signs:**
- bilietai.lt or tiketa.lt changes robots.txt to block your user-agent
- IP blocks / CAPTCHA walls appear on source sites
- Receiving a legal inquiry about data collection

**Phase to address:** Phase 0 / pre-Phase 1. Decide legal strategy before writing first
scraper. Document the "public data, traffic referral" rationale. Consult a Lithuanian IP
lawyer before launch.

---

### Pitfall C3: GDPR Cold Start — Collecting Data Before Legal Basis is Established

**What goes wrong:** Team builds personalization features (wishlists, preferences, tracking)
and behavioral analytics before establishing GDPR documentation. Privacy policy is
copy-pasted from a template. No data processing register (Article 30). No DPA with
infrastructure providers. Product ships, users sign up, and only then does someone ask
"do we have a legal basis for storing this?"

**Why it happens:** GDPR feels like a compliance checkbox, not an architecture decision.
Teams treat it as a post-launch task.

**Consequences:**
- In Lithuania, the State Data Protection Inspectorate (VDAI) can fine up to €20M or 4% of
  global annual turnover
- Retroactive data deletion requirements — if you can't demonstrate lawful basis for stored
  behavioral data, you may need to delete it all
- Rebuilding auth and data model around GDPR principles is expensive; building it right first
  is cheap
- Users from other EU countries (if expansion planned) fall under stricter national
  implementations (e.g., Germany)

**Prevention:**
- Before any user registration feature ships: write Privacy Policy, Cookie Policy, and
  Article 30 Register (internal data processing record)
- Legal bases must be decided per data type before collection:
  - Account data (email, name) → Contract (Article 6(1)(b)) — fine
  - Behavioral tracking (clicks, views, wishlist) → Legitimate Interest or Consent — document
    which and why
  - Marketing emails / Telegram notifications → Consent (explicit opt-in required)
  - Price alert subscriptions → Consent
- Cookie consent must be granular (not "accept all" dark patterns) — VDAI has issued guidance
  on this
- Data minimization: collect only what is needed for the stated purpose
- Right to erasure: user account deletion must cascade through all tables — design this into
  the data model from day one, not retrofitted

**Warning signs:**
- Privacy policy written after data is already being collected
- "We'll add consent flows later" said about any feature
- Single analytics pixel/tag added without cookie consent mechanism
- No DPA signed with hosting provider (AWS, Vercel, etc.)

**Phase to address:** Phase 1 (before any user data is stored). GDPR architecture is a
first-class concern, not a final sprint task.

---

### Pitfall C4: OAuth Refresh Token Expiry Silently Breaks Calendar Sync

**What goes wrong:** Google Calendar OAuth refresh tokens expire if the app has not been
verified by Google (unverified apps get 7-day tokens), if the user revokes access, or if
the user changes their Google password. Apple Calendar and Outlook use different auth flows.
When a token expires, calendar sync silently stops — the user gets no notification and
assumes their calendar is up to date when it is not.

**Why it happens:** OAuth is implemented once for the happy path. Token lifecycle edge cases
(expiry, revocation, re-auth) are handled as "later" work that never arrives.

**Consequences:**
- Users miss events they added to calendar via EventSea because sync broke silently
- Support tickets: "my events disappeared from calendar"
- Google app verification process takes weeks — if skipped for MVP, refresh tokens expire
  every 7 days for all users, making calendar sync basically unusable

**Prevention:**
- Google app verification must happen before calendar sync ships to real users, not after.
  The review process takes 2-4 weeks. Plan for this in the phase timeline.
- Implement token refresh logic with exponential backoff and explicit failure states
- When token refresh fails: notify the user immediately (in-app notification + email)
  with a clear "reconnect your calendar" CTA — never silently fail
- Store token refresh timestamp; surface "calendar sync last updated X" in settings
- For MVP: consider offering ICS/iCal export instead of live OAuth sync — it's simpler,
  has no token lifecycle, works with all calendar apps, and avoids Google app review entirely.
  Live OAuth sync can come in a later phase.
- Apple Calendar: uses CalDAV, separate implementation from Google OAuth. Do not assume
  "calendar integration" means one codebase.

**Warning signs:**
- Calendar sync stop rate spikes 7 days after launch (unverified app token expiry)
- No metrics on token refresh failures
- "Connect calendar" button exists but no "calendar sync broken" notification path exists

**Phase to address:** Phase 2 or 3 (when calendar integration is built). ICS export first,
OAuth sync second. Start Google app verification process at least 4 weeks before OAuth
sync feature ships.

---

### Pitfall C5: Notification Spam Destroys Retention Before It Starts

**What goes wrong:** Email and Telegram notifications are built for features (price alerts,
event reminders) but delivery frequency, volume, and relevance are not governed. Users get
daily digest emails they didn't explicitly request, or Telegram bot sends price updates for
every minor price fluctuation. Users unsubscribe or block the bot, and the primary
re-engagement channel is permanently closed.

**Why it happens:** Notifications feel like engagement. Teams measure send volume not
re-engagement rate. "More notifications = more touchpoints" is the wrong mental model.

**Consequences:**
- Telegram bot blocked rate rises; blocked bots cannot be unblocked by the user easily
- Email deliverability score drops (spam complaints → domain reputation damage → all
  emails go to spam, including transactional)
- Unsubscribe rate spikes after first notification campaign
- Email domain potentially blacklisted

**Prevention:**
- Notification defaults must be conservative: nothing sent without explicit user opt-in for
  each notification type (GDPR requirement anyway)
- Price alert threshold: user sets minimum price drop percentage before notifying (default:
  10%). Never notify for price increases unless user explicitly opted in to that too.
- Telegram bot: rate limit to maximum 1 message per event per user per day. Batch daily
  summaries rather than real-time pings unless user sets "urgent" mode for specific events.
- Email: transactional (booking confirmations, password reset) and marketing (weekly digest)
  must use separate sending domains/subdomains — protects transactional deliverability if
  marketing gets flagged
- Implement global notification frequency cap at user level: no more than N notifications
  per day regardless of how many triggers fire
- Build unsubscribe flow that is one-click and does not require login (CAN-SPAM / GDPR
  requirement)
- Monitor bounce rate, complaint rate, and unsubscribe rate from day one — set alert
  thresholds before a problem becomes irreversible

**Warning signs:**
- No per-user notification frequency cap exists in the data model
- Unsubscribe requires logging in
- Email and transactional on same domain/IP
- Telegram message sent for every scrape cycle update rather than batched

**Phase to address:** Phase 1 (notification architecture). The data model for notification
preferences and frequency caps must exist before any notification is sent.

---

## Moderate Pitfalls

These cause significant rework or UX degradation but are recoverable.

---

### Pitfall M1: Cold Start Personalization — Showing Nothing to New Users

**What goes wrong:** Personalized feed is built as the primary interface. New users with no
history see a blank or generic feed, have no sense of value, and churn immediately. The cold
start problem is treated as a later optimization rather than a core design problem.

**Why it happens:** Personalization is exciting to build. The cold start edge case is
deprioritized until retention data reveals it's destroying activation.

**Consequences:**
- New user activation rate is low; most users never return after first session
- "It's empty" is the most common new user complaint
- A/B testing shows personalized feed performs worse than a simple chronological feed for
  users with < 5 interactions

**Prevention:**
- Design for three distinct user states from the beginning:
  - State 0 (no account, no history): show city-based feed sorted by date + popularity.
    This is the primary value proposition ("see everything in your city") and requires zero
    personalization.
  - State 1 (onboarded, no history): show onboarding interest picker (3-5 category choices,
    city selection). Immediately show filtered feed based on selections. This is not
    personalization — it's preference-based filtering.
  - State 2 (has interaction history): blend preference-based filtering with behavioral
    signals (wishlist, views, click-through). Do not attempt ML-based recommendations until
    you have substantial data volume.
- Never hide the unfiltered city feed — it is always the safe fallback
- "Trending in Vilnius this week" requires no user data and provides immediate value

**Warning signs:**
- Personalized feed is the only feed (no city/date browse mode)
- New user sees empty state with no clear next action
- Recommendation algorithm is planned for Phase 1

**Phase to address:** Phase 1 (browsing experience). Cold start solution is part of the
initial product, not a later enhancement.

---

### Pitfall M2: Price Data Staleness — Displaying Wrong Prices

**What goes wrong:** Event prices are scraped once and stored. Ticket prices change
frequently: early bird pricing expires, tier upgrades occur, events sell out and prices
update. Users arrive at ticket site expecting the EventSea price and find a different
(usually higher) price. This is a trust-destroying experience, especially for price-sensitive
users.

**Why it happens:** Scraping price updates is technically harder than scraping event
discovery (requires re-scraping individual event pages not just listings). Teams solve the
easy 80% (discover events) and defer the hard 20% (keep prices current).

**Consequences:**
- User arrives at ticket site expecting €15, finds €25 — blames EventSea
- Price alert feature becomes unreliable — alerts fire for stale data changes, not real ones
- Early bird alerts are the killer feature, but only work if prices are updated frequently

**Prevention:**
- Separate scrape cadences: discovery scrape (find new events) can be daily; price scrape
  (update prices for known events) must be more frequent for events in next 30 days
- Show price freshness on every event card: "Price as of 2h ago" — sets expectation
- Never show price as definitive — always "from €X" with a clear link to check current price
- Prioritize price refresh for events with active wishlists or many views
- Price change detection: store price history with timestamps, detect change > threshold,
  then fire alert — never alert on first scrape (no baseline)

**Warning signs:**
- Price is displayed without a timestamp
- Price scrape cadence = discovery scrape cadence (one job does both)
- No price history table in data model

**Phase to address:** Phase 1 (data model) and Phase 2 (price tracking feature).

---

### Pitfall M3: Organizer Data Ownership and Takedown Disputes

**What goes wrong:** An event organizer discovers their event is listed on EventSea and
objects — either because they want exclusivity with bilietai.lt, they dislike the price
display, or they didn't know their data was being aggregated. They demand removal. Without a
clear policy and fast removal mechanism, this becomes a PR and legal issue.

**Why it happens:** Aggregators assume public listing = permission to aggregate. Some
organizers disagree. EU sui generis database rights can be invoked by platforms, and
organizers may instruct their platforms to block aggregation.

**Consequences:**
- Hostile organizer relationship blocks B2B monetization (an organizer who asked you to
  remove their events will not buy a featured listing)
- Legal demand to remove data creates operational scramble if no removal process exists
- Public social media complaint from a well-known venue or organizer

**Prevention:**
- Build event removal/claim mechanism from day one: organizer can claim an event (email
  verification) and request changes or removal — treat this as a product feature, not a
  support ticket
- "Claim this event" button on every event card is both a user feature and a legal
  protection mechanism
- Clear public data policy page: "We aggregate publicly listed events. If you are the
  organizer and wish to manage your listing, click here." This demonstrates good faith.
- Target the B2B pitch to organizers before they complain: "We list you for free, for €X/mo
  you get featured placement and analytics" converts a potential adversary into a paying
  customer
- Store source attribution for every event — you need to know exactly where each event came
  from to execute a targeted removal

**Warning signs:**
- No organizer contact mechanism exists on event pages
- You cannot identify the scrape source for a specific event record in the database
- No removal/takedown process documented

**Phase to address:** Phase 1 (data model — source attribution), Phase 2 (organizer claim
feature, even if basic).

---

### Pitfall M4: Performance Collapse at List Scale — N+1 and Missing Indexes

**What goes wrong:** Event catalog is fast with 200 events in development. With 5,000 events
across 3 cities, 12 categories, multiple date ranges, and concurrent users, the browse/search
endpoint degrades. The main feed query does a full table scan because compound indexes for
(city, date, category) don't exist. The wishlist query does N+1 lookups for event details.

**Why it happens:** Performance is tested with seed data that is too small. Production data
volume is an order of magnitude larger. Index design requires knowing the query patterns
upfront, which teams skip.

**Consequences:**
- Main feed load time goes from 200ms to 4000ms at scale
- Mobile users with 4G connections on 4000ms endpoints = immediate bounce
- Database CPU spikes during scrape runs (which also write to the same tables being queried)

**Prevention:**
- Design indexes from the primary query patterns, before writing first migration:
  - events(city, start_date, category, status) — compound index for the main browse query
  - events(source_id, external_id) — unique constraint for dedup during scraping
  - wishlists(user_id, event_id) — for user-specific queries
  - price_history(event_id, recorded_at) — for price tracking queries
- Separate read and write paths early: scrape writes go to a staging/ingestion table,
  promoted to main events table after validation — prevents scrape writes from locking
  browse queries
- Paginate everything: never return unbounded event lists — cursor-based pagination from day one
- Test with realistic data volume: seed development database with 10,000+ synthetic events
  before shipping browse feature

**Warning signs:**
- No index planning document exists
- Development database has < 500 events in seed data
- Browse endpoint is not paginated
- Scraper writes directly to the main events table during active user traffic

**Phase to address:** Phase 1 (data model and query design).

---

### Pitfall M5: Timezone Chaos — Lithuanian Events Shown in Wrong Time

**What goes wrong:** Events stored in UTC, displayed without timezone conversion, or stored
with ambiguous timezone data from scraped sources. An event at 20:00 EET (UTC+3 in summer,
UTC+2 in winter) is displayed as 17:00 or 18:00 UTC to users in Vilnius. Calendar export
events show wrong times. Price alert fires at wrong time relative to event start.

**Why it happens:** Timezone is treated as a formatting concern, not a data modeling concern.
Developers in +0 or different timezone zones don't notice during development.

**Consequences:**
- "Concert starts at 20:00" shows "17:00" to Lithuanian user — immediate trust loss
- Calendar event added at wrong time
- Reminder notification fires 3 hours before actual event start

**Prevention:**
- Store all event datetimes in UTC with explicit timezone metadata (Europe/Vilnius)
- Lithuania uses Europe/Vilnius: UTC+2 (EET) in winter, UTC+3 (EEST) in summer — IANA
  timezone, not a fixed offset
- During scraping: if source provides local time without timezone, assume Europe/Vilnius
  and document this assumption
- Display layer always converts from UTC to Europe/Vilnius using IANA rules — never use
  fixed offsets
- Calendar exports (ICS): use VTIMEZONE component with Europe/Vilnius, not a UTC offset
- Test across DST transitions (last Sunday of March, last Sunday of October for Lithuania)

**Warning signs:**
- datetime columns stored as local time without timezone column
- Timezone handling code uses "+3" or "+2" as fixed offset instead of IANA name
- No DST transition tests

**Phase to address:** Phase 1 (data model). Retrofitting timezone correctness is painful.

---

### Pitfall M6: Facebook Events API Deprecation Risk

**What goes wrong:** Facebook Events public feed access has been progressively restricted
since Cambridge Analytica. The Graph API Events endpoint requires app review for extended
permissions, and access can be revoked with little notice when Facebook changes policy.
Building a significant portion of the event catalog on Facebook Events data creates a
fragile dependency.

**Why it happens:** Facebook Events has many Lithuanian cultural events not on ticket
platforms. The data is valuable. Teams integrate it without considering platform risk.

**Consequences:**
- Facebook revokes API access or requires new review → a significant content source
  disappears overnight
- Scraping Facebook (instead of using API) violates ToS and results in IP/account bans
- If catalog is 30% Facebook events, losing that source is a major user-visible gap

**Prevention:**
- Facebook Events should be a supplementary source, not a primary one. Primary sources
  are platforms that have business incentive to keep data flowing (bilietai.lt, tiketa.lt).
- Use official Graph API only — no scraping Facebook HTML
- Complete Facebook app review before relying on it in production
- Monitor Facebook platform changelog for Events API changes
- Design ingestion pipeline so any single source can be disabled without affecting other
  sources

**Warning signs:**
- Facebook Events account for > 25% of catalog events
- No plan for what happens if Facebook API access is revoked
- Using unofficial scraping instead of Graph API

**Phase to address:** Phase 1 (ingestion architecture) — single-source dependency avoidance
is an architectural decision.

---

## Minor Pitfalls

These cause friction but are straightforward to fix.

---

### Pitfall L1: Telegram Bot Rate Limits and User State Management

**What goes wrong:** Telegram Bot API has rate limits (30 messages/second globally, 1
message/second per chat). Sending bulk notifications (e.g., price alert for a popular event
to 1,000 subscribed users) hits rate limits, some messages fail silently, and bot state
machine gets confused if users send unexpected commands mid-flow.

**Prevention:**
- Queue all outbound Telegram messages through a rate-limited worker with retry logic
- Never send Telegram notifications synchronously in a web request
- Use webhook mode (not polling) for production — polling creates reliability issues
- Implement proper conversation state management (e.g., Redis-backed FSM) before building
  multi-step bot flows (subscription setup, preference configuration)

**Phase to address:** Phase 2 (notification infrastructure).

---

### Pitfall L2: PWA Push Notifications Require HTTPS and Service Worker — iOS Limitations

**What goes wrong:** Web Push Notifications require HTTPS (enforced) and a Service Worker.
On iOS, PWA push notifications were only supported from iOS 16.4+ and require the user to
explicitly add the PWA to their home screen first. Users who open the site in Safari without
adding it to home screen cannot receive push notifications. This significantly limits push
notification reach on iOS, which is a major platform in Lithuania.

**Prevention:**
- Do not architect mobile engagement solely around web push — Telegram bot fills this gap
  for iOS users who won't install PWA
- Email remains the most reliable cross-platform notification channel
- When prompting for push notification permission: only prompt after user has demonstrated
  value (e.g., after adding first item to wishlist), never on landing

**Phase to address:** Phase 2 (notification strategy).

---

### Pitfall L3: Lithuanian Language SEO — Content Generation Without Native Review

**What goes wrong:** Event descriptions are scraped in Lithuanian. Automated processing
(summarization, categorization) may produce grammatically incorrect Lithuanian output.
Lithuanian has complex morphology (7 grammatical cases). Machine-generated Lithuanian text
is often recognizably wrong to native speakers and damages perceived quality.

**Prevention:**
- Do not generate Lithuanian text from AI/ML without native review process
- Preserve original event descriptions from source — do not rewrite them
- Category tags and UI elements must be written by a native Lithuanian speaker
- If English content is shown alongside Lithuanian, ensure it is clearly separated

**Phase to address:** Phase 1 (content display).

---

### Pitfall L4: Local Payment Methods for B2B — SEPA and Lithuanian Bank Transfers

**What goes wrong:** B2B monetization (organizer subscriptions) implemented with Stripe/card
only. Lithuanian businesses strongly prefer SEPA bank transfers and may expect invoicing
workflows rather than card-on-file subscriptions. International payment cards add friction
for SMB organizers.

**Prevention:**
- Stripe supports SEPA Direct Debit in EU — enable this from the start for B2B subscriptions
- Offer invoice + bank transfer option for annual plans (manually handled is fine for MVP)
- Lithuanian business tax IDs (PVM kodas) must be collectable for B2B invoicing — required
  for VAT purposes
- Consult Lithuanian accountant on VAT obligations for B2B SaaS subscriptions in LT

**Phase to address:** Phase 3 (B2B monetization).

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|----------------|------------|
| Scraper v1 | No health monitoring; silent failures | Canary assertions and source health dashboard on day one |
| Scraper v1 | ToS / database rights exposure | Legal strategy before first scraper goes to production |
| User auth + profiles | GDPR legal basis not documented | Privacy policy and Article 30 register before user signup ships |
| User auth + profiles | Right to erasure not in data model | Cascade delete design in initial schema |
| Browse / feed | Cold start shows empty state | City+date fallback feed always available, preference picker for onboarding |
| Price tracking | Stale prices shown as current | Price freshness timestamp on every event card; "from €X" framing |
| Calendar integration | Google unverified app token expiry | Start Google app review 4 weeks before OAuth sync ships; ICS export first |
| Calendar integration | Apple Calendar / Outlook separate stacks | Do not underestimate multi-platform calendar work |
| Notifications | Spam + deliverability damage | Explicit opt-in per type; frequency cap; separate transactional domain |
| Notifications | Telegram blocking | Rate limiter queue; conservative send frequency |
| B2B payments | Card-only friction for Lithuanian SMBs | SEPA + invoice option from launch |
| Data model | Timezone errors | UTC + Europe/Vilnius IANA from day one |
| Data model | N+1 + missing indexes | Index design before first migration, realistic seed data volume |
| Organizer relations | No removal mechanism | Claim/remove flow in Phase 2 minimum |
| Facebook Events | API policy change risk | Supplementary source only; official API only |

---

## Sources

**Confidence:** MEDIUM-HIGH for all findings below. These are well-documented failure
patterns across the aggregator/marketplace and event platform domain, consistent across
multiple public post-mortems, engineering blogs, and official documentation known as of
August 2025. WebSearch and WebFetch were unavailable; claims reflect training knowledge.

- Google Calendar API OAuth — official docs (token expiry behavior, unverified app 7-day
  limit, app review process)
- GDPR Article 6 (lawful basis), Article 17 (right to erasure), Article 30 (records of
  processing) — EU regulation text
- VDAI (Valstybinė duomenų apsaugos inspekcija) — Lithuanian data protection authority
  guidance on cookie consent and marketing communications
- EU Directive 96/9/EC — database rights (sui generis protection for structured data
  collections)
- Telegram Bot API documentation — rate limits (30 msg/sec global, 1 msg/sec per chat),
  webhook vs polling
- Facebook Graph API Events — app review requirements, progressive access restrictions
  post-Cambridge Analytica
- IANA timezone database — Europe/Vilnius (EET/EEST, DST transitions)
- iOS 16.4 PWA push notification support — Apple platform release notes
- hiQ v. LinkedIn (9th Cir. 2022) — public web scraping and CFAA; note: US case law,
  EU database rights are stricter and a separate legal question
- Stripe SEPA Direct Debit — EU payment method support documentation
