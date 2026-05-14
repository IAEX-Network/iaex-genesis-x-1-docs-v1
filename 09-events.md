# Events

Events are not database records. They are independently verifiable facts with continuity.

Once appended, an event cannot be modified, deleted, or reordered. It carries a cryptographic identity — who authorized it, exactly what was asserted, and where it falls in the chain. Any party with access can verify all three independently, without trusting the server that stored it.

## Two Independent Guarantees

An event append in Genesis X-1 delivers two separate proofs:

- **Actor authorization** — the actor holding a specific private key deliberately submitted this exact event type, to this exact ledger, with this exact payload
- **Chain continuity** — this event follows the previous event in an unbroken, tamper-evident sequence

These are independent guarantees. Chain continuity does not prove who wrote an event. Actor authorization does not prove the event's position in the chain. Both are required for full verification.

## Event Types and Payload Design

When appending events via `POST /traceledger/master/{uuid}/events`, the caller defines both the event type and the payload. Genesis X-1 makes no assumptions about schema — design them for the domain.

### Naming Convention

```
{NOUN}_{VERB}         → SHIPMENT_DISPATCHED
{NOUN}_{PAST_STATE}   → INVOICE_RAISED, PAYMENT_CONFIRMED
{NOUN}_{ACTION}       → QUALITY_CHECK_PASSED, CUSTOMS_CLEARED
```

### Example Payloads

```json
// INVOICE_RAISED
{
  "invoice_number": "INV-2026-001",
  "amount": 125000,
  "currency": "INR",
  "due_date": "2026-06-15"
}

// SHIPMENT_DISPATCHED
{
  "carrier": "BlueDart",
  "tracking_number": "BD123456789",
  "dispatched_at": "2026-05-10T08:00:00Z",
  "items": [{"sku": "WHEAT-001", "quantity": 500, "unit": "kg"}]
}

// QUALITY_CHECK_PASSED
{
  "inspector": "QualityLabs India",
  "grade": "A",
  "passed_at": "2026-05-09T14:00:00Z"
}

// PAYMENT_CONFIRMED
{
  "transaction_id": "TXN-987654",
  "amount": 125000,
  "currency": "INR",
  "method": "NEFT"
}

// CUSTOMS_CLEARED
{
  "customs_ref": "CUS-2026-9876",
  "cleared_at": "2026-05-11T16:00:00Z",
  "port": "JNPT"
}
```

## How Events Are Appended

At launch, public event append is exposed through:

- `POST /traceledger/master/{uuid}/events`

Other mutating endpoints also append internal business events as part of their own boundary actions, such as:

- organization onboarding
- ledger creation
- delegation changes
- ledger closure
- master closure
- entity root creation

## Actor Signature Coverage

The actor signature covers:

- event type
- target ledger id
- canonical payload before delivery-chain injection

This proves business intent and actor authorization.

## Chain Integrity Coverage

Chain integrity covers:

- append ordering
- continuity from the prior event
- tamper detection on stored payload and causal linkage

This proves chain continuity for what the platform stored.

## Why They Are Separate

- Business intent answers: which actor authorized this event.
- Chain continuity answers: whether the stored sequence still matches its immutable append history.

One does not replace the other.

## What Is Immutable

The following are immutable after append:

- event type
- event payload
- event hash
- event timestamp
- actor attribution
- causal link

Corrections are represented as later events, not in-place edits.

## Correction Lineage

If a new event corrects or supersedes an earlier event, include `caused_by_hash` in the append payload.

Example request:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Actor-Sig: $BUYER_CORRECTION_SIG" \
  -d '{
    "event_type": "CORRECTION_NOTE",
    "payload": {
      "caused_by_hash": "'"$PRIOR_EVENT_HASH"'",
      "summary": "Supersedes prior AI response",
      "reason": "Updated source document"
    }
  }'
```

The server stores the causal link separately and returns it as `caused_by_hash` in event reads.

## Public Event Read Shape

### `GET /ledgers/{ledger_id}/events`

Response shape:

```json
{
  "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "count": 3,
  "integrity": {
    "verified": true,
    "issues": []
  },
  "events": [
    {
      "id": "7ad7f8d7-76a1-49de-917f-0c5d5313da0b",
      "event_type": "PRODUCTION_STARTED",
      "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
      "actor_type": "service",
      "payload": {
        "traceledger_master_uuid": "aa2fa3c9-5a97-4f84-86f5-f7c2e98bb7ea",
        "product": "wheat-batch-A1",
        "quantity": 500,
        "supplier_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132"
      },
      "hash": "0fb1f53d2a44f6c97c45749bc8cddf6f71f086b74cbf45d252a5baab6bd8fe17",
      "caused_by_hash": null,
      "actor_sig": "base64-encoded-ed25519-signature",
      "signing_key_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132#key-1",
      "actor_sig_payload": {
        "traceledger_master_uuid": "aa2fa3c9-5a97-4f84-86f5-f7c2e98bb7ea",
        "product": "wheat-batch-A1",
        "quantity": 500,
        "supplier_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132"
      },
      "created_at": "2026-04-21T06:42:00Z"
    }
  ]
}
```

## Publicly Returned Fields

Public event reads return:

- `id`
- `event_type`
- `actor_id`
- `actor_type`
- `payload` — full stored payload (may include server-injected fields)
- `hash`
- `caused_by_hash`
- `actor_sig` — base64 Ed25519 signature (null for authority-signed events)
- `signing_key_id` — KID of the key that produced the sig (null for authority-signed)
- `actor_sig_payload` — exact subset the actor signed (null = actor signed full `payload`)
- `created_at`
- top-level `integrity`

## Fields Not Returned Publicly

Public event reads do not return:

- previous hash
- IP address
- user agent
- private audit annotations

## Independent Ed25519 Verification

Any party with the actor's public key can independently verify actor-signed events:

```
# For events where actor_sig_payload is non-null:
sigTarget = actor_sig_payload

# For events where actor_sig_payload is null:
sigTarget = payload

msg    = eventType + "\x00" + ledgerID + "\x00" + canonicalJSON(sigTarget)
digest = SHA-256(msg)
valid  = Ed25519.Verify(actorPublicKey, digest, base64Decode(actor_sig))
```

Authority-signed events (`GENESIS`, `ORDER_OPENED`, `LEDGER_CLOSED`, `ORGANIZATION_REGISTERED`) have `actor_sig = null` and are verified by the server's authority keyring, not by individual actors.

## Short Distinction

- Business intent: what the actor asked to record.
- Authorization: which actor signed that request.
- Chain continuity: whether the immutable sequence still links correctly.
- Integrity verification: the server-side result returned with event reads.
