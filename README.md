<div align="center">

# rune-mcp

**FHE-encrypted organizational memory for AI agents.**

Capture and recall the decisions your team makes — without any of it ever
being exposed in plaintext.

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Go](https://img.shields.io/badge/go-1.26%2B-00ADD8?logo=go&logoColor=white)](go.mod)
[![Status](https://img.shields.io/badge/status-alpha-orange.svg)](#status)
[![MCP](https://img.shields.io/badge/protocol-MCP-7C3AED.svg)](https://modelcontextprotocol.io)

[**Ecosystem**](docs/ecosystem.md) ·
[**Architecture**](docs/architecture.md) ·
[**Quick start**](#quick-start) ·
[**Docs**](#documentation)

</div>

---

`rune-mcp` is a per-session [MCP](https://modelcontextprotocol.io) server that
gives an AI agent (e.g. Claude Code) a long-term, **encrypted** memory of the
decisions an organization makes. The agent does the language work — extracting a
decision from a conversation, understanding a recall query — and `rune-mcp` owns
the contracts around it: schema validation, embedding, novelty scoring,
reranking, encryption, and durable storage.

Everything is stored and searched under **fully homomorphic encryption (FHE)**.
Vectors are scored while encrypted, the FHE secret key never reaches this
process, and decryption is delegated to a separate **Vault** service. The result
is a searchable decision history that an agent can read and write while the data
stays encrypted end to end.

## Highlights

- **Agent-delegated** — the agent owns the LLM work; `rune-mcp` owns the
  deterministic contracts (schema, novelty, reranking, encryption).
- **Encrypted end to end** — vectors are stored and scored under FHE; the secret
  key stays in Vault and only the public encryption key reaches the client.
- **Native MCP** — ten tools over stdio JSON-RPC; drops into any MCP client.
- **Session-isolated, model-free** — one lightweight process per session, with
  the embedding model shared out-of-process, so memory doesn't balloon as
  sessions multiply.

## How it works

`rune-mcp` is one of three cooperating processes. The design exists to solve a
single problem: **don't replicate the embedding model per session.**

```
  [ user machine ]

  Claude window A ──stdio──► rune-mcp A ─┐
  Claude window B ──stdio──► rune-mcp B ─┤   one process per session
  Claude window C ──stdio──► rune-mcp C ─┘
                                 │
              auto-spawns if down │  each rune-mcp also holds its own
              (`rune runed`)      │  gRPC links to Vault and envector
                                 ▼
                        ┌──────────────────┐
                        │  runed           │  shared, one per machine —
                        │  (embedder       │  the embedding model lives
                        │   daemon)        │  here, never in rune-mcp
                        └──────────────────┘
            │                                        │
            ▼                                        ▼
    ┌───────────────┐                        ┌───────────────┐
    │  Vault (gRPC) │                        │  envector     │
    │  key broker + │                        │  FHE vector   │
    │  FHE decrypt  │                        │  database     │
    └───────────────┘                        └───────────────┘
```

- **`rune-mcp`** (this repo) — one process per agent session, spawned over stdio.
  Holds the connections and in-memory keys; holds **no model**.
- **`runed`** — a shared daemon that keeps the embedding model resident and serves
  embeddings over a Unix socket. `rune-mcp` is only a client and starts it on
  demand if it isn't running.
- **Vault + envector** — Vault brokers FHE keys and performs decryption; envector
  stores and scores encrypted vectors. The secret key never reaches `rune-mcp`.

For the three-way naming (`rune` / `runed` / `rune-mcp`), see
[docs/ecosystem.md](docs/ecosystem.md); for the boot state machine and the
capture/recall flows, see [docs/architecture.md](docs/architecture.md).

## Tools

An agent talks to `rune-mcp` over stdio JSON-RPC and gets ten tools:

| Tool | Kind | Gated | Purpose |
|---|---|:---:|---|
| `capture` | write | yes | Store one decision record (the agent supplies the extraction). |
| `batch_capture` | write | yes | Store many records at once (e.g. an end-of-session sweep). |
| `recall` | read | yes | Answer a natural-language question against stored memory. |
| `delete_capture` | write | yes | Soft-delete a record by ID (sets `status=reverted`, re-inserts). |
| `capture_history` | read | no | List recent captures from the local append log. |
| `vault_status` | read | no | Probe Vault connectivity and report secure-search mode. |
| `diagnostics` | read | no | 7-section health snapshot (env, state, vault, keys, pipelines, embedding, envector). |
| `configure` | action | no | Write Vault credentials to `~/.rune/config.json` and mark `state=active`. |
| `activate` | action | no | Pre-check config + daemon, then drive `reload_pipelines` to completion. |
| `reload_pipelines` | action | no | Re-initialize the Vault + envector pipelines (boot replay). |

Gated tools return `PIPELINE_NOT_READY` until the server reaches `active`.
`recall` is read-only but needs live pipelines, so it is gated; the lifecycle
actions deliberately bypass the gate so they can *drive* recovery.

## Quick start

**Requirements:** Go 1.26+.

```sh
git clone https://github.com/CryptoLabInc/rune-mcp
cd rune-mcp
go build ./...     # build
go test ./...      # test
go vet ./...       # vet
```

The build is self-contained (no `replace` directives or vendoring). **Running**,
however, needs the `runed` daemon, a reachable **Vault**, and an **envector**
backend — in normal use the `rune` CLI installs and wires these for you.

**Configure.** Point `rune-mcp` at a Vault by writing `~/.rune/config.json` (or
let the `configure` tool do it):

```json
{
  "vault": {
    "endpoint": "tcp://vault.example.com:8200",
    "token": "<your-vault-token>"
  },
  "state": "active"
}
```

**Run.** You don't launch `rune-mcp` by hand: an MCP client (Claude Code) spawns
one instance per session over stdio. Register the built binary as an MCP server
in your agent. On boot it moves through `starting → waiting_for_vault → active`;
until Vault answers, write tools return `PIPELINE_NOT_READY` and read tools
degrade gracefully.

Full options (endpoint forms, TLS, file layout, environment variables) are in
[docs/configuration.md](docs/configuration.md).

## Documentation

| Document | What's inside |
|---|---|
| [docs/ecosystem.md](docs/ecosystem.md) | How `rune`, `runed`, and `rune-mcp` relate, plus the Vault and envector backing services. |
| [docs/architecture.md](docs/architecture.md) | Why three processes, the boot state machine, and the capture/recall flows. |
| [docs/file-structure.md](docs/file-structure.md) | The directory layout, the package dependency rules, and a per-file map. |
| [docs/configuration.md](docs/configuration.md) | `config.json`, the `~/.rune` and `~/.runed` layout, and environment variables. |
| [docs/security.md](docs/security.md) | The security model, what stays encrypted, and known limitations. |

For exact type signatures, browse the package doc comments with `go doc ./internal/...`.

## Security

`rune-mcp` is built so that **obscurity is never the defense**: the FHE secret
key stays in Vault, only the public `EncKey` reaches the client, metadata is
sealed in an AES envelope, and logs are redacted. The full model and its known
limitations are in [docs/security.md](docs/security.md). Please report
vulnerabilities **privately**, not via public issues.

## Contributing

Contributions are very welcome — `rune-mcp` is young and there's plenty of room
to help. Before opening a PR:

```sh
go build ./... && go vet ./... && go test ./...
gofmt -l .          # should print nothing
```

Please respect the package dependency direction described in
[docs/file-structure.md](docs/file-structure.md): `domain` is a leaf, `policy`
does no I/O, and `service` is the only layer that composes adapters with policy.

## Status

`0.1.0-alpha`. The contracts described here are stable; build ergonomics and
operational tooling are still settling.

## License

[Apache License 2.0](LICENSE).
