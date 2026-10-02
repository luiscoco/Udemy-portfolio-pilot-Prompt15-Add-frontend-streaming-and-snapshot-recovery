# Frontend Streaming and Snapshot Recovery

## Purpose of this learning activity

This prompt asks the coding agent to connect the React application to the authenticated event
stream built in milestone 14. Portfolio, news, watchlist, and quote changes should reach the browser
without a page refresh. The browser must recover after a disconnection and must keep one user's
private data separate from another user's data.

Students learn how to manage a shared connection in React, validate incoming messages, update only
the affected cached data, and recover changes that happen while a connection is being established.
They also learn to test recovery and distinguish a working app connection from fresh market data.

### Terms used in this activity

| Term | Meaning here |
| --- | --- |
| SSE, or Server-Sent Events | A long-lived HTTP connection through which the server sends notifications to the browser. |
| `EventSource` | The browser API that receives SSE messages. |
| Snapshot | An authorized read of the user's current portfolios, watchlist, and relevant news. |
| Cursor | A signed position in the event stream, used to request messages after that position. It is not a login credential. |
| Replay | Sending retained events again so a reconnecting browser can catch up. |
| UUID | A unique domain event identifier used to recognize duplicate deliveries. It is separate from the stream cursor. |
| Server-state cache | React Query's local copy of data fetched from the API. PostgreSQL remains the source of truth. |
| Invalidation | Marking a cached query as needing a fresh read. Active matching queries are fetched again. |
| React StrictMode | Development checks that repeat component setup and cleanup to expose lifecycle bugs. |

## Steps actually performed

### 1. Inspect the existing project and recovery API

The agent read `AGENTS.md`, the project state, milestone plan, architecture decision index, and
existing authentication, query, event contract, and recovery code. It found that
`GET /api/events/recovery` already captured user and market cursors **before** fetching the
authorized database snapshot. No new API route or database migration was needed.

Official SSE and TanStack Query documentation and installed query implementation/types were checked
before using connection and cache APIs. References are recorded in the [lesson notes](docs/lessons/15-frontend-streaming-and-recovery.md).

### 2. Build a reusable connection manager and React provider

The agent created `apps/web/src/lib/stream-manager.ts` and `apps/web/src/streaming.tsx`.
The provider owns one manager for the authenticated application and exposes its connection state
through React context. `useSyncExternalStore` connects React to the manager's state changes.

Cleanup waits one microtask—a short deferred callback—so StrictMode's immediate unsubscribe and
subscribe sequence retains the same connection. A real unmount closes the connection, aborts recovery,
removes network listeners, and clears retry timers, cursors, and remembered UUIDs.

`apps/web/src/auth.tsx` now mounts the provider with the authenticated user ID as its React key.
Logout, session expiry, and detected user changes clear the old query cache. Guards prevent late
authentication/recovery work from restoring a disposed scope.

### 3. Implement snapshot-plus-stream recovery

The browser first fetches the recovery snapshot, cancels older matching reads, and installs the
snapshot. It then subscribes using the cursor captured before those snapshot reads. Changes made
during the snapshot request are therefore available for replay.

Every named event is validated with the existing strict Zod schema, including a check that its
payload type matches the SSE event name. Invalid messages trigger fresh snapshot recovery.
The latest transport cursor advances even for a duplicate domain UUID; duplicate UUIDs do not
repeat cache updates.

Connection errors close the source and retry with the latest cursor. Retry delays increase from
one second to a maximum of 15 seconds. `stream.reset`, including expired retained events, discards
the cursor and starts another authorized snapshot handshake. Cursor and UUID history stay in memory
within the authenticated scope; they are not written to browser storage.

### 4. Reconcile only the affected server-state queries

The events contain notifications, not complete versioned entity records. The implementation therefore
uses targeted authoritative reads rather than patching cached records with partial event data.
For example, a portfolio rename refreshes the portfolio list and that portfolio's summary, while a
quote notification targets summaries containing the affected security.

Verification revealed an important race: invalidation alone can reuse an initial pending fetch with
no cached data. That fetch could return a pre-event value. The manager now explicitly cancels matching
reads before invalidating them; a regression test verifies this behavior.

### 5. Add visible connection status and persisted news

`apps/web/src/market.tsx` now displays the authorized persisted portfolio/watchlist news feed alongside
the existing provider samples. `apps/web/src/styles.css` adds styling for the connection indicator.

The UI displays `Live`, `Reconnecting`, or `Offline` with the explanation:

```text
Live · App updates. Provider polling and delays still apply.
```

`Live` describes the app connection. It does not promise instantaneous provider data. Existing
synthetic, delayed, and timestamp labels remain; provider samples still poll every five seconds.

### 6. Add tests, resolve failures, and record results

