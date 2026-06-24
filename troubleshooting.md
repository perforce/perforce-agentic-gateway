# Logging and troubleshooting

## Logging

PAG writes operational logs to **stderr** by default. Since AI clients capture an MCP server's stderr, PAG's logs usually land in your client's MCP log view (for example, VS Code's **MCP: List Servers -> Show Output**).

Control logging with flags (or environment variables) on the command --- see the [CLI reference](reference.md#command-line):

```sh
uvx perforce-agentic-gateway --log-level debug                  # more detail
uvx perforce-agentic-gateway --log-format json                  # machine-readable
uvx perforce-agentic-gateway --log-file /tmp/pag.log            # also write to a file
uvx perforce-agentic-gateway --log-stderr=false --log-file ...  # file only
uvx perforce-agentic-gateway --log-mcp                          # surface warnings in the AI client
```

- `--log-file` *tees*: PAG logs to the file in addition to stderr unless you turn stderr off.
- `--log-mcp` forwards warnings and errors to the connected AI client as MCP notifications, so the assistant can see and relay problems.
- In an AI client config, flags go in the `args` array after the package, e.g. `["perforce-agentic-gateway@latest", "--log-level", "debug", "--log-file", "/tmp/pag.log"]`. `PAG_LOG_LEVEL=debug` in the client's `env` block works too.

**Downstream server logs** are separate: each server process's stderr is captured by PAG. Retrieve it via the dashboard or by asking your assistant (*"Show me recent logs from klocwork"* --- `pag__server_logs`).

## Troubleshooting

### My AI client says PAG failed to start

1. Run the same command in a terminal: `uvx perforce-agentic-gateway --version`, then plain `uvx perforce-agentic-gateway` from your project directory (it should print `running`; `Ctrl-C` to stop). Startup errors are printed with a `startup:` prefix.
2. If the terminal works but the client doesn't, the client likely can't find `uvx` --- GUI apps have a narrower PATH. Use the absolute path to `uvx` in the client config.
3. Check the `--working-directory` value exists and is a directory --- PAG refuses to start otherwise.
4. A config file error (bad TOML, invalid value) also stops startup; the message names the file and the offending key.

### Windows Defender blocks PAG from starting (Attack surface reduction)

On managed Windows machines, Defender may block PAG from running with a **"Risky action blocked"** notification:

```text
Your organization has blocked this action.
App or process blocked: uv.exe
Blocked by: Attack surface reduction
Rule: Block executable files from running unless they meet a prevalence, age, or trusted list criteria
Affected items: ...\uv\cache\archive-v0\<hash>\Scripts\pag.exe
```

**What's happening.** This is a Microsoft Defender *Attack Surface Reduction* (ASR) rule (GUID `01443614-cd74-433a-b99e-2ecdc07bfc25`) that blocks executables which are freshly created, not widely seen ("low prevalence"), and not on a trusted list. When `uvx` runs an MCP server it materializes brand-new, unsigned `.exe` files into its cache (`%LOCALAPPDATA%\uv\cache\archive-v0\<hash>\Scripts\`) --- exactly what this rule targets.

**This is not a PAG defect, and it is not specific to PAG.** Any MCP server launched with `uvx` trips the same rule --- for example `uvx p4mcp-server` is blocked identically. The rule is off by default on Windows; it applies only where an administrator enabled it via Intune / Microsoft Defender for Endpoint.

Two different executables can be flagged, and they are not equally fixable:

| Executable | What it is | Can it be signed? |
| --- | --- | --- |
| `pag.exe` | the PAG program itself (a Go binary shipped in a Python wheel) | Yes --- it is copied verbatim from the wheel, so a code signature applied during release survives to disk. Signing is planned and will clear the block for this file. |
| `perforce-agentic-gateway.exe` | the launcher `uvx` generates from the package's console-script entry point | No --- it is generated locally on your machine and cannot carry a valid signature. This is inherent to `uvx` for every Python package, not something PAG can fix. |

Because the launcher can never be signed, **code-signing alone will not fully resolve this**. The reliable fix is a Defender policy exclusion, which covers both files.

**Confirm it is this rule.** In Event Viewer under `Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational`, look for event ID **1121** (blocked) or **1122** (audit). The entry names the rule GUID and the blocked path.

**Fix (requires your Defender administrator).** The rule is managed centrally, so the durable fix is an ASR exclusion applied through Intune / Microsoft Defender for Endpoint:

- Exclude the uv cache path: `%LOCALAPPDATA%\uv\cache\**`, or
- For a developer population, switch rule `01443614-cd74-433a-b99e-2ecdc07bfc25` to **Audit** mode.

On an *unmanaged* machine you can apply the same exclusion locally in an elevated PowerShell (on a managed machine a local change may be reverted at the next policy sync):

```powershell
Add-MpPreference -AttackSurfaceReductionOnlyExclusions "$env:LOCALAPPDATA\uv\cache"
```

Switching to a persistent install (`uv tool install`) does not avoid this --- the generated launcher is still unsigned --- so the exclusion is the recommended path on locked-down fleets.

### PAG runs but the tools don't appear

- Reload/restart the client after config changes.
- Confirm the client is in a mode that uses tools (e.g. agent mode in VS Code).
- Check the client's MCP log view for PAG's stderr output.

### A server shows `stopped` or its tools are missing

1. Ask: *"What's the status of my servers?"* (`pag__status`) --- or check the dashboard.
2. Read its logs: *"Show me recent logs from \<alias\>"* (`pag__server_logs`).
3. Common causes:
   - **Missing secret** --- the server's Config tab flags it; follow the link to the secret entry page (see [Secrets](secrets.md)).
   - **Disabled** --- `enabled = false` in some config layer; remember local overrides project overrides global.
   - **Bad command/image/url** --- try running the server's command by hand.
   - **No container runtime** --- container servers (`image = ...`) need `docker` or `podman` on your PATH (or `[runtime] container` set in config).
   - **Tool filtered out** --- check the server's `[tools]` config and the dashboard's Tools tab.
4. After fixing, restart it: *"Restart \<alias\>"* (`pag__restart_server`), or use the dashboard.

### The dashboard URL doesn't open

- `uvx perforce-agentic-gateway view` prints the URL to the terminal --- open it manually if the browser didn't launch.
- The dashboard binds to `127.0.0.1` only; you cannot open it from another machine.

### Secrets aren't being found

- Check the server's Config tab in the dashboard: it shows which secrets are set and which backend (keychain or fallback file) is in use.

### Getting a debug trace

```sh
uvx perforce-agentic-gateway --log-level debug --log-file /tmp/pag.log        # Linux/macOS
uvx perforce-agentic-gateway --log-level debug --log-file "$env:TEMP\pag.log" # Windows (PowerShell)
```

reproduces the problem with full detail in the log file --- attach this when filing an issue.
