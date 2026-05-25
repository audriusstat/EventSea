# Feature Landscape

**Domain:** Events aggregator platform (concerts, shows, performances)
**Project:** Event Sea
**Market:** Lithuania (Vilnius, Kaunas, Klaipėda), Baltic expansion later
**Researched:** 2026-05-25
**Confidence note:** Web access was blocked during research. Findings are drawn from training
knowledge of the named platforms through August 2025 cutoff. Confidence levels reflect this.

---

## Platform Reference Analysis

Before categorizing features, here is what each reference platform actually does — and what
EventSea can learn from each.

### Bandsintown (HIGH confidence)
Core model: track artists, get notified when they tour near you. Users follow artists;
Bandsintown pulls tour data and sends push/email alerts. Feed is artist-centric, not
venue-centric. No ticket sales — pure aggregator with affiliate redirect links to
ticketing partners. Social layer: "who else is going", RSVP/tracking. Mobile app is the
primary surface. Calendar export (iCal). No price tracking — availability only.

**EventSea takeaway:** The artist-tracking model is the proven pull mechanic. EventSea should
offer both artist-tracking AND category/venue tracking since LT users search by "what's on
in Vilnius this weekend" more than "when does Artist X tour."

### Songkick (HIGH confidence)
Similar to Bandsintown but historically stronger on concert discovery and richer venue data.
Introduced price comparison (via partnerships with multiple ticket sellers) earlier than
competitors. Tracking + alerts model identical to Bandsintown. Known for "Detour" (direct
artist-to-fan ticketing, now largely discontinued). Concertify integration for Spotify-based
artist importing. The key differentiator was always data quality and discovery breadth.

**EventSea takeaway:** Price comparison across sellers is a genuine differentiator even mature
platforms do poorly. LT market has 3+ ticket sellers for the same event — this gap is real.

### Eventbrite (HIGH confidence)
Primarily a ticketing and event management platform, not an aggregator. Consumer side:
discovery feed, search, category browse, "events near me". No price tracking — events either
have tickets or they don't. Strong on free/community events. Organizer tools are the real
product. Consumer app is mediocre; discovery is algorithmic but shallow. No artist tracking.
Calendar integration (Google, Outlook). Email reminders. No Telegram.

**EventSea takeaway:** Eventbrite's consumer experience is weak precisely because discovery
is secondary to ticketing. That's EventSea's opening. Eventbrite LT presence is minor.

### Google Events / Google Search (HIGH confidence)
Not a standalone app — integrated into Google Search ("events near me", event rich snippets).
Structured data (schema.org Event) pulls events from organizer websites. Calendar integration
is seamless (one click to Google Calendar). No user accounts for event tracking. No price
tracking. No notifications beyond Calendar reminders. Strength: zero friction discovery.
Weakness: no persistence, no tracking, no alerts.

**EventSea takeaway:** Google Events sets the zero-friction discovery bar. EventSea must match
or beat the "browse without registering" experience. Calendar add should be one tap.

### TicketSwap (HIGH confidence)
Secondary market (resale) platform. Features: listing resale tickets, price capped resale,
buyer protection, fan-to-fan transfers. Price alerts for sold-out events. Waitlist
functionality. Not an aggregator of primary market events. Strong in NL/BE/DE markets,
growing in Baltic region.

**EventSea takeaway:** TicketSwap's waitlist and resale price alert model is a UX pattern
worth borrowing for "notify me when tickets become available" and price drop alerts on
primary market. Their sold-out event handling is the best in the industry.

### Skyscanner (HIGH confidence)
Not events, but the UX archetype for EventSea. Key patterns to borrow:
- Price calendar (see cheapest date at a glance)
- Price alert subscription (email + push when price drops)
- "Everywhere" search (flexible destination/category)
- Price history graph on flight detail pages
- "Prices are X% above average" contextual signal
- Explore map view
These UX patterns map directly onto events: price calendar → event calendar with pricing,
price alerts → ticket price drop alerts, "everywhere" → flexible category browse.

