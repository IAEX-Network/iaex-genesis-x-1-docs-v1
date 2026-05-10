# Resolver

## Public Resolver Surface

### `GET /resolve/{iaex_uri}`

At launch, the resolver supports actor identities only.

Supported:

- `iaex:actor:{uuid}`

Reserved and not implemented at launch:

- `iaex:entity:{uuid}`
- `iaex:ledger:{uuid}`
- `iaex:tlid:{uuid}`

## What `/resolve` Returns

For an actor URI, the resolver returns:

- the actor IAEX id
- the actor id
- the actor role
- public signing key lineage
- current preferred signing key
- creation time

Example:

```bash
curl -sS "$BASE_URL/resolve/$BUYER_IAEX_ID"
```

Example response:

```json
{
  "id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT",
  "status": "ACTIVE",
  "keys": [
    {
      "kid": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132#key-1",
      "algorithm": "Ed25519",
      "public_key": "'"$BUYER_PUBLIC_KEY_B64"'",
      "status": "ACTIVE",
      "preferred_signing": true,
      "valid_from": "2026-04-21T06:25:00Z",
      "created_at": "2026-04-21T06:25:00Z"
    }
  ],
  "preferred_signing": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132#key-1",
  "trust_graph": null,
  "authority_attestations": [],
  "created_at": "2026-04-21T06:20:00Z"
}
```

## What `/resolve` Does Not Return

The public resolver does not return:

- private key material
- raw event signatures
- private authority operations
- internal storage metadata
- organization ownership details
- ledger contents

## What Third Parties Can Verify

Third parties can verify:

- the public actor identity URI
- the actor’s key lineage
- which key is currently preferred for signing
- whether a key was later revoked

Third parties cannot infer:

- that every event for that actor is publicly replayable
- that the resolver is a signed attestation of business truth
- that non-actor namespaces are public at launch

## Caching

Resolver responses are cacheable at a high level.

Launch cache behavior:

- actor resolver responses include `ETag`
- actor resolver responses use `Cache-Control: public, max-age=60`
- `If-None-Match` can be used for conditional reads

## Public Trust Anchor

### `GET /.well-known/iaex-authority`

This returns the public trust anchor document for the IAEX authority identity.

Use it to obtain the published authority public key.

Example:

```bash
curl -sS "$BASE_URL/.well-known/iaex-authority"
```

Example response:

```json
{
  "id": "iaex:authority",
  "algorithm": "Ed25519",
  "public_key": "'"$IAEX_AUTHORITY_PUBLIC_KEY_B64"'",
  "kid_namespace": "iaex:authority#key-<seq>",
  "phase": "1"
}
```

## What a Public Identity Document Is For

A public identity document exists to let external systems:

- resolve an actor URI into a concrete actor id
- read current and historical public signing keys
- validate actor key rotation state

## What Not to Infer From Resolver Output

Do not infer:

- transaction consensus
- public visibility of private ledger data
- private platform trust decisions
- legal certification of business claims
