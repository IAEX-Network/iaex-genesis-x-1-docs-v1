# Signing Reference

Every business event appended to Genesis X-1 carries an Ed25519 actor signature. This signature is what makes the event independently verifiable — any party with the actor's public key can confirm who authorized the event, what was authorized, and that nothing was changed after signing.

This document provides the exact signing formula, canonical JSON definition, and working implementations in Node.js, Python, and Go.

## Signing Formula

```
message = eventType + 0x00 + ledgerID + 0x00 + canonicalJSON(payload)
digest  = SHA-256(message)
sig     = Ed25519.Sign(privateKey, digest)
header  = X-Actor-Sig: base64(sig)
```

**Canonical JSON** — keys sorted A→Z at every nesting level, no spaces between tokens.

The `0x00` byte is a null separator between fields. It prevents ambiguity when field values contain adjacent characters.

## Node.js

```javascript
import { createHash, createPrivateKey, sign } from 'crypto';
import { readFileSync } from 'fs';

const privateKeyPem = readFileSync(process.env.IAEX_PRIVATE_KEY_PATH, 'utf8');

// Sort all object keys recursively — required to match server CanonicalJSON.
// JSON.stringify(obj, sortedKeys) only sorts top-level; this handles nested objects.
function canonicalJSON(value) {
  if (Array.isArray(value)) {
    return '[' + value.map(canonicalJSON).join(',') + ']';
  }
  if (value !== null && typeof value === 'object') {
    const keys = Object.keys(value).sort();
    return '{' + keys.map(k => JSON.stringify(k) + ':' + canonicalJSON(value[k])).join(',') + '}';
  }
  return JSON.stringify(value);
}

function buildSignature(eventType, ledgerID, payload) {
  const canon = canonicalJSON(payload);
  const message = Buffer.concat([
    Buffer.from(eventType, 'utf8'),
    Buffer.from([0x00]),
    Buffer.from(ledgerID, 'utf8'),
    Buffer.from([0x00]),
    Buffer.from(canon, 'utf8'),
  ]);
  const digest = createHash('sha256').update(message).digest();
  const privKey = createPrivateKey(privateKeyPem);
  return sign(null, digest, privKey).toString('base64');
}
```

### Complete Client Class

```javascript
export class IAEXClient {
  constructor({ apiKey, baseUrl = 'https://api.iaexnetwork.com/v1', privateKeyPem }) {
    this.apiKey = apiKey;
    this.baseUrl = baseUrl;
    this.privateKeyPem = privateKeyPem;
  }

  sign(eventType, ledgerID, payload) {
    return buildSignature(eventType, ledgerID, payload);
  }

  async request(method, path, body, sig) {
    const headers = {
      'Authorization': `Bearer ${this.apiKey}`,
      'Content-Type': 'application/json',
    };
    if (sig) headers['X-Actor-Sig'] = sig;
    const res = await fetch(this.baseUrl + path, {
      method,
      headers,
      body: body ? JSON.stringify(body) : undefined,
    });
    if (!res.ok) throw await res.json();
    return res.json();
  }

  openLedger(body) {
    // No signature — ledger ID is server-generated.
    return this.request('POST', '/ledgers', body, null);
  }

  createMaster(ledgerID, businessRef, scope) {
    const sigPayload = { business_ref: businessRef, ledger_id: ledgerID, scope };
    const sig = buildSignature('TRACELEDGER_MASTER_CREATED', ledgerID, sigPayload);
    return this.request('POST', '/traceledger/master', {
      ledger_id: ledgerID, business_ref: businessRef, scope,
    }, sig);
  }

  appendEvent(masterUUID, ledgerID, eventType, payload) {
    const sig = buildSignature(eventType, ledgerID, payload);
    return this.request('POST', `/traceledger/master/${masterUUID}/events`, {
      event_type: eventType, payload,
    }, sig);
  }
}
```

## Python

