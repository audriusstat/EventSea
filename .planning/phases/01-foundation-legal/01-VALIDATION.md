---
phase: 1
slug: foundation-legal
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-05-25
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vitest 4.1.7 (per-package configs) |
| **Config file** | `packages/db/vitest.config.ts`, `apps/worker/vitest.config.ts`, `apps/web/vitest.config.ts` |
| **Quick run command** | `pnpm --filter <package> test` |
| **Full suite command** | `pnpm turbo test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `pnpm --filter <package> test` (package that was modified)
- **After every plan wave:** Run `pnpm turbo typecheck lint test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| schema-event-id | 01 | 1 | DATA-04 | — | URL hash must be deterministic | unit | `pnpm --filter db test -- event-id` | ❌ W0 | ⬜ pending |
| schema-cascade | 01 | 1 | GDPR-04 | — | Deleting user cascades to watchlist, alert_log | migration test | `pnpm --filter db test -- cascade-delete` | ❌ W0 | ⬜ pending |
| schema-sources | 01 | 1 | DATA-07 | — | sources table has last_successfully_scraped_at + consecutive_failures | migration test | `pnpm --filter db test -- schema` | ❌ W0 | ⬜ pending |
| worker-queues | 02 | 1 | DATA-03 | — | bilietai and tiketa are separate BullMQ Queue instances | unit | `pnpm --filter worker test -- queues` | ❌ W0 | ⬜ pending |
| privacy-route | 03 | 2 | GDPR-01 | — | /privacy returns 200 with policy content | smoke | `pnpm --filter web test -- privacy` | ❌ W0 | ⬜ pending |
| cookie-consent | 03 | 2 | GDPR-02 | Cookie consent bypass | Analytics only fires after acceptedCategory('analytics') | unit | `pnpm --filter web test -- cookie-consent` | ❌ W0 | ⬜ pending |
| legal-memo | 04 | 2 | DATA-06 | tiketa legal exposure | robots.txt findings + EU Directive analysis documented | manual | `cat docs/legal/scraping-memo.md` | ❌ W0 | ⬜ pending |
| article-30 | 04 | 2 | GDPR-03 | — | docs/gdpr/article-30-ropa.md exists with all 8 required fields | manual | File exists check | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `packages/db/src/__tests__/schema.test.ts` — stubs for DATA-04, DATA-07, GDPR-04
- [ ] `apps/worker/src/__tests__/queues.test.ts` — stub for DATA-03
- [ ] `apps/web/app/__tests__/privacy.test.ts` — stub for GDPR-01
- [ ] `apps/web/components/__tests__/CookieConsent.test.tsx` — stub for GDPR-02
- [ ] `packages/db/vitest.config.ts` — shared vitest config for db package
- [ ] `apps/worker/vitest.config.ts` — vitest config for worker package
- [ ] `apps/web/vitest.config.ts` — vitest config for web package
- [ ] `pnpm --filter db add -D vitest @vitest/coverage-v8` — install test framework
- [ ] `pnpm --filter worker add -D vitest` — install test framework
- [ ] `pnpm --filter web add -D vitest @testing-library/react @testing-library/jest-dom` — install test framework

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| robots.txt analysis and EU Database Directive documented in legal memo | DATA-06 | Legal document — no automated assertion meaningful | `cat docs/legal/scraping-memo.md` — check bilietai.lt and tiketa.lt sections exist, EU Directive 96/9/EC section present, partner outreach plan included |
| Article 30 ROPA document has all required fields | GDPR-03 | Document content quality — automated check can only verify existence | Open `docs/gdpr/article-30-ropa.md` — verify: controller identity, processing purposes, data categories, recipients, retention periods, security measures, transfers outside EU, data subject rights |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
