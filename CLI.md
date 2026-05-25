# CLI Reference

All commands, global flags, exit codes, and pipeline patterns.

Version: client 0.16.10 / API v1. Global flags must appear before the command name.

AuroraVault is currently in pre-1.0 development. API v1, receipt format v1, and
Continuity ledger v1 are stable; CLI and operational behavior may still evolve.

---

## Global flags

| Flag | Description |
|------|-------------|
| `--machine` | Stable `key=value` output for supported commands. Implies quiet. Unsupported commands fail explicitly with `unsupported_machine_mode`. |
| `--quiet` | Suppress human text. Also forbids interactive prompts. |
| `--no-color` | Disable ANSI colors. |
| `--debug` | Print tracebacks and HTTP bodies on error. |
| `--home <name>` | Use a named home for this invocation only. |

---

## Commands

| Command | Purpose |
|---------|---------|
| `vault setup` | Interactive first-time setup wizard. |
| `vault init` | Generate a local keypair in the active home. |
| `vault home` | List, switch, or inspect registered homes. |
| `vault put` | Encrypt, upload, and obtain a signed receipt. |
| `vault get` | Download and decrypt an object. |
| `vault info` | Show metadata for an object or stream. |
| `vault list` | List stored objects. |
| `vault receipt` | Show receipt fields. |
| `vault verify` | Verify a receipt or file. |
| `vault delete` | Delete an object. |
| `vault status` | Check connectivity, auth, keys, and version state. |
| `vault update` | Show current client version and the unavailable self-update placeholder. |
| `vault config` | Read and write global configuration. |
| `vault auth` | Account onboarding, status, and token management. |
| `vault account` | _(Disabled — use `vault auth`)_ |
| `vault witness retry` | Retry a missing receipt request. |
| `vault witness pubkey` | Fetch the server witness public key. |
| `vault uninstall` | Remove local Vault state; optionally purge server objects or revoke token. |
| `vault ledger` | Show ledger entries. |
| `vault sync` | Check server-side existence for local records. |
| `vault export` | Decrypt and export all objects. |
| `vault tree` | Show the local state tree. |
| `vault stream` | Continuous verifiable capture. |
| `vault admin *` | Operator-only commands. Require an admin secret. |

---

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Runtime failure (network, server, decrypt) |
| `2` | Usage error (bad args, missing config, no key record) |
| `4` | Verification failure (signature, hash, ledger) |
| `5` | Local I/O failure |
| `6` | Stream lock conflict — another vault stream is already running for this stream |
| `7` | Stream not running — `vault stream stop` / `vault stream status` only |
| `8` | Stream child process failed — `vault stream start --process/--stdout/--stderr` only |
| `10` | Stored, but receipt request failed — run `vault witness retry` |

---

## Examples

```sh
vault setup
vault init --home staging --path /srv/vault-staging --activate
vault home list
vault config set api_url https://vault.auroranode.com
vault --machine --quiet status
vault auth status
vault auth create --trial
vault auth create --code clm_your_claim_code
vault put contract.pdf
tar czf - evidence/ | vault put --stdin --name "quarterly-evidence-2026-q1.tar.gz"
cat dump.sql | vault put --stdin --name "db-dump-2026-05-02"
vault get 3a7f9c8b... --stdout | sha256sum
vault verify contract.pdf
vault witness retry 3a7f9c8b...
vault ledger
vault --machine ledger
```

---

## vault setup

Interactive setup wizard. First-time interactive wizard. Generates keys, sets server URL and token. Safe to re-run.

```sh
vault setup
```

---

## vault init

Non-interactive key generation.

```sh
vault init                                         # initialize default home
vault init --home staging --path /path/to/dir      # create named home
vault init --force                                 # regenerate keys (destructive)
```

---

## vault home

Manage named homes.

```sh
vault home list           # list registered homes; active marked
vault home use <name>     # switch active home
vault home path           # print path of active home
```

---

## vault config

Configuration management.

```sh
vault config get                              # show config (token masked)
vault config set api_url <url>               # set server URL
vault config set-token --token vlt_…        # store bearer token (hidden)
```

