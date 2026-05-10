# Product Overview

Genesis X-1 is the verifiable state layer for inter-organizational trade networks.

It defines a protocol for actor identity, ledger boundary, event attribution, and tamper-evident continuity across organizational boundaries. Enterprise systems connect as protocol participants — not as users of a hosted platform.

---

## Protocol Position

Genesis X-1 occupies the infrastructure layer beneath application logic. Its role is analogous to what TCP/IP provides for packet routing or what SWIFT provides for financial messaging: a canonical, independently verifiable record of what happened, who authorized it, and in what order — anchored by cryptographic proof rather than institutional trust.

Protocol participants include:

- Manufacturing and logistics organizations
- Financial institutions, banks, and trade financiers
- Auditors and certification bodies
- ERP platforms and workflow orchestration systems
- IoT device gateways
- Independent verification tools and audit systems

No participant trusts another's internal system. All trust is anchored in signed, immutable, hash-chained events that any permissioned party can independently verify without access to the originating system.

---

## What Genesis X-1 Is Not

Genesis X-1 is not a blockchain. It does not require consensus across unknown participants. It does not use distributed ledger mechanisms.

Genesis X-1 is not a compliance SaaS. It does not evaluate the business content, commercial validity, or legal effect of events appended by participants.

Genesis X-1 is not a middleware platform. It does not transform, translate, or route data between enterprise systems.

Genesis X-1 is protocol infrastructure. It records operational state across organizational boundaries and certifies that state as independently verifiable by any permissioned party — now or at any future point.

---

## Protocol Invariants

The public contract is organized around four invariants:

**Actor identity.** Every participant is an actor with a stable UUID, a network-addressable `iaex_id`, and an enrolled Ed25519 signing key. Every event is attributed to a specific actor. Every actor identity is publicly resolvable at `/resolve/{iaex_id}` without authentication.

**Ledger boundary.** A ledger is the constitutional record for one relationship or one root continuity scope. It cannot be created retroactively, merged, or deleted. It is immutable from the moment of genesis. The genesis event is authority-signed and establishes the ledger's root hash.

**Execution scope.** A TraceLedger master is an execution scope under an existing ledger. It groups operational events — shipments, inspections, disbursements — under a named business reference without creating a new constitutional boundary. Scope creation and closure are actor-signed.

**Event continuity.** Events form a hash chain. Each event seals the prior event's hash into its own digest. Any modification to any event breaks all subsequent hashes. Chain integrity is computed server-side and returned with every event read.

---

## Certification Scope

The protocol certifies:

- **Record integrity** — events are unmodified since append
- **Event attribution** — every event is bound to a specific actor identity and enrolled signing key
- **Append ordering** — events are totally ordered within a ledger by sequence number and timestamp
- **Tamper detection** — hash chain verification is computed and returned on every read

The protocol does not certify:

- Factual truth of business content supplied in event payloads
- Legal validity of off-platform representations or contractual claims
- Commercial correctness of party-supplied data

---

## Integration Model

Systems integrate at the protocol level:

1. Register developer credentials and obtain a regional API key.
2. Register actors representing operational participants.
3. Enroll Ed25519 signing keys at registration or through the key management surface.
4. Onboard organizations or entities to establish root ledger boundaries.
5. Create ledgers for cross-party relationships.
6. Create TraceLedger masters for named execution scopes.
7. Append actor-signed events.
8. Read events, verify integrity, resolve actor identity, and consume webhook deliveries.

The API surface is versioned under `/v1`. All authenticated operations require a bearer API key issued at developer registration. All event append operations that assert actor authorization require a valid Ed25519 actor signature in the `X-Actor-Sig` request header.
