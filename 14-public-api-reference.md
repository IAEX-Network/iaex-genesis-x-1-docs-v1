# Public API Reference

The public base path is `/v1`.

## Layer A Signing Headers

Endpoints that append actor-signed events require two additional headers when the actor has enrolled signing keys:

| Header | Value |
|--------|-------|
| `X-Signing-Key-ID` | KID of the signing key (e.g. `iaex:actor:<uuid>#key-1`) |
| `X-Actor-Sig` | Base64-encoded Ed25519 signature over the sig digest |

**Sig digest protocol:**
```
msg    = eventType + "\x00" + ledgerID + "\x00" + canonicalJSON(sigPayload)
digest = SHA-256(msg)
sig    = Ed25519.Sign(privateKey, digest)   # sign the 32-byte digest
```

`canonicalJSON` = recursive sorted-key JSON (no whitespace). See `internal/event/sig.go`.

Each endpoint section below lists which event type to sign and what `sigPayload` to use.

---

## Health

### `GET /health`

- Purpose: service health and database reachability summary
- Auth: none

```bash
curl -sS "$BASE_URL/health"
```

```json
{
  "status": "ok",
  "db": [{"name": "prod-india", "status": "ok"}]
}
```

---

## Legal

### `GET /legal/terms`

- Purpose: current terms of service version and full text
- Auth: none

```bash
curl -sS "$BASE_URL/legal/terms"
```

```json
{"version": "1.0", "text": "..."}
```

---

## Developer Access

### `POST /developer/register`

- Purpose: issue sandbox and production developer credentials; optionally enroll first signing key
- Auth: none
- Request fields:
  - `name` (required) — developer or application name
  - `email` (required) — contact email
  - `region` (required) — `"in"` or `"eu"`
  - `public_key` (optional) — base64-encoded Ed25519 public key; if provided, the key is enrolled immediately and returned as `signing_key`

```json
{"name": "erp-prod", "email": "dev@example.com", "region": "in", "public_key": "<base64>"}
```

Response (with `public_key`):

```json
{
  "sandbox": {
    "actor_id": "2db3d84c-85f0-4bc8-9b09-7e8e3d6a1c33",
    "iaex_id": "iaex:actor:2db3d84c-85f0-4bc8-9b09-7e8e3d6a1c33",
    "environment": "sandbox",
    "key_id": "27bb9ef9-d98c-4f8e-95ca-6b9d7f6df7d4",
    "api_key": "iaex_sk_test_...",
    "signing_key": {
      "kid": "iaex:actor:2db3d84c-85f0-4bc8-9b09-7e8e3d6a1c33#key-1",
      "algorithm": "Ed25519"
    }
  },
  "production": {
    "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
    "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
    "environment": "production",
    "region": "in",
    "key_id": "df7d0f4d-6626-4dc9-b56f-f847b2d5096b",
    "api_key": "iaex_sk_live_in_..."
  }
}
```

`signing_key` is only present when `public_key` was supplied in the request.

### `GET /developer/me`

- Purpose: read the authenticated actor and active API keys
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/developer/me" -H "Authorization: Bearer $API_KEY"
```

```json
{
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT",
  "environment": "production",
  "keys": []
}
```

### `POST /developer/keys`

- Purpose: create a new API key for the authenticated actor
- Auth: API key (Bearer)

```json
{"name": "rotated-apr-2026"}
```

```json
{
  "key_id": "4ff39fe9-0d62-4f2c-9ca2-7048ca8cb322",
  "api_key": "iaex_sk_live_in_...",
  "environment": "production",
  "name": "rotated-apr-2026"
}
```

### `DELETE /developer/keys/{key_id}`

- Purpose: revoke an API key owned by the authenticated actor
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/developer/keys/$KEY_ID" -X DELETE -H "Authorization: Bearer $API_KEY"
```

```http
204 No Content
```

---

## Actors

### `POST /actors`

- Purpose: create an actor and optionally enroll its first signing key
- Auth: none

```json
{"role": "PARTICIPANT", "public_key": "<base64>", "algorithm": "Ed25519"}
```

```json
{
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT"
}
```

### `GET /actors/{actor_id}`

