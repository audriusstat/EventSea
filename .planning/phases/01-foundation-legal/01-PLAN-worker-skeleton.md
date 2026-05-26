---
phase: 01-foundation-legal
plan: 03
type: execute
wave: 2
depends_on:
  - 01-01
files_modified:
  - apps/worker/src/index.ts
  - apps/worker/src/queues/scraper.ts
  - apps/worker/src/__tests__/queues.test.ts
  - apps/worker/tsconfig.json
  - apps/worker/package.json
autonomous: true
requirements:
  - DATA-03
must_haves:
  truths:
    - "Worker process boots, connects to Redis, and logs a ready message without crashing"
    - "bilietai and tiketa are separate BullMQ Queue instances (independent failure domains)"
    - "A dummy job enqueued to the bilietai queue is received by the worker"
    - "ioredis connection uses maxRetriesPerRequest: null"
  artifacts:
    - path: "apps/worker/src/index.ts"
      provides: "BullMQ worker entry point"
      contains: "maxRetriesPerRequest: null"
    - path: "apps/worker/src/queues/scraper.ts"
      provides: "Per-source queue definitions (DATA-03)"
      contains: "scraper:bilietai"
  key_links:
    - from: "apps/worker/src/index.ts"
      to: "Redis"
      via: "ioredis connection with maxRetriesPerRequest null"
      pattern: "maxRetriesPerRequest: null"
    - from: "scraper.ts"
      to: "two distinct Queue instances"
      via: "separate Queue per source"
      pattern: "new Queue\\('scraper:"
---

<objective>
Build the BullMQ worker skeleton: separate queues per source (bilietai, tiketa) for independent failure domains, an ioredis connection with the BullMQ-required `maxRetriesPerRequest: null`, and a worker that boots and accepts a dummy job. No scrape logic (that is Phase 2). Implements DATA-03.

Purpose: Prove the worker half of the Walking Skeleton works end-to-end against Redis, and establish the per-source queue isolation pattern Phase 2 scrapers plug into. Implements D-02, D-08, D-14.
Output: Worker entry + queues module, passing queue-isolation test, worker boots against local Redis.
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
<!-- BullMQ worker contracts from RESEARCH.md Pattern 6. Phase 2 scrapers fill the worker callback. -->
- ioredis connection: `new IORedis(process.env.REDIS_URL!, { maxRetriesPerRequest: null })` — required or worker crashes (Pitfall 4)
- One Queue per source (DATA-03 independent failure domains):
  - `new Queue('scraper:bilietai', { connection })`
  - `new Queue('scraper:tiketa', { connection })`
- Worker callback is a Phase 1 no-op (console.log only); Phase 2 fills in scrape logic
- worker.on('failed', ...) handler logs error; Phase 2 updates sources.consecutiveFailures
</interfaces>
</context>

<tasks>

