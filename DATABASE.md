# Database design

## Overview
Broadcast email uses a single PostgreSQL instance hosted on Railway. All changes to
the schema are done through Flyway migrations (Hibernate is set tp `validae` 
only so it never modifies directly).

The core principles that schema design follow:

**Outbox pattern**
- Campaigns are never sent synchronously during a web request. Instead, on campaign
confirmation, recipient rows are written anatomically first. Then, a background worker
pools the outbox and sends the emails via a third-party SMTP provider (default is
Resend) If the worker crashes mid-send, it is retried and continues exactly where it 
left off (no duplicate sends, no lost recipients)

**Idempotency**
- Every campaign_recipient rows have a unique `idempotency_key` 
(structured like `campaign_id+external_user_id`). Webhook events are
deduplicated by `provider_event_id`. This means no operation can be applied twice
with the same result.

**Multi-tenant credential isolation**
- Every customer's Supabase credentials and email provider keys are encrypted AES-256-GCM using a
  pass key stored in Railway environment variables. Credentials are
  never stored in plaintext and never logged.

---

## Schema Design Decisions
**Why `external_user_id` has no foreign key:**
`external_user_id` references a user in the customer's Supabase
database (completely separate Postgres instance we don't control;
PostgreSQL cannot enforce referential integrity across database
boundaries). The application layer enforces this so we only write
`external_user_id` values that came directly from querying their
Supabase, so they are guaranteed to be real IDs at the time of
writing.

**Why `idempotency_key` is a UNIQUE constraint not just an index:**
A UNIQUE constraint enforces the invariant at the database level (even
if the application has a bug and tries to insert the same recipient
twice, the database rejects it) An index alone would not prevent the
duplicate insert. The constraint is the safety net.

**Why the outbox index is partial (`WHERE status = 'pending'`):**
The worker only ever queries `WHERE status = 'pending'`. Rows in
`done` or `failed` status are never polled again. A partial index only
indexes the rows the worker actually cares about, because as rows complete, 
they fall out of the index automatically, keeping it small and fast
regardless of total table size.
**Why campaign stats are denormalized onto the campaign table:**
`sent_count`, `delivered_count`, `opened_count`, `bounced_count`,
`failed_count` are stored directly on `campaigns` rather than computed
via `COUNT()` from `campaign_recipients`. Computing live counts on a
table with potentially thousands of rows on every dashboard load would
be slow. These counters are incremented atomically as webhook events
are processed, so a small normalization tradeoff for significantly
better read performance on the dashboard.
**Why `webhook_events.campaign_recipient_id` is nullable:**
Resend occasionally fires webhook events for emails we can't match back
to a recipient, e.g., if the recipient row was deleted during
the retention cleanup window, or if Resend fires an event for a test
email sent during onboarding. Making this nullable means we store the
raw event regardless and don't lose data, even when the match fails.

**Why `unsubscribes` is a separate table not a column on `campaign_recipients`:**
An unsubscribe is permanent and applies to all future campaigns, not
just the campaign the recipient originally received. Storing it as a
column on `campaign_recipients` would only capture the unsubscribing
against one campaign and future campaigns would have no way to check it
without a full table scan. A separate `unsubscribes` table with a
UNIQUE constraint on `(account_id, email)` makes the check fast and
correct across all campaigns.

## Access Patterns

### accounts
**Writes:**
 - INSERT once on account registration — low frequency
 - UPDATE on email verification, plan upgrade, password change, 
 stripe ID assignment — low frequency 

**Reads**
 -  SELECT WHERE email = ? — on every login attempt
 - SELECT WHERE api_key_hash = ? — on every authenticated API request 
 (the hottest read in the system)
 - SELECT WHERE id = ? — PK lookup after authentication, always fast

**Growth:** One row per customer. 1,000 customers = 1,000 rows. Never a concern.

**Deletion:** Hard delete on account closure. ON DELETE CASCADE cleans everything downstream.

**Indexes:**
```sql
CREATE INDEX idx_accounts_email ON accounts(email);
-- justification: login query, hits on every login attempt
 
CREATE INDEX idx_accounts_api_key_hash ON accounts(api_key_hash);
-- justification: API key auth, hits on every single authenticated request
```

