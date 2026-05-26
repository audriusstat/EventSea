---
phase: 01-foundation-legal
plan: 02
type: execute
wave: 2
depends_on:
  - 01-01
files_modified:
  - packages/db/drizzle.config.ts
  - packages/db/src/index.ts
  - packages/db/src/schema/index.ts
  - packages/db/src/schema/users.ts
  - packages/db/src/schema/events.ts
  - packages/db/src/schema/sources.ts
  - packages/db/src/schema/price-history.ts
  - packages/db/src/schema/watchlist.ts
  - packages/db/src/schema/alert-log.ts
  - packages/db/src/schema/organizers.ts
  - packages/db/src/utils/event-id.ts
  - packages/db/src/__tests__/schema.test.ts
  - packages/db/src/__tests__/event-id.test.ts
  - packages/db/drizzle/
autonomous: true
requirements:
  - DATA-03
  - DATA-04
  - DATA-07
  - GDPR-04
must_haves:
  truths:
    - "All 7 tables (events, price_history, users, watchlist, alert_log, organizers, sources) exist in Postgres after migration"
    - "Deleting a user row cascades to delete that user's watchlist and alert_log rows"
    - "generateEventId(url) returns the same hash for the same URL on every call"
    - "sources table has last_successfully_scraped_at and consecutive_failures columns"
    - "price_history has no UPDATE path — only INSERT"
  artifacts:
    - path: "packages/db/src/schema/events.ts"
      provides: "events table with text() URL-hash primary key"
      contains: "pgTable('events'"
    - path: "packages/db/src/schema/sources.ts"
      provides: "sources circuit-breaker state table"
      contains: "lastSuccessfullyScrapedAt"
    - path: "packages/db/src/utils/event-id.ts"
      provides: "Deterministic URL-hash event ID generator"
      exports: ["generateEventId"]
    - path: "packages/db/src/index.ts"
      provides: "Drizzle db client + schema re-export"
      exports: ["db"]
  key_links:
    - from: "watchlist.userId / alertLog.userId"
      to: "users.id"
      via: "references with onDelete cascade"
      pattern: "onDelete: 'cascade'"
    - from: "events.sourceId"
      to: "sources.id"
      via: "foreign key reference"
      pattern: "references\\(\\(\\) => sources"
---

<objective>
Define the complete GDPR-compliant Drizzle schema for all 7 Phase 1 tables, the deterministic URL-hash event-ID generator, and the Drizzle db client. Then run the [BLOCKING] migration against local Postgres to materialize the tables. Implements DATA-03 (independent source records), DATA-04 (URL-hash IDs), DATA-07 (scraper health tracking), GDPR-04 (cascade delete).

Purpose: `packages/db` is the single source of truth for schema types that every future phase imports. Cascade-delete wiring is irreversible to retrofit (GDPR-04), so it must be correct now. Implements D-03, D-09, D-11, D-12.
Output: Schema files, migration SQL in packages/db/drizzle/, tables live in local Postgres, passing schema + cascade + event-id tests.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/ROADMAP.md
@.planning/STATE.md
@.planning/phases/01-foundation-legal/01-RESEARCH.md
@.planning/phases/01-foundation-legal/01-CONTEXT.md
@CLAUDE.md

<interfaces>
<!-- Schema contracts every downstream phase imports. From RESEARCH.md Pattern 3 + Pattern 10. -->
Barrel export shape (packages/db/src/index.ts):
- `export const db = drizzle(client, { schema })` — drizzle-orm/postgres-js
- `export * from './schema/index.js'`

Table column contracts (RESEARCH.md Pattern 3):
- events.id: text().primaryKey()  — URL-hash, DATA-04 (NOT integer identity)
- events.sourceId: integer().references(() => sources.id)
- users.id: integer().primaryKey().generatedAlwaysAsIdentity()
- priceHistory: APPEND ONLY (D-09) — no updatedAt, no UPDATE
- watchlist.userId / alertLog.userId: references(() => users.id, { onDelete: 'cascade' })  — GDPR-04
- sources: consecutiveFailures integer, lastSuccessfullyScrapedAt timestamp, status varchar (active|degraded|disabled)  — DATA-07

