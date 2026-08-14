# 03 — End-to-End Idempotency

*Exactly-once effects on an at-least-once network. Every duplicate-record bug is an idempotency-key lifecycle bug.*

## The contract

- **Client:** every create/submit operation carries an idempotency key, minted **exactly once, at the moment of user intent**, stored durably with the record, and reused verbatim on every retry, resubmit, re-login, and reinstall-restore.
- **Server:** the same key for the same operation type returns the **same outcome** — first request executes, every replay returns the stored result. Not "error: duplicate"; the *same success response*.

When both halves hold, retries become free and the network can be as rude as it likes.

## Where duplicates actually come from

Across everything I have debugged, duplicate business records reduce to five lifecycle bugs:

### 1. Key minted per attempt instead of per intent

```dart
// WRONG — every retry is a "new" operation
Future<void> submit(Entry e) async {
  final key = uuid.v4();               // minted at call time
  await api.submitEntry(e, key: key);
}

// RIGHT — the key is part of the record, born with it
final entry = Entry(
  localId: uuid.v4(),
  idempotencyKey: uuid.v4(),           // minted ONCE at creation/save
  ...
);
// every attempt, forever, sends entry.idempotencyKey
```

### 2. Key dropped on an edit/reopen round-trip

The subtle one. A record is created (key minted), synced, then reopened for editing. If the "load for edit" path — especially a GET from the server — doesn't carry the key back, the next save mints a fresh key, and the resubmit creates a sibling record.

**Rule:** the key is part of the entity's identity. Every serialization — local DB, sync payload, server response, edit form state — round-trips it. Write a test that specifically does *create → sync → reopen → resubmit* and asserts one server row.

### 3. Double-tap / impatient resubmit in the UI

Online mode makes this worse: the first request is in flight, the user taps again, both requests race. Two guards, both needed:

- **Client:** an in-flight flag disables the action (`if (_isSaving) return; _isSaving = true; try {...} finally {_isSaving = false;}`).
- **Server:** the idempotency table catches whatever the UI misses — including the two-devices-same-user case the client can't see.

### 4. Post-commit side-effect failure

The server commits the row, then a downstream step fails — generating an export file, calling a document-management system, sending an email — and returns 500. The client dutifully retries; without a server-side key check the retry **re-creates the row**.

**Rule:** the row commit and the idempotency record must be in the **same transaction**; side-effects run after, retried independently (doc 06). A request must never be able to fail "after the important part succeeded" in a way that invites a duplicating retry.

### 5. Key uniqueness enforced in application code only

Application-level "check then insert" has a race window. The database is the arbiter:

```sql
CREATE TABLE idempotency_records (
  key           uuid        NOT NULL,
  operation     text        NOT NULL,   -- e.g. 'time_entry.create'
  tenant_id     bigint      NOT NULL,
  response_body jsonb       NOT NULL,   -- what we returned the first time
  entity_id     bigint,
  created_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (key, operation)
);
```

Insert this record **in the same transaction as the business row**; on unique-violation, read the stored `response_body` and return it.

## Server-side flow

```mermaid
sequenceDiagram
    participant C as Client (retry #N)
    participant A as API
    participant DB as PostgreSQL

    C->>A: POST /sync/time-entries (Idempotency-Key: K)
    A->>DB: BEGIN
    A->>DB: INSERT idempotency_records(K, op) 
    alt fresh key
        A->>DB: INSERT time_entries ... RETURNING id
        A->>DB: UPDATE idempotency_records SET response_body, entity_id
        A->>DB: COMMIT
        A-->>C: 201 {server_id, ...}
    else unique violation (replay)
        A->>DB: ROLLBACK; SELECT response_body WHERE key=K
        A-->>C: 201 {server_id, ...}   ← identical response
    end
```

Design details worth copying:

- **Scope keys per operation type** (`(key, operation)` PK) so a key can't be accidentally replayed across endpoints.
- **Store the response, return the response.** Replays must be indistinguishable from the original success — same status, same body — or client state machines fork.
- **Keys are opaque UUIDs from the client.** Don't derive them from content hashes; two legitimately identical time entries (same job, same hours, next day — clock issues happen) are still two intents.
- **Retention:** replays arrive days later from offline devices. Keep records for a multiple of your longest realistic offline window (e.g. 90 days), then archive.
- **Concurrent duplicate in flight:** two requests, same key, at the same instant — one wins the insert, the other blocks on the unique index until commit, then reads the response. That's the database doing the mutual exclusion; don't rebuild it with Redis locks unless you must.

## Idempotency for updates and workflow actions

Creates get the headlines, but *actions* duplicate too — "approve", "submit", "send email". Options in increasing strength:

1. **Natural idempotency:** `status = 'approved'` is safe to apply twice. Prefer state-setting over state-toggling semantics everywhere you can.
2. **Version guards:** client sends `expected_version`; server rejects mismatches (this doubles as your optimistic-concurrency conflict detector for bidirectional data).
3. **Keyed actions:** the same idempotency-record mechanism, one key per user intent ("this tap of the Approve button").

## Testing checklist

- [ ] create → retry same key → **one** row, identical responses
- [ ] create → sync → reopen → edit → resubmit → **one** row (key survived the round-trip)
- [ ] double-tap Save online → **one** row
- [ ] server 500 *after* commit (fault-injected) → retry → **one** row
- [ ] two devices, same payload, different keys → **two** rows (intent is the unit, not content)
- [ ] replay after 30 simulated offline days → stored response returned
- [ ] keys visible in logs/traces at both ends (you will need them at 2 a.m.; see doc 09)
