# Core Concepts

Genesis X-1 defines a precise protocol vocabulary. These terms carry exact meanings within the protocol. Implementations must use them as specified.

---

## Actor

The canonical identity unit. An actor holds:

- A stable UUID (`actor_id`) — the internal protocol identifier, permanent and immutable
- A network-resolvable address (`iaex_id`) — the cross-network identifier, formatted as `iaex:actor:<uuid>`
- One or more enrolled Ed25519 signing keys, each identified by a unique key ID (`kid`)

Every authenticated request is scoped to an actor's API key. Every signed event carries the `actor_id` and `kid` of the signing actor. Actor identity is publicly resolvable at `/resolve/{iaex_id}` without authentication.

---

## Organization

A registered business entity used as the continuity anchor for organization-root ledger flows. An organization is created through the onboarding protocol and bound at creation to a Facility Root Ledger (FRL). Organizations are the primary participants in ORDER and FACILITY_ROOT ledger relationships.

---

## Entity

A machine, asset, or device with an operational identity on the network. Entities are onboarded through the entity flow and receive an Asset Root Ledger (ARL). Entity operational records are scoped within engagement ledgers under the ARL.

---

## Ledger

The constitutional boundary record for one relationship or one root continuity scope. A ledger:

- Is created once, immutable thereafter
- Records exactly two direct parties (buyer and supplier for ORDER ledgers; the owning organization for root ledgers)
- Is sealed at genesis by an authority-signed event establishing the root hash
- Can only be closed — never modified or deleted

Ledger boundary determines access scope. Direct parties can read all events within the ledger. Access by non-party actors requires explicit delegation.

---

## TraceLedger Master

An execution scope within an existing ledger. It groups related operational events — shipments, inspections, financing — under a named business reference (such as a purchase order number or shipment ID) without creating a new constitutional boundary.

Masters are created by a direct ledger party. Write access is granted to delegates via `POST /delegation/master`. Each access grant is actor-signed by the granting party and stored as an event.

---

## Event

An immutable, append-only record within a ledger. Properties:

- **Sequenced** — monotonically increasing sequence number within the ledger
- **Hash-chained** — each event seals the prior event's hash into its own digest (Layer B)
- **Attributed** — carries the `actor_id` and `kid` of the appending actor
- **Optionally actor-signed** — events submitted with `X-Actor-Sig` carry a stored Ed25519 signature and the exact signed payload (`actor_sig_payload`)

Events cannot be modified, deleted, or reordered after append.

---

## Genesis Event

The first event in every ledger. It establishes the ledger boundary, records the direct parties, and anchors the hash chain at position zero. Genesis events are authority-signed by the Genesis X-1 authority key, not by the creating actor.

---

## Delegation

A signed grant that allows one actor to act within a defined scope owned by another actor or ledger party. Delegations are:

- Scoped to organization, ledger, or TraceLedger master level
- Actor-signed by the granting party at the time of grant
- Stored as events within the relevant ledger, making them part of the immutable record
- Auditable through the delegation snapshot and proof endpoints

---

## API Key

A bearer credential issued at developer registration. It identifies the calling actor, determines the database region (sandbox, India, EU), and is transmitted as `Authorization: Bearer <key>`. API keys are SHA-256 hashed at rest. The raw key is shown once at issuance.

---

## Actor Signature — Layer A

An Ed25519 signature from the acting actor over the canonical event intent. Computed as:

```
sigPayload = business fields the actor asserts (canonical subset)
msg        = eventType + "\x00" + ledgerID + "\x00" + canonicalJSON(sigPayload)
digest     = SHA-256(msg)
sig        = Ed25519.Sign(actorPrivKey, digest)
```

The signature (`actor_sig`) and the exact signed payload (`actor_sig_payload`) are stored in the event row and returned in all event reads. Any party with the actor's public key can independently verify the signature without server involvement.

---

## Hash Chain — Layer B

The tamper-evident continuity structure of a ledger. Each event seals the prior event's hash:

```
hash(n) = SHA-256(
  hash(n-1) + event_type + ledger_id + actor_id + timestamp + canonicalJSON(payload)
)
```

Any modification to any event in the chain invalidates all subsequent hashes. The hash chain is verified server-side on every event read. The result is returned in `integrity.verified` on every response.

---

## Resolver

The public endpoint returning an actor's identity document and signing key lineage. Resolution requires no authentication. It is the trust anchor for cross-party actor verification — a verifier with no prior interaction with an actor can fetch their public key and verify their event signatures independently.

---

## Webhook

A signed HTTP POST delivery of event data to a subscriber-registered endpoint. Webhook payloads carry an `IAEX-Signature` header signed with the authority key. Subscribers verify delivery authenticity against the published authority public key at `/.well-known/iaex-authority`.

---

## Correction Lineage

A causal link from a later event to an earlier event through the `caused_by_hash` field. Used when a later event corrects, supersedes, or contextualizes a prior record. The causal link is cryptographically bound — the referenced hash must exist in the ledger chain.
