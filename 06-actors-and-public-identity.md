# Actors and Public Identity

An actor is the unit of identity in the Genesis X-1 protocol. Not a user account. Not a session. An actor is a permanent, addressable identity with a cryptographic signing key — the entity that authorizes events, holds access grants, and appears in the immutable record of every action taken under its credentials.

Actor identity is publicly resolvable. Any party on any network — counterparty, auditor, regulator, verification tool — can resolve an actor's public key without authentication. This is the trust anchor for independent verification. No prior relationship required. No server trust required after the initial fetch.

## Public Actor Identity URI

The public actor identity URI is:

```text
iaex:actor:{actor_uuid}
```

Example:

```text
iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132
```

## What a Public Actor Identity Proves

A resolved actor identity proves:

- the actor exists in the public network surface
- the actor has a stable protocol identity
- the actor has a public signing key lineage
- the current preferred signing key can be identified

It does not prove:

- business ownership of an organization
- truth of business claims
- private commercial relationships

## Actor Creation

### `POST /actors`

Creates a public actor identity.

Example:

```bash
curl -sS "$BASE_URL/actors" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "role": "PARTICIPANT",
    "public_key": "'"$PUBLIC_KEY_B64"'",
    "algorithm": "Ed25519"
  }'
```

Response:

```json
{
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT",
  "key": {
    "kid": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132#key-1",
    "algorithm": "Ed25519",
    "public_key": "'"$PUBLIC_KEY_B64"'",
    "status": "ACTIVE",
    "preferred_signing": true
  }
}
```

## Public Actor Read

### `GET /actors/{actor_id}`

Returns:

- `actor_id`
- `iaex_id`
- `role`
- `created_at`

Example:

```bash
curl -sS "$BASE_URL/actors/$ACTOR_ID"
```

## Actor Lookup by IAEX URI

### `GET /actors/lookup?iaex_id=iaex:actor:{uuid}`

Returns a minimal public actor record and a `resolve_url`.

Use this when the caller already has an IAEX actor URI and needs a fast existence check before calling the resolver.

## Actor Key Management

### `GET /actors/{actor_id}/keys`

Returns active keys by default.

Use `include_revoked=true` to include revoked keys.

### `POST /actors/{actor_id}/keys`

Enrolls a new signing key for that actor.

### `PATCH /actors/{actor_id}/keys/{kid}/prefer`

Marks one active key as the preferred signing key.

### `PATCH /actors/{actor_id}/keys/{kid}/revoke`

Revokes a key. Revocation is permanent. Historical visibility remains available through the resolver and key list APIs.

## Key Rotation

Key rotation means:

- a new public key is enrolled for the actor
- the previous key can remain active during overlap
- one active key can be marked preferred
- revoked keys remain visible as historical lineage

Key rotation does not change:

- the actor id
- the actor IAEX URI
- prior event attribution

## Third-Party Actor Verification

A third party verifies an actor by:

1. Reading the actor identity document through `/resolve/iaex:actor:{uuid}`.
2. Checking the active and revoked key lineage.
3. Checking the preferred signing key if a current signer is required.
4. Checking the authority trust anchor through `/.well-known/iaex-authority`.

## Network Discovery

### `GET /network/actors`

Lists actors visible to authenticated network participants.

This endpoint requires authentication.

## Public and Private Boundary

Public actor identity responses include:

- actor identifiers
- actor creation timestamp
- public signing keys
- key status
- preferred signing key

Public actor identity responses do not include:

- private keys
- raw event signatures
- internal authority identities
- internal trust graph state
- private audit fields
