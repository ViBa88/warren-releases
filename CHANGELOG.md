# Changelog

All notable changes to Warren. Images are published as `ghcr.io/viba88/warren:<version>`,
`:<major>.<minor>` and `:latest`.

## 0.9.0 (2026-10-05)

- **When is it empty?** The queue list has an "Empty in" column and each queue page says "empty in ≈ 14 min" at the current pace, "growing 3/s" or "not draining". With metrics history (Team, Pro) it is judged by the depth of the last ten minutes, otherwise by the broker's rates. A running replay drains a dead-letter queue like a consumer, so it shows when the replay is done. `GET /api/clusters/{c}/queues/{q}/forecast`.
- **Choose the columns of the queue list.** Hide what you do not need; Ready is hidden by default, since it equals Messages whenever consumers hold nothing.
- **Docker Hub:** the image is also published as `viba88/warren`, with the same tags as `ghcr.io/viba88/warren`.
- A group filter whose group is gone after a reload (all parked, replayed or discarded) is cleared instead of leaving an empty table; the group filter chip no longer pushes the queue toolbar onto two lines.

## 0.8.0 (2026-10-02)

- **Payloads you can read.** JSON shows as a tree that folds, as a table of field paths, or raw. Files sent as base64 are recognised by their content and shown as files: PDF and images open or preview, CSV previews as a table, JSON is formatted and flagged when it does not parse, XML and text open as plain text; everything downloads under the name from a field such as `fileName`. Java class names shorten, dates say how long ago, every value can be copied, its path too, or become the search.
- **Compare messages.** Select two or more and see only the fields that differ, each value with how many messages carry it: "6 of 8 from customer C000105953". Fields unique per message are summarised in one line, files compared by type.
- **Sensitive values hidden** in the message view: fields and headers named like a password, token, IBAN, e-mail and more, plus anything that looks like an e-mail address, an IBAN or a card number. Operators and admins can show one message's values. Display only; replays, downloads and exports keep the real data. `WARREN_MASKING_ENABLED`, `_FIELDS`, `_VALUES`, `_ALLOW_REVEAL`.
- **Notes on groups of dead letters.** "Known bug, fix in 4.2, do not replay" on a bar of "why they died": shown on the bar, in every message of the group, and in the replay dialog. Operators write, everyone reads. `GET/PUT /api/clusters/{c}/queues/{q}/notes`.
- Message details without repeats: one time for a single death, the death history from the second death on, the original route in "why it is here", headers that only repeat an id behind "show all", rare properties under "Details", one panel instead of three.
- Smaller: the queue page says "6 of 8 shown" while a search narrows the table; tables no longer underline names on hover; text attachments never render in the browser, they open as plain text.

## 0.7.0 (2026-10-02)

- **Park**: move poison messages to `<queue>.parking` in one step, next to replay and discard. They keep their death history, so a replay from the parking queue later goes to the original route; an optional note goes into the audit log and onto each message. Warren creates the parking queue if it is missing. `POST /api/clusters/{c}/queues/{q}/park`, key `m`.
- **Quorum queues with a delivery limit** are safe to open on RabbitMQ 3.13, where every peek counts as a delivery: Warren marks them `LIMIT` in the queue list and reads, replays, discards or exports nothing until you confirm. Replay rules skip them. RabbitMQ 4 is not affected.
- **Audit log** (formerly "Replays") filters by action, queue, user or rule and time range, keeps the filters in the URL and loads older entries page by page; `GET /api/replays` takes `kind`, `queue`, `requestedBy`, `since` and `before`.
- Queue page: the messages come first; flow and history open on demand. Replay is the main action, Park and Discard sit together, Automate, Publish and Export moved into a "More" menu. All shortcuts stay.
- Queue list: starts in triage order, dead-letter queues with messages on top; empty retry wait queues are hidden until asked for or searched.
- Alert rules start from templates (dead letters piling up, growing dead-letter queue, consumers gone, mass failure), with the queue pattern read off the cluster's naming.
- Messages published without any properties can be peeked again.
- Smaller: the sidebar shows the full cluster name with vhost and version below, the account actions sit in a labelled menu, the fingerprint moved to the end of a message's properties.

## 0.6.0 (2026-09-30)

