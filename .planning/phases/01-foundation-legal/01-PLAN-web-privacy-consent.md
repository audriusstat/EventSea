---
phase: 01-foundation-legal
plan: 05
type: execute
wave: 3
depends_on:
  - 01-01
  - 01-02
files_modified:
  - apps/web/next.config.ts
  - apps/web/tsconfig.json
  - apps/web/app/layout.tsx
  - apps/web/app/page.tsx
  - apps/web/app/privacy/page.tsx
  - apps/web/components/CookieConsent.tsx
  - apps/web/app/__tests__/privacy.test.ts
  - apps/web/components/__tests__/CookieConsent.test.tsx
autonomous: true
requirements:
  - GDPR-01
  - GDPR-02
must_haves:
  truths:
    - "/privacy route renders the privacy policy content and returns 200"
    - "Every page shows a footer link to /privacy (privacy policy accessible on all pages)"
    - "Cookie consent banner renders and analytics fires only after consent is granted"
    - "Home page reads the live events table through @eventsea/db (Walking Skeleton DB read)"
  artifacts:
    - path: "apps/web/app/privacy/page.tsx"
      provides: "GDPR-01 privacy policy route"
      contains: "privacy"
    - path: "apps/web/components/CookieConsent.tsx"
      provides: "GDPR-02 consent banner (use client)"
      contains: "'use client'"
    - path: "apps/web/app/page.tsx"
      provides: "Walking Skeleton home page reading events table"
      contains: "@eventsea/db"
  key_links:
    - from: "apps/web/app/layout.tsx"
      to: "CookieConsent component + /privacy footer link"
      via: "import + render"
      pattern: "CookieConsent"
    - from: "apps/web/app/page.tsx"
      to: "events table"
      via: "db.select().from(events)"
      pattern: "from\\(events\\)"
---

<objective>
Build the web half of the Walking Skeleton: the Next.js 16 App Router shell with a global layout, a `/privacy` route serving the privacy policy (GDPR-01) linked in the footer on every page, a vanilla-cookieconsent banner that gates analytics behind explicit consent (GDPR-02), and a home page that reads the live `events` table through `@eventsea/db` to prove the full stack works end-to-end. Implements GDPR-01 and GDPR-02.

Purpose: Closes the Walking Skeleton loop — a real Next.js route reads the real migrated DB. Implements D-13 (cookie consent), D-15 (async dynamic APIs), D-16 (no @vanilla-cookieconsent/react).
Output: Working `/privacy` route, global cookie consent banner, home page DB read, passing privacy + cookie-consent tests.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/ROADMAP.md
@.planning/phases/01-foundation-legal/01-RESEARCH.md
@.planning/phases/01-foundation-legal/01-CONTEXT.md
@CLAUDE.md

<interfaces>
<!-- From RESEARCH.md Pattern 7 (cookie consent) + Walking Skeleton DB Read code example. -->
DB read in a Server Component (RESEARCH.md Walking Skeleton DB Read):
- `import { db } from '@eventsea/db';`
- `import { events } from '@eventsea/db';`  (events re-exported from barrel)
- `const recentEvents = await db.select().from(events).limit(5);`  — Server Component is async

Cookie consent (RESEARCH.md Pattern 7):
- `'use client'` component wrapping `CookieConsent.run({...})` in useEffect
- categories: `necessary` (enabled, readOnly) + `analytics` (opt-in)
- analytics initialization ONLY inside `onConsent`/`onChange` after `acceptedCategory('analytics')` (Pitfall 5)
- import `'vanilla-cookieconsent/dist/cookieconsent.css'`
- NEVER import `@vanilla-cookieconsent/react` — does not exist (D-16)