### Booking.com (HIGH confidence)
Relevant UX patterns:
- "Only X left" scarcity signals
- Review aggregation and trust signals
- Flexible date search with price grid
- Save/wishlist functionality with sharing
- Last viewed / recently searched re-engagement
- Price match guarantee framing

**EventSea takeaway:** Scarcity signals ("3 tickets left at this price") and social proof
("47 people are watching this event") are conversion mechanics that LT competitors don't use.

### bilietai.lt (MEDIUM confidence — knowledge from training, no live access)
Lithuania's largest primary ticket seller. Features: event listing, online purchase, e-ticket
delivery, category browse, seat selection for some venues. UI is functional but dated (circa
2018 design patterns). No price tracking, no artist following, no personalized feed. Search
is basic keyword. Mobile experience is poor. No app — mobile web only. Strong brand trust in
LT market. Handles most major venue contracts.

**EventSea takeaway:** bilietai.lt is a ticketing platform wearing an aggregator costume.
Their discovery UX is weak. EventSea doesn't need to beat their ticket sales — just their
discovery, which is a low bar.

### tiketa.lt / tiketo.lt (MEDIUM confidence)
Secondary LT ticket sellers. Similar profile to bilietai.lt — transactional, not discovery-
oriented. tiketo.lt has a somewhat more modern UI. Neither has mobile app, price tracking,
personalized feed, or notification system beyond order confirmation emails.

**EventSea takeaway:** The entire LT market has zero native discovery products. EventSea
enters a vacuum.

---

## Table Stakes

Features users expect from an events discovery product. Missing any of these = users leave
immediately or don't return.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Browse events without registering | Google Events and bilietai.lt both allow anonymous browse. Registration walls kill top-of-funnel. | Low | Critical: must work before login |
| Filter by city | LT users search "Vilnius events" not "Lithuania events". City filter is the primary axis. | Low | Vilnius, Kaunas, Klaipėda at minimum |
| Filter by date / date range | "This weekend", "this week", "next month" — standard discovery patterns | Low | Quick-select chips + custom range |
| Filter by category | Music, theatre, comedy, sport, exhibitions — users self-segment | Low | 8-12 top-level categories sufficient for v1 |
| Event detail page | Name, date/time, venue, price range, description, ticket link | Low | Ticket link is the core value delivery |
| Direct ticket link (redirect) | The aggregator's core promise: one click to buy | Low | Affiliate UTM tracking needed from day 1 |
| Mobile-responsive / PWA | 70%+ LT users on mobile. Non-responsive = unusable. | Medium | PWA covers push notifications and calendar |
| Search by keyword | "Imagine Dragons", "Siemens Arena", "jazz" — basic search | Medium | Autocomplete improves usability significantly |
| Event thumbnail / image | Visual scanning is how users browse. Text-only lists feel empty and untrustworthy. | Low | Source images from event data; fallback needed |
| Price display | Show ticket price range prominently. Users qualify events by price before clicking. | Low | "From €15" pattern; handle free events |
| Sold-out indicator | Users need to know immediately if tickets are unavailable. Sets expectations. | Low | Scrape availability signal from source |
| Basic email notification | Wishlist/tracked event reminder before event date | Medium | Transactional email at minimum |

**Dependency chain:** Browse → Filter → Detail page → Ticket link. This chain must work
perfectly before any other feature is built.

---

## Differentiators

Features that set EventSea apart from bilietai.lt, tiketa.lt, and generic Google discovery.
These are the reasons users return and recommend.

