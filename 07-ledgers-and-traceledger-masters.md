# Ledgers and TraceLedger Masters

Economic systems do not fail at value. They fail at continuity.

A ledger is the continuity boundary. Once created, it cannot be modified, merged, or deleted. It can only accumulate events and eventually be closed. This permanence is not a limitation — it is the property that makes the record independently verifiable at any future point, by any permissioned party, without cooperation from any other participant.

## Ledger

A ledger is the constitutional boundary for one root continuity scope or one relationship scope.

Launch ledger types:

- `ORDER`
- `FACILITY_ROOT`
- `ENTITY_ROOT`
- `ENGAGEMENT`

## TraceLedger Master

A traceledger master is an execution scope under an existing ledger.

It does not create a new constitutional root.

It groups operational events such as:

- order execution
- facility workflow
- machine inspection
- shipment execution
- AI response chains

## Genesis

Every ledger begins with exactly one `GENESIS` event.

That rule applies to:

- one buyer and supplier relationship boundary
- one facility root boundary
- one entity root boundary
- one engagement boundary

## Execution Events

Execution events are appended after genesis. They live under the existing ledger boundary and can be grouped under a traceledger master.

## FRL or Root-Ledger Style Flows

`FACILITY_ROOT` is the organization sovereignty root.

`ENTITY_ROOT` is the asset continuity root.

Each has:

- one root ledger
- one genesis
- many subsequent events
- optional traceledger masters below that root

## Relationship-Ledger Style Flows

`ORDER` and `ENGAGEMENT` are relationship-ledger types.

They are used for:

- buyer and supplier collaboration
- bilateral operational workflows between any two actors

Each relationship ledger has one genesis for that relationship boundary.

## Root and Master Rules

The following rules define the launch contract:

- One relationship or root boundary has one genesis.
- Traceledger masters are execution scopes under that boundary.
- Traceledger masters do not create new constitutional roots unless the API explicitly creates a new root ledger.
- Multiple execution instances can exist under the same ledger boundary.
- Multiple events can exist under the same master.

## Order Example

- Boundary: one `ORDER` ledger between buyer and supplier
- Genesis: one order genesis
- Execution scope: a traceledger master such as `PO-2026-0001`
- Events: review, inspection, approval, AI response, handoff

## Facility Root Example

- Boundary: one `FACILITY_ROOT` ledger for one organization
- Genesis: one facility root genesis
- Execution scopes: inventory cycle, plant workflow, facility audit

## Engagement Example

- Boundary: one `ENGAGEMENT` ledger between two actors
- Genesis: one engagement genesis
- Execution scopes: machine service job, shipment exception handling, bilateral audit workflow

## Entity Root Example

- Boundary: one `ENTITY_ROOT` ledger for one machine, shipment, sensor, or asset
- Genesis: one entity root genesis
- Execution scopes: inspection record, maintenance record, compliance review, AI decision chain

## Create a Relationship Ledger

### `POST /ledgers`

Supports:

- explicit actor and organization ids
- counterparty actor URI resolution through `supplier_iaex_id` or `buyer_iaex_id`

Example:

```bash
curl -sS "$BASE_URL/ledgers" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "supplier_iaex_id": "'"$SUPPLIER_IAEX_ID"'"
  }'
```

## List and Read Ledgers

### `GET /ledgers`

Filters:

- `ledger_type`
- `status`
- `role`
- `limit`
- `offset`

### `GET /ledgers/{ledger_id}`

Returns the public ledger boundary fields for callers with read access.

### `PATCH /ledgers/{ledger_id}/close`

Closes the ledger. Closure is an appended event. It is not a deletion.

## Create a TraceLedger Master

### `POST /traceledger/master`

Request:

```json
{
  "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "business_ref": "PO-2026-0001",
  "scope": "ORDER"
}
```

Response:

```json
{
  "uuid": "aa2fa3c9-5a97-4f84-86f5-f7c2e98bb7ea",
  "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "ledger_type": "ORDER",
  "business_ref": "PO-2026-0001",
  "scope": "ORDER",
  "status": "OPEN",
  "created_at": "2026-04-21T06:35:00Z"
}
```

## Read and Close a Master

### `GET /traceledger/master?ledger_id={ledger_id}`

Lists masters under one ledger.

### `GET /traceledger/master/{uuid}`

Returns one master record.

### `PATCH /traceledger/master/{uuid}/close`

Closes the master by appending a closure event to the parent ledger.

## Read Master Events

### `GET /traceledger/master/{uuid}/events`

Returns only the events scoped to that master.

Filters:

- `event_type`
- `from`
- `to`
- `limit`
- `offset`
