# Organization Onboarding

An organization is the registered business entity on the Genesis X-1 network. Every organization has an actor — the identity unit that authenticates requests and signs events. The organization provides the business continuity anchor. The actor provides the cryptographic identity.

Onboarding an organization creates two things simultaneously:

- an `organization_id` — the stable network identifier for the business entity
- an `frl_ledger_id` — the Facility Root Ledger, the immutable sovereignty root for that organization

Both are permanent. Neither can be deleted.

## Prerequisites

Before onboarding an organization, the actor must exist and have an enrolled signing key.

## Step 1 — Create an Actor

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
  "actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "iaex_id": "iaex:actor:c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "role": "PARTICIPANT",
  "key": {
    "kid": "iaex:actor:c0fc5897-98ea-4ea1-8df6-00d7f9486703#key-1",
    "algorithm": "Ed25519",
    "status": "ACTIVE",
    "preferred_signing": true
  }
}
```

Save the `actor_id`.

## Step 2 — Retrieve an API Key

API keys are issued during network onboarding. Each actor has exactly one API key. The API key authenticates all requests made by that actor.

Set it as:

```bash
export API_KEY="iaex_sk_live_..."
```

## Step 3 — Onboard the Organization

```bash
curl -sS "$BASE_URL/onboarding/organization" \
  -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "legal_name": "Acme Components Private Limited",
    "country": "IN",
    "region": "TN",
    "city": "Chennai",
    "registered_address": "45 Foundry Road, Chennai",
    "vat_tax_number": "33ABCDE1234F1Z5",
    "primary_email": "ops@acmecomponents.com",
    "terms_version": "2026-04-21",
    "declaration_hash": "acme-onboarding-declaration-v1"
  }'
```

Response:

```json
{
  "request_id": "f5b3cf83-1bb7-4ab6-9efd-2790c510b89a",
  "organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839",
  "frl_ledger_id": "819a736e-3e8b-4e4a-b22f-d0e6f1a2c3b4",
  "status": "APPROVED"
}
```

Save `organization_id` and `frl_ledger_id`.

### What Gets Written

Organization onboarding appends two authority-signed events to the FRL:

- `GENESIS` — opens the Facility Root Ledger
- `ORGANIZATION_REGISTERED` — anchors the organization identity

These events are signed by the IAEX authority (notarial seal). No `X-Actor-Sig` header is required from the caller.

## Step 4 — Read the Organization

```bash
curl -sS "$BASE_URL/organizations/$ORG_ID" \
  -H "Authorization: Bearer $API_KEY"
```

Response:

```json
{
  "organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839",
  "actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "legal_name": "Acme Components Private Limited",
  "country": "IN",
  "status": "APPROVED"
}
```

## Step 5 — Lookup by Actor ID

```bash
curl -sS "$BASE_URL/organizations/?actor_id=$ACTOR_ID" \
  -H "Authorization: Bearer $API_KEY"
```

Response:

```json
{
  "actor_id": "c0fc5897-98ea-4ea1-8df6-00d7f9486703",
  "count": 1,
  "organizations": [
    {
      "organization_id": "a9acfd14-cf2c-4171-b399-4af8e3b5f839",
      "legal_name": "Acme Components Private Limited"
    }
  ]
}
```

## Step 6 — Optional Enrichment

Non-identity fields can be updated after onboarding:

```bash
curl -sS "$BASE_URL/organizations/$ORG_ID/enrich" \
  -X PATCH \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "trade_name": "Acme Precision Components",
    "postal_code": "600001"
  }'
```

Identity fields (`legal_name`, `vat_tax_number`, `country`) are immutable after onboarding.

## Two-Organization Network

Most Genesis X-1 flows require two organizations. The API uses `buyer_*` and `supplier_*` as positional labels — these are not fixed business roles. Any organization type (manufacturer, logistics, bank, regulator) can occupy either side.

Onboard both organizations independently, each under their own actor and API key:

```bash
# Organization A
curl -sS "$BASE_URL/onboarding/organization" \
  -X POST \
  -H "Authorization: Bearer $ORG_A_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "legal_name": "Northwind Buyer Operations Private Limited",
    "country": "IN",
    "region": "KA",
    "city": "Bengaluru",
    "vat_tax_number": "29ABCDE1234F1Z5",
    "primary_email": "ops@northwind.com",
    "terms_version": "2026-04-21",
    "declaration_hash": "northwind-declaration-v1"
  }'

# Organization B
curl -sS "$BASE_URL/onboarding/organization" \
  -X POST \
  -H "Authorization: Bearer $ORG_B_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "legal_name": "Acme Components Private Limited",
    "country": "IN",
    "region": "TN",
    "city": "Chennai",
    "vat_tax_number": "33ABCDE1234F1Z5",
    "primary_email": "ops@acmecomponents.com",
    "terms_version": "2026-04-21",
    "declaration_hash": "acme-declaration-v1"
  }'
```

Once both are onboarded, open an order ledger referencing both `organization_id` and `actor_id` values. See [Quickstart](/quickstart) for the full flow.

## Compatibility Alias

`POST /onboarding/supplier` is an alias for `POST /onboarding/organization`. Both endpoints are identical. Use either.

## Organization vs Actor

| | Actor | Organization |
|---|---|---|
| What it is | Cryptographic identity unit | Registered business entity |
| Created by | `POST /actors` | `POST /onboarding/organization` |
| Used for | Authentication, signing, attribution | Business relationships, FRL, delegation |
| Permanent | Yes | Yes |
| Has signing key | Yes | No (uses actor's key) |

Every organization has exactly one actor. The actor is the signing identity for all operations performed on behalf of that organization.

## Delegation After Onboarding

After onboarding, the organization owner can grant delegation to additional actors — operations staff, ERP systems, automated agents. See [Delegation and Access Control](/delegation-and-access-control) for grant and revocation flows.
