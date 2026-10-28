# Architectural Decisions

Decisions made during BroadcastMail development. Each entry covers what was chosen, what else was on the table, and why it landed where it did.

---

## Auth: sessions over JWT

Spring Boot backend + browser SPA. The typical argument for JWT is statelessness, but stateless auth only matters when you have multiple backend instances that can't share memory. BroadcastMail runs as a single Railway service, so the benefit disappears. Worse, JWT logout requires a denylist, and maintaining a denylist recreates exactly the server-side state you were trying to avoid. Sessions are simpler and give real revocation for free.

---

## Cookie strategy: SameSite=None with CSRF double-submit

The frontend is on Vercel, the API on Railway: different origins. SameSite=Strict blocks the cookie cross-origin, SameSite=Lax only sends it on top-level navigations. SameSite=None (with Secure) is the only option that works for a cross-origin SPA making fetch requests.

That opens up CSRF since the browser will attach the cookie to any cross-origin request. Defense is CSRF double-submit: the server sets a second cookie with a random token, the client reads it and sends it back as a request header, the server checks they match. An attacker's site can trigger a request with the session cookie but can't read the CSRF cookie (cross-origin reads are blocked), so it can't forge the header.

Additional hardening: the CSRF token is tied to the session and regenerated on each login, which also mitigates session fixation. Cookies are HttpOnly and Secure. Sessions expire after 30 days.

---

## Three-service Railway deployment, shared database

Three separate Railway services: `broadcastmail-api`, `broadcastmail-worker`, `broadcastmail-webhooks`. All share a single Railway PostgreSQL instance over the internal `.railway.internal` network.

The services have different scaling requirements and failure modes. The worker processes the outbox in a tight loop; the webhook receiver needs to absorb Resend traffic spikes independently. Separate deployments mean they can be restarted, scaled, or replaced without affecting the others. The shared database keeps the data model simple, with no cross-service HTTP calls or message bus needed for the core workflow.

---

## Modular monolith inside each service, package-by-feature

Packages organized by feature (`campaigns`, `recipients`, `auth`) rather than by layer (`controllers`, `services`, `repositories`). Each feature owns its controller, service, repository, and domain objects.

PayFlow used package-by-layer because it had one deep domain (payments) with significant cross-cutting concerns. BroadcastMail has several independent features with little overlap. Package-by-feature makes each slice self-contained: adding or modifying a feature means touching one package rather than four.

---

## Outbox pattern over WAL CDC

Campaign send requests write to an `outbox_entries` table in the same transaction as the campaign state change. A separate poller picks up entries and calls Resend.

WAL CDC (change data capture from the Postgres write-ahead log) would have worked here, but it makes the most sense when you're reacting to writes you don't control, like another service's database. BroadcastMail owns every write in this flow. The outbox pattern gives explicit control over retry logic, failure semantics, and dead-letter handling without needing a CDC connector. Railway-managed Postgres also makes WAL slot management an unnecessary operational burden.

---

## FOR UPDATE SKIP LOCKED for worker concurrency

The outbox poller claims rows with `SELECT ... FOR UPDATE SKIP LOCKED` before processing them. This pushes concurrency control into the database rather than the application layer.

`SKIP LOCKED` means multiple worker threads can poll the same table simultaneously without blocking each other: each thread atomically claims a row that no other thread holds. Application-level locking (Redis distributed locks, in-memory flags) requires a separate system and introduces failure modes when the lock holder crashes. The database handles it correctly by default.

The claim step (`PROCESSING` status update) runs in its own short transaction, separate from the Resend API call. Holding a transaction open across an external HTTP call would pin a connection and block other work.

---

## BYOK Resend model

Users provide their own Resend API key. BroadcastMail stores it (encrypted at rest) and uses it to send on their behalf. BroadcastMail never holds a shared sending pool.

This shifts the trust boundary: the user is responsible for their own Resend account, rate limits, and domain verification. BroadcastMail's job is to drive Resend correctly. Email delivery costs don't scale with BroadcastMail's user count since there's no shared sending infrastructure. The tradeoff is that users have to set up a Resend account, which adds a small amount of onboarding friction.

---

## Two pricing tiers, not three

Free (500 unique recipients per 30 days) and Pro ($9/month, unlimited).

Three-tier pricing works when you have enough users to fill the middle tier. At early traction (50-100 users), most people are on free, a few upgrade, and the middle tier stays empty. Two tiers creates one clear conversion moment: hit the recipient cap or want filters, pay $9. The upgrade event is obvious and the enforcement logic is simpler.

