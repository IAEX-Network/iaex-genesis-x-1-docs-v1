# Full Network Sandbox

The Genesis X-1 sandbox simulates a complete 5-party agricultural supply-chain trade on the live production API — from actor registration through to cross-party Ed25519 verification.

Run it to validate that the network is working correctly, or as a live demonstration of the full protocol.

---

## Option 1 — Hosted (no install, no download)

Trigger the full verification from the live server — output streams in real time:

```bash
curl -N "https://api.iaexnetwork.com/sandbox/run"
```

Or open directly in any browser:

```
https://api.iaexnetwork.com/sandbox/run
```

No API key required. Multiple concurrent runs are supported — each run generates a unique ID so there are no conflicts.

---

## Option 2 — Download Binary (no Go needed)

Download a pre-built binary for your platform:

**[→ Download Latest Release](https://github.com/IAEX-Network/iaex-genesis-x-1-v1-sandbox/releases)**

| Platform | File |
|----------|------|
| Windows | `e2e_full_windows_amd64.exe` |
| Linux | `e2e_full_linux_amd64` |
| macOS (Apple Silicon) | `e2e_full_macos_arm64` |

```bash
# Linux / macOS
chmod +x e2e_full_linux_amd64
./e2e_full_linux_amd64

# Windows — double-click or run from terminal
e2e_full_windows_amd64.exe
```

---

## Option 3 — Run from Source (Go required)

```bash
# From the backend repo
go run ./cmd/e2e_full

# Against local server
go run ./cmd/e2e_full -base-url http://localhost:8080 -region in

# As a standalone module (no clone needed)
go run github.com/IAEX-Network/iaex-genesis-x-1-sandbox@latest
```

---

## What It Does

A single run executes a complete trade lifecycle:

```
FarmFresh Ltd (Supplier)  ─── ships wheat ───►  RetailCo Ltd (Buyer)
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
  SwiftLogistics          TrustAudit Inc         NationalBank
  (Transport)             (Auditor)              (Bank / Financier)
```

**~30 actors** are registered across 5 organizations. Every actor enrolls an Ed25519 key at registration. Every business event is signed. The final phase independently verifies every signature — without trusting the server.

---

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-base-url` | `https://api.iaexnetwork.com` | API base URL |
| `-region` | `in` | Region for actor registration (`in` or `eu`) |

No environment variables required. Run ID is auto-generated (timestamp-based). Concurrent runs are supported — each run registers actors with a unique suffix.

---

## Phases

| Phase | What happens |
|-------|-------------|
| **1 — Register** | 5 orgs + ~25 staff registered; Ed25519 keys enrolled at registration |
| **2 — Onboard** | Each org onboards via `POST /onboarding/organization` → gets FRL ledger |
| **3 — Delegate** | Each org grants staff delegation via `POST /delegation/organization` (signed) |
| **4 — FRL Masters** | Each org creates a TraceLedger master on their FRL (signed) |
| **5 — Master Access** | Each org grants staff write on their FRL master (signed) |
| **6 — Sup Events** | FarmFresh records: PRODUCTION_STARTED → TEMPERATURE_LOG ×3 → QC_INSPECTION_PASSED → SHIPMENT_READY |
| **7 — ORDER Ledger** | FarmFresh creates ORDER ledger with RetailCo via `POST /ledgers` |
| **8 — ORDER Master** | FarmFresh creates ORDER master; grants write to 13 cross-party delegates (signed) |
| **9 — Order Events** | 5 parties append 15 signed events: pickup, transit, IoT logs, delivery, audit, loan, disbursement |
| **10 — Buy Events** | RetailCo records: GOODS_ACCEPTED → DOCK_SCAN → BUYER_QC_PASSED → WAREHOUSE_RECEIVED |
| **11 — Close** | FarmFresh closes ORDER master + ORDER ledger (both actor-signed) |
| **12 — Verify (10A)** | SigLog self-verify: re-verify every sig locally using stored public keys (zero server trust) |
| **13 — Verify (10B)** | Cross-party: each party fetches ledger via own API key + independently verifies Ed25519 |

---

## Expected Output (summary)

```
PHASE 10A: SIGLOG SELF-VERIFICATION (client-side Ed25519 — zero server trust)
────────────────────────────────────────────────────────────────────────
#    event_type                           actor          result
────────────────────────────────────────────────────────────────────────
1    SUPPLIER_DELEGATION_GRANTED          FarmFresh Ltd  ✓ PASS
2    SUPPLIER_DELEGATION_GRANTED          FarmFresh Ltd  ✓ PASS
...
SigLog: 74 PASS  0 FAIL  74 total

VERIFIER: NationalBank (bnk_loan)
LEDGER:   ORDER ledger  id=8eecc02d-...
────────────────────────────────────────────────────────────────────────
...
integrity:✓  Ed25519-pass:33  fail:0  total:33

CROSS-PARTY VERIFICATION SUMMARY
────────────────────────────────────────────────────────────────────────
  Ed25519 verified : 116
  Failed           : 0
  Proof model: client-side Ed25519 — no server trust required
```

---

## Network Topology Exercised

```
Ledger / Scope          Party Access
─────────────────────────────────────────────────────────────────
Supplier FRL            FarmFresh org + 6 staff (direct)
                        Verified in 10B by sup_mgr API key

Buyer FRL               RetailCo org + 6 staff (direct)
                        Verified in 10B by buy_proc API key

ORDER Ledger            FarmFresh + RetailCo (direct parties)
                        SwiftLogistics — delegated via master access
                        TrustAudit Inc — delegated via master access
                        NationalBank   — delegated via master access
                        Verified in 10B by bnk_loan + trn_pickup API keys
```

---

## Signing Coverage

Every actor-signed event uses the Layer A protocol:

```
msg    = eventType + "\x00" + ledgerID + "\x00" + canonicalJSON(sigPayload)
digest = SHA-256(msg)
sig    = Ed25519.Sign(actorPrivKey, digest)
```

The `SigLog` in `NetworkState` accumulates every `SigRecord` during the flow. Phase 10A replays all of them locally. Phase 10B fetches from the server and re-verifies.

Causal links are sealed cryptographically:
- `QC_INSPECTION_PASSED.caused_by_hash` → `PRODUCTION_STARTED` hash
- `DELIVERED.caused_by_hash` → `GOODS_PICKED_UP` hash
- `LOAN_DISBURSED.caused_by_hash` → `DELIVERED` hash

---

## File Structure

```
cmd/e2e_full/
├── main.go          — phase orchestration (phases 1–13)
├── state.go         — NetworkState, ActorState, SigRecord, MasterState
├── client.go        — HTTP helpers, signing helpers, canonicalJSON
├── sup_phases.go    — FarmFresh registration, onboard, delegate, FRL events
├── buy_phases.go    — RetailCo registration, onboard, delegate, FRL events
├── trade_phases.go  — all 5-party roles: Auditor, Transport, Bank, ORDER ledger/events, close
└── verify.go        — Phase 10A (SigLog) + Phase 10B (cross-party) verification
```

---

## Using as Integration Test

The program exits with a non-zero status on any failure (HTTP error, sig mismatch, missing field). Run in CI to validate the live API:

```bash
go run ./cmd/e2e_full 2>&1 | tee e2e_$(date +%Y%m%d_%H%M%S).log
echo "Exit: $?"
```

Zero exit = all phases passed including cross-party Ed25519 verification.
