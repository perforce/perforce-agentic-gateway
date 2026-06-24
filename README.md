<picture>
  <source media="(prefers-color-scheme: dark)" srcset="logo-pag-rev.svg">
  <img src="logo-pag-reg.svg" alt="PAG --- Perforce Agentic Gateway" width="420">
</picture>

# PAG --- Perforce Agentic Gateway

Perforce Agentic Gateway (PAG) is a local MCP gateway providing a token efficient code mode. AI clients are configured once, to connect to PAG and all other MCP servers are instead configured through PAG. Instead of being set up with and loading the tools of each MCP separately, PAG manages the running MCP server processes, restarts them when they crash, and stores their necessary credentials in your OS Keychain.

## Prerequisites

- **[uv](https://docs.astral.sh/uv/getting-started/installation/)** --- the only hard requirement.
- **Docker or Podman** --- only for servers that run as containers.
- **Each server's own dependencies** --- PAG proxies servers, it doesn't bundle them (e.g. Node for `npx` servers, an account for API servers).

## Quickstart

### 1. Connect your AI client

PAG runs as a **stdio MCP server**, launched via `uvx`.

With **Claude Code**:

```sh
claude mcp add pag -- uvx perforce-agentic-gateway@latest
```

For any client that uses **`.mcp.json`** (Cursor, and others):

```json
{
  "mcpServers": {
    "pag": {
      "command": "uvx",
      "args": ["perforce-agentic-gateway@latest"]
    }
  }
}
```

`@latest` keeps PAG current on each launch. Add `--working-directory <dir>` to pin a project. Other clients (Codex, VS Code, ...): see [Connecting AI clients](connecting-clients.md). Restart the client; `pag__*` tools should appear.

Prefer to install once instead of fetching on demand? `uv tool install perforce-agentic-gateway` puts a `perforce-agentic-gateway` command on your `PATH` that you run in place of `uvx perforce-agentic-gateway` --- see [Installing PAG as a tool](reference.md#installing-pag-as-a-tool).

### 2. See what's available

Ask your assistant:

> "What MCP servers are in the PAG registry?"

Lists everything installable from configured registries --- including Perforce's first-party servers, e.g. P4, BlazeMeter, Perfecto, Delphix, IPLM.

### 3. Enable a server from the catalog

Perforce product support MCP servers are available straight from the catalog. Ask your assistant to recommend servers for what you're working on:

> "What MCP servers would help with this project?"

PAG scans your project for technology signals and suggests catalog servers, always asking before it enables anything. Or name one directly:

> "Enable the P4 server."

PAG records it in your config and starts it. Most Perforce servers need credentials (typically a server URL and an access token); when so, the assistant hands you a dashboard link to enter them --- secrets never travel through the chat. See [Perforce servers](using-pag.md#perforce-servers) and [Secrets](secrets.md).

Browse the full catalog any time with `uvx perforce-agentic-gateway view`. See [Using PAG](using-pag.md).

### 4. Or add a custom server

Anything not in a registry --- a local command, a container, or a remote HTTP endpoint --- you add directly. The MCP **fetch** server runs via `uvx` and needs no credentials, so it's a handy zero-setup server to run end to end and see the flow:

> "Add a custom server named `fetch` that runs `uvx mcp-server-fetch`."

PAG writes it to your config and starts it.

### 5. Run a real command

> "Fetch https://modelcontextprotocol.io and tell me what MCP is."

The assistant calls `fetch__fetch` through PAG and answers from the live page.

## Why it stays fast

```
AI client --one stdio connection--> PAG --> server A (command)
                                     | --> server B (remote HTTP)
                                     +-> server C (container)
```

PAG doesn't forward every downstream tool to your client (20 servers can mean a thousand tools). It exposes a small, fixed `pag__*` set and keeps the full catalog inside the gateway:

- **`pag__explore`** searches the catalog; only the slice the assistant asks for enters its context.
- **`pag__execute`** runs JavaScript that calls downstream tools, so loops and fan-out across servers take **one** round trip instead of many.

Full mechanics (the sandbox, qualified tool names, prompts and resources) are in [Tools and the sandbox](using-pag.md#tools-and-the-sandbox).

## Going deeper

- **[Connecting AI clients](connecting-clients.md)** --- Claude Code, Codex, VS Code, Cursor.
- **[Using PAG](using-pag.md)** --- managing servers, the dashboard, the catalog and registries, Perforce servers, the tools/sandbox model.
- **[Secrets](secrets.md)** --- entering, storing, and looking up credentials.
- **[Configuration](configuration.md)** --- the complete `config.toml` reference.
- **[Troubleshooting](troubleshooting.md)** --- logging and common fixes.
- **[Reference](reference.md)** --- command line, uninstalling, glossary.

## Feedback

Bug reports and feature requests go through [GitHub issues](https://github.com/perforce/perforce-agentic-gateway/issues). For setup and runtime problems, check [Troubleshooting](troubleshooting.md) first --- it covers the common cases.

## License

Use of PAG is governed by the [Perforce Agentic Gateway Terms of Use](LICENSE.md).
