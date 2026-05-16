# Continuity Ledger Format — v1

**Status:** Frozen. Compatibility-bound public proof contract.
**Effective:** 2026-05-07

---

## Overview

The Continuity ledger is an append-only hash chain that proves ordering and completeness of all Witness receipts for an account. It allows independent verification that:
- A receipt was appended to the chain (inclusion proof).
- No events were silently removed or reordered (chain integrity).

The ledger is not a content store. It records only `receipt_hash` values derived from canonical receipts.

---

## Export Format

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

| Field | Type | Description |
|-------|------|-------------|
| `chain_version` | string | Always `"1"` for this format. |
| `account_id` | string | Account this ledger belongs to. |
| `last_hash` | string | `chain_hash` of the last event. Empty string if no events. |
| `events` | array | Ordered list of events, oldest first. |

### Event fields

| Field | Type | Description |
|-------|------|-------------|
| `ts` | integer | Unix timestamp from the receipt. |
| `receipt_hash` | string | `SHA-256(canonical_json(receipt))`, hex. Same value used in the receipt wrapper. |
| `chain_hash` | string | `SHA-256((prev_chain_hash + receipt_hash).encode("utf-8"))`, hex. See formula below. |

---

## Chain Hash Formula

```
chain_hash[0] = sha256_utf8("" + receipt_hash[0])
chain_hash[i] = sha256_utf8(chain_hash[i-1] + receipt_hash[i])
```

Where `sha256_utf8(s)` = `SHA-256(s.encode("utf-8"))`.

The seed for the first event is the empty string `""`.

**Reference implementation:**
```python
running = ""
for event in events:
    computed = sha256(((running or "") + event["receipt_hash"]).encode("utf-8")).hexdigest()
    assert computed == event["chain_hash"]
    running = event["chain_hash"]
```

---

## receipt_hash Derivation

The `receipt_hash` appended to the ledger is identical to the `receipt_hash` in the receipt wrapper:

```python
receipt_hash = sha256_hex(canonical_json_bytes(receipt_obj))
```

This means ledger inclusion can be independently verified from the receipt file alone — no server contact needed after export.

---

## Ordering Guarantees

1. Events are ordered by insertion time (append-only).
2. The chain_hash sequence is deterministic given the list of `receipt_hash` values.
3. Insertion order is preserved in the export.
4. Events are never deleted or reordered — the chain breaks if they are.

---

## Append-Only Invariant

The ledger `events` table must never be modified after a row is written. Violations break the chain_hash sequence and are detectable by any verifier.

---

## Verification Procedure

```python
running = ""
for event in events:
    computed = sha256(((running or "") + event["receipt_hash"]).encode("utf-8")).hexdigest()
    if computed != event["chain_hash"]:
        raise VerificationError(f"chain broken at ts={event['ts']}")
    running = event["chain_hash"]
```

To check ledger inclusion for a specific receipt:
1. Compute `receipt_hash = sha256_hex(canonical_json(receipt))`.
2. Fetch ledger export from `/v1/continuity/export`.
3. Check that `receipt_hash` appears in any `event["receipt_hash"]`.
4. Walk the chain from the start to confirm `chain_hash` sequence is unbroken up to and including that event.

---

## Compatibility Guarantees

1. Ledger v1 chain_hash semantics are frozen.
2. The `chain_hash` formula is frozen: `sha256_utf8(prev + receipt_hash)` with empty-string seed.
3. `receipt_hash` derivation is frozen: `sha256(canonical_json(receipt))`.
4. Any change to chain semantics requires a new `chain_version`.
5. Existing events in a v1 ledger must remain verifiable indefinitely.
6. New code may verify old v1 ledger exports.
7. No silent re-hashing or rewriting of existing chain entries.