Event ID generator (RESEARCH.md Pattern 10):
- generateEventId(sourceUrl: string): string  — createHash('sha256').update(url).digest('hex').slice(0, 32)
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Write all 7 schema files + barrel + db client + event-id utility</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 3 full schema, Pattern 4 drizzle.config.ts, Pattern 10 event-id, packages/db index.ts export shape, Pitfall 7 price_history append-only)
    - .planning/phases/01-foundation-legal/01-CONTEXT.md (D-09 append-only, D-11 URL hash, D-12 cascade delete)
    - packages/db/package.json (created in Plan 01 — confirms drizzle-orm/drizzle-kit/postgres installed)
    - CLAUDE.md (Database section — append-only price_history, stable event IDs, UTC dates, GDPR cascade)
  </read_first>
  <behavior>
    - generateEventId('https://bilietai.lt/event/123') returns a 32-char hex string
    - generateEventId(sameUrl) === generateEventId(sameUrl) on repeated calls (deterministic)
    - generateEventId(urlA) !== generateEventId(urlB) for different URLs
    - All 7 table objects are exported from the barrel and importable as named exports
  </behavior>
  <action>
    Create the 7 schema files in packages/db/src/schema/ exactly per RESEARCH.md Pattern 3:
    - users.ts: `users` table, id integer().primaryKey().generatedAlwaysAsIdentity(), email varchar(255) notNull unique, createdAt timestamp withTimezone defaultNow notNull, deletedAt timestamp withTimezone (soft-delete tombstone). All dates withTimezone (UTC per DATA-07 / CLAUDE.md).
    - events.ts: `events` table, id text().primaryKey() (URL-hash, DATA-04), sourceId integer notNull references(() => sources.id), title text notNull, startsAt timestamp withTimezone notNull, endsAt timestamp withTimezone, venue text, city varchar(100), category varchar(100), minPrice numeric(10,2), currency varchar(3) default 'EUR', ticketUrl text notNull, status varchar(50) default 'active', featured boolean default false, lastSeenAt timestamp withTimezone defaultNow notNull, createdAt timestamp withTimezone defaultNow notNull. Indexes on city, startsAt, sourceId.
    - sources.ts: `sources` table (DATA-07 circuit breaker), id integer identity PK, name varchar(100) notNull unique, baseUrl text notNull, status varchar(50) default 'active' (active|degraded|disabled), consecutiveFailures integer default 0 notNull, lastSuccessfullyScrapedAt timestamp withTimezone, createdAt timestamp withTimezone defaultNow notNull.
    - price-history.ts: `price_history` table, APPEND ONLY — add a comment `/* APPEND ONLY — see D-09, never UPDATE */` at top of file (Pitfall 7). id integer identity PK, eventId text notNull references(() => events.id), price numeric(10,2) notNull, currency varchar(3) default 'EUR', observedAt timestamp withTimezone defaultNow notNull. No updatedAt column.
    - watchlist.ts: `watchlist` table, id integer identity PK, userId integer notNull references(() => users.id, { onDelete: 'cascade' }) (GDPR-04), eventId text notNull references(() => events.id), watchType varchar(50) notNull, priceThreshold numeric(10,2), createdAt timestamp withTimezone defaultNow notNull.
    - alert-log.ts: `alert_log` table, id integer identity PK, userId integer notNull references(() => users.id, { onDelete: 'cascade' }) (GDPR-04), eventId text notNull references(() => events.id), alertType varchar(50) notNull, sentAt timestamp withTimezone defaultNow notNull, channel varchar(50) notNull.
    - organizers.ts: `organizers` table (separate from users), id integer identity PK, name text notNull, email varchar(255) notNull unique, createdAt timestamp withTimezone defaultNow notNull.
    Create packages/db/src/schema/index.ts as a barrel re-exporting all 7 tables. Create packages/db/src/utils/event-id.ts exporting generateEventId per Pattern 10 (sha256, slice 0,32). Create packages/db/src/index.ts per RESEARCH.md export shape (drizzle postgres-js client + `export * from './schema/index.js'`). Create packages/db/drizzle.config.ts per Pattern 4 (schema './src/schema/index.ts', out './drizzle', dialect 'postgresql', dbCredentials.url from process.env.DATABASE_URL).
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && pnpm --filter db exec tsc --noEmit && grep -rq "onDelete: 'cascade'" packages/db/src/schema/watchlist.ts packages/db/src/schema/alert-log.ts && grep -q "APPEND ONLY" packages/db/src/schema/price-history.ts && grep -q "pgTable('events'" packages/db/src/schema/events.ts && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - All 7 files exist: users.ts, events.ts, sources.ts, price-history.ts, watchlist.ts, alert-log.ts, organizers.ts
    - packages/db/src/schema/events.ts contains `pgTable('events'` and `id: text().primaryKey()`
    - packages/db/src/schema/watchlist.ts AND alert-log.ts each contain `onDelete: 'cascade'`
    - packages/db/src/schema/price-history.ts contains `APPEND ONLY` comment and NO `updatedAt` column
    - packages/db/src/schema/sources.ts contains `lastSuccessfullyScrapedAt` and `consecutiveFailures`
    - packages/db/src/utils/event-id.ts exports `generateEventId` using `createHash('sha256')` and `.slice(0, 32)`
    - packages/db/src/index.ts exports `db` and re-exports schema
    - `pnpm --filter db exec tsc --noEmit` exits 0
  </acceptance_criteria>
  <done>All schema files, barrel, db client, drizzle.config, and event-id utility exist and typecheck clean.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Write schema + cascade + event-id tests (replaces Wave 0 placeholder)</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-VALIDATION.md (Per-Task Verification Map: schema-event-id, schema-cascade, schema-sources test commands)
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Validation Architecture, Pitfall 8 — DB tests must not be cached)
    - packages/db/src/index.ts and packages/db/src/schema/*.ts (created in Task 1 — import targets)
    - packages/db/vitest.config.ts (created in Plan 01)
  </read_first>
  <behavior>
    - event-id test: generateEventId returns 32-char hex; deterministic for same URL; distinct for different URLs
    - schema test: inserting a user then a watchlist + alert_log row referencing them succeeds
    - cascade test: deleting that user row removes the dependent watchlist and alert_log rows (count goes to 0)
    - schema test: sources table accepts a row with consecutiveFailures and lastSuccessfullyScrapedAt
  </behavior>
  <action>
    Create packages/db/src/__tests__/event-id.test.ts (pure unit, no DB) covering the three event-id behaviors above — runnable via `pnpm --filter db test -- event-id`.
    Create packages/db/src/__tests__/schema.test.ts as an integration test against local Docker Postgres (DATABASE_URL pointing at localhost:5432). It must: insert a sources row and a users row; insert a watchlist row and an alert_log row referencing them; assert both insert successfully (DATA-07 columns present, FK wiring works); then delete the user and assert watchlist + alert_log rows for that user are gone (GDPR-04 cascade); assert sources row has the consecutiveFailures + lastSuccessfullyScrapedAt fields populated. Use Drizzle query builder only — never raw `sql\`\`` string interpolation (threat T-02-01). Register this integration test under the turbo `test:integration` non-cacheable task (Pitfall 8) so a stale cache never reports a false pass; also runnable directly via `pnpm --filter db test -- schema` / `-- cascade-delete`. Delete the Plan 01 placeholder packages/db/src/__tests__/setup.test.ts.
    Note: this test requires the migration (Task 3) to have run; document that ordering — the executor runs Task 3 before asserting the integration test passes.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && pnpm --filter db test -- event-id</automated>
  </verify>
  <acceptance_criteria>
    - packages/db/src/__tests__/event-id.test.ts exists; `pnpm --filter db test -- event-id` exits 0
    - packages/db/src/__tests__/schema.test.ts exists and contains a user-delete assertion checking watchlist + alert_log rows are removed
    - schema.test.ts uses Drizzle query builder (no `sql\`` template with interpolated user input)
    - Plan 01 placeholder setup.test.ts no longer exists in packages/db
  </acceptance_criteria>
  <done>Event-id unit test passes standalone; schema integration test written for cascade + DATA-07, wired to non-cacheable task, ready to pass once migration runs.</done>
</task>

<task type="auto">
  <name>Task 3: [BLOCKING] Run drizzle migration against local Postgres + verify cascade test green</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 4 push/migrate workflow, Pitfall 3 push is local-only)
    - packages/db/drizzle.config.ts (created in Task 1)
    - docker-compose.yml (created in Plan 01 — provides local postgres)
    - packages/db/package.json (db:push / db:generate / db:migrate scripts)
  </read_first>
  <action>
    [BLOCKING] This task MUST run after all schema files (Task 1) and before phase verification. Build and typecheck pass without the live tables, creating a false-positive — only the migration proves the schema is real.
    Ensure Docker Compose is up (`docker compose up -d`) and Postgres is healthy. Set DATABASE_URL to the local Postgres URL (postgresql://eventsea:eventsea@localhost:5432/eventsea).
    For local materialization run `pnpm --filter db exec drizzle-kit push` (local dev sync, D-07). Then generate versioned migration SQL for CI/prod: `pnpm --filter db exec drizzle-kit generate` (writes SQL into packages/db/drizzle/). Confirm the generated migration file references all 7 tables. Do NOT run `drizzle-kit push` against Supabase ever (Pitfall 3) — push is local-only; CI/prod path is generate+migrate.
    After tables exist, run the schema integration test (Task 2) to prove cascade-delete and DATA-07 columns work against the real database.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && PGPASSWORD=eventsea psql -h localhost -U eventsea -d eventsea -tc "select count(*) from information_schema.tables where table_name in ('events','price_history','users','watchlist','alert_log','organizers','sources')" | grep -q 7 && pnpm --filter db test -- schema && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - All 7 tables exist in local Postgres (information_schema.tables count = 7 for the named tables)
    - packages/db/drizzle/ contains at least one generated .sql migration file referencing the 7 tables
    - `pnpm --filter db test -- schema` exits 0 (cascade-delete + DATA-07 verified against live DB)
    - No `drizzle-kit push` was run against a non-localhost DATABASE_URL
  </acceptance_criteria>
  <done>Migration applied to local Postgres; 7 tables live; versioned SQL generated for CI; cascade-delete integration test green against real DB.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| scraper/worker output → DB writes | Untrusted scraped strings written via Drizzle; must be typed/parameterized, never interpolated |
| migration tooling → database | `push` vs `migrate` boundary — push can destroy schema if pointed at prod |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-02-01 | Tampering | Drizzle queries with raw SQL interpolation | mitigate | Use Drizzle query builder only; tests use no `sql\`\`` with user input (V5 typed schema) |
| T-02-02 | Tampering | `drizzle-kit push` against production Supabase | mitigate | push restricted to localhost; acceptance criterion forbids non-localhost push; CI uses generate+migrate (Pitfall 3) |
| T-02-03 | Repudiation | UPDATE on price_history destroys audit trail | mitigate | price_history schema has no updatedAt; APPEND ONLY comment; D-09 enforced (Pitfall 7) |
| T-02-04 | Information Disclosure | Failure to cascade-delete leaves orphaned PII | mitigate | onDelete:'cascade' on all users.id FKs; integration test asserts deletion (GDPR-04) |
</threat_model>

<verification>
- `pnpm --filter db exec tsc --noEmit` exits 0
- 7 tables present in local Postgres after migration
- Cascade-delete integration test green against real DB
- event-id unit test green
- Generated migration SQL committed to packages/db/drizzle/
</verification>

<success_criteria>
PostgreSQL schema exists with all 7 tables; cascade-delete wiring verified by migration test; sources tracks last_successfully_scraped_at + circuit-breaker state; event IDs are deterministic URL hashes; price_history is append-only. Maps to Phase 1 Success Criterion 2 and partially Criterion 3 (sources health tracking).
</success_criteria>

<output>
Create `.planning/phases/01-foundation-legal/01-02-SUMMARY.md` when done.
</output>
