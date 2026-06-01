# Security model

`rune-mcp` is built so that **obscurity is never the defense** — the design is
meant to be inspected.

## What stays encrypted

- **The FHE secret key never reaches `rune-mcp`.** It lives only in Vault, which
  performs all score and metadata decryption on the client's behalf. The client
  persists only the public `EncKey` (under `~/.rune/keys/<keyID>/`, mode `0600`).
- **Vectors are stored and scored under FHE.** envector never sees plaintext
  vectors; scoring happens on ciphertext, and the encrypted scores are returned to
  `rune-mcp`, which asks Vault to decrypt them.
- **Metadata is sealed** in an AES-256-CTR envelope before it leaves the process.
  The per-agent data-encryption key (DEK) is fetched from Vault at boot, held in
  memory only, and **zeroized on shutdown**.
- **Logs are redacted by default.** Every log path runs through a `slog` handler
  that masks secret-shaped values (token-like strings and labelled secrets), so
  leaking via stderr is treated as seriously as leaking via a file.

## Known limitations

- **No metadata MAC.** The AES-256-CTR envelope currently provides confidentiality
  but **not integrity** — the ciphertext is malleable. Authenticated encryption is
  a tracked follow-up.
- **Alpha maturity.** Treat the contracts as stable and the operational hardening
  (TLS defaults, key rotation ergonomics, audit logging) as still in progress.

These are deliberate, documented gaps rather than hidden ones; the point of
stating them is that the security of the system must not depend on them being
secret.

## Reporting a vulnerability

Please report vulnerabilities **privately** — for example via a GitHub Security
Advisory on this repository — and **not** through public issues or pull requests.
Include enough detail to reproduce, and allow time for a fix before any public
disclosure.