---

### team_members

Not used in V1. Added in V2 when agency tier is introduced.
Access pattern to be documented at that point.
 
---

### management_oauth_tokens


**Writes:**
- INSERT once when the user completes Supabase Management API OAuth flow
  during onboarding Step 1 — extremely low frequency
- UPDATE when the access token is refreshed — access tokens expire every
  hour, a refresh token is used to get a new one automatically.
  Updates access_token + expires_at on the existing row.

**Reads:**
- SELECT WHERE account_id = ? — when BroadcastMail needs to call the
Supabase Management API (run CREATE ROLE SQL during onboarding,
re-introspect schema if the user changes filterable columns).
Not a hot read — only fires on specific user actions, not on
every request.
- After read: check expires_at. If expired, refresh before
    making the API call, then update the row.

**Growth:**
One row per customer. 1,000 customers = 1,000 rows. Never a concern.

**Deletion:**
- CASCADE on account deletion — when the account row is deleted,
  this row goes with it automatically via ON DELETE CASCADE
- Explicitly deleted if a user disconnects their Supabase connection
  (revokes BroadcastMail's OAuth access)

**Indexes:**
 ```sql
CREATE INDEX idx_management_oauth_tokens_account_id
  ON management_oauth_tokens(account_id);
-- justification: load token for account before every
-- Management API call, always queried by account_id
```
---

### connections

**Writes:**
- INSERT once during onboarding when OAuth completes and CREATE ROLE
  SQL runs successfully — extremely low frequency,
- UPDATE estimated_user_count as a side effect of recipient preview
  queries in the campaign editor — opportunistic, not scheduled, low frequency
- UPDATE estimated_user_count when a user refreshes schema in Settings
  — low frequency, most users never return to Settings after onboarding

**Reads:**
- SELECT WHERE account_id = ? — dashboard load, campaign editor open,
  settings page. Most common read, always by account_id.
- SELECT WHERE id = ? — PK lookup by worker when processing outbox
  rows. Needs encrypted_creds to call their Resend account.
  Always fast, covered by PK.

**Growth:**
V1: one row per account. 1,000 customers = 1,000 rows. Negligible.

**Deletion:**
- Explicit DELETE when the user disconnects in the Settings → Connection tab.
  Blocked if any campaign has status = 'sending' — worker would lose
  credentials mid-send. User sees: "A campaign is currently sending emails,
  wait for it to finish before disconnecting."
- CASCADE DELETE when an account is closed — ON DELETE CASCADE handles
  automatically, no manual deletion needed in code.
- Cascade cleans up: filterable_columns rows are deleted automatically.
  Completed campaigns (sent/failed) cascade deleted too — acceptable
  since the user is intentionally disconnecting and the retention policy 
  deletes history anyway.

**Indexes:**
```sql
CREATE INDEX idx_connections_account_id
  ON connections(account_id);
-- justification: SELECT WHERE account_id = ? on every dashboard
-- load, campaign editor open, and settings page — most frequent
-- read in this table

-- UNIQUE constraint on (account_id, project_url) creates an implicit
-- index automatically — no separate index needed for project_url
```
 
---

### filterable_columns


**Writes:**
- Bulk INSERT during onboarding Step 1 after schema introspection —
  multiple rows at once, one per detected column. Pre-ticked based
  on heuristics (BOOLEAN, low-cardinality TEXT, TIMESTAMPTZ) —
  extremely — Low frequency happens once per account lifetime
- UPDATE enabled flag when a user ticks/unticks columns in
 the Settings → Connection tab. — low frequency most users never return
  to Settings after onboarding.
- UPDATE cardinality + cardinality_warning opportunistically
  when distinct values are loaded in campaign editor filter
  builder. Side effect of querying their Supabase — no extra
  API call needed.
**Reads:**
-  SELECT WHERE connection_id = ? AND enabled = true — when
   campaign editor opens, populates the column picker dropdown.
   Most frequent read. column_type, cardinality, and
   cardinality_warning are all on the same row — no separate
   queries needed. Operators and UI decisions derived client-side
   from column_type.
- Run as a separate query alongside the connections load —
  not a JOIN. Two clean repository calls, no complex result
  mapping needed.

**Growth:**
- ~7 rows per connection on average. At 1,000 customers:
  1,000 connections × 7 columns = 7,000 rows total. Never
  a storage concern.

**Deletion:**
- CASCADE when connection is deleted — ON DELETE CASCADE
  handles automatically, no manual deletion in code
- CASCADE when an account is closed — flows through connections
  cascade automatically
- Individual rows are never hard deleted ( disabled columns
  stay in the table with enabled = false so user can
  re-enable them in Settings. Toggle only, never delete.)

**Indexes:**
```sql
CREATE INDEX idx_filterable_columns_connection_id
  ON filterable_columns(connection_id);
-- justification: SELECT WHERE connection_id = ? AND enabled = true
-- on every campaign editor open

-- UNIQUE constraint on (connection_id, column_name) creates an
-- implicit composite index automatically. Postgres can use this
-- for WHERE connection_id = ? queries as a leftmost prefix match.
-- Standard index above may be redundant — worth checking query
-- plan with EXPLAIN ANALYZE before keeping both.
```
---

### email_providers

**Writes:**
- INSERT once during onboarding Step 2 when a user connects their
  Resend or SES account — extremely low-frequency
- UPDATE when a user changes provider or from address in
    Settings → Email provider tab — low-frequency , most users never
    change their provider after setup
- No DELETE on provider change, always UPDATE existing row bc provider can't be null

**Reads:**
- SELECT WHERE account_id = ? — from the Settings page to show
  current provider details
- SELECT WHERE account_id = ? — from fan-out worker before
  every Resend API call, needs encrypted_api_key + from_address
  to send emails on their behalf

**Growth:**
One row per account. 1,000 customers = 1,000 rows. Never
a storage concern.

**Deletion:**

- CASCADE when account is closed — ON DELETE CASCADE handles
  automatically, no manual deletion needed in code
- Never explicitly deleted — no "disconnect provider" action
  exists in the UI. Provider row lives for the lifetime
  of the account. Changing provider = UPDATE, not DELETE and INSERT.

**Indexes:**
**Indexes:**
```sql
CREATE INDEX idx_email_providers_account_id
  ON email_providers(account_id);
-- justification: SELECT WHERE account_id = ? from both
-- Settings page and fan-out worker. Table stays small
-- but index added early to avoid production ALTER TABLE
-- under load later.
```
---

### campaigns


**Writes:**
- INSERT when a user creates a new campaign, it starts in draft status.
  Low frequency
- UPDATE status draft → sending when the user clicks the sent button —
happens atomically in the same @Transactional block as the recipient
  snapshot and outbox row creation. If any part fails, campaign
  stays in draft.
- UPDATE status sending → sent when worker processes last outbox
  row and no pending/processing rows remain for this campaign.
- UPDATE status sending → partially_failed when worker finishes
  and >10% of recipients have failed status.
- UPDATE status sending → failed when all recipients failed.
- UPDATE sent_count, delivered_count, opened_count, bounced_count,
  failed_count — incremented atomically each time a webhook event
  is processed. Written by webhook receiver, not the worker.
  These denormalised counters avoid COUNT() aggregation queries
  on campaign_recipients on every dashboard load.
- UPDATE scheduled_at — Pro only, V2 backend. User schedules
  campaign for future send. Worker checks scheduled_at <= now()
  before processing.

**Reads:**
- SELECT WHERE account_id = ? ORDER BY created_at DESC —
  dashboard campaign list. Most frequent read, always by
  account_id. sent_count, delivered_count, opened_count,
  bounced_count, failed_count all returned on same row —
  no extra aggregation query needed.
- SELECT WHERE id = ? — campaign detail page and worker
  loading campaign to get subject + body for email send.
  PK lookup, always fast, covered by PK index automatically.
  **Growth:**
  ~3 campaigns/month per customer. At 1,000 customers over 12 months
  = 36,000 total rows ever. Retention policy keeps live rows bounded:
- Free (7 days):  ~525 live rows
- Pro (90 days):  ~2,700 live rows
- Total live:     ~3,225 rows at any time

**Deletion:**
-  CASCADE on account deletion — automatic via ON DELETE CASCADE
- CASCADE on connection deletion — automatic. User warned in
  disconnect UI: "This will delete all campaign history for
  this connection." Acceptable since retention policy deletes
  history within 7-90 days anyway.
- Nightly retention cleanup job — primary deletion mechanism.
  Deletes campaigns WHERE sent_at < retention_cutoff for plan.
  Free = 7 days, Pro = 90 days. Cascade cleans campaign_filters,
  campaign_recipients, outbox, webhook_events automatically.
- 
**Indexes:**
 ```sql
CREATE INDEX idx_campaigns_account_id
  ON campaigns(account_id);
-- justification: SELECT WHERE account_id = ? ORDER BY created_at DESC
-- on every dashboard load — most frequent read in this table

CREATE INDEX idx_campaigns_status
  ON campaigns(status);
-- justification: retention cleanup job queries
-- WHERE status = 'sent' AND sent_at < retention_cutoff
-- also used by worker checking campaign completion

-- id covered by PK automatically
-- connection_id has no direct query — no index needed
```
---

### campaign_filters

**Writes:**
- Bulk INSERT during campaign creation in draft state — one row
  per filter condition the user adds. Most campaigns have 0-3
  filters. Free tier users always have 0 — filters are Pro only.
  Low frequency, low volume.
- INSERT triggered by user adding a filter condition in the
  campaign editor — "plan = free" = one row, "plan = free AND
  created_at > 2024-01-01" = two rows.
- No UPDATE ever — filters are immutable after creation.
  Campaign confirm locks everything. Changing filters after
  recipient snapshot is meaningless since recipients are already
  frozen.
- No individual DELETE — no "remove one filter" UI exists.
  Filters are set at campaign creation, never individually modified.

**Reads:**
- SELECT WHERE campaign_id = ? — at campaign confirmation to build
  the dynamic SQL query against their Supabase for recipient
  resolution. Most critical read — wrong filter = wrong recipients.
- SELECT WHERE campaign_id = ? — campaign detail page to show
  historical record of who was targeted ("sent to users where
  plan = free"). Read-only display after send.
- Both reads are the same query, same index covers both.

**Growth:**
~1-2 filter rows per campaign average. At 3,225 live campaigns
= ~4,800 live filter rows. Negligible. Free tier users have 0
filter rows always — only Pro users generate filter rows.

**Deletion:**
- CASCADE on campaign deletion — ON DELETE CASCADE handles
  automatically, no manual deletion in code
- CASCADE on account deletion — flows through campaigns
  cascade automatically
- CASCADE on connection deletion — flows through campaigns
  cascade automatically
- Never explicitly deleted — no individual filter deletion
  action exists in the UI

**Indexes:**
```sql
CREATE INDEX idx_campaign_filters_campaign_id
  ON campaign_filters(campaign_id);
-- justification: SELECT WHERE campaign_id = ? at campaign
-- confirm (recipient resolution) and campaign detail page.
-- Both reads always query by campaign_id.

-- filter_order has no direct query — used for ORDER BY only,
-- covered by the campaign_id index scan naturally.
```
 
---

### campaign_recipients

**Writes:**
- Bulk INSERT when campaign is confirmed — one row per recipient,
  potentially 500 rows at once on Pro tier. Written atomically in
   the same @Transactional block as outbox row creation and campaign
  status → sending transition. If any part fails, the entire transaction 
  rolls back — no partial recipient snapshots ever.
- UPDATE status queued → sent — by fan-out worker after Resend
  accepts sending emails. Also sets resend_message_id on the same UPDATE —
  needed for webhook matching later.
- UPDATE status → delivered/opened/bounced — by webhook receiver
  when Resend fires delivery events. Sets delivered_at, opened_at,
  bounced_at timestamps respectively.
- UPDATE status → failed — by worker after three retry attempts
  exhausted. Sets failed_reason with a Resend error message.
- UPDATE status → unsubscribed — when the recipient clicks the unsubscribe
  link. Permanent, never reversed.
- No DELETE during normal operation — rows live until a retention
  cleanup job runs.

**Reads:**
- SELECT WHERE campaign_id = ? — campaign detail page recipient
  table. Paginated — never load all recipients at once.
- SELECT WHERE resend_message_id = ? — webhook receiver matching
  incoming Resend event back to the recipient row. Hottest read in
  the entire system — fires on every delivery/open/bounce event.
  Must be fast — index on resend_message_id is critical.
- SELECT COUNT(*) WHERE campaign_id = ? AND status = ? — stat
  aggregation per status. Used by worker to check if a campaign is
  complete (no pending/processing rows remaining). Also used by
  webhook receiver to update denormalized campaign counters.
- SELECT WHERE campaign_id = ? AND status = 'failed' — retry
  flow, copies failed recipients into a new mini retry campaign (with
  fresh outbox rows). Failed rows are therefore kept on the original
  campaign after completion, not deleted.
- Each campaign may have at most one direct retry campaign. Further
  retries are started from that retry campaign, preserving each attempt
  as a separate campaign and preventing duplicate retries from the original.

**Growth:**
Most write-heavy table in the schema. At 1,000 customers:
- Average 3 campaigns/month × 300 recipients = 900 rows/customer/month
- 1,000 customers = 900,000 rows/month created
- Retention policy bounds live rows:
  Free (7 days):  ~210,000 live rows
  Pro (90 days):  ~270,000 live rows
  Total live:     ~480,000 rows at any time
  Largest table by far. Index performance here matters most.

**Deletion:**
- Nightly retention cleanup job — primary deletion mechanism.
  Deletes WHERE campaign.sent_at < retention_cutoff for plan.
  Cascade from campaign deletion handles outbox + webhook_events.
- CASCADE on campaign deletion — automatic.
- CASCADE on connection deletion → campaigns → recipients.
- CASCADE on account deletion — automatic.
- Never explicitly deleted individually during normal operation.

**Indexes:**
```sql
CREATE INDEX idx_campaign_recipients_campaign_id
  ON campaign_recipients(campaign_id);
-- justification: SELECT WHERE campaign_id = ? for detail page
-- and stats aggregation. Most frequent read after resend_message_id.

CREATE INDEX idx_campaign_recipients_resend_message_id
  ON campaign_recipients(resend_message_id);
-- justification: SELECT WHERE resend_message_id = ? on every
-- incoming webhook event. Hottest read in the system — Resend
-- fires hundreds of events per campaign. Without this index
-- every webhook event does a full table scan on 480,000 rows.

CREATE INDEX idx_campaign_recipients_status
  ON campaign_recipients(status);
-- justification: SELECT COUNT(*) WHERE status = ? for worker
-- campaign completion check and retry flow filtering.

-- idempotency_key UNIQUE constraint creates implicit index —
-- covers INSERT ON CONFLICT DO NOTHING dedup automatically.
-- resend_message_id UNIQUE constraint creates implicit index —
-- covered by explicit index above, no double-indexing needed.
```

**Critical note on idempotency:**
INSERT uses ON CONFLICT (idempotency_key) DO NOTHING — if worker
crashes mid-bulk-insert and retries, duplicate recipients are
silently ignored. No user ever receives the same campaign twice
regardless of how many times the insert is retried.

---

### outbox

**Writes:**
- Bulk INSERT on campaign confirmation — one row per campaign_recipient,
  status = pending. Atomic with campaign_recipients INSERT and
  campaign status → sending in the same @Transactional block. — low frequency 
- UPDATE pending → processing when worker picks up row via
  FOR UPDATE SKIP LOCKED (not serializable isolation — duplicate - 
  email acceptable, throughput matters more than strict serialization) — high frequency while sending
- UPDATE processing → done on Resend success. Sets last_attempted_at. —high frequency while sending
- UPDATE processing → pending on Resend failure, attempts < 3. — low-frequency
  Sets attempts++, next_attempt_at = now() + backoff (1m → 5m → 15m)
- UPDATE processing → failed after 3 attempts. Worker never 
  retouches. Retry button in the UI creates a new outbox row instead. — extremely low frequency

**Reads:**
- Worker poll every 5 seconds:
  SELECT WHERE status = 'pending' AND next_attempt_at <= now()
  ORDER BY next_attempt_at ASC FOR UPDATE SKIP LOCKED LIMIT 50
  Hottest read in the system — most critical index in schema.

**Reconciliation (two jobs):**
- Every 5 min: reset processing rows older than 5 min → pending
  (handles worker crash mid-batch)
- Nightly: alert if pending rows exist for sent campaigns
  (indicates completion logic bug)

**Growth:**
One row per campaign_recipient. 900,000 created/month at scale
but live pending/processing rows stay tiny — 300-recipient
campaign finishes in ~30 seconds at 10 sends/second.

**Deletion:**
CASCADE on campaign_recipient → campaign → account deletion.
Never explicitly deleted — reconciliation resets, never deletes.

**Indexes:**
```sql
CREATE INDEX idx_outbox_status_next_attempt
  ON outbox(status, next_attempt_at)
  WHERE status = 'pending';
-- composite partial index — done/failed rows fall out
-- automatically, index stays tiny regardless of table size
```
 
---

### webhook_events

**Writes:**
- INSERT on every Resend webhook received — one row per event.
  Event types: delivered / opened / bounced / clicked / failed. —
  high frequency during active campaign sent — 500 recipient
  campaign generates 1,000–1,500 events (multiple events per
  recipient). Fastest growing table in schema.
- UPDATE processed = true after the webhook receiver successfully
matches the event to campaign_recipient and updates their status.
  Frequency: HIGH — immediately after each insert is processed.
  Only write after initial insert — raw payload never changes,
  immutable record of what Resend reported.

**Reads:**
- SELECT WHERE provider_event_id = ? — deduplication check
  before processing any incoming event. Fires on every single
  webhook received. Hottest read in this table.
  UNIQUE constraint prevents duplicate inserts at DB level
  even if the application has a bug.
- SELECT WHERE processed = false AND created_at < now() - 5 min
  — a reconciliation job finds stuck unprocessed events. Low frequency — 
  runs every few minutes, rarely finds anything.

**Growth:**
2-3 events per recipient on average (delivered + opened/bounced).
1,000 customers × 3 campaigns/month × 500 recipients × 2.5 events
= 3,750,000 rows/month at scale. Fastest growing table —
retention cleanup is critical here.

**Deletion:**
- Nightly retention cleanup job — primary deletion mechanism.
  DELETE WHERE created_at < retention_cutoff for plan.
  Free = 7 days, Pro = 90 days.
- Independent of campaign_recipients deletion — no CASCADE.
  campaign_recipient_id set to NULL when recipient deleted,
  row stays until own retention window expires.
- No other deletion paths.

**Indexes:**
```sql
-- UNIQUE constraint on provider_event_id creates index automatically
-- no separate CREATE INDEX needed
-- justification: deduplication check on every incoming webhook,
-- DB-level guarantee against duplicate event processing

CREATE INDEX idx_webhook_events_unprocessed
  ON webhook_events(created_at)
  WHERE processed = false;
-- justification: reconciliation job finding stuck unprocessed events
-- partial index — processed rows fall out automatically,
-- index stays tiny regardless of 3.75M rows/month growth
```
 
---

### unsubscribes
### unsubscribes

**Writes:**
- INSERT when the recipient clicks the unsubscribe link in any campaign
  email — one row per email per account. — low-frequency
  small percentage of recipients unsubscribe per campaign.
- No UPDATE ever — unsubscribe is permanent and binary.
  No "re-subscribe" flow in V1.

**Reads:**
- SELECT WHERE account_id = ? AND email = ? — checked before
  every recipient is added to the campaign snapshot. If a row exists,
 the recipient is silently excluded from campaign_recipients.
  — high frequency, one check per recipient per campaign confirmation.
  500 recipients = 500 checks per confirmation.
- SELECT WHERE account_id = ? — Settings → Connection tab,
  show an unsubscribed list to an account owner. — low frequency
  only when the user visits Settings.

**Growth:**
One row per unsubscribed email per account. Grows slowly —
maybe 1-2% of recipients unsubscribe per campaign. At scale:
1,000 customers × 500 recipients × 2% = 10,000 rows total
ever. Never a storage concern.

**Deletion:**
- CASCADE on account deletion — automatic via ON DELETE CASCADE.
- CASCADE on connection deletion — automatic via ON DELETE CASCADE.
- Never deleted individually — unsubscribe is permanent.
  No "remove from unsubscribed list" action in V1.
- Not subject to retention policy — survives campaign history
  cleanup. Must persist forever to prevent re-emailing
  unsubscribed recipients after campaign rows are deleted.

**Indexes:**
```sql
-- UNIQUE constraint on (account_id, email) creates index automatically
-- justification: SELECT WHERE account_id = ? AND email = ?
-- on every recipient check at campaign confirm time.
-- UNIQUE also enforces one unsubscribe record per email per account
-- at DB level — no duplicate unsubscribes possible.

CREATE INDEX idx_unsubscribes_account_id
  ON unsubscribes(account_id);
-- justification: SELECT WHERE account_id = ? for Settings page
-- UNIQUE composite covers (account_id, email) but a standalone
-- account_id index may be faster for the Settings list query
-- which doesn't filter by email. Worth checking with EXPLAIN ANALYZE.
```
 
---

## Retention Policy

Campaign history is deleted on a rolling basis by a scheduled cleanup
job that runs nightly:

| Plan | Retention |
|---|---|
| Free | 7 days |
| Pro | 90 days |

**What gets deleted:**
- `campaign_recipients` rows where `campaigns.sent_at` is older than
  the retention window — cascade deletes outbox rows automatically
- `webhook_events` rows where `created_at` is older than retention window
  — independent cleanup, not tied to campaign_recipients deletion
- `campaign_filters` rows cascade-deleted with their `campaign`
- `campaigns` rows themselves after all recipients are deleted
  **What is never deleted:**
- `unsubscribes` — permanent, survives account retention policy
- `accounts`, `connections`, `email_providers` — survive until account closure

**What gets deleted:**
- `campaign_recipients` rows where `campaigns.sent_at` is older than
  the retention window
- `outbox` rows cascade-deleted with their `campaign_recipient`
- `webhook_events` rows where `created_at` is older than retention window
- `campaign_filters` rows cascade-deleted with their `campaign`
- `campaigns` rows themselves after all recipients are deleted
  **What is never deleted:**
- `unsubscribes` — permanent, survives account retention policy
- `accounts`, `connections`, `email_providers` — survive until account closure
---

## Scaling Strategy
<!-- TODO: fill in Scaling Strategy based on storage estimation -->


## Indexes Reference

| Index | Table | Columns | Type | Justification |
|---|---|---|---|---|
| idx_accounts_email | accounts | email | Standard | Login query |
| idx_accounts_api_key_hash | accounts | api_key_hash | Standard | API auth on every request |
| idx_connections_account_id | connections | account_id | Standard | Load connections per account |
| idx_campaigns_account_id | campaigns | account_id | Standard | Dashboard campaign list |
| idx_campaigns_status | campaigns | status | Standard | Filter sending campaigns |
| idx_campaign_recipients_campaign_id | campaign_recipients | campaign_id | Standard | Load recipients per campaign |
| idx_campaign_recipients_resend_message_id | campaign_recipients | resend_message_id | Standard | Webhook event matching |
| idx_campaign_recipients_status | campaign_recipients | status | Standard | Count by status for stats |
| idx_outbox_status_next_attempt | outbox | status, next_attempt_at | Partial (WHERE status='pending') | Worker poll — only pending rows |
| idx_webhook_events_processed | webhook_events | processed | Partial (WHERE processed=false) | Unprocessed event backlog |
| idx_unsubscribes_account_email | unsubscribes | account_id, email | Composite | Check unsubscribe before send |