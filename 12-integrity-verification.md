# Integrity Verification

Integrity is not a feature. It is a mathematical property of the ledger.

Every Genesis X-1 ledger is a hash chain. Each event seals the prior event's hash into its own digest. Any modification to any event in the chain — payload, timestamp, actor, sequence — breaks every downstream hash. This is not a policy. It is arithmetic.

`integrity.verified` is returned with every event read. It reflects a server-side recomputation of the chain. You should verify it yourself for any compliance, legal, or audit context. Do not rely solely on the server's assertion.

---

## What `verified: true` Means

`verified: true` means the server recomputed the hash chain across the returned event sequence and found no breaks. Specifically:

- No stored hash mismatches its recomputed value
- No broken link between successive events in the returned sequence

`verified: true` does not mean:

- The business content of any event is factually true
- The payloads are legally valid or commercially approved
- The chain is complete — a filtered query returns a segment, not the full ledger

---

## What `verified: false` Means

A verification failure means the server detected at least one of:

- A stored hash that does not recompute correctly against its stored inputs
- A broken `prev_hash` linkage between successive events

The response includes a machine-readable `issues` array identifying the specific event and failure type:

```json
{
  "integrity": {
    "verified": false,
    "issues": [
      {
        "event_id": "7ad7f8d7-76a1-49de-917f-0c5d5313da0b",
        "type": "hash_mismatch",
        "detail": "stored hash does not match recomputed hash"
      }
    ]
  }
}
```

A verification failure is a protocol-level event. Treat it as a verification failure, log it, and escalate before making any downstream trust decision.

---

## Independent Verification

The server's `verified: true` is a convenience. Independent verification requires no server trust.

Given the raw events returned by `GET /ledgers/{id}/events`, any party can recompute the chain locally:

```python
import hashlib, json

def verify_chain(events: list) -> bool:
    for i, event in enumerate(events):
        expected_prev = '' if i == 0 else events[i - 1]['hash']
        if event['prev_hash'] != expected_prev:
            return False
    return True
```

For actor signature verification (Layer A), fetch the actor's enrolled public key via `GET /resolve/iaex:actor:{actor_id}` and verify against the stored `actor_sig` and `actor_sig_payload` fields. No server involvement required after the initial fetch.

---

## What Integrity Certifies

| Certified | Not Certified |
|-----------|--------------|
| Record is unmodified since append | Factual correctness of payload content |
| Event sequence is correct | Commercial fairness of the transaction |
| Actor attribution is intact | Legal enforceability of off-platform claims |
| Tamper detection is active | Physical-world truth of asserted facts |

Genesis X-1 certifies record integrity, attribution, ordering, and tamper detection.

It does not certify truth.

---

## For Auditors and Regulators

An event chain with `verified: true` and valid actor signatures on every event constitutes:

- A tamper-evident record that no party can unilaterally alter after the fact
- Non-repudiation for every actor-signed event — the signing actor cannot deny having submitted it
- A causal history where events referencing prior hashes form a mathematically sealed sequence

This makes Genesis X-1 event chains suitable as evidence in commercial disputes, regulatory investigations, and compliance audits — without requiring cooperation from any other party to produce the record.
