# Trust Posture

AuroraVault is designed for technical operators who want verifiable evidence without handing over plaintext. This page states the current compatibility posture, what the system deliberately does not do, and where responsibility remains with the operator.

The service keeps the operational model narrow: encrypted objects, signed receipts, continuity metadata, and CLI-first workflows. It avoids dashboards, content interpretation, managed keys, and sync semantics.

---

## Current version

| | |
|--|--|
| Server | `0.8.3` |
| Client | `0.16.10` |
| API | `v1` |
| Access | Trial by invite; Basic and High through checkout claim-code redemption. |

AuroraVault runtime and CLI versions are intentionally pre-1.0. API v1,
receipt format v1, and Continuity ledger v1 remain independently stable.

---

## What is ready

- Client-side AES-256-GCM encryption before upload.
- Ed25519 Witness receipts for uploaded objects.
- Receipt v1 canonicalization, hashing, and signature verification.
- Continuity ledger anchoring for file verification.
- Pipe-first CLI flows for stdin, stdout, and machine-readable output.
- Client-managed stream chains for repeated file, directory, stdin, or process capture.

---

## Compatibility

- Receipt v1 is compatibility-bound.
- Continuity ledger v1 is compatibility-bound.
- Product runtime / CLI uses `0.x.y` and may still evolve before 1.0.0.
- Machine-readable CLI errors are stable and additive.

---

## Known constraints

- The local keystore is not passphrase-encrypted.
- Receipt validity is separate from object availability.
- File verification needs access to the current Continuity ledger.
- API behavior is documented against the current runtime; receipt and Continuity formats remain the compatibility anchors.
- No browser drive, sync client, shared folders, key recovery, or managed interpretation layer.

---

## Trust boundary

AuroraVault can prove that the server signed a receipt for a content commitment and that the receipt is anchored in the account ledger. It does not prove authorship, legal meaning, object availability, or the truth of the underlying content.

The system is intentionally narrow: encrypted object storage, signed receipts, and verifiable continuity metadata.
