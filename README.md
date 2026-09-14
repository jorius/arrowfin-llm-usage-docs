# ArrowFin technical assessment — submission

Candidate: Jose Ríos. Role: Senior Full Stack Engineer / Team Lead. Feature: the Trader Daily
Snapshot. This repository is the single place for everything the assessment asks the candidate
to write; the two code repositories carry only technical documentation and link back here.

| Where | What |
| --- | --- |
| This repository | Background, time and LLM disclosure, Task 2 reconnect answer, Task 3 [SECURITY.md](./SECURITY.md), Task 4 code review, Task 5 reflection, [LLM-USAGE.md](./LLM-USAGE.md) and every prompt under [`prompts/`](./prompts/README.md) |
| <https://github.com/jorius/arrowfin-mt-daily-snap-svc> | NestJS + Prisma + PostgreSQL + Socket.IO backend: schema, migrations, tests, Swagger, Postman collection, deployment runbook |
| <https://github.com/jorius/arrowfin-mt-daily-snap-fe> | Next.js 16 + Tailwind 4 + TanStack Query widget |
| <https://arrowfin-mt-daily-snap.netlify.app> | Live widget, pointed at the staging backend |
| <https://arrowfin-mt-daily-snap-svc-staging.up.railway.app> | Staging backend (the demo; Swagger UI at `/docs`) |
| <https://arrowfin-mt-daily-snap-svc-development.up.railway.app> | Development backend, an isolated sandbox with its own database |

Demo credentials are shared out of band, never through a repository. The dataset CSVs were
treated as production data and are not in any repository.

---

## Background

### Stack ratings (1 never used, 2 tutorial-level, 3 shipped code with it, 4 comfortable in production, 5 could teach it)

- TypeScript: 4
- Next.js: 4
- NestJS: 3
- Prisma: 3
- PostgreSQL: 4
- WebSockets: 3
- Tailwind: 3
- JWT/auth: 3
- Multi-tenant data isolation: 4

### Five short answers

1. **What part of the stack am I strongest with?**

The frontend overall is the part I am strongest with: what the browser holds, what it sends,
and what it must never see. On the UI side I am productive with Tailwind and TanStack Query,
and I lean on the server for every number the user sees, so the backend stays the source of
truth and the frontend reinforces it. On the backend side, TypeScript services with clear
boundaries, PostgreSQL data modelling, and auth and session design are the parts of this
submission I would defend line by line.

2. **What part would be the steepest learning curve?**

The business and compliance rules of this ecosystem: futures and crypto products, data
management, regulatory obligations, the mathematical formulas behind risk. I learn fast, but
fintech and crypto platforms are very high-pace environments and I expect to work to keep up.
On the technical side, real-time at scale: WebSockets beyond a single node, sticky sessions,
a Redis adapter, backpressure. I have shipped WebSockets, but not at the scale a futures
exchange runs at.

3. **Any tech not in the stack I consider a core strength?**

Application security with an offensive mindset. Always thinking about how the application
could be broken through the code I am writing is a skill I have developed over the last years
studying cybersecurity on my own, and it shows in this submission in the threat modelling,
the IDOR and injection classes I designed against, and the PII hygiene in logs. Alongside it,
pragmatic DevOps for small teams: Docker, Postgres operations, platforms like Railway and
Netlify, CI with signed commits.

4. **What kind of work do I most want to do day-to-day?**

Building and owning whole ecosystems end to end with the grand scheme of things always in
mind: clear API contracts, the data model behind the services, reviewing other people's
code, keeping the design system honest across a team, and pairing with the team to keep the
bar high while shipping small increments to production.

5. **What do I most want to avoid?**

Pixel-pushing without product context, client-side business logic that the backend does not
agree with, UI work with no way to verify it, systems that live in one person's head with no
documentation, and heroic on-call without observability.

## Time and LLM disclosure

- **Time actually spent:** about 2 h 45 min of wall-clock time on 2026-09-14, in one sitting.
  I spent the first half hour after the technical interview reading the assessment document
  to understand everything clearly and setting up the initial Claude Code prompt; then roughly
  40 minutes of design and scaffolding, 30 minutes of parallel build (the frontend track ran
  alongside the backend for about 25 minutes), 20 minutes of deployment, and two iterations
  (light theme, Spanish, responsive layout, WebSocket settings; then Swagger and a Postman
  collection) of about 40 minutes, plus the consolidation of these documents.
- **What I used an LLM for:** Claude Code (Claude Fable 5.1 with MAX effort) for most of the
  typing. It read the brief and the dataset (and checked them for prompt injection), analysed
  the fills, proposed the architecture through a pre-installed brainstorming skill that I
  approved step by step, wrote the Prisma schema, the design spec and the implementation plan,
  and then two parallel agents wrote the backend and frontend code and tests from that plan.
  It verified the widget in a headless browser against the real backend, deployed to Railway
  and Netlify, and drafted the Task 4 review, the security writeup, the reconnect answer,
  Task 5 and this section, which I rewrote in my own words. I made every architectural
  decision myself: the opaque API-key auth model instead of the JWT it proposed, the
  `application/services/controllers/infrastructure` layering, app-layer isolation first with
  row-level security as the last task, the auth flow in the browser (opaque key in
  `sessionStorage`, sent as Bearer and in the socket handshake), that the client never
  computes P&L or risk, the `staging`/`development` topology on Railway, the scope of both
  iterations, and keeping red for the high-risk state. I reviewed the security-critical files
  (guard, use case, repositories, gateway, RLS migration) before pushing. The full account is
  in [LLM-USAGE.md](./LLM-USAGE.md) and every prompt I typed is published verbatim under
  [`prompts/`](./prompts/README.md).
