# Webhooks

## Subscription Model

A webhook subscription is owned by one authenticated actor.

Create one with:

- `POST /webhooks`

List owned subscriptions with:

- `GET /webhooks`

Deactivate a subscription with:

- `DELETE /webhooks/{id}`

## Delivery Model

IAEX fan-outs matching event records to active subscriptions.

A subscription can filter by:

- `event_types`
- `ledger_id`

If no `event_types` are supplied, the subscription receives all matching accessible events.

## Minimal Subscription Example

```bash
curl -sS "$BASE_URL/webhooks" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://erp.example.com/iaex/webhooks",
    "event_types": ["AI_RESPONSE", "LEDGER_CLOSED"],
    "ledger_id": "'"$LEDGER_ID"'"
  }'
```

Response:

```json
{
  "id": "8bb61f70-4871-4d14-bb14-a587f2d0204e",
  "url": "https://erp.example.com/iaex/webhooks",
  "event_types": ["AI_RESPONSE", "LEDGER_CLOSED"],
  "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "active": true,
  "created_at": "2026-04-21T06:50:00Z",
  "secret": "m2xZ0FjV6vr3-0KQX8Ff8tdU2bEmB8B_iKAKxqW6eR8",
  "note": "Store this secret securely. It cannot be retrieved again."
}
```

## Delivery Envelope

Webhook deliveries use this JSON envelope:

```json
{
  "api_version": "2026-04-14",
  "event": {
    "id": "7ad7f8d7-76a1-49de-917f-0c5d5313da0b",
    "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
    "event_type": "AI_RESPONSE",
    "payload": {
      "traceledger_master_uuid": "aa2fa3c9-5a97-4f84-86f5-f7c2e98bb7ea",
      "model": "genesis-x1-audit",
      "decision": "PASS"
    },
    "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
    "created_at": "2026-04-21T06:42:00Z"
  },
  "created_at": "2026-04-21T06:50:05Z"
}
```

## Webhook Signing Model

Each delivery includes:

```http
X-IAEX-Signature: sha256=<hex_hmac>
```

The signature input is:

```text
HMAC-SHA256(secret, "iaex-webhook-v1:" + raw_request_body)
```

## Verification Example

```python
import hmac
import hashlib

def verify_webhook(raw_body: bytes, secret: str, header_value: str) -> bool:
    expected = hmac.new(
        secret.encode("utf-8"),
        b"iaex-webhook-v1:" + raw_body,
        hashlib.sha256,
    ).hexdigest()
    return header_value == f"sha256={expected}"
```

## Retry Behavior

Delivery attempts follow this schedule:

- attempt 1: immediate
- attempt 2: 30 seconds later
- attempt 3: 5 minutes later
- attempt 4: 30 minutes later
- attempt 5: 2 hours later

After the final attempt, the delivery is marked `FAILED`.

## Idempotency Expectations

Webhook receivers must treat delivery as at-least-once.

Use the event id as the idempotency key on the receiver side.

## Debugging Failed Delivery

Use:

- `GET /webhooks`
- `GET /webhooks/{id}/deliveries`

Delivery history returns:

- delivery id
- event id
- ledger id
- status
- attempt count
- last error
- delivery timestamps

## Cross-System Integration

Webhooks are intended for:

- ERP ingestion
- audit mirrors
- workflow triggers
- external notification services
- regulator or partner sinks that consume new event records
