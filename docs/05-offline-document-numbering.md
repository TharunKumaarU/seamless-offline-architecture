# 05 — Offline Document Numbering

*Humans need `TN1-DN-2026-00417`, not a UUID — including humans with no signal.*

## Why this is hard

Business documents (delivery notes, service reports, invoices) need short, human-readable, often legally meaningful numbers: unique, roughly sequential, prefixed by tenant/type/year, and **printed on a signed PDF the moment the document is created** — which happens offline. So the classic answer, "the server assigns the number," is off the table for exactly the documents that matter most, and "wait for connectivity" means a customer standing at a loading dock, not signing.

That leaves a genuinely distributed-systems problem hiding in a business-formatting feature. There are three honest strategies; most real systems need two of them side by side (chosen per document type).

## Strategy A — Server-minted (online-only documents)

For documents only ever created in the office (purchase orders, invoices): a counters table, incremented in the row-insert transaction.

```sql
CREATE TABLE doc_counters (
  tenant_id  bigint NOT NULL,
  doc_type   text   NOT NULL,
  year       int    NOT NULL,
  next_value int    NOT NULL DEFAULT 1,
  PRIMARY KEY (tenant_id, doc_type, year)
);

-- atomic claim, safe under concurrency:
UPDATE doc_counters SET next_value = next_value + 1
 WHERE tenant_id=$1 AND doc_type=$2 AND year=$3
RETURNING next_value - 1;
```

Notes: a plain table beats PostgreSQL sequences here because you need one counter *per (tenant, type, year)* with a yearly reset, and you want the claim inside the business transaction. Gaps still happen on rollbacks — see "gap-free" below.

## Strategy B — Device-minted with collision-proof identity (offline documents)

The device must print a number now. Make the number **globally unique by construction**, so two offline devices can never mint the same one — embed a per-actor component:

```
{TENANT}-{TYPE}-{DATE}-{ENGINEER_CODE}-{PER_DEVICE_SEQ}
e.g.  TN1-DN-20260813-E042-3
```

- Uniqueness needs no coordination: the engineer code partitions the space, the local sequence orders within it.
- **Mint exactly once, then freeze.** The number is part of the record (like its idempotency key, doc 03): minted at creation, stored, reused on every retry/edit/resubmit. Never re-derive it from "current date" or "current user" at render time — a document created Monday and completed Wednesday keeps Monday's number; a document drafted by one engineer must not re-stamp with whoever reopened it. Every re-derivation is a future duplicate or identity bug.
- The server treats a device-minted number as an **opaque natural key**: uniqueness-checked on ingest (with the idempotency key resolving true retries vs. genuine collisions), never rewritten.

Trade-off: numbers are not globally sequential and encode structure some back offices dislike. That is the price of offline; put it in front of stakeholders explicitly rather than letting anyone assume A-style sequences.

## Strategy C — Pre-allocated ranges (when format is non-negotiable)

If policy demands office-style sequential numbers even offline: devices lease ranges from the server while online (`DN-2026-004200…004299`), mint locally from the lease, renew when low.

Works, but brings real costs: leases must be persisted and re-issued safely across reinstalls (or blocks are burned), devices offline longer than their lease runway **stop minting** (which is a hard stop for the field), and unused ranges become permanent gaps. Choose C only when B is truly unacceptable — and size leases generously.

## The reseed trap (B's one sharp edge)

A per-device sequence lives in local storage. Reinstall or clear-data wipes it, and a naive `seq = 1` re-mints numbers the device already used. Recovery order on fresh login: ask the server for the max sequence previously seen from *this engineer* (+ device, if numbers encode device), reseed above it, and only then enable creation. The idempotency layer will stop exact retries regardless, but reseeding prevents *new* documents from colliding with old ones.

## "Gap-free" numbering

Auditors sometimes ask for gapless sequences. Be precise about what that costs: gapless means the number is assigned only at final commit (never at draft/print time), rollbacks force renumber-or-reserve logic, and offline minting is essentially excluded. In practice: scope gap-free to the few document types that legally require it (A-style, online, assigned at posting time), and give everything else uniqueness + monotonicity per actor. Write the decision down per document type — this is a compliance conversation, not an engineering preference.

## Numbers are not identity

The join key between systems and devices is the **server ID / idempotency key**, never the human-readable number. Numbers get retyped, restamped by legacy imports, occasionally corrected by admins. Any pipeline that joins on document number will eventually mis-link two documents — I have watched it happen in migrated data (doc 11). Display the number everywhere; join on it nowhere.

## Checklist

- [ ] Each document type explicitly assigned strategy A, B, or C — written down
- [ ] Device-minted numbers embed an actor component (no coordination-free collisions)
- [ ] Number minted once at creation, frozen, round-trips every serialization
- [ ] No render-time re-derivation of date/user components
- [ ] Server ingest: unique-check + idempotency key distinguishes retry vs. collision
- [ ] Reseed-above-server-max on reinstall/fresh login, before creation is enabled
- [ ] Gap-free demands scoped to the document types that truly require them
- [ ] All joins use IDs/keys; numbers are display-only
