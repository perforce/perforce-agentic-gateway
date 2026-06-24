# Secrets

Many MCP servers need credentials --- API keys, tokens, passwords. PAG keeps these out of your config files, out of version control, and out of your AI conversations.

## How secrets are entered

Secrets are entered through the **dashboard**, not through chat. When a server needs a secret that isn't set:

- If your assistant is enabling the server, it replies with a dashboard link for secret entry. Open it, enter the value, done.
- In the dashboard, each server's secret entry page (linked from its Config tab) shows each required secret, whether it is set, and forms to enter or replace values.

Your AI assistant never sees secret values --- config files reference secrets by *name only*.

## Where secrets are stored

PAG stores secrets in your operating system's native keychain:

| Platform | Backend |
| --- | --- |
| macOS | Keychain |
| Windows | Credential Manager |
| Linux | Secret Service (GNOME Keyring, KWallet, etc. via D-Bus) |

If no keychain is available (for example, a headless Linux machine without a Secret Service), PAG falls back to a plain file in its data directory (`secrets.toml`). The file is not encrypted, but it is created with permissions that make it readable only by your user (mode `0600`). The dashboard shows which backend is in use.

## How secrets are looked up

When PAG starts a server, each secret is resolved by searching, in order:

1. Project-scoped entry in the keychain
2. Global entry in the keychain
3. Project-scoped entry in the fallback secret file
4. Global entry in the fallback secret file
5. An environment variable (see below)

Project-scoped entries let two projects use different credentials for the same server; global entries cover the common case of one credential everywhere.

### Environment variable fallback

For CI machines or environments where a keychain isn't practical, PAG also accepts secrets from environment variables named `<ALIAS>__<SECRET>` --- the server alias uppercased with dashes replaced by underscores, then a double underscore, then the secret name. For example, the `TOKEN` secret of server `my-tool` can be provided as:

```sh
export MY_TOOL__TOKEN=...      # Linux/macOS
$env:MY_TOOL__TOKEN = "..."    # Windows (PowerShell)
```

## How servers receive secrets

Depends on the server kind:

- **Local command / container servers** --- secrets are injected as environment variables in the server's process.
- **Remote HTTP servers** --- secrets are substituted into headers using `${NAME}` placeholders:

  ```toml
  [servers.internal-api]
  url = "https://mcp.example.internal/v1"
  secrets = ["API_TOKEN"]

  [servers.internal-api.headers]
  Authorization = "Bearer ${API_TOKEN}"
  ```

Registry servers declare their secrets in their catalog metadata, so PAG knows what to prompt for. For custom servers, list the secret names in the `secrets` field as above. After the custom server has been added with a required set of secrets, you will be able to input them using the web UI the same as registry provided servers.

## Good practices

- Commit `.pag/config.toml` freely --- it holds secret *names*, never values.
- Use project-scoped secrets when different projects need different credentials.
- On shared/CI machines, prefer `<ALIAS>__<SECRET>` environment variables over the fallback file.