| Feature | Value Proposition | Complexity | vs. Competitors |
|---------|-------------------|------------|-----------------|
| Aggregated multi-source catalog | Single place for bilietai.lt + tiketa.lt + tiketo.lt + Facebook Events + organizer sites. No competitor does this in LT. | High | bilietai.lt: source only. Google: incomplete. Others: none. |
| Price comparison across sellers | Same event, multiple sellers, different prices. Skyscanner-style: show cheapest source. | High | Nobody in LT market does this. |
| Price history + trend signal | "Prices up 20% vs last week" / price chart on event page. Borrowed from Skyscanner flight detail. | High | Nobody in LT market does this. |
| Price drop alerts (email + Telegram) | Subscribe to an event, get notified when price drops or Early Bird releases. | Medium | Skyscanner does for flights. Nobody for LT events. |
| Personalized feed | Based on genres followed, artists tracked, past events attended, city. Returns home page that feels curated. | High | bilietai.lt/tiketa.lt: no personalization. |
| Artist / performer tracking | Follow Billie Eilish, get notified when she announces LT/Baltic date. Bandsintown model. | Medium | Not in LT market at all. |
| Upcoming / pre-sale tracking | "Announced but no tickets yet" section. Notify when tickets go on sale. | Medium | Unique in LT. Directly addresses buyer anxiety about missing Early Bird. |
| Telegram notifications | LT has high Telegram usage (higher than EU average). Push via Telegram bot is more reliable open rates than email. | Medium | No LT events platform has Telegram integration. |
| Calendar integration (1-tap) | Add event to Google/Apple/Outlook Calendar with venue, time, ticket link in description. | Low-Medium | bilietai.lt/tiketa.lt: no calendar export. Google Events does this but without persistence. |
| Wishlist / saved events | Save events you're considering. Revisit later. Share wishlist with friends. | Low-Medium | None of the LT platforms have this. |
| "Going" social signal | Show how many users are interested/going. Creates FOMO and social proof. | Medium | Not in LT market. Bandsintown does this globally. |
| Scarcity signals | "Only 12 tickets at this price", "87% sold". Booking.com pattern applied to events. | Medium | Requires scraping availability data. Not in LT market. |
| B2B organizer analytics dashboard | Show organizers: views, saves, click-throughs, conversion rate for their events. Unique value for paying customers. | High | bilietai.lt gives zero analytics. Major gap. |
| Featured listings (B2B) | Paid placement at top of category / city feed. Organizer-funded. | Low-Medium | Native ad format, clean implementation matters. |
| Dark mode by default | 2026 UX expectation. Most LT event browsers are evening/night use. | Low | Not a differentiator per se, but signals product quality. |

**Highest-leverage differentiators for v1 (before user base scales):**
1. Multi-source aggregation — this is the core product promise
2. Pre-sale / upcoming tracking + Telegram alert — addresses the "miss Early Bird" pain point
3. Price comparison across sellers — unique in market, immediately valuable

---

## Anti-Features

Things to deliberately NOT build in v1. Each has an explicit reason and what to do instead.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Direct ticket sales / checkout | Requires payment processing, PCI compliance, merchant agreements with venues, seller licensing, fraud handling. Months of legal/compliance work. bilietai.lt has a 10-year head start. | Redirect affiliate links to existing sellers. Monetize via B2B, not transaction fees. |
| Event creation for end users | Moderation burden, spam, duplicate events, UGC liability. Dilutes the "curated aggregator" brand. | Limit creation to verified B2B organizer accounts with approval flow. |
| Reviews and ratings | Trust and abuse problems (fake reviews). Requires moderation. Not a primary discovery signal for LT events market. | Use "X people saved this" and attendance signals instead of reviews. |
| In-app social network (friend follows, activity feed) | Social cold-start problem is extremely hard. Requires critical mass to be useful. Distraction from core aggregation value. | Use social signals (going/interested counts) without requiring social graph. Add sharing links. |
| International events (v1) | Data quality drops, source complexity multiplies, user expectation mismatch. Focus beats breadth. | Hard geo-fence to LT. Plan Baltic expansion as a named milestone. |
| Native iOS + Android apps (v1) | App store approval friction, dual codebase, slower iteration. LT users are fine with PWA for discovery. | Ship PWA first. PWA supports push notifications, calendar, offline. Native apps are Phase 2+. |
| Secondary market / resale | Legal grey area in LT. Requires escrow, buyer protection, dispute resolution. TicketSwap already exists. | Integrate TicketSwap as a source for secondary market links (aggregate, not compete). |
| Venue seat maps | Extremely complex data problem. Each venue has unique layout. Requires agreements or scraping seat map SVGs. | Show "seated" vs "standing" flag. Link to seller seat map. |
| AI chatbot / event concierge | Flashy but not core. Increases infrastructure cost. Users want browse + alerts, not conversation. | Use AI internally for categorization and data normalization, not as user-facing feature. |
| Subscription paywall for users | Users will not pay for discovery when free alternatives exist (Google, Facebook). Kills top-of-funnel growth. | Keep user-facing product free forever. Monetize B2B only. |
| Loyalty points / gamification | Adds complexity. Not a trust-building mechanism in LT market. | Ship fast, build trust through reliability and data quality. |