---

## Rolling 30-day window via PeriodResetJob

The free tier limit is 500 unique recipients per rolling 30-day period, not per calendar month. `period_reset_at` is set to `now() + 30 days` when an account is created or reset. A scheduled `PeriodResetJob` runs a single native query: `UPDATE accounts SET unique_recipients_this_period = 0, period_reset_at = now() + interval '30 days' WHERE period_reset_at <= now()`.

Calendar-month resets create a cliff effect: users burn through their limit in the first week of the month and hit a wall. A rolling window distributes usage more evenly and feels fairer.

---

## Plan enforcement via Strategy + Facade

`PlanEnforcementFacade` wraps the plan check. Each tier implements a `PlanEnforcementStrategy`. The facade resolves the right strategy at runtime based on the account's plan.

The alternative was a chain of if/else checks scattered across services. The Strategy pattern keeps plan logic in one place and makes adding a third tier later a matter of adding a new strategy class rather than auditing the codebase for enforcement points.

---

## Caffeine over Redis for rate limiting and caching

Rate limiting on the test-send endpoint uses Caffeine (in-process cache). Caching for `ResendClient` (1h per API key) and `EmailSendService` (30s) also uses Caffeine.

BroadcastMail runs as a single Railway instance per service. Redis would add a network hop and an additional service to operate for no benefit. If BroadcastMail ever runs multiple API instances, the rate limit becomes per-instance rather than global, and at that point moving to Redis is the right call. For now, Caffeine is simpler and fast.

---

## Webhook idempotency via UNIQUE constraint

`webhook_events` has a `UNIQUE` constraint on `provider_event_id` (the Svix `svix-id` header). Duplicate webhook deliveries fail the insert and are silently discarded.

Application-level deduplication (check-then-insert) has a race condition: two threads can both pass the check before either inserts. A database constraint handles this correctly without coordination overhead.

---

## CampaignCompletionService: single native query with COUNT(*) FILTER

`countStatusesByCampaignId` is a single native query using `COUNT(*) FILTER (WHERE status = ...)` to count delivered, opened, bounced, and failed recipients in one database round trip.

Three separate queries (one per status) were replaced because the counts are always read together. One query is faster and keeps the data consistent: the counts reflect the same snapshot of the table.

---

## Unsubscribe links: custom JJWT tokens over provider-managed

Unsubscribe links are signed with JJWT using a shared `UNSUBSCRIBE_SECRET` environment variable. Tokens expire after 30 days. The `List-Unsubscribe` header is appended to every outgoing email.

Resend's BYOK model doesn't provide managed unsubscribe links for transactional sending. Building it with JJWT keeps the implementation inside the existing codebase, avoids another external dependency, and gives explicit control over the expiry and validation logic. The `UNSUBSCRIBE_SECRET` is shared between `broadcastmail-api` and `broadcastmail-worker` since both need to generate and validate tokens.

---

## Cursor-based pagination over offset

Recipient and campaign list endpoints use cursor-based pagination. Offset pagination degrades as the table grows: `OFFSET 5000` scans and discards 5000 rows. Cursor pagination uses an indexed column (typically `id` or `created_at`) as the anchor, so performance stays constant regardless of how far into the list you are.

---

## Verification token errors collapse to 400

Email verification endpoints return a generic 400 for any token error: expired, not found, already used. No detail about which case applies.

Returning distinct errors (404 for not found, 410 for expired) would let an attacker probe which tokens exist and which have been used. A collapsed response gives nothing away.

---

## broadcastmail-common: shared Maven library via GitHub Packages

Entities, repositories, and shared DTOs used by more than one service live in `broadcastmail-common`, published to GitHub Packages. Services declare it as a Maven dependency.

The alternative was duplication across services, which creates drift over time. Publishing to GitHub Packages keeps the library private without requiring a separate artifact registry. The CI pipelines authenticate with `server-id: github` using `GITHUB_TOKEN`.

---

## Variable substitution: allowedColumns whitelist + HTML escaping

`{{variable}}` placeholders in campaign content are resolved server-side at send time. Only columns in `allowedColumns`, derived from the filterable columns configuration, can be referenced. Values are HTML-escaped before insertion (`&`, `<`, `>`, `"`).

Allowing arbitrary column names would let a user reference columns they shouldn't (internal flags, credentials stored in the database). The whitelist matches the columns already exposed for filtering, which keeps the surface area consistent. HTML escaping prevents a recipient's profile data from being used to inject markup into the campaign content.
