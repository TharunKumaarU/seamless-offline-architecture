# 06 — Background Jobs

*Everything slow, flaky, or external runs behind the request — and everything behind the request will one day run twice.*

## What belongs in the queue

In a field platform: PDF rendering, spreadsheet exports, email dispatch, push fan-out, document-store uploads, nightly cleanups, scheduled report digests. The request path only ever: validates, commits, enqueues, returns. Users (and the mobile drain loop, doc 02) never wait on a third-party system.

Stack assumed here: Celery + Redis broker + a beat scheduler, but every rule below is queue-agnostic.

## Rule 1 — Enqueue after commit

The classic race: task enqueued inside the transaction, worker picks it up before the commit lands, reads the database, finds nothing.

```python
# WRONG
with session.begin():
    report = create_report(session, payload)
    render_pdf.delay(report.id)        # worker may run before COMMIT

# RIGHT — enqueue on the after-commit hook
with session.begin():
    report = create_report(session, payload)
    on_commit(lambda: render_pdf.delay(report.id))
```

For jobs that must survive a broker outage too, go full **transactional outbox** (same pattern the mobile app uses): write the job intent to a table in the business transaction; a relay drains it to the broker. One mechanism, learned once, used on both edges of the system.

## Rule 2 — Every job is idempotent

At-least-once delivery is the contract; retries, worker crashes, and visibility-timeout re-deliveries all re-run jobs. Same discipline as doc 03:

- Natural idempotency where possible: `render_pdf(report_id)` that *overwrites* the artifact is safely re-runnable; "append a row" / "send an email" is not.
- For jobs with external side effects (email, document-store upload): record the side effect (`email_log`, `upload_log` with a unique key per intent) and check before acting. The log doubles as your support tool for "did the customer get it?".
- **Idempotent ≠ harmless to duplicate *concurrently***: two workers rendering the same PDF simultaneously can interleave partial writes. Guard hot jobs with a cheap distributed lock or a uniqueness claim, and write artifacts atomically (temp name → rename).

## Rule 3 — Own your connection lifecycle

The bug class that produces mysterious deploy-time hangs and pool exhaustion: resources acquired at task start and released only on the *happy* path — or worse, disposed at the start of the **next** task, which leaves the last task's connection dangling forever.

```python
# The shape that works: acquire late, release in finally — every task, no exceptions
@app.task(bind=True, max_retries=5)
def render_pdf(self, report_id):
    engine = get_engine()
    try:
        with engine.connect() as conn:
            ...
    finally:
        engine.dispose()      # symmetric cleanup, even on failure/retry
```

Symptoms that this rule is being violated somewhere: idle-in-transaction sessions accumulating in `pg_stat_activity`, schema migrations (`ALTER TABLE`) hanging behind invisible locks during deploys, and pool-checkout warnings under modest load. Put a `pg_stat_activity` panel on the ops dashboard (doc 09) — leaked connections announce themselves there long before an incident.

## Rule 4 — Retries with shape, dead-letters with owners

- Exponential backoff + jitter; cap attempts (5-ish for external calls).
- Classify errors like the mobile outbox does: transient (retry) vs. permanent (park). A malformed payload will not become well-formed on attempt 4.
- Parked/dead-lettered jobs go somewhere a human *actually looks* — a table surfaced in the admin portal beats a broker DLQ nobody opens. Every dead-letter category needs an owner and a replay path.

## Rule 5 — Schedulers are single-instance, jobs are guarded anyway

Run exactly one beat/scheduler instance (it's a singleton by design). But protect scheduled jobs against overlap regardless — a slow nightly job overlapping its next firing is the classic 3 a.m. pile-up:

```python
@app.task
def nightly_cleanup():
    if not acquire_lock("nightly_cleanup", ttl=3600):
        return                      # previous run still going — skip, don't stack
    try:
        ...
    finally:
        release_lock("nightly_cleanup")
```

Also: containerized schedulers write state files (e.g. a schedule database) — make sure the runtime user owns that path, or the scheduler crash-loops on restart. It's a five-minute fix that has taken down more than one otherwise-healthy deploy.

## Rule 6 — Jobs emit evidence

Minimum telemetry per job: started/succeeded/failed counters by task name, duration histogram, retry count, and — for pipeline jobs — the business key (report id, tenant) in structured logs. "The export didn't arrive" must be answerable by grepping one log stream (doc 09).

## Checklist

- [ ] Request path never calls external systems inline; commit → enqueue → return
- [ ] Enqueue on after-commit hooks (or a transactional job-outbox table)
- [ ] Every task idempotent; external side effects deduped via a keyed log
- [ ] Concurrent-duplicate guards on hot artifact-producing jobs; atomic artifact writes
- [ ] Connections/engines acquired per task, released in `finally` — verified via `pg_stat_activity` panel
- [ ] Backoff + jitter + attempt caps; permanent failures parked visibly with an owner
- [ ] Single scheduler instance; overlap locks on long scheduled jobs; writable state dir
- [ ] Per-task metrics + business-keyed structured logs
