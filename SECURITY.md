# Security Model

Understanding boundaries matters more than assuming them. This document describes what the server sees, what it cannot see, and which risks are explicitly accepted rather than eliminated.

---

## What the server sees

| Item | Visible to server |
|------|-------------------|
| Ciphertext bytes | Yes (opaque) |
| Object address (SHA-256 of ciphertext) | Yes |
| Ciphertext size | Yes |
| Content commitment (hides plaintext hash) | Yes |
| Timestamp and nonce | Yes |
| Account identifier | Yes |
| Plaintext content | No |
| Plaintext hash | No |
| Filename or source path | No |
| MIME type or content type | No |
| Encryption key | No |

---

## Encryption keys are client-owned

The keystore at `<home>/keystore.json` holds the AES-256-GCM key for every stored object. Losing this file means losing the ability to decrypt stored objects. The server cannot help. There is no key recovery service.

> The keystore is not passphrase-protected in the current release. It is stored as plaintext JSON with restricted file permissions (mode 600). Protecting the machine and filesystem is the operator's responsibility.

---

## Third-party integrations

Aurora Vault does not enforce encryption at the API layer.

The official Vault CLI encrypts locally before upload, so the server receives only ciphertext when the CLI is used as intended. Third-party integrations may implement the same model, or they may choose to upload plaintext directly. In those cases, the server stores plaintext blobs exactly as received.

Aurora Vault itself does not decrypt, interpret, or transform uploaded data. The API is transport-agnostic: encryption behavior depends on the client implementation and trust model. If you integrate the Vault API directly in your own software, we strongly recommend encrypting data locally before upload to preserve the intended zero-knowledge security model.

Key management is entirely your responsibility. Lost keys cannot be recovered. Compromised keys must be treated as permanently compromised.

If a third-party service integrates Vault on your behalf, you should understand where encryption occurs, who can access plaintext, and how keys are handled before trusting the integration. Independent Witness receipts and Continuity proofs remain valid regardless of whether uploaded payloads are encrypted or plaintext. What changes is the confidentiality boundary.

---

## Server compromise

If the server is compromised, the attacker obtains: encrypted blobs, receipt metadata, and account tokens. They cannot decrypt the blobs without client-side keys.

The appropriate response: revoke compromised account tokens, rotate the server signing key.

---

## Metadata still exists

The server knows: which accounts exist, when uploads occur, how large uploaded objects are, and which content commitments are associated with which objects. This metadata is not zero-knowledge. Concealing the fact that uploads are happening requires additional measures outside this system's scope.

---

## Receipt validity vs. storage availability

A signed receipt proves an object existed at a certain time. It does not prove the object is still stored. If an object is deleted, the receipt remains valid and verifiable — but `vault get` returns 404.

Use `vault info` for local proof state and `vault sync` when you need to check server-side object existence for local records.

---

## Server signing key

The server signing key is the trust anchor for receipt verification. If it is compromised, an attacker can issue fraudulent future receipts. Past receipts remain tied to the `server_key_id` and public key used for verification.

Protecting and rotating this key is an operator responsibility.

---

## Nonce replay protection

The server rejects any witness request whose nonce has been used before for the same account. This prevents a captured signed request from being replayed to obtain a second receipt.

---

## Accepted risks

A system that names its own weaknesses is easier to trust than one that doesn't. These are risks we have decided to accept rather than eliminate. We state them here because you should know them before you rely on the system.

- The local keystore is not passphrase-encrypted.
- Server signing-key compromise can allow fraudulent future receipts until the key is rotated and clients verify against the intended public key.
- Metadata still exists: account identifiers, timing, object sizes, object IDs, content commitments, and operational outcomes.
- Receipt validity is separate from object availability.
- Abuse, cost exposure, and lawful requests cannot be eliminated by the architecture.

---

## What is not planned

- Browser drive or sync client
- Shared folders or collaboration
- Server-side key recovery
- Content inspection or classification
- Multi-node failover as a product commitment
