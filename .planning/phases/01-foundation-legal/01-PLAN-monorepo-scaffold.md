---
phase: 01-foundation-legal
plan: 01
type: execute
wave: 1
depends_on: []
files_modified:
  - package.json
  - pnpm-workspace.yaml
  - turbo.json
  - tsconfig.base.json
  - docker-compose.yml
  - .env.example
  - .gitignore
  - .github/workflows/ci.yml
  - .github/workflows/deploy-worker.yml
  - apps/web/package.json
  - apps/worker/package.json
  - packages/db/package.json
  - packages/db/vitest.config.ts
  - apps/worker/vitest.config.ts
  - apps/web/vitest.config.ts
autonomous: true
requirements: []
must_haves:
  truths:
    - "pnpm install completes with a frozen lockfile across all three workspace packages"
    - "turbo typecheck lint test runs and reports per-package results (not 0 tasks)"
    - "docker compose up -d starts postgres, redis, and meilisearch with healthy postgres"
    - "Vitest is installed and runnable in packages/db, apps/worker, apps/web"
  artifacts:
    - path: "turbo.json"
      provides: "Turborepo v2 task graph"
      contains: "\"tasks\""
    - path: "pnpm-workspace.yaml"
      provides: "Workspace package globs"
      contains: "apps/*"
    - path: "docker-compose.yml"
      provides: "Local postgres:16, redis:7, meilisearch"
      contains: "postgres:16"
    - path: ".env.example"
      provides: "Documented env vars, no real secrets"
      contains: "DATABASE_URL="
    - path: ".github/workflows/ci.yml"
      provides: "CI typecheck/lint/test on PR"
      contains: "pnpm turbo"
  key_links:
    - from: "turbo.json"
      to: "tasks key (not pipeline)"
      via: "Turborepo v2 schema"
      pattern: "\"tasks\""
    - from: "apps/web/package.json + apps/worker/package.json"
      to: "@eventsea/db"
      via: "workspace:* dependency"
      pattern: "workspace:\\*"
---

<objective>
Scaffold the Turborepo + pnpm monorepo with three packages (apps/web, apps/worker, packages/db), Docker Compose local services, Turborepo-aware GitHub Actions CI, and per-package Vitest configs. This is the Walking Skeleton foundation every other Phase 1 plan builds on.

Purpose: Establish the exact project structure, task graph, and CI pipeline so downstream plans (DB schema, worker, web) drop into a working monorepo without re-deciding tooling. Implements D-01 through D-08.
Output: Working `pnpm install`, `turbo` task graph, `docker compose up`, CI workflows, and Wave 0 test framework installs.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/PROJECT.md
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/01-foundation-legal/01-RESEARCH.md
@.planning/phases/01-foundation-legal/01-CONTEXT.md
@CLAUDE.md

