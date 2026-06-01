# File structure

A map of the codebase: the directory layout, the dependency rules the packages
follow, and a per-file summary of each package. For exact type signatures, browse
the doc comments with `go doc ./internal/...`.

## Directory layout

```
cmd/rune-mcp/      entry point: stdio server + lifecycle bootstrap
internal/          the implementation (see the package map below)
  domain/          pure types — the leaf of the dependency graph
  policy/          pure, deterministic functions — no I/O
  adapters/        external I/O: config, embedder, envector, vault, keymanager, logio
  service/         orchestration — composes adapters + policy into the flows
  mcp/             MCP SDK wiring: the 10 tools, the state gate, result shaping
  lifecycle/       boot state machine, boot logging, graceful shutdown
  recovery/        gRPC interceptor that retriggers boot on transient failures
  spawn/           auto-spawn coordinator for the runed daemon
  obs/             structured logging with sensitive-data redaction
docs/              this documentation
scripts/           release tooling
```

## Dependency direction

The Go packages follow a strict **one-directional** dependency rule. Each package
imports only packages below it; reverse imports are forbidden.

```
  cmd/rune-mcp
       │
       ▼
  internal/mcp           wraps the MCP SDK; the 10 tool handlers + state gate
       │
       ▼
  internal/service       business orchestration (capture / recall / lifecycle)
       │
   ┌───┴─────────────┬───────────────┬──────────────┐
   ▼                 ▼               ▼              ▼
 policy           adapters       lifecycle        obs
 (pure, no I/O)   (external I/O) (state machine) (logging)
   │                 │           ▲   ▲
   │                 │           │   └── recovery (gRPC interceptor)
   └────────┬────────┘           │
            ▼                     │
        internal/domain  ◄────────┘
        (pure types — leaf, imports no other internal package)
```

**Rules**

- `domain` is a leaf: it imports no other `internal/*` package (stdlib only).
- `policy` is pure: no I/O, and it never calls `adapters`.
- `service` is the only layer that composes `adapters` + `policy`; adapter errors
  are wrapped into `domain.RuneError` here, not inside the adapters.
- `mcp` handlers delegate to `service` — no business logic in the handlers.

## Packages

### `domain/` — pure types (leaf)

| File | Responsibility |
|---|---|
| `schema.go` | `DecisionRecord` v2.1, six enums (Domain ×19, Sensitivity ×3, Status ×4, Certainty ×3, ReviewState ×4, SourceType ×7), nine sub-models, ID generation, validation. `MaxTitleLen=60`, `MaxPhases=7`. |
| `extraction.go` | Agent-extraction types; parses the agent's flat JSON into an `ExtractionResult`. |
| `capture.go` | Capture wire types: `CaptureRequest`, `CaptureResponse`, `RawEvent`. |
| `query.go` | Recall types: `QueryIntent` (×8), `TimeScope` (×5), `RecallArgs`/`RecallResult`, `SearchHit`, `ParsedQuery`, `Detection`; `ExtractPayloadText`. |
| `novelty.go` | `NoveltyClass` (×4) + `NoveltyInfo` + `RelatedRecord`. |
| `errors.go` | The ten error codes, `RuneError`, and `MakeError`. |
| `logio.go` | The capture-log JSONL entry type. |
| `boot_error.go` | Structured boot failure (kind / phase / detail / hint) with a `Retryable` flag. |

### `policy/` — pure functions, no I/O

| File | Responsibility |
|---|---|
| `query.go` | Query parsing: 7 intents (28 regex), 4 time rules (19 regex), 88 stopwords, 4 tech patterns. |
| `record_builder.go` | Builds 1–7 `DecisionRecord`s from an extraction. `MaxInputChars=12_000`. |
| `payload_text.go` | Renders the Markdown "decision card" payload text. |
| `pii.go` | Redacts five PII classes (email, phone, API key, long hex, credit card). |
| `novelty.go` | `ClassifyNovelty` with thresholds `0.3 / 0.7 / 0.95`. |
| `rerank.go` | Recency-weighted rerank: `(0.7·sim + 0.3·recency) · statusMultiplier`, 90-day half-life. |
| `utf8_safe.go` | Rune-aware truncation that never splits a multi-byte codepoint. |

### `adapters/` — external I/O

