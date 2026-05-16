# Witness Receipt Format — v1

**Status:** Frozen. Compatibility-bound public proof contract.
**Effective:** 2026-05-07

---

## Overview

A Witness receipt is a tamper-evident proof that:
- A specific object was stored on the server at a recorded time.
- The client committed to a specific local file version at that time (`content_commitment`).
- The server signed the commitment without seeing the plaintext.

Receipts are portable verification artifacts. Once issued, they must remain verifiable offline indefinitely.

---

## Wrapper Structure

```json
{
  "receipt_version": "1",
  "receipt": { ... },
  "receipt_hash": "<sha256-hex>",
  "server_sig": "<base64>"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `receipt_version` | string | Always `"1"` for this format. NOT signed. |
| `receipt` | object | The signed receipt object (see below). |
| `receipt_hash` | string | `SHA-256(canonical_json(receipt))`, hex-encoded. NOT separately signed. |
| `server_sig` | string | Ed25519 signature over `canonical_json(receipt)`, base64-encoded. |

**Invariants:**
- `receipt` is the ONLY signed object.
- `receipt_hash = sha256(canonical_json(receipt))`
- `server_sig = Ed25519.sign(canonical_json(receipt))`
- `receipt_version` is NOT signed.
- `receipt_hash` is NOT separately signed.
- `server_sig` is NOT part of `receipt_hash`.

---

## Signed Receipt Object

```json
{
  "account_id": "...",
  "content_commitment": "<sha256-hex>",
  "nonce": "...",
  "object_id": "<sha256-hex>",
  "put_id": "<sha256-hex>",
  "server_key_id": "...",
  "size": 123,
  "ts": 1778009184
}
```

| Field | Type | Description |
|-------|------|-------------|
| `account_id` | string | Server-assigned account identifier. |
| `content_commitment` | string | `SHA-256(commitment_salt \|\| plaintext_sha256)`, hex. Binds receipt to a local file version without revealing the plaintext hash. |
| `nonce` | string | Random per-request string. Server rejects duplicates (replay protection). |
| `object_id` | string | `SHA-256(ciphertext)`, hex. Immutable object address. |
| `put_id` | string | Unique event ID: `SHA-256(object_id_bytes \|\| content_commitment_bytes \|\| random_16)`, hex. Client-generated. |
| `server_key_id` | string | Identifies which server signing key was used. |
| `size` | integer | Ciphertext size in bytes. |
| `ts` | integer | Unix timestamp (seconds). |

**Fields that are NOT included in the signed scope:**
- `filename`, `source_name`, `path`, MIME type
- `plaintext_sha256`, `commitment_salt`
- `local_proof`
- `client_sig`, `client_pubkey`
- Ledger metadata, wrapper fields

---

## Canonicalization Rules

These rules are frozen for v1. Any change requires a new version.

1. Encoding: UTF-8.
2. Keys sorted lexicographically.
3. Separators: exactly `","` and `":"` — no spaces.
4. No trailing whitespace or newlines.
5. No floats — all numeric fields are integers.
6. No nulls — absent optional fields are omitted entirely.
7. Lowercase hex for all hash and ID fields.
8. `size` is integer (not string).
9. `ts` is integer (not string).
10. Signatures and public keys are base64 (not hex).

**Reference implementation:**
```python
json.dumps(receipt_obj, separators=(",", ":"), sort_keys=True).encode("utf-8")
```

**Pinned example** — the following receipt object:
```json
{
  "account_id": "test-account",
  "content_commitment": "cccc...cc",
  "nonce": "testnonce",
  "object_id": "aaaa...aa",
  "put_id": "bbbb...bb",
  "server_key_id": "witness-1",
  "size": 100,
  "ts": 1000000
}
```
produces canonical bytes:
```
{"account_id":"test-account","content_commitment":"cccc...cc","nonce":"testnonce","object_id":"aaaa...aa","put_id":"bbbb...bb","server_key_id":"witness-1","size":100,"ts":1000000}
```
with `receipt_hash = 688a373ae72e1f3befd87f243a30931ab2773a6b936db6f36ca99bf69ce31553`

---

## Local Proof Block

The `local_proof` block is client-only. It is never sent to the server and is not covered by `server_sig`. It allows offline verification of which local plaintext version was committed.

```json
{
  "local_proof": {
    "put_id": "...",
    "object_id": "...",
    "plaintext_sha256": "<sha256-hex>",
    "commitment_salt": "<hex>",
    "content_commitment": "<sha256-hex>",
    "plaintext_size": 123,
    "ciphertext_size": 456
  }
}
```

| Field | Description |
|-------|-------------|
| `plaintext_sha256` | SHA-256 of the original plaintext. Never sent to server. |
| `commitment_salt` | Random 32-byte salt. Never sent to server. |
| `content_commitment` | `SHA-256(commitment_salt_bytes \|\| plaintext_sha256_bytes)`. Must equal `receipt.content_commitment`. |

**Zero-knowledge invariant:** `plaintext_sha256` and `commitment_salt` remain local-only at all times.

---

## Full Receipt File on Disk

```json
{
  "receipt_version": "1",
  "receipt": {
    "account_id": "...",
    "content_commitment": "...",
    "nonce": "...",
    "object_id": "...",
    "put_id": "...",
    "server_key_id": "witness-1",
    "size": 33,
    "ts": 1778009184
  },
  "receipt_hash": "...",
  "server_sig": "...",
  "local_proof": {
    "put_id": "...",
    "object_id": "...",
    "plaintext_sha256": "...",
    "commitment_salt": "...",
    "content_commitment": "...",
    "plaintext_size": 123,
    "ciphertext_size": 456
  }
}
```

---

## Verification Procedure

### Step 1 — Load receipt

Read the JSON file. Check that `receipt_version = "1"`.

### Step 2 — Recompute receipt_hash

```python
receipt_bytes = canonical_json(receipt)
computed_hash = sha256_hex(receipt_bytes)
assert computed_hash == wrapper["receipt_hash"]
```

### Step 3 — Verify server signature

```python
verify_key.verify(receipt_bytes, base64.b64decode(wrapper["server_sig"]))
```

### Step 4 (optional, if local_proof present) — Verify content commitment

```python
recomputed = sha256(bytes.fromhex(local_proof["commitment_salt"]) +
                    bytes.fromhex(local_proof["plaintext_sha256"]))
assert recomputed == receipt["content_commitment"]
```

### Step 5 (optional) — Verify ledger inclusion

Fetch `/v1/continuity/export`. Check that `receipt_hash` appears in `events[].receipt_hash`. Verify `chain_hash` sequence (see [LEDGER_FORMAT.md](LEDGER_FORMAT.md)).

---

## Compatibility Guarantees

1. `receipt_version="1"` receipts must remain verifiable indefinitely.
2. The signed field set for v1 is frozen. Fields cannot be added, removed, or renamed.
3. Canonicalization rules for v1 are frozen.
4. `receipt_hash` semantics for v1 are frozen.
5. Any incompatible change requires a new `receipt_version`.
6. New code may read old v1 receipts.
7. New servers may issue newer versions but must never redefine v1.
8. No silent migrations of proof artifacts.
9. `local_proof` fields may be added without version bump — they are not signed.
10. Additional wrapper fields may be added without version bump — they are not signed.
