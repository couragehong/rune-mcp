# rune-mcp documentation

Start with the [project README](../README.md) for the big picture and a quick
start. These documents go deeper, one topic each:

| Document | What's inside |
|---|---|
| [ecosystem.md](ecosystem.md) | How `rune`, `runed`, and `rune-mcp` relate, plus the Vault and envector backing services. |
| [architecture.md](architecture.md) | Why three processes, the boot state machine, and the capture/recall flows. |
| [file-structure.md](file-structure.md) | The directory layout, the package dependency rules, and a per-file map. |
| [configuration.md](configuration.md) | `config.json`, the `~/.rune` and `~/.runed` layout, and environment variables. |
| [security.md](security.md) | The security model, what stays encrypted, and known limitations. |

For exact type signatures, browse the package doc comments with `go doc ./internal/...`.