| File | Responsibility |
|---|---|
| `config/loader.go` | Load `~/.rune/config.json` (vault / state / metadata sections). `DirPerm=0700`, `FilePerm=0600`. |
| `config/dormant.go` | Mark the config dormant (reason + timestamp); idempotent. |
| `embedder/client.go` | gRPC client to the `runed` daemon over a Unix socket; splits oversized batches. |
| `embedder/socket.go` | Resolve the embedder socket path (default `~/.runed/embedding.sock`). |
| `embedder/retry.go` | Retry transient gRPC calls; backoffs `[0, 500ms, 2s]`. |
| `embedder/cache.go` | Cache the embedder `Info` response (sticky, with a short error cooldown). |
| `embedder/errors.go` | Map gRPC status codes to typed embedder errors. |
| `envector/client.go` | envector-go SDK wrapper (Insert / Score / GetMetadata / …); loads the public `EncKey` only. |
| `envector/aes_ctr.go` | AES-256-CTR metadata envelope (16-byte IV, JSON `{"a","c"}`, no MAC). |
| `envector/errors.go` | Map SDK and gRPC errors to typed envector errors. |
| `vault/client.go` | gRPC client: `GetAgentManifest`, `DecryptScores`, `DecryptMetadata`; 256 MB max message. |
| `vault/endpoint.go` | Normalize the endpoint string (`tcp://` \| `http(s)://` \| `host:port` \| `host`). |
| `vault/health.go` | Tier-2 HTTP `/health` fallback probe. |
| `vault/errors.go` | Map gRPC status codes to typed vault errors + sentinels. |
| `keymanager/keys.go` | Persist the **public** `EncKey` to `~/.rune/keys/<keyID>/` (0600). The FHE secret key never appears here or on disk. |
| `logio/capture_log.go` | Append-only JSONL capture log (`~/.rune/capture_log.jsonl`); in-process mutex + cross-process `flock`. |

### `service/` — orchestration

| File | Responsibility |
|---|---|
| `capture.go` | Capture pipeline (validate → embed → score → novelty → build → seal → insert) and batch capture. |
| `recall.go` | Recall pipeline (parse → embed → score → decrypt → metadata → filter → rerank). |
| `lifecycle.go` | The lifecycle tools: `vault_status`, `diagnostics`, `capture_history`, `delete_capture`, `configure`, `activate`, `reload_pipelines`. |
| `search.go` | `SearchByID` helper shared by recall and `delete_capture`. |
| `diagnostics_classify.go` | Classify envector probe errors by gRPC code into typed hints. |
| `recovery.go` | Retry envector Insert/Score after a boot retrigger on transient failures. |

### `mcp/` — MCP SDK wiring

| File | Responsibility |
|---|---|
| `tools.go` | `Deps` injection + `Register` (binds all 10 tools); adapter-client injection; `ApplyVaultBundle`. |
| `handlers.go` | The 10 tool handlers; write tools call `CheckState`, read tools bypass it. |
| `state.go` | `CheckState` gate (`PIPELINE_NOT_READY` + per-state recovery hints) and input validation. |
| `result.go` | Shape success / error tool results for the SDK. |

### `lifecycle/` — process lifecycle

| File | Responsibility |
|---|---|
| `boot.go` | The boot state machine (4 states) and retry loop; backoff `1s → 60s`. |
| `boot_classify.go` | Classify boot errors (config / TLS / vault / network) into a kind + hint. |
| `boot_log.go` | Append boot failures to `~/.rune/logs/boot.log` (rotates at 1 MiB). |
| `shutdown.go` | Graceful shutdown: drain inflight (30s) → close adapters → zeroize the DEK. |

### `recovery/` — transport recovery

| File | Responsibility |
|---|---|
| `interceptor.go` | gRPC unary interceptor; on a retryable status code it retriggers the boot loop. |

### `spawn/` — runed auto-spawn

| File | Responsibility |
|---|---|
| `spawn.go` | Ensure `runed` is up: probe the socket, else exec `rune runed --detach` and wait up to 15s. |
| `lockunix.go` | `flock(2)` on `~/.runed/spawn.lock` so concurrent sessions don't double-spawn the daemon. |

### `obs/` — observability

| File | Responsibility |
|---|---|
| `slog.go` | `slog` handler that redacts secret-shaped values, plus request-id context propagation. |

### `cmd/`

| File | Responsibility |
|---|---|
| `rune-mcp/main.go` | Entry point: wire `Deps`, start the boot loop, register the tools, serve over stdio, handle signals / EOF. |
