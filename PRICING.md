# Pricing

Same product. Different limits.

Pricing exists to protect calm operation, predictable cost, and a small system that can keep being maintained.

---

## Tiers

### Trial — 14 days

Evaluation only. No production use promise.

- 1 GiB storage
- 25 MiB max object
- 10,000 monthly reads
- 10,000 monthly writes
- 1 GiB monthly egress
- 1 req/sec reads and writes

[Request invite](mailto:auth@auroranode.com?subject=Aurora%20Vault%20trial%20invite)

---

### Basic — $10 / month ($100 / year)

For everyday individual and developer use.

- 50 GiB storage
- 128 MiB max object
- 2,000,000 monthly reads
- 1,000,000 monthly writes
- 50 GiB monthly egress
- 2 req/sec reads and writes

---

### High — $16 / month ($160 / year)

For heavier automation, more of everything.

- 250 GiB storage
- 256 MiB max object
- 5,000,000 monthly reads
- 2,500,000 monthly writes
- 250 GiB monthly egress
- 4 req/sec reads and writes

---

## Boundaries

All tiers use the same product, API, CLI, client-side encryption model, Witness receipt model, and Continuity ledger model.

Tiers are limits, not feature bundles. If included usage is exceeded, requests may be throttled or rejected with stable machine-readable errors.

Trial is invite-based and limited to 50 active Trial accounts. Paid accounts are unaffected by Trial capacity.

No open-ended storage, open-ended bandwidth, priority support, managed key recovery, or custody of plaintext, filenames, or encryption keys.

After payment, a claim-code email tells you how to run `vault account claim --code <code>`. Basic and High expiry includes a 30-day read-only grace window for object GET, object HEAD, continuity export, and account status.