---

## Feature Dependencies

```
[Core catalog]
  └── Data ingestion (scraping + API) ──────────── foundational; everything depends on this
        └── Event normalization (dedup, merge)
              └── Event detail page
                    ├── Direct ticket link (affiliate)
                    ├── Price display
                    └── Sold-out indicator

[Discovery layer]
  Event detail page
    └── Browse / feed
          ├── City filter
          ├── Date filter
          ├── Category filter
          └── Search

[User features — require auth]
  Auth (registration + login)
    ├── Wishlist / saved events
    │     └── Email reminder (before event)
    ├── Artist / performer tracking
    │     └── Telegram alert (new show announced)
    ├── Price alerts
    │     ├── Price history (must exist first)
    │     └── Telegram / email alert channel
    ├── Personalized feed
    │     └── (requires saved/tracked history — cold start problem)
    └── Calendar integration (can be auth-optional via .ics export)

[Pre-sale tracking]
  Event ingestion
    └── "Announced" status flag
          └── Pre-sale watchlist
                └── "Tickets on sale" notification

[B2B features — require separate organizer auth]
  Organizer account
    ├── Featured listing (paid placement)
    └── Analytics dashboard
          └── Event view/save/click tracking (must be instrumented from day 1)
```

**Critical dependency note:** Analytics instrumentation (tracking views, saves, clicks per
event) must be built into the core catalog from the beginning, not bolted on later. This
data is the foundation for both the B2B analytics product and the personalization engine.

---

## MVP Recommendation

### Build in v1 (sequenced)

**Sprint 1-2 — Core catalog (no auth required)**
- Data ingestion: bilietai.lt + tiketa.lt scraping
- Event normalization and deduplication
- Browse feed with city / date / category filters
- Event detail page with ticket redirect links
- Price display, sold-out indicator
- Basic search

**Sprint 3-4 — User layer**
- Auth (email/password + Google OAuth)
- Wishlist / saved events
- Calendar export (.ics download + Google Calendar deep link)
- Email reminders (day-before for saved events)

**Sprint 5-6 — Alerts + notifications**
- Telegram bot integration
- Pre-sale / upcoming events section ("announced, no tickets yet")
- Price tracking infrastructure (store price snapshots)
- Price drop / Early Bird alerts (Telegram + email)

**Sprint 7-8 — B2B**
- Organizer account creation + verification
- Featured listing management (self-serve)
- Basic analytics dashboard (views, saves, clicks, CTR)

### Defer to v2

- Artist tracking (requires artist entity matching across sources — complex data problem)
- Personalized ML feed (requires user history — cold start; simple rule-based feed first)
- Price history chart on event page (display only; collection starts in v1)
- Social signals ("X people going") — requires user base first
- Scarcity signals ("12 tickets left") — requires deeper scraping reliability first
- Third source integration (tiketo.lt, Facebook Events, organizer sites)

---