`vault config set-token` may prompt only in human interactive mode. Under
`--machine` or `--quiet`, `--token` is required.

---

## vault auth

Account onboarding, status, and token management.

```sh
vault auth create --trial --invite ivt_…       # Trial account via invite code
vault auth create --trial                      # Trial account via invite code (prompted)
vault auth create --code clm_…                 # Paid account via claim code
vault auth create                              # Paid account via claim code (prompted)
vault auth claim --code clm_…                  # low-level escape hatch (same as auth create --code)
vault auth status                              # plan, quota, and monthly usage
vault auth logout                              # remove locally saved token
```

Invite and claim codes may be prompted in human interactive mode. Under
`--machine` or `--quiet`, they must be supplied explicitly.
Paid tier (Basic or High) is resolved server-side from the claim code; the CLI
does not select it. The removed `--basic` and `--high` flags fail immediately
with an explicit message pointing to `vault auth create --code clm_...`.

---

## vault account _(disabled)_

`vault account` is disabled and hidden from root help. Use `vault auth` instead.
Old scripts fail with exit code 2 and a replacement hint.

| Old | Preferred |
|-----|-----------|
| `vault account create --invite ivt_…` | `vault auth create --trial --invite ivt_…` |
| `vault account claim --code clm_…` | `vault auth create --code clm_…` |
| `vault account --status` | `vault auth status` |

---

## vault status

Check connectivity and configuration.

```sh
vault status            # server / auth / keys / version / home rows
vault status --verbose  # adds fingerprint and url
vault --machine status  # key=value output
```

---

## vault put

Encrypt, upload, get a signed receipt.

Three identifiers per put: `object_id` (SHA-256 of ciphertext), `put_id` (unique event ID), `content_commitment` (binds receipt to file version without revealing the plaintext hash to the server). Exit `10` = stored but receipt failed — run `vault witness retry <object_id>`.

```sh
vault put <file>
vault put --stdin --name "db-dump"       # read from stdin and store a local display name
```

---

## vault get

Download and decrypt.

```sh
vault get 1                                       # by short index — exits 2 without --out or --stdout
vault get <object_id> --out /tmp/file             # explicit output path
vault get <object_id> --out /tmp/file --force     # overwrite existing file
vault get <object_id> --stdout                    # stream to stdout (diagnostics → stderr)
vault get stream s001 --idx 000003 --out file     # restore a stream event
vault get stream s001 --event <event_id> --stdout # stream event by event ID
```

Ordinary object restore rejects stream-only flags: `--idx`, `--event`,
`--extract`, `--all`, and `--concat`. Stream restore rejects `--idx` with
`--event`, and rejects `--all` with `--concat`.

---

## vault list

List stored objects.

```sh
vault list               # active objects
vault list --objects     # explicit object-only view
vault list --all         # all lifecycle states + STATUS column
vault list --deleted     # deleted objects + deleted stream events (for offline receipt audit)
vault list --streams     # streams instead of objects
```

`--objects`, `--streams`, `--all`, and `--deleted` are mutually exclusive.

---

## vault info

Show object metadata.

```sh
vault info 1             # by short index
vault info <object_id>   # by full ID or prefix
vault info s001          # stream by s-index
```

---

## vault receipt

Show receipt fields. Inspection only. Use `vault verify` for cryptographic verification.

```sh
vault receipt 1              # by index
vault receipt <object_id>    # by full ID
```

---

## vault verify

Verify a receipt or file.

Receipt mode verifies the receipt hash, Ed25519 signature, and local proof block when present. It can run fully offline with `--pubkey`. File mode verifies local proof material and requires Continuity ledger anchoring. Exit `4` = verification failed. Exit `5` = unreadable file.

```sh
vault verify <receipt.json>                          # verify signature
vault verify <receipt.json> --pubkey server.pub      # offline, no network
vault verify <file>                                  # verify file against proof + ledger
```

---

## vault delete

Delete an object. Ciphertext removed from server, key material blanked, receipt and metadata kept.

```sh
vault delete 1              # exits 2 without --confirm
vault delete 1 --confirm    # execute deletion
```

---

## vault witness

Server key and receipt recovery.