```python
import hashlib, base64, json, os
import httpx
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from cryptography.hazmat.primitives.serialization import load_pem_private_key

def load_private_key() -> Ed25519PrivateKey:
    with open(os.environ['IAEX_PRIVATE_KEY_PATH'], 'rb') as f:
        return load_pem_private_key(f.read(), password=None)

def build_signature(private_key: Ed25519PrivateKey, event_type: str, ledger_id: str, payload: dict) -> str:
    canon = json.dumps(payload, sort_keys=True, separators=(',', ':')).encode()
    message = event_type.encode() + b'\x00' + ledger_id.encode() + b'\x00' + canon
    digest = hashlib.sha256(message).digest()
    return base64.b64encode(private_key.sign(digest)).decode()


class IAEXClient:
    def __init__(self, api_key: str, base_url: str = 'https://api.iaexnetwork.com/v1'):
        self.api_key = api_key
        self.base_url = base_url
        self.private_key = load_private_key()
        self.http = httpx.Client(base_url=base_url)

    def _headers(self, sig: str = None) -> dict:
        h = {'Authorization': f'Bearer {self.api_key}'}
        if sig:
            h['X-Actor-Sig'] = sig
        return h

    def open_ledger(self, buyer_actor_id, buyer_org_id, supplier_actor_id, supplier_org_id):
        r = self.http.post('/ledgers', headers=self._headers(), json={
            'buyer_actor_id': buyer_actor_id,
            'buyer_organization_id': buyer_org_id,
            'supplier_actor_id': supplier_actor_id,
            'supplier_organization_id': supplier_org_id,
        })
        r.raise_for_status()
        return r.json()

    def create_master(self, ledger_id: str, business_ref: str, scope: str):
        sig_payload = {'business_ref': business_ref, 'ledger_id': ledger_id, 'scope': scope}
        sig = build_signature(self.private_key, 'TRACELEDGER_MASTER_CREATED', ledger_id, sig_payload)
        r = self.http.post('/traceledger/master',
            headers=self._headers(sig),
            json={'ledger_id': ledger_id, 'business_ref': business_ref, 'scope': scope})
        r.raise_for_status()
        return r.json()

    def append_event(self, master_uuid: str, ledger_id: str, event_type: str, payload: dict):
        sig = build_signature(self.private_key, event_type, ledger_id, payload)
        r = self.http.post(f'/traceledger/master/{master_uuid}/events',
            headers=self._headers(sig),
            json={'event_type': event_type, 'payload': payload})
        r.raise_for_status()
        return r.json()


# Usage
client = IAEXClient(api_key=os.environ['IAEX_API_KEY'])

ledger = client.open_ledger('actor-a', 'org-a', 'actor-b', 'org-b')
master = client.create_master(ledger['id'], 'PO-2026-001', 'ORDER')
event  = client.append_event(master['uuid'], ledger['id'], 'SHIPMENT_DISPATCHED', {
    'carrier': 'BlueDart',
    'tracking_number': 'BD123456789',
    'quantity_kg': 500,
})
```

## Go

```go
package iaex

import (
    "bytes"
    "crypto/ed25519"
    "crypto/sha256"
    "encoding/base64"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

func BuildSignature(privKey ed25519.PrivateKey, eventType, ledgerID string, payload map[string]any) (string, error) {
    canon, err := json.Marshal(payload) // Go sorts map keys A→Z
    if err != nil {
        return "", fmt.Errorf("marshal: %w", err)
    }
    var msg []byte
    msg = append(msg, eventType...)
    msg = append(msg, 0x00)
    msg = append(msg, ledgerID...)
    msg = append(msg, 0x00)
    msg = append(msg, canon...)
    digest := sha256.Sum256(msg)
    return base64.StdEncoding.EncodeToString(ed25519.Sign(privKey, digest[:])), nil
}

type Client struct {
    BaseURL string
    APIKey  string
    PrivKey ed25519.PrivateKey
    HTTP    *http.Client
}

func (c *Client) do(method, path string, body any, sig string) ([]byte, error) {
    b, _ := json.Marshal(body)
    req, _ := http.NewRequest(method, c.BaseURL+path, bytes.NewReader(b))
    req.Header.Set("Authorization", "Bearer "+c.APIKey)
    req.Header.Set("Content-Type", "application/json")
    if sig != "" {
        req.Header.Set("X-Actor-Sig", sig)
    }
    resp, err := c.HTTP.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}

func (c *Client) AppendEvent(masterUUID, ledgerID, eventType string, payload map[string]any) ([]byte, error) {
    sig, err := BuildSignature(c.PrivKey, eventType, ledgerID, payload)
    if err != nil {
        return nil, err
    }
    return c.do("POST", "/traceledger/master/"+masterUUID+"/events", map[string]any{
        "event_type": eventType,
        "payload":    payload,
    }, sig)
}
```

