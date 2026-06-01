# Configuration

`rune-mcp` keeps its state under `~/.rune` and talks to the embedder under
`~/.runed`. The only file you normally edit is `config.json`; everything else is
created and managed for you.

## `config.json`

Lives at `~/.rune/config.json`. The `configure` tool writes it, or you can edit
it by hand:

```json
{
  "vault": {
    "endpoint": "tcp://vault.example.com:8200",
    "token": "<your-vault-token>",
    "ca_cert_path": "",
    "tls_disable": false
  },
  "state": "active"
}
```

| Field | Required | Meaning |
|---|:---:|---|
| `vault.endpoint` | yes | Vault address. Accepts `tcp://host:port`, `http(s)://…`, a bare `host:port`, or a bare `host`. |
| `vault.token` | yes | The Vault access token. |
| `vault.ca_cert_path` | no | Path to a CA certificate for TLS verification. |
| `vault.tls_disable` | no | Disable TLS (plaintext gRPC). Default `false`. |
| `state` | no | `active` to boot the pipelines; `dormant` to stay deactivated until `activate`. |

Other credentials — the envector connection and the FHE keys — are fetched from
Vault at boot and kept in memory only. They are never written to `config.json`.

## File layout

```
~/.rune/
  config.json              Vault endpoint, token, TLS, state
  keys/<keyID>/EncKey.json the public FHE encryption key (0600)
  capture_log.jsonl        append-only local record of captures
  logs/rune-mcp.log        optional log tee (see RUNE_MCP_LOG_FILE)
  logs/boot.log            boot-failure log (rotates at 1 MiB)
~/.runed/
  embedding.sock           the runed daemon's Unix socket
  spawn.lock               cross-session lock guarding daemon startup
```

Directories are created with mode `0700` and files with `0600`.

## Environment variables

| Variable | Effect |
|---|---|
| `RUNE_MCP_LOG_FILE` | Unset → log to stderr only. Empty → also tee to `~/.rune/logs/rune-mcp.log`. A path → tee to that path. (Failures to open the file are non-fatal; logging falls back to stderr.) |
| `RUNE_HOME` | Overrides `~/.rune` when resolving the `rune` binary (`$RUNE_HOME/bin/rune`). |
| `CLAUDE_PLUGIN_ROOT` | Fallback location for the `rune` binary (`$CLAUDE_PLUGIN_ROOT/bin/rune`). |

The `rune` binary is resolved in order: `$RUNE_HOME/bin/rune`, then
`$CLAUDE_PLUGIN_ROOT/bin/rune`, then `rune` on `PATH`. It is used to auto-spawn
the `runed` daemon — see [ecosystem.md](ecosystem.md).