<interfaces>
<!-- Workspace package names downstream plans import. Use these exact names. -->
- DB package name: `@eventsea/db` (referenced as `"@eventsea/db": "workspace:*"`)
- App package names: `web`, `worker` (used with `pnpm --filter web` / `pnpm --filter worker`)
- Turborepo tasks (from RESEARCH.md Pattern 1): `build`, `typecheck`, `lint`, `test`, `dev`
- DB integration tests use a separate non-cacheable `test:integration` task (RESEARCH.md Pitfall 8)
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Scaffold Turborepo workspace root + Docker Compose + env</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 1 turbo.json, Pattern 2 pnpm-workspace.yaml, Pattern 5 docker-compose, Recommended Project Structure)
    - CLAUDE.md (Environment Variables section — required env var list; Code Style — TypeScript strict, Prettier, ESLint)
    - .env.example (if it exists)
  </read_first>
  <action>
    Initialize the repo root. Create pnpm-workspace.yaml with package globs `apps/*` and `packages/*`. Create root package.json named `eventsea` (private: true) with `"packageManager": "pnpm@9"` (per RESEARCH.md Open Question 2), root devDependencies turbo@2.9.14 and typescript@6.0.3, and scripts that delegate to turbo (`build`, `typecheck`, `lint`, `test`, `dev`).
    Create turbo.json using the Turborepo v2 `tasks` key (NOT `pipeline` — `pipeline` is silently ignored and runs 0 tasks per Pitfall 1). Include `"$schema": "https://turborepo.dev/schema.json"`, `globalDependencies: ["tsconfig.base.json"]`, `globalEnv: ["NODE_ENV", "DATABASE_URL", "REDIS_URL"]`, and tasks: `build` (dependsOn `^build`, outputs `.next/**`, `!.next/cache/**`, `dist/**`), `typecheck` (dependsOn `^build`), `lint`, `test` (outputs `coverage/**`), `test:integration` (`"cache": false` per Pitfall 8), `dev` (`cache: false`, `persistent: true`).
    Create tsconfig.base.json with strict mode true, target ES2022, moduleResolution bundler, that app/package tsconfigs extend.
    Create docker-compose.yml with services postgres (image `postgres:16`, POSTGRES_USER/PASSWORD/DB all `eventsea`, port 5432, named volume, pg_isready healthcheck), redis (image `redis:7`, port 6379, redis-cli ping healthcheck), meilisearch (image `getmeili/meilisearch:v1.13.3` — verify latest v1.x tag at hub.docker.com/r/getmeili/meilisearch/tags and pin the confirmed tag; MEILI_MASTER_KEY local-dev-master-key, MEILI_ENV development, port 7700). Declare named volumes postgres_data, redis_data, meilisearch_data.
    Create .env.example with documented keys and NO real values: DATABASE_URL=, REDIS_URL=, plus a comment block noting worker uses direct Postgres URL (Session pooler) and web uses Transaction pooler per RESEARCH.md Open Question 3. Include the local-dev defaults as comments (e.g. `# local: postgresql://eventsea:eventsea@localhost:5432/eventsea`).
    Create .gitignore covering node_modules, .next, dist, .turbo, .env.local, .env, coverage.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && grep -q '"tasks"' turbo.json && ! grep -q '"pipeline"' turbo.json && grep -q 'apps/\*' pnpm-workspace.yaml && grep -q 'postgres:16' docker-compose.yml && grep -q 'DATABASE_URL=' .env.example && grep -q '.env.local' .gitignore && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - turbo.json contains `"tasks"` and does NOT contain `"pipeline"`
    - turbo.json contains `"test:integration"` with `"cache": false`
    - pnpm-workspace.yaml contains `apps/*` and `packages/*`
    - root package.json contains `"packageManager": "pnpm@9"`
    - docker-compose.yml contains `postgres:16`, `redis:7`, and a `getmeili/meilisearch:v1` image tag
    - .env.example contains `DATABASE_URL=` and `REDIS_URL=` with NO real credential values after the `=`
    - .gitignore contains `.env.local` and `.env`
  </acceptance_criteria>
  <done>Workspace root files exist; turbo uses `tasks` key; Docker Compose defines all three services; env vars documented without secrets.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Create three package skeletons + install deps + Vitest configs (Wave 0)</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Standard Stack install commands, packages/db package.json shape, packages/db index.ts export shape, Validation Architecture Wave 0 Gaps)
    - .planning/phases/01-foundation-legal/01-VALIDATION.md (Wave 0 Requirements list, Test Infrastructure)
    - turbo.json (created in Task 1 — task names must match)
    - pnpm-workspace.yaml (created in Task 1)
  </read_first>
  <behavior>
    - `pnpm install --frozen-lockfile` exits 0 after lockfile is generated
    - `pnpm --filter db test` runs Vitest and reports a result (passing placeholder test is acceptable at this stage)
    - `pnpm --filter worker test` and `pnpm --filter web test` each run Vitest
    - `pnpm turbo typecheck` runs across all three packages without "0 tasks"
  </behavior>
  <action>
    Create apps/web/package.json (name `web`), apps/worker/package.json (name `worker`), packages/db/package.json (name `@eventsea/db`, main + types `./src/index.ts`, scripts `db:push` `drizzle-kit push`, `db:generate` `drizzle-kit generate`, `db:migrate` `drizzle-kit migrate`). Each package.json declares `typecheck`, `lint`, `test` scripts; web and worker declare `"@eventsea/db": "workspace:*"`.
    Install dependencies per RESEARCH.md Standard Stack: root `pnpm add -w turbo typescript`; web `pnpm --filter web add next@16.2.6 react react-dom vanilla-cookieconsent@3.1.0` and `pnpm --filter web add -D @types/react @types/react-dom vitest@4.1.7 @testing-library/react @testing-library/jest-dom`; worker `pnpm --filter worker add bullmq@5.77.3 ioredis@5.10.1` and `pnpm --filter worker add -D tsx@4.22.3 typescript vitest@4.1.7`; db `pnpm --filter db add drizzle-orm@0.45.2 postgres@3.4.9` and `pnpm --filter db add -D drizzle-kit@0.31.10 vitest@4.1.7 @vitest/coverage-v8`.
    Create vitest.config.ts in each of packages/db, apps/worker, apps/web (test environment node for db/worker, jsdom for web). Create placeholder test files so the test task is wired and non-empty: packages/db/src/__tests__/setup.test.ts, apps/worker/src/__tests__/setup.test.ts, apps/web/app/__tests__/setup.test.ts — each with a single trivially passing assertion that will be replaced by real tests in downstream plans.
    Do NOT install slopcheck-flagged or non-existent packages: never install `@vanilla-cookieconsent/react` (does not exist on npm per RESEARCH.md). postgres and ioredis are marked [ASSUMED] in the audit — see threat T-01-SC; a verification checkpoint precedes nothing here because both are widely-used core packages already verified on npm registry by the researcher.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && pnpm install --frozen-lockfile && pnpm --filter db test && pnpm --filter worker test && pnpm --filter web test</automated>
  </verify>
  <acceptance_criteria>
    - packages/db/package.json contains `"name": "@eventsea/db"` and scripts `db:push`, `db:generate`, `db:migrate`
    - apps/web/package.json and apps/worker/package.json each contain `"@eventsea/db": "workspace:*"`
    - `pnpm install --frozen-lockfile` exits 0
    - `pnpm --filter db test` exits 0 (placeholder test passes)
    - `pnpm --filter worker test` exits 0
    - `pnpm --filter web test` exits 0
    - No dependency named `@vanilla-cookieconsent/react` appears in any package.json
  </acceptance_criteria>
  <done>All three packages install, Vitest runs in each, workspace dependency `@eventsea/db` resolves, turbo sees real tasks.</done>
