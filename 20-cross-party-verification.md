# Cross-Party Verification

Genesis X-1 supports independent Ed25519 signature verification by any party that has been granted access to a ledger. No trust in the server is required — verification uses only the actor's public key and the event data returned by the API.

---

## What It Proves

Cross-party verification proves:

- The named actor held the private key at signing time
- The actor committed to the exact payload fields recorded in `actor_sig_payload`
- No substitution is possible — a different actor or a different payload produces a different (invalid) digest
- The server could not have fabricated the signature (it does not hold actor private keys)

---

## Two Verification Layers

### Layer A — Actor Authorization (non-repudiation)

Verifies: which actor signed what payload.

```
sigTarget = actor_sig_payload  (if non-null)
          = payload             (fallback — actor signed full payload)

msg    = eventType + "\x00" + ledgerID + "\x00" + canonicalJSON(sigTarget)
digest = SHA-256(msg)
valid  = Ed25519.Verify(actorPublicKey, digest, base64Decode(actor_sig))
```

- `actorPublicKey` is fetched from `GET /actors/{actor_id}/keys` (public endpoint, no auth).
- This verification is fully client-side. No server call required beyond reading the event.

### Layer B — Chain Continuity (tamper detection)

Verifies: the append sequence was not modified.

Returned as `integrity.verified` in every event read. The server recomputes every hash in the returned chain and reports any mismatch in `integrity.issues`.

The two layers are **cryptographically independent**. Layer A covers actor intent; Layer B covers platform continuity.

---

## Delegation Scope Proof

Cross-party verification also proves delegation scope: a party can only fetch events on ledgers they are delegated to access. The server enforces this at read time using the caller's API key.

Example scope enforcement in the 5-party model:

| Verifier | API Key | Can fetch | Cannot fetch |
|----------|---------|-----------|--------------|
| NationalBank (bnk_loan) | bnk_loan key | ORDER ledger (delegated via master access) | Supplier FRL, Buyer FRL |
| SwiftLogistics (trn_pickup) | trn_pickup key | ORDER ledger | FRL ledgers of other parties |
| FarmFresh (sup_mgr) | sup_mgr key | Supplier FRL (direct org member) | ORDER ledger, Buyer FRL |
| RetailCo (buy_proc) | buy_proc key | Buyer FRL (direct org member) | ORDER ledger, Supplier FRL |

---

## Verification Protocol (Step by Step)

### Prerequisites

- Ledger ID(s) to verify
- The verifying party's API key (scoped to the ledger)
- Actor public keys (fetched from `GET /actors/{actor_id}/keys`)

### Steps

1. **Fetch events**

```bash
curl -sS "$BASE_URL/ledgers/$LEDGER_ID/events?limit=1000" \
  -H "Authorization: Bearer $VERIFIER_API_KEY"
```

2. **Check chain integrity**

```json
{"integrity": {"verified": true, "issues": []}}
```

`verified=false` means Layer B chain break — escalate immediately.

3. **For each actor-signed event** (`actor_sig` non-null):

```python
# Determine sigTarget
if event["actor_sig_payload"] is not None:
    sig_target = event["actor_sig_payload"]
else:
    sig_target = event["payload"]

# Reconstruct sig message
canon  = canonical_json(sig_target)          # recursive sorted-key, no whitespace
msg    = event_type + "\x00" + ledger_id + "\x00" + canon
digest = sha256(msg)                          # 32 bytes

# Fetch actor public key (Ed25519, base64-encoded)
pub_key = fetch_actor_public_key(event["actor_id"], event["signing_key_id"])

# Verify
valid = ed25519_verify(pub_key, digest, base64_decode(event["actor_sig"]))
```

4. **For authority-signed events** (`GENESIS`, `ORDER_OPENED`, `LEDGER_CLOSED`, `ORGANIZATION_REGISTERED`):

`actor_sig` is null. These events are sealed by the platform authority keyring. Verify Layer B chain integrity covers them; for Phase 2 authority key verification, use `GET /.well-known/iaex-authority`.

---

## canonicalJSON Definition

Canonical JSON is deterministic: keys sorted lexicographically, no whitespace, recursive.

```python
def canonical_json(value):
    if isinstance(value, dict):
        sorted_items = sorted(value.items())
        inner = ",".join(f'"{k}":{canonical_json(v)}' for k, v in sorted_items)
        return "{" + inner + "}"
    elif isinstance(value, list):
        return "[" + ",".join(canonical_json(v) for v in value) + "]"
    else:
        return json.dumps(value, ensure_ascii=False, separators=(',', ':'))
```

Go reference implementation: `internal/event/sig.go → CanonicalJSON`.

---

## Reference: actor_sig_payload vs payload

| Scenario | actor_sig_payload | What client signed |
|----------|------------------|--------------------|
| Master business events | null | full `payload` (client injects all fields before signing) |
| `TRACELEDGER_MASTER_CREATED` | `{"ledger_id","business_ref","scope"}` | subset (server adds master UUID after creation) |
| `SUPPLIER_DELEGATION_GRANTED` | `{"delegate_actor_id","organization_id","permissions","role"}` | subset (server adds link_id, granted_at) |
| `MASTER_DELEGATION_GRANTED` | `{"traceledger_master_id","delegate_actor_id","role","permissions"}` | subset |
| `MASTER_CLOSED` | `{"traceledger_master_uuid","ledger_id","ledger_type","requested_by_actor_id","status","supplier_actor_id",...}` | subset stored explicitly |
| `LEDGER_CLOSED` | n/a (authority-signed) | client sig stored in payload as `requestor_sig` |

When `actor_sig_payload` is non-null, **always use it** for verification. Using `payload` instead may include server-injected fields (link IDs, delivery chain) that the client did not sign.

---

## Go Implementation Reference

The `cmd/e2e_full` program includes a working Phase 10 verification implementation:

- `verify.go → verifySigLog()` — Phase 10A: client-side SigLog self-verification (zero server calls)
- `verify.go → verifyCrossParty()` — Phase 10B: per-party API key fetch + Ed25519 verify
- `verify.go → verifyEventSig()` — single event verification against `actor_sig_payload` or `payload`
- `verify.go → buildPubKeyMap()` — builds actorID → ed25519.PublicKey map from NetworkState

---

## Result Interpretation

| Result | Meaning |
|--------|---------|
| `[✓ Ed25519 VERIFIED]` | Sig valid — actor committed to this exact payload |
| `[⚡ authority]` | Authority-signed — platform sealed, not actor-attributed |
| `(unsigned — infrastructure)` | No key enrolled at time of event (grace period, dev/test only) |
| `✗ Ed25519 INVALID` | Sig does not match pubkey + payload — investigate immediately |
| `integrity.verified=false` | Layer B chain break — stored events may have been tampered |
