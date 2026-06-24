# Reference

Short reference material: the command line, uninstalling, and a glossary.

- [Command line](#command-line)
- [Uninstalling](#uninstalling)
- [Glossary](#glossary)

---

## Command line

PAG runs via `uvx` --- there's no install step. The command is `uvx perforce-agentic-gateway` (add `@latest` to always use the newest release).

### Synopsis

```
uvx perforce-agentic-gateway [flags]         Run as an MCP gateway (what AI clients launch)
uvx perforce-agentic-gateway view [flags]    Open the browser dashboard
uvx perforce-agentic-gateway --version       Print version and exit
```

### Installing PAG as a tool

The examples in these docs use `uvx perforce-agentic-gateway`, which downloads and runs PAG on demand with no install step. If you'd rather install it once --- or you already manage CLI tools with uv --- install it as a uv tool:

```sh
uv tool install perforce-agentic-gateway
```

This puts `perforce-agentic-gateway` on your `PATH`. From then on, just drop the `uvx` prefix --- the flags and subcommands are identical:

```sh
perforce-agentic-gateway              # run as an MCP gateway
perforce-agentic-gateway view         # open the dashboard
perforce-agentic-gateway --version    # print version
```

Unlike `uvx perforce-agentic-gateway@latest`, an installed tool stays pinned at the version you installed. Keep it current with `uv tool upgrade perforce-agentic-gateway`. To launch the installed command from a client, set the command to `perforce-agentic-gateway` with no arguments --- see [Connecting AI clients](connecting-clients.md).

### Modes

#### Default mode

`uvx perforce-agentic-gateway` with no subcommand runs the MCP gateway: it speaks MCP on stdin/stdout, starts your enabled servers, and also serves the dashboard in the background. This is the command your AI client launches --- you rarely run it by hand. When ready, it prints `running` to stderr.

#### `uvx perforce-agentic-gateway view`

Dashboard-only mode: starts your servers and the dashboard, prints the dashboard URL, and opens it in your browser. No MCP connection on stdio. Use this to manage PAG outside of an AI session. Stop it with `Ctrl-C`.

### Flags

| Flag | Default | Description |
| --- | --- | --- |
| `--version` | --- | Print version information and exit |
| `--working-directory <dir>` | current directory | Project root for config resolution. Must exist |
| `--log-level <level>` | `info` | Minimum log level: `debug`, `info`, `warn`, `error` |
| `--log-format <format>` | `text` | Log output format: `text` or `json` |
| `--log-file <path>` | --- | Also write logs to this file |
| `--log-stderr` | `true` | Write logs to stderr (`--log-stderr=false` to disable) |
| `--log-mcp` | `false` | Forward warnings and errors to the connected AI client as MCP notifications |

See [Logging and troubleshooting](troubleshooting.md) for how to use the logging flags.

### Environment variables

| Variable | Effect |
| --- | --- |
| `PAG_LOG_LEVEL` | Default for `--log-level` when the flag is not given |
| `PAG_LOG_FORMAT` | Default for `--log-format` when the flag is not given |
| `XDG_CONFIG_HOME` | Overrides the base of the global config directory (`$XDG_CONFIG_HOME/pag/`) |
| `XDG_DATA_HOME` | Overrides the base of the data directory (`$XDG_DATA_HOME/pag/`) |
| `<ALIAS>__<SECRET>` | Secret fallback values --- see [Secrets](secrets.md) |

### Exit status

- `0` --- clean shutdown (interrupt/termination signal)
- `1` --- startup failure or fatal runtime error (details on stderr)

### Version scheme

```
$ uvx perforce-agentic-gateway --version
pag 2026.1 (pypi, 1970.1)
```

Versions are calendar-based: `YYYY.N` --- the release year and the release number within that year. The parenthetical shows the distribution channel the binary came from and the source commit.

---

## Uninstalling

Because PAG runs via `uvx`, there's nothing to uninstall --- `@latest` keeps it current automatically. To reclaim disk space, clear the cached download:

```sh
uv cache clean perforce-agentic-gateway
```

(If you installed PAG with `uv tool install`, remove it with `uv tool uninstall perforce-agentic-gateway`.)

To remove all traces, also delete:

- the global config directory (`~/.config/pag/`, or `%APPDATA%\pag\` on Windows)
- the data directory (`~/.local/share/pag/`, or `%LOCALAPPDATA%\pag\` on Windows)
- `.pag/` directories in your projects
- any PAG entries in your OS keychain (service name `pag`)

---

## Glossary

**alias** --- The short name you give a server in your configuration (e.g. `klocwork`). Used as the prefix in qualified tool names (`klocwork__get_issues`). Letters, digits, dashes, underscores; no `__`.

**catalog** --- Two related senses: (1) the set of installable servers PAG aggregates from your registries (what you browse on the dashboard's Catalog page); (2) the searchable index of your enabled servers' tools that your assistant queries via `pag__explore`.

**dashboard** --- PAG's browser UI, served on `127.0.0.1` and opened with `uvx perforce-agentic-gateway view`. Manage servers, browse the catalog, enter secrets.

**downstream server** --- Any MCP server PAG connects to on your behalf: a local command, a container, or a remote HTTP endpoint.

**gateway** --- A program that presents one MCP server to a client while connecting to many MCP servers behind it. PAG is a gateway.

**input** --- A named setting a registry server declares it needs (a path, URL, project name, ...). You provide values when enabling the server or later via `pag__configure_server`, the dashboard, or `[servers.<alias>.inputs.<NAME>]` in config.

**MCP (Model Context Protocol)** --- The open standard AI clients use to talk to tool servers. PAG speaks MCP on both sides: as a server to your AI client, as a client to your downstream servers.

**qualified name** --- A downstream tool's full address through PAG: `<alias>__<toolname>`, e.g. `p4__submit`. Resources use `<alias>+<original uri>`.

**registry** --- An HTTP catalog of installable MCP servers (`[[registries]]` in config). PAG ships pointed at the Perforce registry. A `devcatalog` registry is the same thing as a local directory.

**sandbox** --- The locked-down JavaScript environment inside PAG where `pag__execute` and `pag__explore` programs run. No network, no filesystem, no imports; its only outside capability is `callTool()`.

**secret** --- A credential a server needs (API key, token, password). Referenced by name in config; values live in your OS keychain. See [Secrets](secrets.md).

**stdio** --- Standard input/output. The transport between your AI client and PAG: the client launches `uvx perforce-agentic-gateway` and exchanges MCP messages over the process's stdin/stdout.

**streamable-http / sse** --- The two transports PAG can use to reach *remote* downstream servers. `streamable-http` is the current MCP standard and the default; `sse` is the legacy HTTP+SSE transport for older servers.

**working directory** --- The directory PAG resolves project configuration from (`.pag/config.toml`). Defaults to where PAG was launched; override with `--working-directory`.
