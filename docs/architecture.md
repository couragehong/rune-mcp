# Architecture

`rune-mcp` exists to solve one problem cleanly: **don't replicate the embedding
model per session.** Each agent window gets its own cheap, stateless bridge; the
expensive model is shared out-of-process in the `runed` daemon. See
[ecosystem.md](ecosystem.md) for how the processes relate; this document covers
how they behave at runtime.

## The three processes

- **`rune-mcp`** (one per session) — spawned by the agent over stdio, lives as
  long as the window is open, exits on stdio EOF. It holds the Vault and envector
  gRPC connections and the in-memory key material, reads `~/.rune/config.json`,
  and appends to a local capture log. It holds **no model**.
- **`runed`** (shared, external) — keeps the embedding model resident and answers
  embedding requests over gRPC on a Unix socket. `rune-mcp` is only a client and
  will start it on demand if it isn't already running.
- **Vault + envector** (backing services) — Vault brokers keys and performs
  decryption; envector stores and scores encrypted vectors. The secret key never
  reaches `rune-mcp` — ciphertext is sent to Vault for decryption.

## Boot state machine

`rune-mcp` answers MCP requests immediately on launch, before its pipelines are
ready, and heals in the background. A small state machine drives this:

```
  starting ──► waiting_for_vault ──► active ◄──► dormant
                    ▲                              │
                    └───── retry (backoff 1s→60s) ─┘
```

- **`starting`** — process up, services wired, adapters not yet connected.
- **`waiting_for_vault`** — retrying the Vault handshake with exponential backoff
  (starting at 1s, capped at 60s). Read tools degrade gracefully; write tools are
  gated.
- **`active`** — Vault returned the key bundle, the pipelines are live, all tools
  work.
- **`dormant`** — deactivated, either by the user or after a fatal config/Vault
  error. Re-enabled with the `activate` / `reload_pipelines` tools.

Boot failures are classified (config / TLS / Vault / network) into a recovery
hint and appended to `~/.rune/logs/boot.log`. On a transient gRPC failure, a
recovery interceptor retriggers the boot loop rather than failing hard.

State gating: the write tools (`capture`, `batch_capture`, `recall`,
`delete_capture`) return `PIPELINE_NOT_READY` with a state-specific hint until the
server is `active`. The read/diagnostic tools and the lifecycle actions
(`configure`, `activate`, `reload_pipelines`) bypass the gate so they can run
during boot and *drive* recovery.

## Data flows

Both main flows run once the server is `active`.

**capture** — store one decision the agent has already extracted:

1. Validate the request (text present, extraction parseable).
2. Embed the payload text via `runed`.
3. Run a novelty check against existing records (novel / evolution / related /
   near-duplicate, by similarity thresholds).
4. Build the `DecisionRecord` (redact PII, render the payload card).
5. Seal the metadata in an AES envelope using the per-agent key from Vault.
6. Insert the encrypted vector + sealed metadata into envector.
7. Append an entry to the local capture log.

**recall** — answer a natural-language question:

1. Parse the query into intent, time scope, keywords, and expansions.
2. Embed the expansions via `runed`.
3. Score against envector under FHE (encrypted scores come back).
4. Ask Vault to decrypt the scores and the matched metadata.
5. Classify and filter the hits (status, time, sensitivity).
6. Rerank by `(0.7·similarity + 0.3·recency) · statusMultiplier`, with a 90-day
   recency half-life.
7. Return the top hits (capped at 10).

## Design notes

- **One direction of dependency.** The Go packages are layered so that pure types
  and pure logic never depend on I/O. See [file-structure.md](file-structure.md).
- **Fail soft, heal in the background.** The server is useful (for diagnostics and
  configuration) before it is fully `active`, and recovers without a restart.
- **Encryption is delegated, not reimplemented.** `rune-mcp` orchestrates; Vault
  and envector own the cryptography. See [security.md](security.md).
