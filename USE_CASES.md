# Use Cases

Each case below starts with the situation before vault. The problem is always the same: something exists, but nothing proves it, and you are the only witness.

---

## 01 — Evidence retention

Before: an incident report sits in a folder. The file modification time is whatever it was when you last saved it. If anyone asks when it was created or whether it has been altered, you have no answer that doesn't depend on trusting you.

After: a signed receipt proves the exact byte content was committed at a specific time. The continuity ledger proves receipt ordering and makes silent removal or insertion detectable. The original content can stay private.

```sh
vault put incident-report-2026-05.pdf
vault verify incident-report-2026-05.pdf   # verify later against ledger
```

---

## 02 — Artifact verification

Before: a build output sits in S3. The timestamp is the upload time. Nothing proves what the bytes were when the job ran, or that the object hasn't been overwritten since.

After: a receipt is issued at upload time, tied to the SHA-256 of the ciphertext. The object ID is deterministic — the same bytes always produce the same ID. Proving a specific binary existed at a specific pipeline step is a receipt check.

```sh
vault put build-output-v3.tar.gz
vault --machine put build-output-v3.tar.gz | grep ^object_id=   # capture ID for CI
```

---

## 03 — AI provenance

Before: a model was trained on "version 3 of the dataset." Version 3 is a directory. Nobody wrote down its exact contents at training time. Reproducing the result requires trusting that nothing has changed.

After: the dataset snapshot is committed before training begins. The receipt proves which exact bytes were present. The content commitment hides the dataset hash from the storage server while still binding the receipt to that specific version.

```sh
tar czf - ./dataset/ | vault put --stdin --name "training-data-2026-05"
vault verify dataset-receipt.json
```

---

## 04 — Compliance evidence chains

Before: controls exports and audit artifacts live in a shared drive. The audit trail is a spreadsheet. Proving what was submitted when, and that nothing was added retroactively, requires trusting whoever manages the drive.

After: each artifact is committed with a receipt. Receipts are anchored into the continuity ledger in order. The ledger is a hash chain — inserting, removing, or reordering entries breaks the chain. Sequence proof is structural, not administrative.

```sh
vault put controls-export-2026-q1.json
vault verify controls-export-2026-q1.json   # verify file proof and ledger anchoring
vault ledger                                # show Continuity entries
```

---

## 05 — Continuous capture with vault stream

Before: a log file is rotated nightly. A config directory is git-committed whenever someone remembers. The history exists in fragments across different systems. Reconstructing what a file looked like at 14:30 on a specific date is archaeology.

After: vault stream snapshots the target on an interval. Each snapshot is a full vault put — encrypted, receipted, ledger-anchored. The stream builds a per-stream hash chain across all snapshots. Any specific version is retrievable by index.

```sh
vault stream --target ~/audit.log --interval 300          # snapshot every 5 min
vault stream --target ./project --interval 3600 --background   # hourly dir snapshot
vault stream report <stream_id>    # integrity summary
vault get stream <stream_id>       # restore latest snapshot
```

---

## 06 — Encrypted archival copies

Before: sensitive files are backed up to cloud storage. The provider can read them. If the provider is subpoenaed, acquired, or breached, the content is exposed. Encrypting before upload means managing keys separately with no standard workflow.

After: vault handles client-side encryption, key storage, and receipt issuance in one command. The provider sees only opaque ciphertext addressed by SHA-256. The decryption key never leaves your machine.

```sh
vault put keydata.tar.gz
vault get <object_id> --stdout | sha256sum   # confirm decrypted output matches expected hash
```

---

## 07 — Research archives

Before: a preprint is posted with a date. The date is whatever the server says. If a priority dispute arises, you have a timestamp and a claim. You do not have independent proof that the file existed in that form before the dispute began.

After: a receipt is issued at commit time, signed by a server key whose public key is independently verifiable. The receipt can be shared with reviewers or a journal. The content stays encrypted — the proof does not require disclosure.

```sh
vault put preprint-draft-001.pdf
vault receipt <object_id>   # show timestamped receipt for reviewers
```

---

## 08 — Pipeline integration (machine mode)

Before: CI stores artifacts to S3 with no receipt. Cron jobs run and produce output that disappears into logs. Proving what ran, when, and with what result means parsing log files and hoping the timestamps are trustworthy.

After: vault's machine mode emits stable `key=value` output on stdout. Scripts capture the object ID and receipt path. Standard UNIX tools compose with it directly. Scripts should ignore unknown future keys — the output contract is stable and additive.

```sh
# Capture and verify in CI
OID=$(vault --machine --quiet put build.tar.gz | grep ^object_id= | cut -d= -f2)
vault --machine verify <home>/receipts/${OID}.witness.json | grep -q "verified=true"

# Cron: daily snapshot with ledger check
vault stream --target ./data --interval 86400 --name daily-data
vault stream report s001
```

---

[Ready to start? → Quickstart](QUICKSTART.md)
