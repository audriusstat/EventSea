# Phase 1: Foundation + Legal - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-05-25
**Phase:** 1-Foundation + Legal
**Areas discussed:** Monorepo structure, Local dev environment

---

## Monorepo Structure

### Q1: How should the monorepo be organized?

| Option | Description | Selected |
|--------|-------------|----------|
| Turborepo + pnpm workspaces | Build caching + task orchestration; Vercel auto-detects | ✓ |
| pnpm workspaces only | Simpler, no cache; fine early but loses parallel execution | |
| Separate repos | No code sharing; would duplicate schema types and DB client | |

**User's choice:** Turborepo + pnpm workspaces

---

### Q2: What packages should exist from Phase 1?

| Option | Description | Selected |
|--------|-------------|----------|
| apps/web + apps/worker + packages/db | Three packages; types co-located in db package | ✓ |
| apps/web + apps/worker + packages/db + packages/shared | Fourth package for non-db utilities | |
| apps/web + apps/worker only | Defer shared packages until needed | |

**User's choice:** apps/web + apps/worker + packages/db

---

### Q3: Where should Drizzle schema + migrations live?

| Option | Description | Selected |
|--------|-------------|----------|
| packages/db/src/schema/ + packages/db/drizzle/ | Standard Drizzle Kit layout; both apps import from packages/db | ✓ |
| Inside apps/web only | Worker imports via path alias; creates implicit coupling | |

**User's choice:** packages/db/src/schema/ + packages/db/drizzle/

---

### Q4: How should CI/CD be wired up?

| Option | Description | Selected |
|--------|-------------|----------|
| GitHub Actions + Vercel auto-deploy | GH Actions for type-check/lint/test; Vercel for web; flyctl for worker | ✓ |
| Vercel CI only | No automated checks for worker or shared packages | |
| You decide | Standard GH Actions + Vercel is the obvious call | |

**User's choice:** GitHub Actions + Vercel auto-deploy

---

## Local Dev Environment

### Q1: How should the local dev environment be set up?

| Option | Description | Selected |
|--------|-------------|----------|
| Docker Compose: local Postgres + Redis | Fully offline-capable; no cloud billing; exact prod version match | ✓ |
| Cloud Supabase + Upstash Redis from day one | No Docker needed; requires internet + cloud resources | |
| Docker Compose Postgres + Upstash Redis | Hybrid split between local and cloud | |

**User's choice:** Docker Compose: local Postgres + Redis

---

### Q2: Should Docker Compose include local Meilisearch?

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, include Meilisearch from Phase 1 | Avoids config drift when Phase 3 arrives | ✓ |
| No, add Meilisearch when Phase 3 needs it | Minimal docker-compose.yml initially | |
| Use Meilisearch Cloud | No self-hosted; requires network for every dev search call | |

**User's choice:** Yes, include Meilisearch in Docker Compose from Phase 1

---

### Q3: How should migrations run against local Docker Postgres?

| Option | Description | Selected |
|--------|-------------|----------|
| drizzle-kit push (local) + drizzle-kit migrate (prod) | Fast local iteration; versioned for CI/prod | ✓ |
| drizzle-kit migrate everywhere | Disciplined but slower local iteration cycle | |
| You decide | Standard push/migrate split is the obvious call | |

**User's choice:** drizzle-kit push for local dev, drizzle-kit migrate for prod

---

### Q4: How should the Fly.io worker run locally?

| Option | Description | Selected |
|--------|-------------|----------|
| pnpm --filter worker dev (tsx watch mode) | Fast iteration; no Dockerfile needed locally | ✓ |
| Worker in Docker Compose as a service | Production parity; adds Docker build to dev loop | |
| Deploy to Fly.io to test worker | Requires a deploy cycle per change | |

**User's choice:** Worker runs locally via pnpm --filter worker dev

---

## Claude's Discretion

- Cookie consent library selection (area not selected for discussion)
- Legal memo depth — self-authored best-effort was inferred from STATE.md, not discussed
- GitHub Actions workflow YAML structure
- `.env.example` variable naming
- Exact Fly.io `fly.toml` configuration

## Deferred Ideas

None — discussion stayed within phase scope.