- Purpose: read one public actor record
- Auth: none

```bash
curl -sS "$BASE_URL/actors/$ACTOR_ID"
```

```json
{
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT",
  "created_at": "2026-04-21T06:20:00Z"
}
```

### `GET /actors/lookup?iaex_id=...`

- Purpose: resolve a public actor URI into a minimal actor record
- Auth: none

```bash
curl -sS "$BASE_URL/actors/lookup?iaex_id=$BUYER_IAEX_ID"
```

```json
{
  "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT",
  "resolve_url": "/resolve/iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132"
}
```

### `GET /actors/{actor_id}/keys`

- Purpose: list actor public signing keys
- Auth: none

```bash
curl -sS "$BASE_URL/actors/$ACTOR_ID/keys?include_revoked=true"
```

```json
[{"kid": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132#key-1", "algorithm": "Ed25519", "status": "ACTIVE", "preferred_signing": true}]
```

### `POST /actors/{actor_id}/keys`

- Purpose: enroll a new actor signing key
- Auth: API key (Bearer token for the actor_id)

```bash
curl -sS "$BASE_URL/actors/$ACTOR_ID/keys" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"public_key": "<base64>", "algorithm": "Ed25519"}'
```

```json
{"kid": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132#key-2", "algorithm": "Ed25519", "status": "ACTIVE"}
```

### `PATCH /actors/{actor_id}/keys/{kid}/prefer`

- Purpose: mark a key as preferred signing key
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/actors/$ACTOR_ID/keys/$KID/prefer" -X PATCH \
  -H "Authorization: Bearer $API_KEY"
```

```http
204 No Content
```

### `PATCH /actors/{actor_id}/keys/{kid}/revoke`

- Purpose: revoke a key
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/actors/$ACTOR_ID/keys/$KID/revoke" -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"reason": "rotation-complete"}'
```

```http
204 No Content
```

### `GET /network/actors`

- Purpose: list visible network actors
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/network/actors?limit=20" -H "Authorization: Bearer $API_KEY"
```

```json
{
  "count": 1,
  "limit": 20,
  "offset": 0,
  "actors": [{"actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132", "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132", "role": "PARTICIPANT"}]
}
```

---

## Organizations

### `POST /onboarding/organization`

- Purpose: onboard an organization and create its facility root ledger (FRL)
- Auth: API key (Bearer)

```json
{
  "legal_name": "Acme Components Ltd",
  "country": "IN",
  "region": "Tamil Nadu",
  "city": "Chennai",
  "vat_tax_number": "33ABCDE1234F1Z5",
  "primary_email": "supplier.ops@example.com",
  "terms_version": "1.0",
  "declaration_hash": "e2e-test-declaration"
}
```

```json
{
  "request_id": "f5b3cf83-1bb7-4ab6-9efd-2790c510b89a",
  "organization_id": "16b82f79-...",
  "frl_ledger_id": "819a736e-...",
  "status": "APPROVED"
}
```

### `POST /onboarding/supplier`

- Purpose: compatibility alias for organization onboarding (same as `/onboarding/organization`)
- Auth: API key (Bearer)
- Request and response: identical to `POST /onboarding/organization`

### `GET /organizations/{id}`

- Purpose: read one organization by organization_id or actor_id
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/organizations/$ORG_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{
  "organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839",
  "actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "legal_name": "Acme Components",
  "country": "IN",
  "status": "APPROVED"
}
```

### `GET /organizations/?actor_id={actor_id}`

