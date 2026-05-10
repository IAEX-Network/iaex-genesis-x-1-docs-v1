# Delegation and Access Control

Access in Genesis X-1 is not assumed. It is declared, signed, and recorded.

No party can read or write to a ledger scope unless they are a direct constitutional party to it, or hold a delegation grant signed by a party with authority over that scope. Every grant is an actor-signed event. Every grant is part of the immutable ledger record. Every grant can be independently audited.

There is no implicit access. There is no ambient authorization. Access boundaries are the protocol's definition of trust.

## Direct Party Access

Direct parties to a ledger always have access to their own boundary.

For launch ledger types this means:

- `ORDER`: buyer and supplier actors
- `FACILITY_ROOT`: the founding organization actor
- `ENTITY_ROOT`: the entity actor
- `ENGAGEMENT`: the primary actor and the counterparty actor

## Delegated Access

Genesis X-1 supports delegated access at three public scope levels:

- organization delegation
- ledger delegation
- traceledger master execution access

## Read vs Write

Read and write permissions are explicit on traceledger master execution access:

- `read`: can read the master and master-scoped events
- `write`: can append events to the master and read the resulting chain

Organization and ledger delegations use role-bearing grants. Traceledger master execution access uses direct `read` or `write`.

## How Access Is Granted

### Organization delegation

- `POST /delegation/organization`

### Ledger delegation

- `POST /delegation/ledger`

### Master execution access

- `POST /delegation/master`

## How Access Is Revoked

### Organization delegation

- `PATCH /delegation/organization`

### Ledger delegation

- `PATCH /delegation/ledger`

### Master execution access

- `PATCH /delegation/master`

Revocation deactivates the grant. Historical visibility remains through list and event surfaces.

## How Expiry Works

Expiry is available on traceledger master execution access.

Organization and ledger delegations do not expose a public expiry field at launch.

If a master access grant carries `expires_at`, the server enforces expiry at read and write time.

## Owners and Delegates

Owners can:

- grant organization delegation
- revoke organization delegation
- grant ledger delegation when they are direct parties or active ledger owners
- grant master access when they are direct parties to the parent ledger
- list active master access grants

Delegates can:

- use the granted scope
- read their own master access grant through the master access listing endpoint

Delegates cannot:

- revoke someone else’s grant unless they are also an authorized owner in that scope
- append to a master when the grant is `read`
- continue access after revocation
- continue access after expiry

## Expired Access Behavior

When a master access grant is expired:

- reads fail with access denial for that expired scope
- writes fail with `access_expired`
- the historical grant record remains visible to owners

## Revoked Access Behavior

When a grant is revoked:

- new reads under that grant stop
- new writes under that grant stop
- historical records remain queryable where the caller still has another valid access path

## Listing Behavior

### Organization and ledger grant lists

- `GET /delegation/supplier`
- `GET /delegation/buyer`
- `GET /delegation/ledger`

These are authenticated scope reads.

### Point-in-time delegation snapshot

- `GET /delegation/snapshot`

This returns active delegations for one scope at a requested time.

### Master execution access list

- `GET /traceledger/master/{uuid}/access`

Visibility:

- direct owners see the full active grant list
- a delegated actor sees only its own active grant

## Owner Example

Grant organization delegation:

```bash
curl -sS "$BASE_URL/delegation/supplier" \
  -X POST \
  -H "Authorization: Bearer $SUPPLIER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "'"$SUPPLIER_ORG_ID"'",
    "delegate_actor_id": "'"$OPS_ACTOR_ID"'",
    "role": "OPS",
    "permissions": {
      "orders": true,
      "shipments": true
    }
  }'
```

## Read Delegate Example

Grant master read access:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/access" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "actor_id": "'"$AUDITOR_ACTOR_ID"'",
    "access": "read",
    "expires_at": "2026-05-01T00:00:00Z"
  }'
```

## Write Delegate Example

Grant master write access:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/access" \
  -X POST \
  -H "Authorization: Bearer $BUYER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "actor_id": "'"$ERP_ACTOR_ID"'",
    "access": "write"
  }'
```

## Expired Access Example

If the read delegate above calls:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -H "Authorization: Bearer $AUDITOR_API_KEY"
```

after `2026-05-01T00:00:00Z`, the request is denied because the grant is expired.

If the same actor attempts:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" \
  -X POST \
  -H "Authorization: Bearer $AUDITOR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"event_type":"AUDIT_NOTE","payload":{"summary":"late write"}}'
```

the request fails with an expiry-related access error.

## Revoked Access Example

Revoke a master execution grant:

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/access/$LINK_ID" \
  -X DELETE \
  -H "Authorization: Bearer $BUYER_API_KEY"
```

Revoke an organization delegation:

```bash
curl -sS "$BASE_URL/delegation/supplier" \
  -X PATCH \
  -H "Authorization: Bearer $SUPPLIER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "organization_id": "'"$SUPPLIER_ORG_ID"'",
    "delegate_actor_id": "'"$OPS_ACTOR_ID"'"
  }'
```