<task type="auto" tdd="true">
  <name>Task 1: Worker entry + per-source queues module + tsconfig</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 6 BullMQ worker skeleton, Pitfall 4 maxRetriesPerRequest, Security Domain — no PII in job payloads, worker.on('failed') handler)
    - .planning/phases/01-foundation-legal/01-CONTEXT.md (D-02 worker package, D-14 maxRetriesPerRequest)
    - apps/worker/package.json (created in Plan 01 — confirms bullmq, ioredis, tsx installed)
    - CLAUDE.md (Database — Redis for BullMQ job queues)
  </read_first>
  <behavior>
    - Worker module exports the two queue instances (bilietaiQueue, tiketaQueue) so tests can assert their names differ
    - Connecting against local Redis (localhost:6379) does not throw on boot
    - The two queues have distinct names 'scraper:bilietai' and 'scraper:tiketa'
  </behavior>
  <action>
    Create apps/worker/src/queues/scraper.ts exporting an ioredis connection factory (or shared connection) created with `{ maxRetriesPerRequest: null }` (Pitfall 4 / D-14), plus two named Queue instances `scraper:bilietai` and `scraper:tiketa` (DATA-03 — separate failure domains). For Upstash TLS in prod, note `tls: {}` is needed when REDIS_URL uses rediss:// (comment only; local uses redis://).
    Create apps/worker/src/index.ts per RESEARCH.md Pattern 6: import the connection + queues, create a Worker on 'scraper:bilietai' whose callback is a Phase 1 no-op that console.logs the received job name (Phase 2 fills this in), attach worker.on('failed', ...) that logs error message (no stack trace leakage — V7), and console.log a ready message. Never put user emails/PII in job payloads (Security Domain) — Phase 1 jobs carry no PII anyway.
    Create apps/worker/tsconfig.json extending ../../tsconfig.base.json with outDir dist (so fly.toml's `node dist/index.js` works). Add a `build` script (tsc) and `dev` script (`tsx watch src/index.ts`) to apps/worker/package.json.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && grep -q 'maxRetriesPerRequest: null' apps/worker/src/queues/scraper.ts && grep -q "scraper:bilietai" apps/worker/src/queues/scraper.ts && grep -q "scraper:tiketa" apps/worker/src/queues/scraper.ts && pnpm --filter worker exec tsc --noEmit && echo PASS</automated>
  </verify>
  <acceptance_criteria>
    - apps/worker/src/queues/scraper.ts contains `maxRetriesPerRequest: null`
    - scraper.ts creates two Queue instances with names `scraper:bilietai` and `scraper:tiketa`
    - apps/worker/src/index.ts contains a `worker.on('failed'` handler and a `new Worker(`
    - apps/worker/package.json has a `build` script using tsc and a `dev` script using `tsx watch`
    - `pnpm --filter worker exec tsc --noEmit` exits 0
  </acceptance_criteria>
  <done>Worker entry + queues module typecheck clean; two distinct per-source queues; ioredis configured for BullMQ.</done>
</task>

<task type="auto" tdd="true">
  <name>Task 2: Queue-isolation test + dummy-job boot proof (replaces Wave 0 placeholder)</name>
  <read_first>
    - .planning/phases/01-foundation-legal/01-VALIDATION.md (worker-queues row: `pnpm --filter worker test -- queues`)
    - .planning/phases/01-foundation-legal/01-RESEARCH.md (Pattern 6, Pitfall 8 — DB/queue tests not cached)
    - apps/worker/src/queues/scraper.ts and apps/worker/src/index.ts (created in Task 1)
    - apps/worker/vitest.config.ts (created in Plan 01)
  </read_first>
  <behavior>
    - Test asserts bilietaiQueue.name !== tiketaQueue.name (DATA-03 independent domains)
    - Test asserts both queue names match the expected 'scraper:bilietai' / 'scraper:tiketa'
    - Boot proof: enqueuing a dummy job to the bilietai queue and the worker receiving it (against local Redis) resolves
  </behavior>
  <action>
    Create apps/worker/src/__tests__/queues.test.ts covering: the two queues are separate instances with distinct names (unit-level, no Redis needed for the name assertions); and an integration assertion that adds a dummy job to scraper:bilietai and the worker processes it (requires local Redis from Docker Compose). Register the Redis-touching assertion under the non-cacheable `test:integration` turbo task (Pitfall 8). Runnable via `pnpm --filter worker test -- queues`. Delete the Plan 01 placeholder apps/worker/src/__tests__/setup.test.ts.
    After the test is written, prove the worker actually boots: with Docker Compose Redis up and REDIS_URL=redis://localhost:6379, run the worker briefly (e.g. via tsx) and confirm it logs the ready message and processes the enqueued dummy job, then exits cleanly.
  </action>
  <verify>
    <automated>cd /Users/audrius/BEARING/EventSea && pnpm --filter worker test -- queues</automated>
  </verify>
  <acceptance_criteria>
    - apps/worker/src/__tests__/queues.test.ts exists and asserts the two queue names are distinct
    - `pnpm --filter worker test -- queues` exits 0
    - Plan 01 placeholder setup.test.ts no longer exists in apps/worker
    - Redis-touching assertion is under the `test:integration` (cache:false) task
  </acceptance_criteria>
  <done>Queue-isolation test green; worker boots against local Redis and processes a dummy job (DATA-03 proven).</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| worker → Redis | Job payloads cross into Redis; must not carry PII |
| worker failure handler → logs | Error output must not leak stack traces / secrets |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-03-01 | Information Disclosure | BullMQ job payloads containing user PII | mitigate | Phase 1 jobs carry no user data; convention documented — use user IDs only in future phases (Security Domain) |
| T-03-02 | Denial of Service | Worker crash-loop from missing maxRetriesPerRequest | mitigate | ioredis created with `maxRetriesPerRequest: null`; acceptance criterion asserts it (Pitfall 4) |
| T-03-03 | Information Disclosure | Stack traces in worker.on('failed') logs | mitigate | Handler logs `err.message` only, not full stack (V7 error handling) |
</threat_model>

<verification>
- `pnpm --filter worker exec tsc --noEmit` exits 0
- `pnpm --filter worker test -- queues` green
- Worker boots against local Redis, processes dummy job, logs ready
- Two distinct per-source queues confirmed
</verification>

<success_criteria>
Scraper worker skeleton runs BullMQ jobs with per-source queue isolation (DATA-03); ioredis correctly configured; worker boots against Redis and accepts a dummy job; deploy config (fly.toml/Dockerfile from Plan 01) targets the worker entry. Maps to Phase 1 Success Criterion 3.
</success_criteria>

<output>
Create `.planning/phases/01-foundation-legal/01-03-SUMMARY.md` when done.
</output>