- **What I would change if I had written it myself:** money math with a decimal type end to
  end instead of converting Prisma `Decimal` to JavaScript numbers at the repository boundary;
  FIFO lot matching instead of the average-cost ledger, because prop-firm rules care about
  which contracts closed; feature modules (auth, snapshot, stream) with explicit exports
  instead of one core module; a `positions` projection maintained from fills instead of
  replaying every fill on each request; a backend-for-frontend cookie plus a short-lived
  socket ticket instead of keeping the API key in `sessionStorage`; sequence numbers on the
  fill stream so the client can detect a gap instead of refetching on every reconnect; an
  i18n library with ICU plural rules instead of the hand-rolled dictionary once there are more
  than two languages; and component tests for the connection badge state machine, which today
  is only verified by hand. And the honest one: none of this fits in 2 to 3 hours without an
  agent. What the agent bought me was time to read and review instead of time to type.

---

## Task 2 — Reconnect behaviour

**What the client does today.** `hooks/useFillStream.ts` opens one Socket.IO connection per
API key with the client's built-in reconnection (`lib/socket.ts`: infinite attempts, back-off
capped at the configured maximum). The connection badge is the only thing on screen that
claims liveness:

| Event | Badge | Figures |
| --- | --- | --- |
| socket `connect` (first time) | `Live` | normal |
| socket `disconnect` | `Reconnecting` (amber) | unchanged, still bright |
| stale timer elapses without a connection | `Disconnected, figures may be stale` (red) | whole widget dimmed to 60% |
| socket `connect` after a drop | `Live` | every snapshot query is invalidated and refetched (an "updating" hint shows while it runs) |
| handshake refused (`unauthorized`) | `Session expired` | session cleared, redirect to `/login` |

**How the client knows whether it missed fills.** It does not try to find out from the stream.
A `fill` event is a hint to refetch, never data to apply: the hook invalidates the TanStack
Query entry for the selected account and the server recomputes the snapshot from the `fills`
table (`components/snapshot/SnapshotWidget.tsx` renders only server numbers; there is no
client-side P&L math). A gap in the stream therefore cannot corrupt what is shown, it can only
delay it, and the reconnect refetch closes the gap. The snapshot also carries `lastFillId`, so
a stricter client could compare the id of the last event it saw with the `lastFillId` of the
refetched snapshot and raise a warning when they differ; that comparison is not built.

**Where this would show stale numbers, honestly.**

1. Between the `disconnect` event and the stale timer the last figures stay bright with only
   the amber badge as a signal. A shorter timer trades false alarms for faster warnings; the
   default matches Socket.IO's own reconnection back-off.
2. The badge is pessimistic by design: if only the WebSocket is down, REST may still refresh
   the numbers on window focus while the badge says stale. Stale-but-fresh is acceptable;
   live-but-stale is not.
3. A fill event whose refetch fails (a network blip on the REST call) is recovered only by the
   next event, a window focus, or a reconnect. There is no retry queue.
4. Browsers throttle timers in background tabs, so the stale badge can appear late when the
   tab is not visible.

**With a second day.** Give every fill a monotonically increasing `seq` per account, include
the latest `seq` in the snapshot, and on reconnect send `since=<seq>` so the server replays the
gap (or answers "too far behind, refetch"). That turns "did I miss something?" into arithmetic
instead of a full refetch, and lets the client render fill-by-fill without trusting its own
math.

The stale threshold and the client back-off are configuration
(`NEXT_PUBLIC_WS_STALE_AFTER_MS`, `NEXT_PUBLIC_WS_RECONNECT_DELAY_MS`,
`NEXT_PUBLIC_WS_RECONNECT_DELAY_MAX_MS`), and the values in effect are visible in the widget's
Connection panel during a demo, next to the server heartbeat announced by the backend.

## Task 3 — Security and PII

Written in [SECURITY.md](./SECURITY.md): auth model, the six layers of tenant isolation and
what catches a forgotten `WHERE`, PII handling, the vulnerability class designed out (IDOR),
what production would add, the development-only surfaces, and the browser side.

## Task 4 — Code review

Review of the `positions/:accountId` pull request, written as I would leave it on the PR.

```ts
@Get('positions/:accountId')
async getPositions(@Param('accountId') accountId: string, @Req() req) {
  const positions = await this.prisma.position.findMany({ where: { accountId } });
  this.logger.log(`positions for ${accountId}: ${JSON.stringify(positions)}`);
  return positions.map(p => ({ ...p, pnl: (p.markPrice - p.avgPrice) * p.qty }));
}
```

