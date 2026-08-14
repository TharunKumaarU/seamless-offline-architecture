# 12 — Reference Implementation Spec ("FieldSync")

*The blueprint for a runnable, open-source demonstration of this playbook. Built from scratch, seeded with fake data, scoped to be finishable.*

## Purpose

Prove the patterns in docs 01–11 with working code a reviewer can run in ten minutes:

```bash
git clone <repo> && cd fieldsync
docker compose up
# → API on :8000 (OpenAPI docs), admin portal on :3000
# → make seed-demo  → demo tenants, users, work orders
# → Flutter app in mobile/ — point at localhost, log in, go airplane-mode, keep working
```

## Scope (deliberately small)

**In:** two tenants; users/roles with layered permissions (doc 04); work orders (bidirectional sync); time entries + one report type — a service report with photos and signature (push-only); master data: customers + sites (pull-only, snapshot swap); document numbering strategy B (doc 05); outbox + idempotency end-to-end (docs 02–03); PDF render + email log as background jobs (docs 06–07); sync-warning feed; seeded fake data (Faker); tests as a first-class deliverable.

**Out (documented as such):** payments/invoicing, i18n, iOS store packaging, SSO, more report types. A small complete system demonstrates more than a large half-finished one — and the omissions are stated in the README so they read as scoping, not gaps.

## Architecture

Exactly doc 01's shape — FastAPI modular monolith (`apps/` + `shared/`), PostgreSQL 15+, Redis, Celery worker + beat, React + MUI admin (built fresh — no purchased admin template), Flutter + Drift + Riverpod mobile, Docker Compose with an optional `--profile monitoring` (Prometheus/Grafana/Loki per doc 09).

## The sync protocol (concrete)

### Push (device → server)

```
POST /api/v1/sync/{entity}
Headers: Authorization: Bearer <jwt>
         Idempotency-Key: <op_id>
Body:    { client_record, client_created_at, depends_on_server_ids? }
201 →    { server_id, canonical_record, applied: true }
200 →    { server_id, canonical_record, applied: false }   # idempotent replay
422 →    { park_reason, field_errors[] }                    # device parks op
409 →    { conflict: {server_version, fields} }             # bidirectional only
```

### Pull (server → device)

```
GET /api/v1/sync/{entity}?cursor=<opaque>&limit=500
200 → { changes: [...], deletions: [...], next_cursor, snapshot_version }
```

Server-issued cursors; device applies batch + stores cursor atomically; `POST /api/v1/sync/full-resync/{entity}` as the escape hatch (doc 02).

### Device schema (Drift)

`outbox(op_id PK, entity_type, entity_local_id, operation, payload, depends_on, state, attempt_count, last_error, created_at)` — plus per-entity tables carrying `local_id`, `server_id?`, `idempotency_key`, `doc_number?`, `sync_state` *derived from outbox joins*, never stored independently (doc 02's chip rule).

## Test plan (the point of the exercise)

- **Backend:** idempotency suite (all seven checklist cases from doc 03), tenant-isolation suite (the five IDOR shapes from doc 04/10), cursor-atomicity property test, counter-concurrency test (doc 05), enqueue-after-commit test (doc 06).
- **Mobile:** outbox state-machine unit tests (no absorbing states except DONE); integration: create-offline → kill app mid-drain → relaunch → exactly-once on server; migration upgrade-path test N→N+1 with non-empty outbox (doc 08).
- **End-to-end:** docker-compose CI job that seeds, runs a scripted device session (Flutter integration test) against the live stack, and asserts parity.
- **Chaos flag:** `CHAOS_HTTP_FAILURE_RATE=0.3` in the API for demos — the app should feel *identical* to use, which is the whole thesis.

## Repository plan

```
fieldsync/
├── README.md            # pitch, 10-minute quickstart, screenshots/GIF, honest scope
├── docs/                # ADRs + this spec + sync-protocol.md + threat-model.md
├── api/                 # FastAPI app (apps/, shared/, tests/)
├── web/                 # React admin (fresh MUI theme)
├── mobile/              # Flutter app (lib/, test/, integration_test/)
├── deploy/              # compose files, monitoring profile, seed scripts
└── .github/workflows/   # lint + tests + compose e2e
```

Development happens in the open: incremental conventional commits, CI green from the first week, ADRs written when decisions are made (not backfilled). The commit history is part of the portfolio.

## Milestones

| # | Milestone | Proof |
|---|---|---|
| 1 | Skeleton + compose + CI + auth + tenancy | `docker compose up` → login works; isolation tests green |
| 2 | Outbox + idempotency, time entries e2e | kill-mid-drain test green; chaos flag demo |
| 3 | Work orders bidirectional + conflicts + parking UI | conflict scenario scripted in integration test |
| 4 | Master-data snapshot pull + doc numbering | airplane-mode delivery-note demo with printed number |
| 5 | Service report + photos + PDF pipeline | signed PDF artifact from fake data |
| 6 | Sync-warning feed + monitoring profile + polish | Grafana board screenshot; README GIF |

---

*This spec is the "what." Docs 01–11 are the "why." The build is the receipts.*
