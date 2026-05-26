---
phase: 01-foundation-legal
plan: 04
type: execute
wave: 2
depends_on:
  - 01-01
files_modified:
  - docs/legal/scraping-memo.md
  - docs/gdpr/article-30-ropa.md
autonomous: true
requirements:
  - DATA-06
  - GDPR-03
must_haves:
  truths:
    - "Scraping legal memo documents bilietai.lt and tiketa.lt robots.txt findings"
    - "Memo records tiketa.lt Disallow: / as a HIGH-risk Phase 2 legal gate"
    - "Memo includes EU Directive 96/9/EC analysis and a partner outreach plan"
    - "Article 30 ROPA contains all 8 mandatory controller fields"
  artifacts:
    - path: "docs/legal/scraping-memo.md"
      provides: "Scraping legal strategy (DATA-06)"
      contains: "Disallow: /"
    - path: "docs/gdpr/article-30-ropa.md"
      provides: "GDPR Article 30 processing register (GDPR-03)"
      contains: "Article 30"
  key_links:
    - from: "scraping-memo.md"
      to: "Phase 2 tiketa.lt scraper"
      via: "documented legal gate"
      pattern: "legal gate"
---

<objective>
Author the two Phase 1 legal/compliance documents: the scraping legal memo (robots.txt findings per source, EU Database Directive 96/9/EC analysis, partner outreach plan, and the tiketa.lt legal gate) and the GDPR Article 30 processing register (ROPA) with all 8 mandatory controller fields. Implements DATA-06 and GDPR-03.

Purpose: Establish documented legal cover before any scraping fires (Phase 2) and the GDPR compliance baseline before any user signs up. The tiketa.lt `Disallow: /` finding must be recorded as a hard Phase 2 gate. Implements D-14 (self-authored best-effort memo; LT lawyer review is pre-Phase 2, not a Phase 1 blocker).
Output: docs/legal/scraping-memo.md and docs/gdpr/article-30-ropa.md.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
@.planning/phases/01-foundation-legal/01-RESEARCH.md
@.planning/phases/01-foundation-legal/01-CONTEXT.md
@CLAUDE.md
</context>

<tasks>

<task type="auto">
  <name>Task 1: Scraping legal memo (DATA-06)</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Scraping Legal Analysis: bilietai.lt robots.txt, tiketa.lt robots.txt, EU Database Directive 96/9/EC Key Points, Partner Outreach Plan; Pitfall 6 tiketa legal gate)
    - .planning/phases/01-foundation-legal/01-CONTEXT.md (D-14 self-authored best-effort memo)
    - .planning/STATE.md (External Dependencies: LT IP lawyer, bilietai/tiketa outreach; Open Risks: legal exposure)
    - CLAUDE.md (Scraping section — robots.txt, User-Agent with contact, rate limits)
  </read_first>
  <action>
    Create docs/legal/scraping-memo.md with these sections, drawn directly from RESEARCH.md Scraping Legal Analysis:
    1. bilietai.lt robots.txt findings — only /search and /beta/ blocked for `*`; sitemap published; risk level LOW-MEDIUM; sitemap is lowest-risk consumption path; ToS review still required before Phase 2.
    2. tiketa.lt robots.txt findings — `Disallow: /` for all crawlers; risk level HIGH. State explicitly that the Phase 2 tiketa.lt scraper is BLOCKED behind a legal gate: either (a) partnership/API agreement, or (b) written LT IP lawyer opinion. Use the literal phrase "legal gate".
    3. EU Database Directive 96/9/EC analysis — sui generis right, 2021 ECJ ruling (substantial extraction + significant detriment), Article 8(1) insubstantial-parts exception, individual event facts likely not copyright-protected, safe-harbor pattern (link to source + summary data + deep-link, no full reproduction).
    4. Partner outreach plan — draft outreach email principles for bilietai.lt and tiketa.lt (aggregator not competitor, discovery increases their sales, UTM-attributed "buy on X", ask about API/feed, company registration details).
    5. Rate-limit / well-behaved-crawler policy — min 1s between requests, identifying User-Agent with contact info (per CLAUDE.md).
    6. Pre-Phase 2 prerequisites — engage LT IP lawyer; re-verify tiketa.lt/robots.txt before Phase 2 (Assumption A5); note bilietai.lt scraper may proceed concurrently with tiketa lawyer review (RESEARCH.md Open Question 4).
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && test -f docs/legal/scraping-memo.md && grep -q 'bilietai.lt' docs/legal/scraping-memo.md && grep -q 'Disallow: /' docs/legal/scraping-memo.md && grep -q '96/9/EC' docs/legal/scraping-memo.md && grep -qi 'legal gate' docs/legal/scraping-memo.md && grep -qi 'outreach' docs/legal/scraping-memo.md && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - docs/legal/scraping-memo.md exists
    - Contains a bilietai.lt section noting only /search and /beta/ are blocked
    - Contains a tiketa.lt section with `Disallow: /` and the literal phrase "legal gate"
    - Contains `96/9/EC` (EU Database Directive analysis)
    - Contains a partner outreach plan section (matches `outreach`, case-insensitive)
    - Documents min-1s rate limit and identifying User-Agent policy
  </acceptance_criteria>
  <done>Scraping legal memo exists with bilietai.lt + tiketa.lt robots.txt findings, EU Directive analysis, outreach plan, and the tiketa.lt Phase 2 legal gate.</done>