Next.js 16 (Pitfall 2 / D-15): all `params`, `cookies()`, `headers()` are async — always await.
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: App shell, /privacy route, cookie consent banner, home DB read</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 7 vanilla-cookieconsent, Walking Skeleton DB Read example, Pitfall 2 async APIs, Pitfall 5 consent gating, Anti-Patterns: @vanilla-cookieconsent/react, middleware.ts→proxy.ts)
    - .planning/phases/01-foundation-legal/01-CONTEXT.md (D-13 consent library, GDPR-01/02 requirements)
    - apps/web/package.json (created in Plan 01 — confirms next, vanilla-cookieconsent installed)
    - packages/db/src/index.ts (created in Plan 02 — db client + events export the home page imports)
    - CLAUDE.md (GDPR & Privacy; Code Style — App Router required, next/image only, eslint . not next lint)
  </read_first>
  <behavior>
    - GET /privacy returns 200 and the rendered HTML contains the privacy policy heading/content
    - Every page footer contains a link with href="/privacy"
    - CookieConsent component renders without throwing; analytics callback is only reachable after acceptedCategory('analytics') is true
    - Home page server component awaits db.select().from(events) and renders the count
  </behavior>
  <action>
    Create apps/web/next.config.ts (TypeScript config; use top-level `turbopack` key not experimental — Next 16) and apps/web/tsconfig.json extending ../../tsconfig.base.json with the Next.js plugin and `@/*` path alias to the app root.
    Create apps/web/components/CookieConsent.tsx per RESEARCH.md Pattern 7: `'use client'` directive, useEffect calling CookieConsent.run with `necessary` (enabled, readOnly) and `analytics` (opt-in) categories, Lithuanian (`lt`) translations for the consent + preferences modals, and analytics initialization placed ONLY inside the `onConsent`/`onChange` callbacks guarded by `CookieConsent.acceptedCategory('analytics')` (Pitfall 5). Import `'vanilla-cookieconsent/dist/cookieconsent.css'`. Do NOT import `@vanilla-cookieconsent/react` (D-16).
    Create apps/web/app/layout.tsx: root layout (async per Next 16), renders children, a footer with a link `href="/privacy"` (so GDPR-01 privacy policy is reachable on every page), and `<CookieConsentBanner />` before the closing body tag.
    Create apps/web/app/privacy/page.tsx: a Server Component rendering the privacy policy minimum content per RESEARCH.md GDPR Compliance Notes (controller identity, what data + why, legal basis, retention, user rights, right to complain to VDAI, cookie summary, DPO contact). Static SSR, no client JS required.
    Create apps/web/app/page.tsx: async Server Component that does `await db.select().from(events).limit(5)` via `@eventsea/db` and renders the count (RESEARCH.md Walking Skeleton DB Read) — this proves the route→DB path. Handle the empty-events case (count 0) gracefully.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && grep -q "'use client'" apps/web/components/CookieConsent.tsx && ! grep -rq '@vanilla-cookieconsent/react' apps/web && grep -q 'acceptedCategory' apps/web/components/CookieConsent.tsx && grep -q 'href="/privacy"' apps/web/app/layout.tsx && grep -q 'from(events)' apps/web/app/page.tsx && pnpm --filter web exec tsc --noEmit && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - apps/web/components/CookieConsent.tsx contains `'use client'` and `acceptedCategory('analytics')` guarding analytics init
    - No file under apps/web imports `@vanilla-cookieconsent/react`
    - apps/web/app/layout.tsx contains `href="/privacy"` and renders the CookieConsent component
    - apps/web/app/privacy/page.tsx contains privacy policy content (controller, retention, user rights, VDAI)
    - apps/web/app/page.tsx contains `db.select().from(events)` and awaits it
    - `pnpm --filter web exec tsc --noEmit` exits 0
  </acceptance_criteria>
  <done>App shell, /privacy route, consent banner (analytics gated), and home DB read all typecheck clean; privacy link in global footer.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Privacy route smoke test + cookie-consent gating test (replaces Wave 0 placeholders)</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-VALIDATION.md (privacy-route + cookie-consent rows; commands `pnpm --filter web test -- privacy` and `-- cookie-consent`)
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 7, Pitfall 5 consent-before-analytics)
    - apps/web/app/privacy/page.tsx and apps/web/components/CookieConsent.tsx (created in Task 1)
    - apps/web/vitest.config.ts (created in Plan 01 — jsdom environment, testing-library)
  </read_first>
  <behavior>
    - privacy test: rendering the privacy page output contains the policy heading and key required content (user rights, controller)
    - cookie-consent test: the component renders; analytics callback does NOT fire when consent is not granted; fires when acceptedCategory('analytics') returns true
  </behavior>
  <action>
    Create apps/web/app/__tests__/privacy.test.ts: render the /privacy page component and assert it returns 200-equivalent content containing the privacy policy heading and at least one required field (e.g. user rights / controller identity). Runnable via `pnpm --filter web test -- privacy` (GDPR-01).
    Create apps/web/components/__tests__/CookieConsent.test.tsx: render CookieConsent; assert the analytics init path is NOT invoked before consent and IS invoked only after `acceptedCategory('analytics')` is true (mock/stub the vanilla-cookieconsent API to drive the callbacks) (GDPR-02, Pitfall 5). Runnable via `pnpm --filter web test -- cookie-consent`.
    Delete the Plan 01 placeholder apps/web/app/__tests__/setup.test.ts.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && pnpm --filter web test -- privacy && pnpm --filter web test -- cookie-consent</automated>
  </verify>
  <acceptance_criteria>
    - apps/web/app/__tests__/privacy.test.ts exists; `pnpm --filter web test -- privacy` exits 0
    - apps/web/components/__tests__/CookieConsent.test.tsx exists and asserts analytics does NOT fire before consent
    - `pnpm --filter web test -- cookie-consent` exits 0
    - Plan 01 placeholder setup.test.ts no longer exists in apps/web
  </acceptance_criteria>
  <done>Privacy route smoke test green (GDPR-01); cookie-consent gating test proves analytics fires only after consent (GDPR-02).</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| browser → analytics scripts | Consent boundary — no tracking script may load before explicit opt-in |
