# Event Sea — CLAUDE.md

Project instruction file for AI coding agents working on Event Sea.

## Project Overview

Event Sea is a Lithuanian events aggregator — like Skyscanner, but for concerts, shows, and performances. It aggregates events from multiple sources (bilietai.lt, tiketa.lt, Facebook Events, etc.) and presents them in one place with direct ticket purchase links.

**Core Value:** Open the app — see the entire city. All events this week, prices, ticket links — no registration, no filtering required.

**Tech stack:** Mobile-first (PWA/React Native), Next.js + TypeScript, PostgreSQL, Redis, scraping infrastructure.

**Business model:** B2B first — event organizers pay for featured listings and analytics. Users get everything free.

## Karpathy Coding Principles

### 1. Think Before Coding

**Don't assume. Surface tradeoffs. Ask when unclear.**

Before implementing:
- State assumptions explicitly. If uncertain, say so.
- If multiple approaches exist, name the tradeoff — don't pick silently.
- If a simpler approach exists, propose it.
- If something is unclear, stop and ask.

**Critical for this project:**
- Scraping code: always check `robots.txt` before writing a scraper. Assume targets will change their HTML.
- Calendar APIs: OAuth flows are complex. Confirm the exact scope and redirect URI before building.
- Price tracking: confirm what "price change" means — absolute value, percentage, or both.

### 2. Simplicity First

**Minimum code that solves the problem.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

**For this project specifically:**
- Scrapers: one function per source, no clever meta-scrapers. A simple `cheerio` parser beats a dynamic Playwright crawler unless dynamic rendering is proven necessary.
- Database queries: raw SQL or a thin ORM (Drizzle/Prisma). No complex query builders for simple reads.
- Notifications: use existing services (Resend for email, Telegram Bot API). Don't build custom notification infrastructure.

### 3. Surgical Changes

**Touch only what you must.**

When editing existing code:
- Don't improve adjacent code unless asked.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated issues, mention them — don't fix them.

When your changes create orphans:
- Remove imports/variables/functions YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

For every task:
- "Add event scraper" → "Scraper fetches events, stores in DB, handles 404/rate-limits, has tests for main cases"
- "Add price tracking" → "Price history stored, change detected, alert triggered within 1 job cycle"
- "Add calendar integration" → "User can add event to Google Calendar in < 3 taps, event appears within 60s"

## Project-Specific Rules

### Scraping

- Always check `robots.txt` before scraping a new source.
- Use `User-Agent` headers that identify the bot (include contact info).
- Respect rate limits — add delays between requests (min 1s between pages).
- Cache scraped HTML before parsing — makes debugging much easier.
- Store raw scraped data alongside normalized data — parsing logic will change.
- Never store personally identifiable information from scraped pages.

### Data Quality

- All event dates must be stored in UTC. Display in user's local timezone.
- Event "price" is the minimum available price unless otherwise specified.
- Mark events as "unverified" until human review if confidence < threshold.
- Stale events (last seen > 48h ago) should be flagged, not deleted.

### Calendar Integration

- Implement Google Calendar first (largest LT user base), then Apple/Outlook.
- Always use calendar events with `organizer` field populated.
- Calendar events must include: title, start/end time, location, ticket URL, description.
- Timezone: Europe/Vilnius by default unless user's calendar timezone differs.

### Notifications

- Email via Resend (or similar). Never use raw SMTP.
- Telegram via official Bot API — no third-party wrappers.
- Always include unsubscribe link in every email.
- Notification deduplication: don't send same alert twice within 24h.
- Rate limit: max 1 email/day per user for price alerts.

### GDPR & Privacy

- Minimal data collection — only what's needed.
- Users can delete their account and all data.
- Price alert email addresses are never shared with third parties.
- Log what you store and why in comments near data models.

### Database

- PostgreSQL for primary data.
- Redis for job queues (BullMQ) and caching (hot events).
- Event ID must be stable across scrapes (use source URL hash, not scrape order).
- Price history: never delete old prices — append only.

### B2B Organizer Dashboard

- Organizers authenticate separately from regular users.
- Organizer can only see analytics for their own events.
- Featured listing = same data model as regular event + `featured: true` flag + priority ordering.

## GSD Workflow

This project uses the GSD (Get Shit Done) planning system.

- Planning docs: `.planning/`
- Current state: `.planning/STATE.md`
- Requirements: `.planning/REQUIREMENTS.md`
- Roadmap: `.planning/ROADMAP.md`

**Before starting any phase:** Read `.planning/STATE.md` and the current phase's `PLAN.md`.

**After completing a phase:** Update `.planning/STATE.md` with what was completed.

**Commits:** Use conventional commits (`feat:`, `fix:`, `chore:`, `docs:`). One logical change per commit.

## Code Style

- **Language:** TypeScript everywhere (strict mode).
- **Formatting:** Prettier with default config.
- **Linting:** ESLint with TypeScript rules.
- **Testing:** Vitest for unit tests, Playwright for E2E.
- **API:** REST for external clients, tRPC for internal server-client communication.
- **Naming:** camelCase for variables/functions, PascalCase for components/types, SCREAMING_SNAKE_CASE for constants.
- **File structure:** feature-based (`/features/events/`, `/features/scraping/`, `/features/notifications/`).

## Environment Variables

Always use `.env.local` for secrets locally. Never commit secrets. Document all required env vars in `.env.example`.

Required:
```
DATABASE_URL=
REDIS_URL=
RESEND_API_KEY=
TELEGRAM_BOT_TOKEN=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
NEXTAUTH_SECRET=
NEXTAUTH_URL=
```