</task>

<task type="auto">
  <name>Task 2: GDPR Article 30 processing register / ROPA (GDPR-03)</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (GDPR Compliance Notes: Article 30 Processing Register — 8 minimum fields)
    - .planning/phases/01-foundation-legal/01-CONTEXT.md (D-10 Supabase Frankfurt EU residency, D-12 cascade delete)
    - CLAUDE.md (GDPR & Privacy section — minimal data, account deletion, no third-party sharing of alert emails)
    - docs/legal/scraping-memo.md (created in Task 1 — keep docs/ structure consistent)
  </read_first>
  <action>
    Create docs/gdpr/article-30-ropa.md as the GDPR Article 30 processing register. Include all 8 mandatory controller fields per RESEARCH.md GDPR Compliance Notes, each as a labeled section:
    1. Name and contact details of controller (Event Sea / company registration placeholder + data-protection contact email).
    2. Purposes of processing (event aggregation, user watchlist, price alerts).
    3. Categories of data subjects (website visitors, registered users, B2B organizers).
    4. Categories of personal data (email, preferences, watch history).
    5. Recipients / categories of recipients (Resend for email, Telegram, Supabase).
    6. International transfers (Supabase Frankfurt = EU/EEA; no transfer outside EEA — D-10).
    7. Retention periods (user data until account deletion + 30 days; price_history retained indefinitely as aggregate non-personal data).
    8. Security measures (TLS in transit, encryption at rest via Supabase, access control, GDPR-04 cascade delete on account deletion).
    State at the top that this register is maintained in-repo and must be kept up to date and available to the supervisory authority (Lithuania: VDAI). Title the document with "Article 30".
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && test -f docs/gdpr/article-30-ropa.md && grep -q 'Article 30' docs/gdpr/article-30-ropa.md && grep -qi 'retention' docs/gdpr/article-30-ropa.md && grep -qi 'controller' docs/gdpr/article-30-ropa.md && grep -qi 'Supabase' docs/gdpr/article-30-ropa.md && grep -qi 'security measures' docs/gdpr/article-30-ropa.md && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - docs/gdpr/article-30-ropa.md exists and the title contains `Article 30`
    - All 8 mandatory fields present: controller identity, processing purposes, data subject categories, personal data categories, recipients, international transfers, retention periods, security measures
    - International-transfers section states Supabase Frankfurt is EEA / no transfer outside EEA
    - Retention section states user data deleted on account deletion (+30 days) and price_history retained indefinitely
  </acceptance_criteria>
  <done>Article 30 ROPA exists with all 8 mandatory controller fields and EU-residency + cascade-delete statements.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| Event Sea aggregator → third-party sources | Scraping crosses a legal/repudiation boundary governed by robots.txt + EU law |
| controller → data subjects / supervisory authority | GDPR accountability boundary — ROPA is the evidentiary record |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-04-01 | Repudiation / Legal | tiketa.lt scraping without authorization | mitigate | Memo records `Disallow: /` and a HARD Phase 2 legal gate (partnership or LT lawyer opinion) before any tiketa scraper (Pitfall 6) |
| T-04-02 | Repudiation / Legal | bilietai.lt bulk extraction under DB Directive | mitigate | Safe-harbor pattern documented (summary data + deep-link, no full reproduction); ToS review gate before Phase 2 |
| T-04-03 | Compliance | Missing Article 30 register on regulator request | mitigate | ROPA created in-repo with all 8 fields; statement that it must be kept current and available to VDAI |
| T-04-04 | Information Disclosure | International transfer of personal data outside EEA | accept | Supabase Frankfurt (eu-central-1) keeps data in EEA; no transfer outside — documented in ROPA (D-10) |
</threat_model>

<verification>
- docs/legal/scraping-memo.md exists with both sources, EU Directive, outreach, legal gate
- docs/gdpr/article-30-ropa.md exists with all 8 mandatory fields
- tiketa.lt legal gate documented as Phase 2 blocker
</verification>

<success_criteria>
GDPR Article 30 processing register created; scraping legal memo written with per-source robots.txt policy, EU Directive 96/9/EC analysis, and partner outreach plan; tiketa.lt legal gate recorded. Maps to Phase 1 Success Criteria 4 (ROPA) and 5 (legal memo).
</success_criteria>

<output>
Create `.planning/phases/01-foundation-legal/01-04-SUMMARY.md` when done.
</output>
