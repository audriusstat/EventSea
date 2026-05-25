# Phase 1: Foundation + Legal - Context

**Gathered:** 2026-05-25
**Status:** Ready for planning

<domain>
## Phase Boundary

Infrastructure skeleton, GDPR-compliant PostgreSQL schema, and scraping legal strategy — before a single user signup or scrape fires. This phase produces no user-facing features; it produces the foundations every subsequent phase builds on.

**Delivers:** Working Turborepo monorepo, Docker Compose local env, Drizzle schema with all Phase 1–7 tables, Fly.io worker skeleton running BullMQ, GitHub Actions CI, GDPR Article 30 register, privacy policy page, cookie consent architecture, and scraping legal memo.

**Does not deliver:** Any scraping logic (Phase 2), any UI beyond `/privacy` (Phase 3), any auth (Phase 4), any notifications (Phase 5).

</domain>

<decisions>
## Implementation Decisions

### Monorepo Structure
- **D-01:** Use **Turborepo + pnpm workspaces** as the monorepo tool. Turborepo handles build caching and parallel task orchestration; Vercel auto-detects Turborepo root configs.
- **D-02:** Three packages in Phase 1: `apps/web` (Next.js 16 app), `apps/worker` (Fly.io BullMQ worker), `packages/db` (Drizzle schema + migrations + DB client). No `packages/shared` yet — add when actual shared non-DB code exists.
- **D-03:** Drizzle schema lives in `packages/db/src/schema/`; generated migration files in `packages/db/drizzle/`. Both `apps/web` and `apps/worker` import from `packages/db`.
- **D-04:** CI/CD: **GitHub Actions** for type-check, lint, and test on every PR. **Vercel** handles Next.js deploy automatically on merge to main. Fly.io worker deployed via `flyctl` in a separate GitHub Actions job.

### Local Development Environment
- **D-05:** **Docker Compose** with `postgres:16`, `redis:7`, and `getmeili/meilisearch:v1.x` — fully offline-capable, no cloud billing during development, exact prod version match.
- **D-06:** Meilisearch **included in Docker Compose from Phase 1** to avoid config drift when Phase 3 wires search.
- **D-07:** Migration strategy: **`drizzle-kit push`** for local dev (instant schema sync, no migration files needed); **`drizzle-kit migrate`** in CI and production (versioned SQL files). Standard Drizzle workflow.
- **D-08:** Fly.io worker runs locally via **`pnpm --filter worker dev`** (tsx watch mode, connecting to Docker Compose Postgres + Redis). No Dockerfile needed for local dev; Dockerfile only for Fly.io deploys.

### Architecture Constraints (carried from planning)
- **D-09:** `price_history` is **append-only** — zero UPDATE operations on price columns, ever. This is a locked constraint from STATE.md.
- **D-10:** Supabase in **Frankfurt (eu-central-1)** for GDPR data residency. Local dev uses Docker Compose Postgres; prod/staging uses Supabase.
- **D-11:** Event ID = **URL hash** (DATA-04). Stable across scrapes; generated from source URL, not scrape order.
- **D-12:** **GDPR cascade delete** wired in Phase 1 schema (GDPR-04). Irreversible — cannot be retrofitted after user data exists.

### Cookie Consent Architecture
- **D-13:** Cookie consent area was not selected for discussion — left to planner's discretion. Use a pre-built GDPR-compliant library (e.g., vanilla-cookieconsent) rather than custom implementation. Analytics tracking must fire only after consent is granted (GDPR-02).

### Legal Memo
- **D-14:** Legal memo area was not selected for discussion — left to planner's discretion. Phase 1 deliverable is a **self-authored best-effort memo** (robots.txt analysis, EU Directive 96/9/EC review, partner outreach plan). Actual LT IP lawyer review is a pre-Phase 2 prerequisite (tracked in STATE.md todos), not a Phase 1 blocker.

### Claude's Discretion
- Cookie consent library selection (vanilla-cookieconsent or similar — planner's call)
- GitHub Actions workflow YAML structure
- `.env.example` variable naming
- Exact Fly.io app configuration (`fly.toml`)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Planning
- `.planning/ROADMAP.md` — Phase 1 success criteria (5 items), dependency chain, phase goals
- `.planning/REQUIREMENTS.md` — DATA-03, DATA-04, DATA-06, DATA-07, GDPR-01, GDPR-02, GDPR-03, GDPR-04 requirement definitions
- `.planning/STATE.md` — Locked decisions, external dependencies, open risks, architecture constraints
- `.planning/PROJECT.md` — Core value, tech stack anchor, constraints, business model

### No external specs yet
This is Phase 1 — no ADRs or feature specs exist yet. The canonical references above are the complete context.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — this is a greenfield project. Only `CLAUDE.md` exists.

### Established Patterns
- None yet — Phase 1 establishes the patterns all subsequent phases follow.

### Integration Points
- Phase 1 schema tables (`events`, `price_history`, `users`, `watchlist`, `alert_log`, `organizers`, `sources`) must be designed with Phase 2–7 requirements in mind — all foreign keys and cascade deletes must be correct from the start.
- `packages/db` becomes the single source of truth for schema types; all future phases import from it.

</code_context>

<specifics>
## Specific Ideas

No specific UI or implementation references from discussion — open to standard approaches for Phase 1 tooling.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope. Cookie consent and legal memo depth were noted as areas that could have been discussed but user deferred to planner's discretion.

</deferred>

---

*Phase: 1-Foundation + Legal*
*Context gathered: 2026-05-25*
