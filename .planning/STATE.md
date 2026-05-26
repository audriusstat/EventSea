---
gsd_state_version: 1.0
milestone: v1.0
milestone_name: milestone
current_phase: Phase 1 — Foundation + Legal
status: planned
last_updated: "2026-05-27T00:00:00.000Z"
progress:
  total_phases: 7
  completed_phases: 0
  total_plans: 5
  completed_plans: 0
  percent: 0
---

# Event Sea — Project State

**Last updated:** 2026-05-27
**Project status:** Ready to execute Phase 1
**Current phase:** Phase 1 — Foundation + Legal

---

## Project Reference

**Core value** (from PROJECT.md):
> Atidaro — ir iškart mato visą miestą: visi šios savaitės renginiai, kainos, bilietų nuorodos — be registracijos, be filtravimo, tiesiog viskas vienoje vietoje.

**One-line summary:** Lithuanian events aggregator — like Skyscanner for concerts, theatre, and shows in Vilnius, Kaunas, Klaipėda.

**Monetization:** B2B first (organizers pay for featured listings + analytics); consumers free.

**Stack anchor:** Next.js 16 PWA monorepo + Supabase (Frankfurt) + Fly.io scraping worker + Meilisearch + BullMQ + Inngest + Better Auth + Resend + grammY + Stripe.

---

## Current Position

| Field | Value |
|-------|-------|
| Active phase | Phase 1: Foundation + Legal |
| Active plan | Ready to execute (5 plans, 3 waves) |
| Phase status | Planned — ready to execute |
| Overall progress | 0 / 7 phases complete |

```
Progress: [░░░░░░░░░░░░░░░░░░░░░░░░░░░░] 0%
          Phase 1 of 7
```

---

## Phase Summary

| Phase | Name | Requirements | Status |
|-------|------|--------------|--------|
| 1 | Foundation + Legal | DATA-03,04,06,07 + GDPR-01..04 (8 req) | **Planned** (5 plans, 3 waves) |
| 2 | Data Ingestion | DATA-01,02,05,08 (4 req) | Not started |
| 3 | Public Catalog | CAT-01..11 (11 req) | Not started |
| 4 | Auth + User Layer | AUTH-01..05 + WATCH-01..04 (9 req) | Not started |
| 5 | Price Tracking + Alerts | PRICE-01..04 + NOTIFY-01..06 (10 req) | Not started |
| 6 | Calendar Integration | CAL-01..04 (4 req) | Not started |
| 7 | B2B Dashboard | B2B-01..05 (5 req) | Not started |

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases complete | 0 / 7 |
| Requirements shipped | 0 / 51 |
| Plans written | 5 (Phase 1) |
| Plans verified | 5 (checker passed, 2 passes) |

---

## Accumulated Context

### Key Decisions Locked

| Decision | Rationale |
|----------|-----------|
| 8-phase build order (collapsed to 7 active phases) | Data before UI; legal before users; traffic before B2B — non-negotiable dependency chain |
| Phase 8 (Source Expansion) deferred to v2 | tiketo.lt, Eventbrite, Facebook Events scrapers are v2 scope; 51 v1 requirements fully covered in phases 1-7 |
| GDPR-04 cascade delete in Phase 1 schema | Irreversible — retrofitting cascade deletes after user data exists is expensive and risky |
| Notification data model in Phase 4, sends in Phase 5 | Prevents accidental notification send before opt-in UX exists |
| Price history collection starts Phase 2 | Accumulates 3+ phases of data before users see charts in Phase 5 |
| NOTIFY-04 (unsubscribe link) in Phase 5 | Belongs with first email delivery, not schema design |

### External Dependencies (must resolve before phase starts)

| Dependency | Needed by | Status |
|------------|-----------|--------|
| LT IP lawyer consultation (scraping legal strategy) | Phase 1 completion | Pending |
| bilietai.lt ToS / partnership outreach | Phase 2 start | Pending |
| tiketa.lt ToS / partnership outreach | Phase 2 start | Pending |
| LT accountant (B2B VAT/PVM obligations) | Phase 7 start | Pending |
| Facebook Graph API app review | Phase 8 (v2) | Pending |

### Architecture Constraints

- Scraper worker runs on Fly.io (NOT Vercel) — Playwright memory pressure incompatible with serverless
- `price_history` is append-only — zero UPDATE operations on price columns, ever
- Supabase in Frankfurt (eu-central-1) — GDPR data residency
- UTM tracking on all outbound ticket links from day one (required for B2B analytics product)
- Analytics instrumented in Phase 3 — B2B dashboard in Phase 7 reads data that has been collecting since Phase 3

### Open Risks

| Risk | Mitigation | Phase |
|------|------------|-------|
| Silent scraper failure (stale data rot) | `last_successfully_scraped_at` + canary assertions + circuit breaker | Phase 2 |
| Legal exposure from scraping at scale | Document strategy + partner outreach before scraping | Phase 1 |
| GDPR non-compliance before first signup | Schema cascade + privacy policy + Article 30 register in Phase 1 | Phase 1 |
| Notification spam (Telegram block / email blacklist) | Explicit opt-in + rate limits + dedup in Phase 5 | Phase 5 |
| Price staleness displayed as current | Freshness timestamps on every price display ("prieš Xh") | Phase 3 |

---

## Todos

- [ ] Engage LT IP lawyer for scraping legal memo (pre-Phase 2 prerequisite)
- [ ] Draft outreach email to bilietai.lt for partnership / API access
- [ ] Draft outreach email to tiketa.lt for ToS clarification
- [x] Run `/gsd:plan-phase 1` — 5 plans created, verified, ready to execute

---

## Session Continuity

**To resume work:**

1. Read `.planning/ROADMAP.md` for phase goals and success criteria
2. Read `.planning/phases/01-foundation-legal/01-PLAN-*.md` for execution plans
3. Current phase: Phase 1 — run `/gsd:execute-phase 1`
4. `/clear` first for a fresh context window before execution

**Last action:** Phase 1 planned (2026-05-27). 5 plans in 3 waves, checker passed (2 passes). Ready to execute.

---

*State initialized: 2026-05-25*
