# Configuration reference

PAG is configured with TOML files. This page documents every file, key, and rule.

## Files and layering

PAG merges up to three configuration files, in this order:

| Layer | Path | Purpose |
| --- | --- | --- |
| 1. Global | `$XDG_CONFIG_HOME/pag/config.toml` (Linux/macOS) or `%APPDATA%\pag\config.toml` (Windows) | Your personal defaults, available in every project |
| 2. Project | `<project>/.pag/config.toml` | Shared with your team; commit it to version control |
| 3. Local | `<project>/.pag/config.local.toml` | Machine-specific overrides; keep out of version control |

Later layers override earlier ones. For servers the merge is per alias and field-by-field: a `[servers.p4]` block in the local layer deep-merges over the project layer's block for that alias, with unset fields falling through from the lower layer.

`<project>` is PAG's working directory --- the directory it was launched in, or the value of `--working-directory` (see the [CLI reference](reference.md#command-line)).

PAG watches these files and applies changes live; no restart needed.

You don't create these files by hand --- PAG creates the global config on first run and writes project files as you add servers.

Unknown top-level keys are ignored (so configs written for newer PAG versions don't break older ones); invalid values are errors. Two exceptions are not silently ignored: a removed `[servers.<alias>.settings]` table is rejected outright, and unknown registry-input keys are logged as warnings.

## Top level

### `version` (required)

```toml
version = 1
```

The schema version. Must be `1`. Required in every config file.

### `[[registries]]`

Ordered list of registry sources. See [Catalog and registries](using-pag.md#catalog-and-registries).

```toml
[[registries]]
url = "https://registry.example.com/mcp/v1"
type = "mcp-registry"   # optional; this is the default
```

| Key | Type | Notes |
| --- | --- | --- |
| `url` | string | Registry address. `http(s)://` for `mcp-registry`; `file://` with an absolute path for `devcatalog` |
| `type` | string | `"mcp-registry"` (default) or `"devcatalog"` (a local catalog directory) |

### `[runtime]`

```toml
[runtime]
container = "docker"
```

| Key | Type | Notes |
| --- | --- | --- |
| `container` | string | The container runtime command used to run container servers (`image = ...`). When unset, PAG auto-detects: `docker` first, then `podman`, on your PATH. Set it to use a different runtime or an absolute path |

### `[analytics]`

```toml
[analytics]
enabled = false   # opt out; telemetry is on by default
```

| Key | Type | Notes |
| --- | --- | --- |
| `enabled` | bool | Anonymous usage telemetry, **on by default (opt-out)**. Set `false` to turn it off. **Only valid in the global layer** --- PAG rejects it in project or local config |

## Servers: `[servers.<alias>]`

Each server is a table keyed by the **alias** you choose.

**Alias rules**: letters, digits, dashes, and underscores only; must not contain a double underscore (`__`); must not be empty; the names `pag` and `ui_builder` are reserved for PAG's own use.

A server is one of four kinds, selected by which identity field it sets. The kinds are mutually exclusive:

| Kind | Identity field | May not combine with |
| --- | --- | --- |
| Registry-sourced | `source` | `command`, `image`, `url` |
| Local command | `command` | `url` |
| Container | `image` | `url` (but `command` + `image` together is valid: a custom entrypoint in a container) |
| Remote HTTP | `url` | `command`, `image` |

### Common fields

| Key | Type | Applies to | Notes |
| --- | --- | --- | --- |
| `enabled` | bool | all | Whether PAG starts the server. PAG starts a server only when `enabled = true` is set explicitly; an omitted `enabled` is treated as off. (The dashboard and `pag__enable_server` set `enabled = true` for you.) A later layer can override either way |
| `secrets` | string array | all | Names of secrets this server needs. Values come from the keychain --- see [Secrets](secrets.md) |
| `env` | string table | registry, command, container | Extra environment variables for the server process |

### Registry-sourced servers

```toml
[servers.p4]
source = "com.perforce/p4"
version = "1.2.0"                  # optional version pin
registry = "https://pag-registry.public.prd.shared.perforce.com/v0.1"  # optional disambiguation
enabled = true
```

| Key | Type | Notes |
| --- | --- | --- |
| `source` | string | The server's name in the registry |
| `version` | string | Pin a specific version. Without it, the dashboard shows when updates are available |
| `registry` | string | Which registry to resolve `source` from, when more than one offers it |

#### Inputs: `[servers.<alias>.inputs.<NAME>]`

Registry servers declare **inputs** --- named settings (environment variables, arguments, or headers). Configure each by name:

```toml
# Simple input --- provide a value:
[servers.p4.inputs.PROJECT_ROOT]
value = "/src/myapp"

# Templated input --- the registry defines a template with {variables};
# you provide the variable values:
[servers.p4.inputs.SERVER_URL.variables]
host = "p4.example.com"
port = 8080
```

Setting `value` on a templated input is an error --- set its `variables` instead. Variable values may be strings, numbers, or booleans, matching the variable's declared format.

#### Remote variables: `[servers.<alias>.remote.variables]`

Some registry servers expose a remote endpoint whose URL contains `{variables}` (for example a region or tenant). Bind them here:

```toml
[servers.cloudtool.remote.variables]
region = "eu-west-1"
```

### Local command servers

```toml
[servers.mytool]
command = "npx"
args = ["-y", "@example/mcp-server"]
env = { LOG_LEVEL = "debug" }
secrets = ["API_KEY"]            # injected as environment variables
enabled = true
```

| Key | Type | Notes |
| --- | --- | --- |
| `command` | string | Executable to run; speaks MCP on stdin/stdout |
| `args` | string array | Command arguments |

### Container servers

```toml
[servers.containertool]
image = "example/mcp-server:latest"
volumes = ["/src/myapp:/workspace"]
enabled = true
```

| Key | Type | Notes |
| --- | --- | --- |
| `image` | string | Container image. Runs via your `[runtime] container` command |
| `volumes` | string array | Volume mounts |
| `command` | string | Optional: override the container entrypoint |

### Remote HTTP servers

```toml
[servers.internal-api]
url = "https://mcp.example.internal/v1"
transport = "streamable-http"    # optional; this is the default
secrets = ["API_TOKEN"]

[servers.internal-api.headers]
Authorization = "Bearer ${API_TOKEN}"
```

| Key | Type | Notes |
| --- | --- | --- |
| `url` | string | The remote MCP endpoint |
| `transport` | string | `"streamable-http"` (default) or `"sse"` (legacy HTTP+SSE). Only valid together with `url` |
| `headers` | string table | HTTP headers sent with every request. `${NAME}` placeholders are replaced with secret values |

### Tool visibility: `[servers.<alias>.tools]`

Controls which of the server's tools are exposed. See [Managing servers](using-pag.md#controlling-which-tools-are-exposed).

```toml
[servers.p4.tools]
default = "enabled"               # or "disabled"; default is "enabled"
enabled = ["get_issues"]          # force-enable, overrides default
disabled = ["delete_project"]     # force-disable, overrides default
```

| Key | Type | Notes |
| --- | --- | --- |
| `default` | string | Fate of tools not named in either list: `"enabled"` or `"disabled"` |
| `enabled` | string array | Tools always exposed |
| `disabled` | string array | Tools always blocked |

A tool name may not appear in both lists.

## Complete example

```toml
# .pag/config.toml --- project layer, committed to version control
# Perforce's registry is built in, so com.perforce/* servers resolve
# with no [[registries]] entry. Add blocks only for other registries.
version = 1

[servers.p4]
source = "com.perforce/p4"
enabled = true

[servers.p4.inputs.PROJECT_ROOT]
value = "/src/myapp"

[servers.p4.tools]
default = "enabled"
disabled = ["delete_project"]

[servers.docs]
command = "npx"
args = ["-y", "@example/docs-mcp"]

[servers.internal-api]
url = "https://mcp.example.internal/v1"
secrets = ["API_TOKEN"]

[servers.internal-api.headers]
Authorization = "Bearer ${API_TOKEN}"
```

```toml
# .pag/config.local.toml --- machine-local overrides, not committed
version = 1

[servers.internal-api]
enabled = false                    # not reachable from this machine
```
