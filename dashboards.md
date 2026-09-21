# Dashboards

Perforce Agentic Gateways's **UI Builder** lets your AI assistant build rich web dashboards over the MCP servers you have configured — a P4 changelist board, a BlazeMeter results view, a deployment status page. If you have not connected PAG to your AI client yet, start with the [README](README.md).

You describe what you want in chat, the assistant writes the code, and PAG serves it at a URL you can bookmark.

- [What a dashboard is](#what-a-dashboard-is)
- [Creating one](#creating-one)
- [The authoring loop](#the-authoring-loop)
- [Viewing and managing dashboards](#viewing-and-managing-dashboards)
- [Declaring requirements](#declaring-requirements)
- [Reusable components](#reusable-components)
- [Where the files live](#where-the-files-live)
- [Sharing within a project](#sharing-within-a-project)
- [Turning authoring off](#turning-authoring-off)
- [The UI Builder tools](#the-ui-builder-tools)

## What a dashboard is

A dashboard is a small React application that lives in your project and calls MCP servers' tools through PAG at render time.

Three properties are worth understanding before you build one:

- **It is project code.** Dashboards are stored in `.pag/dashboards/` and are meant to be committed. Anyone who checks out the project gets them.
- **It runs with your configured access.** A dashboard can call any tool PAG exposes on any server you have enabled, without asking you per call. Per-server tool filters still apply.
- **The assistant writes the code, not PAG.** The UI Builder gives your assistant tools for scaffolding, metadata, validation, and listing. The actual `.jsx` is written by the model with its own file tools.

## Creating one

Ask your assistant. A useful first prompt, and the one PAG itself suggests on an empty gallery:

> "Create a dashboard showing server health status and recent activity."

Two things are required, and your assistant will ask if you haven't said:

- A **name**, which becomes the dashboard's URL. Lowercase letters, digits, and hyphens only.
- A **title**, which is free text and is what you see in the gallery.

Everything else is optional and editable later: a **description** shown on the card and the dashboard's own page, a **type** that the gallery turns into filter tabs, **tags** that gallery search matches on, and the **servers the dashboard will call** — see [Declaring requirements](#declaring-requirements).

PAG creates the dashboard, drops in a starter file for your assistant to write into, and returns the URL:

```text
Created dashboard "server-overview" (Server Overview).
URL: http://localhost:<port>/dashboards/server-overview/
Path: <project>/.pag/dashboards/server-overview
Files: dashboard.json, App.jsx
```

The port is derived from your project's identity, so the URL is normally stable across restarts. If that port is already in use, PAG chooses a random free port; the URL PAG prints is always authoritative. See [The dashboard](using-pag.md#the-dashboard).

## The authoring loop

A well-behaved assistant works in this order. It is worth knowing, because a dashboard that renders nothing usually means a step was skipped:

1. **Create** the dashboard, declaring the servers it will call.
2. **Read the design system** — `ui_builder__get_design_system` returns PAG's component library and styling reference. The assistant should always do this before writing code, or it will invent styles that don't exist.
3. **Write the `.jsx` files** into the dashboard's directory.
4. **Validate** — `ui_builder__validate_dashboard` runs the same build PAG uses to serve the page, so it catches syntax errors and bad imports before you ever load it.
5. **Return the URL.**

Once you have looked at the dashboard, iterate by asking for changes. There is no live reload — PAG rebuilds on each request, so a plain browser refresh picks up whatever the assistant just wrote. You never need to restart PAG or your client.

If the assistant calls a tool to see what it actually returns before rendering it, that is a good sign rather than a detour. Guessing the shape of a tool's response is the most common cause of a dashboard that builds cleanly and renders nothing.

It is sometimes worth giving your AI client browser troubleshooting tools, such as Playwright, on top of that. This pays off when a server returns differently shaped data across its own tools — one tool answers with a raw string while the others answer with JSON, say.

An assistant tends to assume the second response looks like the first, and nothing guarantees it will. Letting it look at the rendered page cuts the number of turns it takes to sort that out.

## Viewing and managing dashboards

The gallery is at `/dashboards/` in the PAG dashboard.

### The gallery

Each dashboard gets a card with a thumbnail, its title, its type badge, its description, and up to three tags. PAG captures the thumbnail automatically when a human opens the dashboard and refreshes it on later visits. Until the first capture, you get a colored placeholder.

The controls above the cards:

- **Search**, filtering on title, description, and tags.
- **Type tabs**, generated from the `type` values actually in use, plus **All**.
- **Refresh**, with a "last refreshed" label. The gallery also polls every 30 seconds and re-checks when you return to the tab, so a dashboard your assistant just created shows up on its own, briefly highlighted.
- **Import**, for installing a dashboard exported from another project. See [Sharing dashboards](sharing-dashboards.md).

There is no pagination and no sort control — the listing is sorted by title.

### Per-dashboard actions

Every dashboard's actions menu carries the same four items, in the same order, on both the gallery card and the dashboard's own page:

| Action | What it does |
| --- | --- |
| **Duplicate** | Copies every file to a new dashboard, appending " (copy)" to the title. The copy counts as a different dashboard from then on |
| **Export** | Packages the dashboard for another project. See [Sharing dashboards](sharing-dashboards.md) |
| **Change URL** | Changes the slug and the directory name. **Bookmarks and cross-dashboard links using the old URL stop working** — the dialog says so |
| **Delete** | Permanently removes the directory and all its files. Not undoable, and PAG keeps no revision |

Note the naming: **"Change URL" is not the same as changing the title.** The title is edited inline on the dashboard's own page — click the pencil beside it — and changes nothing about the URL or the directory. Type, tags, and description are editable inline the same way.

The list of servers a dashboard needs is deliberately **not** editable in the browser. Ask your assistant to change it, which keeps the declaration and the dashboard's code in the same pair of hands.

## Declaring requirements

Every dashboard carries a list of **the servers it needs** — one entry per name its code calls. That list does two jobs:

- It is what makes the dashboard shareable with a differently configured project. Export refuses a dashboard that declares nothing, because it has nothing to tell a recipient. See [Sharing dashboards](sharing-dashboards.md).
- It lets you point a name at a different server of yours without touching the dashboard's code.

PAG does not read the code to work the list out. Your assistant declares it, when the dashboard is created or later:

> "Add recent blazemeter reports to server-overview dashboard."

Your assistant will update the dashboard's code and the list together.

Each entry can also carry two optional things:

- A **note** — free text for whoever imports the dashboard later. This is the only place to say "you'll need your own API token for this one".
- **Which of your servers the name resolves to.** Leave it out and the name is taken as the alias itself. Setting it is how you aim a dashboard at a different configured server without editing its code.

A dashboard built before PAG v2026.6 has no such list, and needs a one-time upgrade before it can be exported or duplicated. See [Upgrading a dashboard built before portability](sharing-dashboards.md#upgrading-a-dashboard-built-before-portability).

## Reusable components

A component is a `.jsx` file exporting one React component, importable by any dashboard as `@components/<Name>`.

There are two places one can live, and the difference matters:

| Location | Scope | How it's created |
| --- | --- | --- |
| `.pag/ui-components/<Name>.jsx` | Shared across every dashboard in the project | `ui_builder__create_component` |
| `.pag/dashboards/<name>/components/<Name>.jsx` | That one dashboard only | The assistant just writes the file |

`@components/<Name>` resolves **dashboard-local first**, then the shared pool. So a dashboard overrides a shared component by carrying its own file of the same name — that is a supported move, not a mistake. Validation emits a warning naming both files so the override is never silent, and the dashboard-local file is the one that gets bundled.

Notes:

- A shared component can have its own CSS file beside it and import it; the styles fold into every dashboard that uses the component.
- A component's JSDoc header (`@component`, `@description`, `@props`) improves search results but is optional — the component works without it.
- **Dashboards cannot import from each other.** Shared code goes in the pool.

## Where the files live

```text
.pag/
  dashboards/
    server-overview/
      dashboard.json        metadata and requirements -- PAG writes this
      App.jsx               the entry point -- PAG scaffolds, the assistant rewrites
      StatusCard.jsx        any other files the assistant writes
      styles.css            your styles, if the assistant wrote any
      components/           dashboard-local components
  ui-components/            the shared component pool
  cache/
    thumbnails/             gallery thumbnails -- machine-local
```

Only two things are required: `dashboard.json` at the root, and a root React component. Everything else is up to the dashboard.

### Version control

`.pag/dashboards/` and `.pag/ui-components/` are **meant to be committed**. PAG deliberately leaves them out of the `.gitignore` entries it manages, which cover `config.local.toml`, `cache/`, `exports/`, and lock files.

So: thumbnails stay out of your repository, dashboards and shared components go in. There is no notion of a private or draft dashboard — if it's in the project, it's shared with the project. And there are no timestamps in `dashboard.json`, because git already has that history.

## Sharing within a project

Commit `.pag/dashboards/` and `.pag/ui-components/`. Anyone who checks out the project and runs PAG gets the dashboards, at the same paths, with no import step. For the dashboards to work, the required server aliases must also be available in that checkout. Commit those definitions in `.pag/config.toml`; aliases that exist only in global configuration or `.pag/config.local.toml` do not travel with the project. If a dashboard was built against either global or local aliases it will be non-functional for project repository collaborators.

Handing a dashboard to a project that is configured *differently* is a separate flow, because the server names won't line up. See [Sharing dashboards](sharing-dashboards.md).

## Turning authoring off

The UI Builder's tools can be disabled like any other server — ask your assistant, or use the dashboard's Servers page. They are enabled by default.

Disabling them removes the authoring tools from your assistant's reach. It does **not** take your dashboards away: the gallery, the dashboard pages, and the rendering pipeline all keep working, because those are part of PAG's web UI rather than MCP tools. The gallery shows a notice saying authoring is off and where to turn it back on.

`ui_builder__*` tools will not appear in your AI client's tool list. They are not published to your client, on purpose — same reason PAG doesn't publish your downstream servers' tools. Your assistant is required to discover them with `pag__explore` and calls them through `pag__execute`, like any other tool behind the gateway. See [Tools and the sandbox](using-pag.md#tools-and-the-sandbox).

`ui_builder` is a reserved alias, and the built-in server cannot be deleted.

## The UI Builder tools

For reference, the twelve tools your assistant has. You call these by asking, not by name.

| Tool | What it does |
| --- | --- |
| `ui_builder__create_dashboard` | Scaffold a new dashboard: directory, `dashboard.json`, skeleton entry file |
| `ui_builder__list_dashboards` | List every dashboard with its title, type, and URL |
| `ui_builder__update_dashboard` | Edit the title, description, type, tags, entry file, and the list of servers the dashboard needs. Anything not supplied is left alone |
| `ui_builder__rename_dashboard` | Change the name in the URL, and the directory with it |
| `ui_builder__duplicate_dashboard` | Copy a dashboard under a new name, as a separate dashboard |
| `ui_builder__delete_dashboard` | Remove a dashboard and its files |
| `ui_builder__validate_dashboard` | Run the real build and report errors and warnings |
| `ui_builder__get_design_system` | Return the styling and component reference |
| `ui_builder__create_component` | Scaffold a shared component in `.pag/ui-components/` |
| `ui_builder__list_components` | Search built-in, shared, and dashboard-local components |
| `ui_builder__export_dashboard` | Package a dashboard for another project. See [Sharing dashboards](sharing-dashboards.md) |
| `ui_builder__import_dashboard` | Stage a dashboard another project exported, show you the review, then publish it once you decide. See [Sharing dashboards](sharing-dashboards.md) |

There is no read tool and no file-read or file-write tool. Your assistant uses its own file tools for the code, and the UI Builder handles everything structural.
