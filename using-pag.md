# Using PAG

This guide covers operating PAG day to day, once you're connected (if you're not yet, start at the [README](README.md)). It goes deeper than the getting-started page on each topic.

- [Managing servers](#managing-servers) --- add, enable, disable, restart, configure, and control which tools are exposed
- [The dashboard](#the-dashboard) --- PAG's browser UI
- [Catalog and registries](#catalog-and-registries) --- discover and install servers
- [Perforce servers](#perforce-servers) --- the first-party servers
- [Tools and the sandbox](#tools-and-the-sandbox) --- how your assistant reaches downstream tools, in full

## Managing servers

Everything you can do to a server --- add, enable, disable, restart, configure --- is available three ways:

1. **Conversationally**, by asking your AI assistant (PAG's management tools)
2. **Visually**, in the [dashboard](#the-dashboard) (`uvx perforce-agentic-gateway view`)
3. **Directly**, by editing config files (see the [configuration reference](configuration.md))

All three write to the same configuration. PAG watches its config files, so changes apply live --- running servers are started, stopped, or reconfigured without restarting PAG or your AI client.

### Server kinds

A server entry is one of four kinds, depending on which fields it sets:

| Kind | Key fields | PAG runs it as |
| --- | --- | --- |
| Registry-sourced | `source` (e.g. `"com.perforce/p4"`) | Whatever the registry entry specifies |
| Local command | `command`, `args` | A subprocess speaking MCP over stdio |
| Container | `image`, `volumes` | A container via your configured runtime |
| Remote HTTP | `url`, `headers`, `transport` | A connection to a remote MCP endpoint (`streamable-http` by default, or `sse`) |

Every server has an **alias** --- the short name you choose for it (e.g. `p4`, `blazemeter`). Aliases may contain letters, digits, dashes, and underscores, but not double underscores (`__`), which PAG reserves as its namespacing separator. The names `pag` and `ui_builder` are reserved for PAG's own use.

### Adding a server

**Conversationally** --- ask your assistant to find and enable servers:

> "What MCP servers would help with this project?" --- runs `pag__discover`, which scans your project for technology signals and recommends catalog servers. It always asks before enabling.

> "Search the catalog for BlazeMeter and enable it." --- runs `pag__search_catalog` then `pag__enable_server`.

> "Add a custom server called mytool that runs `npx -y @example/mcp-server`." --- runs `pag__add_server`.

If the server requires secrets, the assistant replies with a dashboard link where you enter them --- secrets never travel through the chat. See [Secrets](secrets.md).

**Dashboard** --- the Catalog page lists everything from your configured registries; the Add page covers custom servers. See [Catalog and registries](#catalog-and-registries).

**Config file** --- add a `[servers.<alias>]` block:

```toml
[servers.p4]
source = "com.perforce/p4"
enabled = true

[servers.mytool]
command = "npx"
args = ["-y", "@example/mcp-server"]
enabled = true

[servers.internal-api]
url = "https://mcp.example.internal/v1"
enabled = true
```

### Enabling and disabling

A disabled server stays in your configuration but is not started, and its tools disappear from the catalog your assistant sees.

- Ask: *"Disable p4 for now"* / *"Enable p4 again"*
- Dashboard: toggle on the server's page
- Config: `enabled = true` / `enabled = false` on the server block

### Restarting

If a server gets into a bad state:

- Ask: *"Restart the p4 server"* (`pag__restart_server`)
- Dashboard: restart button on the server's page

### Checking health and logs

- Ask: *"What's the status of my servers?"* (`pag__status`) --- reports each server's state: `starting`, `running`, `stopped`, or `disabled`, plus available updates.
- Ask: *"Show me recent logs from p4"* (`pag__server_logs`) --- returns the server process's recent stderr output.
- Dashboard: health is shown per server, and the Activity tab shows the server's health history and recent logs.

### Configuring a server

Registry servers declare **inputs** --- settings such as paths, URLs, project names, and secrets. Update them any time:

- Ask: *"Change p4's project path to /src/myapp"* (`pag__configure_server`)
- Dashboard: the server's Config tab
- Config file: the server's `env`, `args`, `[inputs.*]`, and `[remote.variables]` sections --- see the [configuration reference](configuration.md)

### Controlling which tools are exposed

You can trim a server's tool list --- useful for keeping the catalog focused or blocking tools you don't want used. Each downstream tool is either enabled (visible in the catalog and callable) or disabled (blocked entirely).

```toml
[servers.p4.tools]
default = "enabled"        # or "disabled"
disabled = ["delete_project", "admin_config"]
```

With `default = "disabled"` you can flip to an allow-list:

```toml
[servers.p4.tools]
default = "disabled"
enabled = ["get_issues", "list_projects"]
```

The dashboard's Tools tab shows the same switches visually. A tool may not appear in both lists.

### Project vs. global servers

Servers defined in your **global** config (`~/.config/pag/config.toml`) are available in every project. Servers in a **project's** `.pag/config.toml` apply to that project and can be committed to share with your team. Machine-specific tweaks go in `.pag/config.local.toml`, which overrides the project layer and is meant to stay out of version control. Later layers override earlier ones per server alias.

## The dashboard

The dashboard is PAG's browser UI for managing servers, browsing the catalog, and entering secrets.

### Opening it

From your project directory:

```sh
uvx perforce-agentic-gateway view
```

PAG starts, prints the dashboard URL, and opens it in your default browser. The dashboard serves on `127.0.0.1` (localhost) only --- it is never reachable from other machines.

Each project gets a **stable port**, derived from the project's identity (its version-control remote URL, falling back to its path), in the range 49152--65535. Bookmarks keep working across restarts. If the derived port happens to be in use, PAG falls back to a random free port --- the printed URL is always authoritative.

The dashboard is also available while PAG is running inside an AI client session --- when your assistant needs you to enter secrets, it hands you a link into the same UI.

### What's on it

#### Servers

The home page lists your configured servers with their health state:

| State | Meaning |
| --- | --- |
| `starting` | PAG is launching/connecting the server |
| `running` | Healthy and serving |
| `stopped` | Exited or unreachable |
| `disabled` | Configured but turned off |

An `update available` badge appears when a registry server has a newer version than the one you're pinned to.

Each server has a detail page with three tabs:

- **Activity** --- the server's health-state history and its recent log output
- **Config** --- its settings: inputs, environment variables, arguments; edit and save here. Missing secrets are flagged, with a link to the secret entry page
- **Tools** --- every tool the server exposes, with per-tool enable/disable switches (see [Controlling which tools are exposed](#controlling-which-tools-are-exposed))

Secret entry is its own page (linked from the Config tab and from the links your assistant hands you), where you can enter or replace each secret the server requires.

Plus actions: enable/disable, restart, and remove.

#### Catalog

Searchable listing of every server available from your configured registries. Pick a server, fill in its inputs and secrets, choose an alias, and add it to your project or global configuration. A refresh action re-fetches the registries. See [Catalog and registries](#catalog-and-registries).

#### Registries

A read-only list of your active registries (the Perforce default plus any you've added), each showing its health --- reachable, serving cached results, failed, or not yet checked --- and a refresh action. You add or remove registries by editing config, not here.

#### Add

A form for custom servers that aren't in any registry --- local commands, containers, or remote HTTP endpoints.

### Security

- Binds to `127.0.0.1` only; never exposed to the network.
- Browser-level protections (origin checks, CSRF protection, a strict content-security policy) prevent other websites from scripting against it.
- There is no login: anyone with local access to your machine session can open it, same as your terminal.

Secrets entered in the dashboard go straight into your OS keychain (or PAG's fallback secret store) --- see [Secrets](secrets.md).

## Catalog and registries

A **registry** is an HTTP catalog of installable MCP servers. PAG fetches server listings from your configured registries, caches them locally, and merges them into the **catalog** --- the set of servers you can browse in the dashboard, search conversationally, and enable with one step.

### The default registry

Out of the box, your catalog is populated from the Perforce registry --- no setup needed to get the first-party Perforce servers (see [Perforce servers](#perforce-servers)) and other published MCP servers. To pull from additional registries, add `[[registries]]` blocks (see [Adding registries](#adding-registries)).

### Browsing and searching

- **Dashboard**: the Catalog page lists everything, with search.
- **Conversationally**: *"Search the catalog for performance testing"* runs `pag__search_catalog`. With no query it lists all available servers.
- **Project-aware**: *"What servers would help with this project?"* runs `pag__discover`, which combines a scan of your project's technology signals with catalog searches and presents recommended servers for your approval.

### Installing from the catalog

When you pick a catalog server (in the dashboard, or by asking your assistant to enable it), PAG:

1. records it in your config under the alias you choose, with `source` set to the server's registry name (e.g. `com.perforce/p4`);
2. prompts for any **inputs** the server declares --- paths, project names, options;
3. prompts for any **secrets** --- these go to the dashboard's secret entry, never through chat (see [Secrets](secrets.md));
4. starts it and exposes its tools in the catalog your assistant queries.

In config form, an installed catalog server is compact --- the registry supplies the run details:

```toml
[servers.p4]
source = "com.perforce/p4"
enabled = true

# Inputs the server declares, if any --- for example:
[servers.p4.inputs.PROJECT_ROOT]
value = "/src/myapp"
```

You can pin a version with `version = "..."`; otherwise PAG shows an `update available` badge in the dashboard when the registry has something newer.

If two registries offer the same server name, disambiguate with `registry = "<registry url>"` on the server block.

### Adding registries

Add entries to your global or project config:

```toml
[[registries]]
url = "https://registry.example.com/mcp/v1"
```

Registries are fetched in order and merged. Entries must be `http://` or `https://` URLs.

#### Local catalogs (`devcatalog`)

For air-gapped setups or testing, a registry can be a local directory:

```toml
[[registries]]
url = "file:///absolute/path/to/catalog"
type = "devcatalog"
```

### Caching and refresh

Registry listings are cached on disk (in PAG's data directory) so startup doesn't depend on the network. The dashboard's Catalog page has a refresh action that re-fetches all registries.

## Perforce servers

Perforce publishes first-party MCP servers for its products through the default registry, so they appear in your catalog by default. First-party servers include:

| Server | What it gives your assistant |
| --- | --- |
| **P4** (`com.perforce/p4`) | Perforce P4 version control --- manage files, changelists, shelves, workspaces, jobs, and reviews |
| **BlazeMeter** (`com.perforce/blazemeter`) | Cloud-based performance testing --- load-testing workflows from creation to execution and reporting |
| **BlazeMeter API Monitoring** (`com.perforce/blazemeter-apitest`) | API monitoring --- teams, buckets, tests, schedules, environments, and results |
| **BlazeMeter Service Virtualization** (`com.perforce/blazemeter-sv`) | Create, configure, and manage virtual services |
| **Perfecto** (`com.perforce/perfecto`) | Cloud-based web and mobile testing --- workflows from creation to execution |
| **Delphix DCT** (`com.perforce/delphix-dct`) | Delphix Data Control Tower --- self-service data provisioning and management |
| **IPLM** (`com.perforce/iplm`) | Perforce IPLM (IP Lifecycle Management) --- IPs, libraries, releases, and design data |
| **IPLM Docs** (`com.perforce/iplm-docs`) | Searchable access to Perforce IPLM product documentation |

The catalog is the authoritative, up-to-date list --- the registry gains servers over time without you updating PAG.

### Enabling a Perforce server

Same as any catalog server (see [Adding a server](#adding-a-server)):

- Ask your assistant: *"Enable the P4 server"* --- or run *"What servers would help with this project?"* and PAG's discovery will recommend the relevant Perforce servers based on the technology signals in your project.
- Or use the dashboard's Catalog page.

Each server prompts for the connection settings and credentials it needs (for example, your server URL and an access token). Credentials are entered through the dashboard and stored in your OS keychain --- see [Secrets](secrets.md).

## Tools and the sandbox

This section explains what your assistant sees when it connects to PAG and how it reaches your downstream servers' tools. (For the short version, see [Why it stays fast](README.md#why-it-stays-fast) in the README.)

### What the assistant sees

PAG does **not** forward every downstream tool definition to your AI client. A setup with 20 servers might expose a thousand tools --- flooding the assistant's context window before the conversation even starts.

Instead, PAG presents a fixed set of eleven `pag__*` tools:

| Tool | Purpose |
| --- | --- |
| `pag__explore` | Search the tool catalog with JavaScript --- find tools, read their schemas |
| `pag__execute` | Run JavaScript that calls downstream tools and returns structured results |
| `pag__discover` | Scan the project for technology signals and recommend servers to enable |
| `pag__search_catalog` | Search the configured registries for installable servers |
| `pag__enable_server` | Enable a registry server for this project |
| `pag__add_server` | Add a custom server (command, container, or URL) |
| `pag__disable_server` | Disable a server |
| `pag__restart_server` | Restart a server |
| `pag__configure_server` | Update a server's inputs, environment, or arguments |
| `pag__status` | Report server health states |
| `pag__server_logs` | Retrieve a server's recent stderr output |

Downstream tools are reached *through* `pag__execute`, and discovered *through* `pag__explore`. The full catalog of downstream tools lives inside PAG --- it enters the assistant's context only in the filtered slices the assistant asks for.

### Names: how downstream tools are addressed

Every downstream tool gets a **qualified name**: `<alias>__<toolname>` --- the server alias you chose, a double underscore, and the tool's own name. For example, a `get_issues` tool on the server you aliased `klocwork` is `klocwork__get_issues`. (This is why aliases may not contain `__`.)

Resources from downstream servers are addressed as `<alias>+<original uri>`. Prompts follow the same `__` convention as tools.

### The explore -> execute flow

A typical request flows in two steps:

**1. Explore.** The assistant queries the catalog with a small JavaScript program against a `catalog` object:

```js
// Find tools related to "issues" and inspect their schemas
const allTools = Object.entries(catalog).flatMap(
  ([alias, {tools}]) => tools
);
const hits = fuzzy("issues", allTools, "name");
return hits.map(h => ({
  qualifiedName: h.item.qualifiedName,
  description: h.item.description,
  inputSchema: h.item.inputSchema
}));
```

`pag__explore` is read-only: it can search and inspect, never call. A built-in `fuzzy()` helper does approximate matching so typos and partial terms still find the right tools.

**2. Execute.** The assistant writes a program that calls the tools it found:

```js
// One round trip: fetch two things in parallel, combine them.
// Promise.allSettled keeps one failure from sinking the batch; each
// thrown error already names the failed tool --- its .message is
// prefixed "<server__tool>: <reason>" --- so surface it whole.
const names = ["klocwork__get_issues", "klocwork__get_metrics"];
const settled = await Promise.allSettled(
  names.map(name => callTool(name, { project: "myapp" }))
);
return settled.map(r =>
  r.status === "fulfilled"
    ? { data: JSON.parse(r.value[0].text) }
    : { error: r.reason.message }   // e.g. "klocwork__get_metrics: ..."
);
```

Because the program runs inside PAG, multi-step workflows --- loops, conditionals, fan-out across servers, error handling per server --- happen in **one** tool call instead of many model round trips, and intermediate results never bloat the conversation.

### The sandbox

`pag__execute` and `pag__explore` programs run in a locked-down JavaScript sandbox inside the PAG process:

- **Plain JavaScript** with `async`/`await`. No TypeScript, no imports.
- **A fresh sandbox per call** --- no state persists between calls.
- **No ambient capabilities**: no `fetch`, no `require`, no `process`, no timers, no `eval`. The *only* way a program can affect the outside world is `callTool()` --- and in `pag__explore`, not even that.
- **Execution timeout**: a runaway program is stopped after 120 seconds.
- `console.log()` output is captured and returned alongside the result, which helps the assistant debug its own programs.

`callTool()` reaches only tools that are **enabled** --- servers you have enabled, minus any tools you switched off (see [Controlling which tools are exposed](#controlling-which-tools-are-exposed)). Disabled tools are absent from the catalog and blocked from calling.

### Prompts and resources

Beyond tools, PAG also aggregates downstream servers' **prompts** and **resources** and serves them to your client through the standard MCP methods (listing, reading, resource subscriptions, and argument completions), using the qualified-name conventions above. If a client asks for a resource by its bare original URI, PAG resolves it to the right server when the URI is unambiguous.

### Server health notifications

When a downstream server fails in a way that needs your attention, PAG sends a notification through the MCP connection so your assistant can tell you about it --- for example, a server that exited because a required secret is missing.