- Overview page, now the start page: every cluster on one screen with its dead letters, dead-letter queues and queues without consumers, unreachable clusters marked, the firing alerts, the queues that gained most messages in the last hour, six or 24 hours, and everything Warren did in the last 24 hours. Cards and rows open the cluster, the queue or the audit entry; `g o` gets back to it; `GET /api/overview`. Growth and alerts need Team or Pro, the rest works in Community.
- Alert condition `INFLOW_RATE`: messages arrive faster than a threshold per minute, sustained for a time. On a dead-letter queue that is the dead-letter rate, so it sees a mass failure even while a replay rule keeps draining the queue. Needs the broker's rate statistics (`rates_mode`).

## 0.5.0 (2026-09-30)

- Replay, discard or export everything that matches: a death reason and/or a text, judged per message across the whole queue (up to 10,000 scanned), not only the messages on screen. The replay and discard dialogs offer it as soon as the queue view has a group or a search active; the API takes `selection.type: MATCHING`.
- The message search also finds text in header values, where consumers leave their exception messages; "Why they died" groups show their oldest and newest death.
- Several Warren instances on one PostgreSQL share the work: sampling and alerting, replay rules and the hourly housekeeping run on one instance at a time and move over when it stops (Pro).
- `WARREN_AUDIT_RETENTION` deletes finished actions with their message records and resolved alert events beyond an age; the default keeps them forever.
- Much less broker load with many queues: the management API is asked only for the columns Warren reads, the flow graph fetches each list once, clusters are sampled in parallel.
- Faster database writes: samples, audit messages and sightings go in as batches; expired samples are deleted in chunks; indexes for the alert list.
- A full-speed replay confirms a hundred publishes at a time, each tracked by its sequence number, so large replays no longer pay one round trip per message. Every message still gets its own verdict.
- `WARREN_PEEK_CACHE_MAX_BYTES` bounds the memory peeked message bodies may take (default 256 MiB).
- A throttled replay is sized by what the queue holds, so "first 10,000" or "everything matching" on a small queue is no longer refused; a cleared peek field falls back to 50.

## 0.4.0 (2026-09-25)

- Stop a running throttled replay: what went stays replayed, the rest stays in the queue (status `STOPPED`).
- Export selected messages or the first N (up to 10,000) as NDJSON; import such a file in the publish dialog as one audited action.
- Replay rules: dead letters go back to their origin on a schedule, with limits (Team and Pro).
- "Dead since" for messages a consumer republished itself, without an `x-death` header.
- A paced replay shows its progress as a bar with the time left; the replay dialog shows one quiet plan.
- Queues sort by message count; the publish dialog takes dropped files.
- A broker that never answers no longer holds up the whole UI.
- Keyboard: `j`/`k` keep working after switching the cluster; Enter opens the row under the cursor.
- A message's first sighting reads back exactly as it was returned.
- `docker-compose.try.yml`: a broker, Warren and real dead letters in one file.
- Team and Pro keys are bought once: a key runs every version released within its update period.

## 0.3.0 (2026-09-24)

- Team edition between Community and Pro.
- Keyboard: everything reachable without a mouse; `?` lists the shortcuts, `f` labels every visible control, ⌘K quick search over queues, replays, clusters and pages.
- Purge a whole queue in one broker step, with a reason in the audit log.
- Discard messages with a reason, recorded in the audit log.
- Download a message with its headers; publish a new message to a queue or exchange, typed in, from a file or as a copy.
- Throttled replay: at most N messages per second, running in the background.
- The audit log keeps each message's details: open a replayed message as it sat in the queue.
- Expanded message: cause card with a plain-language verdict, highlighted JSON, filtered headers, death table.
- "Why they died": loaded dead letters grouped by reason, queue, routing key and exception.
- Original route also from `x-original-*` headers of application-republished messages.
- Queue flow graph: publishers, consumers and dead-letter wiring around a queue.
- Peek only the head of each body: much faster listing of large messages.
- Retry wait queues are no longer classified as dead-letter queues.
- Management API over HTTP/1.1: no more "EOF reached while reading".
- Replay keeps what it did when the connection drops; clear 400s for bad requests.
- UI refresh: sidebar shell, Warren theme, stat strip on the queue list, message table polish.

## 0.2.0 (2026-09-22)

- Embedded database by default: one container, one volume. An external PostgreSQL becomes a Pro feature.

## 0.1.1 (2026-09-22)

- Queue list: a Pro-only alert call no longer breaks loading in Community.
- The image reports its release version.

## 0.1.0 (2026-09-22)

- First release: queue overview with dead-letter detection, message browser with `x-death` history, replay to the original route, a queue or an exchange, edit before replay, audit log, metrics history, alerting to Slack, Teams and webhooks, users and roles, OIDC sign-in.