## Partial Signing Reference

Some operations involve server-generated fields (IDs, timestamps) that cannot be known before the call. For these, sign only the fields the caller provides. Keys sorted A→Z in all cases.

Formula applies in all cases: `SHA-256(eventType + 0x00 + ledgerID + 0x00 + canonicalJSON(sigPayload))`

| Operation | Event type (exact) | ledgerID | Sign over |
|---|---|---|---|
| Create master | `TRACELEDGER_MASTER_CREATED` | Ledger the master belongs to | `{business_ref, ledger_id, scope}` |
| Grant org delegation | `SUPPLIER_DELEGATION_GRANTED` | Org's FRL ledger ID | `{delegate_actor_id, organization_id, permissions, role}` |
| Revoke org delegation | `SUPPLIER_DELEGATION_REVOKED` | Org's FRL ledger ID | `{delegate_actor_id, organization_id}` |
| Grant ledger delegation | `LEDGER_DELEGATION_GRANTED` | The ledger being delegated | `{delegate_actor_id, ledger_id, permissions, role}` |
| Revoke ledger delegation | `LEDGER_DELEGATION_REVOKED` | The ledger being delegated | `{delegate_actor_id, ledger_id}` |
| Grant master access | `MASTER_DELEGATION_GRANTED` | Parent ledger of the master | `{delegate_actor_id, permissions, role, traceledger_master_id}` |
| Revoke master access | `MASTER_DELEGATION_REVOKED` | Parent ledger of the master | `{delegate_actor_id, traceledger_master_id}` |
| All other events | Your `UPPER_SNAKE_CASE` string | The event's ledger | Full payload |

For master delegation: use the `ledger_id` of the ledger the master belongs to, not the master UUID. Retrieve it from `GET /traceledger/master/{uuid}` → `ledger_id`.

## Operations That Do Not Require X-Actor-Sig

| Operation | Reason |
|---|---|
| `POST /onboarding/organization` | Authority seals all structural events |
| `POST /entities/onboard` | Actor being created — no prior key |
| `POST /entities/root-ledger` | No keys enrolled at onboard time in Genesis X-1 |
| `POST /engagements` | Authority seals GENESIS |
| `POST /ledgers` | Ledger ID is server-generated |

## Debugging

```python
# 1. Print canonical JSON — verify key sort order
import json
print(json.dumps(payload, sort_keys=True, separators=(',', ':')))

# 2. Print SHA-256 digest (hex) — compare with server logs
import hashlib
canon = json.dumps(payload, sort_keys=True, separators=(',', ':')).encode()
msg = event_type.encode() + b'\x00' + ledger_id.encode() + b'\x00' + canon
print(hashlib.sha256(msg).hexdigest())
```

Common errors:

| Error | Cause |
|---|---|
| `"no signing key enrolled"` | Enroll key via `POST /actors/{id}/keys` first |
| `"X-Actor-Sig header required"` | Key enrolled but header missing |
| `"actor_sig verification failed"` | Wrong key sort order, missing SHA-256 step, or wrong ledgerID |