**Verdict: Request changes.** Small, readable endpoint and the route shape is fine, but two of the
things in it must not ship on a shared-tenant platform, and one of them is our P0 class.

**Fix first: the tenant scope.** The only filter is `accountId` from the URL. Nothing ties the
account to the caller, so any caller who can guess or enumerate an id reads another trader's
positions, across brokers. That is the cross-broker access we define as a P0. The fix is one
predicate plus one test: take the trader and broker from the authenticated principal, never
from the request, and query `where: { accountId, account: { brokerId: principal.brokerId,
traderId: principal.traderId } }` (or resolve the account first with the same three keys and
return 404 when it isn't theirs; 404, not 403, so we don't confirm the id exists). Longer term
I'd rather the controller couldn't express an unscoped query at all: route this through the
tenant-scoped repository so the `brokerId` argument is not optional.

**Blocking**

- **Regulated data in the logs.** `JSON.stringify(positions)` writes every position row,
  exposure, average prices, whatever else the model carries, to the info log on every request,
  next to the account id. Our log pipeline is not a system of record for regulated data, and
  this is the kind of leak we have to report. Log the account id and the row count, or use the
  structured logger with explicit fields. Never serialize an entity.
- **Log injection.** `accountId` is user-controlled and interpolated raw into the line. A value
  with a newline or ANSI escape forges log entries. Fixed for free by the structured-logging
  change above; I'd still add a param validation pipe on the id.
- **Where is the guard?** `@Req() req` is injected and unused, and I don't see `@UseGuards` on
  the handler or the controller. If the guard is global, say so in the PR description and drop
  the unused param; if it isn't, this endpoint is open.

**Comments (fix in this PR or a follow-up, your call)**

- The controller talks to Prisma and does math. Move the query and the P&L into a service so
  it can be tested without HTTP, and so the tenant rule lives in one place.
- The P&L formula ignores the contract multiplier. `(mark − avg) × qty` is right for shares and
  wrong for futures: MES and ES move the same points and differ 10× in dollars. Multiply by the
  instrument's point value. Also confirm what `qty` is for shorts; if it's unsigned this formula
  is inverted for every short position.
- If those columns are `Decimal`, `p.markPrice - p.avgPrice` on Prisma `Decimal` objects doesn't
  do what it looks like. Convert at the boundary or use the decimal API.
- `...p` returns every column of the row to the client, including any column we add next
  quarter. Return an explicit DTO with the fields the UI needs.
- Type `req` (or delete it), add a test that a foreign account id returns 404, and one that
  the P&L uses the point value.

**Nits**

- Route: `/accounts/:accountId/positions` (or `/me/positions` if the account comes from the
  session) reads better than `positions/:accountId`.
- `logger.log` at info level on a hot read path is noise even when it's safe; debug level.

Happy to pair on the tenant-scoping change this afternoon. It's a ten-minute fix and it's the
one I'd like to see landed before anything else in this PR.

## Task 5 — Architecture and handoff reflection

1. **The decision I'm most proud of, and the trade-off it cost me.** Putting `broker_id` on
   every tenant-bearing row and backing it with composite foreign keys and a
   `TenantContext`-first repository interface. It means every query is scoped with one
   predicate, the schema itself rejects a row that disagrees with its parent's tenant, the
   main index leads with the tenant, and row-level security could be attached without a
   join. The cost is on the write side and in ceremony: the seed and the fill simulator have
   to derive the tenant from the parent row instead of trusting input, there are two extra
   unique indexes on small tables, and reference data (instruments, marks) needed documented
   exceptions to the "every port takes a tenant" rule.
2. **The one thing I'd change with a second day.** Sequence numbers on the fill stream. Today
   a fill event is only a hint to refetch, which is safe but blunt: after a reconnect the
   client refetches the whole snapshot instead of asking for "everything since sequence N".
   With per-account sequence numbers in both the snapshot and the events, the client could
   detect a gap arithmetically, the server could replay just the gap, and the widget could
   render fill by fill without trusting its own math. Close behind: a `positions` projection
   so the snapshot is not a full replay, and a non-superuser database role so row-level
   security also covers the pre-authentication paths.
3. **If I handed this repo to two engineers tomorrow: what I'd tell them first, and what I
   would not let them change.** First: the tenant rule and where it lives. The principal
   comes only from the API-key row, every repository method takes a `TenantContext`, every
   query spells `brokerId`, and the cross-tenant test is the one test they are not allowed to
   delete. I'd walk them through the Task 4 PR above as the exact shape of the bug we are
   guarding against. What I would not let them change without a design conversation: the
   `TenantContext`-first port signatures, the composite foreign keys and the
   `fills(broker_id, account_id, filled_at)` index, the socket design where the server derives
   rooms from the principal and there is no subscribe message, the rule that the fill
   simulator and Swagger only exist outside production, and the pre-commit hook that keeps
   dataset files out of git. Everything else, the UI, the P&L method, the module layout, the
   risk thresholds, is theirs to improve.
