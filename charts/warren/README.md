# Warren

Dead-letter operations for RabbitMQ: see what is stuck in your dead-letter queues, read the messages
and why they died, replay them to their original route with a full audit trail, and get alerted
before a queue becomes a problem. Self-hosted next to your broker. [warrenops.io](https://warrenops.io)

## Install

```bash
helm install warren oci://ghcr.io/viba88/charts/warren \
  --set rabbitmq.managementUrl=http://rabbitmq:15672 \
  --set rabbitmq.host=rabbitmq \
  --set rabbitmq.username=warren \
  --set rabbitmq.password='…'
kubectl port-forward svc/warren 8080:8080
```

Sign in as `admin`; the generated password is printed in the notes. The RabbitMQ user needs the
`management` tag and read/write on the vhost, plus `monitoring` for disk and memory of the nodes.

## What the chart runs

- One pod with the embedded database on a persistent volume (`persistence.size`, default 2 Gi). The
  volume is kept when the release is uninstalled; the audit log lives there.
- Updates use the `Recreate` strategy, since the embedded database belongs to one instance. With
  `externalDatabase` (Pro) several replicas share a PostgreSQL.
- Non-root (UID 10001), read-only root file system, all capabilities dropped; liveness and readiness
  from `/actuator/health`.

## Common settings

| Value | Default | |
|---|---|---|
| `rabbitmq.*` | `rabbitmq:5672`, vhost `/` | The broker Warren watches; `rabbitmq.existingSecret` for the password |
| `clusters` | `[]` | Several brokers or vhosts (Team, Pro); replaces `rabbitmq` |
| `admin.password` / `admin.existingSecret` | generated | The first administrator; further users in the UI |
| `license.key` / `license.existingSecret` | – | Team or Pro; without a key Warren runs the free Community edition |
| `externalDatabase.*` | off | Pro: `jdbc:postgresql://…` instead of the embedded database |
| `oidc.*` | off | Pro: single sign-on |
| `ingress.*` | off | Hosts, class and TLS |
| `env` | `{}` | Any other `WARREN_*` setting, e.g. `WARREN_AUDIT_RETENTION: 90d` |

All settings are in [values.yaml](values.yaml); Warren's own configuration in the
[README](https://github.com/ViBa88/warren-releases#readme).

## Editions

Community is free: one broker with up to three vhosts. Team adds more brokers, roles, history,
alerting and replay rules; Pro adds SSO, an external database, four-eyes approval and audit export.
[Pricing](https://warrenops.io/#pricing)