```sh
vault witness pubkey              # fetch and save server public key
vault witness pubkey --no-save    # print base64 instead
vault witness retry <object_id>   # request missing receipt (after exit 10)
```

By default the public key is saved to `<active_home>/server.pub`.

---

## vault uninstall

Remove local Vault data. Optionally purge server-side objects and revoke the active bearer token before wiping local state.

```sh
vault uninstall                                              # exits 2 without --confirm
vault uninstall --confirm                                    # wipe local state only
vault uninstall --purge-server --confirm                     # delete all server objects first
vault uninstall --revoke-token --confirm                     # revoke token, then wipe local state
vault uninstall --purge-server --revoke-token --confirm      # full cleanup
```

`--purge-server` deletes each stored object from the server before wiping local state. Non-fatal server errors are printed as warnings.
`--revoke-token` rotates the bearer token server-side (making the current token invalid), then removes the local token file. Exits 1 if revocation fails.

---

## vault ledger

Show ledger entries. Fetches and displays Continuity ledger entries for the active account. This is not a standalone offline chain verifier.

```sh
vault ledger                  # show ledger entries (fetches live)
vault --machine ledger        # key=value per entry
```

---

## vault sync

Check server-side existence.

```sh
vault sync            # check server for all local records
vault sync --verbose  # include present objects
```

---

## vault export

Decrypt and export all objects. Exports named local objects by default. Unnamed stdin objects are skipped unless exported through stream mode. `--stream` decrypts and restores plaintext stream contents.

```sh
vault export /tmp/export-dir              # exits 2 without --confirm
vault export /tmp/export-dir --confirm    # execute
vault export /tmp/export-dir --latest --confirm
vault export /tmp/export-dir --stream s002 --confirm
```

---

## vault tree

Local state tree.

```sh
vault tree    # local state tree (no network)
```

---

## vault stream

Continuous verifiable capture. Each event is a normal Vault object with its own Witness receipt, linked into a client-managed per-stream hash chain. The server does not understand streams.

```sh
vault stream start --target <file-or-dir> --name evidence  # snapshot now, then watch
vault stream start --target <file> --interval 600      # snapshot every 10 minutes
vault stream start --stdin                             # from stdin (line-buffered)
vault stream start --process -- <command>              # stdout+stderr from a process
vault stream list                                # active streams
vault stream list --deleted                      # tombstoned streams only
vault stream list --all                          # all streams
vault stream status <stream_id>                  # exit 7 if stopped
vault stream stop <stream_id>
vault stream delete <stream_id> --confirm        # delete server objects; marks deleted, ledger kept
vault stream delete <stream_id> --confirm --purge-receipts  # also remove local receipts
vault stream chain <stream_id>                  # proof / continuity / metadata JSON only
vault stream report <stream_id>                  # human summary + chain check
vault get stream <stream_id>                     # restore latest snapshot
vault get stream <stream_id> --idx 000003        # restore specific event
```

`vault stream start` is the canonical start command. The older
`vault stream --target ...`, `vault stream --stdin`, and
`vault stream --process ...` forms remain as hidden compatibility aliases.
`vault stream chain` does not export plaintext stream contents; use
`vault export --stream <id> --confirm` or `vault get stream <id>` for
plaintext restore.

---

## vault admin

Operator-only. Requires `VAULT_ADMIN_SECRET` env var or `--secret`. Admin subcommands are not documented publicly.

```sh
vault admin *  # requires admin secret
```

---

## Pipeline patterns

```sh
# Put from stdin
cat dump.sql | vault put --stdin --name "db-dump-2026-05"
tar czf - /etc | vault put --stdin --name "etc-snapshot"

# Get to stdout
vault get <id> --stdout | sha256sum
vault get <id> --stdout | tar xz -C /restore/

# Script-safe
OID=$(vault --machine --quiet put file.bin | grep ^object_id= | cut -d= -f2)
vault --machine status | grep -q "auth_valid=true" || exit 1

# Verify all local receipts offline
find <home>/receipts -name "*.witness.json" | \
    xargs -I{} vault --machine verify "{}" --pubkey <home>/server.pub | \
    grep "status=error"   # empty = all verified
```