- Purpose: list active organizations owned by one actor
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/organizations/?actor_id=$ACTOR_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{
  "actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "count": 1,
  "organizations": [{"organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839", "legal_name": "Acme Components"}]
}
```

### `PATCH /organizations/{organization_id}/enrich`

- Purpose: update non-identity organization fields
- Auth: API key (Bearer)

```json
{"trade_name": "Acme Precision Components", "postal_code": "600001"}
```

```json
{"organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839", "status": "enriched"}
```

---

## Entities

### `POST /entities/onboard`

- Purpose: create an entity record and entity actor
- Auth: none

```json
{"display_name": "CNC-17", "entity_type": "MACHINE", "unique_id": "CNC-17-2026-0001", "metadata": {"facility_code": "PLANT-01"}}
```

```json
{
  "entity_id": "1f5b2ff6-13e7-4f77-a88c-c3a1a2d1f7d2",
  "actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "actor_type": "device",
  "entity_type": "MACHINE",
  "display_name": "CNC-17",
  "trust_level": "BASIC"
}
```

### `GET /entities/`

- Purpose: public entity search
- Auth: none

```bash
curl -sS "$BASE_URL/entities/?entity_type=MACHINE&display_name=CNC"
```

```json
{"count": 1, "entities": [{"entity_id": "1f5b2ff6-...", "actor_id": "cc6d3f7b-...", "entity_type": "MACHINE", "display_name": "CNC-17"}]}
```

### `GET /entities/{actor_id}`

- Purpose: public entity read
- Auth: none

```bash
curl -sS "$BASE_URL/entities/$ENTITY_ACTOR_ID"
```

```json
{
  "entity_id": "1f5b2ff6-13e7-4f77-a88c-c3a1a2d1f7d2",
  "actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "actor_type": "device",
  "display_name": "CNC-17",
  "entity_type": "MACHINE",
  "trust_level": "BASIC",
  "metadata": {"facility_code": "PLANT-01"}
}
```

### `POST /entities/root-ledger`

- Purpose: create the asset root ledger for the acting entity
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/entities/root-ledger" -X POST \
  -H "Authorization: Bearer $ENTITY_API_KEY"
```

```json
{
  "ledger_id": "5c8dbe20-61e4-4f5f-bf3a-c726e6321f55",
  "entity_actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "ledger_type": "ENTITY_ROOT",
  "status": "OPEN"
}
```

### `GET /entities/root-ledger/{actor_id}`

- Purpose: read one entity root ledger
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/entities/root-ledger/$ENTITY_ACTOR_ID" \
  -H "Authorization: Bearer $ENTITY_API_KEY"
```

```json
{
  "ledger_id": "5c8dbe20-61e4-4f5f-bf3a-c726e6321f55",
  "primary_actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "ledger_type": "ENTITY_ROOT",
  "status": "OPEN"
}
```

### `PATCH /entities/root-ledger/{actor_id}/close`

- Purpose: close an entity root ledger
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/entities/root-ledger/$ENTITY_ACTOR_ID/close" -X PATCH \
  -H "Authorization: Bearer $ENTITY_API_KEY"
```

```json
{"ledger_id": "5c8dbe20-61e4-4f5f-bf3a-c726e6321f55", "status": "CLOSED"}
```

### `PATCH /entities/arl-gate`

- Purpose: require or bypass ARL before engagement creation for the acting entity
- Auth: API key (Bearer)

```json
{"require_arl": true}
```

```json
{"require_arl": true, "updated_at": "2026-04-21T07:00:00Z"}
```

### `POST /engagements`

- Purpose: create an engagement ledger
- Auth: API key (Bearer)

```json
{"counterparty_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132", "purpose": "MAINTENANCE"}
```

```json
{
  "ledger_id": "6f8f0ad9-c5e4-4eb8-b7d4-52b7fc9f6b22",
  "primary_actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "counterparty_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "ledger_type": "ENGAGEMENT",
  "purpose": "MAINTENANCE",
  "status": "OPEN"
}
```

### `GET /engagements/{ledger_id}`

- Purpose: read one engagement ledger
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/engagements/$ENGAGEMENT_ID" -H "Authorization: Bearer $ENTITY_API_KEY"
```

```json
{
  "ledger_id": "6f8f0ad9-c5e4-4eb8-b7d4-52b7fc9f6b22",
  "primary_actor_id": "cc6d3f7b-b2a1-4f20-a7d1-c35d513a4f3e",
  "counterparty_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "ledger_type": "ENGAGEMENT",
  "purpose": "MAINTENANCE",
  "status": "OPEN"
}
```

### `PATCH /engagements/{ledger_id}/close`

- Purpose: close an engagement ledger
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/engagements/$ENGAGEMENT_ID/close" -X PATCH \
  -H "Authorization: Bearer $ENTITY_API_KEY"
```