The agent added five unit tests in `apps/web/src/lib/stream-manager.test.ts` and six Chrome browser
tests in `apps/web/e2e/streaming.spec.ts`. Dependencies were restored from the local npm cache:

```powershell
npm.cmd ci --ignore-scripts --offline --cache .npm-cache
```

Initial failures were investigated and corrected: an unrealistic far-future test expiry overflowed
the session timer; a native reconnect test needed coordination to observe a short-lived intermediate
state; the freshly recovered default news feed was being unnecessarily refetched; and a quote
assertion expected a dollar sign instead of the application's `USD` display format.

The agent updated `docs/project-state.md` and created
`docs/lessons/15-frontend-streaming-and-recovery.md`. Generated frontend and shared build/Prisma
outputs were refreshed. Dependency versions, the lockfile, API routes, migrations, and project
instructions were unchanged. This follow-up adds this README only.

## Results achieved and verified

The milestone 15 checks recorded on **2026-10-02** were:

| Command | Observed result |
| --- | --- |
| Offline dependency restore shown above | 348 packages added; installation reported zero audit vulnerabilities. |
| `npm.cmd run typecheck` | Initially failed with Prisma cache `EPERM`; passed across all workspaces after the documented engine-path workaround. |
| `npm.cmd run test --workspace=@portfolio-pilot/web` | 10 tests passed: five existing tests and five new tests. |
| `npm.cmd run build --workspace=@portfolio-pilot/web` | Passed; existing module directive warnings remained. |
| `npm.cmd run check:browser-boundary` | Passed; the browser workspace dependency graph was clean. |
| `npm.cmd run test:browser --workspace=@portfolio-pilot/web -- streaming.spec.ts` | All six Chrome browser tests passed; none were skipped. |

The final browser run included this output:

```text
6 passed (44.6s)
```

Browser tests observed one shared connection under StrictMode, cursor advancement for duplicate UUIDs,
native EventSource replay and reconnect, news updates without unrelated portfolio reads, and a holding
quote changing from `USD 100.00` to `USD 125.00` without refresh. They also verified Offline/Online
transitions, expired-stream reset, malformed-message recovery, and Alice-to-Bob cache/cursor isolation.

**Test boundary:** these tests run the real React application with controlled API responses. Five
control EventSource delivery; one uses the native browser EventSource with controlled SSE response
bytes. They do not establish that the entire PostgreSQL/Redis/worker/browser pipeline was rerun.

## How to run and verify

### Prerequisites

- Node.js **24.21.0** and npm **11.19.0**, matching the pinned project tooling.
- Google Chrome, because the Playwright configuration selects `channel: 'chrome'`.
- Free port **5173** for Vite. The full demo also uses API port **3001**.
- For the full demo only: Docker with a working engine and Compose, or existing local PostgreSQL
  and Redis instances. The controlled milestone browser tests do not need them or external credentials.

All commands below run from the repository root in PowerShell. Use `npm` instead of `npm.cmd` on
other operating systems and adapt environment-variable assignments to your shell.

### Install and run the verified frontend checks

```powershell
node --version
npm.cmd --version
npm.cmd ci --ignore-scripts --offline --cache .npm-cache
npm.cmd run typecheck
npm.cmd run test --workspace=@portfolio-pilot/web
npm.cmd run build --workspace=@portfolio-pilot/web
npm.cmd run check:browser-boundary
```

The offline install requires the supplied npm cache. If it is incomplete, use
`npm.cmd ci --ignore-scripts --cache .npm-cache` with registry access. That fallback command is
setup guidance, not an additional installation verified in milestone 15.

The repository typecheck also builds shared packages and generates Prisma's database client.
For the permission error described below, apply the workaround before rerunning it.

Start the frontend in **terminal 1**:

```powershell
npm.cmd run dev --workspace=@portfolio-pilot/web
```

Run the targeted browser suite in **terminal 2**:

```powershell
npm.cmd run test:browser --workspace=@portfolio-pilot/web -- streaming.spec.ts
```

Keep Vite running while the tests execute. These tests provide their own authenticated API fixtures;
opening the app manually without an API is not equivalent to running those tests. Stop Vite with
`Ctrl+C` when finished.

### Windows Prisma permission workaround

The initial repository typecheck failed with:

```text
EPERM: operation not permitted, utime '.../windows/schema-engine'
```

Pointing Prisma at the existing readable cached engine allowed client generation and the complete
typecheck to pass in the implementation environment:

```powershell
$env:PRISMA_SCHEMA_ENGINE_BINARY = 'C:/Users/luisc/AppData/Roaming/Prisma/master/0edf323efd1d98336f3f0a68684b56f689b900d3/windows/schema-engine'
npm.cmd run typecheck
```

That path is specific to this machine. Use your installed engine path if needed. This workaround
was verified for generation/typechecking; it does not demonstrate that the extensionless cached
file can execute database migrations on Windows. Migration commands need a working Prisma engine
installation; see [lesson 14](docs/lessons/14-replayable-authenticated-sse.md) and the project state
for the earlier migration verification.

