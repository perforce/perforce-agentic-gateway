# Connecting AI clients

PAG runs as a **stdio MCP server**, launched via `uvx`. Any client that can launch a local stdio MCP server can use PAG. (Clients that connect only to *remote* MCP servers --- a URL, not a local command --- cannot.)

## The pattern

Every client comes down to the same command and arguments:

| Setting | Value |
| --- | --- |
| command | `uvx` |
| args | `perforce-agentic-gateway@latest` (plus `--working-directory <dir>` to pin a project) |
| transport | stdio |

- **`@latest`** makes each launch use the newest PAG. Drop it, or pin `perforce-agentic-gateway@2026.1`, to lock a version.
- **`--working-directory`** sets the project root PAG reads `.pag/config.toml` from. Clients launched from your project (Claude Code, Codex, Cursor, VS Code) already use the right directory --- only add the flag to pin a project regardless of where the client starts.

If you've installed PAG as a tool (`uv tool install perforce-agentic-gateway` --- see [Installing PAG as a tool](reference.md#installing-pag-as-a-tool)), set command to `perforce-agentic-gateway` with no arguments; everything else is the same.

Generic `.mcp.json` (Cursor and others):

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

After configuring, restart the client and confirm `pag__*` tools appear (ask: *"Call pag__status and tell me what you see."*). If they don't, see [Troubleshooting](troubleshooting.md).

## Claude Code

```sh
claude mcp add pag -- uvx perforce-agentic-gateway@latest
```

User scope (every project): add `--scope user`. Verify with `claude mcp list`; inside a session, `/mcp` shows the connection.

## Codex CLI

`~/.codex/config.toml`:

```toml
[mcp_servers.pag]
command = "uvx"
args = ["perforce-agentic-gateway@latest"]
```

Or: `codex mcp add pag -- uvx perforce-agentic-gateway@latest`. Verify with `codex mcp list`.

## VS Code (GitHub Copilot)

`.vscode/mcp.json` (per workspace), or the **MCP: Open User Configuration** command (global). Note the key is `servers` and the type is explicit:

```json
{
  "servers": {
    "pag": {
      "type": "stdio",
      "command": "uvx",
      "args": ["perforce-agentic-gateway@latest"]
    }
  }
}
```

Verify with **MCP: List Servers** (also shows status and logs), or the tools icon in agent mode.

## Cursor

`.cursor/mcp.json` (per project) or `~/.cursor/mcp.json` (global) --- the generic `.mcp.json` block above. Verify in **Cursor Settings -> MCP**.
