# Network Model

A Genesis X-1 deployment is a multi-party protocol network. Each participant class operates an independent organizational boundary. Trust between parties is not assumed — it is established through signed delegation grants, enforced at the ledger boundary, and independently verifiable by any permissioned party.

The reference topology models a five-party agricultural trade network. The same structural principles apply to any inter-organizational domain: manufacturing, logistics, financial services, or supply chain finance.

---

## Party Classes

| Party | Role | Ledger Position |
|-------|------|-----------------|
| **Supplier** | Produces and ships goods | Owns FRL; direct party to ORDER ledger |
| **Buyer** | Receives and pays for goods | Owns FRL; direct party to ORDER ledger |
| **Transport** | Carries goods between parties | Delegated write on ORDER master (Zone E) |
| **Auditor** | Certifies compliance and inspection | Delegated write on ORDER master (Zone E) |
| **Bank / Financier** | Funds transactions; issues trade loans | Delegated write on ORDER master (Zone E) |

Each party operates its own Facility Root Ledger (FRL) — a private continuity boundary for internal operations — and participates in shared ledgers only through explicit, signed access grants.

---

## Ledger Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                        GENESIS X-1 NETWORK                           │
│                                                                      │
│  ┌─────────────────────┐        ┌─────────────────────┐             │
│  │   FarmFresh Ltd      │        │   RetailCo Ltd       │             │
│  │   (Supplier)         │        │   (Buyer)            │             │
│  │                      │        │                      │             │
│  │  ┌───────────────┐   │        │  ┌───────────────┐   │             │
│  │  │  FRL Ledger   │   │        │  │  FRL Ledger   │   │             │
│  │  │  (FACILITY    │   │        │  │  (FACILITY    │   │             │
│  │  │   ROOT)       │   │        │  │   ROOT)       │   │             │
│  │  │               │   │        │  │               │   │             │
│  │  │ ┌──────────┐  │   │        │  │ ┌──────────┐  │   │             │
│  │  │ │ Master   │  │   │        │  │ │ Master   │  │   │             │
│  │  │ │ STOCK    │  │   │        │  │ │ STOCK    │  │   │             │
│  │  │ └──────────┘  │   │        │  │ └──────────┘  │   │             │
│  │  └───────────────┘   │        │  └───────────────┘   │             │
│  └────────────┬─────────┘        └──────────┬──────────┘             │
│               │                             │                        │
│               └──────────┬──────────────────┘                        │
│                          │                                           │
│               ┌──────────▼──────────────────┐                        │
│               │       ORDER Ledger           │                        │
│               │  (buyer ↔ supplier corridor) │                        │
│               │                             │                        │
│               │  ┌─────────────────────────┐│                        │
│               │  │  ORDER Master (shipment) ││                        │
│               │  │                         ││                        │
│               │  │  Events (time-ordered):  ││                        │
│               │  │  • PICKUP_SCHEDULED      ││                        │
│               │  │  • GOODS_PICKED_UP       ││ ← SwiftLogistics       │
│               │  │  • IN_TRANSIT ×n         ││ ← SwiftLogistics IoT   │
│               │  │  • DELIVERED             ││ ← SwiftLogistics       │
│               │  │  • FIELD_INSPECTION_DONE ││ ← TrustAudit           │
│               │  │  • AUDIT_REPORT_ISSUED   ││ ← TrustAudit           │
│               │  │  • LOAN_APPLICATION_*    ││ ← NationalBank         │
│               │  │  • LOAN_DISBURSED        ││ ← NationalBank         │
│               │  │  • GOODS_ACCEPTED        ││ ← RetailCo             │
│               │  │  • BUYER_QC_PASSED       ││ ← RetailCo             │
│               │  └─────────────────────────┘│                        │
│               └─────────────────────────────┘                        │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  SwiftLogistics  │  │  TrustAudit Inc  │  │  NationalBank    │   │
│  │  (Transport)     │  │  (Auditor)       │  │  (Bank)          │   │
│  │                  │  │                  │  │                  │   │
│  │  Own FRL         │  │  Own FRL         │  │  Own FRL         │   │
│  │  Delegated write │  │  Delegated write │  │  Delegated write │   │
│  │  on ORDER master │  │  on ORDER master │  │  on ORDER master │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Delegation Zones

