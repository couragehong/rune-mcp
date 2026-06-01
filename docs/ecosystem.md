# The rune ecosystem

`rune-mcp` is one piece of a larger system. The names are close, so here is the
distinction up front.

| Component | What it is | Role |
|---|---|---|
| **`rune`** | A Claude Code plugin and CLI, installed at `~/.rune/bin/rune`. | The umbrella product. Installs the toolchain, registers `rune-mcp` as an MCP server, and launches the embedding daemon (`rune runed`). |
| **`runed`** | The **rune d**aemon — a separate Go module, [`CryptoLabInc/runed`](https://github.com/CryptoLabInc/runed). | Keeps the embedding model resident and answers embedding requests over a Unix socket. One shared instance per machine. |
| **`rune-mcp`** | **This repository.** | The per-session MCP server. One process per agent window. A *client* of `runed`, Vault, and envector. |

The `d` in `runed` is the usual Unix convention for a daemon: it is the long-lived
**rune** process that holds the model, distinct from the short-lived `rune-mcp`
sessions that talk to it.

## How they fit together

- The **`rune` CLI** is what a user installs. Its plugin manifest registers the
  `rune-mcp` binary as an MCP server, so Claude Code knows to launch it.
- **Claude Code spawns one `rune-mcp` process per session** over stdio. It lives
  as long as the window is open and exits on stdio EOF.
- `rune-mcp` needs embeddings, so it talks to **`runed`** over a Unix socket
  (`~/.runed/embedding.sock`). The model lives in `runed`, never inside
  `rune-mcp` — that is why many concurrent sessions don't multiply the model's
  memory cost.
- If the `runed` socket isn't up, `rune-mcp` **auto-spawns the daemon** by
  exec'ing `rune runed --detach`, guarded by a file lock (`~/.runed/spawn.lock`)
  so concurrent sessions don't race to start it. The coordinator probes the
  socket, starts the daemon only if needed, and waits up to 15s for it to become
  reachable.

## Backing services

Two services sit behind the trio. They are *not* part of the `rune` family, but
`rune-mcp` is a client of both:

- **Vault** ([`CryptoLabInc/rune-admin`](https://github.com/CryptoLabInc/rune-admin)) —
  brokers FHE keys and performs all decryption. The FHE **secret key lives here
  and only here**; `rune-mcp` receives only the public encryption key. `rune-mcp`
  reaches it over gRPC (`GetAgentManifest`, `DecryptScores`, `DecryptMetadata`).
- **envector** ([`CryptoLabInc/envector-go-sdk`](https://github.com/CryptoLabInc/envector-go-sdk)) —
  the FHE vector database that stores and scores encrypted vectors. Scoring
  happens under encryption; the encrypted scores come back to `rune-mcp`, which
  asks Vault to decrypt them.

See [architecture.md](architecture.md) for how these processes cooperate during
boot and during a capture or recall, and [security.md](security.md) for the key
model.
