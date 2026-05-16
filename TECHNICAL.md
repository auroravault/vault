# Technical Reference

Encryption model, receipt format, ledger format, and verification procedure. For CLI usage, see [CLI.md](CLI.md).

---

## Encryption model

**Per-object random keys.** Each `vault put` generates a fresh 32-byte AES-256-GCM key. Keys are never reused across objects. The key never leaves the client machine.

**Wire format:** `[12-byte nonce || ciphertext+tag]`. No deviations.

**Object ID:** `SHA-256(ciphertext_bytes)`, hex-encoded, lowercase. Derived from content — not assigned by the system. The same plaintext encrypted with a different key produces a different object ID.

**Content commitment:** Before upload, the client computes:

```
content_commitment = SHA-256(commitment_salt || plaintext_sha256)
```

where `commitment_salt` is 32 random bytes and `plaintext_sha256 = SHA-256(plaintext)`.

The `content_commitment` is sent to the server and signed into the receipt. The `plaintext_sha256` and `commitment_salt` remain on the client machine only, stored in the receipt's `local_proof` block. This allows offline verification that the receipt corresponds to a specific file version — without the server ever seeing the plaintext hash.

---

## Receipt format (v1 — frozen)

```json
{
  "receipt_version": "1",
  "receipt": {
    "account_id": "...",
    "content_commitment": "<sha256-hex>",
    "nonce": "...",
    "object_id": "<sha256-hex>",
    "put_id": "<sha256-hex>",
    "server_key_id": "witness-1",
    "size": 12345,
    "ts": 1778009184
  },
  "receipt_hash": "<sha256-hex>",
  "server_sig": "<base64>"
}
```

`receipt_hash = SHA-256(canonical_json(receipt))`.
`server_sig = Ed25519.sign(canonical_json(receipt))`.
`receipt_version` and `receipt_hash` are NOT signed — only the inner `receipt` object is.

**Canonicalization:** UTF-8, keys sorted lexicographically, separators `","` and `":"`, no whitespace, integers not floats, lowercase hex.

> Format is frozen — old v1 receipts remain verifiable against the published v1 specification. See [RECEIPT_FORMAT.md](RECEIPT_FORMAT.md) for the full specification.

---

## Continuity ledger format (v1 — frozen)

```json
{
  "chain_version": "1",
  "account_id": "...",
  "last_hash": "<sha256-hex>",
  "events": [
    {
      "ts": 1778009184,
      "receipt_hash": "<sha256-hex>",
      "chain_hash": "<sha256-hex>"
    }
  ]
}
```

**Chain hash formula:**

```
chain_hash[0] = sha256_utf8("" + receipt_hash[0])
chain_hash[i] = sha256_utf8(chain_hash[i-1] + receipt_hash[i])
```

> Formula is frozen for v1 ledger verification. See [LEDGER_FORMAT.md](LEDGER_FORMAT.md) for the full specification.

---

## Verification procedure

### Receipt verification (offline)

1. Load the receipt JSON file.
2. Recompute `canonical_json(receipt)`.
3. Assert `SHA-256(canonical_bytes) == receipt_hash`.
4. Verify `Ed25519.verify(canonical_bytes, base64_decode(server_sig), server_pubkey)`.
5. If `local_proof` present: assert `SHA-256(commitment_salt || plaintext_sha256) == receipt.content_commitment`.

### File verification (requires Continuity ledger access)

1. Compute `SHA-256(file)`.
2. Scan local receipts for one whose `local_proof.plaintext_sha256` matches.
3. Verify the content commitment (step 5 above).
4. Verify the receipt hash and signature (steps 3–4 above).
5. Fetch `/v1/continuity/export` and verify the receipt_hash appears in an unbroken chain.

Exit codes: `0` pass, `4` verification failed, `5` file unreadable.

---

## Server endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Server health |
| `GET /v1/version` | Server and API version |
| `GET /v1/witness/pubkey` | Server Ed25519 signing public key |
| `POST /v1/witness/receipts` | Request a receipt for a stored object |
| `GET /v1/continuity/export` | Export full hash-chained ledger |
| `PUT /v1/vault/objects/{object_id}` | Upload ciphertext |
| `GET /v1/vault/objects/{object_id}` | Download ciphertext |
| `HEAD /v1/vault/objects/{object_id}` | Check object existence |
| `DELETE /v1/vault/objects/{object_id}` | Delete ciphertext |

Object, receipt, and continuity endpoints require `Authorization: Bearer vlt_…`. Public key and version endpoints are exposed for client setup and verification workflows.

---

## Keystore

Per-object AES keys are stored locally in `keystore.json` (schema v4). The keystore is not passphrase-encrypted in the current release. Protecting the machine and filesystem is the operator's responsibility.

Fields per record: `object_id`, `aes_key`, `status` (`active | deleted | missing_remote`), `put_id`, `content_commitment`, `source_name`, `receipt_path`, and related timestamps.

---

## Compatibility and boundaries

Receipt v1 and Continuity ledger v1 are compatibility-bound. Stream ledger schema v2 is local stream state and remains pre-freeze.

The keystore is not passphrase-encrypted, object availability is separate from receipt validity, and file verification requires access to the current Continuity ledger.

---

## Interoperability

The implementation keeps golden test vectors for receipt v1 canonicalization, hashing, and signature verification.

Independent verification requires only canonical JSON, SHA-256, and Ed25519 verification with the server public key.