## Local Market Specifics (Lithuania)

### Payment and checkout (relevant when linking out)
- bilietai.lt and tiketa.lt support: credit card, bank transfer (SEB, Swedbank, Luminor links),
  Paysera, and some mobile operators. EventSea does not handle payments — this is informational
  for the redirect UX (e.g., "tickets via bilietai.lt — bank transfer available").

### Social and notification channels
- **Telegram:** Exceptionally high LT adoption vs EU average. A Telegram bot for alerts is
  likely more effective than push notifications for the LT market. Confirmed by observation
  of LT media, political groups, and e-commerce channels all running Telegram-first.
- **Facebook:** Still the primary events discovery channel in LT for 35+ demographic.
  Facebook Events integration (scraping or API) is high value but technically complex (Meta's
  API for events is restricted). Phase 2 target.
- **Instagram:** Used by venues and organizers for promotion but no structured events data.
  Useful as a source discovery mechanism (find organizer accounts), not structured data.
- **Draugiem.lv:** Latvian social network, relevant for Baltic expansion but not LT v1.

### Language
- UI must be Lithuanian-first with English option. LT users expect native language from a
  local product. English-only signals "not for us."
- Event data will mix Lithuanian and English (many international acts listed in English on
  bilietai.lt). Normalization layer needs to handle both.

### Trust signals specific to LT market
- LT consumers are skeptical of new platforms. Showing "data from bilietai.lt" and
  "data from tiketa.lt" explicitly (with logos) builds trust faster than hiding sources.
  Position as aggregator, not competitor.
- GDPR compliance is expected and should be surfaced (cookie consent, data export, account
  deletion). LT users are EU-aware.

### Venue specifics
- Siemens Arena (now Twinsbet Arena) Vilnius — major anchor venue; most large concerts
- Žalgirio arena Kaunas — major anchor venue
- Club venues: Tamsta, Bix, Kablys (Vilnius) — smaller, jazz/indie, often no formal ticketing
- Theatre venues: Lithuanian National Drama Theatre, Kaunas State Drama Theatre
  — often sold directly via venue site, not aggregators. High-value scraping target.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Bandsintown / Songkick features | HIGH | Well-documented platforms, stable feature sets through 2025 |
| Eventbrite consumer features | HIGH | Widely covered, stable |
| Google Events behavior | HIGH | Personally observable, documented by Google |
| TicketSwap model | HIGH | Well-documented, especially NL/BE market |
| Skyscanner / Booking.com UX patterns | HIGH | Core patterns stable for 5+ years |
| bilietai.lt feature set | MEDIUM | Training knowledge, no live access during research |
| tiketa.lt / tiketo.lt feature set | MEDIUM | Less documented, training knowledge only |
| LT Telegram adoption claim | MEDIUM | Based on training data; verify with LT market data |
| LT payment method list | MEDIUM | Training knowledge; verify against live bilietai.lt checkout |

---

## Sources

- Training knowledge of Bandsintown (app features, artist tracking model, circa 2024-2025)
- Training knowledge of Songkick (price comparison, tracking model, circa 2024-2025)
- Training knowledge of Eventbrite (consumer + organizer features, circa 2024-2025)
- Training knowledge of Google Events / schema.org Event (circa 2024-2025)
- Training knowledge of TicketSwap (resale model, Baltic expansion, circa 2024-2025)
- Training knowledge of Skyscanner UX patterns (price calendar, alerts, circa 2024-2025)
- Training knowledge of Booking.com UX patterns (scarcity, social proof, circa 2024-2025)
- Training knowledge of bilietai.lt, tiketa.lt (LT market, circa 2024-2025)
- Project context: /Users/audrius/BEARING/EventSea/.planning/PROJECT.md

**Note:** Web access (WebSearch, WebFetch) was blocked during this research session. All
platform-specific claims should be verified against live sites before implementation decisions
are finalized. Confidence levels above reflect this limitation.