</task>

<task type="auto">
  <name>Task 3: GitHub Actions CI + Fly.io worker deploy workflows</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 8 GitHub Actions CI, Pattern 9 fly.toml + Worker Dockerfile, Pitfall 3 drizzle-kit push in CI destroys prod)
    - turbo.json (task names referenced by CI)
    - CLAUDE.md (Environment Variables — secrets must never be committed)
  </read_first>
  <action>
    Create .github/workflows/ci.yml: triggers on push to main and pull_request (opened, synchronize); single job on ubuntu-latest with steps actions/checkout@v4 (fetch-depth 2), pnpm/action-setup@v3 (version 9 to match local pnpm), actions/setup-node@v4 (node-version 20, cache pnpm), `pnpm install --frozen-lockfile`, then `pnpm turbo typecheck lint test`. Do NOT add `drizzle-kit push` anywhere in CI (Pitfall 3 — push destroys prod schema; CI/prod use generate+migrate only). Do NOT `echo` any secret env var in any step (threat: secrets in CI logs).
    Create .github/workflows/deploy-worker.yml: triggers on push to main; job on ubuntu-latest with concurrency `deploy-worker`; steps actions/checkout@v4, superfly/flyctl-actions/setup-flyctl@master, `flyctl deploy --remote-only --config apps/worker/fly.toml` with env `FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}`.
    Create apps/worker/fly.toml (app `eventsea-worker`, primary_region `fra`, build.dockerfile `Dockerfile`, process `worker = "node dist/index.js"`, vm shared-cpu-1x 512mb) and apps/worker/Dockerfile (node:20-alpine, install pnpm, frozen-lockfile install, pnpm build, CMD node dist/index.js) per RESEARCH.md Pattern 9. The worker source entry is created in Plan 03 — these deploy files reference `dist/index.js` which Plan 03 produces.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && grep -q 'pnpm turbo typecheck lint test' .github/workflows/ci.yml && ! grep -q 'drizzle-kit push' .github/workflows/ci.yml && grep -q 'FLY_API_TOKEN' .github/workflows/deploy-worker.yml && grep -q 'primary_region = "fra"' apps/worker/fly.toml && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - .github/workflows/ci.yml contains `pnpm turbo typecheck lint test`
    - .github/workflows/ci.yml does NOT contain `drizzle-kit push`
    - No CI step contains `echo $DATABASE_URL` or `echo $REDIS_URL` or `echo ${{ secrets`
    - .github/workflows/deploy-worker.yml references `secrets.FLY_API_TOKEN`
    - apps/worker/fly.toml contains `primary_region = "fra"`
    - apps/worker/Dockerfile contains `node:20-alpine`
  </acceptance_criteria>
  <done>CI runs typecheck/lint/test via turbo with no push/secret-leak; worker deploy workflow + fly.toml + Dockerfile defined.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| developer machine → git remote | Secrets in env files must not cross into version control |
| GitHub Actions → external services | CI secrets (FLY_API_TOKEN, DATABASE_URL) must not be logged |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-01-01 | Information Disclosure | .env / .env.local committed to git | mitigate | .gitignore covers `.env` and `.env.local`; .env.example has documented keys with empty values only (V14) |
| T-01-02 | Information Disclosure | Secrets logged in CI output | mitigate | No `echo $DATABASE_URL` / `echo ${{ secrets.* }}` in any workflow step; FLY_API_TOKEN passed via `env:` not inline |
| T-01-03 | Tampering | `drizzle-kit push` in CI drops prod columns | mitigate | CI workflow forbidden from calling `drizzle-kit push`; acceptance criterion asserts absence (Pitfall 3) |
| T-01-SC | Tampering | npm installs (postgres, ioredis marked [ASSUMED]) | accept | Both verified on npm registry by researcher (vercel/drizzle-team/taskforcesh ecosystem); widely-used core packages, no postinstall scripts; no [SUS]/[SLOP] packages present |
</threat_model>

<verification>
- `pnpm install --frozen-lockfile` exits 0
- `pnpm turbo typecheck lint test` runs real tasks across 3 packages
- `docker compose up -d` brings postgres to healthy state
- turbo.json uses `tasks` not `pipeline`
- No secrets committed; .env.local gitignored
</verification>

<success_criteria>
Monorepo scaffold complete: three packages install and typecheck, Docker Compose runs all local services, CI pipeline green via turbo, Vitest wired in every package (Wave 0 test framework ready for downstream plans). Maps to Phase 1 Success Criterion 1.
</success_criteria>

<output>
Create `.planning/phases/01-foundation-legal/01-01-SUMMARY.md` when done.
</output>