Access on a Genesis X-1 network is structured into five delegation zones. Each zone corresponds to a distinct scope boundary. Grants flow outward from the owning actor; access does not flow inward without explicit grant.

```
Zone A  ─  Network level     Actor registration; Ed25519 key enrollment
Zone B  ─  Organization      Org-level delegation  POST /delegation/organization
Zone C  ─  FRL               Facility Root Ledger — internal org scope
Zone D  ─  Ledger            Cross-party ledger access  POST /delegation/ledger
Zone E  ─  Master            TraceLedger master write   POST /delegation/master
```

Cross-party access flows exclusively through Zone E. The supplier creates the ORDER master and grants write access to each external delegate — transport, auditor, bank — by actor ID. Each grant is actor-signed, stored as an event in the ORDER ledger, and independently auditable.

No implicit cross-party access exists. A bank actor cannot read the supplier's FRL. The supplier cannot read the bank's internal FRL. Only events on the shared ORDER ledger are accessible to all granted parties.

---

## Access Scope Rules

| Party | Own FRL | ORDER Ledger | ORDER Master Write |
|-------|---------|--------------|--------------------|
| Supplier org | ✓ direct | ✓ direct party | ✓ creator |
| Buyer org | ✓ direct | ✓ direct party | delegated by supplier |
| Transport actors | ✓ own FRL only | via master access | ✓ Zone E grant |
| Auditor actors | ✓ own FRL only | via master access | ✓ Zone E grant |
| Bank actors | ✓ own FRL only | via master access | ✓ Zone E grant |

---

## Signing Model — Layer A

Every actor-authored event is signed before transmission. The signature is computed over the canonical event intent — not over the full stored payload — ensuring that server-side additions (such as delivery chain metadata) do not invalidate the actor's signature.

```
sigPayload  = business fields the actor asserts
msg         = eventType + "\x00" + ledgerID + "\x00" + canonicalJSON(sigPayload)
digest      = SHA-256(msg)
X-Actor-Sig = base64(Ed25519.Sign(actorPrivKey, digest))
```

The server verifies the signature before appending. The signature and `actor_sig_payload` are stored in the event row and returned in all event reads, enabling independent verification by any permissioned party at any point in time.

---

## Hash Chain — Layer B

Each appended event seals the prior event's hash into its own digest:

```
hash(event_n) = SHA-256(
  hash(event_{n-1}) +
  event_type       +
  ledger_id        +
  actor_id         +
  timestamp        +
  canonicalJSON(payload)
)
```

This creates a tamper-evident continuity proof. Any modification to any event in the chain breaks all subsequent hashes. `integrity.verified` in every event read response reflects the result of this server-side check.

---

## Five-Party Flow Reference

The following sequence describes a complete ORDER lifecycle across all five party classes:

```
Phase 1   Register       5 orgs, ~30 actors; Ed25519 keys enrolled at registration
Phase 2   Onboard        Each org establishes FRL via POST /onboarding/organization
Phase 3   Delegate       Each org grants staff access via POST /delegation/organization
Phase 4   FRL Masters    Each org creates a TraceLedger master on their FRL
Phase 5   Master access  Each org grants staff write on FRL master
Phase 6   Sup events     Supplier records production, QC, IoT, and shipment on FRL
Phase 7   ORDER ledger   Supplier creates ORDER ledger with buyer as counterparty
Phase 8   ORDER master   Supplier creates ORDER master; grants 13 cross-party delegates
Phase 9   Order events   5 parties append signed events on ORDER master
Phase 10  Close          Supplier closes ORDER master + ledger (both actor-signed)
Phase 11  Buy events     Buyer records acceptance, QC, warehouse receipt on FRL
Phase 12  Verify 10A     SigLog self-verification: client re-verifies all signatures locally
Phase 13  Verify 10B     Cross-party: each party fetches via own API key + Ed25519 verify
```

A complete runnable implementation of this flow is provided in `cmd/e2e_full`. See [Full Network Sandbox](/e2e-full-sandbox) for execution instructions.