| client → server (route handlers) | Next.js 16 dynamic APIs are async; sync access crashes at runtime |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-05-01 | Privacy violation | Analytics fires before cookie consent | mitigate | Analytics init only inside onConsent guarded by acceptedCategory('analytics'); test asserts it does not fire pre-consent (Pitfall 5, GDPR-02) |
| T-05-02 | Tampering | Importing non-existent @vanilla-cookieconsent/react (supply-chain/typosquat risk) | mitigate | Use vanilla-cookieconsent directly; acceptance criterion forbids the scoped package (D-16) |
| T-05-03 | Denial of Service | Sync cookies()/params crash route at runtime | mitigate | All dynamic APIs awaited per Next 16 (Pitfall 2 / D-15); tsc + render test catch misuse |
| T-05-04 | Information Disclosure | Privacy policy missing required GDPR fields | mitigate | /privacy renders controller identity, retention, user rights, VDAI complaint route (GDPR-01) |
</threat_model>

<verification>
- `pnpm --filter web exec tsc --noEmit` exits 0
- `pnpm --filter web test -- privacy` green
- `pnpm --filter web test -- cookie-consent` green
- /privacy reachable via footer link on every page
- Home page reads live events table (Walking Skeleton closed)
</verification>

<success_criteria>
Privacy policy page live at /privacy and linked on all pages (GDPR-01); cookie consent banner gates analytics behind explicit opt-in (GDPR-02); home page reads the migrated events table proving the full Walking Skeleton stack. Maps to Phase 1 Success Criterion 4 (privacy page + consent) and completes Criterion 1 (Next.js runs locally with real DB read).
</success_criteria>

<output>
Create `.planning/phases/01-foundation-legal/01-05-SUMMARY.md` when done.
</output>
