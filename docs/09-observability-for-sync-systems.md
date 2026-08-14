# 09 — Observability for Sync Systems

*"It synced… probably" is not a state a business runs on.*

## Why sync systems need their own telemetry

A request/response app fails loudly: the user sees the error. An offline-first system fails **quietly and later** — the engineer's save succeeded (locally), and the failure happens hours afterward in a background drain nobody is watching. Without deliberate observability, your first signal is an accountant missing a week of records at month-end.

## The stack (boring on purpose)

Compose-deployed sidecar: **Prometheus** (metrics) + **Grafana** (dashboards/alerts) + **Loki + promtail** (logs) + **cAdvisor/node exporter** (host/containers). One `docker-compose.monitoring.yml`, deployable next to the app stack. Nothing exotic — the value is in *what* you measure:

## The metrics that matter

### Sync health (the product-level SLOs)

| Metric | Source | Alert shape |
|---|---|---|
| **Outbox depth per device** (p50/p95/max) | devices report queue length on each drain | p95 above threshold for > N hours |
| **Drain latency** — record created → server ack | server computes from client-created-at | p95 > 24h sustained |
| **Park rate** — ops parked / ops attempted | server + device counters | any sustained nonzero, by reason |
| **Auth-failure rate on sync endpoints** | API 401 counter by endpoint | spike vs. baseline (token-lifecycle bug signature) |
| **Devices silent > 72h** (enrolled but not syncing) | last-seen table | list, reviewed by ops weekly |
| **Sync-warning feed size** (doc 02) | backend table | growth trend |

These are business metrics wearing engineering clothes: "outbox depth p95" *is* "how much field data is currently at risk."

### API & workers

- Request rate / error rate / duration by endpoint (the sync endpoints get their own panels — they have different traffic shapes than the web portal).
- Job metrics per task: started/succeeded/failed/retried + duration histograms (doc 06).
- **`pg_stat_activity` panel**: total connections, idle-in-transaction count, oldest transaction age. Idle-in-transaction growth predicts two distinct incidents — pool exhaustion, and deploy-time migrations hanging behind zombie locks — days before either fires. Alert on `idle in transaction > 5m`.
- Queue depth in the broker; scheduler heartbeat (a beat that silently died is a classic).

### Mobile-side signals

Devices batch-report on each successful drain: app version, OS, queue depth, park count, last error class, DB schema version. Cheap to collect, and it turns fleet questions ("is 6.2 draining slower?") into dashboard filters instead of support-call archaeology. Version-code adoption curves also gate staged rollouts (doc 08).

## Logging that answers 2 a.m. questions

- **Structured JSON logs**, shipped via promtail, with the trinity on every sync-path line: `tenant_id`, `entity_type`, and the **idempotency key / op id**. The op id is the correlation id that exists *on both sides* of the sync boundary — the device log and the server log meet at it (doc 03).
- Log the **decision**, not just the exception: "op K parked: validation field=hours reason=overlap" beats a stack trace.
- Auth events get first-class logs: token issued/refreshed/rejected(reason). Token-lifecycle incidents are diagnosed almost entirely from this trail plus the 401 panel.
- Retention: sync disputes surface at month-end; keep at least 45–60 days searchable.

## Alerting philosophy

- Page on **user-impact SLOs** (drain latency, park rate, API error rate), tick-tock everything else into a daily digest. A sync system generates endless benign transients; alert fatigue here is self-inflicted.
- Every alert names its **runbook**: the query to run, the table to inspect, the safe remediation. An alert without a runbook is a notification, not an operation.
- Alert *channels* (email relay, webhook) are part of the deployment: config-manage them, and test the path after every stack redeploy — the failure mode "alerts configured but the relay env vars didn't survive the redeploy" is silent and total. A weekly synthetic test alert is two lines of cron and worth it.

## Dashboards as shared language

Three audiences, three dashboards:

1. **Ops:** the SLO board — sync health row on top, API/workers below, DB/host at the bottom.
2. **Back office:** the sync-warning feed *inside the admin portal* (not Grafana) — parked records, silent devices, rejected payloads, each with a resolution action (doc 02).
3. **Engineering:** per-release board — error classes, drain latency, and adoption by app version, watched during every staged rollout.

## Checklist

- [ ] Outbox depth, drain latency, park rate, 401 rate, silent-device list — dashboarded and alerted
- [ ] Idempotency key/op id present in logs on **both** device and server sides
- [ ] Per-task job metrics + broker depth + scheduler heartbeat
- [ ] `pg_stat_activity` panel with idle-in-transaction alert
- [ ] Devices batch-report version/queue/error telemetry on drain
- [ ] Structured logs, 45–60 day retention, tenant + entity + op id on every sync line
- [ ] Every page-level alert has a runbook; alert delivery synthetically tested after deploys
- [ ] Back-office sync-warning feed with resolution actions, in the product itself
