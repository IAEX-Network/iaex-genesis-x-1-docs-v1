# Quickstart

This quickstart creates two production actors, onboards both organizations, opens an order ledger, creates a traceledger master, appends a signed event, reads the resulting chain, verifies integrity, and registers a webhook.

The public API is served under `/v1`. The examples use:

```bash
export BASE_URL="https://api.iaexnetwork.com/v1"
```

## 1. Obtain developer access

Run the registration flow once for the buyer system and once for the supplier system.

```bash
curl -sS "$BASE_URL/developer/register" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"buyer-prod","region":"in"}'
```

```bash
curl -sS "$BASE_URL/developer/register" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"supplier-prod","region":"in"}'
```

Save these values from the two responses:

- `production.actor_id`
- `production.iaex_id`
- `production.api_key`

## 2. Onboard both organizations

Buyer:

```bash
curl -sS "$BASE_URL/onboarding/organization" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "supplier_name": "Northwind Buyer Operations",
    "legal_name": "Northwind Buyer Operations Private Limited",
    "country": "IN",
    "region": "KA",
    "city": "Bengaluru",
    "registered_address": "1 Industrial Avenue, Bengaluru",
    "vat_tax_number": "29ABCDE1234F1Z5",
    "primary_email": "buyer.ops@example.com",
    "terms_version": "2026-04-21",
    "declaration_hash": "buyer-onboarding-declaration-v1"
  }'
```

Supplier:

```bash
curl -sS "$BASE_URL/onboarding/organization" \
  -X POST \
  -H "Authorization: Bearer $SUPPLIER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "supplier_name": "Acme Components",
    "legal_name": "Acme Components Private Limited",
    "country": "IN",
    "region": "TN",
    "city": "Chennai",
    "registered_address": "45 Foundry Road, Chennai",
    "vat_tax_number": "33ABCDE1234F1Z5",
    "primary_email": "supplier.ops@example.com",
    "terms_version": "2026-04-21",
    "declaration_hash": "supplier-onboarding-declaration-v1"
  }'
```

Fetch organization ids:

```bash
curl -sS "$BASE_URL/suppliers/?actor_id=$BUYER_ACTOR_ID" \
  -H "Authorization: Bearer $BUYER_API_KEY"
```

```bash
curl -sS "$BASE_URL/suppliers/?actor_id=$SUPPLIER_ACTOR_ID" \
  -H "Authorization: Bearer $SUPPLIER_API_KEY"
```

Save the returned `organization_id` values as `BUYER_ORG_ID` and `SUPPLIER_ORG_ID`.

## 3. Create an order ledger

```bash
curl -sS "$BASE_URL/ledgers" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: quickstart-order-001" \
  -d '{
    "supplier_iaex_id": "'"$SUPPLIER_IAEX_ID"'"
  }'
```

Response:

```json
{
  "id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "buyer_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "supplier_actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "buyer_organization_id": "65ad9f1a-7d5b-4c18-89ae-4fd62d6726ba",
  "supplier_organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839",
  "status": "OPEN",
  "created_at": "2026-04-21T06:30:00Z"
}
```

Save the `id` as `LEDGER_ID`.

## 4. Create a traceledger master

```bash
curl -sS "$BASE_URL/traceledger/master" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "ledger_id": "'"$LEDGER_ID"'",
    "business_ref": "PO-2026-0001",
    "scope": "ORDER"
  }'
```

Save the returned `uuid` as `MASTER_ID`.

## 5. Enroll actor signing keys

Before using caller-defined event append, enroll an Ed25519 public key for each actor that will sign events.

```bash
curl -sS "$BASE_URL/actors/$BUYER_ACTOR_ID/keys" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "public_key": "'"$BUYER_PUBLIC_KEY_B64"'",
    "algorithm": "Ed25519"
  }'
```

```bash
curl -sS "$BASE_URL/actors/$SUPPLIER_ACTOR_ID/keys" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "public_key": "'"$SUPPLIER_PUBLIC_KEY_B64"'",
    "algorithm": "Ed25519"
  }'
```

## 6. Submit a signed event

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Actor-Sig: $BUYER_EVENT_SIG" \
  -H "Idempotency-Key: quickstart-master-event-001" \
  -d '{
    "event_type": "AI_RESPONSE",
    "payload": {
      "model": "genesis-x1-audit",
      "summary": "Initial review complete",
      "decision": "PASS",
      "reference": "review-0001"
    }
  }'
```

## 7. Read events

Ledger view:

```bash
curl -sS "$BASE_URL/ledgers/$LEDGER_ID/events" \
  -H "Authorization: Bearer $BUYER_API_KEY"
```

Master-scoped view:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -H "Authorization: Bearer $BUYER_API_KEY"
```

## 8. Verify integrity

The read response includes a top-level integrity block:

```json
{
  "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "count": 3,
  "integrity": {
    "verified": true,
    "issues": []
  },
  "events": []
}
```

`verified: true` means the returned chain segment passed server-side tamper and continuity verification.

## 9. Register a webhook

```bash
curl -sS "$BASE_URL/webhooks" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://erp.example.com/iaex/webhooks",
    "event_types": ["AI_RESPONSE"],
    "ledger_id": "'"$LEDGER_ID"'"
  }'
```

Store the returned `secret`. It is shown once.

## 10. Receive deliveries

List recent webhook delivery attempts:

```bash
curl -sS "$BASE_URL/webhooks/$WEBHOOK_ID/deliveries" \
  -H "Authorization: Bearer $BUYER_API_KEY"
```

## 11. Verify actor identity

Resolve the actor that signed the workflow:

```bash
curl -sS "$BASE_URL/resolve/$BUYER_IAEX_ID"
```

This returns the actor identity document and enrolled signing keys needed for public actor verification.
