# Warren

**Dead-letter operations for RabbitMQ.** Warren shows you what is stuck in your dead-letter
queues, lets you read the actual messages, replays them with a full audit trail, keeps a
history of every queue, and alerts you before a queue becomes a problem.

The RabbitMQ management UI tells you *that* a queue has 300 messages. Warren tells you
*what* they are, *why* they died, how the number developed, and moves them back with one
click, logged, without a hand-written script at 3 a.m.

![Warren: from 200 dead letters to a throttled replay, keyboard only](docs/demo-main.gif)

Watch it with controls on YouTube: [Replay RabbitMQ dead letters without a script](https://youtu.be/0ojsYE-jMFA) (31 s).

## About this repository

This is Warren's public home: the README, the compose files, the [changelog](CHANGELOG.md),
[issues](../../issues) and [discussions](../../discussions). Warren ships as a ready-to-run
container image, `ghcr.io/viba88/warren` and on Docker Hub as
[`viba88/warren`](https://hub.docker.com/r/viba88/warren); its source code is not published. Bug reports, feature
requests and questions are welcome here. Landing page: https://warrenops.io

## What it does

- **Overview for whoever is on call**: every cluster on one page with its dead letters, free disk
  and memory of its broker, unreachable clusters marked, the firing alerts, the queues that gained
  most messages in the last hour, six or 24 hours, the largest queues, and everything Warren did in
  the last 24 hours. Narrow it to some clusters with the chips on top; a cluster left out that fires
  or is down is still named. It is the start page; `g o` gets back to it.
- **Multiple clusters and vhosts.** Configure any number of cluster entries, one per broker and
  vhost; entries for several vhosts of one broker count as one broker for the edition limits.
  Switch between them in the sidebar; the queues, audit log, alerts and replay rules follow it.
- **Queue overview** with dead-letter queues detected automatically (by bindings to a
  dead-letter exchange, or by name pattern) and sorted to the top, plus a bell on queues
  with a firing alert. Empty retry wait queues stay out of the way until you ask for them or
  search by name. Parking queues are tagged `PARKED`, quorum queues whose peeks count as
  deliveries `LIMIT`.
- **Message browser**: peek at messages without consuming them, see payload (pretty JSON),
  headers, and the complete `x-death` history: which queue rejected it, how often, when. A message a
  consumer republished itself carries no death time; "dead since" then shows how long Warren has seen
  it waiting.
  Large payloads are listed as a preview and loaded in full on demand; long embedded strings
  such as base64 attachments are collapsed to a placeholder so the fields you care about stay
  visible.
- **Replay** selected messages or the first N to their **original exchange and routing key**
  (from `x-death`), to a named queue, or to any exchange. Death headers are stripped so the
  message gets a fresh retry budget. Publisher confirms guarantee nothing is lost.
- **Throttled replay**: limit a replay to N messages per second so the consumers that just
  failed are not flooded again. A throttled replay runs in the background; its audit page
  shows the progress and the time left, and has a **Stop** button: it stops after the message it
  is on, what went stays replayed, the rest stays in the queue (status `STOPPED`).
- **Edit before replay**: select one message, fix the payload (with JSON validation), the
  content type or the headers, replay. "Remove retry bookkeeping" drops headers such as
  `x-retry-count` or `x-last-exception-*` that many consumers use to give up early, so the
  message gets a real second chance. The audit log marks the message as edited and the
  republished message carries an `x-warren-edited` header.
- **Download and publish**: download a message as JSON with its payload, properties and headers.
  Publish writes one new message to a queue or an exchange, typed in, loaded from a file (a
  downloaded message fills in everything) or as a copy of a message in the queue, for example to
  test a consumer. Nothing is taken out of any queue; the message carries `x-warren-published-by`.
  Bodies up to 1 MiB (`WARREN_MAX_PUBLISH_BYTES`).
- **Export and import**: copy the selected messages or the first N (up to 10,000) into an NDJSON
  file, one message per line in the same format as a single download; the queue keeps them and the
  audit log records who exported how many. Loading such a file in the publish dialog publishes all its
  messages to one target, as one audited action (up to 1,000 messages, 50 MiB).
- **Purge** a whole queue in one step (operators only): the discard dialog's "All messages"
  option asks for a reason. Unlike a discard it works on
  queues of any size; the audit log records who, why and how many, not the messages themselves.
  Streams cannot be purged.
- **Park** poison messages (operators only): the selected messages, the first N or every match of
  the filter move to `<queue>.parking`, out of the way of replays and rules but not lost. Unlike a
  replay they keep every header, `x-death` included, so a later replay from the parking queue goes
  to the original route. Each carries `x-warren-parked-from`, `-parked-at`, `-parked-by` and an
  optional note as `x-warren-park-reason`. Warren creates the parking queue as a plain durable
  queue if it does not exist (this needs configure permission; otherwise create it once yourself)
  and uses an existing one as it is.
- **Audit log** of every replay, park, discard, purge, export and publish: who, when, from which cluster
  and queue to where, per-message outcome. Filter it by action, queue, user or rule and time range;
  the filters stay in the URL, so a filtered view can be shared. Each message can be opened as it sat in the queue: why it died,
  properties, headers, death history and the first 16 KiB of its body (for an edited message
  also what was published instead). Configure the kept head with `WARREN_AUDIT_PAYLOAD_BYTES`.
- **Payloads you can read**: JSON as a tree that folds (large payloads open on two levels), as a
  table of field paths, or raw. Files sent as base64 are recognised by their content and shown as
  files: PDF and images open in a tab or preview inline, CSV previews as a table, JSON formatted,
  with a warning when it does not parse; everything downloads under the name from a field such as
  `fileName`. Java class names show their short name, dates how long ago, epoch fields a date. Each
  value can be copied, its path too, or become the search ("show the messages with this value").
- **Compare messages**: select two or more, and Warren lists only the payload fields that differ,
  each value with how many messages carry it; fields unique per message (ids) in one line.
- **When is it empty?** The queue list has an "Empty in" column and the queue page says
  "empty in ≈ 14 min" at the current pace, "growing 3/s", or "not draining". It is judged by the
  depth of the last ten minutes, or by the broker's rates while there is no history yet
  (acknowledged minus published). A running replay drains a dead-letter queue the same way, so it
  shows when the replay will be done. `GET /api/clusters/{c}/queues/{q}/forecast`.
- **Choose the columns of the queue list**: hide what you do not need. Ready is hidden by default,
  since it equals Messages whenever nothing is being processed.
- **Sensitive values hidden** in the message view: JSON fields and headers whose name contains
  `password`, `token`, `apikey`, `iban`, `email` and the like, plus anything that looks like an e-mail
  address, an IBAN or a card number (Luhn-checked) inside payloads, headers and exception texts.
  Operators and admins can show one message's values; viewers cannot. Display only: copies,
  downloads, exports, edits and replays keep the real values, and the API returns them to anyone
  with access. `WARREN_MASKING_ENABLED`, `WARREN_MASKING_FIELDS`, `WARREN_MASKING_VALUES`,
  `WARREN_MASKING_ALLOW_REVEAL`.
- **Notes on a group of dead letters** (operators write, everyone reads): "known bug in the invoice
  consumer, fix in 4.2, do not replay". A note belongs to one bar of "why they died" in one queue,
  for example the exception `OrderNotFoundException` or the reason `expired`. It shows under that
  bar, in each message of the group, and at the top of the replay dialog when the replay would take
  such messages.
- **Keyboard**: everything is reachable without a mouse. `?` lists the shortcuts of the page;
  `/` searches it, `g o`/`g q`/`g r`/`g a`/`g u` go to the overview, queues, audit log, alerts and users, `c` switches
  the cluster. In a search field ↓/↑ move through the hits and Enter opens the marked one. In
  tables `j`/`k` move, Enter opens or expands, `x` selects; on a queue `r`, `d`
  and `p` replay, discard and publish. Dialogs submit with ⌘/Ctrl+Enter and pick options with
  Alt and the underlined letter. `f` puts a letter on every visible button and field, and ⌘K
  runs the page's actions too.
- **Broker resources**: the overview shows each cluster's free disk and memory use in its card
  (per node in the tooltip) and the queues holding the most bytes. RabbitMQ blocks every publisher
  of the cluster once a node has less free disk than `disk_free_limit` or uses more memory than
  its high watermark; a dead-letter queue that grows unnoticed is a common way to get there.
  Reading the nodes needs the `monitoring` tag on Warren's RabbitMQ user; without it the card says so.
- **Metrics history**: every queue is sampled periodically (default 30 s, 7 days retention).
  The queue page shows messages, ready, unacked and consumers over the last hour (Community),
  up to 7 days (Team) or 90 days (Pro).
- **Alerting**: rules match queues by regex on one or all clusters. A new rule starts from a
  template (dead letters piling up, a growing dead-letter queue, consumers gone, mass failure) with the
  queue pattern read off the cluster's naming convention. Conditions: messages
  above a threshold, no consumers while messages wait, growth within a time window, inflow
  above a rate per minute (on a dead-letter queue: the dead-letter rate, which shows a mass
  failure even while a replay rule keeps draining the queue). Per broker node: free disk above
  the limit below N MB, memory above N % of the limit, publishers blocked by a resource alarm
  (templates "Disk running low", "Memory high", "Publishers blocked"). Each can
  require the condition to hold for a number of seconds. Notifications go to Slack, Microsoft
  Teams or any JSON webhook; firing alerts are re-notified after a configurable interval and
  a resolved notification follows when the queue recovers.
- **Users and roles**: `VIEWER` reads, `OPERATOR` replays, `ADMIN` manages alerts and users.
  Local users are managed in the UI (create, roles, enable/disable, password reset; everyone
  can change their own password). The users from configuration are only seeded on first start.
  Optionally sign in through any **OIDC** provider (Keycloak, Entra ID, Okta, …) with roles
  mapped from a claim.

## Try it in a minute

Docker is all you need. This starts a RabbitMQ, Warren and a few dead-letter queues with real dead
letters (rejected, expired, and republished by a consumer with its exception):

```bash
curl -O https://warrenops.io/docker-compose.try.yml
docker compose -f docker-compose.try.yml up -d
```

Open http://localhost:8080 and sign in as `admin` / `admin`. Open `orders.dlq`, look at why the
messages died, select a few and replay them. `docker compose -f docker-compose.try.yml down -v`
removes everything again. The demo broker's management UI is on port 15680, off the default so it
never collides with a RabbitMQ you already run. Ports taken? `TRY_WARREN_PORT=8081 TRY_RABBIT_UI_PORT=15690` in front.

To run Warren next to your own broker, see [Running against your own RabbitMQ](#running-against-your-own-rabbitmq).

## Running against your own RabbitMQ

Warren runs next to your broker, never in someone else's cloud. Your message payloads stay
in your network. See `docker-compose.example.yml`.

### Database: embedded by default, Postgres when you want it

Warren stores only its own bookkeeping: replay audit, users, alert rules, metric samples and
the licence. Without `WARREN_DB_URL` it uses an embedded H2 database in `WARREN_DATA_DIR`
(`/app/data` in the image, declared as a volume so Docker persists it even when you forget to
mount one). Every transaction is written to disk immediately; data survives restarts and
crashes. On startup Warren logs where the data lives and warns loudly if the directory is not a
mounted volume inside a container. Back up by copying the volume while Warren is stopped.

With a Pro licence, set `WARREN_DB_URL=jdbc:postgresql://host:5432/warren` plus credentials to
use your own Postgres instead, see `docker-compose.postgres.example.yml`. Postgres is the right
choice for several Warren instances behind a load balancer or when you already back up a
database server. Migrations run automatically on both. Without a licence Warren refuses to start
against an external database and says so in the log.

Several instances on one database coordinate through it: sampling and alerting, replay rules and
the hourly housekeeping each run on one instance at a time, which takes a short lease in the
`job_lock` table at every tick and renews it. An instance that stops renewing, because it died or
was scaled down, is replaced within a few intervals; a clean shutdown hands over at once. Every
instance serves the UI and runs manual replays. The log says which instance runs which job.

The metrics history is what makes the database grow: one row per queue per sample, so at the
default 30 s interval about 20,000 rows per queue and week. A broker with 100 queues stays well
under a gigabyte; with a thousand queues plan for a few gigabytes and use Postgres, or lengthen
`WARREN_METRICS_SAMPLE_INTERVAL`. The audit log is kept forever unless `WARREN_AUDIT_RETENTION`
is set (for example `90d`); then finished actions with their per-message records and resolved
alert events older than that are deleted once an hour. Firing alerts and running replays stay.
With Pro the samples are also rolled up into one row per queue and hour, kept for
`WARREN_METRICS_LONG_RETENTION` (90 days), for the 30 and 90 day history: about 2,200 rows per queue.

### One cluster, environment variables

| Variable | Default | Purpose |
|---|---|---|
| `WARREN_RABBIT_MANAGEMENT_URL` | `http://localhost:15672` | Management HTTP API |
| `WARREN_RABBIT_HOST` / `WARREN_RABBIT_PORT` | `localhost` / `5672` | AMQP endpoint used for replay |
| `WARREN_RABBIT_USERNAME` / `WARREN_RABBIT_PASSWORD` | `guest` / `guest` | Needs `management` tag plus read/write on the vhost; `monitoring` for disk and memory of the nodes |
| `WARREN_RABBIT_VHOST` | `/` | The vhost this cluster entry covers |
| `WARREN_RABBIT_ID` / `WARREN_RABBIT_NAME` | `default` / – | Id used in URLs, display name |
| `WARREN_DB_URL` / `_USERNAME` / `_PASSWORD` | embedded H2 in `WARREN_DATA_DIR` | Audit log, users, metrics, alerts; set a `jdbc:postgresql://` URL for Postgres |
| `WARREN_DATA_DIR` | `/app/data` (image) / `./data` | Directory of the embedded database |
| `WARREN_ADMIN_USERNAME` / `WARREN_ADMIN_PASSWORD` | `admin` / `admin` | The single local admin. Change it. |
| `WARREN_PORT` | `8080` | HTTP port |
| `WARREN_PUBLIC_URL` | – | Base URL used for links in notifications |
| `WARREN_METRICS_SAMPLE_INTERVAL` / `WARREN_METRICS_RETENTION` | `30s` / `7d` | Sampling and history retention |
| `WARREN_AUDIT_RETENTION` | `0` (forever) | Age after which finished actions and resolved alert events are deleted, e.g. `90d` |
| `WARREN_AUDIT_MIN_RETENTION` | `0` | Pro: nothing younger is ever deleted; a shorter `WARREN_AUDIT_RETENTION` stops Warren at startup |
| `WARREN_AUDIT_WEBHOOK_URL` | – | Pro: every audit event as a JSON POST, see [Audit export and forwarding](#audit-export-and-forwarding-pro) |
| `WARREN_AUDIT_SYSLOG_HOST` / `_PORT` / `_PROTOCOL` | – / `514` / `UDP` | Pro: every audit event as an RFC 5424 syslog message, `UDP`, `TCP` or `TLS` |
| `WARREN_APPROVAL_ACTIONS` | – | Pro: `REPLAY,DISCARD,PURGE` or a part of it wait for a second operator, see [Four-eyes approval](#four-eyes-approval-pro) |
| `WARREN_APPROVAL_EXPIRES_AFTER` | `24h` | Pro: a request nobody decided on expires |
| `WARREN_METRICS_LONG_RETENTION` | `90d` | Pro: hourly roll-ups behind the 30 and 90 day history |
| `WARREN_PEEK_CACHE_MAX_BYTES` | 256 MiB | Memory for peeked message bodies kept for the payload view; oldest go first |
| `WARREN_ALERTING_ENABLED` / `WARREN_ALERTING_RENOTIFY_AFTER` | `true` / `4h` | Alert evaluation and repeat notifications |
| `WARREN_DLQ_NAME_PATTERN` | see `application.yml` | Regex for name-based DLQ detection |
| `WARREN_MASKING_ENABLED` | `true` | Hide sensitive values in the message view |
| `WARREN_MASKING_FIELDS` | `password,passwd,secret,token,apikey,…` | Field and header names to hide, matched as part of the name, ignoring case, `-`, `_` and `.` |
| `WARREN_MASKING_VALUES` | `EMAIL,IBAN,CARD` | Value patterns hidden wherever they appear |
| `WARREN_MASKING_ALLOW_REVEAL` | `true` | Operators and admins may show the hidden values of one message |

### Several clusters, several users

Lists are easiest in a YAML file mounted as `/app/config/application.yml` (Spring Boot picks
up `./config/application.yml` next to the jar automatically), or as indexed env vars such as
`WARREN_CLUSTERS_0_ID`, `WARREN_CLUSTERS_0_MANAGEMENT_URL`, `WARREN_USERS_0_USERNAME`:

```yaml
warren:
  clusters:
    - id: prod
      name: Production
      management-url: https://rabbit.prod.internal:15672
      host: rabbit.prod.internal
      username: warren
      password: ${PROD_RABBIT_PASSWORD}
      vhost: /
    - id: prod-billing
      name: Production billing
      management-url: https://rabbit.prod.internal:15672
      host: rabbit.prod.internal
      username: warren
      password: ${PROD_RABBIT_PASSWORD}
      vhost: billing
  users:
    - { username: admin, password: ${WARREN_ADMIN_PASSWORD}, roles: [ADMIN] }
    - { username: oncall, password: ${ONCALL_PASSWORD}, roles: [OPERATOR] }
    - { username: support, password: ${SUPPORT_PASSWORD}, roles: [VIEWER] }
```

When `warren.clusters` is set, the `WARREN_RABBIT_*` variables are ignored; when
`warren.users` is set, `WARREN_ADMIN_*` is ignored. Users from configuration are imported
**once**, when the user table is still empty. From then on manage them under **Users** in the
UI; later changes to the configured users have no effect. If you lock yourself out, delete
the rows in `app_user` and restart: the seed runs again.

### Single sign-on (OIDC)

```yaml
warren:
  oidc:
    enabled: true
    issuer-uri: https://login.example.com/realms/platform
    client-id: warren
    client-secret: ${OIDC_CLIENT_SECRET}
    roles-claim: realm_access.roles        # dot path into the ID token / userinfo claims
    role-mapping:                          # external role -> Warren role
      mq-admins: ADMIN
      mq-oncall: OPERATOR
    default-role: VIEWER                   # or null to deny users without a mapped role
    username-claim: preferred_username
```

Redirect URI to register at the provider: `https://<warren>/login/oauth2/code/oidc`.
Local users keep working next to SSO; the login page shows both.

## Editions and licence

Warren ships as one artifact with three editions. Without a licence key it runs as **Community**;
a valid key switches it to **Team** or **Pro** at runtime, no restart.

| | Community | Team | Pro |
|---|---|---|---|
| Browse, search, replay, park, discard, purge, publish, audit log, local users, keyboard | yes | yes | yes |
| Brokers | the first configured one, up to 3 of its vhosts | up to 3, any number of vhosts | unlimited, one flat price per installation |
| Roles viewer / operator / admin | every local user is admin (and may show masked values) | yes | yes |
| Metrics history | the last hour | 15 minutes to 7 days | up to 90 days |
| Alerting and replay rules | off (endpoints answer 402) | yes | yes |
| OIDC single sign-on | off | off | yes |
| Four-eyes approval for replay, discard, purge | off | off | yes |
| Audit export (CSV), forwarding to a SIEM, minimum retention | off | off | yes |
| Database | embedded (one container, one volume) | embedded | embedded or your own PostgreSQL |

A key may carry its own broker cap for special deals; the edition sits in the signed key.

Limits count brokers, not cluster entries: entries that differ only in the vhost (same management URL)
are one broker. Entries beyond the edition's limit stay configured but inactive; the UI shows how many.
In Community, raw samples are kept for two hours, enough for the hour it shows; with a licence for
`WARREN_METRICS_RETENTION`. Stored user
roles are kept and take effect at the next sign-in once a licence is installed.

Install a key under **Edition & licence** (topbar badge) as admin, or supply it through
configuration: `WARREN_LICENSE_KEY=WARREN-…` or `WARREN_LICENSE_FILE=/app/config/license.key`.
A key installed in the UI takes precedence. Keys are `WARREN-<claims>.<signature>`, signed with
Ed25519 and verified offline against the public key embedded in Warren. An expired key drops
Warren back to Community with a warning in the UI.

`GET /api/edition` returns the current state; a call the edition lacks answers
`402 Payment Required` with the feature and the edition that has it.

### Four-eyes approval (Pro)

With `WARREN_APPROVAL_ACTIONS=REPLAY,DISCARD,PURGE` (or a part of it) these actions do not run when an
operator asks for them. The request is checked as if it ran (queue, target, delivery limit, throttle) and
then waits under **Approvals** with a one-line summary; the API answers `202` with `{"approval": …}`
instead of `201` and the action. Another user with the operator role approves it, and it runs exactly as
asked, in the requester's name with the approver recorded next to it (`approvedBy` in the audit log). The
requester may withdraw it, others may reject it with a reason; after `WARREN_APPROVAL_EXPIRES_AFTER` it
expires. A purge removes what is in the queue at approval time. Replay rules are not affected: an admin
sets them up to run without anyone at hand.

### Audit export and forwarding (Pro)

`GET /api/replays/export` returns the audit log as CSV with the filters of the list, one row per action or
with `messages=true` one per message (no payloads), at most 100,000 actions. Text a spreadsheet would run
as a formula is prefixed with `'`.

With `WARREN_AUDIT_WEBHOOK_URL` and/or `WARREN_AUDIT_SYSLOG_HOST` every finished action and every approval
decision is forwarded as it happens: `action.finished`, `approval.requested`, `.approved`, `.rejected`,
`.cancelled`, `.expired`, `.failed`. Each event is JSON with `id`, `type`, `at`, `source` and the `action` or
`approval`. Events are written to an outbox in the transaction that records them and sent in order; a
receiver that is down holds them until it is back. Delivery is at least once, so deduplicate by `id`.
The webhook carries `X-Warren-Event`, `X-Warren-Delivery`, `X-Warren-Timestamp`, an optional static header
(`WARREN_AUDIT_WEBHOOK_HEADER_NAME` / `_VALUE`) and with `WARREN_AUDIT_WEBHOOK_SIGNING_SECRET`
`X-Warren-Signature: sha256=<hmac of "timestamp.body">`. Syslog messages are RFC 5424 with the event type as
MSGID, facility `WARREN_AUDIT_SYSLOG_FACILITY` (16, local0), severity notice, warning for failed actions.

`WARREN_AUDIT_MIN_RETENTION` (e.g. `365d`) is the floor for the retention: nothing younger is deleted, and a
`WARREN_AUDIT_RETENTION` below it stops Warren at startup with a clear message.

## Roles

| Role | May |
|---|---|
| `VIEWER` | see clusters, queues, messages, history, replays and alert events |
| `OPERATOR` | everything above, plus replay (including edited payloads) |
| `ADMIN` | everything above, plus alert rules and channels, plus user management |

## How replay works, and what it promises

1. Warren opens a dedicated AMQP channel with publisher confirms.
2. It fetches messages with `basic.get` **without acknowledging**. Unacked messages stay
   outstanding on the channel, so each fetch returns the next message.
3. Selected messages are published to the target with `mandatory=true`. Only after the broker
   confirms is the source message acknowledged. At full speed Warren publishes up to 100
   messages and then waits for their confirms together, tracking each by its sequence number;
   a throttled replay settles every message on its own. Unroutable publishes are detected via
   `basic.return` and reported as failed.
4. Everything not selected is released with `nack + requeue`. Closing the channel releases
   whatever is left, even if Warren crashes mid-way.

**Guarantee:** no message is ever lost. **Trade-off:** at-least-once. If Warren dies between
the broker's confirm and its own ack, the messages of that window (one when throttled) are
duplicated. Your consumers should be idempotent anyway; this is the same promise RabbitMQ
itself makes.

**Throttling:** with `ratePerSecond` the publishes are spaced evenly and the replay runs in the
background: the request returns the `RUNNING` action right away, and `GET /api/replays/{id}`
shows the counts growing. The fetched messages stay unacknowledged on Warren's channel for the
whole run, and RabbitMQ closes channels whose deliveries stay unacked beyond its
`consumer_timeout` (30 minutes by default). A throttled replay may therefore take at most
`warren.replay.max-throttled-duration` (default 25 minutes, selection size divided by rate);
longer ones are refused up front. If Warren shuts down mid-way, the replay stops after the
current message, is recorded as `PARTIAL`, and everything not yet replayed stays in the queue.

Peeking through the management API marks messages as `redelivered`. That is a property of
RabbitMQ, not something Warren can avoid.

**Quorum queues with a delivery limit:** on RabbitMQ 3.13 every peek, and every message a
replay, discard or export reads and puts back, counts as a delivery. A message that reaches the
limit (`x-delivery-limit` or the policy's `delivery-limit`) is dropped or dead-lettered. On such
queues Warren reads nothing until the user confirms: the API answers `409` with
`code: DELIVERY_COUNTED` until the request carries `acknowledgeDeliveryCount`, and replay rules
skip them. RabbitMQ 4 counts only abnormal returns, so peeks are safe there.

## API

All endpoints need a session (`POST /api/auth/login` with `{"username","password"}`) or an
OIDC login. Roles as in the table above.

```
GET  /api/auth/providers                                   which login methods exist
GET  /api/overview?growth=1h                               all clusters, firing alerts, growing queues, last 24 h of actions
GET  /api/clusters                                         configured clusters with reachability
GET  /api/clusters/{c}/queues                              queues with dead-letter classification
GET  /api/clusters/{c}/queues/{q}/messages?limit=          peek at messages (max 200)
GET  /api/clusters/{c}/queues/{q}/history?range=1h         sampled metrics, range 15m…7d, up to 90d with Pro
GET  /api/clusters/{c}/queues/{q}/forecast                 when the queue is empty at the current pace
POST /api/clusters/{c}/queues/{q}/replay                   run a replay (OPERATOR); 202 when it waits for approval
POST /api/clusters/{c}/queues/{q}/park                     move messages to {q}.parking (OPERATOR)
GET  /api/replays?clusterId=&kind=&queue=&requestedBy=&since=&before=   audit log, newest first
GET  /api/replays/export?…&messages=                       audit log as CSV (Pro)
GET  /api/approvals?status=, GET /api/approvals/{id}       requests waiting for a second person (Pro)
POST /api/approvals/{id}/approve|reject|cancel             decide: approve runs it, reject needs a note, cancel is the requester's (OPERATOR)
GET  /api/clusters/{c}/queues/{q}/notes                    notes on groups of this queue's dead letters
PUT  /api/clusters/{c}/queues/{q}/notes                    write a group's note: dimension, groupKey, groupLabel, text (OPERATOR)
DELETE /api/clusters/{c}/queues/{q}/notes/{id}             remove a note (OPERATOR)
GET  /api/settings                                         masking, approval and audit settings for the UI
GET  /api/replays/{id}                                     one replay with per-message outcomes
GET  /api/replays/{id}/messages/{messageId}                one message as it sat in the queue
GET  /api/alerts/events?firing=true                        alert events
GET/POST/PUT/DELETE /api/alerts/rules[/{id}]               rules (write: ADMIN)
GET/POST/PUT/DELETE /api/alerts/channels[/{id}]            channels (write: ADMIN)
POST /api/alerts/channels/{id}/test                        send a test notification (ADMIN)
GET/POST /api/users, PUT/DELETE /api/users/{id}            local users (ADMIN)
POST /api/users/{id}/password                              reset a user's password (ADMIN)
POST /api/auth/password                                    change own password
```

Replay body:

```json
{
  "selection": { "type": "FINGERPRINTS", "fingerprints": ["…"] },
  "target":    { "type": "ORIGINAL" },
  "edits":     [{ "fingerprint": "…", "payload": "{\"fixed\":true}", "contentType": "application/json",
                  "headers": { "x-tenant": "acme", "x-retry-count": 0 } }],
  "ratePerSecond": 20
}
```

`selection.type` is `COUNT` (with `count`), `FINGERPRINTS` (with fingerprints from the
messages endpoint) or `MATCHING` (with `match`: `reasons` such as `rejected` or `republished`
and/or `text` found in the payload or a header value; every match of the scan is taken, or at
most `count` of them). A `MATCHING` selection applies the queue view's filter to the whole
queue, not only to the messages it has loaded; it scans up to `warren.inspection.max-scan`
messages. `target.type` is `ORIGINAL`, `QUEUE` (with `queue`) or `EXCHANGE` (with
`exchange` and `routingKey`). `edits` is optional and only allowed with `FINGERPRINTS`; each
of `payload`, `contentType` and `headers` is optional, `headers` replaces the application
headers completely (death bookkeeping is stripped either way). `ratePerSecond` (1 to 1000) is
optional; with it the replay is throttled and runs in the background (see above).

## Roadmap

Message editing for several messages at once, e-mail as a notification channel, per-cluster
permissions.

## License

Proprietary, all rights reserved. Warren is distributed as a container image; the Community
edition may be used free of charge for any purpose, the Pro edition requires a licence key
(see "Editions and licence" and [LICENSE.md](LICENSE.md)). The source is not open at this time.