```json
{"ledger_id": "6f8f0ad9-c5e4-4eb8-b7d4-52b7fc9f6b22", "status": "CLOSED"}
```

---

## Ledgers

### `POST /ledgers`

- Purpose: create an ORDER ledger between buyer and supplier
- Auth: API key (Bearer)
- The caller can be either party; pass the counterparty's `iaex_id`:
  - Caller = **buyer** → send `supplier_iaex_id`
  - Caller = **supplier** → send `buyer_iaex_id`

Buyer creates (passes supplier's iaex_id):

```json
{"supplier_iaex_id": "iaex:actor:c0fc5897-98ea-4ea1-8df6-00d7f9486703"}
```

Supplier creates (passes buyer's iaex_id):

```json
{"buyer_iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132"}
```

```json
{
  "id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "buyer_actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "supplier_actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "status": "OPEN"
}
```

### `GET /ledgers`

- Purpose: list visible ledgers
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/ledgers?ledger_type=ORDER&status=OPEN" -H "Authorization: Bearer $API_KEY"
```

```json
{"actor_id": "2f7a5f0f-...", "count": 1, "ledgers": [{"id": "8eecc02d-...", "ledger_type": "ORDER", "status": "OPEN"}]}
```

### `GET /ledgers/{ledger_id}`

- Purpose: read one ledger boundary
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/ledgers/$LEDGER_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{"id": "8eecc02d-...", "buyer_actor_id": "2f7a5f0f-...", "supplier_actor_id": "c0fc5897-...", "status": "OPEN", "ledger_type": "ORDER"}
```

### `GET /ledgers/{ledger_id}/events`

- Purpose: read one ledger event chain with integrity proof and actor signatures
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/ledgers/$LEDGER_ID/events?limit=100" -H "Authorization: Bearer $API_KEY"
```

```json
{
  "ledger_id": "8eecc02d-...",
  "count": 3,
  "integrity": {"verified": true, "issues": []},
  "events": [
    {
      "id": "7ad7f8d7-...",
      "event_type": "PRODUCTION_STARTED",
      "actor_id": "2f7a5f0f-...",
      "actor_type": "service",
      "payload": {"traceledger_master_uuid": "aa2fa3c9-...", "product": "wheat-batch-A1", "quantity": 500},
      "hash": "0fb1f53d...",
      "caused_by_hash": null,
      "actor_sig": "base64-ed25519-sig",
      "signing_key_id": "iaex:actor:2f7a5f0f-...#key-1",
      "actor_sig_payload": {"traceledger_master_uuid": "aa2fa3c9-...", "product": "wheat-batch-A1", "quantity": 500},
      "created_at": "2026-04-21T06:42:00Z"
    }
  ]
}
```

`actor_sig` and `signing_key_id` are null for authority-signed events (`GENESIS`, `ORDER_OPENED`, `LEDGER_CLOSED`, `ORGANIZATION_REGISTERED`).

`actor_sig_payload` is the exact JSON subset the actor signed; null means the actor signed the full `payload`. Use it (or `payload` as fallback) for independent Ed25519 verification — see Layer A Signing Headers section above.

### `PATCH /ledgers/{ledger_id}/close`

- Purpose: close a ledger; caller signs close intent, server authority-seals `LEDGER_CLOSED`
- Auth: API key (Bearer) + signing headers
- Note: `LEDGER_CLOSED` is authority-signed (server-side). The caller's `X-Actor-Sig` is stored inside the event payload as `requestor_sig` for non-repudiation proof.
- Signing: `eventType=LEDGER_CLOSED`, `ledgerID=<ledger_id>`
  - `sigPayload = {"ledger_id": "...", "ledger_type": "...", "requested_by_actor_id": "...", "status": "CLOSED", "buyer_actor_id": "...", "supplier_actor_id": "...", "buyer_organization_id": "...", "supplier_organization_id": "..."}`

```bash
curl -sS "$BASE_URL/ledgers/$LEDGER_ID/close" -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG"
```

```json
{"ledger_id": "8eecc02d-...", "status": "CLOSED"}
```

---

## TraceLedger Masters

### `POST /traceledger/master`

- Purpose: create a TraceLedger master under a ledger; actor-signed (`TRACELEDGER_MASTER_CREATED`)
- Auth: API key (Bearer) + signing headers
- Signing: `eventType=TRACELEDGER_MASTER_CREATED`, `ledgerID=<ledger_id>`
  - `sigPayload = {"ledger_id": "...", "business_ref": "...", "scope": "..."}`

```bash
curl -sS "$BASE_URL/traceledger/master" -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{"ledger_id": "8eecc02d-...", "business_ref": "PO-2026-0001", "scope": "ORDER"}'
```

```json
{"uuid": "aa2fa3c9-5a97-4f84-86f5-f7c2e98bb7ea", "ledger_id": "8eecc02d-...", "scope": "ORDER", "status": "OPEN"}
```

### `GET /traceledger/master?ledger_id={ledger_id}`

- Purpose: list masters under a ledger
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/traceledger/master?ledger_id=$LEDGER_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{"ledger_id": "8eecc02d-...", "count": 1, "masters": [{"uuid": "aa2fa3c9-...", "scope": "ORDER", "status": "OPEN"}]}
```

### `GET /traceledger/master/{uuid}`

- Purpose: read one master
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{"uuid": "aa2fa3c9-...", "ledger_id": "8eecc02d-...", "business_ref": "PO-2026-0001", "scope": "ORDER", "status": "OPEN"}
```

### `POST /traceledger/master/{uuid}/events`

- Purpose: append a master-scoped business event; actor-signed
- Auth: API key (Bearer) + signing headers
- Signing: `eventType=<event_type>`, `ledgerID=<parent_ledger_id>`
  - Sign the full injected payload (including server-injected `traceledger_master_uuid`, `supplier_actor_id`, etc.)

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{"event_type": "PRODUCTION_STARTED", "payload": {"product": "wheat-batch-A1", "quantity": 500, "traceledger_master_uuid": "aa2fa3c9-...", "supplier_actor_id": "c0fc5897-...", "business_ref": "PO-2026-0001", "scope": "ORDER"}}'
```

```json
{
  "traceledger_master_uuid": "aa2fa3c9-5a97-4f84-86f5-f7c2e98bb7ea",
  "ledger_id": "8eecc02d-d2e8-4185-89ec-79fc00ced9e1",
  "event_type": "PRODUCTION_STARTED",
  "status": "recorded"
}
```

### `GET /traceledger/master/{uuid}/events`

- Purpose: read the event chain for one master
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/events" -H "Authorization: Bearer $API_KEY"
```

```json
{"master_uuid": "aa2fa3c9-...", "ledger_id": "8eecc02d-...", "integrity": {"verified": true, "issues": []}, "events": []}
```

### `PATCH /traceledger/master/{uuid}/close`

- Purpose: close a master; actor-signed (`MASTER_CLOSED`)
- Auth: API key (Bearer) + signing headers
- Signing: `eventType=MASTER_CLOSED`, `ledgerID=<parent_ledger_id>`
  - `sigPayload` for `ORDER` / `FACILITY_ROOT` ledger types:
    ```json
    {
      "traceledger_master_uuid": "<master_uuid>",
      "ledger_id": "<parent_ledger_id>",
      "ledger_type": "ORDER",
      "requested_by_actor_id": "<caller_actor_id>",
      "status": "CLOSED",
      "supplier_actor_id": "<supplier_actor_id>",
      "supplier_organization_id": "<supplier_org_id>",
      "buyer_organization_id": "<buyer_org_id>"
    }
    ```
  - `sigPayload` for `ENGAGEMENT` / `ENTITY_ROOT` ledger types: same base fields plus `entity_id` (if present), omit org/actor cross-party fields

```bash
curl -sS "$BASE_URL/traceledger/master/$MASTER_ID/close" -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG"
```

```json
{"traceledger_master_uuid": "aa2fa3c9-...", "ledger_id": "8eecc02d-...", "status": "CLOSED"}
```

---

## Delegation

### `POST /delegation/organization`

- Purpose: grant organization delegation (Zone B/C) to a staff actor; actor-signed (`SUPPLIER_DELEGATION_GRANTED`)
- Auth: API key (Bearer) + signing headers
- Signing: `eventType=SUPPLIER_DELEGATION_GRANTED`, `ledgerID=<frl_ledger_id>`
  - `sigPayload = {"delegate_actor_id": "...", "organization_id": "...", "permissions": {}, "role": "..."}`

```bash
curl -sS "$BASE_URL/delegation/organization" -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{"organization_id": "a9acfd14-...", "delegate_actor_id": "19fc65fc-...", "role": "OPS", "permissions": {}}'
```

```json
{
  "link_id": "c7363992-f8c1-40fd-81f9-b50d28af0d4d",
  "delegate_actor_id": "19fc65fc-aa0c-4ec9-b0f9-2ae43ef5baf9",
  "role": "OPS",
  "is_active": true,
  "granted_at": "2026-04-21T07:12:00Z"
}
```

### `PATCH /delegation/organization`

- Purpose: revoke organization delegation; actor-signed (`SUPPLIER_DELEGATION_REVOKED`)
- Auth: API key (Bearer) + signing headers

```json
{"organization_id": "a9acfd14-...", "delegate_actor_id": "19fc65fc-..."}
```

```json
{"link_id": "c7363992-...", "delegate_actor_id": "19fc65fc-...", "revoked_at": "2026-04-21T07:13:00Z"}
```

### `GET /delegation/organization?organization_id={organization_id}`

- Purpose: list organization delegation rows
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/delegation/organization?organization_id=$ORG_ID&include_revoked=true" \
  -H "Authorization: Bearer $API_KEY"
```

```json
{
  "organization_id": "a9acfd14-...",
  "count": 1,
  "members": [{"link_id": "c7363992-...", "delegate_actor_id": "19fc65fc-...", "role": "OPS", "is_active": true}]
}
```

### `POST /delegation/ledger`

- Purpose: grant ledger-scoped delegation (Zone D); actor-signed (`LEDGER_DELEGATION_GRANTED`)
- Auth: API key (Bearer) + signing headers

```json
{"ledger_id": "8eecc02d-...", "delegate_actor_id": "19fc65fc-...", "role": "AUDITOR"}
```

```json
{"link_id": "e4b8a1da-...", "delegate_actor_id": "19fc65fc-...", "role": "AUDITOR", "is_active": true}
```

### `PATCH /delegation/ledger`

- Purpose: revoke ledger-scoped delegation; actor-signed (`LEDGER_DELEGATION_REVOKED`)
- Auth: API key (Bearer) + signing headers

```json
{"ledger_id": "8eecc02d-...", "delegate_actor_id": "19fc65fc-..."}
```

```json
{"link_id": "e4b8a1da-...", "delegate_actor_id": "19fc65fc-...", "revoked_at": "2026-04-21T07:15:00Z"}
```

### `GET /delegation/ledger?ledger_id={ledger_id}`

- Purpose: list ledger delegation rows
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/delegation/ledger?ledger_id=$LEDGER_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{"ledger_id": "8eecc02d-...", "count": 1, "members": [{"link_id": "e4b8a1da-...", "delegate_actor_id": "19fc65fc-...", "role": "AUDITOR"}]}
```

### `POST /delegation/master`

- Purpose: grant master-scoped write access (Zone E) to a delegate; actor-signed (`MASTER_DELEGATION_GRANTED`)
- Auth: API key (Bearer) + signing headers
- Signing: `eventType=MASTER_DELEGATION_GRANTED`, `ledgerID=<parent_ledger_id_of_master>`
  - `sigPayload = {"traceledger_master_id": "...", "delegate_actor_id": "...", "role": "...", "permissions": {"access": "write"}}`

```bash
curl -sS "$BASE_URL/delegation/master" -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "X-Signing-Key-ID: $KID" \
  -H "X-Actor-Sig: $SIG" \
  -H "Content-Type: application/json" \
  -d '{"master_id": "aa2fa3c9-...", "delegate_actor_id": "19fc65fc-...", "role": "OPS", "permissions": {"access": "write"}}'
```

```json
{
  "link_id": "3d298b08-f89a-440e-8737-cfcf3b160f52",
  "delegate_actor_id": "19fc65fc-aa0c-4ec9-b0f9-2ae43ef5baf9",
  "role": "OPS",
  "is_active": true,
  "granted_at": "2026-04-21T07:05:00Z"
}
```

### `PATCH /delegation/master`

- Purpose: revoke master delegation; actor-signed (`MASTER_DELEGATION_REVOKED`)
- Auth: API key (Bearer) + signing headers

```json
{"master_id": "aa2fa3c9-...", "delegate_actor_id": "19fc65fc-..."}
```

```json
{"link_id": "3d298b08-...", "delegate_actor_id": "19fc65fc-...", "revoked_at": "2026-04-21T07:10:00Z"}
```

### `GET /delegation/snapshot`

- Purpose: read point-in-time active delegations for one organization, ledger, or master
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/delegation/snapshot?ledger_id=$LEDGER_ID&at=2026-04-21T07:00:00Z" \
  -H "Authorization: Bearer $API_KEY"
```

```json
{"scope": "ledger_id", "scope_id": "8eecc02d-...", "snapshot_at": "2026-04-21T07:00:00Z", "count": 1, "delegations": []}
```

### `GET /delegation/proof`

- Purpose: return full delegation event chain + hash integrity proof for one scope (non-repudiation)
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/delegation/proof?ledger_id=$LEDGER_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{"scope": "ledger_id", "scope_id": "8eecc02d-...", "integrity": {"verified": true, "issues": []}, "events": []}
```

### `GET /audit/access`

- Purpose: return the caller's own access history
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/audit/access?ledger_id=$LEDGER_ID" -H "Authorization: Bearer $API_KEY"
```

```json
{"count": 1, "access_events": [{"method": "GET", "path": "/ledgers/8eecc02d-.../events", "response_status": 200}]}
```

---

## Resolver

### `GET /resolve/{iaex_uri}`

- Purpose: resolve a public actor identity
- Auth: none

```bash
curl -sS "$BASE_URL/resolve/$BUYER_IAEX_ID"
```

```json
{"id": "iaex:actor:2f7a5f0f-...", "actor_id": "2f7a5f0f-...", "keys": []}
```

### `GET /.well-known/iaex-authority`

- Purpose: read the public trust anchor document
- Auth: none
- Phase 1 (development): `public_key` is absent; `phase` is `"1"`
- Phase 2 (production): full key enrolled; `phase` is `"2"`

```bash
curl -sS "$BASE_URL/.well-known/iaex-authority"
```

Phase 2 response:

```json
{
  "id": "iaex:authority",
  "algorithm": "Ed25519",
  "public_key": "<base64>",
  "fingerprint": "<hex>",
  "kid_namespace": "iaex:authority#key-<seq>",
  "phase": "2"
}
```

---

## Webhooks

### `POST /webhooks`

- Purpose: create a webhook subscription
- Auth: API key (Bearer)

```json
{"url": "https://erp.example.com/iaex/webhooks", "event_types": ["PRODUCTION_STARTED"], "ledger_id": "8eecc02d-..."}
```

```json
{"id": "8bb61f70-...", "secret": "m2xZ0FjV6vr3-...", "active": true}
```

### `GET /webhooks`

- Purpose: list owned subscriptions
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/webhooks" -H "Authorization: Bearer $API_KEY"
```

```json
{"count": 1, "subscriptions": [{"id": "8bb61f70-...", "url": "https://erp.example.com/iaex/webhooks", "active": true}]}
```

### `DELETE /webhooks/{id}`

- Purpose: deactivate a webhook subscription
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/webhooks/$WEBHOOK_ID" -X DELETE -H "Authorization: Bearer $API_KEY"
```

```http
204 No Content
```

### `GET /webhooks/{id}/deliveries`

- Purpose: inspect recent delivery attempts
- Auth: API key (Bearer)

```bash
curl -sS "$BASE_URL/webhooks/$WEBHOOK_ID/deliveries" -H "Authorization: Bearer $API_KEY"
```

```json
{"subscription_id": "8bb61f70-...", "count": 1, "deliveries": [{"status": "DELIVERED", "attempt": 0}]}
```
