# FAQ

## Is Genesis X-1 a blockchain?

No. Genesis X-1 does not use a distributed ledger, consensus mechanism, or public chain. It is a protocol infrastructure layer — a verifiable state record for inter-organizational operations, anchored by Ed25519 signatures and SHA-256 hash chains. No tokens. No mining. No anonymous participants.

## Is this a SaaS platform?

No. Systems integrate with Genesis X-1 as protocol participants, not as users of a hosted application. The API surface is a protocol boundary. Genesis X-1 does not manage your business logic, transform your data, or act as middleware. It records operational state and certifies it as independently verifiable.

## What does "independently verifiable" mean?

It means any permissioned party can verify the integrity of a ledger and the attribution of every event without trusting the Genesis X-1 server. Given: the raw event records, the actor's enrolled public key (from `/resolve/`), and the authority public key (from `/.well-known/iaex-authority`) — verification is pure arithmetic. Ed25519 signature check + SHA-256 chain recomputation. No server call required after the initial fetch.

## What happens if Genesis X-1 goes offline?

Events already sealed in the ledger remain verifiable. The proofs are self-contained: event records + public keys. Genesis X-1 as a service can be unavailable. The cryptographic proofs do not expire and do not require the server to remain operational for verification. This is protocol infrastructure, not SaaS dependency.

## What is the difference between an actor and an organization?

An actor is the operational identity unit: it authenticates requests, signs events, and appears in event attribution. An organization is the registered business entity used as the continuity anchor for organization-root ledger flows. Every organization has an actor. Not every actor is an organization.

## What is the difference between a ledger and a TraceLedger master?

A ledger is the constitutional boundary: one relationship or one root continuity scope, with one genesis event, immutable thereafter. A TraceLedger master is an execution scope under an existing ledger — a named grouping of operational events (shipment, inspection, workflow) that does not create a new constitutional root.

## What is the difference between FRL and ARL?

FRL (Facility Root Ledger) is the organization sovereignty root — the continuity anchor for a registered business entity and its internal operations. ARL (Asset Root Ledger) is the asset continuity root — the operational record for a machine, device, shipment, or physical asset onboarded as an entity.

## What is the difference between integrity and truth?

Integrity means the record sequence, hash chain, and event attribution passed mathematical verification. Truth is the factual correctness of the business content asserted in event payloads. Genesis X-1 certifies integrity. It does not evaluate truth. A perfectly verified ledger can contain factually incorrect business data. These are different claims.

## What is non-repudiation?

Once an actor signs and submits an event, they cannot deny having submitted it. Their private key produced the signature. The signature covers the exact event type, ledger target, and payload. The hash chain seals the timestamp. The actor cannot subsequently claim the event was different, did not happen, or occurred at a different time. The proof is mathematical.

## What is the difference between owner and delegate?

An owner controls a scope and can grant or revoke access within it. A delegate operates within a granted scope subject to the permissions of that grant. Delegation grants are actor-signed by the granting party and stored as events — part of the immutable record.

## Can multiple parties verify the same ledger independently?

Yes. This is the core property of the protocol. Each permissioned party fetches the ledger events using their own API key, fetches the relevant actor public keys via the resolver, and runs the verification locally. No party needs to trust another party's verification result. No party needs to contact the other party's systems.

## Does public actor resolution mean public ledger contents?

No. The resolver (`/resolve/{iaex_id}`) returns an actor's identity document and enrolled public keys. It does not expose ledger contents. Ledger events are accessible only to parties with direct ledger access or valid delegation grants.

## What is the difference between `actor_sig` and `actor_sig_payload`?

`actor_sig` is the stored Ed25519 signature. `actor_sig_payload` is the exact subset of the payload that the actor signed — captured before any server-side additions (such as delivery chain metadata). For independent verification, use `actor_sig_payload` when non-null; fall back to `payload` otherwise.

## Does a TraceLedger master create a new constitutional root?

No. A master remains under the ledger that owns it. Closure of a master appends an event to the parent ledger. A new constitutional root requires creating a new ledger via the ledger creation endpoints.
