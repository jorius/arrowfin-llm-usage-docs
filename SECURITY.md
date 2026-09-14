# Security and PII — Trader Daily Snapshot

Task 3 of the assessment. Every claim below points at a file and, where it matters, a line.
Paths are relative to the backend repository
(<https://github.com/jorius/arrowfin-mt-daily-snap-svc>) unless prefixed with `frontend/`,
which means the widget repository (<https://github.com/jorius/arrowfin-mt-daily-snap-fe>).

## 1. Auth model

**REST.** The browser never holds a password. It exchanges a trader id and secret once at
`POST /auth/api/key` for an opaque key (`afk_` + 32 random bytes, base64url,
`src/services/api-key.service.ts:10`). The service stores only `sha256(key)` in `api_keys`,
together with the `trader_id` and the `broker_id` it read from the trader's own row; the
client cannot choose its tenant. Every request then carries `Authorization: Bearer afk_…`;
`ApiKeyGuard` (`src/controllers/guards/api-key.guard.ts:16-19`) hashes the presented key,
loads the row where `revoked_at IS NULL AND expires_at > now()`, and attaches
`Principal { traderId, brokerId }` to the request. Nothing in the path, query or body can
change the principal.

**Secret storage.** Secrets are hashed with scrypt (N=16384, r=8, p=1, 64-byte key, per-user
salt) using Node's built-in `crypto.scrypt` and compared with `timingSafeEqual`
(`src/services/secret-hasher.service.ts`). When the trader id does not exist the service still
runs one scrypt verification against a dummy hash
(`src/application/auth/issue-api-key.usecase.ts:24`), so the response time does not reveal
which ids exist. Suspended traders (`kyc_status = suspended`) are refused with 403 only after a
correct secret, so the status is not an oracle.

**WebSocket.** Socket.IO's handshake carries the same key in `auth.apiKey`. A server-side
middleware registered in `afterInit` (`src/controllers/snapshot.gateway.ts:28-34`) verifies it
once, before the connection is accepted. The socket is then joined to
`broker:<brokerId>:account:<accountId>` rooms for the accounts the principal owns
(`snapshot.gateway.ts:46`), computed server-side from the same tenant-scoped repository as the
REST path. There is no client-to-server "subscribe" event at all, so a trader from one tenant
has no message they could send to reach another tenant's stream; the only way into a room is
to hold a key issued for that tenant. The `hello` event sent after admission carries heartbeat
configuration only.

**Why opaque keys instead of JWT.** Revocation is immediate (`DELETE /auth/api/key` flips
`revoked_at`; the next request fails), there is no signing key to rotate, and the cost is one
indexed lookup on `api_keys(key_hash)` per request. The trade-off is that cost: in production
the lookup would sit behind a short-TTL Redis cache keyed on the hash, with revocation deleting
the cache entry.

## 2. Tenant isolation

Where it is enforced, outermost first:

1. **Guard** — `src/controllers/guards/api-key.guard.ts:16-19`: the key is taken from the
   `Authorization` header only, resolved to an `api_keys` row, and the principal is whatever
   that row says; no key, no principal, 401.
2. **Application** — every repository port method that touches tenant data takes
   `TenantContext { brokerId, traderId }` as its **first** argument
   (`src/application/ports/*.repository.ts`). You cannot call the data layer without naming a
   tenant; there is no overload without it. The snapshot use case
   (`src/application/snapshot/get-account-snapshot.usecase.ts:27-29`) resolves the account
   through `findOwned(ctx, accountId)` and returns 404 when it is not the caller's.
3. **Repositories** — every Prisma query spells the tenant in `where`: the account lookup is
   `{ id: accountId, brokerId: ctx.brokerId, traderId: ctx.traderId }`
   (`src/infrastructure/prisma/prisma-accounts.repository.ts:27`) and the snapshot replay is
   `{ brokerId: ctx.brokerId, accountId, filledAt: { lte: upTo } }`
   (`src/infrastructure/prisma/prisma-fills.repository.ts:17`).
4. **Schema** — `broker_id` is denormalized onto `accounts`, `fills` and `api_keys`, with
   composite foreign keys `accounts(trader_id, broker_id) → traders(id, broker_id)` and
   `fills(account_id, broker_id) → accounts(id, broker_id)` (`prisma/schema.prisma`,
   `prisma/migrations/20260914154020/migration.sql`). A row that claims the wrong tenant cannot
   be inserted.
5. **Database** — row-level security
   (`prisma/migrations/20260914155429_row_level_security/migration.sql`): `traders`,
   `accounts`, `fills` and `api_keys` have `tenant_isolation` policies on
   `current_setting('app.broker_id')`, and tenant reads run inside
   `PrismaService.forTenant` (`src/infrastructure/prisma/prisma.service.ts:30-33`), which
   opens a transaction, runs `SET LOCAL ROLE arrowfin_tenant` (a `NOBYPASSRLS` role) and
   `set_config('app.broker_id', …, true)` before the query.
6. **Tests** — `get-account-snapshot.usecase.spec.ts`: the same account id requested by a
   principal from another broker returns `NotFoundException`; the guard spec: no key → 401;
   an RLS integration spec runs the tenant role against the real database and expects zero
   rows for another tenant's account.

**"Someone forgets `WHERE broker_id = ?` six months from now."** Three things catch it before
production: the port signature (you have to receive a `TenantContext` to call anything, and
reviewers see an unused one), the query plan (a query without `broker_id` cannot use
`fills(broker_id, account_id, filled_at)`, which shows up as a seq scan in the PR's `EXPLAIN`),
and the cross-tenant test that every repository must pass. What catches it in production is
the database: inside `forTenant` the forgotten `WHERE` returns nothing instead of everything.
The honest limit: RLS only applies inside `forTenant`. The connection owner is a superuser (the
only role Railway's Postgres hands out), which always bypasses RLS, so pre-auth lookups
(credentials, key hash), the seed and the dev simulator run unscoped by design. Forgetting the
`WHERE` **and** the wrapper is not caught; in production the app would connect as a
non-superuser role with `FORCE ROW LEVEL SECURITY` on every table so the wrapper is not
optional either.

## 3. PII handling

**Considered sensitive.** On `traders`: name, email, phone, date of birth, `ssn_last4`,
address, and above all `notes` and `audit_notes`, which in this dataset contain bank account and
routing numbers, a Swedish personnummer, a UK National Insurance number, S3 paths to KYC
documents, compliance case numbers, a spouse's phone number and an instruction that a trader
must not be surfaced in client-facing views. On `accounts`: `balance` and `buying_power`
(regulated under the rules cited in the brief). Trader secrets and API keys are credentials, not
PII, but they get the same treatment.

**Logged.** Trader id, broker id, account id, room counts, request outcome
(`src/controllers/snapshot.gateway.ts:47` is the only log line with identifiers). Nothing
else: no request bodies (the login body carries the secret), no entity rows, no fill payloads.

**Redacted / never selected.** The snapshot path never selects a single PII column: the
repositories `select` explicit fields, and the only query that touches `traders` is the
credential lookup, which selects `id`, `broker_id`, `kyc_status`, `credential.secret_hash` and
the broker's white-label name. `notes` and `audit_notes` are loaded once by the seed and never
read again by application code. Secrets never leave the seed process except into
`.local/trader-secrets.json` (gitignored) and are never printed. The OpenAPI schemas under
`src/application/dto/*.response.ts` contain no PII field, so the documented API surface cannot
grow one by accident without a visible diff.

**DB vs. socket.** In the database PII is at rest, access-controlled, and would be encrypted
per column (KMS envelope, see §5); the risk is bulk exposure through a bad query, which the
tenant scoping and the selective `select`s address. On the socket the risk is per-message
misdelivery to the wrong client and persistence in browser dev tools, proxies and client logs,
so the socket carries the minimum: `{ id, accountId, symbol, side, quantity, price, filledAt }`.
No names, no balances, no notes. The snapshot itself (which does contain the balance) goes over
REST with TLS and a per-request key check, not over the stream.

## 4. One vulnerability I didn't introduce

**IDOR on `GET /accounts/:accountId/snapshot`.** The Task 4 PR shows the shape of the bug: the
id in the URL is the only filter. In this service the id in the URL is never enough. The use
case (`src/application/snapshot/get-account-snapshot.usecase.ts:27`) begins with
`this.accounts.findOwned(ctx, accountId)`, and the repository turns that into
`where: { id: accountId, brokerId: ctx.brokerId, traderId: ctx.traderId }`
(`src/infrastructure/prisma/prisma-accounts.repository.ts:27`): three keys, two of which come
from the API-key row and none from the client. A foreign id returns 404
(`get-account-snapshot.usecase.ts:29`), the same response as a non-existent id, so the endpoint
cannot be used to enumerate which account ids exist in other tenants. The test that pins this:
requesting `ACC-1006` (Summit) as a Meridian principal → `NotFoundException`; the same check
was run against the deployed staging environment (404) before submission.

The same rule holds for the stream: `src/controllers/snapshot.gateway.ts:28-34` refuses the
handshake unless the key resolves, and `:46` joins the socket only to rooms built from the
principal's own accounts. There is no `@SubscribeMessage` handler in the gateway at all.

Runner-up worth a line: **user enumeration through timing** —
`src/application/auth/issue-api-key.usecase.ts:24` verifies against a dummy scrypt hash
(`src/services/secret-hasher.service.ts:26-30`) when the trader id does not exist, so a wrong
id and a wrong secret cost the same time and return the same 401.

## 5. What I'd do in production but didn't

Rate-limit `POST /auth/api/key` per trader id and per IP with a lockout after repeated
failures; audit-log every key issuance, revocation and cross-tenant 404 with the principal and
the requested id, because that 404 is an attack signal, not noise; structured logging with an
allow-list of fields and a redaction filter as the last line of defence; Redis-cached key
lookups with revocation invalidating the cache; KMS envelope encryption for `notes`,
`audit_notes`, `ssn_last4`, `dob` and `address_line1`, with the KYC document references moved
to a separate documents service behind its own access control; a non-superuser database role
so row-level security also covers the pre-authentication paths; a `positions` projection
maintained from fills so the snapshot is O(open positions) instead of a full replay; sequence
numbers on the fill stream so a reconnecting client can ask for fills since `lastFillId`; and,
on the browser side, a Content-Security-Policy with `connect-src` pinned to the API origin, a
backend-for-frontend so REST uses a same-site `httpOnly` cookie and the socket gets a
single-use, seconds-lived ticket minted by that BFF, key rotation on every reconnect, a
build-time check that the `NEXT_PUBLIC_*` values point at the expected environment, and
component tests for the connection state machine so a regression that shows "Live" while
disconnected cannot ship.

## 6. Development-only surfaces

**Fill simulator.** `POST /dev/fills` is registered only when `DEV_FILLS_ENABLED === 'true'`
**and** `NODE_ENV !== 'production'` (`src/infrastructure/config/env.ts:45`,
`src/app.module.ts:10`). In production the route does not exist; there is no guard to
misconfigure. The fill's `broker_id` is derived from the account row, never from the request
body, and the event is emitted only to that account's room.

**Swagger.** `/docs` and `/docs-json` follow the same pattern: `SWAGGER_ENABLED` defaults to
true and is forced off under `NODE_ENV=production` (`src/main.ts:21`).

**Hosted environments.** Both Railway environments (`staging`, `development`) run with
`NODE_ENV` set to their own name, so the simulator and Swagger are available for the demo.
Neither is production; a production environment would set `NODE_ENV=production`, which makes
both flags inert regardless of their values. That guarantee is in code, not in configuration.

## 7. The browser side

**Where the API key lives.** `frontend/lib/auth.ts` keeps the opaque key returned by
`POST /auth/api/key` in `sessionStorage` (tab-scoped, gone when the tab closes). It is sent as
`Authorization: Bearer` by `frontend/lib/api.ts` and as `auth.apiKey` in the Socket.IO
handshake by `frontend/lib/socket.ts`. It is never written to `localStorage`, the URL, or a
log line. The client never compares `expiresAt` with its own clock: expiry and revocation are
the server's decision, and any `401` clears the session and redirects to `/login`
(`frontend/app/snapshot/page.tsx`, `signOut`).

**Trade-off accepted.** Anything JavaScript can read, an XSS payload can read. An `httpOnly`
cookie would protect the REST path but cannot be attached to a cross-origin WebSocket
handshake from the browser; the mitigation here is a key that is revocable server-side and
short-lived (12 h), plus rendering that never bypasses React's escaping (no
`dangerouslySetInnerHTML`; API error messages are rendered as text). For a real white-label
portal I would put a backend-for-frontend in front of the API, as described in §5. It was out
of scope for the time budget and I chose to say so rather than half-build it.

**What the client sends.** Trader id and secret once, at login; an account id in a path;
nothing over the socket. `frontend/hooks/useFillStream.ts` only registers listeners: there is
no subscribe message, so there is nothing a modified client could send to ask for another
tenant's stream. The `event.accountId === selectedAccount` check in the hook is a UX filter,
not a security boundary: the server only ever emits to rooms derived from the authenticated
trader's own accounts.

**What the client receives.** Its own account list (`id`, number, type, status), its own
snapshot, `fill` events limited to `{ id, accountId, symbol, side, quantity, price, filledAt }`,
and a `hello` event with heartbeat configuration only. No names, contact data, notes or other
traders' rows reach the browser; the only regulated figure shown is the trader's own balance,
inside the snapshot.

**Rendering.** All values are rendered through React's escaping; numbers come from the server
(the client does not compute P&L or risk); translations are static dictionaries compiled into
the bundle, not user-supplied strings.
