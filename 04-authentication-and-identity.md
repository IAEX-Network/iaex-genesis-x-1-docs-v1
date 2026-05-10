# Authentication and Identity

Every request to Genesis X-1 is scoped to a specific actor. There is no anonymous access to authenticated surfaces. There is no shared credential. Every operation is attributed to the actor whose credential authorized it.

Authentication is not a gate. It is the identity anchor for every event, every delegation, and every access decision recorded in the protocol.

## Identity Paths

### 1. API key authentication

Use `Authorization: Bearer <iaex_sk_...>` for developer and system access.

This is the primary production authentication model.

`POST /developer/register` issues:

- one sandbox actor and API key
- one production actor and API key

### 2. Explicit actor identity compatibility path

If a request does not carry an API key, the public API accepts:

- `X-ACTOR-ID`
- `X-ACTOR-TYPE`

This path exists for explicit actor-native operations where the request must act as a specific actor.

## Request Identity Resolution

Identity resolution follows a strict order:

1. If `Authorization: Bearer` contains a valid IAEX API key, that actor is the caller.
2. Otherwise the server reads `X-ACTOR-ID` and `X-ACTOR-TYPE`.

If both are present, the bearer key takes precedence.

## Required Headers

### API key requests

- `Authorization: Bearer <iaex_sk_...>`
- `Content-Type: application/json` for JSON request bodies

Optional:

- `X-Actor-Type` to override the default `service` actor type for API-key-authenticated requests
- `X-Request-Id` for client correlation
- `Idempotency-Key` for `POST`, `PUT`, and `PATCH`
- `X-Delivery-Chain` when forwarding a caller lineage

### Explicit actor requests

- `X-ACTOR-ID: <actor_uuid>`
- `X-ACTOR-TYPE: human|service|external|device`

## Actor Signature Model

Actor signatures authorize caller-signed event writes.

The public header is:

```http
X-Actor-Sig: <base64-ed25519-signature>
```

Use `X-Actor-Sig` on caller-defined event append requests such as `POST /traceledger/master/{uuid}/events`.

Do not assume every mutating route requires a caller-supplied signature header. Some public routes generate their own platform event payloads as part of the route contract.

The actor signature proves:

- which actor authorized the event append
- the exact event type
- the exact ledger target
- the exact canonical payload submitted by the actor before delivery-chain injection

The actor signature does not prove:

- truth of the business content
- downstream business approval by another party
- chain continuity on its own

## What the Actor Signature Covers

The signature input is:

```text
SHA-256(
  event_type + "\x00" + ledger_id + "\x00" + canonical_json(payload_without_delivery_chain)
)
```

The digest is then signed with Ed25519.

For `POST /traceledger/master/{uuid}/events`, `ledger_id` in the signature input is the parent ledger id of that master, not the master uuid.

Canonical JSON means deterministic JSON with recursively sorted object keys.

## Minimal Signature Example

The example below signs an event request using a base64-encoded 32-byte Ed25519 seed in `ED25519_SEED_B64`.

```bash
export EVENT_TYPE=AI_RESPONSE
export LEDGER_ID="$LEDGER_ID"
export PAYLOAD_JSON='{"model":"genesis-x1-audit","summary":"Inspection complete","traceledger_master_uuid":"'"$MASTER_ID"'"}'
```

```js
import { createHash, createPrivateKey, sign } from "node:crypto";

const canonicalize = (value) => {
  if (Array.isArray(value)) {
    return `[${value.map(canonicalize).join(",")}]`;
  }
  if (value && typeof value === "object") {
    return `{${Object.keys(value).sort().map((k) => `${JSON.stringify(k)}:${canonicalize(value[k])}`).join(",")}}`;
  }
  return JSON.stringify(value);
};

const pkcs8Prefix = Buffer.from("302e020100300506032b657004220420", "hex");
const seed = Buffer.from(process.env.ED25519_SEED_B64, "base64");
const privateKey = createPrivateKey({
  key: Buffer.concat([pkcs8Prefix, seed]),
  format: "der",
  type: "pkcs8"
});

const payload = canonicalize(JSON.parse(process.env.PAYLOAD_JSON));
const message = Buffer.concat([
  Buffer.from(process.env.EVENT_TYPE),
  Buffer.from([0]),
  Buffer.from(process.env.LEDGER_ID),
  Buffer.from([0]),
  Buffer.from(payload)
]);

const digest = createHash("sha256").update(message).digest();
const signature = sign(null, digest, privateKey);
process.stdout.write(signature.toString("base64"));
```

## What Developers Must Store

Store:

- API keys
- actor private signing keys
- webhook secrets returned at subscription creation
- actor ids and organization or entity ids needed by your integration

Keep secret:

- API keys
- actor private signing keys
- webhook secrets

Do not send private keys to IAEX.

## Public Key Enrollment

Actor public keys are enrolled through the actor key endpoints.

Launch algorithm support:

- `Ed25519`

The enrolled value is the raw 32-byte Ed25519 public key in base64.

## Auth Failure Behavior

Authentication failures return a JSON error envelope with `401` or `404` depending on the failure mode.

Examples:

- invalid or revoked API key: `401`
- missing identity headers when no API key is present: `401`
- invalid actor type: `401`
- non-public authority identity: `404`

## Minimal API Key Example

```bash
curl -sS "$BASE_URL/developer/me" \
  -H "Authorization: Bearer $API_KEY"
```

Example response:

```json
{
  "actor_id": "2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "iaex_id": "iaex:actor:2f7a5f0f-8cc1-4f20-95b8-a2f5488d6132",
  "role": "PARTICIPANT",
  "environment": "production",
  "created_at": "2026-04-21T06:20:00Z",
  "keys": [
    {
      "key_id": "df7d0f4d-6626-4dc9-b56f-f847b2d5096b",
      "key_prefix": "iaex_sk_live_in_",
      "environment": "production",
      "name": "default",
      "created_at": "2026-04-21T06:20:00Z"
    }
  ]
}
```
