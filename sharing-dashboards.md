# Sharing dashboards

A dashboard you build in one project can be handed to another project — a co-worker on a different machine, with a different set of MCP servers configured under different names. This page covers that path: **export**, which packages a dashboard into a single file, and **import**, which installs one into a project.

If you only need to share a dashboard with people who work in the same project, you don't need any of this — commit `.pag/dashboards/` to version control and they'll have it. See [Sharing within a project](dashboards.md#sharing-within-a-project).

- [Why export exists](#why-export-exists)
- [Exporting](#exporting)
- [Importing](#importing)
- [Importing the same dashboard twice](#importing-the-same-dashboard-twice)
- [The trust decision](#the-trust-decision)
- [Other reasons an import is refused](#other-reasons-an-import-is-refused)
- [Upgrading a dashboard built before portability](#upgrading-a-dashboard-built-before-portability)
- [What the export file carries](#what-the-export-file-carries)
- [Limits](#limits)

## Why export exists

A dashboard's code calls its MCP servers by name — the alias you had configured when the dashboard was authored. A dashboard written against a server you aliased `p4` breaks in a project where the same server is aliased `perforce`.

So an exported dashboard travels with a list of **servers this dashboard needs**, described in terms that mean the same thing on someone else's machine. Import reads that list, checks whether the receiving project can satisfy it, and lets the recipient point each entry at whatever they call that server locally. The dashboard's code is never rewritten; PAG translates the names at runtime.

That list is something your assistant declares — PAG does not read the dashboard's code to guess it. See [Declaring requirements](dashboards.md#declaring-requirements).

## Exporting

Export runs in two steps, whether you use your assistant or the browser:

1. **Preview.** PAG works out the list of servers the dashboard needs from your configuration and shows it to you. Nothing is written.
2. **Confirm.** You approve or edit the list, and PAG writes the archive.

The review step is why export works this way. The list PAG works out is a good guess from your configuration, not a statement of fact — it can't know that the `internal-api` server you built by hand is the one your recipient calls `api`, and it can't know which credentials they will have to supply themselves. Read every entry before you confirm.

Each entry can carry a **note**, which is free text for whoever imports the dashboard. That note is the only place to tell them things like "you'll need your own P4 credentials". PAG neither prompts you for it nor checks what you wrote.

### What never travels

PAG leaves credentials and machine-specific settings out of the export. Tokens, API keys, secret names and values, environment variables, request headers, mounted volumes, and your per-server tool filters are all excluded.

One caveat: for a server you configured by hand, the command, arguments, URL, image name, and transport are copied as written so the recipient can recognize or recreate it. PAG does not scan or redact those values, so never put a credential in one. A credential embedded there travels in the export file.

### Export with your assistant

Ask for it:

> "Export the server-overview dashboard."

The assistant should then show you each server the dashboard needs, where it comes from, and any note. Once you're satisfied, you confirm and the archive is written to your project:

```text
.pag/
  exports/
    server-overview.zip
```

You don't choose the path — it is always `.pag/exports/<dashboard name>.zip`.

- **`.pag/exports/` is gitignored.** PAG adds the entry for you.
- **An export file already at that path is replaced**, and the report tells you so. Move the old file first if you want to keep it.
- **Nothing is uploaded anywhere.** Export produces a local file; getting it to your recipient is up to you.

### Export from the browser

Open the dashboard gallery, then open the actions menu on a dashboard's card or its own page and choose **Export**.

The page shows one row per server the dashboard needs, with a badge for where the server comes from and the version range it needs, plus a notice naming what is excluded from the export. The download action streams a `.zip` to your browser's downloads; nothing is written into your project.

### Before you download

The list can come back with warnings, which the browser groups under **Before you download**. None of them block the download by themselves:

| Warning | What it means |
| --- | --- |
| An entry is incomplete | PAG couldn't determine something the export file has to carry, usually because the server isn't in a catalog you have cached or its version isn't pinned. **Supply the value yourself, or the export is refused when you confirm.** |
| An entry's value is malformed | A value PAG worked out doesn't parse, typically a version pin. It travels as written. Fix the entry or the pin. |
| A local component takes priority over a shared one | A component name exists both in the dashboard's own directory and in your shared components. The archive carries the dashboard's own file. |
| The content is over a size or file-count limit | A warning in the preview, and a refusal when you confirm. Almost always your shared components — the warning names their contribution. |

The first of those is worth re-reading. An incomplete entry in the preview predicts a hard refusal when you confirm, so don't hand the list straight back without looking at it.

### When export is refused

If the dashboard itself is missing the information export depends on, the whole dashboard is refused. That means one of two things: it was built before PAG v2026.6, or its stored metadata has been damaged. Either way the fix is the same — ask your assistant to upgrade it, which records what is missing. See [Upgrading a dashboard built before portability](#upgrading-a-dashboard-built-before-portability). If the dashboard is in version control, restoring it there is worth trying first.

The remaining refusals name a single entry or file:

| Condition | What to do |
| --- | --- |
| An entry points at `pag` or `ui_builder` | Those are PAG's own and are never installable dependencies. Remove the entry or point it at a real server. |
| An entry points at a server that isn't in your configuration | Correct the declaration or the configuration. Validating the dashboard lists every name that has drifted. |
| PAG can't determine where a server comes from | Its catalog isn't cached, or a version doesn't resolve. Refresh the catalog and export again — the browser offers a **Refresh the catalog** button inline. |
| Over a size limit | See [Limits](#limits). Any export file already at the destination is left intact. |
| A file naming conflict | A file named `components` where a directory is needed, or two names differing only in letter case. Rename one and export again. |

## Importing

Import runs in two steps, whether you use your assistant or the browser:

1. **Stage.** PAG takes the dashboard file into a holding area and runs every automated check. Nothing is published.
2. **Review and decide.** You read what arrived, resolve any collision with a dashboard you already have, point each required server at one of your own, and decide whether to trust it. That decision is what writes the dashboard.

Every automated check runs in the first step, before you are asked anything. A file that fails one is refused there and nothing is kept.

One import is staged at a time per running PAG process, shared between the browser and your assistant. Staging a second dashboard file displaces the first, and the displaced one's next step tells you it is no longer staged.

### What your project needs first

Before any human step, import checks that your project can satisfy the servers the dashboard needs. If one is missing or its version is incompatible, import is refused and the staged file is discarded. There is no mid-import remediation: fix your configuration through the normal flows, then stage the file again.

| Refusal | What to do |
| --- | --- |
| A Perforce server is missing | Add it from the catalog and enable it, then import again. |
| A server from a third-party registry is missing | Add a registry that supplies it, add the server, enable it, then import again. |
| No suitable server exists for one the exporter configured by hand | Create and enable it using the recorded details as guidance, then import again. PAG cannot verify that kind of match for you. |
| A server's version doesn't match | You have the server, but your pinned version is outside the range the dashboard needs. The refusal names a version that would work, or tells you to refresh the catalog first. |
| A catalog is unavailable | Refresh it and try again. |

An unsatisfied requirement is not always your problem — the file you received can equally be at fault, and PAG says so. Only whoever exported the dashboard can author a corrected one.

Import never changes your configuration. It does not create a server, add a registry, or ask for a secret.

### Import with your assistant

Ask for the import and name the file:

> "Import the dashboard in ~/Downloads/server-overview.zip."

The path can be absolute, start with `~`, or be relative to the project, and the file does not need to be inside the project — it is usually wherever you downloaded it to.

That first call publishes nothing. What comes back is for review: the servers the dashboard needs and the assistant's best understanding of candidate aliases, the files in the archive, and the warning about what a trusted dashboard potentially has access to.

Pointing the required servers at your own aliases is meant to be a conversation. A chat session offers no guaranteed picker, though some agent harnesses do have tools for asking a human a question, so your assistant may decide to interview you interactively instead.

Where PAG matched a requirement to exactly one of your servers, the review says so. Where it could not, your assistant proposes something, labels the proposal as its own suggestion rather than as something PAG verified, and asks you when it has no basis to choose.

Once you decide, your assistant makes a second call, and **that second call is the trust grant** — it is what writes the dashboard. It has to name a server for every entry, including the ones PAG matched for you; an omission is refused.

There is no abandon step and none is needed. A staged import you walk away from is cleared by the next import, by PAG exiting, or by the next startup.

### Import from the browser

From the dashboard gallery, use the **Import** action in the page header. It is a gallery-level action, not something on an individual card.

The browser spreads the two steps over four screens:

1. **Upload** the `.zip`.
2. **Resolve the collision**, if the incoming dashboard clashes with one you already have.
3. **Point each required server** at one of your own.
4. **Review and trust** the dashboard, which is also the confirmation that writes it.

The review itself is on that fourth screen. The two screens before it ask you questions; they do not show you the dashboard.

Each screen has its own URL and the staged import survives a refresh. Refreshing the trust screen returns you to the screen before it, so the final review stays a forward-only step. **Cancel import** is available on every screen after upload.

### Pointing required servers at your own

The browser shows one row per required server, and the step is never skipped. Where a requirement matches exactly one enabled server of yours, the row arrives prefilled and marked automatic — but it is still shown to you and still editable. PAG's own aliases (`pag` and `ui_builder`) are never offered.

Where more than one of your servers matches, or where the exporter configured the server by hand, you pick. A hand-configured one has to be chosen before you can continue, and the choice is doing real work:

> Choosing a server here is your own assertion that it is the one this requirement describes. PAG has not verified that, and it has not checked whether the server exposes the capabilities this dashboard calls.

If your project configures no server the step can offer, it says so, and you'll need to stop, add the server, and import again.

## Importing the same dashboard twice

PAG tracks a dashboard's identity separately from its name, so it can recognize a newer copy of a dashboard you already have. Renaming a dashboard keeps that identity; duplicating one creates a new one, so a duplicate counts as a different dashboard.

Two independent questions decide what the collision step offers:

- Does a dashboard you already have share the incoming dashboard's identity?
- Does the incoming dashboard's name clash with one you already have?

| Your situation | What you get |
| --- | --- |
| One dashboard of yours is the same dashboard | A choice: **Replace** it, or **Import as new** |
| More than one is | **Replace is not offered.** Import as new is the only path, and a notice names the dashboards |
| Only the name clashes | Unrelated dashboards. Pick a different name |
| Nothing matches | The step never opens |

**Replace** is wholesale. There is no merge and no update in place — it deletes the target and writes the imported dashboard in its place, under the target's own name:

> This permanently deletes the `server-overview` dashboard and all its files, then writes the imported dashboard in its place. Any local changes you made to that dashboard are lost. PAG keeps no prior revision, so the only way back is restoring it from version control. A dashboard that is not in version control cannot be recovered.

A replaced dashboard goes back to waiting for your trust review. The decision you made last time does not carry over.

**Import as new** gives the copy a fresh identity, so import treats it as a different dashboard from then on, and asks you for a name to use in the URL.

If you end up with two local dashboards that share an identity, nothing warns you at authoring time — it shows up only as a withdrawn **Replace** offer at import. The fix is to get back to one copy: delete the extra, restore a single copy from version control, or ask your assistant to duplicate one (which gives the duplicate a new identity) and delete the copy it came from.

## The trust decision

An imported dashboard **waits for your trust review**. It will not render or run until you have read it and chosen to trust it.

The dashboard's reach is your reach, and the judgment has to be yours:

> **Unrestricted access to your MCP servers**
>
> This dashboard executes arbitrary code and will have unrestricted access to your PAG MCP servers.

The dashboard can call any tool PAG exposes on any server you have enabled, with nothing stopping to ask you per call. Per-server tool filters still apply. Confirming trust is the same judgment you would apply to code a teammate committed.

The review shows the servers the dashboard needs and every file in the archive. That listing is the whole archive, including every component the exporter's project happened to carry. If you want to read the code first, stop there and inspect the `.zip`.

Confirming trust is what writes the dashboard. The decision is not recorded anywhere, so trustworthiness stays your judgment rather than something an export file can claim for itself.

### Keep your AI client asking

Nothing renders until a second, separate step publishes the dashboard. What PAG can see of the human behind that step differs by path:

- **In the browser**, the confirm action *is* the publishing request. The review and the write are one step.
- **With your assistant**, PAG sees a second tool call and nothing about the human behind it. It cannot tell a decision you made from one made on your behalf.

So your AI client's approval prompt is where a gate on that call belongs. The first call only reads the file and reports; the second is the one worth putting in front of a human.

Two things to keep in mind:

- **Check that your AI client asks you before running a tool, and keep it asking for this one.** If it runs tools unprompted, the publishing call runs unprompted too.
- **Staging and publishing are the same tool.** An always-allow granted at the harmless first call covers every publishing call after it, including ones for dashboards you haven't seen.

## Other reasons an import is refused

| Condition | What to do |
| --- | --- |
| There is no file at the path your assistant named, or PAG can't read it | Correct the path and import again. If the file is there, grant PAG read access or copy it somewhere PAG can read. |
| The archive isn't safe to unpack | The dashboard has to be exported again. PAG does not report which check it failed. |
| The file is larger than import accepts | See [Limits](#limits). |
| A browser upload didn't carry exactly one file | Choose exactly one file and upload again. |
| The list of servers the dashboard needs isn't valid | The refusal lists every problem. No change of yours fixes this — only whoever exported the dashboard can author a corrected file. |
| The file was made by a newer PAG than yours | Either upgrade PAG, or get a corrected file. |
| The dashboard expects a runtime this PAG doesn't serve | Same two options. "Unsupported" means "not in the served set", not necessarily "old". |
| The import you were working on is no longer staged | It was completed, cancelled, or displaced by a newer one. Stage the file again. To see whether an earlier one landed, open the dashboard gallery. |
| The dashboard you were replacing changed | It is gone, or it is no longer the same dashboard it was matched on. Nothing was written. |
| The write timed out | Nothing was written. Import the same file again. |
| The cache and the project are on different drives | Put the project's cache directory on the same drive as the project, then import again. |

## Upgrading a dashboard built before portability

A dashboard created before PAG v2026.6 has no recorded identity and no list of the servers it needs. It keeps rendering fine, but it **cannot be exported or duplicated** until it is upgraded.

Upgrading is a one-time act, and only your assistant can do it — there is no button for it in the browser. PAG hands you the exact prompt to paste:

> This dashboard cannot be exported: it was created by an old version of the Perforce Agentic Gateway. Ask your AI agent to upgrade it. Paste this exactly:
>
> "Use the dashboard update tool (`ui_builder__update_dashboard`) to upgrade the `server-overview` dashboard."
>
> The agent will analyze the dashboard to determine which MCP servers it uses, confirm the list with you, and record it. If this dashboard is in version control, upgrade it once and commit: upgrading it again elsewhere would give the same dashboard two different identities.

**Upgrade once, commit the result, and have every other checkout pull it.** Upgrading the same dashboard independently in two checkouts gives the two copies different identities. Import then treats them as unrelated dashboards and never offers to replace one with the other.

Until a dashboard is upgraded, your assistant cannot change its metadata without also declaring the servers it needs. Browser edits to title, description, type, and tags stay available throughout. There is no bulk upgrade; each dashboard is done on its own.

## What the export file carries

The archive is a plain `.zip` holding the dashboard's own files — its metadata, its code, its styles — plus every component it can reach.

That component set is the exporter's **entire pool of shared components**, not just the ones this dashboard uses. Two consequences:

- An imported dashboard carries a full copy of the exporter's shared components. Importing five dashboards from the same person means five copies.
- The file listing on the trust step is correspondingly long, and most of it is usually shared components.

An imported dashboard's components land private to that dashboard. **Import never writes to your shared components** and never overwrites one of yours.

Nothing re-checks the servers a dashboard needs after it lands. If you later disable or remove one, you'll see the dashboard fail when it next tries to call it.

Re-exporting the same dashboard produces an equivalent archive, not a byte-identical one. Don't diff two export files to check whether a dashboard changed.

## Limits

The same three limits apply in both directions:

| Limit | Value |
| --- | --- |
| Archive size | 32 MiB |
| Unpacked size | 128 MiB |
| File count | 512 files |

When a limit is hit on export, the message names your shared components' contribution, because that is almost always what pushed it over. Trimming the shared pool is the usual fix.
