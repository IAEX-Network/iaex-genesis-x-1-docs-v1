# Entity and Root Ledger Model

## Entity

An entity is a non-human operational identity that must be tracked, verified, and anchored in trust continuity.

Examples:

- machine
- RFID unit
- shipment
- container
- production batch
- asset
- sensor
- vehicle
- facility object
- compliance object
- digital product passport object

An entity in Genesis X-1 is both:

- an actor for authorization and event attribution
- a registered entity record for continuity, classification, and public lookup

## ARL

ARL means Asset Root Ledger.

In the public API the ARL is the `ENTITY_ROOT` ledger for an entity. It is the continuity root for that entity. It is self-referential. The entity actor is both sides of the ledger boundary.

An ARL is used when developers need immutable continuity for a machine, shipment, batch, sensor, asset, or other tracked operational object.

## FRL vs ARL

FRL means Facility Root Ledger. It is the organization sovereignty root.

ARL means Asset Root Ledger. It is the entity continuity root.

FRL and ARL are different:

- FRL anchors organization continuity.
- ARL anchors entity continuity.
- A relationship ledger anchors a business relationship between parties.

These can coexist. A single integration may use all three:

- a supplier and buyer relationship ledger for commercial exchange
- a facility root ledger for the supplier organization
- an asset root ledger for a shipment, machine, batch, or compliance object

## Relationship Ledger vs Entity Root

A relationship ledger is not an entity root ledger.

Example:

- `ORDER` ledger: buyer and supplier relationship boundary
- `ENTITY_ROOT` ledger: shipment or machine continuity boundary

The same shipment can appear in events under both boundaries:

- its own ARL for asset continuity
- a buyer and supplier ledger for commercial coordination

## One Boundary, One Genesis

Genesis X-1 applies one genesis per constitutional boundary.

- One organization root has one genesis.
- One entity root has one genesis.
- One buyer and supplier relationship ledger has one genesis.
- One engagement ledger has one genesis.

Traceledger masters operate under those boundaries. They do not create a new constitutional root unless the API explicitly exposes a new root ledger.

## Entity Onboarding

### `POST /entities/onboard`

Creates an entity record and an actor identity for that entity.

Request fields:

- `display_name`: operator-facing name for the entity
- `entity_type`: caller-defined class such as `MACHINE`, `SHIPMENT`, `RFID`, `SENSOR`, or another operational label
- `unique_id`: external identifier such as a serial number, RFID code, asset tag, shipment number, or batch code
- `owner_actor_id`: optional actor that owns the entity at registration time
- `metadata`: caller-defined metadata map

Example:

```bash
curl -sS "$BASE_URL/entities/onboard" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "display_name": "CNC-17",
    "entity_type": "MACHINE",
    "unique_id": "CNC-17-2026-0001",
    "owner_actor_id": "'"$SUPPLIER_ACTOR_ID"'",
    "metadata": {
      "facility_code": "PLANT-01",
      "model": "CNC-X17"
    }
  }'
```

Response:

```json
{
  "entity_id": "1f5b2ff6-13e7-4f77-a88c-c3a1a2d1f7d2",
  "actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "actor_type": "device",
  "entity_type": "MACHINE",
  "display_name": "CNC-17",
  "trust_level": "BASIC",
  "unique_id": "CNC-17-2026-0001",
  "fingerprint": "4d5ab4d6d4a0a0e15b6602f03b22fd7e6127eaaf8dc04b5f4c0b0dcb72b5f8d5"
}
```

Meaning of public response fields:

- `entity_id`: stable entity record identifier
- `actor_id`: entity actor identifier used in authenticated and signed operations
- `actor_type`: event attribution class resolved for the entity
- `fingerprint`: stable deduplication fingerprint for this entity registration request
- `trust_level`: registration trust state exposed by the platform

## Fingerprint Meaning

`fingerprint` is a stable deduplication digest derived from the entity type and unique identifier. It helps the caller detect duplicate onboarding attempts and reconcile the same operational object across systems.

The public read APIs do not expose the fingerprint after onboarding.

## Trust Level Meaning

At launch, entity onboarding returns `BASIC`.

`BASIC` means the entity was registered through the public API and has a platform record.

`BASIC` does not mean:

