# Launchable Integration Flows

## Buyer Opens a Ledger with a Supplier

1. Buyer and supplier each obtain a production actor and API key through `POST /developer/register`.
2. Both enroll actor signing keys through `POST /actors/{actor_id}/keys`.
3. Both onboard organizations through `POST /onboarding/organization`.
4. Buyer creates the ledger through `POST /ledgers` using `supplier_iaex_id`.

Request:

```bash
curl -sS "$BASE_URL/ledgers" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"supplier_iaex_id":"'"$SUPPLIER_IAEX_ID"'"}'
```

## Supplier Joins and Receives Delegated Access

The supplier joins automatically as the direct counterparty when the order ledger is created.

Additional supplier-side delegates can be granted through:

```bash
curl -sS "$BASE_URL/delegation/organization" \
  -X POST \
  -H "Authorization: Bearer $SUPPLIER_API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "'"$SUPPLIER_ORG_ID"'",
    "delegate_actor_id": "'"$SUPPLIER_OPS_ACTOR_ID"'",
    "role": "OPS",
    "permissions": {}
  }'
```

## A Delegated Actor Appends a Write Event

1. Buyer creates a traceledger master.
2. Buyer grants `write` access on that master.
3. The delegate appends an event to the master.

Grant master write access:

```bash
curl -sS "$BASE_URL/delegation/master" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{"master_id":"'"$MASTER_ID"'","delegate_actor_id":"'"$ERP_ACTOR_ID"'","role":"OPS","permissions":{"access":"write"}}'
```

Append:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -X POST \
  -H "Authorization: Bearer $ERP_API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $ERP_EVENT_SIG" \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "ERP_SYNC",
    "payload": {
      "traceledger_master_uuid": "'"$MASTER_ID"'",
      "reference": "sap-0001",
      "status": "SENT"
    }
  }'
```

## A Read-Only Delegate Views Events

Grant master read access:

```bash
curl -sS "$BASE_URL/delegation/master" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{"master_id":"'"$MASTER_ID"'","delegate_actor_id":"'"$AUDITOR_ACTOR_ID"'","role":"OPS","permissions":{"access":"read"}}'
```

Read:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -H "Authorization: Bearer $AUDITOR_API_KEY"
```

## A Master Scope Receives Execution Access

Execution access is granted through:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/access" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "actor_id": "'"$WORKFLOW_ACTOR_ID"'",
    "access": "write",
    "expires_at": "2026-04-30T23:59:59Z"
  }'
```

## An AI Response Event Is Recorded and Verified

Append:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Actor-Sig: $BUYER_AI_SIG" \
  -d '{
    "event_type": "AI_RESPONSE",
    "payload": {
      "model": "genesis-x1-audit",
      "decision": "PASS",
      "summary": "No material exception detected"
    }
  }'
```

Verify:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -H "Authorization: Bearer $BUYER_API_KEY"
```

Check:

- `integrity.verified == true`
- the returned `AI_RESPONSE` event hash
- the actor identity document through `/resolve/{iaex_actor_uri}`

## A Regulator or Auditor Verifies a Chain

1. Receive the ledger id or master id from the reporting party.
2. Read the event chain through the appropriate public read endpoint using an authorized actor.
3. Read `integrity.verified` and `integrity.issues`.
4. Resolve the actor URIs involved through `/resolve/iaex:actor:{uuid}`.
5. Read the authority trust anchor through `/.well-known/iaex-authority`.

## A Third-Party ERP System Pushes or Receives Events

Push:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -X POST \
  -H "Authorization: Bearer $ERP_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Actor-Sig: $ERP_EVENT_SIG" \
  -H 'X-Delivery-Chain: [{"actor_id":"'"$ERP_ACTOR_ID"'","actor_type":"service","role":"erp"}]' \
  -d '{
    "event_type": "ERP_SYNC",
    "payload": {
      "system": "SAP",
      "reference": "sap-0002",
      "status": "POSTED"
    }
  }'
```

Receive:

```bash
curl -sS "$BASE_URL/webhooks" \
  -X POST \
  -H "Authorization: Bearer $ERP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://erp.example.com/iaex/webhooks",
    "event_types": ["ERP_SYNC", "AI_RESPONSE"],
    "ledger_id": "'"$LEDGER_ID"'"
  }'
```

## Machine Onboarding

```bash
curl -sS "$BASE_URL/entities/onboard" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "display_name": "CNC-17",
    "entity_type": "MACHINE",
    "unique_id": "CNC-17-2026-0001",
    "owner_actor_id": "'"$SUPPLIER_ACTOR_ID"'"
  }'
```

## Shipment Trace

1. Onboard the shipment entity through `POST /entities/onboard`.
2. Create its ARL through `POST /entities/root-ledger`.
3. Create a traceledger master under the ARL for shipment execution.
4. Append shipment events to that master.

## Entity-Root Execution

Create master:

```bash
curl -sS "$BASE_URL/traceledger/master" \
  -X POST \
  -H "X-ACTOR-ID: $ENTITY_ACTOR_ID" \
  -H "X-ACTOR-TYPE: device" \
  -H "Content-Type: application/json" \
  -d '{
    "ledger_id": "'"$ENTITY_LEDGER_ID"'",
    "business_ref": "MACHINE-INSPECTION-2026-04-21",
    "scope": "PROCESS"
  }'
```

Append execution record:

```bash
curl -sS "$BASE_URL/traceledger/master/$ENTITY_MASTER_ID/events" \
  -X POST \
  -H "X-ACTOR-ID: $ENTITY_ACTOR_ID" \
  -H "X-ACTOR-TYPE: device" \
  -H "Content-Type: application/json" \
  -H "X-Actor-Sig: $ENTITY_EVENT_SIG" \
  -d '{
    "event_type": "INSPECTION_READING",
    "payload": {
      "temperature_c": 18.4,
      "status": "NORMAL"
    }
  }'
```

## Compliance Object Verification

For a compliance object or DPP object:

1. Onboard it as an entity.
2. Create its ARL if continuity is required.
3. Record compliance events under a master.
4. Verify the returned integrity block and actor identities through resolver reads.
