# Changelog

All notable changes to Warren. Images are published as `ghcr.io/viba88/warren:<version>`,
`:<major>.<minor>` and `:latest`.

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