- independent factual validation
- legal certification
- physical inspection
- regulatory approval

## Root Ledger Creation

### `POST /entities/root-ledger`

Creates the ARL for the authenticated entity actor.

Use this endpoint when the entity needs a constitutional continuity root before participating in operational event flows.

The ARL genesis marks the beginning of trace continuity for that entity. It is the operational commencement point for the entity inside IAEX.

Example:

```bash
curl -sS "$BASE_URL/entities/root-ledger" \
  -X POST \
  -H "X-ACTOR-ID: $ENTITY_ACTOR_ID" \
  -H "X-ACTOR-TYPE: device"
```

Response:

```json
{
  "ledger_id": "5c8dbe20-61e4-4f5f-bf3a-c726e6321f55",
  "entity_actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "ledger_type": "ENTITY_ROOT",
  "status": "OPEN",
  "genesis_hash": "95f7bde1a7d879eb0739c9d938034fbdb7b70d4e8863f255b318dbd5651f5c9b"
}
```

If the entity already has an ARL, the endpoint returns the existing ledger instead of creating another one.

## ARL Gate

### `PATCH /entities/arl-gate`

Controls whether an entity must have an ARL before it can open an `ENGAGEMENT` ledger.

Example:

```bash
curl -sS "$BASE_URL/entities/arl-gate" \
  -X PATCH \
  -H "Content-Type: application/json" \
  -H "X-ACTOR-ID: $ENTITY_ACTOR_ID" \
  -H "X-ACTOR-TYPE: device" \
  -d '{"require_arl": true}'
```

When `require_arl` is:

- `true`: engagement creation for that entity requires an existing ARL
- `false`: engagement creation can proceed without an ARL

Enforcement applies to the entity actor that owns the setting. The owner actor of the entity cannot change this setting unless it is also the entity actor making the request.

## Public Entity Reads

### `GET /entities/`

Public search endpoint for entity records.

Supported query parameters:

- `entity_type`
- `unique_id`
- `display_name`
- `trust_level`
- `limit`
- `offset`

### `GET /entities/{actor_id}`

Public fetch endpoint for one entity record by actor id.

Public entity reads expose:

- `entity_id`
- `actor_id`
- `actor_type`
- `display_name`
- `entity_type`
- `trust_level`
- `unique_id`
- `metadata`
- `created_at`

Public entity reads do not expose:

- fingerprint
- signing secrets
- owner private state
- internal audit material

## Entity-Scoped Ledgers

### `POST /engagements`

Creates an `ENGAGEMENT` ledger between a primary actor and a counterparty actor.

This is the relationship ledger for non-buyer-supplier flows such as:

- machine inspection
- shipment handoff
- maintenance workflow
- AI compliance review
- system-to-system interaction

Example:

```bash
curl -sS "$BASE_URL/engagements" \
  -X POST \
  -H "Content-Type: application/json" \
  -H "X-ACTOR-ID: $ENTITY_ACTOR_ID" \
  -H "X-ACTOR-TYPE: device" \
  -d '{
    "counterparty_actor_id": "'"$SUPPLIER_ACTOR_ID"'",
    "purpose": "MAINTENANCE"
  }'
```

Response:

```json
{
  "ledger_id": "6f8f0ad9-c5e4-4eb8-b7d4-52b7fc9f6b22",
  "primary_actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "counterparty_actor_id": "7e9c5aef-fb5c-43c2-b6af-b0a2fd9d3f2a",
  "ledger_type": "ENGAGEMENT",
  "purpose": "MAINTENANCE",
  "status": "OPEN"
}
```

## TraceLedger Master Under Entity Continuity

A traceledger master can sit under an ARL or an engagement ledger.

Examples:

- machine inspection sequence under an ARL
- shipment execution sequence under an ARL
- maintenance workflow under an engagement ledger
- AI compliance decision sequence under an engagement ledger or ARL

The master does not create a second root. It remains under the existing entity or relationship boundary.

## Resolver Boundary for Entities

Entities are publicly readable through `/entities/` and `/entities/{actor_id}`.

Entities are not publicly resolvable through `/resolve/iaex:entity:...` at launch.

At launch:

- actor identities use the resolver
- entity records use the entity endpoints
- entity resolver namespaces are reserved and return `501 Not Implemented`
