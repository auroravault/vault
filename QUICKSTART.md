# Quickstart

Install, configure, store a file, get a signed receipt, and verify what the receipt proves.

---

## Prerequisites

- Linux or macOS
- A server URL
- A Trial invite code or paid account claim code

---

## Step 1 — Install

Install with pipx (recommended) or pip, then confirm the CLI is available.

```sh
pipx install auroravault
vault update
```

---

## Step 2 — Set up

Interactive wizard — generates your local Ed25519 keypair, sets the server URL, and stores your bearer token.

```sh
vault setup
```

Or non-interactive:

```sh
vault init
vault config set api_url https://vault.auroranode.com
vault config set-token --token vlt_your_token_here
```

---

## Step 3 — Check connectivity

```sh
$ vault status

  server       ● reachable
  auth         ● valid
  keys         ● present
  version      client 1.13.5
  home         default  ~/.local/share/auroravault/homes/default
```

If auth shows invalid, re-run `vault config set-token`.

---

## Step 4 — Store a file

```sh
$ vault put myfile.txt

  stored   ✓
  proof    ✓

  id       3a7f9c8bd1e2f4a5…
  receipt  ~/.local/share/auroravault/homes/default/receipts/3a7f9c8b….witness.json
```

The file is encrypted on your machine before upload. The server receives ciphertext only.

You now have three things: an encrypted object on the server addressed by its SHA-256 hash, a receipt file at the path shown above containing the server's Ed25519 signature over a commitment to your file, and a keystore entry holding the AES-256-GCM key needed to decrypt it. None of these existed a moment ago. The receipt is the proof.

---

## Step 5 — List what you have

```sh
vault list
```

---

## Step 6 — Retrieve and decrypt

```sh
vault get <object_id> --out recovered.txt
```

By short index:

```sh
vault get 1 --out recovered.txt
```

---

## Step 7 — Verify a receipt

Cache the server public key once, then verify the receipt signature offline:

```sh
$ vault witness pubkey
$ vault verify <receipt.witness.json> --pubkey ~/path_to_active_home/server.pub

  signature  ✓
```

No server contact is required for receipt-mode verification once the public key is cached locally. File verification with `vault verify <file>` also checks Continuity anchoring by fetching the current ledger.

What `signature ✓` means: the receipt was signed by the server's Ed25519 private key. The signature can be verified by anyone holding the public key — including you, offline, in three years, without this service running. The server cannot retroactively alter the receipt. The content commitment in the receipt binds the signature to your specific file version without revealing the plaintext hash to the server.