### Full local demo: prepared instructions, not rerun in milestone 15

The following setup uses the existing local infrastructure and seed scripts. It is provided for
students to verify the complete pipeline themselves. Use a dedicated local learning database.

Start the configured PostgreSQL and Redis services:

```powershell
npm.cmd run infra:start
npm.cmd run infra:status
```

In each application/worker terminal, set the shared local configuration below. These database
credentials match `compose.yaml` and are for local learning only. PowerShell environment settings
apply to the current terminal and its child processes; repeat them in newly opened terminals.

```powershell
$env:NODE_ENV = 'development'
$env:DATA_MODE = 'mock'
$env:AGENT_MODE = 'mock'
$env:DATABASE_URL = 'postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
$env:REDIS_URL = 'redis://127.0.0.1:6379'
$env:DEMO_AUTH_ENABLED = 'true'
$env:AUTH_BASE_URL = 'http://127.0.0.1:5173'
$env:AUTH_SECRET = 'local-learning-only-shared-secret-15'
```

The signing secret above is a public local example. API instances must use the same signing secret
to accept each other's cursors. No AI or market-provider credentials are needed in mock mode.

Build shared packages and the worker, apply migrations, and explicitly enable the demo seed:

```powershell
npm.cmd run build:types
npm.cmd run build --workspace=@portfolio-pilot/worker
npm.cmd run migrate:deploy --workspace=@portfolio-pilot/db
$env:ALLOW_DEMO_SEED = 'true'
npm.cmd run seed:demo --workspace=@portfolio-pilot/db
npm.cmd run dev
```

The seed requires `NODE_ENV=development` or `test`, `ALLOW_DEMO_SEED=true`, and a loopback database
URL. `npm.cmd run dev` starts the API and frontend, but does not start workers.

After repeating the shared environment settings, start ingestion in another terminal:

```powershell
$env:WORKER_ROLE = 'ingestion'
npm.cmd run start --workspace=@portfolio-pilot/worker
```

Start outbox delivery in one more terminal with the same shared settings:

```powershell
$env:WORKER_ROLE = 'outbox'
npm.cmd run start --workspace=@portfolio-pilot/worker
```

**Ingestion** collects provider data. **Outbox delivery** publishes committed database notifications
to Redis so the API can stream them. Both roles are needed to demonstrate the news event pipeline.

### Expected manual behavior to verify

1. Open `http://127.0.0.1:5173` in two tabs and sign in as Alice Demo in both.
2. Rename a portfolio in one tab. Expect the other tab's portfolio list to update without refresh.
3. Open News. Relevant persisted stories should appear as ingestion and outbox delivery run;
   the separate provider samples retain their polling and freshness labels.
4. Toggle browser DevTools network mode to Offline, then Online. Expect Offline followed by
   recovery through Reconnecting to Live when services are available.
5. Sign out and sign in as Bob Demo. Expect Bob's authorized data and a fresh Bob cursor, with
   no Alice private data reused. DevTools Network can show requests to `/api/events/recovery`
   and `/api/events?cursor=...`.

These are expected integrated behaviors to check locally, not newly observed full-stack results.
Stop application processes with `Ctrl+C`. `npm.cmd run infra:stop` stops the Compose services
while retaining their named data volumes.

## Limitations and unfinished work

- The initial Prisma typecheck failed; the engine-path workaround resolved the verified typecheck.
  Normal engine permissions may still need attention on another machine.
- Initial browser failures were corrected before the final passing run. Build directive warnings
  remain. No milestone 15 test in the targeted suite was skipped.
- The full infrastructure-backed browser journey and older browser suites that require seeded
  services were not rerun during this milestone. Server integration results belong to milestone 14.
- Live provider, live AI, Azure deployment, and deployed ingress behavior were not verified here.
- Cursor and UUID history intentionally do not survive logout or page reload. A reload takes a new
  authorized snapshot. UUIDs older than the 2,000-entry deduplication window can cause harmless
  targeted refetches.
- Agent event payloads are validated, but agent streaming UI belongs to milestones 18–19.
  Complete news filters and detail views belong to milestone 16.
- This working directory was not a Git repository during implementation; no commit, push, or
  deployment was performed. This README follow-up changes documentation only and does not rerun
  the application checks reported above.

## Further reading

- [Project state and actual check results](docs/project-state.md)
- [Milestone sequence and acceptance criteria](docs/project-plan.md)
- [Milestone 15 implementation and recovery algorithm](docs/lessons/15-frontend-streaming-and-recovery.md)
- [Milestone 14 authenticated SSE foundation](docs/lessons/14-replayable-authenticated-sse.md)
- [Pinned versions](docs/versions.md)
- [Architecture decisions](docs/decisions/README.md)
