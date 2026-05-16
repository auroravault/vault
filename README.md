# Aurora Vault

Encrypted object storage with signed receipts and an append-only continuity ledger.

---

You hope nobody asks *how you know* they haven't been touched.

Config exports, build artifacts, vendor API responses, the output of a script you ran once and might need to defend later. They live in folders with timestamps in the filename. The timestamp is the upload time. Nothing proves the content.

Aurora Vault is the boring layer underneath that.

Files are encrypted on your machine before they leave it. The server stores opaque ciphertext — no filenames, no plaintext, no keys, no opinions about what you sent. Each upload receives a signed Ed25519 receipt and is added to an append-only hash-chain ledger. Later, from any machine, without contacting the server, you can verify the receipt signature and confirm ledger anchoring with a public key.

---

## It speaks pipe

Vault reads from stdin. Anything your shell can produce, it can preserve.

```sh
terraform show -json | vault put --stdin --name "tf-$(date +%F)"
kubectl get all -A -o yaml | vault put --stdin --name "k8s-$(date +%F)"
curl -s https://api.vendor.com/export | vault put --stdin --name "vendor-$(date +%F)"
```

Add those to cron and you have a verifiable record of how three systems looked over time. No agent. No SDK. No platform to adopt. Just one more verb at the end of a pipe.

---

## Who it's for

Operators who want evidence that survives a vendor change. Researchers and engineers who need to prove a result existed before a publication or a decision. Builders who already work in a terminal and prefer infrastructure they can reason about in an evening.

If you want a dashboard, a sales call, or someone to manage your keys, this isn't for you. That's deliberate.

---

## What it does not do

No browser drive. No sync client. No shared folders. No analysis of your data. No managed platform trying to grow into your stack. If we disappear tomorrow, your local receipts, keys, and any ciphertext you have exported or downloaded still make sense without us.

---

## Try it

```sh
$ vault setup
$ vault put myfile.txt
$ vault witness pubkey
$ vault verify <home>/receipts/<object_id>.witness.json --pubkey <home>/server.pub
```

---

## Documentation

| File | Description |
|------|-------------|
| [QUICKSTART.md](QUICKSTART.md) | Install, configure, store a file, verify a receipt |
| [CLI.md](CLI.md) | All commands, global flags, exit codes, pipeline patterns |
| [TECHNICAL.md](TECHNICAL.md) | Encryption model, receipt format, ledger format, verification |
| [SECURITY.md](SECURITY.md) | What the server sees, what it doesn't, accepted risks |
| [TRUST.md](TRUST.md) | Compatibility posture, known constraints |
| [USE_CASES.md](USE_CASES.md) | Evidence retention, artifact verification, AI provenance, and more |
| [PRICING.md](PRICING.md) | Tiers and hard usage boundaries |
| [TERMS.md](TERMS.md) | Terms of service |
| [RECEIPT_FORMAT.md](RECEIPT_FORMAT.md) | Witness receipt v1 specification (frozen) |
| [LEDGER_FORMAT.md](LEDGER_FORMAT.md) | Continuity ledger v1 specification (frozen) |

---

**vault@auroranode.com** · [auroranode.com/vault](https://auroranode.com/vault)
